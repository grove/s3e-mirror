# s5mirror — SlateDB Cross-Store Mirroring Tool

A massively parallel tool that mirrors a SlateDB database from one object
store to another (e.g. S3 → Azure Blob Storage), keeps the target
continuously up-to-date with the source, and scales horizontally across
multiple worker processes.

---

## 1. Background: how a SlateDB database is laid out

A SlateDB database is rooted at a single object-store prefix. Every piece of
durable state is a single object under that prefix:

| Path | Purpose | Mutability |
|---|---|---|
| `<root>/manifest/NNNNNNNNNNNNNNNNNNNN.manifest` | Sequenced manifest versions (latest id wins) | Immutable; new ids appended via `put_if_not_exists` |
| `<root>/compactions/NNNNNNNNNNNNNNNNNNNN.compactions` | Sequenced compactor state | Immutable; new ids appended |
| `<root>/gc/manifest.boundary`, `<root>/gc/compactions.boundary` | GC fencing watermark | Mutable but small |
| `<root>/wal/NNNNNNNNNNNNNNNNNNNN.sst` | Write-ahead-log SSTs, monotonic `u64` id | Immutable |
| `<root>/compacted/<ULID>.sst` | L0 + compacted sorted-run SSTs | Immutable |

Key invariants that make mirroring tractable:

- All data files (`*.sst`) are **write-once and content-addressable** by their
  filename. A given `wal/00000000000000000123.sst` or
  `compacted/01HF…ULID.sst` either does not exist or has the exact bytes that
  every reader expects forever.
- The `manifest` is the single source of truth for **which** files are live.
  GC only deletes files that the latest manifest (and any active checkpoint)
  no longer references, with a `min_age` grace period.
- Sequenced metadata uses `put_if_not_exists` CAS plus boundary files
  (RFC 0026) for fencing — perfect for an OCC writer on the target side.
- The Rust `object_store` crate already abstracts S3 / GCS / Azure / local /
  in-memory behind one trait, so the same copy code works for any pair.
- The `Checkpoint` primitive (RFC 0004) lets us "pin" a consistent view of the
  source so its GC cannot reclaim files we still need to copy.

These properties give us an embarrassingly parallel copy problem with a tiny
amount of serialised coordination at the manifest layer.

## 2. Goals & non-goals

### Goals
1. **Initial bulk copy** of an existing SlateDB database from source to
   target object store, byte-identical at the file level.
2. **Continuous tail** keeping the target within seconds of the source.
3. **Massively parallel** copy of independent files, both within one process
   (Tokio task pool) and **across many processes** (sharded workers).
4. **Crash-safe and resumable**: a killed worker, a killed coordinator, or a
   restarted whole fleet leaves the target in a valid state and resumes
   without re-copying completed objects.
5. **Cross-cloud**: S3 ↔ Azure ↔ GCS ↔ local, using only generic
   `object_store` operations.
6. **No source modifications** beyond holding a checkpoint and (optionally)
   listing — the tool is a reader on source and the writer on target.

### Non-goals (v1)
- Cycle-consistent two-way replication.
- Filtered / projected mirroring (we copy the whole keyspace; SlateDB already
  has `ProjectionConfig` we could plug in later).
- Replicating the compactor state file (`*.compactions`) — the target runs
  no compactor, so it does not need it.
- Decoding individual KV records — we operate exclusively at the
  manifest-and-SST file level.

## 3. High-level architecture

```
              ┌───────────────────────────────────────────┐
              │              Coordinator                  │
              │  (single elected process; fenced by the   │
              │   target manifest's writer_epoch)         │
              │                                           │
              │  ┌─────────────┐    ┌──────────────────┐  │
              │  │ Source poll │───▶│ Planner / diff   │  │
              │  └─────────────┘    └────────┬─────────┘  │
              │                              ▼            │
              │                     ┌──────────────────┐  │
              │                     │ Job queue (sharded│  │
              │                     │  in target store) │  │
              │                     └────────┬─────────┘  │
              │                              ▼            │
              │                     ┌──────────────────┐  │
              │                     │ Target manifest  │  │
              │                     │ writer (commits) │  │
              │                     └──────────────────┘  │
              └─────────────────┬─────────────────────────┘
                                │
        ┌──────────┬────────────┼────────────┬──────────┐
        ▼          ▼            ▼            ▼          ▼
    Worker 0   Worker 1     Worker 2    …          Worker N
   (claims     (claims      (claims                  (claims
    shard 0,   shard 1,     shard 2,                  shard N,
    GET/PUT)   GET/PUT)     GET/PUT)                  GET/PUT)
```

Components:

- **Coordinator** — single fenced process that owns the target manifest. It
  polls source state, computes deltas, publishes copy jobs, and commits new
  target manifests once jobs are complete.
- **Worker** — stateless processes (one or many machines) that claim copy
  jobs from a sharded queue and stream bytes source → target.
- **Job queue** — implemented on top of the target object store so that we
  add **zero external infrastructure** (no Kafka/Redis/Dynamo needed).

A single-process deployment is the same code with `N` workers as Tokio
tasks in-process and the coordinator running inline.

## 4. Crate layout

A Cargo workspace under `s5mirror/`:

```
Cargo.toml                    # workspace
crates/
  s5mirror-core/              # types, manifest read/translate, job model
  s5mirror-copy/              # parallel/multipart file copy primitives
  s5mirror-coord/             # planner, deltas, manifest commit, source poll
  s5mirror-worker/            # job claim, copy, ack
  s5mirror-cli/               # `s5mirror` binary: bulk, tail, worker, status
```

Key external deps:

- `slatedb` (path or pinned version) — re-use `ManifestStore`,
  `PathResolver`, `Manifest`, `SsTableId`, `FenceableManifest`,
  `Checkpoint`, error types. We **do not** re-implement the manifest codec.
- `object_store` — same version as `slatedb` uses.
- `tokio`, `futures`, `async-trait`, `bytes`.
- `clap` for the CLI; `tracing` + `tracing-subscriber` for logs;
  `metrics` + `metrics-exporter-prometheus` for metrics.
- `serde`, `serde_json` for job descriptors.
- `ulid`, `uuid`, `chrono`.

## 5. Source-side state we read

We use the existing SlateDB APIs as a library, not by reimplementing them.

- **List manifests** — `ManifestStore::list_manifests` (returns ids).
- **Latest manifest id** — already cached behind the boundary file, so this
  is one conditional GET in the common case.
- **Read a specific manifest** — `ManifestStore::read_manifest(id)`.
- **Decode `Manifest`** — gives us `ManifestCore` with:
  - `core.l0: VecDeque<SsTableView>` — L0 views.
  - `core.compacted: Vec<SortedRun>` — sorted runs.
  - segments (each its own `LsmTreeState`).
  - `next_wal_sst_id`, `last_compacted_wal_sst_id`, `last_l0_seq` etc.
  - `checkpoints`.
- **Resolve file paths** — `PathResolver` produces `wal/<id>.sst` and
  `compacted/<ulid>.sst` paths.

From a single manifest we can compute the **complete set of object keys**
that the source database currently considers live:

```
live_files(manifest) =
      { manifest_path(manifest.id) }
    ∪ { wal_path(id)        | id ∈ (last_compacted_wal_sst_id .. next_wal_sst_id) }
    ∪ { compacted_path(sst) | sst ∈ all L0 + all sorted-run SSTs in all segments }
```

## 6. Source pinning: checkpoints

Before any data is copied we create an **ephemeral checkpoint** on the source
via `FenceableManifest::write_checkpoint(CheckpointOptions{ lifetime: Some(T),
… })`. The checkpoint:

- pins a specific source `manifest_id`, so its referenced SSTs cannot be
  deleted by the source GC;
- has a finite lifetime that the coordinator **renews on a ticker** (every
  `lifetime / 3`) for as long as the mirror is running;
- is replaced periodically by a **fresher** checkpoint as we advance the
  target manifest — old checkpoints are dropped via
  `FenceableManifest::delete_checkpoint`.

The checkpoint id is persisted in the target's `mirror/state.json` so a
restarted coordinator can immediately reattach instead of starting from a new
checkpoint (which would force a full re-listing).

## 7. Target-side coordination & fencing

The target database is **owned by the mirror**: only the mirror writes
manifests there. We re-use SlateDB's existing primitives so the target stays
a valid SlateDB database that any reader/writer can open later.

1. **Open or initialise** the target `ManifestStore`. If empty,
   `StoredManifest::create_new_db` with `initialized = false` and an empty
   `ManifestCore`.
2. **Bump writer epoch** via `FenceableManifest::init_writer(...)`. This
   gives single-writer fencing for the coordinator across processes —
   another would-be coordinator that starts up bumps the epoch and the old
   one's next manifest commit fails with `Fenced` and it exits.
3. **All target manifest commits go through** `FenceableManifest::update`,
   which is `put_if_not_exists` against the next manifest id and is
   automatically guarded by the GC boundary file.

We also keep a `mirror/state.json` (transactional object using the existing
`SimpleTransactionalObject` machinery in `slatedb-txn-obj`) for our own
metadata: source path, last replicated source manifest id, source
checkpoint id, planner generation, current batch id.

## 8. Manifest translation (source → target)

A source manifest references files by `SsTableId` and resolves to physical
paths via `PathResolver(source_root)`. When we copy a file unchanged to
`<target_root>/wal/<same id>.sst` or
`<target_root>/compacted/<same ulid>.sst`, the **`SsTableId`s remain valid**
on the target — there is no manifest rewriting at the SST-reference level.

What we do change when emitting the target manifest:

- Strip `external_dbs` / clone-source entries — the mirror produces a
  self-contained DB. If the source itself is a clone, v1 supports it by
  recursively resolving and copying from the actual underlying paths.
- Reset `writer_epoch` / `compactor_epoch` to the values the target's
  `FenceableManifest` requires (it manages them).
- Drop source-side checkpoints by default (configurable: copy-through).
- Keep core LSM state, segment state, wal id watermarks, and seq numbers
  verbatim — they describe data that, by virtue of having been copied, now
  exists on the target with the same ids.

Target manifest ids are produced monotonically by
`FenceableTransactionalObject` and have no relationship to source ids.

## 9. The job model

A **Job** is the atomic unit of work for a worker:

```rust
#[derive(Serialize, Deserialize)]
enum Job {
    CopyObject {
        job_id: Uuid,
        kind: ObjectKind,           // Wal | Compacted | Manifest
        src_path: String,           // object-store path on source
        dst_path: String,           // object-store path on target
        expected_size: Option<u64>,
        // For multipart-split jobs:
        byte_range: Option<(u64, u64)>,
        upload_id: Option<String>,
        part_number: Option<u32>,
    },
    FinalizeMultipart {
        job_id: Uuid,
        dst_path: String,
        upload_id: String,
        parts: Vec<PartId>,
    },
}
```

`Batch` groups jobs that must all finish before the coordinator commits the
next target manifest:

```rust
struct Batch {
    batch_id: Ulid,
    source_manifest_id: u64,
    source_checkpoint_id: Uuid,
    jobs: Vec<Job>,
}
```

### Job queue layout (in the target store)

```
<target>/mirror/
  state.json                              (transactional)
  batches/<batch_id>/
    plan.json                             (immutable; written first)
    shards/<shard_idx>/
      pending/<job_id>.json               (created by planner)
      claimed/<job_id>.<worker_id>.json   (renamed-on-claim marker)
      done/<job_id>.json                  (created by worker after copy)
    COMPLETE                              (created when all done counts match)
```

Claiming a job is a `put_if_not_exists` of the `claimed/` marker (a
`copy` from `pending/`), then a `delete` of the `pending/` object. If two
workers race, only one wins the `put_if_not_exists` — the same primitive
SlateDB uses for manifest CAS.

Shard count `S` is a deployment knob (default: 64). A worker is configured
with an integer `worker_id`; it owns shards where `shard_idx % world_size ==
worker_id`. This gives **stateless** workers — add or remove worker
processes at any time and the rebalance is implicit on the next batch.

## 10. Parallel copy primitive

Every copy job streams source → target. The implementation lives in
`s5mirror-copy`:

```rust
async fn copy_object(
    src: &dyn ObjectStore, src_path: &Path,
    dst: &dyn ObjectStore, dst_path: &Path,
    opts: CopyOpts,
) -> Result<CopyOutcome>
```

Algorithm:

1. **HEAD** the destination; if `size` and `e_tag`/`content-length` match
   the planner's `expected_size`, return `AlreadyPresent` (idempotent
   resume).
2. **HEAD** the source for size + ETag.
3. If `size <= multipart_threshold` (default 16 MiB): one `get` → one
   `put_if_not_exists` to `dst_path`. The `put_if_not_exists` is the
   correctness guarantee — a concurrent worker that lost the race observes
   `AlreadyExists` and treats the job as done.
4. Otherwise:
   - `dst.put_multipart(dst_path)` → `MultipartUpload`.
   - Compute `n = ceil(size / part_size)` parts (default `part_size = 64
     MiB`, configurable per backend).
   - Spawn up to `intra_object_parallelism` concurrent tasks, each doing a
     ranged GET on the source and `put_part` on the destination.
   - `complete()` the upload.
   - For **very large** SSTs we can also farm out individual parts as
     separate `Job::CopyObject{byte_range, upload_id, part_number}` items
     so they execute on different machines, then a single
     `Job::FinalizeMultipart` commits the upload.
5. **Verify**: re-HEAD destination, compare `size` (and `e_tag` when the
   pair supports comparable hashing — same-cloud copies typically do).
6. For `*.sst` files, optionally validate by reading the SST footer using
   `slatedb`'s existing `SsTableFormat`/`SstReader` paths — cheap O(1)
   bytes near the tail.

Concurrency knobs:
- `worker.inter_job_parallelism` — concurrent jobs per worker (default 32).
- `worker.intra_object_parallelism` — concurrent parts per multipart upload
  (default 8).
- Per-backend overrides (S3 vs Azure have different sweet spots).

## 11. Bulk (initial) copy

```
1. open_or_init_target()                       # writer_epoch bump = fence
2. create_or_renew_source_checkpoint()
3. m_src = source_manifest_at(checkpoint)
4. plan = enumerate_live_files(m_src)
        - wal SSTs in [last_compacted_wal_sst+1, next_wal_sst)
        - all L0 + sorted-run compacted SSTs
        - the chosen source manifest itself (so we know what we copied)
5. dedupe against any objects already on target (HEAD-based filter
   in parallel; expected to short-circuit on a fresh target)
6. write_batch_to_target_store(batch_id, plan)
7. wait_for_batch_complete(batch_id)           # poll done/ counts
8. translate_manifest(m_src) → m_dst
9. fenceable_target.update(m_dst)              # initial committed manifest
10. mark mirror/state.json with last_source_manifest_id
```

Within one batch all jobs are independent, so steps 6–7 saturate the
worker pool. Bulk-copying a multi-TB database is purely network-bound.

## 12. Continuous tail

A small loop on the coordinator:

```rust
loop {
    sleep(poll_interval).await;          // e.g. 1–5s
    refresh_source_checkpoint_if_needed().await?;

    let latest_src_id = source_manifest_store.latest_id().await?;
    if latest_src_id == state.last_source_manifest_id { continue; }

    let m_src_new = source_manifest_store.read_manifest(latest_src_id).await?;
    let m_src_old = source_manifest_store.read_manifest(state.last_source_manifest_id).await?;

    let delta = file_set(m_src_new).difference(&file_set(m_src_old));
    if delta.is_empty() {
        commit_target_manifest(m_src_new).await?;     // metadata-only update
        state.last_source_manifest_id = latest_src_id;
        continue;
    }

    let batch = plan_batch(latest_src_id, delta);
    publish_batch(batch).await?;                       // workers start copying
    wait_for_batch_complete(batch.id).await?;
    commit_target_manifest(m_src_new).await?;
    update_state(latest_src_id).await?;
}
```

Properties:

- **Adds only**: a manifest delta during normal operation is "a few new WAL
  SSTs and (occasionally) some compacted SSTs". Typical batches are tiny
  and finish in well under one source flush interval.
- **Compactions are handled implicitly**: when the source compactor
  produces a new compacted SST and rewrites the manifest to drop covered
  L0s, the delta is "new compacted SST" — we copy it; the next target
  manifest naturally drops the covered L0 refs; the target GC reclaims them
  after `min_age`.
- **Source GC safety**: our rolling checkpoint keeps the previous
  `m_src_old` alive long enough to diff against. If, despite that, the
  diff manifest is missing (e.g. operator deleted the checkpoint), we fall
  back to a **full reconcile** (re-enumerate `m_src_new`'s live set and
  copy anything missing on target).

## 13. Failure model & resumption

| Failure | Effect | Recovery |
|---|---|---|
| Worker crashes mid-copy | Partial multipart upload abandoned; pending/claimed job orphaned | TTL-based reclaim: planner moves `claimed/` markers older than `claim_timeout` back to `pending/`. Multipart uploads older than `multipart_ttl` are aborted via `object_store.abort_multipart`. |
| Coordinator crashes after publishing batch, before commit | Workers finish jobs; new coordinator reads `mirror/state.json` and `batches/<latest>/`, sees `COMPLETE`, performs the deferred `commit_target_manifest` | Idempotent commit. |
| Coordinator crashes mid-commit | `put_if_not_exists` either succeeded (visible to new coordinator) or did not (new coordinator retries) | `FenceableManifest::maybe_apply_update` handles CAS conflicts. |
| Two coordinators racing | New one bumped `writer_epoch`; old one's next commit fails `Fenced` and the process exits | Existing SlateDB invariant. |
| Network blip on a copy | `put_if_not_exists` is idempotent; resumed copy is a no-op | Built into the copy primitive. |
| Source checkpoint expired | Diff may fail with `ManifestMissing`/`CheckpointMissing` | Full reconcile fallback. |

## 14. Scaling out

- **Vertical**: each worker is a Tokio runtime with bounded concurrency. A
  single fat node can drive tens of gigabits with a few hundred concurrent
  copies.
- **Horizontal**: launch `N` worker processes anywhere with read access to
  the source and write access to the target. Each is given a distinct
  `worker_id` and the global `world_size`. Adding workers requires no
  coordinator change — they pick up shards on the next batch. Removing them
  is handled by the claim-timeout reclaim path.
- **Geographic**: run workers close to either source or target depending on
  egress pricing; the same binary works either way because both sides are
  generic `object_store` URIs.
- **Per-object scale**: huge SSTs are split into byte-range part jobs that
  flow through the same shard queue, so a single 100 GiB SST does not
  become a single worker's bottleneck.

## 15. Observability

- **Metrics** (Prometheus):
  - `s5mirror_bytes_copied_total{kind}`
  - `s5mirror_objects_copied_total{kind}`
  - `s5mirror_copy_seconds{kind}` (histogram)
  - `s5mirror_lag_seconds` (target manifest commit time vs. source commit
    time on the latest replicated source manifest)
  - `s5mirror_lag_manifests` (count)
  - `s5mirror_jobs_inflight{worker}` / `pending` / `claimed_stale`
- **Logs** (`tracing`): structured spans per batch / job, including
  `batch_id`, `job_id`, `kind`, source/target paths, byte counts.
- **CLI** subcommands:
  - `s5mirror status` — current lag, batch state, checkpoint info.
  - `s5mirror verify` — sample N random keys via SlateDB read API on both
    sides and compare (separate, slower confidence check).

## 16. CLI surface

```
s5mirror init      --source URL --target URL [--checkpoint-ttl 1h]
s5mirror bulk      --source URL --target URL --workers N [--world-size W --worker-id I]
s5mirror tail      --source URL --target URL --poll 2s
s5mirror worker    --target URL --world-size W --worker-id I
s5mirror status    --target URL
s5mirror verify    --source URL --target URL [--sample 10000]
s5mirror gc        --target URL                # delete orphaned mirror/ artefacts
```

In single-process mode `bulk` and `tail` start workers internally; in
multi-process mode they only run the coordinator and you launch
`s5mirror worker` separately.

## 17. Configuration

A single `s5mirror.toml` (or env vars) for both coordinator and workers:

```toml
[source]
url = "s3://my-bucket/prod-db"
region = "us-east-1"

[target]
url = "az://my-container/prod-db-mirror"

[mirror]
checkpoint_ttl   = "1h"
poll_interval    = "2s"
shard_count      = 64
state_path       = "mirror/state.json"

[worker]
inter_job_parallelism   = 32
intra_object_parallelism = 8
multipart_threshold     = "16MiB"
part_size               = "64MiB"
claim_timeout           = "10m"
multipart_ttl           = "24h"

[backend.s3]
# object_store-specific knobs
[backend.azure]
```

## 18. Testing strategy

1. **Unit tests** for the manifest translator (round-trip translate;
   prove no clone/external_db references leak through; epochs reset).
2. **Integration tests** with `InMemory` object stores on both sides:
   - bulk copy then assert SlateDB reads return identical values for the
     same key set;
   - run a writer on the source under load and tail to convergence,
     comparing scan outputs at stable points;
   - kill-and-resume scenarios using the existing SlateDB `fail_parallel`
     fault-injection framework.
3. **Property tests** for the copy primitive: random sizes incl. exact
   part boundaries; injected NotFound / AlreadyExists / partial writes.
4. **Cross-cloud smoke tests** in CI against MinIO + Azurite.
5. **DST harness reuse**: `slatedb-dst` is a deterministic simulation
   harness — wire the mirror in to validate convergence under chaos.

## 19. Phased rollout

1. **Phase 1 — single-process bulk copy.** Coordinator + in-process worker
   pool. No tail, no multi-process. Validates manifest translation and the
   copy primitive end-to-end.
2. **Phase 2 — tail loop.** Add the polling delta loop, rolling
   checkpoint, and `mirror/state.json` resume.
3. **Phase 3 — multi-process workers.** Move the job queue onto the
   target object store and add the claim/done/timeout machinery.
4. **Phase 4 — per-object splitting.** Byte-range part jobs for very
   large SSTs.
5. **Phase 5 — observability + ops.** Metrics, `status` / `verify` /
   `gc` CLI, runbooks.
6. **Phase 6 (optional) — projection.** Plug SlateDB's `ProjectionConfig`
   through so users can mirror only a prefix / key range.

## 20. Open questions

- Should we also replicate `*.compactions` files so that a compactor
  could be started on the target without recovery cost? (v1: no.)
- For S3→S3 same-region copies we should detect this and use `CopyObject`
  (server-side copy) instead of GET+PUT. The `object_store` crate exposes
  `copy_if_not_exists`; gate this on identical backend + region.
- Do we want to expose mirror state via a SlateDB-readable transactional
  object (e.g. as a special manifest annotation) so that downstream tools
  can introspect lag without parsing our JSON? Probably yes, post-v1.

---

### Appendix A — minimal trait sketches

```rust
// s5mirror-core
pub trait ManifestSource {
    async fn latest_id(&self) -> Result<u64>;
    async fn read(&self, id: u64) -> Result<Manifest>;
    async fn live_files(&self, m: &Manifest) -> Result<BTreeSet<FileRef>>;
}

pub trait JobQueue {
    async fn publish(&self, batch: Batch) -> Result<()>;
    async fn claim(&self, shard: u32, worker_id: u32) -> Result<Option<ClaimedJob>>;
    async fn ack(&self, job: ClaimedJob) -> Result<()>;
    async fn reclaim_stale(&self, older_than: Duration) -> Result<usize>;
    async fn batch_complete(&self, batch_id: Ulid) -> Result<bool>;
}

pub trait FileCopier {
    async fn copy(&self, job: &CopyObjectJob) -> Result<CopyOutcome>;
}
```

These trait boundaries let us:
- swap the `JobQueue` for a Redis/SQS implementation later without
  touching workers or coordinator logic;
- mock everything in unit tests;
- inject the SlateDB `InstrumentedObjectStore` for metrics on every
  request, identical to how SlateDB itself instruments object IO.
