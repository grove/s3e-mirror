# Mirroring a RockLake Data Lake With s3e-mirror

**Status:** Research report and design proposal.
**Audience:** Same as [DESIGN.md](DESIGN.md) — engineers implementing the system, with introductory sections accessible to non-technical readers (platform leads, SREs, decision-makers).
**Companion document to:** [DESIGN.md](DESIGN.md), which covers the SlateDB mirroring core.

---

## TL;DR

[RockLake](https://github.com/trickle-labs/rocklake) is an open-source lakehouse catalog that puts both *catalog* and *data* in a single object store: catalog state lives in a SlateDB database under one prefix, and the actual Parquet data files live under another prefix. s3e-mirror as described in [DESIGN.md](DESIGN.md) handles the catalog. What it does **not** yet handle is the Parquet layer — and you need both to have a working mirror, because a catalog without its data files is a catalog full of broken pointers.

The good news is that RockLake's data layer is, if anything, *easier* to mirror than the SlateDB layer: Parquet files in RockLake are strictly immutable, content-addressed by UUID, and deleted only by the explicit, audited, and rarely-used `rocklake excise` command. The same two-phase ordering rule that makes the SlateDB mirror safe — *copy data first, publish manifest last* — generalizes cleanly to Parquet: copy Parquet files first, then mirror the catalog snapshot that references them. The end result is a RockLake warehouse at the target that is openable, queryable, and time-travelable to any snapshot the mirror has captured.

This document proposes a concrete design for that extension, called **s3e-mirror-rocklake**, and lays out the operational rules, failure modes, configuration shape, and small upstream changes to RockLake that would make the whole thing simpler. As with s3e-mirror itself, none of the proposed RockLake changes are required to ship; each one just removes friction.

---

## How To Read This Document

- **Part I — The Problem and the Shape of the Solution** is the elevator pitch. It explains why mirroring RockLake is a two-layer problem and what the clean answer looks like.
- **Part II — RockLake Facts** is the data model: what RockLake actually writes to an object store, what the immutability and snapshot guarantees are, and how paths are resolved. Skim if you already know RockLake.
- **Part III — Why This Is Tractable** is the conceptual argument that mirroring RockLake reduces to mirroring SlateDB plus a clean second copy stage.
- **Part IV — Three Architectures, And The One We Recommend** is the design trade-off discussion.
- **Part V — The Design** is the precise system: data flow, snapshot pinning, path translation, excise handling, configuration.
- **Part VI — Failure Modes** is what goes wrong and how the system recovers.
- **Part VII — RockLake Changes That Would Help** is the upstream wish list.
- **Part VIII — Roadmap and Open Questions** is what we are deferring.

---

# Part I — The Problem and the Shape of the Solution

## RockLake In One Paragraph

A RockLake warehouse is two things sitting in the same bucket: a **catalog**, which is a SlateDB database that records every table, every column, every snapshot, and a pointer to every Parquet file the warehouse owns; and a pile of **Parquet data files**, which is where the actual rows live. The catalog answers "what tables exist, what is their schema, and which files contain their rows," and the Parquet files answer "what are those rows." DuckDB clients connect to a stateless RockLake binary over the PostgreSQL wire protocol, the binary reads the catalog to plan the query, and DuckDB reads the Parquet files directly from object storage to execute it. The whole architecture rests on a single load-bearing premise: committed catalog facts and committed Parquet files are both *never overwritten in place*. New writes always produce new bytes; old bytes are retired logically, not physically.

## The Mirror Problem In One Paragraph

To mirror a RockLake warehouse, you need to keep a copy of *both layers* in the target object store, and you need them to stay consistent with each other. A target catalog that references a Parquet file the target does not have is a corrupt warehouse — queries will fail with "object not found." A target that has the Parquet file but not the catalog row that references it is just wasted space. The mirror's job is to make sure that, at every moment, the target catalog only ever names Parquet files that are *already present* on the target. As with mirroring SlateDB itself, the ordering rule does all the heavy lifting: data first, catalog second, and the target is always a valid warehouse.

## The Shape of the Solution

The clean answer is to extend s3e-mirror with a thin layer that knows about RockLake's data file layout. Call it **s3e-mirror-rocklake**. It piggybacks on s3e-mirror's existing safety machinery — the lease, the queue, the two-phase rule, the verification — and adds two things on top:

1. **A data-file resolver** that, given a translated catalog snapshot the mirror is about to commit, walks the snapshot's `ducklake_data_file` and `ducklake_delete_file` rows, extracts the Parquet paths, and enqueues copy jobs for any that are not yet on the target.
2. **A snapshot pin** at the source RockLake catalog, analogous to s3e-mirror's SlateDB checkpoint, that prevents `rocklake excise` from deleting Parquet files the mirror still needs while it is mid-copy.

Everything else — the worker pool, the deterministic destination naming, the create-if-absent commits, the lease, the recovery — is shared with the existing s3e-mirror codebase. From the operator's point of view, mirroring a RockLake warehouse is a single tool with a single config file: it just happens to manage two object prefixes instead of one.

---

# Part II — RockLake Facts

This section is the minimum data model you need before the rest of the design makes sense. Every fact here was verified directly against the RockLake source at `/Users/geir.gronmo/projects/rocklake2` rather than inferred from documentation.

## Storage Layout

A RockLake warehouse uses two distinct object-store prefixes under a single bucket root:

| Prefix              | What lives there                              | Naming                                                   |
| ------------------- | --------------------------------------------- | -------------------------------------------------------- |
| `{catalog_prefix}/` | A single SlateDB database (the catalog)       | SlateDB-internal: `wal/`, `compacted/`, `manifest/`, etc. |
| `{data_prefix}/`    | All Parquet data files and delete files       | `{schema_id}/{table_id}/{partition}/{file_uuid}.parquet` |

There is **no separate CDC stream** in the object store — RockLake's CDC functionality (`cdc.rs`) produces in-memory `CdcSnapshot` JSON values returned to clients, not files on disk. There is **no separate WAL or manifest prefix** for RockLake — those are entirely SlateDB-internal and live under `{catalog_prefix}/`. There are **no per-table sidecar metadata files** — every catalog fact is a key in SlateDB.

This is operationally a very clean layout: there are exactly two prefixes to mirror, and one of them is already a SlateDB database that s3e-mirror knows how to handle natively. The CLI today bundles both prefixes under a single bucket, but the underlying `CatalogPath` type (defined in `rocklake-core/src/path.rs`) supports them being on different object stores entirely — a seam we will exploit.

## Data Files: Format, Immutability, Path Semantics

Data files are **Parquet only**. There are no Avro, ORC, JSON, or binary delete-vector files. Delete files are *also* Parquet — they are Parquet files containing either position-based or equality-based delete markers, following the DuckLake v1.0 specification.

Both data files and delete files are **strictly immutable once committed**. RockLake makes this immutability load-bearing: the `DataFileRow` schema (in `rocklake-core/src/rows.rs`) carries `begin_snapshot` and `end_snapshot` fields that bracket the MVCC visibility window of a file. When a file is "deleted" via a SQL DELETE, RockLake sets `end_snapshot` on the row; the Parquet bytes on disk are not touched, and a reader at any snapshot ≤ `end_snapshot` still sees the file. The only operation that physically removes a Parquet file from object storage is `rocklake excise`, which is explicitly scoped to compliance erasure and is documented to be rare.

The `path` field on `DataFileRow` (tag 3) and `DeleteFileRow` (tag 3) stores either an absolute path or a path relative to the table's data root, distinguished by the `path_is_relative` field (tag 13 on data files, tag 10 on delete files). When `path_is_relative = Some(true)`, the catalog resolves the path through `CatalogPath::resolve_data_path()`, which prepends the warehouse's data prefix. When `path_is_relative = Some(false)` or `None`, the path is treated as an absolute object-store URI. Both forms exist in real warehouses, but only the relative form survives a cross-cloud mirror without rewriting; the absolute form embeds the source bucket name, which is wrong at the target.

## Snapshot Semantics

A RockLake warehouse has exactly one global, monotonically-increasing snapshot counter, allocated through `COUNTER_NEXT_SNAPSHOT_ID` via a CAS loop inside the catalog writer. Every transaction — INSERT, DELETE, ALTER, CREATE TABLE — bumps the counter, writes a `SnapshotRow`, and stamps every mutated row with the new `snapshot_id`. The current snapshot is simply `peek_snapshot_id() - 1`; there is no explicit "tip pointer" file.

This counter is the natural watermark for a replication slot: at any moment, the mirror has copied up to snapshot `N`, and the mirror's job is to advance `N` toward `latest`. Because the counter is global and monotonic, there is never ambiguity about what "snapshot 12345" means, and the mirror's progress is a single integer.

## Garbage Collection vs. Excision

This distinction matters more for RockLake mirroring than almost any other detail. `rocklake gc` advances the `retain_from` floor — it makes queries at very old snapshots refuse to run with `SnapshotOutOfRetention` — but it does **not delete any Parquet files** and does not delete any catalog keys. It is a logical operation. `rocklake excise`, on the other hand, *physically deletes* both the catalog rows past `end_snapshot` and the corresponding Parquet files from object storage. It is intended for GDPR-style "the right to be forgotten" workflows and is gated on `retain_from >= before_snapshot` to prevent deleting still-visible data.

For mirroring, this asymmetry is a gift. As long as the operator runs only `gc` and never `excise`, the source's Parquet files are append-only: every file that was ever committed is still there. The mirror can fall arbitrarily behind without losing data. Only when `excise` is run does the source actually shrink, and that is exactly the moment when the mirror needs the equivalent of a checkpoint pin to stay safe. We design for both cases below.

## Configuration Today

The current RockLake CLI accepts a single `--catalog s3://bucket/path` argument and uses the resulting bucket and store for both catalog and data. The underlying types support separating the two, and we recommend extending the CLI to make this explicit, but it is not required to mirror — the mirror only reads from the source and writes to the target.

---

# Part III — Why This Is Tractable

## The Generalization of the Two-Phase Rule

s3e-mirror's core safety property for SlateDB is: *publish a target manifest only after every file the manifest references is physically present at the target*. For RockLake, that property generalizes to two nested layers:

> For every target catalog snapshot `S` we commit, every Parquet file referenced by `S` (data files and delete files) must be physically present at the target data prefix, **and** every SlateDB SST file referenced by the catalog's manifest at the moment `S` was captured must be physically present at the target catalog prefix.

This is just the two-phase rule applied twice: once to the data layer, once to the catalog layer. The order is total: Parquet files first, then catalog SSTs, then the catalog manifest commit. A reader opening the target at any moment sees a fully-formed warehouse, because the catalog will never reference a missing Parquet file (we copied them first) and a SlateDB reader will never see a manifest that references a missing SST (we copied those next).

## Why Parquet Mirroring Is Easier Than SlateDB Mirroring

The Parquet layer is simpler than the SlateDB layer in three ways. First, Parquet files are *content-addressed by UUID* at write time, so two workers attempting the same copy converge to the same destination filename and overlap harmlessly. Second, there is no per-file metadata that needs round-tripping — a Parquet file is what it is; we copy bytes and we are done. Third, the lifecycle is much simpler: a Parquet file appears once via INSERT and disappears only via `excise`, with no compaction churn between (compaction in DuckLake happens at the SQL level by replacing one set of files with another, but the old files are still referenced by old snapshots and live on until `excise`).

The hard parts of mirroring a Parquet layer are not the parts that are hard for mirroring SlateDB. They are: (a) the cardinality (a warehouse can have millions of small files, many more than a typical SlateDB database has SSTs); (b) the path translation question (relative vs absolute, and what to rewrite); and (c) coordinating with `excise` so that the mirror does not get blindsided by a deletion. All three are addressed below.

## The Snapshot Counter Is The Watermark

Because RockLake's snapshot counter is global, monotonic, and incremented inside every catalog transaction, it gives us the cleanest possible replication watermark: a single `u64` that says "the mirror has fully captured everything through snapshot N." That number can be reported on dashboards, compared to the source's latest, and exposed as `s3e_mirror_rocklake_target_snapshot_id` for alerting. There is no per-table tracking, no compound key, and no ambiguity about ordering.

---

# Part IV — Three Architectures, And The One We Recommend

There are essentially three ways to compose s3e-mirror with a Parquet mirror, and they trade off integration tightness against coordination guarantees.

## Option A — Run s3e-mirror Alongside a Generic File Sync

The minimal-code option. s3e-mirror mirrors the catalog as it would for any SlateDB database. A separate process — `rclone sync`, `aws s3 sync`, a custom worker, anything — mirrors the data prefix on a separate schedule. The two are decoupled.

**Why it does not work as a primary architecture:** there is no coordination between the catalog mirror and the data mirror. The catalog mirror can commit a target snapshot that references Parquet files the data mirror has not yet copied, and queries against the target will fail with "object not found." You can make this *eventually consistent* by running both fast enough and accepting that the target may be temporarily broken during a window, but that is not a foolproof mirror — it is a hopeful one.

This option is, however, useful as a **bootstrap** for very large existing warehouses: do an initial `rclone sync` of the data prefix to seed the target, then switch on the integrated mirror, which will then only have to handle ongoing files. It can also be a stop-gap when the integrated mirror is not yet implemented.

## Option B — Tightly-Integrated Mirror (Recommended)

A single `s3e-mirror-rocklake` binary owns the entire pipeline. It mirrors the SlateDB catalog using the existing s3e-mirror primitives, and inside its coordinator's commit path it inserts a step that resolves the inbound catalog snapshot's Parquet file references and enqueues data-file copy jobs alongside the SlateDB SST copy jobs. The catalog manifest commit waits on both. This is the option that satisfies the two-phase rule by construction.

This is the recommended architecture for production. The rest of the document specifies this design.

## Option C — Push Replication Into RockLake Itself

Add replication as a first-class feature of RockLake: the source RockLake process holds a replication slot per consumer, streams catalog snapshots and Parquet file references to subscribed consumers, and the consumers materialize them at the target. This is the most elegant long-term design — it eliminates the need for an external mirror entirely — but it requires substantial RockLake changes and ties the implementation to RockLake's release cadence.

We treat this as a future direction in [Part VII](#part-vii--rocklake-changes-that-would-help) but do not pursue it as the first deliverable.

---

# Part V — The Design

This is the specification of Option B: `s3e-mirror-rocklake`.

## In Plain English

Imagine a continuously-running process that watches a RockLake warehouse the way a careful archivist watches a growing library. Every time a new snapshot is committed at the source — somebody added a table, ingested a batch of rows, deleted some data — the archivist takes note. Before the archivist updates the destination library's card catalog, it makes sure every new book mentioned in the new card catalog is already physically on the destination's shelves. It does this in parallel, by sending many delivery vans to copy books, and it only updates the destination's card catalog after every delivery van has reported back. If a delivery van breaks down, no card-catalog update happens, and the destination library remains in its last fully-coherent state until the delivery van's job is retaken by another van. The destination is never inconsistent, even mid-update.

## Data Flow

```
Source warehouse                               Target warehouse
┌────────────────────────────┐                 ┌────────────────────────────┐
│  s3://prod-lake/           │                 │  az://dr-lake/             │
│  ├── catalogs/main/        │  s3e-mirror     │  ├── catalogs/main/        │
│  │   (SlateDB)             │ ───────────────▶│  │   (SlateDB)             │
│  └── data/                 │  + rocklake     │  └── data/                 │
│      (Parquet files)       │    extension    │      (Parquet files)       │
└────────────────────────────┘                 └────────────────────────────┘
            ▲                                              ▲
            │ holds snapshot pin                           │ commits last
            │ (replication slot)                           │
            │                                              │
            └────────────  Coordinator   ──────────────────┘
                            │
                ┌───────────┴────────────┐
                │   Workers (data-file   │
                │   + SST copy jobs)     │
                └────────────────────────┘
```

The flow for one mirror tick:

1. The coordinator polls the source catalog for the latest committed `snapshot_id`. (Or, in event-driven mode, it is woken by an object-store notification on the catalog's `manifest/` prefix.)
2. If the new snapshot is greater than the last mirrored snapshot, the coordinator opens a read-only view of the source catalog at the new snapshot ID, using RockLake's existing `read_at_snapshot` API.
3. The coordinator scans the source catalog at that snapshot for:
   - All `DataFileRow` entries with `begin_snapshot ≤ N` and (`end_snapshot` is None or `> last_mirrored_snapshot`).
   - All `DeleteFileRow` entries in the same MVCC window.
4. The coordinator computes the set of Parquet paths in those rows that are not already on the target. Combined with the set of SlateDB SSTs the catalog's manifest needs (the existing s3e-mirror plan), this is the plan for this tick.
5. Workers stream Parquet bytes from source to target. Destination paths are deterministic (same path semantics as the source — either preserved as relative, or rewritten if absolute, per the path-translation policy).
6. When all data-file jobs are acked **and** all SlateDB SST jobs are acked, the coordinator commits the translated catalog manifest to the target SlateDB. The two-phase rule holds for both layers.
7. The coordinator advances the snapshot pin at the source (releases the previous one).

## Snapshot Pinning: The Replication Slot

To prevent `rocklake excise` on the source from deleting Parquet files the mirror still needs, the mirror holds a **snapshot pin** at the source. This pin is the RockLake equivalent of s3e-mirror's SlateDB checkpoint: it tells the source's garbage-collection / excise logic "do not delete anything referenced by any snapshot ≥ pinned_snapshot_id."

There are two ways to implement this pin:

- **Today (no RockLake changes):** the mirror writes a small marker row into RockLake's `ducklake_metadata` table (e.g., key `s3e_mirror.replication_slot.{slot_id}.min_snapshot = N`), and the operator agrees, by policy and by `excise --dry-run` review, to never excise past any active replication slot. This works but relies on discipline.
- **With a small RockLake change:** add a first-class `ducklake_replication_slot` table whose rows are read by `excise_plan()` to compute a hard floor that excise refuses to cross. This makes the pin enforced by RockLake itself, not by operator discipline. Proposed in [Part VII](#part-vii--rocklake-changes-that-would-help).

The pin advances every time the mirror commits a new target snapshot: pin moves from `N` to `N+k`, and `excise` is now free to remove anything that was retired before `N+k`.

## Path Translation

Three cases, in order of preference:

1. **Source uses relative paths (`path_is_relative = true`) and target uses the same data-prefix structure.** No rewriting needed. The catalog at the target is byte-identical to the catalog at the source for the `path` and `path_is_relative` fields, and the relative-path resolver at the target naturally points to the target's data prefix. This is the clean case and we recommend new RockLake warehouses be configured this way.
2. **Source uses absolute paths to a bucket we are mirroring.** The catalog mirror's "translate manifest" step is extended to rewrite each `DataFileRow.path` and `DeleteFileRow.path` from `s3://source-bucket/data/...` to `az://target-bucket/data/...`. A regex or a structured URL substitution suffices, because RockLake paths follow a predictable format. The rewrite is performed during catalog translation, before the target SlateDB manifest is committed, and is fully deterministic.
3. **Source mixes relative and absolute paths, or absolute paths point to multiple source buckets.** This is rare but possible (older warehouses, manual file registration, federated tables). The mirror handles each row individually using a configurable URL mapping table in the mirror config (`[data_remap]` in TOML). Rows whose paths do not match any mapping cause the mirror to refuse the snapshot and require operator intervention; we never silently leave dangling references.

In all three cases, the catalog rewrite is part of the *translation* step, not the *copy* step. The source catalog is read read-only; the target catalog is constructed in memory; the bytes are then written via the existing s3e-mirror commit path. There is no mutation of the source.

## Handling Excise at the Source

If the source operator runs `rocklake excise` while the mirror is active and the excise removes a file the mirror has not yet copied, the mirror detects this on the next attempt to copy that file (404 / object-not-found from the source). The behavior is:

- **If the snapshot pin was held**, this is a bug in RockLake's excise (it should have refused to delete files protected by the pin). The mirror raises a hard alarm and stops; the operator must investigate.
- **If no snapshot pin was held**, the operator made a deliberate decision to allow data loss. The mirror logs the missing file, marks the affected snapshot as un-mirrorable, and continues from a later snapshot. The target's most recent committed snapshot remains coherent and queryable.

If the source operator runs `rocklake excise` for a file the mirror *has already copied* and committed to the target, the mirror has two options based on policy:

- **Replicate the excise** (default): the mirror propagates the deletion to the target, calling the equivalent of `rocklake excise` against the target SlateDB and deleting the Parquet file. This keeps the target a faithful copy of the source.
- **Retain forever** (opt-in): the mirror records the excise in an audit log but does not delete on the target. Useful when the target is intended as a long-term archival mirror that retains data the source has compliance-erased. *(Note: this mode has its own compliance implications and must be enabled deliberately.)*

## CDC and Streaming

RockLake's CDC machinery today produces in-memory JSON values returned to clients; it does not write objects to the data store. So the mirror has no extra CDC objects to copy. If a future RockLake release writes CDC output to S3 / Kafka / a webhook (as planned for v0.23), that output is *not* part of the warehouse — it is a downstream pipeline — and the mirror does not handle it. Downstream consumers of CDC should subscribe to the target after promotion, not be replicated from source to target.

Streaming ingest in RockLake (`RockLakeSink::commit_batch()`) just registers Parquet files that the client has already written. The mirror sees those as ordinary `DataFileRow` entries appearing in new snapshots; nothing special is needed.

## Configuration

```toml
[source.catalog.main]                  # The SlateDB catalog store
provider = "s3"
bucket   = "prod-lake"
prefix   = "catalogs/main"
region   = "us-east-1"

[source.catalog.wal]                   # Optional separate WAL store
# (omitted if WAL shares catalog store)

[source.data]                          # The Parquet data store
provider = "s3"
bucket   = "prod-lake"
prefix   = "data"
region   = "us-east-1"

[target.catalog.main]
provider = "azure"
account  = "drcompany"
container = "dr-lake"
prefix   = "catalogs/main"

[target.data]
provider = "azure"
account  = "drcompany"
container = "dr-lake"
prefix   = "data"

[mirror]
mode = "continuous"                    # snapshot | continuous | failover | verify
snapshot_pin_ttl_secs = 3600
snapshot_poll_interval_ms = 5000
data_file_concurrency_per_worker = 16
preserve_excise = true                 # propagate source excise to target

[data_remap]                           # Optional, for absolute-path migrations
"s3://prod-lake/data/" = "az://dr-lake/data/"

[worker]
count = 32                             # data files dominate; expect higher count than pure SlateDB mirror
placement_hint = "source_region"

[observability]
metrics_endpoint = "0.0.0.0:9090"
```

## CLI

```text
s3e-mirror-rocklake init     --config rocklake-mirror.toml
s3e-mirror-rocklake plan     --config rocklake-mirror.toml      # dry-run, estimates bytes & cost
s3e-mirror-rocklake run      --config rocklake-mirror.toml
s3e-mirror-rocklake status   --config rocklake-mirror.toml      # source vs target snapshot_id, lag, queue depth
s3e-mirror-rocklake verify   --config rocklake-mirror.toml --level 1|2|3
s3e-mirror-rocklake promote  --config rocklake-mirror.toml      # cleanly promote target to primary
s3e-mirror-rocklake reconcile --config rocklake-mirror.toml     # one-off: catalog says X, scan target for missing
```

## Observability Additions

In addition to all the SlateDB-mirror metrics from [DESIGN.md](DESIGN.md), the RockLake mirror exposes:

- `s3e_mirror_rocklake_source_snapshot_id` (gauge)
- `s3e_mirror_rocklake_target_snapshot_id` (gauge)
- `s3e_mirror_rocklake_snapshot_lag` (gauge)
- `s3e_mirror_rocklake_data_files_copied_total` (counter)
- `s3e_mirror_rocklake_data_bytes_copied_total` (counter)
- `s3e_mirror_rocklake_data_files_in_flight` (gauge)
- `s3e_mirror_rocklake_excise_propagations_total` (counter)
- `s3e_mirror_rocklake_path_rewrites_total` (counter, labeled `rule`)
- `s3e_mirror_rocklake_orphan_files_at_target` (gauge, periodic scan)

## Verification Levels

- **Level 1 (cheap, default in CI):** for each `DataFileRow` and `DeleteFileRow` in the target's current snapshot, HEAD the referenced Parquet file at the target. Pass if all present.
- **Level 2 (medium):** Level 1 plus size match (HEAD response size = `DataFileRow.file_size_bytes`).
- **Level 3 (deep, opt-in):** Level 2 plus full-file digest re-computation against a stored digest map. Expensive; intended for periodic audits.

---

# Part VI — Failure Modes

| Failure                                                | Detection                                  | Effect on target                                 | Recovery |
| ------------------------------------------------------ | ------------------------------------------ | ------------------------------------------------ | -------- |
| Worker crash mid-Parquet copy                          | claim marker stale beyond TTL              | none (no manifest committed)                     | another worker retakes; deterministic dest name overwrites OK |
| Source `excise` removes file before mirror copies      | 404 from source GET                        | snapshot un-mirrorable; alarm                    | operator confirms intent; mirror restarts from later snapshot |
| Source path-rewrite rule missing for absolute path     | unmappable path during translation         | snapshot refused; no commit                      | operator adds rule to config; mirror retries |
| Catalog snapshot references Parquet not yet copied     | impossible by construction (two-phase rule) | n/a                                              | n/a |
| Two RockLake mirrors target same warehouse             | second mirror's catalog lease create fails | second mirror exits                              | operator chooses one |
| Target has orphan Parquet files (rare; from crashes)   | `verify --level 1` reports                 | no correctness issue (catalog never names them)  | `s3e-mirror-rocklake reconcile --gc-orphans` |
| Target catalog out of sync with target data (corrupted manually) | `verify --level 2` reports mismatch | reads at affected snapshot fail                  | re-run snapshot copy; in worst case re-bootstrap |
| Snapshot pin lost at source (RockLake bug or operator) | mirror detects missing file in pinned range | snapshot un-mirrorable                          | restart from fresh pin; raise alarm |
| Source mixes path styles unexpectedly                  | path-translation rule misses               | snapshot refused                                 | operator updates config |
| Excise propagation fails at target                     | target `excise` returns error              | source and target diverge; alarm                 | operator investigates; manual reconcile |

The most important row in that table is the second-to-last: the **two-phase rule is impossible to violate by construction**. The coordinator does not commit a target catalog manifest until every Parquet file it references is durably on the target with a verified digest, and the target SlateDB manifest is itself committed only after every catalog SST it references is present. Two nested two-phase rules, both enforced via `PutMode::Create`, both verified before commit.

---

# Part VII — RockLake Changes That Would Help

In the same spirit as [DESIGN.md § How SlateDB Could Help](DESIGN.md#part-xi--how-slatedb-could-help), this is a focused list of small RockLake changes that would make the mirror simpler, safer, or faster. None are blockers.

## Safety

### 1. First-class replication slots

A `ducklake_replication_slot` table (well-known well-keyed metadata) with rows `{slot_id, holder_id, min_snapshot_id, last_acked_snapshot_id, expires_at}`. The slot is consulted by `excise_plan()` as a hard floor: excise refuses to remove any file referenced by snapshots `≥ min(min_snapshot_id across slots)`. This converts the snapshot-pin contract from policy into mechanism.

### 2. Excise-propagation handshake

A documented contract for "this excise happened" events: an audit row in the catalog (already partially present today) plus a stable API for downstream consumers (mirrors, audit dashboards) to subscribe to excise events. Today the mirror has to scan for files-gone-missing; an explicit event makes the propagation immediate and unambiguous.

## Latency

### 3. Snapshot-commit subscription

RockLake's catalog writer already knows when a new snapshot is committed. Exposing a `CatalogStore::subscribe() -> watch::Receiver<SnapshotEvent>` channel (analogous to SlateDB's `Db::subscribe`) would let an in-process mirror react in microseconds rather than polling. An IPC variant for sidecar mirrors would extend this to cross-process consumers.

### 4. Snapshot-tip pointer object

A small, periodically-rewritten `catalogs/main/SNAPSHOT_TIP` object containing the current `snapshot_id` as text. Cheap to GET, no LIST required. Race-tolerant because the catalog itself is the source of truth.

## Throughput

### 5. Bulk file-reference export

`CatalogStore::list_files_at_snapshot(snapshot_id, since: Option<u64>) -> Stream<FileRef>` would return the (path, size, optional digest) tuples for every Parquet file visible at a snapshot, optionally filtered to files added since some earlier snapshot. This is what the mirror would otherwise have to compute by scanning `DataFileRow` and `DeleteFileRow` prefixes; bundling it in one call saves both code duplication and round-trips.

### 6. Per-file content digest in DataFileRow

Adding a `content_digest` field to `DataFileRow` and `DeleteFileRow` (xxh3-128 or blake3-256), populated by the writer, makes cross-provider integrity verification a metadata check instead of a bytes-re-read. This is also valuable to RockLake itself for detecting silent corruption on read.

## Configuration

### 7. Explicit separation of catalog and data stores in the CLI

`--catalog s3://catalog-bucket/path --data s3://data-bucket/path` (or two separate `--store` flags). The underlying `CatalogPath` already supports this; exposing it makes it possible to run a warehouse where catalog and data live on different providers from day one. Useful for hot-tier-cold-tier setups even without mirroring.

### 8. Always-relative-paths mode for new warehouses

A warehouse-level flag that forces `path_is_relative = true` on every new file registration. Eliminates the path-translation complexity for any warehouse created with the flag on. Recommended default for new deployments.

## Operational

### 9. `rocklake mirror-status` / `rocklake slot-status`

If replication slots become first-class (item 1), then `rocklake slot-status` shows operators what mirrors are active, where they are, and how far behind. Reduces the chance of an operator running `excise` without realizing a mirror is depending on the data.

### 10. NDJSON-export and ingest of replication state

The existing NDJSON export (`docs/operations/backup-restore.md`) is logical and full. A "delta export since snapshot N" would let a mirror do a logical bootstrap from a snapshot dump rather than a per-file byte copy, which is sometimes preferable for very large warehouses.

---

# Part VIII — Roadmap and Open Questions

## Phases

We propose shipping in the same phased style as s3e-mirror itself:

- **Phase 0 — Spec sign-off.** This document, plus targeted upstream issues against RockLake for the items in [Part VII](#part-vii--rocklake-changes-that-would-help). Land items 1, 7, and 8 in upstream as the highest-value changes.
- **Phase 1 — Read-only enumeration.** Implement the source catalog reader: scan `DataFileRow`/`DeleteFileRow` at a snapshot, build the file-set, dry-run the plan. No writes.
- **Phase 2 — Cold start to local target.** Copy a complete warehouse from one local path to another. End-to-end test: query the target with DuckDB + the ducklake extension, get identical results.
- **Phase 3 — Continuous tail.** Snapshot watermark advance. Snapshot-pin protocol. Two-phase commit across data and catalog layers.
- **Phase 4 — Cross-cloud.** Real S3 and Azure. Adaptive concurrency for the high-cardinality data layer. Path translation.
- **Phase 5 — Excise propagation, event-driven freshness, promotion CLI.**
- **Phase 6 — Advanced.** Logical (NDJSON-based) bootstrap. Cross-warehouse replication. Multi-target fan-out.

## Open Questions

1. **Bootstrap strategy for very large warehouses.** A 100 TB warehouse with hundreds of millions of Parquet files is feasible to mirror byte-by-byte, but takes time and egress. Should we offer a "seed from existing snapshot dump" workflow? Likely yes — see item 10 above.
2. **Multi-warehouse mirrors.** Is one mirror process per warehouse the right model, or should a single mirror coordinator manage a fleet of warehouses? Default proposal: one-per-warehouse for v1.
3. **Time-travel preservation across mirror.** If the source has snapshots back to N=1 and the mirror starts at N=50000, the target only has snapshots from N=50000 forward. Is this acceptable, or do we want a "historical backfill" mode? Default proposal: acceptable; backfill via reconcile if needed.
4. **Mirror-of-mirror.** Can a target serve as the source for another mirror? In principle yes, but the snapshot-pin chain needs explicit reasoning. Defer to a later phase.
5. **Cross-provider Parquet path schemes.** DuckDB / the ducklake extension reads `s3://`, `gs://`, `az://`, `file://`. Does the mirror promise that the target catalog's paths are openable by the client at the target? Yes — the path translation step ensures this. We need a test matrix.
6. **Compliance retention conflicts.** What if the source compliance policy says "delete after 30 days" (drives `excise`) but the target compliance policy says "retain for 7 years"? The `preserve_excise = false` mode supports this, but the compliance officer should sign off. Documented operational warning.

---

## Bottom Line

Mirroring a RockLake warehouse cleanly is a small extension to s3e-mirror, not a separate project. The data-file layer is operationally simpler than the catalog layer (immutable Parquet, content-addressed UUIDs, no compaction churn), and the catalog layer is just SlateDB, which s3e-mirror already handles. The two new design elements specific to RockLake are the **snapshot-pin replication slot** (so that `excise` cannot pull the rug) and the **path translation step** (so that absolute paths in old warehouses can be remapped at the target). Both fit naturally into the existing s3e-mirror coordinator and worker architecture.

The result is a single tool, a single config file, and a target warehouse that DuckDB can query exactly the same way it queries the source — same SQL, same time-travel, same schema — but living in a different bucket, a different region, or a different cloud entirely. The promotion path is the same as for any RockLake warehouse: point the `rocklake-pgwire` binary at the target. The disaster-recovery story for a RockLake data lake becomes "we have it on Azure too, ready to serve queries within minutes."
