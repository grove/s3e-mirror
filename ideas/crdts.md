# Getting Maximum Value From CRDTs in s3e-mirror

**Status:** research report and design recommendations.
**Scope:** s3e-mirror core, s3e-mirror-rocklake, and future active-active data-lake replication.
**Companion documents:** [../DESIGN.md](../DESIGN.md) and [../ROCKLAKE.md](../ROCKLAKE.md).

---

## Executive Summary

CRDTs are useful for this project, but not in the way people often first imagine. They should **not** be used to merge arbitrary SlateDB database writes, and they should **not** replace the ordered manifest commit protocol. SlateDB is still a single-writer database, and the target manifest sequence still has to be committed in strict order. That part of the design should stay boring and conservative.

Where CRDTs *do* bring major value is the mirror's **control plane**: plans, copy jobs, acknowledgements, copy ledgers, progress watermarks, repair state, and multi-target replication status. Those are naturally append-only or monotonic pieces of state. If we model them explicitly as CRDTs — using **state-based CRDTs with delta optimization** on top of the object store, with **HLCs for tiebreaking**, **deterministic plan IDs for cross-replica convergence**, and **safe compaction via a proven join law** — the mirror becomes easier to recover, easier to run with multiple coordinators, easier to audit, and easier to extend toward multi-target fan-out and eventually active-active RockLake.

The near-term path:

1. Reframe `mirror_state/` as a set of **typed, versioned, state-based CRDT documents** stored in the object store.
2. Keep the target manifest commit as the only non-CRDT ordered operation.
3. Add a **copy ledger** (OR-Map from file identity to per-target copy status).
4. Add **progress vectors** keyed by source epoch.
5. Use CRDT state to enable **multi-coordinator high availability** without adding Raft, ZooKeeper, Postgres, or any other external control-plane service.
6. Treat **active-active RockLake** as a future research track, requiring upstream RockLake changes (globally unique snapshot/operation IDs, site IDs, explicit conflict policy).

The headline recommendation for [../DESIGN.md](../DESIGN.md) is: add a new section called **CRDT-Shaped Mirror State** under the safety/architecture model, then describe the queue, ledger, progress, coordinator-HA, and verification using CRDT terminology. This makes the design more rigorous without changing its core safety story.

---

## Foundations

This section pins down terminology, the consistency model, and the engineering choices that the rest of the document assumes. Skim if you already know CRDTs; do not skip it, because the engineering choices are non-default and surprising readers can disagree without realising.

### Glossary

- **CRDT (Conflict-free Replicated Data Type).** A data structure with a merge operator that is commutative, associative, and idempotent. Replicas can be updated independently and merged later, in any order, with the same final result.
- **State-based CRDT (CvRDT).** Replicas exchange full or partial *states*; merge is computed locally as the join of states. Works on top of any best-effort transport, including object-store listings.
- **Operation-based CRDT (CmRDT).** Replicas exchange *operations* and require causal broadcast. Not viable here because object stores have no built-in causal broadcast.
- **Delta-state CRDT (δ-CRDT).** A state-based CRDT optimization where replicas exchange small "deltas" that join into the full state. Compatible with object-store-as-medium: each delta is one immutable object.
- **G-Set (Grow-only Set).** Set where the only operation is `add`. Merge is set union. Used here for plans, acks, and discovered files.
- **2P-Set (Two-Phase Set).** Two G-Sets: one for adds, one for removes. Removed elements cannot be re-added. Used here for excise propagation.
- **OR-Set (Observed-Remove Set).** Set where elements carry unique tags; only observed adds can be removed. Re-add is possible. Used here for claim attempts.
- **OR-Map.** OR-Set whose elements are keys mapping to merge-able values. Used here for the copy ledger.
- **G-Counter / PN-Counter.** Grow-only counter / counter that supports increment and decrement. Used here for bytes-copied and in-flight metrics.
- **HLC (Hybrid Logical Clock).** A 64-bit timestamp that combines physical wall-clock time with a logical counter, monotonic across a process and within bounded skew across the cluster. Used here for event tiebreaking.
- **Tombstone.** A marker recording that an element was removed. CRDTs that allow removal generally need tombstones; bounding their size is a design concern.
- **Causal context.** The set of operations a replica has observed before producing its current state. State-based CRDTs do not need explicit causal context; they encode it in the value space.
- **Join (⊔).** The CRDT merge operator. Must be a [join-semilattice](https://en.wikipedia.org/wiki/Semilattice).
- **Anti-entropy.** A background process that exchanges state between replicas to reconcile any divergence. CRDTs make anti-entropy safe by construction.
- **Lease.** The exclusive right to perform a critical-section operation for a bounded time. In s3e-mirror, the mirror lease names the publisher of the next target manifest.
- **Fencing token.** A monotonically increasing identifier attached to a lease; later operations can refuse to act on behalf of an older fencing token.

### Consistency Model

s3e-mirror provides:

- **Linearizable target manifest history.** Target manifests are committed in strict monotonic order, one at a time, with `PutMode::Create` and an expected predecessor. There is no merge of two divergent target manifests.
- **Strict serializability of target reads.** A reader that opens the target sees a snapshot consistent with exactly one committed manifest, with all referenced files present.
- **Eventually consistent control plane.** Plans, acks, claims, copy ledger, progress, and repair facts are eventually consistent across all replicas (coordinators, workers, auditors), with **monotonic convergence**: once a replica observes a fact, every later observation by any replica includes that fact (modulo compaction, which preserves the join).
- **Deterministic plan convergence.** Two replicas that observe the same source state produce identical plan documents, identical plan IDs, and identical target-manifest byte sequences.
- **No silent divergence.** If two replicas compute different bytes for the same publication-step output (target manifest, deterministic plan), the conflict is detected by `PutMode::Create` failing-and-comparing, and the losing replica halts.

These five properties are the precise meaning of "fool-proof" for the control plane.

### Engineering Choices

These are the non-default engineering decisions that the rest of the report depends on. State them out loud so readers can object.

- **State-based CRDTs with delta-state optimization.** Every CRDT fact in `mirror_state/` is one immutable object. Replicas reconstruct state by listing, optionally accelerated by **checkpoint pointers** (see Operational Concerns). We never assume causal broadcast.
- **Object store as the merge medium.** There is no gossip protocol, no Raft cluster, no Postgres. The object store's `LIST` + `GET` + `PutMode::Create` is the entire substrate.
- **Deterministic serialization for hashing.** All CRDT documents are serialized as canonical JSON (RFC 8785 JCS) before hashing. Sort order is lexicographic by UTF-8 bytes; nested maps are sorted recursively; numbers are normalized to RFC 7159 grammar. Plan and manifest digests use **blake3-256** by default, xxh3-128 only for in-flight stream verification.
- **HLC for tiebreaking, never for ordering.** Wall clock skew is a real operational risk on heterogeneous worker pools. We use **HLCs** (Hybrid Logical Clocks; Kulkarni et al., 2014) for total-order tiebreaks within a single CRDT element, but ordering of database operations comes from the manifest sequence, not from clocks.
- **Versioned event schemas.** Every event document carries `schema_version: u32`. New fields are added as optional; old replicas tolerate unknown fields by passing them through verbatim during merge. Schema version is bumped only for breaking changes.
- **Compaction is mandatory, with a proven join law.** Unbounded CRDT state is not acceptable on object stores. Every CRDT in the design ships with a compaction operator and a small proof obligation (see Operational Concerns).
- **Tombstones bounded by epoch and TTL.** OR-Set tombstones are valid only within the current mirror epoch and are pruned at a TTL well past any plausible delivery delay.

If any of these choices changes, the rest of the report needs revisiting.

---

# Part I - CRDTs in Plain Terms

## What a CRDT Gives Us

A CRDT is a data structure that can be updated independently in multiple places and later merged without conflict. If two workers add different facts to the same CRDT, the merged result contains both facts. If the same fact is added twice, the merged result is still just that one fact. The order in which replicas see updates does not matter; everyone eventually converges to the same answer.

That is exactly the shape of much of the mirror's internal state. A worker can say "file X copied successfully." Another worker can say "file Y copied successfully." A coordinator can say "plan P exists." An auditor can say "target T is missing file Z." These facts can be added independently and merged later. There is no deep conflict to resolve, because these facts are append-only observations about the world.

## What a CRDT Does Not Give Us

A CRDT does not magically make SlateDB multi-writer. It does not remove the need to publish manifests in sequence. It does not make cross-cloud object writes atomic. It does not let us safely merge two arbitrary target manifests that were produced by two independent SlateDB writers.

The right mental model is:

- Use CRDTs for **mirror metadata**.
- Use the existing two-phase rule for **database correctness**.
- Use `PutMode::Create` for **publication and fencing**.
- Keep active-active database writes as a separate, explicitly designed future system.

The mistake to avoid is treating CRDTs as a replacement for ordering. In s3e-mirror, CRDTs help many processes agree on what work has been done; they do not decide which target manifest is next. The manifest sequence remains a linear history.

---

# Part II - Where the Current Design Is Already CRDT-Like

The current design already has several CRDT-shaped pieces, even though [../DESIGN.md](../DESIGN.md) does not call them that.

## Immutable Object Sets

SlateDB SSTs, WAL SSTs, manifest files, and RockLake Parquet files are immutable once published. A mirror copying these files is building a grow-only set of bytes at the target. Adding a file twice is harmless if the content is the same. Two workers copying different files commute: the final target contains both files regardless of which finished first.

This is essentially a **G-Set**:

```text
CopiedFiles = { file_ref_1, file_ref_2, file_ref_3, ... }
merge(A, B) = union(A, B)
```

The only caveat is deletion. SlateDB GC and RockLake excise mean the set is not literally grow-only forever, but deletion is a separate, explicit lifecycle stage. The copy path itself is grow-only.

## Plans

A plan under `mirror_state/plans/{plan_id}.json` is immutable. Once written, it should never be edited. Multiple coordinators seeing the same source snapshot can compute the same deterministic plan ID and converge on the same plan document. This is another G-Set:

```text
Plans = { plan_id -> plan_document }
```

If two coordinators attempt to create the same plan, `PutMode::Create` makes one win. If they compute *different* plan IDs for the same source snapshot, that is a determinism violation — a bug we catch with property tests and same-bytes-on-conflict checks (see Engineering Choices and Testing Strategy).

## Job Acknowledgements

A job ack at `mirror_state/jobs/acked/{plan_id}/{job_id}.ack` is also a grow-only fact. Once a file has been copied and verified, that fact should not be removed. Recovery is just listing the ack set.

```text
AckedJobs(plan_id) = { job_id_1, job_id_2, ... }
plan_done(plan_id) = AckedJobs(plan_id) ⊇ Plan(plan_id).jobs
```

This is the highest-value CRDT in the current design. Any coordinator replica can determine completion by reading shared object-store state. The coordinator needs no local memory of what is done.

## Claim Markers

Claim markers are subtle. Today a worker writes a claim marker with `PutMode::Create`; whoever wins owns the job until a TTL expires. Formally, claims are a **G-Set of attempts** plus a **derived register** that names the effective owner.

```text
ClaimAttempts(job_id) = { (claim_id, worker_id, fencing_token, hlc) | written by worker }
LiveClaims(job_id, now)  = { c ∈ ClaimAttempts(job_id) | c.hlc + ttl > now }
EffectiveOwner(job_id, now) = argmax over LiveClaims by (fencing_token, hlc)
                              with deterministic tiebreak on worker_id
```

The attempt history is append-only. Expired claims do not need to be deleted for correctness; they just stop being considered live. Recovery becomes inspection of historical evidence rather than mutation of in-place state.

## Progress Watermarks

Today the design mostly talks in scalar terms: last mirrored manifest ID, last target manifest ID, latest source manifest ID. That works for one source and one target. But once we add RockLake, multi-target fan-out, mirror-of-mirror, or active-active research, scalar progress is not expressive enough.

A CRDT framing models progress as a monotonic map keyed by **source epoch**:

```text
Progress = {
  (replica_id, source_epoch) -> {
    max_seen_manifest_id,
    max_committed_target_manifest_id,
    max_seen_rocklake_snapshot_id,
    max_committed_rocklake_snapshot_id
  }
}

merge(A, B) = pointwise max per (replica_id, source_epoch);
              different epochs do NOT merge — they represent different histories
```

Keying by epoch is what prevents a source rewind from silently merging with pre-rewind progress and producing nonsense.

---

# Part III - The Best Uses of CRDTs

## 1. Make `mirror_state/` an Explicit CRDT Store

`mirror_state/` should be a collection of typed, versioned, mergeable documents. This is more than naming. It gives us crisp invariants for recovery, testing, and future multi-coordinator operation.

Recommended object layout:

```text
mirror_state/
  epochs/
    {target_root_hash}/{epoch_id}.json
  plans/
    {source_epoch}/{candidate_manifest_id}/{plan_digest}.json
  jobs/
    planned/{plan_id}/{job_id}.json
    claims/{plan_id}/{job_id}/{claim_id}.json
    acks/{plan_id}/{job_id}/{ack_id}.json
    failures/{plan_id}/{job_id}/{failure_id}.json
  copy_ledger/
    by_file/{file_hash_prefix}/{file_hash}/{target_id}/{event_id}.json
    checkpoints/{checkpoint_id}.json
  progress/
    {replica_id}/{event_id}.json
    checkpoints/{checkpoint_id}.json
  repairs/
    observations/{target_id}/{file_hash}/{event_id}.json
  counters/
    {counter_id}/{replica_id}.json
```

Each file is immutable and written with `PutMode::Create`. Current state is derived by listing and joining these facts. **Checkpoint** files (compaction summaries; see Operational Concerns) accelerate cold-restart so a replica does not need to LIST tens of millions of small events.

**Value:** recovery becomes deterministic. If every coordinator and worker dies, a new coordinator can reconstruct the exact state of the mirror from object-store facts alone.

**DESIGN.md adaptation:** replace the Queue subsection in Part V with a broader **CRDT-Shaped Mirror State** subsection. Keep the existing paths, but describe them as typed G-Sets, OR-Sets, OR-Maps, and derived registers.

## 2. Add a Copy Ledger

The most valuable concrete CRDT is a copy ledger: an OR-Map from file identity to per-target copy state.

```rust
struct FileIdentity {
    source_root_id: String,
    logical_path: String,
    size: Option<u64>,
    digest: Option<Digest>,
}

struct CopyLedgerEntry {
    file: FileIdentity,
    target_id: String,
    events: BTreeSet<CopyEvent>,
}

enum CopyEvent {
    Observed       { hlc: Hlc, observer_id: String },
    CopyStarted    { hlc: Hlc, worker_id: String, claim_id: Ulid },
    CopyVerified   { hlc: Hlc, worker_id: String, digest: Digest, size: u64 },
    ReferencedByCommit { manifest_id: u64, hlc: Hlc },
    RepairNeeded   { hlc: Hlc, reason: String },
    RepairVerified { hlc: Hlc, worker_id: String },
}
```

The effective state is derived from events:

```text
missing    = no CopyVerified event for target
verified   = at least one CopyVerified event with expected digest and size
committed  = verified AND referenced by a committed manifest
repairing  = RepairNeeded with HLC > latest CopyVerified
healthy    = committed AND no later RepairNeeded
```

Merge is OR-Map join: union the key set, union event sets within each key. There is no last-writer-wins; every event is preserved.

This helps SlateDB mirroring and helps RockLake more, because RockLake warehouses can have millions of Parquet files. A ledger lets the mirror avoid re-planning from scratch on every tick and lets an auditor mark specific files for repair without touching the catalog.

**Value:** faster planning, better auditing, better repair, easier multi-target fan-out.

**DESIGN.md adaptation:** add `copy_ledger/` to the queue/data-model section; add `CopyLedgerEntry` and `CopyEvent` to the data model.

## 3. Multi-Coordinator High Availability Without Consensus

The current design intentionally has one active coordinator per target. That is safe, but it means coordinator failure causes a pause until lease takeover. CRDT state lets us run multiple coordinator replicas safely.

The key is to separate **observation** from **publication**:

- Many coordinators may observe source manifests, compute deterministic plans, and observe job acks.
- Many coordinators may conclude that a plan is complete.
- **Only one** can publish the next target manifest, because the manifest write uses `PutMode::Create` with an expected predecessor.

If two coordinators race to publish the *exact same* target manifest bytes, that is harmless: one wins, the other sees the object already exists and **verifies** the bytes match its own computed bytes. If the bytes differ, that is a hard invariant violation (determinism bug or supply-chain compromise) and the losing replica halts with an alarm. The winning replica also re-reads and verifies, to defend against last-write-wins races on the read path.

Recommended phases:

- **Phase A:** standby coordinators are read-only observers; only the lease holder can plan and commit.
- **Phase B:** standby coordinators can plan and enqueue, but only the lease holder can commit.
- **Phase C:** multiple coordinators attempt the same deterministic commit; `PutMode::Create` picks the winner; same-bytes-on-conflict verifies safety.

Phase C requires: deterministic plan IDs, deterministic target-manifest translation (including bit-for-bit FlatBuffer encoding), and explicit same-bytes-on-conflict verification with mandatory halt-on-mismatch.

**Value:** the coordinator is no longer a practical single point of failure; failover time drops from lease TTL to object-store/event-bus latency.

**DESIGN.md adaptation:** reframe the Coordinator section from "single-process per target" to "single *publisher* per manifest ID; multiple observer/coordinator replicas are a roadmap item." Add a failure-matrix row for "two coordinators race to commit same manifest" with both same-bytes and different-bytes outcomes.

## 4. Progress Vectors Instead of One Watermark

A single scalar is enough for one-way SlateDB mirroring. It is not enough for RockLake, mirror-of-mirror, multi-target fan-out, or active-active.

The project should use a progress vector keyed by source epoch:

```rust
struct SourceEpochId { source_root_id: String, epoch_id: Ulid }

struct ProgressVector {
    replica_id: String,
    observed: BTreeMap<SourceEpochId, SourceProgress>,
    targets:  BTreeMap<TargetId,    TargetProgress>,
    hlc: Hlc,
}
```

Merge is pointwise max within the same source epoch; values from *different* source epochs do not merge — they record a fork.

This is important because the design already detects source rewinds. A progress vector makes that detection explicit and auditable: instead of "manifest ID went backward," we say "source epoch changed; old progress belongs to a different epoch."

**Value:** better observability, safer mirror-of-mirror, groundwork for fan-out and active-active.

**DESIGN.md adaptation:** keep the simple scalar in v1 implementation; replace the conceptual description with progress-vector language.

## 5. Anti-Entropy Repair

CRDTs shine in systems where replicas periodically compare what they know and repair missing facts. An auditor scans the target and adds observations:

```text
RepairObservations = OR-Map<(target_id, file_hash), Set<Observation>>

enum Observation {
  Missing       { hlc, observer_id }
  SizeMismatch  { hlc, observer_id, observed_size, expected_size }
  DigestMismatch{ hlc, observer_id, observed_digest, expected_digest }
  Present       { hlc, observer_id }
}
```

Workers treat `RepairNeeded` derived states as ordinary copy jobs. When a repair succeeds, they add `RepairVerified` to the copy ledger. No special coordinator path is needed; repair is just another worker workflow with the same fencing rules.

For RockLake, this is especially valuable: a target with millions of Parquet files will eventually have *some* drift (cosmic-ray byte flips, partial deletes, cross-region replication oddities). Continuous, decentralized repair beats periodic manual reconcile.

**DESIGN.md adaptation:** Verification levels 2 and 3 can write repair facts into `mirror_state/repairs/`; workers consume these as jobs. Add metrics for repair-backlog and repair-bytes.

## 6. Multi-Target Fan-Out

Once the copy ledger is an OR-Map by `(file, target)`, mirroring one source into multiple targets becomes straightforward:

```text
file A -> target Azure:           verified
file A -> target GCS:             missing
file A -> target S3-other-account: verified
```

Workers can be target-specialized or source-region-specialized. The coordinator can commit target manifests independently once each target's required file set is present. A single source checkpoint protects all targets while they copy.

**DESIGN.md adaptation:** add multi-target fan-out as an advanced Phase 6/7 item enabled by the copy ledger.

## 7. Counters: Bytes, Files, Latency Histograms

Worker progress reporting is naturally a **G-Counter** problem (bytes-copied) or **PN-Counter** problem (in-flight jobs). Each worker increments its own slot; the global counter is the sum.

```text
GCounter = Map<replica_id, u64>
value(c) = sum(c.values())
merge(a, b) = pointwise max per replica_id
```

PN-Counters store one G-Counter per direction (increments and decrements) and report the difference. They are the right structure for in-flight gauges that must converge even with delayed observations.

These let the mirror compute global metrics from per-replica writes without a central aggregator and without losing information when workers come and go.

**DESIGN.md adaptation:** rework the `s3e_mirror_*_total` counters to be reported as per-replica facts under `mirror_state/counters/`; expose the sum at the metrics endpoint.

---

# Part IV - CRDTs and RockLake

RockLake is where CRDTs become most tempting, because its data model is already immutable and fact-like. Distinguish carefully between **safe CRDT use for mirroring** and **ambitious CRDT use for active-active writes**.

## Safe Now: RockLake Mirror State

For one-way RockLake mirroring, CRDTs are immediately useful for:

- The set of discovered Parquet files at snapshot N.
- The set of copied Parquet files per target.
- The set of verified Parquet files per target.
- The set of path-rewrite observations and outcomes.
- The set of excise events observed and propagated.
- The source/target snapshot progress vector.

This is all mirror metadata. It does not require changing RockLake's catalog semantics.

## Future: Active-Active RockLake

Active-active RockLake means two warehouses are writable at the same time and converge. CRDTs are the only plausible way to do this cleanly, but it is not a mirror feature alone — RockLake itself must change.

The promising parts (well-supported by prior art):

- **Parquet data files as an add-wins OR-Set.** Files from site A and site B union cleanly. This is the same shape as Riak's add-wins set, Automerge's `Add` operations, and Yjs's array CRDTs.
- **Delete files as immutable facts.** Same OR-Set semantics. Delete files do not conflict with anything; they additively narrow query results.
- **Data-file retirement as a remove-wins 2P-Set.** Once retired by any site, the retirement propagates and cannot be undone. This matches GDPR-style erasure: once forgotten, forever forgotten.
- **Catalog operations as an append-only operation log** rather than mutations of a single counter. The log can be merged via union and replayed deterministically. This is similar to event-sourced systems like Bedrock and Cassandra-LWW with the addition of HLC-ordering.

The hard parts (open research):

- **Global snapshot IDs.** RockLake's current `u64` counter cannot be allocated independently. Replacement: ULIDs or `(site_id, local_counter)` tuples with HLC ordering.
- **Schema conflicts are semantic, not syntactic.** No CRDT can decide whether renaming column `name` to `full_name` at site A while changing its type at site B is correct. Policy options: reject concurrent schema changes (most conservative), last-writer-wins by HLC (Cassandra-style), require manual resolution (Automerge-style). Each has known failure modes.
- **Global table/column IDs.** Need globally unique identifiers or site-scoped UUIDs.
- **Query-time ordering.** A reader at "snapshot 12345" must be replaced by "snapshot at HLC ≥ T" or "all causally before op X." Partial order replaces total order.

A plausible operation-log sketch:

```rust
struct SiteId(Uuid);
struct OpId { site: SiteId, ulid: Ulid }
struct Hlc { physical_ms: u64, logical: u64, site: SiteId }

struct CatalogOperation {
    op_id: OpId,
    deps: Vec<OpId>,
    hlc: Hlc,
    kind: CatalogOperationKind,
}

enum CatalogOperationKind {
    AddDataFile     { table_id: GlobalTableId, file: FileRef },
    AddDeleteFile   { table_id: GlobalTableId, file: FileRef },
    RetireDataFile  { file_id: GlobalFileId },
    CreateTable     { table_id: GlobalTableId, schema: TableSchema },
    AlterTable      { table_id: GlobalTableId, patch: SchemaPatch },
}
```

Each site merges operation sets and derives the visible catalog by deterministic replay. Data-file operations are mechanical; schema operations need an explicit and documented policy.

Prior art worth studying before any implementation: Riak's CRDT types (especially Maps and Sets), Automerge's columnar log, Yjs's optimized RGA, Bedrock's distributed merging, and the academic literature on JSON CRDTs (Kleppmann & Beresford, 2017) and OpSets (Kleppmann et al., 2018).

**Recommendation:** keep active-active out of v1 s3e-mirror. Add it as a research appendix. The near-term tool should make active-active *easier later* by using progress vectors, source epochs, HLCs, and CRDT-shaped mirror state now.

---

# Part V - What Not To Do

## Do Not Use CRDTs To Merge SlateDB Manifests

SlateDB manifests are not an unordered set of facts. They describe a precise LSM-tree state: WAL windows, L0 files, compacted runs, sequence numbers, checkpoints, and GC boundaries. Merging two independently-produced manifests with set union is not correct.

The target manifest history must remain linear. A CRDT can help coordinators agree that all files for manifest N are present; the manifest itself must still be published as a single ordered object with an expected predecessor.

## Do Not Use CRDTs To Justify Concurrent Writers To The Target

While the mirror is active, the target must remain read-only to SlateDB/RockLake writers. A read-only target can serve clients safely; a writable target would produce its own WAL, SSTs, and manifests, creating a second history. That is not a CRDT merge problem the current design can solve.

Promotion remains the boundary: stop mirror, verify, then open target as writer.

## Do Not Build A General CRDT Framework First

The project does not need a general CRDT library before shipping. It needs a small number of concrete mergeable records:

- G-Sets for plans and acks,
- a G-Set of attempts plus a derived register for claims,
- OR-Maps for copy ledger and repairs,
- progress vectors,
- G-Counters and PN-Counters for metrics.

Implement these directly and keep them obvious. If a general abstraction emerges after two or three use cases, extract it then.

## Do Not Let CRDT Compaction Change Meaning

Compaction must be semantics-preserving. See Operational Concerns for the proof obligation. Compaction bugs in control-plane state can cause lost work or false completion — treat them as seriously as manifest import bugs.

## Do Not Use Wall Clocks For Ordering

Wall-clock skew is real, and at multi-region scale can reach seconds. Use HLCs for tiebreaks. Use the manifest sequence for ordering. Never order CRDT events by raw wall clock.

## Do Not Trust Unauthenticated Writes To `mirror_state/`

If anyone with write access to the target bucket can write to `mirror_state/`, they can forge acks. See Security for IAM scoping and optional signing.

---

# Part VI - Proposed Data Structures

## Mirror Epoch

An epoch names one continuous history of a target root paired with one continuous history of a source root. It is how we avoid merging across rewinds and promotions.

```rust
struct MirrorEpoch {
    schema_version: u32,            // start at 1
    epoch_id: Ulid,
    source_root_id: String,
    target_root_id: String,
    started_at: Hlc,
    source_initial_manifest: Option<u64>,
    target_initial_manifest: Option<u64>,
    predecessor_epoch: Option<Ulid>,
    reason: EpochReason,
}

enum EpochReason {
    InitialBootstrap,
    NormalRestart,
    SourceRewindAcknowledged,
    TargetPromoted,
    OperatorForced,
}
```

Merge rule: epochs do not merge automatically. If two active epochs exist for the same `(source_root, target_root)`, the mirror halts and asks for operator resolution.

## Plan Set (G-Set)

```rust
struct PlanId {
    source_epoch: Ulid,
    candidate_manifest_id: u64,
    plan_digest: Digest,            // blake3-256(canonical-JSON of inputs)
}

struct PlanDocument {
    schema_version: u32,
    plan_id: PlanId,
    based_on_target_manifest: Option<u64>,
    files: Vec<FileRef>,            // sorted lexicographically by canonical key
    deterministic_manifest_digest: Digest,
    hlc: Hlc,
}
```

Merge rule: set union by `plan_id`. If two plans share `(source_epoch, candidate_manifest_id)` but disagree on `plan_digest`, this is a determinism violation — halt and alarm.

## Ack Set (G-Set)

```rust
struct AckEvent {
    schema_version: u32,
    plan_id: PlanId,
    job_id: Ulid,
    worker_id: String,
    target_path: String,
    size: u64,
    digest: Digest,
    hlc: Hlc,
}
```

Merge rule: set union. Effective ack for a job exists if at least one ack matches expected size and digest.

## Claim Attempts (G-Set) + Derived Owner

```rust
struct ClaimAttempt {
    schema_version: u32,
    plan_id: PlanId,
    job_id: Ulid,
    claim_id: Ulid,
    worker_id: String,
    fencing_token: u64,
    ttl_ms: u32,
    hlc: Hlc,
}

fn effective_owner(attempts: &[ClaimAttempt], now: Hlc) -> Option<&ClaimAttempt> {
    attempts.iter()
        .filter(|a| a.hlc.physical_ms + a.ttl_ms as u64 > now.physical_ms)
        .max_by_key(|a| (a.fencing_token, a.hlc, &a.worker_id))
}
```

Append-only; expired attempts are historical evidence only.

## Copy Ledger (OR-Map)

```rust
struct CopyLedgerKey {
    file_identity_hash: Digest,
    target_id: String,
}

struct CopyLedgerValue {
    schema_version: u32,
    events: BTreeSet<CopyEvent>,
}
```

Merge rule: union keys; union events per key. Effective status is derived (see Part III §2).

## Progress Vector

```rust
struct ProgressFact {
    schema_version: u32,
    reporter_id: String,
    source_epoch: SourceEpochId,
    target_id: String,
    max_manifest_seen: Option<u64>,
    max_manifest_planned: Option<u64>,
    max_manifest_committed: Option<u64>,
    max_rocklake_snapshot_seen: Option<u64>,
    max_rocklake_snapshot_committed: Option<u64>,
    hlc: Hlc,
}
```

Merge rule: pointwise max within the same source epoch and target. Across different source epochs: no merge — records a fork.

## Counters

```rust
struct GCounterSlot {
    schema_version: u32,
    counter_id: String,
    replica_id: String,
    value: u64,
    hlc: Hlc,
}
```

Merge rule per `(counter_id, replica_id)`: pointwise max on `value`. Global value: sum across replicas. PN-Counter: store two GCounters (`incs` and `decs`); value is `sum(incs) - sum(decs)`.

## Repair Observations (OR-Map)

```rust
struct RepairObservation {
    schema_version: u32,
    target_id: String,
    file_hash: Digest,
    observation: ObservationKind,
    observer_id: String,
    hlc: Hlc,
}

enum ObservationKind {
    Missing,
    SizeMismatch { observed: u64, expected: u64 },
    DigestMismatch { observed: Digest, expected: Digest },
    Present,
}
```

Merge rule: union all observations per `(target_id, file_hash)`. Effective state is the most recent (by HLC) non-`Present` observation, or `Healthy` if the most recent is `Present`.

---

# Part VII - Operational Concerns

These concerns are what separates a CRDT design that looks elegant on paper from one that actually runs in production for years.

## Compaction With A Proven Join Law

Unbounded CRDT state is not acceptable. Listing tens of millions of small JSON objects every restart is impractical and expensive. Every CRDT in this design ships with a **compaction operator** `compact(S)` and a **join law** that must be proven before deployment.

The join law is:

```text
For all states S and future deltas Δ:
    join(compact(S), Δ) = join(S, Δ)
```

Informally: compacting today must not change what a replica concludes tomorrow when it sees new facts. Concretely:

- **Plan set compaction.** Keep all plans whose target manifest has not been committed yet, plus the most recent N committed plans. Older committed plans are summarized into a `plans/checkpoints/{checkpoint_id}.json` document listing their `(plan_id, plan_digest, committed_manifest_id)`. Any future delta cannot legitimately reference a discarded plan; if it does, that is a forged fact and we halt.
- **Ack set compaction.** Once a plan's target manifest is committed AND every ack is reflected in the copy ledger, drop individual ack files; keep one summary per plan in `jobs/acked/checkpoints/`. The summary preserves `{(plan_id, job_id, digest, size)}` as a G-Set.
- **Claim attempts compaction.** Drop attempts whose HLC is older than `max(ttl) * 100` and whose plans have committed. Operationally rare to need older.
- **Copy ledger compaction.** Per `(file, target)`: collapse `Observed → CopyStarted → CopyVerified → ReferencedByCommit` sequences into a single `{verified_at, manifest_id}` summary. Preserve all `RepairNeeded`/`RepairVerified` events for the trailing TTL.
- **Progress compaction.** Per `(replica_id, source_epoch, target_id)`: keep only the highest-HLC fact. Older are strictly dominated.
- **Counter compaction.** Per `(counter_id, replica_id)`: keep only the highest `value`. Older are strictly dominated.

Compaction itself is performed by a designated coordinator and is published as new immutable summary objects with `PutMode::Create`. Originals are deleted only after the summary is verified by a second replica. Compaction events are themselves recorded in `mirror_state/audit/`.

Property tests for compaction (see Testing Strategy) verify the join law on randomized state and delta inputs.

## Listing Cost And Checkpoint Pointers

Even with compaction, the steady-state `LIST mirror_state/` cost grows linearly with active fact count. Two mitigations:

- **Sharded prefixes** (already in the layout): `copy_ledger/by_file/{file_hash_prefix}/...` distributes load across S3 partitions.
- **Checkpoint pointer.** A `mirror_state/CHECKPOINT_TIP` file (small, periodically rewritten) names the most recent compaction checkpoint. Replicas read the pointer first, load the checkpoint, then list *only* events newer than the checkpoint's HLC. This collapses cold-restart from "list everything" to "load one checkpoint plus a small recent tail."

The checkpoint pointer is racy (rewritten in place) but tolerable because the checkpoint files themselves are immutable and the pointer is only a hint. If the pointer is stale or wrong, the worst case is reading slightly more events than necessary; correctness is unaffected.

## Schema Versioning And Forward Compatibility

Every event document carries `schema_version: u32`. Rules:

- **Additive changes** (new optional fields) do not bump version; old replicas pass unknown fields through verbatim during merge.
- **Breaking changes** bump the version. Replicas that see a higher version than they understand refuse to merge those documents and emit `crdt_schema_skew` metrics; an operator must upgrade.
- **Documents of mixed versions** in the same set are allowed; merge is per-document.
- **Compaction may downgrade-summarize** (write summaries at a lower version) but never upgrade.

A small `mirror_state/schemas/v{N}.json` JSON-Schema file documents each version and is loaded by `s3e-mirror verify --schemas`.

## Security And Tampering

`mirror_state/` is in the *target* bucket, and write access to the target bucket is required for any worker. A malicious or compromised worker could in principle write false `Ack` events claiming work it never did, causing the coordinator to commit a manifest referencing missing data — a corruption.

Three layers of defense:

1. **IAM scoping.** Workers' credentials grant write access only to `mirror_state/jobs/claims/...` and `jobs/acks/...` and to the specific target paths their jobs name. Coordinators have additional write access to `manifest/` and `mirror_state/plans/...`. Auditors have read access plus write access only to `mirror_state/repairs/`. Operator-level access is required to write `mirror_state/epochs/` or `mirror_state/CHECKPOINT_TIP`.
2. **Coordinator verification.** Before the coordinator commits a manifest, it does not just trust the ack G-Set — it performs a HEAD on every target object the new manifest references and verifies size, with optional digest spot-check at configurable rate. A worker who forged an ack but did not actually copy the file is caught at this gate.
3. **Optional signed events.** A `signature: Option<Ed25519Signature>` field on ack and claim events, signed by the worker's key, allows post-hoc forensics on which worker forged what. Verification is optional in v1, mandatory in security-sensitive deployments.

The two-phase rule continues to hold: even with a compromised worker, the target is never published with missing data, because the coordinator verifies the bytes are present at the target before committing.

## Audit Trail

A `mirror_state/audit/` G-Set records compaction events, schema-version transitions, and operator-initiated state changes (epoch creation, promotion, forced restart). Audit events are append-only and never compacted within the audit log's own retention window (default 1 year).

---

# Part VIII - Testing Strategy

CRDTs make testing easier because each state object has a clear merge law. Property tests verify the algebraic properties; deterministic simulation tests verify behaviour under realistic schedules.

## Algebraic Properties

For each CRDT type, test on randomized inputs:

1. **Commutativity:** `merge(a, b) == merge(b, a)`.
2. **Associativity:** `merge(merge(a, b), c) == merge(a, merge(b, c))`.
3. **Idempotence:** `merge(a, a) == a`.
4. **Monotonicity:** adding a new fact never removes an already-derived true fact unless removal is explicitly modeled with tombstones.
5. **Compaction equivalence:** `join(compact(S), Δ) = join(S, Δ)` for all S and Δ.

## End-to-End Properties

6. **Deterministic planning:** two coordinators reading the same source manifest produce identical `PlanId` and `plan_digest`.
7. **Same-commit race:** two coordinators attempting the same deterministic manifest commit leave exactly one manifest object with the expected bytes; both successfully verify.
8. **Different-commit race:** two coordinators attempting different bytes for the same target manifest ID fail closed; both halt and emit `crdt_determinism_violation` alarm.
9. **Source rewind isolation:** progress facts from a different source epoch never merge with current epoch.
10. **Schema-version skew:** documents at unknown schema versions are isolated; lower-version replicas continue without them.
11. **Forged ack rejection:** an ack for a job whose target file is missing or has wrong size fails coordinator pre-commit verification.
12. **Compaction safety under failure:** kill the compactor at any point; the next compactor produces a state that still satisfies the join law.

## Deterministic Simulation Tests

Add schedules where coordinators and workers observe events in different orders, miss list results temporarily, crash after writing claims but before acks, race to publish manifests, and have wildly different HLC clocks. Final state must be independent of event order.

Lean on SlateDB's existing DST harness (`slatedb-dst`) for the underlying object-store fault injection.

---

# Part IX - Recommendations For DESIGN.md

This section is intentionally concrete: it describes exactly how to adapt [../DESIGN.md](../DESIGN.md) without turning the core design into a CRDT research project.

## Recommendation 1: Add A CRDT Posture Statement

Add a short subsection under **Part IV - Safety Model**, after **The Lease Model**:

> s3e-mirror uses state-based CRDTs for its control plane, not for SlateDB's database state. Plans, job acknowledgements, copy-ledger entries, repair observations, and progress facts are append-only mergeable facts stored under `mirror_state/`. The target manifest sequence remains linear and is committed with `PutMode::Create` and an expected predecessor. CRDTs help many workers and coordinators agree on what has happened; they do not replace the two-phase rule, the lease, or single-publisher per manifest ID. See [ideas/crdts.md](ideas/crdts.md) for the full treatment.

## Recommendation 2: Rename Queue Section To CRDT-Shaped Mirror State

In **Part V - Architecture**, replace the **Queue** subsection with **CRDT-Shaped Mirror State**. Preserve existing paths; describe them as CRDTs:

- `plans/` — G-Set of immutable plan documents.
- `jobs/planned/` — G-Set of planned jobs.
- `jobs/claims/` — G-Set of attempts with derived effective owner.
- `jobs/acked/` — G-Set of verified completions.
- `copy_ledger/` — OR-Map from `(file, target)` to copy-event sets.
- `progress/` — version-vector fact set keyed by source epoch.
- `repairs/` — OR-Map of repair observations.
- `counters/` — G-Counters and PN-Counters for metrics.

## Recommendation 3: Add CopyLedgerEntry To The Data Model

Extend the data model in **Part V** with `CopyLedgerEntry`, `CopyEvent`, `ProgressVector`, `MirrorEpoch`, and `GCounterSlot`. Use the data structures in Part VI of this report as the source of truth.

## Recommendation 4: Reframe Coordinator HA

Change the **Coordinator** section from "single-process per target" to:

- v1: one active coordinator, optional read-only standby.
- future: multiple coordinator replicas observe, plan, and race to commit deterministic target manifests; `PutMode::Create` arbitrates publication; same-bytes-on-conflict verifies safety.

Add the invariant:

> Multiple coordinators may compute and observe; only a manifest object created with the expected predecessor and the deterministic expected bytes changes target visibility.

## Recommendation 5: Progress Vectors In Observability

Add to Part IX's observability section:

- `s3e_mirror_progress_vector_source_epoch{replica}` (gauge)
- `s3e_mirror_progress_vector_manifest_seen{replica,source_epoch}` (gauge)
- `s3e_mirror_progress_vector_manifest_committed{target}` (gauge)
- `s3e_mirror_progress_vector_rocklake_snapshot_committed{target}` (gauge)
- `s3e_mirror_crdt_schema_skew_total{type}` (counter)
- `s3e_mirror_crdt_determinism_violation_total` (counter)
- `s3e_mirror_crdt_compaction_seconds` (histogram)
- `s3e_mirror_crdt_state_object_count{type}` (gauge)

## Recommendation 6: Anti-Entropy Repair

Extend **Verification** (Part VII) to say that level-2 and level-3 verification can emit repair facts into `mirror_state/repairs/`, which workers consume as ordinary copy jobs. Add a "repair backlog" metric and a "repair-bytes" counter.

## Recommendation 7: Multi-Target Fan-Out On The Roadmap

Add to **Part XII Roadmap**:

> Phase 7 — Multi-target fan-out, enabled by the copy ledger: one source checkpoint protects copies to multiple independent targets, each with its own target manifest commit and progress vector.

## Recommendation 8: CRDT Non-Goals Paragraph

Add to **Part XIII Open Questions** or **Part IV Safety Model**:

> CRDTs are not used to merge SlateDB manifests, bypass SlateDB's single-writer model, or make the target writable while mirroring is active. Active-active RockLake is a separate future design requiring RockLake-level changes (ULID snapshot IDs, schema conflict policy). Wall-clock timestamps are never used for ordering — only HLCs for tiebreaking and the manifest sequence for ordering.

## Recommendation 9: Compaction And Schema Versioning In Operations

Add a new operations subsection covering:

- CRDT state compaction (with the join law) and its scheduling.
- Schema versioning of CRDT documents and the upgrade path.
- IAM scoping for coordinators, workers, and auditors.
- Optional Ed25519 event signing for security-sensitive deployments.

## Recommendation 10: Failure Matrix Additions

Add rows for:

| Failure                                                  | Detection                            | Effect                          | Recovery |
| -------------------------------------------------------- | ------------------------------------ | ------------------------------- | -------- |
| Two coordinators commit same manifest bytes              | one wins `PutMode::Create`; other verifies match | none on target | both verify and continue |
| Two coordinators compute different bytes for same manifest ID | losing create fails; bytes mismatch on read | mirror halts; alarm | operator investigates determinism bug |
| Forged ack from compromised worker                       | coordinator HEAD-verifies pre-commit | snapshot aborts                 | revoke worker creds, audit `mirror_state/audit/` |
| CRDT schema-version skew                                 | `schema_version > known`             | document isolated; metrics rise | upgrade replicas |
| Compaction corruption (join law violated by bug)         | property tests; runtime hash check on summary | mirror halts | restore from previous summary; fix code |
| `mirror_state/` listing too slow                         | `crdt_state_object_count` exceeds threshold | latency degrades | trigger compaction; check checkpoint pointer freshness |

---

# Part X - Recommendations For ROCKLAKE.md

ROCKLAKE.md gets a smaller CRDT section than DESIGN.md, because most CRDT value belongs in the shared mirror control plane. The RockLake-specific additions:

1. **CRDT Mirror Metadata** subsection in Part V. Explain that discovered Parquet files, copied Parquet files per target, path rewrites, excise observations, and snapshot progress are all mergeable facts in `mirror_state/`. Cross-reference [ideas/crdts.md](ideas/crdts.md).
2. **Active-Active Is Future Work** in Open Questions. Active-active requires globally unique operation IDs and a schema conflict policy; out of scope for v1.
3. Upstream RockLake suggestion: **optional globally unique snapshot/operation IDs**. Keep the current `u64` counter for single-writer warehouses; add a ULID-based mode for future active-active.
4. Upstream RockLake suggestion: **bulk file-reference export with digests**, which makes the copy ledger much cheaper to populate.
5. **CRDT-friendly excise propagation**. Model excise events as a 2P-Set entry in `mirror_state/excises/`; targets observe and apply on schedule.

---

# Part XI - Alternatives Considered

A defensible CRDT design must explain why simpler or more conventional alternatives are not used.

## Why Not Raft Or Paxos?

Adding Raft (via `raft-rs` or similar) would give us strong consensus on plans and progress without CRDT subtleties. We chose not to because:

- It introduces a new persistent state machine that lives outside the object store, breaking the project's "the object store is the only durable substrate" property.
- It requires reliable inter-process networking between coordinators, which is at odds with the deployment model (sidecars, ephemeral pods, possibly multi-region).
- It does not solve the worker → coordinator fact-flow problem; we would still need an eventually-consistent layer for workers.
- The number of decisions Raft would arbitrate per second is small (one per manifest commit); the cost-benefit is poor.

## Why Not ZooKeeper Or etcd?

Same reasons as Raft, plus operational burden. Operators of a mirror tool should not be required to also operate a ZK ensemble.

## Why Not A Database (Postgres / DynamoDB) For Mirror State?

Adding a database for control-plane state is a common pattern (e.g., AWS DataSync uses DynamoDB internally). We chose not to because:

- The project's foundational promise is "your mirror has no infrastructure beyond the buckets." A DB violates that.
- Cross-cloud DB choices are complicated (DynamoDB only in AWS; Postgres needs a server somewhere).
- The CRDT approach gives the same operational benefits (HA, durability, observability) without the dependency.

The tradeoff: implementing CRDTs correctly is harder than calling a DB. We accept that, with mitigations (property tests, simulation, narrow scope of CRDT use).

## Why Not Operation-Based CRDTs?

Operation-based CRDTs require causal broadcast. Object stores provide no broadcast. Simulating causal broadcast on top of polling and LIST is possible but expensive and error-prone. State-based / delta-state CRDTs map naturally to "one immutable object per fact, LIST + GET to reconstruct."

## Why Not Just Use ETags For Versioned State Mutations?

We could in principle implement compare-and-swap on `mirror_state/CURRENT_STATE` using ETags, treating control-plane state as a single mutable document. We chose not to because:

- ETag CAS semantics vary by backend; not portable across S3, Azure, GCS in subtle ways.
- A single mutable document becomes a contention point at high worker count.
- It would degrade to "single writer of mirror state at a time," reintroducing the very bottleneck CRDTs avoid.

## Why Not A Centralized Queue (SQS, Pub/Sub)?

We use object-store events optionally for low-latency wake-up, but the queue's *durable state* lives in the object store. Reasons:

- Vendor-specific queues are not portable across clouds.
- Queue retention windows are short; mirror jobs sometimes need weeks of history.
- Acks-as-CRDT-facts give us replay and audit for free; queues do not.

## Why Not Just Use Files Without CRDT Framing?

This is the strongest alternative: treat `mirror_state/` as "a folder of facts" without invoking CRDT terminology. The current pre-CRDT design is essentially this.

We propose the CRDT framing because:

- It gives invariants (commutativity, associativity, idempotence) that are testable as property tests.
- It clarifies which operations are safe under concurrency.
- It maps cleanly to multi-coordinator HA, multi-target fan-out, and (eventually) active-active.
- It documents intent: "this is meant to be merged" vs "this is the source of truth."

The cost is vocabulary load. Mitigated by the glossary.

---

# Part XII - Implementation Roadmap

Each phase is small, shippable, and cross-references the matching phase in [../DESIGN.md](../DESIGN.md)'s roadmap.

## Phase 1: Document And Type The CRDT State

*Aligns with DESIGN.md Phase 0/1.*

- Land the DESIGN.md adaptations in Recommendations 1-3, 8, 10.
- Define JSON schemas under `mirror_state/schemas/v1.json`.
- Implement HLC type and tests.
- Implement canonical-JSON serialization and blake3-256 digest helpers.
- Add property tests for merge laws on empty G-Sets, OR-Maps, and counters.

## Phase 2: Copy Ledger For v1

*Aligns with DESIGN.md Phase 2-3.*

- Write `copy_ledger/by_file/...` events whenever a file copy is observed, started, verified, committed, or repaired.
- Use the ledger to skip already-verified files during planning.
- Implement copy ledger compaction with proven join law.
- Keep the existing queue as the primary execution path.

## Phase 3: Anti-Entropy Repair

*Aligns with DESIGN.md Phase 5.*

- Let `verify --level 2|3` emit repair events.
- Let workers consume repair events as jobs.
- Add metrics for repair backlog and repaired bytes.

## Phase 4: Progress Vectors

*Aligns with DESIGN.md Phase 4-5.*

- Replace local scalar progress state with persisted progress facts.
- Keep scalar CLI output by deriving it from the vector.
- Add source-epoch handling for rewinds/restores.

## Phase 5: Coordinator Standby

*Aligns with DESIGN.md Phase 5-6.*

- Run standby coordinators that reconstruct state and dry-run plans.
- Validate deterministic plan IDs and deterministic manifest bytes.
- Fail over faster than lease TTL, but keep a single active publisher.

## Phase 6: Multi-Coordinator Commit Race

*Aligns with DESIGN.md advanced Phase 6.*

- Allow multiple coordinators to attempt identical deterministic commits.
- Verify same-bytes-on-conflict.
- Fail closed on different-bytes-for-same-ID.

## Phase 7: Multi-Target Fan-Out

*Aligns with DESIGN.md Phase 6/7.*

- Extend `CopyLedgerKey` with `target_id` (already in v1 schema).
- Let each target progress independently.
- One source checkpoint protects N target copies.

## Phase 8: Active-Active RockLake Research

*Out of v1 scope; aligns with RockLake upstream changes.*

- Prototype globally unique operation IDs in RockLake.
- Model data files as add-wins OR-Set and retired files as 2P-Set.
- Define schema conflict policy with operator sign-off.
- Keep separate from the one-way mirror implementation until semantics are proven.

## Success Metrics

The CRDT redesign is successful when:

- **Coordinator failover time** drops from `lease_ttl_seconds` (currently 30-120s) to < 5s with two coordinators.
- **Cold restart time** for a mirror with 10M tracked file-target pairs drops below 30s (vs current O(N) list time).
- **Verification can heal** at least 99% of synthetic corruption within one verify cycle without operator intervention.
- **No determinism violation** is observed in 90 days of production for a single mirror; if one is observed, root cause is found within 24h.
- **Multi-target fan-out** to 3 targets costs less than 1.5× single-target in source-side egress (because of shared checkpoint and ledger).
- **CRDT state object count** stays under 100k under normal compaction policy for a warehouse with 10M Parquet files.

---

## Bottom Line

CRDTs bring real value to s3e-mirror when used at the right layer. The mirror's control plane is already mostly append-only, immutable, and mergeable; naming that explicitly, picking state-based with delta optimization, pinning down determinism and compaction, and adding HLC, schema versioning, IAM scoping, and a copy ledger turns the implicit design into a rigorous one. Recovery becomes deterministic, coordinator failover stops being a single point of failure, verification becomes self-healing, multi-target fan-out becomes a small extension, and the door to active-active RockLake stays open without committing to it.

The database layer remains conservative: data first, manifest last, one ordered target history, single publisher per manifest ID, two-phase rule enforced by `PutMode::Create`.

The practical recommendation is not "make the database a CRDT." It is "make the mirror's bookkeeping a CRDT — formally, testably, and with explicit operational concerns." That delivers most of the operational value quickly, without weakening the correctness story that makes s3e-mirror safe in the first place.
