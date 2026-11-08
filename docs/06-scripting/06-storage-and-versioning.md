# 06 Script storage and versioning

Status: design draft. Conforms to [../00-contracts/](../00-contracts/README.md), especially tag-model.md section 9 (config revisions, drafts, lossless export) and api.md section 3 (Configuration). Depends on P2.

## 1. Scripts are config

`Script` and `ScriptLibrary` documents (03 section 1) are documents in the config set, stored in `config_revisions` like `Gateway`, `Connection`, `Template` and `TagGroup`. Therefore, with no extra machinery:

- **Drafts and diffs.** `POST /config/drafts` validates the whole document set; the diff lists `Script/leak_webhook` as `changed` with `fields: ["source", "permissions"]`.
- **Atomic apply** as revision N+1, optimistic concurrency on `base_revision` (409 if stale).
- **History, author, comment** per revision (`GET /config/revisions`), `restore` creates a draft equal to an old revision. Rolling back a script is restoring a revision.
- **Audit**: every apply is audited with user and diff (api.md section 2).
- **Lossless export/import:** export then import yields an empty diff (tag-model.md section 9). `source` is a YAML block scalar; its exact bytes (including trailing newline policy) round-trip. The importer does not reformat code.
- **Secrets**: only `secret://` references appear (tag-model.md section 9).

Not in config revisions: `state` KV values, run records, logs (runtime data, not configuration). `state` is in backups.

A revision touching only scripts does not restart drivers or disturb tags: the tag engine's hot reload leaves unchanged tags alone (tag-model.md section 9), and the script host swaps only changed `ScriptRev` snapshots.

## 2. Export layout for git

`GET /api/v1/config` returns a multi-document YAML (the lossless export). For teams that keep config in git, an additional export mode splits large scripts into their own `.py` files:

```
GET /api/v1/config?layout=split   (Accept: application/zip or application/x-tar)

config/
  gateway.yaml                         # Gateway
  connections/local-sparkplug.yaml
  templates/pulse_water_meter.yaml
  taggroups/pumphouse.yaml
  scripts/leak_webhook.yaml            # Script, with source_file: leak_webhook.py
  scripts/leak_webhook.py              # the code, byte-for-byte equal to spec.source
  libraries/units.yaml                 # ScriptLibrary with source_file: units.py
  libraries/units.py
  MANIFEST.json                        # {"revision": 43, "exported_at": "...", "files": {path: sha256}}
```

```yaml
# scripts/leak_webhook.yaml
apiVersion: skyranger/v1
kind: Script
metadata: { name: leak_webhook }
spec:
  source_file: leak_webhook.py        # relative to this YAML file
  triggers: [ ... ]
  permissions: { ... }
```

- Import accepts the same layout (a tarball or zip, or a directory through a CLI that uploads it), resolves `source_file` into `source`, and stores the resolved form. The stored revision always has inline `source`; `source_file` is an export/import affordance only, so the database has one representation.
- The `.py` files are real Python, so editors, `ruff`, `mypy`, and `pytest` work on them (07). That is the point of the split layout.
- `MANIFEST.json` has hashes so a CI job can verify a checkout equals revision N.
- Path rules: `source_file` must be a relative path without `..` and must stay inside the export root; the importer rejects anything else (path traversal).
- Round-trip test (conformance, 09-testing): export split, import split, diff is empty.
- A GitOps flow (git is the source of truth, the gateway applies) is possible with the API: CI posts a draft from the checkout and applies. The gateway itself does not pull from git in v1 (no outbound network assumptions; edge-first).

## 3. Validation at draft time

Config validation (api.md `validation_failed`, 422, JSON Pointer errors) for `Script`/`ScriptLibrary`, executed by `skyrangerd`, with the Python-specific checks delegated to scriptd:

| Check | Where | Pointer example |
|---|---|---|
| Schema (types, enums, durations parse, name grammar) | host (Go, JSON Schema) | `/spec/timeout` |
| Exactly one of `source`, `source_file` | host | `/spec` |
| `compile(source)` succeeds (syntax) | scriptd `Validate` | `/spec/source` with `line`, `col` in `message` |
| Entrypoint exists as top-level `def` taking one parameter | scriptd (AST) | `/spec/entrypoint` |
| Imports are on the allowlist or `libs.*`; every `libs.x` exists in the draft | scriptd (AST) | `/spec/source` |
| Permissions within pool ceiling | host (reads pool ceiling from attach handshake) | `/spec/permissions/http/0/host` |
| Trigger paths valid grammar; exact `paths` resolve to tags in the draft | host | `/spec/triggers/0/tag_change/paths/0` |
| Cron valid, `every` >= 1s | host | `/spec/triggers/1/timer/cron` |
| `params_schema` is valid JSON Schema (subset) | host | `/spec/triggers/2/manual/params_schema` |
| Library import cycles | scriptd | `/spec/source` |
| Unused-permission and over-broad-permission **warnings** (`write: [""]`, prefix matches no tag) | host | warnings array |
| Reference to a removed tag in `permissions` / triggers | host | warning (error for exact `tag_change.paths`) |

The AST check is a static check, not a guarantee: dynamic `__import__("os")` is caught at run time by the import guard (04). `warnings` is an additive field on the draft response (open question 1).

If the scripts container is absent when a draft is validated, the Go-side checks still run, the Python-side checks are **skipped with a warning** `script_validation_unavailable`, and the revision applies; scripts enter state `unavailable`. (Refusing to apply would make config edits depend on an optional container.)

On pool restart or library change, scripts are re-validated against the new set; a script that no longer validates becomes `invalid` (03 section 6) and the host emits `state_changed`.

## 4. Library rules and rename safety

- `libs.<name>` imports are resolved **at apply time** from the revision's `ScriptLibrary` documents, statically. A draft that deletes or renames a library still imported by a script fails validation, with the importing script named in the error. This is the "rename safety" guarantee.
- Libraries do not import scripts. Libraries may import other libraries; cycles are rejected.
- Library version pinning: none in v1. A script uses the libraries of the same revision, so atomic apply keeps them consistent. (Template live-linking has the same property, tag-model.md section 10.)
- A library change restarts workers that loaded it (01 section 3), and the next run of every dependent script uses the new code. Dependents appear in the draft diff as `changed (library dependency)` so an engineer sees the blast radius.
- Renaming a **script** is a delete plus create (like template tag renames): the old `_system/scripts/{old}/...` tags disappear (clear retained topics per messages.md), the run history stays under the old name. A rename keeping history is possible if the script has a stable `id` (UUIDv7) the way tags have `tag_id`:

```yaml
metadata:
  name: leak_webhook
  id: 01933b95-aa10-7b11-8c22-9d33e44f5a66     # written by export; assigned on first apply if absent
```

  Decision: scripts get a stable `id` (UUIDv7) written by export and assigned at first apply, run records key on `script_id` plus the name at run time. Import matching mirrors tags: by `id`, else by name, else new.
- Renaming a **tag** referenced by a script: tag paths are mutable, no redirects (tag-model.md section 1). Scripts hold paths as strings, so a rename silently breaks them. Mitigations: (1) apply-time validation of exact `paths` and a warning for any string literal in the source that *looks like a tag path of a removed tag* (heuristic, best effort); (2) at run time `NotFound` fails loudly; (3) recommended style in docs: keep tag paths in a `ScriptLibrary` constants module (`libs.tagnames`) so a rename is one edit.

## 5. Run-to-revision traceability

Every run record stores `config_revision` and `source_sha256` (and the sha of each library: `libs`), and `ScriptRev` snapshots are immutable and loaded from the revision itself. To answer "what code ran at 03:12 last Tuesday":

1. `GET /scripts/{name}/runs?start=...` gives the run with `config_revision: 41`.
2. `GET /config/revisions/41` gives the full content of that revision, including the source.
3. `source_sha256` proves the match.

Writes carry the same trace through audit: `audit_log` row (principal `script:default/low_flow_flag`, `run_id`) links to the run.

Revisions are retained as long as the config retention policy keeps them; a policy that prunes revisions must keep any revision referenced by retained run records or audit rows (owned by the storage area; flagged as a requirement here).

## 6. Size and format limits

| Item | Limit | Why |
|---|---|---|
| `source` per script/library | 256 KiB | review-sized; protects config diff and pipe |
| Scripts per gateway | 500 | queue and worker accounting |
| Libraries per gateway | 100 | |
| Total script source | 8 MiB | config revision size |
| Source encoding | UTF-8, LF or CRLF preserved as given | lossless round trip |

## Decisions made
1. Scripts and libraries are config kinds stored in the normal revision stream: drafts, diff, atomic apply, restore, audit and lossless export all come for free.
2. The database stores one form (inline `source`); the split `.py` layout is an export/import convenience with a manifest of hashes.
3. Validation runs at draft time; Python-specific checks run in scriptd and degrade to a warning if the container is absent, never blocking config edits.
4. `libs.*` imports resolve statically per revision; deleting a library still in use fails validation; library changes show dependents in the diff.
5. Scripts carry a stable UUIDv7 `id` so run history survives renames (same pattern as `tag_id`).
6. Run records store revision and source hash; any past run can be mapped to the exact code that ran.
7. `state` values and run data are runtime data, outside revisions, inside backups.
8. The gateway does not pull from git; GitOps is "CI pushes drafts through the API".

## Alternatives rejected and why
- **Scripts as files on a mounted volume, outside the DB**: breaks atomic apply with the tags they reference, loses audit and revision history, and drifts between boxes.
- **Scripts in a separate revision stream** (as the HMI store does for screens, 05-hmi-api/02-screen-format.md): scripts reference tags and permissions that must be validated against the same revision; one stream avoids cross-stream inconsistency. (The HMI can tolerate separate revisions because screens only bind to tags by path and degrade visibly.)
- **Git as the database**: needs git on the edge box and breaks the "export, then import yields empty diff" contract.
- **Per-script version numbers chosen by users**: revision numbers already exist; two version schemes confuse.
- **Library pinning in v1**: adds a dependency resolver for little value at this scale.
- **Refuse to apply when the scripts container is down**: makes an optional component mandatory.
- **Rewriting/reformatting code on import** (black/ruff format): destroys round trip.

## Open questions
1. A `warnings` array on the draft response is an additive change to api.md section 3's draft shape; needs recording with P2/P4.
2. The `id` field in `metadata` for Script: tag-model.md `metadata` is `{name, description?, annotations?}`; adding `id` to Script (and later others) is additive, to be recorded.
3. Config revision retention vs referenced revisions (storage area, 03-storage is empty at time of writing).
4. A CLI (`skyranger config push`) for the split layout: out of scope for this area; the API supports it.
5. Heuristic detection of tag-path string literals in source: worth the false positives?
6. Per-environment overrides (same script on dev, prod gateways with different prefixes): templates with params exist for tags; a `ScriptTemplate` or `params` on `Script` might be wanted by integrators with many identical sites (deferred).
