# 05 Configuration Storage

Status: draft. Implements tag-model.md section 9 ("the authoritative store is PostgreSQL, `config_revisions`, one row per applied revision; export then import produces an empty diff; applied atomically as revision N+1 or rejected") and the draft/apply/restore flow of api.md section 3. The compile, validation and diff logic live in the tag engine ([registry.md](../01-tag-engine/registry.md)); this file stores their inputs and outputs.

## 1. What is stored

The unit of storage is the **document**: one `Gateway`, `Connection`, `Template` or `TagGroup` YAML document in its decoded, canonical form, after the config service has written minted ids back into it (registry.md section 5: `id` on plain tags, `tag_ids` on instances). Templates are stored unexpanded; expansion is a pure function of the stored documents and is recomputed on every compile.

A **revision** is an ordered set of `(kind, name, document)`. Two consecutive revisions usually share almost all documents, so documents are stored once, by content hash, and a revision is a list of references.

```sql
-- core/0005_config.sql
CREATE TABLE config_objects (
  sha256       bytea       PRIMARY KEY,                 -- SHA-256 of canonical JSON of doc (section 3)
  kind         text        NOT NULL CHECK (kind IN ('Gateway','Connection','Template','TagGroup')),
  name         text        NOT NULL,                    -- metadata.name
  doc          jsonb       NOT NULL,
  size_bytes   integer     NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE config_revisions (                          -- one row per applied revision (contract wording)
  revision        bigint      PRIMARY KEY,               -- gapless: previous + 1, assigned under the config lock
  applied_at      timestamptz NOT NULL,
  applied_by      text        NOT NULL,                  -- 'user:alice', 'system:bootstrap', 'service:git-sync'
  comment         text        NOT NULL DEFAULT '',
  base_revision   bigint,                                -- NULL for revision 1
  restored_from   bigint,                                -- set when produced by a restore
  git_commit      text,                                  -- recorded by 05-config-as-code (deployment README)
  manifest_sha256 bytea       NOT NULL,                  -- SHA-256 over the sorted (kind, name, sha256) list
  summary         jsonb       NOT NULL,                  -- {"tags_added":9,"tags_changed":0,"tags_removed":0,...}, same as the event
  audit_seq       bigint,                                -- audit_log.seq of the config.applied record
  retention_days  integer                                -- value of Gateway.spec.history.default_retention at this revision, denormalized for warnings
);

CREATE TABLE config_revision_items (
  revision  bigint NOT NULL REFERENCES config_revisions(revision),
  kind      text   NOT NULL,
  name      text   NOT NULL,
  sha256    bytea  NOT NULL REFERENCES config_objects(sha256),
  PRIMARY KEY (revision, kind, name)
);
CREATE INDEX config_revision_items_sha ON config_revision_items (sha256);

CREATE TABLE config_drafts (
  draft_id       uuid        PRIMARY KEY,                -- UUIDv7
  base_revision  bigint      NOT NULL,
  created_by     text        NOT NULL,
  created_at     timestamptz NOT NULL,
  expires_at     timestamptz NOT NULL,                   -- created_at + 7 days; daily job deletes expired
  comment        text        NOT NULL DEFAULT '',
  documents      jsonb       NOT NULL,                   -- the full candidate document set (array of docs)
  validation     jsonb       NOT NULL                    -- {"valid":bool,"errors":[...],"diff":{...}} as returned by the API
);
```

- There is deliberately **no `config_head` table**. The head is `SELECT max(revision) FROM config_revisions` (index lookup). "Config table empty" stays the exact bootstrap condition deployment 04 relies on (`if config_revisions is empty: import /etc/skyranger/bootstrap/*.yaml as revision 1`).
- Revisions are never updated or deleted by the application. `sr_app` has `INSERT` and `SELECT` on `config_revisions` and `config_revision_items`, no `UPDATE`/`DELETE`. A deletion trigger is not needed because the privilege is not granted; the audit chain protects against a superuser (section 6).
- Size arithmetic (09, section 8): a 10,000-tag site is roughly 2.5 MB of YAML; as jsonb with TOAST compression, about 1 MB. Editing one `TagGroup` of 500 tags costs one new object of about 60 KB plus a manifest of a few dozen item rows. A thousand revisions is about 100 MB at the extreme and usually a few MB.

## 2. Apply: one transaction

Compile and validation run **before** the transaction, as a pure function (registry.md section 1). Their output is: the final documents with ids written back, the resolved tag set, and the summary. The transaction only stores:

```
BEGIN;
  SELECT pg_advisory_xact_lock(7301);                        -- the config lock: serializes applies, one key for the whole system
  head := SELECT coalesce(max(revision), 0) FROM config_revisions;
  if head != draft.base_revision  ->  ROLLBACK, return ErrConflict (api 409 conflict)
  INSERT INTO audit_log  ...  action 'config.applied' (hash chain, under the audit lock)  -> audit_seq
                              -- 'after' = {"revision": head+1, "manifest_sha256": ...}, both known before any config row exists
  INSERT INTO config_objects (...)  ON CONFLICT (sha256) DO NOTHING;      -- changed or new documents only
  INSERT INTO config_revisions (revision = head+1, ..., audit_seq);
  INSERT INTO config_revision_items ...;                     -- full manifest (all documents of the revision)
  UPSERT tag_catalog ...  (live tags: path, kind, datatype, unit, history cfg, last_revision = head+1)
  UPDATE tag_catalog SET removed_revision = head+1, removed_at = now()
         WHERE removed_revision IS NULL AND tag_id <> ALL ($live_ids);
  UPDATE derived_state SET orphaned_since = now()
         WHERE tag_id <> ALL ($live_ids) AND orphaned_since IS NULL;
  UPDATE derived_state SET orphaned_since = NULL WHERE tag_id = ANY ($live_ids) AND orphaned_since IS NOT NULL;   -- restored tags
  INSERT INTO events     ...  skyranger.config.revision_applied.v1
COMMIT;
```

Properties:
- **All or nothing:** a crash at any point leaves the previous revision as head. The tag engine hot-reload happens **after** commit and uses the in-memory compile result; if `skyrangerd` crashes between commit and reload, startup loads the head revision and recompiles (registry compile is deterministic).
- **Optimistic concurrency** is exactly the `409` of api.md: the lock serializes, the `base_revision` check detects staleness.
- **Audit and event in the same transaction** (07-security/08 section 4: "audit and change are in the same transaction where possible"). If the audit insert fails, the apply fails. The `config.applied` audit record's `after` contains `{"revision": N, "manifest_sha256": "..."}`, so the audit hash chain also protects the config history: altering a stored revision changes its manifest hash and no longer matches the chained record.
- **Rename and path swaps:** `tag_catalog_live_path` is a unique partial index that cannot be deferred. A revision that swaps two paths (`a` becomes `b` and `b` becomes `a`) would violate it mid-statement. The catalog update is therefore two statements inside the transaction: first set the `path` of every renamed tag to `'~' || tag_id` (the `~` character can never appear in a valid path, so it cannot collide), then set final paths.
- **No-op revisions** (identical manifest hash) are rejected before the transaction with `422 validation_failed` (`code: no_change`), so the revision counter means something. (The registry's diff already computes this.)
- The advisory lock key `7301` is documented here so no other code reuses it. Audit uses its own key (07-security/08 section 2 does not name one; proposed `7302`).
- Long applies (a 100,000-tag catalog upsert) are bounded by `statement_timeout` of the `state` pool (10 s). The catalog upsert is chunked in 5,000-row statements within the same transaction; measured cost is expected at about a second per 100,000 rows (UNVERIFIED).

## 3. Canonical form and hashing

The document hash must be stable across Go versions, platforms and map iteration order.

- Canonical JSON: keys sorted bytewise, no insignificant whitespace, UTF-8, numbers rendered by a fixed rule (integers as integers, floats by the shortest round-trip representation `strconv.FormatFloat(v, 'g', -1, 64)` normalized to JSON number syntax). One function in package `canonjson`, **shared with the audit hash chain** (07-security/08 leaves JCS versus a frozen struct encoding open; that decision must be made once for both). RFC 8785 (JSON Canonicalization Scheme) is the candidate; its exact number serialization rules are UNVERIFIED here, and it is acceptable to freeze our own simpler rule because only our code ever computes the hash.
- 64-bit integers in documents (`modulus: 4294967296`, `max_delta`) are stored in `jsonb` as exact numerics. The canonical form writes them as integers. A `uint64` above 2^63 in a document stays exact in `jsonb` (arbitrary precision `numeric`) and in canonical text.
- Durations are canonicalized by the registry before hashing (`150s` becomes `2m30s`, registry.md section 4), so equivalent spellings hash identically.
- `manifest_sha256` is the SHA-256 of the concatenation of `kind NUL name NUL sha256 LF` over items sorted by `(kind, name)`.

## 4. Export and import fidelity

- `GET /api/v1/config` renders the head revision as multi-document YAML in a **fixed order** (kinds in the order Gateway, Connection, Template, TagGroup; documents by `metadata.name`; map keys in the schema's declared order, not alphabetical, so the output reads like hand-written config). Because the order and formatting are deterministic, two exports of the same revision are byte-identical, and the `manifest_sha256` is printed in a header comment (`# skyranger config revision 43 manifest sha256:...`) so a file can be checked against the database.
- **Lossless means semantic**, as the contract defines it (export then import yields an empty diff). It does **not** mean comment-preserving: comments and the author's formatting are not stored in the database. They are preserved by the **git mirror** (deployment README decision 6; the Git commit of every apply is recorded in `config_revisions.git_commit`), which is the engineering workflow of record. This is stated on the config page so no one is surprised that a comment typed in the HMI editor disappears from the next export.
- `include_ids=true` (API default) emits `id`/`tag_ids`; the importer matches by id, then by path (registry.md section 5).
- **Property test (mandatory):** generate random valid document sets, apply as revision, export, re-import into a scratch database, assert `manifest_sha256` equality and an empty diff. Also fuzz YAML with anchors and aliases (rejected by the parser per registry rules) and Unicode normalization (`display_name` is arbitrary UTF-8 and must round-trip byte for byte).
- **Secrets never appear:** the schema requires `secret://` references for credential fields, validation rejects literals in those fields, and storage never sees a resolved secret. A defense-in-depth check in the `ConfigStore.Apply` implementation refuses any document containing a key literally named `password`, `secret`, `token` or `api_key` whose value does not start with `secret://` (cheap, catches driver schema mistakes).

## 5. Reading history of configuration

| Question | Query |
|---|---|
| List revisions | `SELECT revision, applied_at, applied_by, comment, summary FROM config_revisions ORDER BY revision DESC LIMIT n` (keyset on `revision`) |
| Content of revision N | join `config_revision_items` to `config_objects` for `revision = N` |
| What changed in revision N | compare item hashes of N and N-1; for changed documents diff the jsonb; tag-level via the registry diff, which the API recomputes on demand |
| Which revisions contained document X version V | `config_revision_items WHERE sha256 = V` |
| Who changed tag T's deadband | walk revisions backwards through the document containing `T` (the `Origin` field in the registry) until the field value changes; an offline helper in `skyrangerctl config blame <path>` (proposed) |

## 6. Restore, redundancy, and drift

- **Restore of a revision** (`POST /config/revisions/{r}/restore`) creates a draft whose documents equal revision `r`, with `base_revision` = current head. Applying it produces a **new** revision N+1 with `restored_from = r`; history is never rewritten. Tags that existed in `r` but not in head come back with their old `tag_id` and keep history and `derived_state` (registry.md section 5), and `derived_state.orphaned_since` is cleared by the apply transaction above.
- **Three copies of the truth**, on purpose, because config is the one thing a customer cannot regenerate:
  1. PostgreSQL (`config_*` tables), authoritative at runtime.
  2. `/var/lib/skyranger/state/config-snapshot.yaml`: a full export of the head, rewritten after every committed apply by atomic rename (write temp, fsync, rename, fsync dir), with `revision` and `manifest_sha256` in its header. Specified in [02-containers.md](../08-deployment/02-containers.md) section 3 and [01-topologies.md](../08-deployment/01-topologies.md) T2; this file defines the write discipline. It lets a gateway boot with correct configuration even from an empty database (deployment 04 bootstrap import).
  3. The git mirror (deployment 05), including comments, off-box if configured.
- **Drift detection at start:** compare `max(revision)` and its `manifest_sha256` in the database against the snapshot header. If they differ (database restored from an older backup, or a snapshot from a different box), `skyrangerd` starts with the **database** content, emits `skyranger.config.drift_detected.v1` (proposed) and raises a medium alarm; it never silently overwrites either side. The engineer chooses with `skyrangerctl config adopt-snapshot` (creates a draft from the snapshot) or accepts the database.
- **Database loss with no backup:** import the snapshot or the git head as revision 1 of the new database. `tag_id`s are preserved because exports carry ids (`include_ids=true`), so history exports taken earlier and `derived_state` snapshots (07) still line up.

## 7. Drafts

- Drafts live in the database (`config_drafts`) so they survive a restart and can be inspected by another engineer. They are **not** audited per edit (a draft is not a change; `config.draft_created` and `config.draft_deleted` are, per 07-security/08).
- Limits: 20 open drafts per user, 7 day expiry, 16 MiB per draft. The daily job deletes expired drafts.
- A draft pins `base_revision`. If the head moves, the draft is still readable and re-validatable (`POST /config/drafts` with the new base), and apply fails with `409` until it is rebased (api.md). Rebase is the config service's merge logic, not storage's.

## 8. Go surface

See 11, section 4, for `ConfigStore`: `Head`, `Revision(n)`, `ListRevisions`, `Apply(ctx, ApplyRequest) (RevisionMeta, error)`, draft CRUD, `ExpireDrafts`. `Apply` takes an `ApplyRequest` that already contains the compiled catalog rows, the summary and the audit and event payloads, so the store can run the whole transaction without importing the tag engine or the audit package.

## Decisions made
1. Documents are stored once by content hash; a revision is a manifest of references. One `config_revisions` row per revision (contract wording), items in a child table.
2. No separate head pointer: head is `max(revision)`, serialized by one advisory lock.
3. Revisions are append-only by privilege; the `config.applied` audit record carries the manifest hash so tampering is detectable.
4. Config apply writes revision, catalog, state orphan marks, audit record and event in one transaction.
5. Export is deterministic and semantically lossless, not comment-preserving; comments live in the git mirror.
6. Three copies of the config (database, snapshot file, git); disagreement is reported, never auto-resolved.
7. Restore always creates a new revision.

## Alternatives rejected and why
- **Full copy of all documents per revision**: simplest, but a 10,000-tag site stores megabytes per edit and thousands of revisions become gigabytes of near-duplicates.
- **Diff or patch chains** (revision N = revision N-1 plus a patch): small, but reading revision N costs N patch applications and one corrupt link breaks everything after. Content addressing gives random access and cheap integrity.
- **Config as YAML text blobs in the database**: preserves comments, but the database can no longer query or validate it, and canonical hashing needs a parse anyway. Comments are better served by git.
- **Per-tag rows as the primary config store** (relational schema for tags, templates, instances): attractive for SQL, but the contract's unit is the YAML document and export/import must be lossless; a relational decomposition invents a second schema to keep in sync. `tag_catalog` is a derived index, not the source of truth.
- **Git as the only store**: great for history and review, but atomic apply with optimistic concurrency, audit in the same transaction, and a running gateway that must work without git are all easier in PostgreSQL.
- **A `config_head` pointer table**: a second row to keep consistent with the revision table and a place for them to disagree; `max(revision)` has no such state.
- **Deleting old revisions after N**: saves almost nothing and destroys evidence.

## Open questions
- `canonjson`: freeze our own rule or adopt RFC 8785 (UNVERIFIED details). Shared with audit; needs one decision.
- Event and audit additions: `skyranger.config.drift_detected.v1`, advisory lock key assignment (`7301` config, `7302` audit proposed), `skyrangerctl config blame`, `adopt-snapshot`.
- Whether `tag_catalog` should also record the `Origin` (document, template, instance) of each tag for faster "blame" and HMI drill-down; registry already computes it, so it is cheap to store, but it grows the catalog row.
- 16 MiB per document and about 5,000 tags per `TagGroup` as soft guidance: confirm against a 100,000-tag generated config in the performance tests (tag-engine performance doc).
- Is `retention_days` denormalized on the revision useful, or should the retention-shortening warning be computed only from the previous revision's document? Kept now for cheap warnings; remove if unused by milestone 2.
- Multi-gateway central: does central store each gateway's `config_revisions` as-is (gateway_id column) or as a separate fleet config? Not specified until cloud sync is designed.
