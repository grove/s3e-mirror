# s5mirror - SlateDB Cross-Store Mirroring

This document describes a fault-tolerant, massively parallel Rust system for
mirroring a SlateDB database from one object-store location to another, such as
S3 to Azure Blob Storage. It is based on a deeper read of the current SlateDB
source tree and RFCs in `../slatedb`, especially manifest storage, checkpoints,
separate WAL stores, WAL reading, garbage collection, clones, and distributed
compaction.

The important correction from the first draft is that a production mirror must
be designed around SlateDB's real API boundaries. SlateDB exposes useful
read-side admin APIs today, but most write-side manifest internals are still
`pub(crate)`. The safest path is therefore either:

1. Add a small public replication/import API to SlateDB and keep `s5mirror` as a
   standalone crate that uses it.
2. Start `s5mirror` as a crate inside the SlateDB workspace, or temporarily
   vendor a schema-level compatibility layer, until those APIs exist.

The design below treats option 1 as the target architecture and calls out the
fallbacks explicitly.

## Executive Summary

SlateDB is unusually mirror-friendly because almost all durable database data is
stored in immutable object-store files:

- WAL SSTs under `wal/`, identified by monotonically increasing `u64` ids.
- L0 and sorted-run SSTs under `compacted/`, identified by ULIDs.
- Sequenced metadata under `manifest/` and `compactions/`, written with
  create-if-absent semantics.
- Boundary files under `gc/`, used to prevent stale sequenced metadata writes
  after GC deletes old metadata files.

The mirror should use a two-phase rule:

1. Copy every object that a chosen source snapshot needs.
2. Only then publish a target manifest that references those target objects.

That manifest write is the visibility toggle. Before it, copied files are just
unreferenced bytes. After it, the target database is a valid SlateDB snapshot.

For continuous mirroring, the coordinator repeats this loop: create or refresh a
source checkpoint, enumerate the checkpoint's file set, enqueue missing objects,
wait for workers to copy and verify them, then commit a translated target
manifest. Workers are stateless and scale horizontally by claiming immutable copy
jobs from a sharded queue stored in the target object store.

## Current SlateDB Facts That Shape the Design

### Object Layout

For a database rooted at `<root>`, the main object store contains:

| Path | Meaning | Notes |
|---|---|---|
| `<root>/manifest/00000000000000000001.manifest` | Manifest version 1 | Sequenced metadata; id comes from the filename. |
| `<root>/manifest/...` | Later manifests | Latest id is current. |
| `<root>/compactions/00000000000000000001.compactions` | Compactor state | Also sequenced metadata. |
| `<root>/gc/manifest.boundary` | Manifest GC boundary | ASCII `u64`. |
| `<root>/gc/compactions.boundary` | Compactions GC boundary | ASCII `u64`. |
| `<root>/compacted/<ULID>.sst` | L0 and sorted-run SSTs | Immutable compacted-data files. |

The WAL path is logically `<root>/wal/<20-digit-u64>.sst`, but it may live in a
separate WAL object store configured with `DbBuilder::with_wal_object_store` or
`AdminBuilder::with_wal_object_store`. The path structure is the same in either
store.

WAL objects can be zero bytes. SlateDB uses zero-byte WAL files as writer-fence
markers. A mirror must preserve WAL id contiguity, so it must copy zero-byte WAL
fence objects when they fall in the live WAL range.

### Manifest State

`VersionedManifest` is public and exposes enough read-only access to enumerate
most file references:

- `id()`
- `l0()` and `compacted()` for the root tree
- `segments()` for RFC-0024 segmented databases
- `external_dbs()` for clone/external SST references
- `next_wal_sst_id()`
- `replay_after_wal_id()`
- `checkpoints()`
- `wal_object_store_uri()` (currently useful only for older V1 manifests; V2
  decodes this as `None`)

The manifest's live WAL rule is the same one SlateDB uses for GC:

```text
wal_id is live iff replay_after_wal_id < wal_id < next_wal_sst_id
```

The first draft used a vague `last_compacted_wal_sst_id`; the actual field is
`replay_after_wal_id`.

### Public APIs Available Today

Read-side APIs that are useful now:

- `Admin::read_manifest(Some(id) | None)`
- `Admin::list_manifests(range)`
- `Admin::read_compactions(Some(id) | None)`
- `Admin::list_compactions(range)`
- `Admin::read_compactor_state_view()`
- `Admin::list_checkpoints(name_filter)`
- `Admin::create_detached_checkpoint(options)`
- `Admin::refresh_checkpoint(id, lifetime)`
- `Admin::delete_checkpoint(id)`
- `WalReader::new(path, wal_object_store)` with `list(range)` and `get(id)`

Important APIs that are not public enough today:

- `ManifestStore`, `StoredManifest`, `FenceableManifest`
- `PathResolver`
- `Manifest` and `ManifestCore` mutation/encoding
- `FlatBufferManifestCodec`
- `TableStore` compacted-SST listing and metadata helpers
- `ObjectStores` and `ObjectStoreType`

This means a standalone `s5mirror` cannot safely materialize translated target
manifests using only the current public SlateDB crate. It can read manifests and
copy object bytes, but it needs new public SlateDB APIs, or a temporary
schema-level writer, to commit target manifests.

### Checkpoints and GC

Checkpoints pin manifest ids. SlateDB GC preserves manifests and SSTs referenced
by active checkpoints. A checkpoint can expire; `Admin::refresh_checkpoint` can
extend it, and `Admin::delete_checkpoint` removes it.

`Admin::create_detached_checkpoint` creates a checkpoint from the latest
persisted manifest without talking to an in-process writer. `Db::create_checkpoint`
has stronger source-side semantics when called on the running writer:

- `CheckpointScope::Durable` includes writes already durable at call time.
- `CheckpointScope::All` attempts to flush current state before checkpointing.

For the fastest possible tail, the mirror should support an optional source
sidecar or integration path that can call `Db::create_checkpoint` on the live
writer. Without that, an external-only mirror follows the persisted manifest and
may lag behind freshly written WAL objects until the source writer persists a
manifest update.

### Clones and External DBs

SlateDB clones can contain `external_dbs`: compacted SST ids that physically
live under another database root. A self-contained mirror must resolve each
`external_db` entry, copy those SSTs into the target's own `compacted/` prefix,
and emit a target manifest with those references made local.

The current public `CloneBuilder` is not a cross-object-store clone API. It
clones within one supplied object store. Cross-cloud mirroring needs an export /
import style API.

## Critique of the Previous Draft

The first plan had the right broad shape, but it was too optimistic in several
places:

1. It assumed private APIs were reusable from a standalone crate.
   `ManifestStore`, `PathResolver`, `FenceableManifest`, and the manifest codec
   are not public.
2. It treated target manifest writing as solved. Today there is no public API to
   import a translated manifest into a different object store while preserving
   SlateDB's sequenced metadata and boundary semantics.
3. It under-specified WAL freshness. An offline `Admin` checkpoint pins the
   latest persisted manifest, not necessarily the live writer's newest durable
   WAL frontier. Fast tailing needs WAL prefetch and, ideally, a source-side
   checkpoint hook.
4. It did not account for separate WAL object stores. Source and target config
   must include both main and WAL stores.
5. It proposed cross-process multipart uploads using a generic upload id. The
   `object_store` trait exposes multipart upload as an object, not as a portable
   serializable upload id. Cross-worker part uploads are backend-specific future
   work, not a generic v1 feature.
6. It described job claiming as a rename-like pending to claimed transition.
   Object stores do not provide atomic rename. The queue must use immutable plan
   files plus create-if-absent claim markers.
7. It did not mention target GC. Target GC can delete copied-but-not-yet-
   referenced files during a large batch. It must be disabled or configured with
   a min age larger than the worst-case batch duration.
8. It treated source clone manifests as a minor rewrite. In reality, external
   SST references require explicit source-path resolution, collision detection,
   and a choice between self-contained mirroring and preserving external refs.
9. It implied compactions files should probably be copied. For a readable target
   they are not required; target compactor resumption is a separate feature.
10. It over-relied on ETags. ETags are not portable checksums across providers
    and are often multipart-specific. Verification needs size plus optional
    checksums, and SlateDB would be better if manifests carried object digests.

The revised design fixes these points.

## Guarantees

### Safety Guarantees

- The target manifest is never advanced until every object it references has
  been copied and verified in the target store.
- A crash can leave extra unreferenced objects in the target, but not a manifest
  that references missing objects.
- Re-running a batch is idempotent.
- A stale worker may duplicate work, but it cannot make a committed target
  snapshot invalid.
- The source checkpoint is renewed before expiry and is not deleted until the
  target manifest and mirror state have advanced past it.

### Freshness Guarantees

There are two tail modes:

1. Manifest-following mode: the target catches up to the latest persisted source
   manifest. This is fully external and works with today's public read APIs plus
   a manifest import API.
2. Writer-integrated mode: a source helper asks the live writer to create
   durable checkpoints periodically. This can expose freshly durable WAL state
   faster than waiting for natural manifest persistence.

Both modes can prefetch WAL files ahead of the visible manifest to reduce the
copy backlog. Prefetched WAL objects are not advertised as a target snapshot
until a source checkpoint or manifest includes their frontier.

## Architecture

```text
                 source main store         source WAL store
                        |                        |
                        v                        v
                  +-----------------------------------+
                  |           Coordinator             |
                  |                                   |
                  |  checkpoint lease manager         |
                  |  manifest/compactions reader      |
                  |  file-set planner                 |
                  |  batch publisher                  |
                  |  target manifest importer         |
                  +-----------------+-----------------+
                                    |
                                    v
                       target-store mirror queue
                                    |
                  +-----------------+-----------------+
                  |                 |                 |
                  v                 v                 v
               Worker 1          Worker 2          Worker N
                  |                 |                 |
                  +--------- copy and verify ---------+
                                    |
                                    v
                 target main store         target WAL store
```

### Coordinator

The coordinator owns batch planning and target manifest commits. There should be
exactly one active coordinator per target database.

Coordinator responsibilities:

- Maintain the source checkpoint lease.
- Read source manifests and compute target file sets.
- Publish immutable batch plans.
- Reclaim stale job claims.
- Wait until all jobs in a batch are done.
- Translate/import a target manifest.
- Advance mirror state transactionally.
- Emit lag and health metrics.

Leader coordination should not depend on target SlateDB writer epochs until the
target manifest import API exists. Use a dedicated `mirror/lease` transactional
object in the target store:

```rust
struct MirrorLease {
    holder_id: String,
    fencing_token: u64,
    expires_at_ms: u64,
}
```

The lease is updated with compare-and-swap. Every batch and manifest import
includes the current fencing token. If the token changes, the old coordinator
stops before committing anything further.

Once SlateDB exposes a replication import API, that API should also fence target
manifest writers using the standard sequenced metadata/boundary protocol.

### Workers

Workers are stateless copy executors. Any worker can process any shard. Shards
exist to bound list sizes, not to impose fixed ownership.

Worker responsibilities:

- Poll one or more shard prefixes using jitter.
- Create a claim marker with create-if-absent.
- Copy the object, possibly using in-process multipart concurrency.
- Verify the target object.
- Write a done marker.
- Heartbeat long jobs.

Workers do not write target manifests and do not mutate mirror state except job
claim/done/heartbeat markers.

### Target-Store Job Queue

The queue is stored under the target main store, outside the SlateDB database
layout:

```text
<target>/mirror/
  lease/00000000000000000001.lease
  state/00000000000000000001.state
  batches/<batch_id>/
    plan.json
    shards/<shard>/jobs/<job_id>.json
    claims/<job_id>/<attempt_id>.json
    heartbeats/<job_id>/<attempt_id>.json
    done/<job_id>.json
    failed/<job_id>/<attempt_id>.json
    COMPLETE
  tmp/<batch_id>/<job_id>/...
```

There is no rename. A job remains in `jobs/` forever as part of the immutable
plan. Claiming is:

1. Worker reads a job file.
2. Worker checks whether `done/<job_id>.json` exists.
3. Worker creates `claims/<job_id>/<attempt_id>.json` with `PutMode::Create`.
4. If another live claim exists, the worker skips the job.
5. If the claim is stale, the coordinator can mark it abandoned and allow a new
   attempt.

Duplicate attempts are harmless because the source object is immutable and the
job describes the exact source path plus expected metadata.

## Crate Layout

```text
Cargo.toml
crates/
  s5mirror-core/        data model, file refs, state machine, errors
  s5mirror-slate/       SlateDB adapter: Admin reads, format/import APIs
  s5mirror-copy/        source-to-target object copy and verification
  s5mirror-queue/       object-store queue and lease implementation
  s5mirror-coord/       checkpoint lease, planner, commit loop
  s5mirror-worker/      worker runtime
  s5mirror-cli/         binary
```

The `s5mirror-slate` adapter should support two implementations:

- `slatedb-public-api`: the long-term path, using new public APIs proposed in
  this document.
- `schema-compat`: a temporary path that reads/writes FlatBuffer schemas
  directly and is tested against the exact SlateDB version in use.

The schema-compat path is acceptable for a prototype, but production should
prefer upstream SlateDB APIs so mirror correctness evolves with the database.

## Data Model

Every planned copy is represented as a fully resolved file reference:

```rust
enum StoreRole {
    Main,
    Wal,
}

enum FileKind {
    Manifest,
    WalSst,
    CompactedSst,
}

struct FileRef {
    kind: FileKind,
    source_store: StoreRole,
    target_store: StoreRole,
    source_path: object_store::path::Path,
    target_path: object_store::path::Path,
    source_manifest_id: u64,
    logical_id: String,
    expected_size: Option<u64>,
    expected_version: Option<String>,
    expected_digest: Option<String>,
}
```

`expected_version` is advisory. It can be an ETag or provider object version.
It must not be treated as a cross-provider digest. `expected_digest` should be
used when SlateDB or the source provider supplies a real checksum.

## Enumerating the Source Snapshot

Given a source `VersionedManifest`, enumerate:

1. The manifest object itself:

   ```text
   main:<source_root>/manifest/<manifest_id:020>.manifest
   ```

2. Every live WAL object, including zero-byte fence objects:

   ```text
   for wal_id in (replay_after_wal_id + 1)..next_wal_sst_id
       wal:<source_root>/wal/<wal_id:020>.sst
   ```

3. Every compacted SST referenced by:

   - root `l0()`
   - root `compacted()` sorted runs
   - each segment's `l0()`
   - each segment's `compacted()` sorted runs

4. For source clones, resolve compacted SSTs through `external_dbs()`:

   - If an SST id appears in an external DB entry, use that external DB path as
     the source root for that compacted object.
   - Copy it into the target's own `compacted/<ULID>.sst` when building a
     self-contained mirror.
   - Detect duplicate ids mapping to different source roots. If sizes or
     checksums differ, fail the batch.

The target paths are always local to the target database root unless the user
explicitly selects a future "preserve external references" mode.

## Source Checkpoint Protocol

The mirror maintains two source checkpoints:

- `stable_checkpoint`: the source manifest currently committed on the target.
- `candidate_checkpoint`: the source manifest being copied next.

Algorithm:

1. If this is the first run, create `stable_checkpoint` and copy it as the
   initial snapshot.
2. On each poll, create or refresh `candidate_checkpoint` with a name such as
   `s5mirror:<target-id>:candidate`.
3. Read the candidate manifest id from the checkpoint result.
4. Plan and copy all files required by the candidate manifest.
5. Import the translated candidate manifest into the target.
6. Persist mirror state with the new source manifest id and checkpoint id.
7. Delete the previous stable checkpoint only after step 6 succeeds.
8. Rename the candidate role to stable in mirror state.

If a coordinator crashes, the new coordinator reads mirror state, lists
checkpoints by name, and conservatively refreshes any mirror-owned checkpoints
before resuming. It is safe to retain extra checkpoints; the cleanup command can
delete stale mirror-owned checkpoints after verifying the target no longer needs
them.

Checkpoint TTL should be at least:

```text
max_expected_batch_duration + coordinator_failover_time + safety_margin
```

The coordinator refreshes at `ttl / 3`, with jitter.

## Bulk Copy Algorithm

```text
1. Acquire target mirror lease.
2. Ensure target DB is empty or already owned by this mirror.
3. Create source stable checkpoint.
4. Read source manifest at the checkpoint's manifest_id.
5. Enumerate the snapshot file set.
6. HEAD source objects and enrich FileRefs with size/version metadata.
7. HEAD target objects and remove already-verified files.
8. Publish a batch plan under mirror/batches/<batch_id>/.
9. Workers copy and verify every file.
10. Coordinator verifies done count and spot-checks target HEADs.
11. Translate/import target manifest.
12. Persist mirror state.
13. Mark batch COMPLETE.
```

The target manifest is committed only after all referenced files exist in the
target.

## Continuous Tail Algorithm

```text
loop every poll_interval with jitter:
    acquire_or_renew_target_lease()
    refresh_stable_checkpoint_if_needed()

    candidate = create_or_refresh_candidate_checkpoint()
    if candidate.manifest_id == state.last_source_manifest_id:
        continue

    m_candidate = read_manifest(candidate.manifest_id)
    candidate_files = enumerate_file_set(m_candidate)

    old_files = state.last_committed_file_set
    delta = candidate_files - old_files

    publish_batch(delta)
    wait_for_workers(batch)
    import_translated_manifest(m_candidate)
    update_state(candidate, candidate_files)
    delete_old_stable_checkpoint()
```

Store `last_committed_file_set` or at least a compact digest of it in mirror
state. This avoids needing to read an old source manifest after a long outage.
If the stored file set is missing or incompatible, perform a full reconcile:
enumerate the candidate file set and copy anything missing from the target.

## WAL Prefetch Mode

To keep lag low, workers can copy WAL SSTs before the source manifest catches
up:

1. Track the highest copied source WAL id for each source WAL store.
2. Use `WalReader::get(last + 1).metadata()` or bounded `list()` calls to find
   new contiguous WAL objects.
3. Copy WAL objects to the target WAL store immediately.
4. Do not advance the visible target manifest based only on prefetched WALs.
5. When a later source checkpoint includes those WAL ids in its manifest range,
   the manifest batch will find the objects already present and commit quickly.

This is a useful RPO/RTO optimization. It does not replace source manifest or
checkpoint semantics.

## Target Manifest Translation

Target manifest import is the only part that requires stronger SlateDB support.

Desired API shape:

```rust
pub struct ReplicationSnapshot {
    pub source_manifest_id: u64,
    pub manifest: VersionedManifest,
    pub file_refs: Vec<ReplicationFileRef>,
}

pub struct ImportManifestOptions {
    pub make_self_contained: bool,
    pub drop_source_checkpoints: bool,
    pub reset_epochs: bool,
    pub expected_previous_source_manifest_id: Option<u64>,
}

impl Admin {
    pub async fn import_replication_snapshot(
        &self,
        snapshot: ReplicationSnapshot,
        options: ImportManifestOptions,
    ) -> Result<VersionedManifest, Error>;
}
```

Translation rules:

- Keep the LSM tree state, segment state, sequence tracker, `last_l0_seq`,
  `recent_snapshot_min_seq`, `last_l0_clock_tick`, `replay_after_wal_id`, and
  `next_wal_sst_id` consistent with the source snapshot.
- Rewrite `manifest_id` to the target sequenced id.
- Clear source checkpoints by default.
- Clear `external_dbs` in self-contained mode after all external compacted SSTs
  have been copied locally.
- Reset writer and compactor epochs under SlateDB's import/fencing protocol.
- Preserve `segment_extractor_name`. Target readers/writers must be opened with
  the same extractor implementation if they later write to the mirrored DB.
- Preserve format version semantics; V2 must remain V2 when segments exist.

The import API must write through SlateDB's sequenced metadata protocol, using
create-if-absent and boundary checks. It should not perform an overwrite PUT of
a manifest filename.

Temporary schema-compat implementation:

- Read raw source manifest bytes from `manifest/<id>.manifest`.
- Decode with generated FlatBuffer schema files pinned to the SlateDB version.
- Rewrite the fields above.
- Encode a new target manifest.
- Write `manifest/<target_id>.manifest` with `PutMode::Create`.
- Check and respect `gc/manifest.boundary` like `slatedb-txn-obj` does.

This fallback is version-sensitive and must be tested against every supported
SlateDB version.

## Copy Primitive

Object copy is deliberately independent of SlateDB record decoding:

```rust
async fn copy_file_ref(
    source: &dyn ObjectStore,
    target: &dyn ObjectStore,
    file: &FileRef,
    options: CopyOptions,
) -> Result<CopyOutcome>
```

Steps:

1. HEAD the source and compare with the job's expected metadata.
2. HEAD the target. If size and a trusted digest match, return `AlreadyPresent`.
3. For small objects, stream source GET to target PUT with create-if-absent when
   the backend supports it.
4. For large objects, use one worker process to drive multipart/ranged copy with
   bounded in-process concurrency.
5. Write to a temporary key when backend semantics do not allow conditional
   multipart writes to the final key.
6. Verify final target size and optional checksum/digest.
7. Optionally validate SST footer/index using SlateDB public SST APIs when
   available.
8. Write the job done marker only after verification succeeds.

Cross-worker multipart uploads are not portable with the generic `object_store`
trait today. They should be introduced later behind backend-specific adapters
that expose resumable upload ids or native compose/copy primitives.

## Queue and Batch State Machine

Batch states:

```text
Planned -> Copying -> Copied -> Importing -> Imported -> Complete
                    \-> Failed
```

The batch plan is immutable. Completion is derived from done markers and then
recorded in mirror state by the coordinator.

Each job attempt has states:

```text
Unclaimed -> Claimed -> Done
                    \-> Failed
                    \-> Abandoned (stale heartbeat)
```

Stale claim handling:

- Workers heartbeat long jobs at `min(claim_timeout / 3, heartbeat_interval)`.
- The coordinator lists claims and heartbeats.
- If no heartbeat is newer than `claim_timeout`, the claim is considered stale.
- A new attempt may claim the same immutable job.
- Late completion by a stale attempt is accepted if the copied target object
  verifies against the job metadata.

This avoids requiring deletes or atomic renames for correctness.

## Failure Handling

| Failure | Safe outcome | Recovery |
|---|---|---|
| Worker dies before copying | Claim becomes stale | Coordinator allows a new attempt. |
| Worker dies during multipart | Temp object or abandoned upload remains | Cleanup removes old temp data; job retries. |
| Worker finishes after being reclaimed | Same immutable bytes may already exist | Verification and done marker are idempotent. |
| Coordinator dies before import | Files may be copied but unreferenced | New coordinator resumes batch and imports. |
| Coordinator dies during import | Manifest create either happened or did not | New coordinator reads latest target manifest and retries or advances state. |
| Source checkpoint expires | Source may GC needed files | Full reconcile if possible; otherwise fail loudly and require a new base snapshot. |
| Target GC runs during batch | It may delete unreferenced copied objects | Disable target GC during mirroring or set min_age above max batch time. |
| Two coordinators run | Fencing token changes | Old coordinator stops before commit. |
| Source clone has conflicting SST ids | Ambiguous bytes for one target key | Fail unless trusted checksums prove identical. |

## Target Database Operating Rules

During active mirroring:

- Do not run a SlateDB writer on the target database.
- Do not run a target compactor unless it is explicitly mirror-aware.
- Disable target GC or configure conservative retention.
- Readers are allowed only after the first target manifest is imported. They may
  observe stale but valid snapshots.

At cutover:

1. Stop source writes or obtain a final writer-integrated checkpoint.
2. Copy and import the final source manifest.
3. Stop the mirror coordinator.
4. Optionally run target verification.
5. Open target with the intended writer/compactor/GC configuration.

## Configuration

```toml
[source.main]
url = "s3://source-bucket/prod-db"
region = "us-east-1"

[source.wal]
# optional; omit to use source.main
url = "s3://source-wal-bucket/prod-db"

[target.main]
url = "az://target-container/prod-db"

[target.wal]
# optional; omit to use target.main
url = "az://target-wal-container/prod-db"

[mirror]
checkpoint_ttl = "1h"
poll_interval = "2s"
lease_ttl = "30s"
shard_count = 256
queue_prefix = "mirror"
mode = "manifest-following" # or "writer-integrated"
wal_prefetch = true

[worker]
max_jobs = 64
max_bytes_in_flight = "4GiB"
intra_object_parallelism = 8
multipart_threshold = "64MiB"
part_size = "64MiB"
claim_timeout = "10m"
heartbeat_interval = "30s"

[target_gc]
require_disabled = true
minimum_safe_min_age = "24h"
```

Configuration must include enough information to construct both main and WAL
object stores. Do not rely on `wal_object_store_uri()` from V2 manifests; it is
not persisted there today.

## CLI

```text
s5mirror init      --config s5mirror.toml
s5mirror bulk      --config s5mirror.toml
s5mirror tail      --config s5mirror.toml
s5mirror worker    --config s5mirror.toml [--shards 0-63]
s5mirror status    --config s5mirror.toml
s5mirror verify    --config s5mirror.toml --sample 10000
s5mirror cutover   --config s5mirror.toml
s5mirror cleanup   --config s5mirror.toml
```

`bulk` and `tail` can spawn embedded workers for small deployments. Large
deployments run one coordinator and many independent `worker` processes.

## Observability

Metrics:

- `s5mirror_source_manifest_id`
- `s5mirror_target_manifest_id`
- `s5mirror_lag_manifests`
- `s5mirror_lag_seconds`
- `s5mirror_batch_state{batch_id}`
- `s5mirror_jobs_total{state,kind}`
- `s5mirror_bytes_copied_total{kind,source_store,target_store}`
- `s5mirror_copy_seconds_bucket{kind}`
- `s5mirror_checkpoint_seconds_until_expiry`
- `s5mirror_stale_claims_total`
- `s5mirror_retries_total{reason}`

Logs should include `batch_id`, `job_id`, `attempt_id`, `source_manifest_id`,
`target_manifest_id`, source path, target path, bytes, and verification method.

Status output should show:

- Active coordinator holder and fencing token.
- Stable and candidate checkpoint ids/expirations.
- Latest source and target manifest ids.
- Queue depth by kind.
- Oldest pending job age.
- Estimated lag in bytes and manifests.

## Verification

Levels of verification:

1. Object verification: every copied object has expected size and optional
   digest/checksum.
2. Manifest verification: every target manifest reference resolves to an object
   in the expected target store.
3. SlateDB open verification: open a `DbReader` on the target snapshot.
4. Data verification: compare sampled point gets or bounded scans between source
   and target at the same source checkpoint.
5. Full verification: scan both databases and compare every key/value/tombstone
   visible at the snapshot. This is expensive and should be optional.

If custom block transformers, merge operators, compaction filters, filter
policies, or segment extractors are needed to read the data, verification must
use the same application configuration. The mirror copies bytes; it does not
automatically know application-level readers.

## Testing Strategy

Unit tests:

- File-set enumeration for root manifests, segmented manifests, WAL ranges, and
  clone/external DB manifests.
- WAL range off-by-one tests: copy `(replay_after, next_wal_sst_id)` only.
- Zero-byte WAL fence preservation.
- Manifest translation rules.
- Queue claim and stale-claim state machine.
- Copy idempotency with duplicate attempts.

Integration tests with `object_store::memory::InMemory`:

- Bulk copy of a simple DB, then open/read target.
- Bulk copy with separate source and target WAL stores.
- Continuous tail while source writes and compacts.
- Source clone made self-contained on target.
- Target GC disabled/enabled safety tests.
- Coordinator crash before and after manifest import.
- Worker crash mid-copy and stale claim recovery.

Cross-provider tests:

- MinIO to Azurite.
- Local filesystem to MinIO.
- Same-backend copy fast path if implemented.

Deterministic simulation:

- Reuse `slatedb-dst` patterns for injected object-store faults, delayed lists,
  stale workers, failover, checkpoint expiry, and CAS conflicts.

Compatibility tests:

- Run schema-compat manifest decode/encode against real SlateDB V1 and V2
  manifests.
- Golden files for manifests with segments, projections, external DBs, and
  sequence trackers.

## Phased Implementation Plan

### Phase 0 - Upstream API RFC/PR

Add or agree on minimal SlateDB public APIs:

- Replication file-set enumeration.
- Raw or structured manifest export.
- Safe target manifest import.
- Public path resolution.
- Public manifest codec or a `slatedb-format` crate.

This phase removes the largest correctness risk from `s5mirror`.

### Phase 1 - Single-Process Bulk Copy

- Implement source manifest reading through `Admin`.
- Implement file-set enumeration from public getters.
- Implement object copy and verification.
- Implement target import via the new API or schema-compat fallback.
- No distributed queue yet; use an in-process bounded task pool.

### Phase 2 - Resumable Mirror State

- Add `mirror/state` transactional object.
- Add source stable checkpoint creation/refresh/delete.
- Add resume after crash.
- Add full reconcile mode.

### Phase 3 - Continuous Tail

- Add candidate checkpoints.
- Add manifest-following tail loop.
- Add WAL prefetch.
- Add lag metrics and `status`.

### Phase 4 - Distributed Workers

- Add object-store queue.
- Add leases and fencing tokens.
- Add worker processes, claims, heartbeats, stale claim recovery.
- Add cleanup for temp objects and old batches.

### Phase 5 - Production Hardening

- Cross-provider test matrix.
- Prometheus metrics.
- Structured logging.
- Cutover command.
- Verification modes.
- Runbooks for checkpoint expiry and target GC safety.

### Phase 6 - Advanced Features

- Writer-integrated source checkpoint sidecar.
- Backend-specific distributed multipart for huge single files.
- Optional preservation of external DB references.
- Filtered/projection mirroring when SlateDB exposes safe projection APIs.
- Same-provider server-side copy acceleration.

## Proposed SlateDB Improvements

The mirror becomes much simpler and safer if SlateDB grows the following APIs
and metadata.

### 1. Public Replication Snapshot API

Expose a stable API that returns a checkpointed snapshot plus all object refs
needed to materialize it elsewhere:

```rust
impl Admin {
    pub async fn create_replication_snapshot(
        &self,
        options: ReplicationSnapshotOptions,
    ) -> Result<ReplicationSnapshot, Error>;
}
```

The snapshot should include main/WAL store role, paths, ids, sizes, optional
checksums, and clone/external DB resolution.

### 2. Public Manifest Import API

Expose a safe way to write a target manifest from a snapshot. It should handle
target manifest ids, epochs, checkpoints, external DB rewriting, FlatBuffer
versions, and boundary checks.

This is the single most important improvement for a standalone mirror.

### 3. Public Format/Path Crate

Move stable format concerns into a public crate, for example `slatedb-format`:

- `PathResolver`
- `SsTableId`
- manifest and compactions codecs
- schema version constants
- file-set enumeration helpers

This keeps operational tooling from copying private internals.

### 4. Replication Slots / Leases

Add first-class replication slots that act like named checkpoints with consumer
progress:

```text
slot name
source manifest id
minimum WAL id needed
expire time
consumer metadata
```

GC would preserve all files needed by active slots. This is more explicit than
using generic checkpoints for long-running mirrors.

### 5. Writer-Visible Durable Checkpoint Requests

Provide a way for an external admin process to ask the live writer to persist a
durable checkpoint/frontier, perhaps through an object-store command file or a
small optional RPC sidecar. This would let mirrors track acknowledged durable
writes within seconds even when the source would otherwise delay manifest
persistence.

### 6. Persisted Object Store Descriptors

The separate WAL store RFC notes the difficulty of representing object stores.
For mirroring and clones, manifests should carry a stable store descriptor or
store alias for WAL and external DB references. V2 currently drops
`wal_object_store_uri`, so operators must provide WAL-store mapping out of band.

### 7. Checksums in Manifests

Add optional content digests for SST and WAL objects. ETags are provider- and
multipart-dependent; real checksums would make cross-cloud verification much
stronger and cheaper.

### 8. Cross-Store Clone

Generalize clone creation so source and target can have different object stores.
A cross-store clone is nearly the same primitive as a mirror's initial snapshot.

### 9. Mirror-Aware Target GC Guard

Add an admin guard or lease that tells target GC not to delete mirror-staged
objects younger than a mirror-owned watermark. This prevents operator mistakes
during long initial copies.

### 10. Public Compaction Output Visibility

Expose a public helper that returns compaction output SSTs that are safe/unsafe
to GC. The mirror does not need in-flight outputs for readable snapshots, but
operational tooling benefits from a stable view of compaction liveness.

## Open Questions

- Should v1 require upstream SlateDB import APIs, or should it ship a pinned
  schema-compat writer first?
- Should the target be a self-contained DB by default when the source is a
  clone? This design says yes, but preserving external references may be useful
  for cheap local mirrors.
- How should `s5mirror` discover application-level read configuration for deep
  verification, especially custom block transformers and segment extractors?
- Is a source-side RPC sidecar acceptable for low-lag production mirroring, or
  should all coordination happen through object-store files?
- Which providers expose checksums and conditional multipart semantics cleanly
  enough for backend-specific fast paths?

## Bottom Line

The core mirroring strategy is sound: pin a source snapshot, copy immutable
objects in parallel, then atomically publish a target manifest. The foolproof
version depends on being precise about WAL ranges, checkpoints, target GC,
source clones, queue semantics, and SlateDB's current public API boundaries.

The most valuable next step is to add a small SlateDB replication/export/import
surface. With that in place, `s5mirror` can stay a clean standalone Rust project
instead of becoming a fragile reimplementation of private SlateDB format logic.
