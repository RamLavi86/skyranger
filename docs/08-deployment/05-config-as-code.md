# 05 Config as Code

Binding inputs: [tag-model.md](../00-contracts/tag-model.md) section 9 (YAML schema, revisions, lossless export/import), [api.md](../00-contracts/api.md) (drafts, diff, optimistic concurrency, restore). This document adds **no new config kinds and no new API**. It defines how an integrator manages those documents in git and how the box and the repo stay in agreement.

## 1. The rule: database authoritative, git is the workflow

[tag-model.md](../00-contracts/tag-model.md) makes PostgreSQL `config_revisions` the authoritative store. We keep that. Git is:

1. the **engineering workflow** (branches, review, blame, diff, CI validation),
2. an **off-box mirror** that survives the loss of the box,
3. the **exit artifact** a customer can read without our software ([12-exit-plan.md](12-exit-plan.md)).

Why not make git authoritative: the HMI edits config (discovery approval produces a draft, tag-model/messages flow). A gateway whose truth is a git working tree needs a second apply path, a second validation path and a merge story for HMI edits. With the DB authoritative there is one apply path (`POST /config/drafts` then `apply`), and git follows it.

Two directions, both through the public API:

```
 HMI / API client ──apply──► skyrangerd ──► config_revisions (rev N+1)
                                │ event skyranger.config.revision_applied.v1
                                ▼
                 skyrangerctl agent: GET /config (yaml, include_ids=true)
                                ▼
                 /var/lib/skyranger/config-git  ── commit ("rev 43 by user:alice: <comment>")
                                ▼ (optional, outbound) git push to customer / integrator remote

 engineer ── git push / merge ──► remote ──► skyrangerctl config pull
                                              ▼
                 skyrangerctl config apply: POST /config/drafts (base_revision = last known)
                                              ▼ show diff, require confirmation (or --yes in CI/fleet)
                 POST /config/drafts/{id}/apply  comment "git:<sha> <subject>"
```

## 2. Repository layout

One repository per **site** (gateway) by default. A monorepo with `sites/<gateway_id>/` is allowed and what `skyrangerctl config init --monorepo` produces for integrators with many similar sites; the layout under each site directory is identical.

```
site-repo/
  skyranger-repo.yaml            repo metadata (see below); not applied to the gateway
  config/                        everything here becomes ONE multi-document config revision
    gateway.yaml                 kind: Gateway          (exactly one)
    connections/
      local-sparkplug.yaml       kind: Connection       (one document per file, file name = metadata.name)
      sim1.yaml
    templates/
      pulse_water_meter.yaml     kind: Template
    tag-groups/
      pumphouse.yaml             kind: TagGroup
  host/                          host-level files, applied by skyrangerctl, NOT by the API
    site.yaml                    copy of /etc/skyranger/site.yaml (no secrets)
    sync.yaml                    optional, see 09-cloud-sync.md
    fleet.yaml                   optional enrollment pointer (no keys), see 08-fleet.md
  secrets.manifest.yaml          NAMES of required secret:// references, never values
  scripts/                       Python user scripts, if the scripting design stores them as files (UNVERIFIED, owned by 06-scripting)
  docs/                          site handover notes, wiring, commissioning log
  .gitattributes                 *.yaml text eol=lf
  .github/ or ci/                validation pipeline (section 7)
```

`skyranger-repo.yaml`:

```yaml
repo_version: 1
gateway_id: gw01                 # must equal config/gateway.yaml metadata.name
api_version: skyranger/v1        # config apiVersion this repo targets
min_gateway_version: 0.4.0       # refuses to apply on an older gateway
```

Rules:
- One document per file, file name equals `metadata.name`. This keeps diffs and blame useful. The importer does not care about file layout: `skyrangerctl` concatenates `config/**/*.yaml` in a **fixed order** (kind order Gateway, Connection, Template, TagGroup, then name) into one multi-doc submission.
- **Exports are normalized.** `skyrangerctl config pull` (and the automatic mirror) writes exports through a canonicalizer: fixed key order, 2-space indent, sorted lists where the schema says order is irrelevant (instances by name, tags by name) and *unsorted* where order is meaningful (derived-tag input lists, `inputs` order breaks quality ties per tag-model.md section 4). A no-op re-export produces an empty git diff. This is a testable property: export, import, export, `diff` is empty (matches the lossless round-trip rule in tag-model.md).
- **`tag_id`s are committed.** Exports include `id` and `tag_ids` (`include_ids=true`). Without them, a path rename in git would look like delete plus create, and the tag would get a new UUIDv7 and lose its history link. Engineers must not delete `id:` lines; CI warns on removal. New tags authored by hand omit `id` and receive one at apply; the next automatic mirror commit adds it.
- **No secrets, ever.** Config holds `secret://mqtt/gateway`-style references only (tag-model.md). `secrets.manifest.yaml` lists the reference names a repo needs so that a new box can be checked (`skyrangerctl config check-secrets`) before apply:

```yaml
secrets:
  - ref: secret://mqtt/gateway
    used_by: Connection/local-sparkplug
  - ref: secret://plc1/modbus_password
    used_by: Connection/plc1
```

## 3. `skyrangerctl config` commands

All commands talk to the gateway REST API with an **API key** (`sr_...`, role `engineer`) stored at `/etc/skyranger/secrets/api/skyrangerctl` and created by `skyrangerctl init`. They never write the database directly. That keeps validation, audit (`audit_log`) and the `revision_applied` event identical for every path.

| Command | What it does |
|---|---|
| `config init [--monorepo]` | Create the repo at `/var/lib/skyranger/config-git` from the current revision. `init` (04) calls it after revision 1 is applied. |
| `config export [--out DIR]` | `GET /config` as YAML, split into files per section 2, normalized. No git. Also used by the exit plan. |
| `config status` | Current revision, `HEAD` sha, **drift** state (section 5), pending local changes. |
| `config diff [REF]` | Draft the working tree (or `REF`) via `POST /config/drafts` and print the API's `diff` (document-level and resolved-tag-level). Creates a draft, never applies. Drafts are deleted afterwards. |
| `config apply [REF] [-m MSG] [--yes]` | `diff`, confirm, apply. Uses `base_revision`; on `409 conflict` it stops and says someone applied a newer revision (usually the HMI). Comment is `git:<sha> <subject>` plus `-m`. |
| `config rollback [N]` | `POST /config/revisions/{N}/restore` creates a draft equal to revision N, then apply. Default N is the previous revision. See section 6. |
| `config pull` | `git pull --ff-only` from the configured remote into the local repo. Never merges. |
| `config push` | Push the local mirror to the remote. |
| `config validate DIR` | Offline structural validation (section 7). |
| `config check-secrets` | Compare `secrets.manifest.yaml` with `/etc/skyranger/secrets`. |

## 4. Mirroring HMI and API applies into git

The agent follows `skyranger.config.revision_applied.v1` (messages.md section 4), via `GET /events?type=skyranger.config.revision_applied` with a stored cursor, not via MQTT, so a missed message cannot lose a commit and the agent has no MQTT credentials. On each new revision:

1. `GET /config/revisions/{revision}` (full content of a past revision) fetches the content, so catch-up after downtime commits each missed revision in order.
2. Write normalized files, `git add -A`, commit as author `<applied_by> <applied_by@gw01.local>`, committer `skyrangerctl`, message:

```
rev 43: add meter01 (Commission meter01)

applied-by: user:alice
revision: 43
previous-revision: 42
summary: tags_added=9 tags_changed=0 tags_removed=0
```

3. Commits made by `config apply` from git already match the working tree. The agent recognizes a revision whose comment starts with `git:<sha>` and does not create a duplicate commit; it only records the revision number in a git note (`git notes --ref=skyranger`) on `<sha>`.
4. If a remote is configured, `git push` is attempted with backoff. Push failure never blocks anything and raises a low-severity alert ([10-observability.md](10-observability.md)).

Revision numbers are monotonically increasing integers in the DB; git history is the human-readable log of the same sequence. A revision exists in git within seconds of being applied, as long as the agent is up.

## 5. Drift detection

Drift means **the gateway's current revision is not the content at the repo's `HEAD`** (normalized comparison). Because the HMI can legitimately edit config, drift is information, not an error.

`skyrangerctl config status` and the agent's periodic check (every 5 minutes) compute:

| State | Meaning | Action |
|---|---|---|
| `in_sync` | current revision content equals `HEAD` | none |
| `repo_behind` | gateway has revisions not in git | agent commits them (section 4); transient |
| `gateway_behind` | git has commits not applied | shown as pending; applied by an engineer or by fleet policy `auto` ([08-fleet.md](08-fleet.md)) |
| `diverged` | both changed since the last common point | **never auto-resolved**. Shows both diffs. Engineer picks: apply git over gateway (`config apply`, explicit), or adopt gateway into git (`git merge` the mirror commits, then push). |

The agent exposes `skyranger_config_drift{state=...}` and the HMI System page shows the state. `diverged` for more than 24 h raises a warning alert. The last-applied snapshot used when the DB is down ([01-topologies.md](01-topologies.md) T2) is separate: `state/config-snapshot.yaml` is a safety copy, not the repo.

## 6. Rollback

Rollback never rewrites history. `config rollback 41` when current is 43:

1. `POST /config/revisions/41/restore` returns a draft whose content equals revision 41.
2. Apply creates **revision 44** with content of 41 and comment `rollback to 41`. The tag engine hot-reloads; unchanged tags keep their state (tag-model.md section 9).
3. The mirror commits revision 44. In git this looks like `git revert` of 42 and 43: the history stays linear and shows why.

What rollback does **not** undo (document this to operators):
- Data already collected under the newer config (history rows stay; tags removed by a rollback keep their history keyed by `tag_id`, orphaned but exportable until retention removes it. Retention for orphaned tags is a historian question, see Open questions).
- `derived_state` for derived tags that were removed (counter totals). If revision 43 removed `total_volume` and 44 re-adds it with the same `tag_id`, state resumes only if the historian/engine keeps the state of removed tags for a grace period (tag engine owner, Open questions).
- Anything outside the config: secrets, `host/` files, firmware on devices.
- Template **semantics** changes in a software update. A config rollback does not downgrade the software; use [06-updates-and-rollback.md](06-updates-and-rollback.md).

Rollback is also what the update health gate calls when a config apply, not a software update, is the suspect. `config apply --health-gate 10m` (the default for fleet-driven applies): after applying, the agent watches `/system/health`, connection states and the proportion of `good` tags for the window and auto-restores the previous revision on failure. Manual HMI applies are not gated (an engineer is watching).

## 7. Validation before it reaches a box

Three layers, cheapest first:

1. **Offline structural** (`skyrangerctl config validate DIR`, no gateway needed): YAML parse, `apiVersion`, kinds, identifier grammar (`^[a-z0-9][a-z0-9_-]{0,62}$`), path length limits, duplicate names, template parameter references. It uses the JSON Schemas generated from the Go config types (the same ones the HMI uses). It cannot check driver-specific `settings` or path collisions that depend on template expansion unless it embeds those. It embeds the first-party drivers' JSON Schema (`Info.config_schema`, api.md `/drivers`), so first-party checks work offline. Third-party drivers need `--driver-schema`.
2. **Full validation against a throwaway gateway** (CI): the pipeline starts the release's compose project with profile `sim` (a sandbox `gateway_id`, no devices, no secrets: `secret://` refs resolve to dummy values in a `--ci` mode, UNVERIFIED that the secret resolver can support that; otherwise CI provides a dummy secret dir), submits the repo as a draft, and fails on `valid: false` or on unexpected `diff` shape. This runs exactly the production validator, including driver `ValidateConfig`.
3. **On the box**: `config apply` always drafts first. A draft that fails validation cannot be applied.

CI also checks: `id:` lines not removed, no `secret` values (a regex for common patterns plus the rule that all credential fields must be `secret://`), file name equals `metadata.name`, repo version matches.

## 8. Review and branching convention (recommended, not enforced)

- `main` = what should be running. Boxes follow `main` of their remote (`track: main`) when fleet policy is `auto`, otherwise an engineer applies a specific commit.
- Changes arrive as pull requests; CI runs layer 1 and 2; merge; apply. For urgent field edits through the HMI, the mirror commit lands on a branch `mirror/gw01` (not `main`) so a field edit never silently changes `main`; the integrator merges it, which restores `in_sync`.
- Tags (`git tag site/gw01/commissioned`) mark handover points. The handover document references the tag.

## 9. Config across software versions

The repo pins `api_version` and `min_gateway_version`. Config schema migrations (new `apiVersion` major) are handled in [07-data-and-migrations.md](07-data-and-migrations.md): `skyrangerctl config migrate --to skyranger/v2 DIR` rewrites the repo files; the gateway never reads two config versions at once after the migration step. Within `skyranger/v1` all changes are additive (README conventions), so a repo written for 0.4 applies to 0.9.

## 10. Host-level files

`host/` files (`site.yaml`, `sync.yaml`, `fleet.yaml`) are not in the database. `skyrangerctl host apply` renders `/etc/skyranger/*` from them, validates, restarts the affected unit, and records the repo sha in `/var/lib/skyranger/state/host-applied.json`. Secrets and keys are never in `host/`. This is deliberately small: network interfaces, firewall ranges and NTP servers stay in `site.yaml` because `init` captured them (04 section 3). Everything about the OS beyond that is out of scope for config-as-code (use the customer's own tooling or the appliance image's config partition).

## Decisions made
1. PostgreSQL `config_revisions` stays authoritative. Git is a workflow, mirror and exit artifact. One apply path: the public API.
2. Revisions are mirrored into git automatically by the agent, driven by the events endpoint with a cursor, one commit per revision, `tag_id`s included, normalized so no-op exports are empty diffs.
3. `config apply` from git always drafts, shows the diff, uses `base_revision`, and records `git:<sha>` in the revision comment.
4. Drift has four states and `diverged` is never auto-resolved.
5. Rollback is a forward revision that restores older content. History is never rewritten.
6. Fleet-driven applies are health-gated and auto-restore; manual HMI applies are not.
7. Field edits through the HMI land on a mirror branch, not on `main`.
8. Repo per site by default, monorepo layout supported. Host-level files live in `host/` and are applied by `skyrangerctl`, outside the config API.

## Alternatives rejected and why
- **Git as the authoritative store (GitOps pull-and-reconcile like Flux/Argo)**: needs a reconcile loop that overwrites HMI edits or fights them, and a gateway that cannot start without git access violates edge-first. Fine for Kubernetes; wrong for a plant box with an engineer at a keyboard.
- **Writing config straight into PostgreSQL from git**: bypasses validation, audit and the `revision_applied` event. Rejected: one apply path only.
- **A different DSL (HCL, Jsonnet, Helm-style templating) in the repo**: a second language to learn and a second thing that can diverge from the exported YAML. Templates (UDTs) in the contract already cover reuse. Revisit only if monorepo fleets need cross-site generation, and then as a generator whose output is the plain YAML.
- **Config in files on disk read at start (like nginx)**: loses atomic apply, diff, optimistic concurrency and hot reload semantics from the contract.
- **Auto-merge of diverged states**: a wrong merge changes a plant's alarm limits. Humans decide.
- **Mirroring through MQTT events only**: a missed QoS 1 message at the wrong moment drops a commit; the cursor-based events endpoint cannot.

## Open questions
- **API request (additive)**: `GET /events` cursor semantics must be stable across restarts so the agent can resume (api.md says cursors are opaque; confirm they are durable and ordered). Alternatively a `GET /config/revisions?after=N` filter, which `/config/revisions` lacks today.
- **API request**: step 1 of section 4 relies on `GET /config/revisions/{revision}` returning IDs; confirm `include_ids` applies there too.
- **Secret resolver in CI mode** (section 7): can `skyrangerd` run with dummy secrets for validation? Security doc and config owner.
- Retention and state grace period for tags removed by config (history of orphaned `tag_id`s, `derived_state` of removed derived tags): historian and tag-engine owners. Affects how safe rollback is for counters.
- Scripting storage: are Python scripts config documents, files, or DB rows? Needed for the repo layout (`scripts/` is a placeholder).
- Git on the box: `git` must be installed on the host (Debian package, boring) or `skyrangerctl` embeds go-git (UNVERIFIED maturity for notes and `--ff-only` semantics). Leaning on the `git` binary.
- Canonicalizer ownership: it must live with the config schema code so it cannot drift from the exporter. Proposed: a `skyranger/config` Go package imported by both `skyrangerd` and `skyrangerctl`.
