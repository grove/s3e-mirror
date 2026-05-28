# s3e-mirror — Cross-Store SlateDB Mirroring

**Status:** Design / RFC, second hardening pass.
**Audience:** This document is written to be read end-to-end by an engineer who will implement the system, but the opening sections, the section introductions, and the "in plain English" paragraphs are written so that a curious non-technical reader (a product manager, an SRE lead, an executive sponsor) can follow along, understand the trade-offs, and ask sharp questions.

---

## TL;DR

s3e-mirror is a tool that takes a [SlateDB](https://slatedb.io) database that lives in one object store — say, an Amazon S3 bucket — and keeps a continuously-updated, byte-faithful, **read-openable** copy of it in a completely different object store, possibly in a different cloud provider, a different region, or a different account. It does this without ever asking the source database to slow down, without ever requiring two processes to write to the same database, and without inventing any new on-disk format: the copy is just a normal SlateDB database that any SlateDB reader can open. When the original is healthy, the copy is a cheap insurance policy that costs roughly the bandwidth of the WAL plus a small amount of metadata overhead. When the original is unhealthy — a region outage, a compromised account, a vendor problem — the copy is already a database you can promote and serve from.

The design is deliberately conservative. It assumes the network is hostile, that workers crash, that object stores lie about what they have, and that engineers will sometimes misconfigure things. Every committed step is idempotent, every published artifact is named so that two independent workers attempting the same step converge to the same bytes, and every transition from "we copied something" to "the target manifest claims it exists" is gated by a content-level check, not just a file-existed check.

---

## How To Read This Document

The document is long because mirroring is subtle, but it is also layered so you can stop reading at the depth you need.

- **Part I — What and Why** is the elevator pitch and the conceptual model. Read this if you only want to understand what the project is.
- **Part II — Background on SlateDB** explains what SlateDB actually puts in an object store. If you already know SlateDB internals, skim it.
- **Part III — Why Naive Mirroring Goes Wrong** is the most important section for reviewers. It is the catalog of mistakes the first draft made and the new mistakes we are explicitly designing around in this draft.
- **Part IV–IX** is the detailed system design: safety model, architecture, algorithms, copy mechanics, performance and cost, and operations.
- **Part X — Testing** is how we will convince ourselves the system is correct before letting it touch a real database.
- **Part XI — How SlateDB Could Help** is a wish list for upstream SlateDB changes that would make this tool simpler, faster, and lower-latency. We can ship a useful tool without them, but each one removes a category of risk or a category of polling cost.
- **Part XII–XIII** is the rollout plan and the unresolved questions.

---

# Part I — What and Why

## The Problem in One Paragraph

If you run a SlateDB database in production, that database lives inside one object store, owned by one vendor, in one region. That is a single point of failure that no amount of in-database replication can remove: SlateDB is durable because object stores are durable, but if the object store as a whole is unavailable, slow, expensive, compromised, or politically inaccessible (sanctions, lawsuits, billing disputes, account suspension), your database is unavailable too. The standard answer in industry is "make a copy somewhere else." The standard answer is annoyingly hard to do well for an LSM-tree database with a live writer, because the database is a moving target: files are appearing, the manifest is being rewritten, garbage collection is deleting things you might still want, and you cannot just point `rsync` at the bucket and hope.

## What s3e-mirror Is, In Plain English

Think of a SlateDB database as a warehouse that only ever receives new boxes — boxes are never edited in place, only stacked on top of older boxes and occasionally consolidated into bigger boxes. The "manifest" is a clipboard at the front door that lists which boxes currently form the official inventory. s3e-mirror is a fleet of delivery vans that watches the clipboard, copies every box that gets stacked, copies the clipboard updates last, and refuses to update the clipboard at the destination until every box that clipboard mentions is already physically present at the destination. Because the warehouse never edits boxes in place, the copy is always a coherent snapshot. Because we copy box-then-clipboard, the destination is always a valid warehouse — never a half-updated one — even if our vans crash midway.

## Why This Works At All

SlateDB has a property most databases do not: **every file it writes is content-addressed and immutable**. New WAL entries get new monotonically increasing IDs; new compacted SSTs get fresh ULIDs; the manifest itself is just one in a long sequence of numbered, write-once files. SlateDB never overwrites a published file in place. Combined with object-store "create-if-absent" semantics that all three major clouds now expose, this means a mirroring tool can be implemented entirely in terms of read-only operations on the source, append-only operations on the target, and ordering rules between them. There is no diffing, no merging, and no "last writer wins." It is fundamentally a copy problem, not a synchronization problem — provided we respect the ordering rules.

## What "Fool-Proof" Means Here

We use the word carefully. We cannot prove the absence of bugs, and we are not claiming to. What we are claiming is a design where:

1. The **target database is always a valid SlateDB database**, openable read-only at any time, even mid-copy and even if the mirror process is killed with `SIGKILL` at any instruction. There is never a "halfway state" published to the target's manifest.
2. The **source database is never modified, never blocked, and never has its garbage-collection latency increased** beyond a configurable checkpoint retention window the operator explicitly opts into.
3. Every committed artifact on the target is **named deterministically** so that two workers racing on the same job produce the same result, and conflicts are resolved by the object store via create-if-absent rather than by application-level locking.
4. Every operational mistake we have actually been able to think of — bad config, killed worker, paused worker, clock-skewed worker, half-finished multipart upload, partially-failed `CopyObject`, source-GC pressure, target-GC pressure, network partition, accidental dual-mirror — has a documented failure mode in [§ Failure Matrix](#failure-matrix) and a recovery procedure that does not require human reasoning beyond running `s3e-mirror recover`.

That is what we mean by fool-proof, and the rest of this document is the evidence.

---

# Part II — Background: How SlateDB Lays Out a Database

This section is the minimum amount of SlateDB internals you need before the design makes sense. Engineers comfortable with SlateDB can skim. Non-technical readers should at least read the paragraph before each table.

## The Layout on Disk

A SlateDB database is a directory in an object store, identified by a *root path*. Under that root there are a handful of well-known subdirectories. Files in those subdirectories are immutable once published — they may be deleted by garbage collection, but they are never edited. Here is the layout in summary form:

| Subdirectory   | What lives there                         | Naming                                              | Mutable? |
| -------------- | ---------------------------------------- | --------------------------------------------------- | -------- |
| `wal/`         | Write-ahead-log SSTs (recent writes)     | `0000000000000001.sst` (monotonic 19-digit u64)     | No, append-only; zero-byte files act as fences |
| `compacted/`   | Compacted SSTs (L0 and deeper levels)    | `01HXXXXXXXXXXXXXXXXXXXXXXX.sst` (ULID)             | No       |
| `manifest/`    | Manifest snapshots (the "clipboard")     | `00000000000000000001.manifest` (monotonic u64)     | No, sequenced |
| `compactions/` | Distributed-compaction journal entries   | sequenced                                           | No, sequenced; not needed for read-only target |
| `gc/`          | Garbage-collection boundary markers      | sequenced                                           | No       |
| `checkpoints/` | (Logically) named retention pins on manifests | logical, stored inside manifest entries        | Pins added/removed via manifest rewrites |

The two facts that matter most for mirroring are: (a) every published `.sst` file is byte-identical across the lifetime of the database — once a worker has copied `wal/0000000000000042.sst`, it never has to copy it again; and (b) the manifest is the only thing that has any concept of "current state" — and it is itself just another immutable, sequenced file.

## The Manifest, In One Paragraph

A SlateDB manifest is a FlatBuffer document describing which WAL SSTs are still relevant, which L0 SSTs exist, which deeper-level SSTs exist organized into runs, which logical checkpoints exist (so GC can preserve their working sets), and pointers into the sequence-number space. The two manifest fields that drive the WAL window are `replay_after_wal_id` (everything up to and including this ID is already compacted into L0, so on recovery you can skip it) and `next_wal_sst_id` (the next ID the writer will allocate). The set of WAL SSTs the writer currently considers live is therefore the open interval `(replay_after_wal_id, next_wal_sst_id)`. Mirroring must preserve every WAL file in that interval, **plus** any zero-byte WAL fence files that appear in the same range, since those are the writer's mechanism for detecting that a previous writer crashed.

## Public APIs We Can Use Today

SlateDB v0.13.0 already exposes enough of an admin surface to read a remote database's structure without forking the crate. The relevant public APIs are:

- `Admin::{read_manifest, list_manifests, read_compactions, list_compactions, list_checkpoints, create_detached_checkpoint, refresh_checkpoint, delete_checkpoint, run_gc_once, run_compactor, create_clone_builder}`
- `WalReader::{list, get}` — enumerate and fetch WAL SSTs
- `VersionedManifest` getters — read every field of a manifest we have fetched
- Builders: `AdminBuilder`, `DbBuilder`, `CloneBuilder`, `GarbageCollectorBuilder`

What is **not** yet public, and which we will either work around or upstream, is direct construction and persistence of arbitrary manifest objects (we want `Manifest::import_from`), direct use of the internal `ManifestStore`, `StoredManifest`, `FenceableManifest`, `PathResolver`, `FlatBufferManifestCodec`, and the typed `ObjectStores`/`ObjectStoreType` enums for configuring separate WAL stores. The design tracks these as upstream API requests in [Part XI](#part-xi--how-slatedb-could-help) and provides short-term workarounds in [§ Two Implementation Tracks](#two-implementation-tracks).

## Checkpoints, GC, and the Source's Safety Net

A SlateDB **checkpoint** is a logical pin on a specific manifest version. As long as a checkpoint exists, garbage collection will not delete any file referenced by the pinned manifest, even if more recent manifests have superseded that view. This is precisely the mechanism we need to keep the source's GC out of the mirror's way: before the mirror starts copying a snapshot, it asks the source to create a checkpoint with a finite lifetime; while the mirror is copying, GC cannot pull the rug out from under it; when the mirror has finished copying *and* committed that snapshot to the target, it refreshes the checkpoint to track the next snapshot and lets the old checkpoint expire.

The checkpoint TTL is the only operationally visible cost the mirror imposes on the source: a longer TTL means more headroom for slow mirroring, but it also means GC keeps more data alive on the source for longer. We default to a small multiple of the expected end-to-end snapshot lag, and we make this explicit in the configuration in [§ Configuration](#configuration).

## GC Boundary Markers and Enumeration Order

SlateDB also writes GC boundary markers such as `gc/manifest.boundary` and `gc/compactions.boundary` to fence stale writers after old sequenced files have been removed. The mirror treats these as source-side safety signals.

Correct enumeration order:

1. Create or refresh the source checkpoint.
2. Read the pinned manifest and compute the live set of WAL, compacted, manifest, compaction, checkpoint, and external-db files.
3. List/fetch the live files.
4. Read GC boundary markers **after** listing.
5. If a required file is missing and the relevant boundary advanced during the enumeration window, abort the snapshot and retry from a fresh checkpoint. If the checkpoint still claims to protect the file, escalate as a source GC contract violation.

Reading boundaries before listing is weaker: GC could advance between the boundary read and the file listing, producing a false sense of safety. Reading them after listing turns the race into a detectable abort.

## Clones and external_dbs

SlateDB supports **clones**: a database can declare in its manifest that some of its files actually live at a different root path, by listing those references in an `external_dbs` field. The clone reader resolves these transparently. The mirroring design uses this in a focused, controlled way for compacted SSTs that originated in the source: see [§ Target Manifest Translation](#target-manifest-translation). External DB resolution must be carefully designed so that the target does not accidentally depend on the source being reachable forever.

## WAL Fences and Zero-Byte Files

Sometimes a SlateDB writer crashes and a new writer takes over. The new writer cannot tell from the manifest alone exactly which WAL ID the old writer had reserved, so it uses a **fence**: it writes a zero-byte object at the next plausible WAL ID via `PutMode::Create`. If the create succeeds, the new writer knows it has fenced out any zombie writer; if it fails because the ID already exists, the new writer advances and tries again. From the mirror's point of view, those zero-byte files are part of the source's truth and **must** be preserved verbatim on the target. A mirror that "optimizes" them away will silently break the target's fencing protocol on promotion.

Fence handling rules:

- A file is treated as a WAL fence only if `size == 0` and the path matches the WAL SST naming pattern inside the live WAL interval `(replay_after_wal_id, next_wal_sst_id)`.
- Zero-byte WAL-looking files outside the live interval are not copied as part of the current snapshot; source GC owns them.
- Copying a fence uses the same `PutMode::Create` path as any other WAL file, with empty bytes and the deterministic empty digest.
- Promotion runs `verify-fences`: for the target's latest manifest, enumerate the live WAL interval and verify every zero-byte source fence appears on the target with size 0.
- A missing fence aborts promotion even if all non-empty SSTs verify, because promotion must preserve SlateDB's writer-fencing history.

Metric: `s3e_mirror_wal_fence_files_copied_total`.

---

# Part III — Why Naive Mirroring Goes Wrong

This section exists because the first draft of this design got most of the easy parts right and then ran straight into the hard parts. Anyone reviewing this document should treat the items below not as historical curiosities but as the constraints the current design is built to satisfy.

## Second-Pass Critique of the First Draft

The original draft was elegant and almost entirely wrong about the failure modes that actually matter. The new design fixes each of the following:

1. **It assumed SlateDB's internal APIs were callable.** `ManifestStore`, `PathResolver`, `FenceableManifest`, and the FlatBuffer codec are all crate-private in v0.13.0. The new design splits work into a *public-API track* that ships today by re-implementing the path conventions and FlatBuffer parsing in our own crate, and a *upstream-API track* that lobbies for these to become public and removes the duplication later.
2. **It assumed atomic rename for the queue.** Object stores do not have atomic rename. The new design replaces rename-based handoff with **create-if-absent claim markers**: when a worker picks up a job, it writes `queue/in-progress/{job_id}/{worker_id}.claim` with `PutMode::Create`. Whoever wins the create owns the job; everyone else backs off. Recovery is just listing the directory.
3. **It assumed portable cross-process multipart uploads.** They are not portable: each backend has its own upload-ID semantics and you cannot, in general, have one worker start a multipart upload and a different worker complete it via the generic `object_store` trait. The new design restricts cross-worker multipart to a *Phase 6 advanced feature* implemented per-backend (S3 first), and the default fast path is per-worker streaming GET → streaming PUT, possibly using `ObjectStore::copy_if_not_exists` when source and target are colocated.
4. **It did not handle target-side GC.** If a SlateDB GC process is pointed at the target while the mirror is also writing, GC can delete files the mirror has just copied but not yet referenced from the manifest, and the next manifest commit will reference a missing file. The new design explicitly requires the target to have **no garbage collector running** unless that GC is the mirror's own [§ Mirror-Aware Target GC](#mirror-aware-target-gc), which respects mirror-held leases.
5. **It assumed `wal_object_store_uri` was a portable description.** Manifest V2 dropped the field, and even in V1 the URI is provider-specific and may include credentials. The new design treats the source WAL store location as **operator-supplied configuration**, not as a value to read from the manifest, and it allows the target WAL store location to be entirely different.
6. **It over-trusted ETags.** ETags are not portable across providers and are not always content hashes (S3 multipart ETags famously are not). The new design uses ETags only as same-provider hints and uses an **explicit content digest** (xxh3-128 or blake3 of the bytes, computed in-stream during copy) for cross-provider integrity.
7. **It mishandled `external_dbs` resolution.** The new design treats `external_dbs` as a graph that must be walked, deduplicated, and collision-checked at the source before we even plan a copy. Two external DBs at the same root with different effective contents is a hard error, not a heuristic.
8. **It used the wrong WAL field name.** The first draft referred to `last_compacted_wal_sst_id`. The actual field is `replay_after_wal_id`, and the live interval is `(replay_after_wal_id, next_wal_sst_id)` exclusive on both ends. This is fixed throughout.
9. **It treated the candidate manifest as guaranteed to be a successor of the previous candidate.** It is not: a source writer crash, a checkpoint restore, or a clock-skew episode can produce a new candidate whose contents diverge from what the mirror expected. The new design treats every candidate as needing **a fork check** against the previously-mirrored manifest, and treats a real fork as a *cold restart from the new candidate*, not as a delta.
10. **It assumed one mirror per target.** Multiple mirrors writing to the same target is a recipe for divergence. The new design requires an exclusive **mirror lease** on the target root, enforced via a sequenced lease file that mirrors fence each other with.

## New Issues Caught In This Pass

The second hardening pass uncovered several more problems the first draft did not address:

11. **Source "rewind" detection.** If the source operator deliberately rolls the source back to an earlier manifest (restored from a backup, for example), the mirror's next poll will see a manifest whose ID is *less than* the last one it mirrored. The mirror must detect this and refuse to continue without operator acknowledgement; otherwise it can publish a target manifest that references files the source's new timeline never produced.
12. **Manifest version skew.** Manifest V1 and V2 have different fields. The mirror must parse and round-trip whichever version the source produced, and it must not silently "upgrade" a V1 manifest to V2 in the target — that would change the source-of-truth schema.
13. **Multi-region cost asymmetry.** Egress is charged on the *source* side; ingress is usually free. A naive mirror placed near the *target* will pay double egress (source-region GET pulls data out of source region, then a second hop into the target region's network). The new design specifies worker **placement** policy in [§ Placement and Cost](#placement-and-cost).
14. **Object-store rate limits.** S3 prefix throttling, Azure account throttling, GCS per-bucket QPS — naive parallel listing on a large `compacted/` directory can hit per-prefix limits. The new design introduces **adaptive concurrency** ([§ Adaptive Concurrency](#adaptive-concurrency)) and **partitioned listing** by ULID prefix.
15. **Polling latency dominates freshness at low write rates.** On a slow database, the lag is bounded by `manifest_poll_interval`, not by network speed. The new design adds an **event-driven low-latency tail** path using S3 EventBridge / Azure Event Grid / GCS Pub/Sub object notifications, with polling as the always-on fallback ([§ Event-Driven Tail](#event-driven-tail)).
16. **Promotion semantics.** The first draft hand-waved "you can open the target with SlateDB." It is true, but only if the mirror is stopped first; otherwise opening the target as a writer races the mirror's manifest commits. The new design specifies a documented **promotion protocol** in [§ Promotion and Cutover](#promotion-and-cutover).
17. **Secret hygiene.** Manifest V1 historically embedded a WAL store URI that could include credentials. The mirror's logs and observability must never echo manifest bytes blindly.
18. **Concurrent-reader compatibility.** Read-only readers opening the target while the mirror is committing can race the manifest list. SlateDB's reader is already designed around this, but the mirror must publish manifests in strictly monotonic order with no gaps, otherwise readers can skip forward over a manifest that referenced a file we have not yet copied. The new design's [§ Two-Phase Rule](#two-phase-rule) makes gaps impossible.

These eighteen items are the design constraints. The rest of the document is how they are satisfied.

---

# Part IV — Safety Model

## Two-Phase Rule

The single most important property of the system is the **two-phase rule**: a new target manifest is published **only after** every object that manifest references is durably present on the target with a verified content digest. This is the rule that makes the target always-valid and that makes worker crashes safe. We state it formally:

> For every target manifest `M` with ID `N`, at the instant `M` is created via `PutMode::Create` at `manifest/000…N.manifest`, the target object store contains a complete byte-faithful copy of every `wal/*.sst`, `compacted/*.sst`, and external-DB-resolved file that any field of `M` references.

The two-phase rule has three corollaries:

- A reader opening the target at any moment will see a valid manifest plus all of its referenced data.
- A mirror process killed at any instruction restarts cleanly because any half-copied state is either named in a way that lets it be discovered (workers' claim markers, in-flight upload directories) or named in a way that lets it be safely overwritten on retry (the destination object name is content-deterministic).
- We never need a "rollback" path on the target. There are only two states: "manifest committed and complete" and "manifest not yet committed." There is no in-between.

## Determinism Contract

The second most important safety property is **determinism**. For a fixed tuple `(source_epoch, candidate_source_manifest_id, target_predecessor_manifest_id, mirror_config_digest)`, every coordinator replica MUST compute identical:

1. `Plan` bytes,
2. `plan_digest = blake3-256(canonical-plan-bytes)`,
3. translated target manifest bytes,
4. `target_manifest_digest = blake3-256(target-manifest-bytes)`.

The deterministic input set is deliberately narrow: source manifest bytes, source root identifiers, target root identifiers, mirror configuration digest, resolved external-db descriptors, the previous target manifest digest, and the sorted file-reference set. Runtime values such as worker IDs, claim IDs, wall-clock timestamps, process start time, event arrival order, object-store listing page boundaries, and hash-map iteration order are forbidden inputs to plan or manifest bytes.

Rules:

- **Canonical plan bytes:** plans are serialized as RFC 8785 canonical JSON: UTF-8 lexical map ordering, deterministic array ordering, no insignificant whitespace, normalized numbers. `files` are sorted by `(role, source_root, logical_path, kind, expected_size, expected_digest)`.
- **Pinned FlatBuffer encoding:** manifest translation uses one vendored/pinned `slatedb-format` schema adapter per source manifest version. Generated code version is part of the `mirror_config_digest`. Multi-coordinator manifest races are disabled for any schema version whose encoder has not passed golden-byte determinism tests.
- **Digest metadata:** the target manifest object stores `s3e-plan-digest`, `s3e-target-manifest-digest`, `s3e-source-manifest-id`, `s3e-source-manifest-digest`, `s3e-mirror-version`, and `s3e-config-digest` as object metadata. On takeover, same-bytes race, or audit, the coordinator re-computes these values and compares before committing any later manifest.
- **Fail closed:** a mismatch in plan digest, target manifest digest, source manifest digest, or config digest is a determinism violation. The mirror halts, refuses further target commits, records an audit event under `mirror_state/audit/`, and requires operator intervention.

This is what makes future multi-coordinator HA safe: `PutMode::Create` picks the winner, but determinism proves that all honest coordinators were trying to publish the same bytes.

## The Lease Model

Exactly one mirror process at a time may publish manifests to a given target root. We enforce this with a sequenced **mirror lease** file at `mirror_state/lease/000…N.lease`, written with `PutMode::Create`. A lease declares a `holder_id`, a `fencing_token` (the lease's own ID `N`), and an `expires_at_ms` wall-clock deadline. A coordinator MUST renew its lease before any manifest commit; a worker MUST include its coordinator's fencing token in every claim marker. A second coordinator can take over after the deadline has passed by writing the next sequenced lease; subsequent manifest commits gated on stale tokens are then refused.

```rust
struct MirrorLease {
    holder_id: String,       // Stable ID of the coordinator process / pod
    fencing_token: u64,      // = the lease file's sequence number
    expires_at_ms: u64,      // Wall-clock TTL; advisory only, the file is the truth
    coordinator_version: String, // For operator visibility
}
```

The lease deliberately does not depend on wall-clock skew for correctness: even if two coordinators believe they hold valid leases simultaneously, the create-if-absent on the manifest commit path will reject whichever one's fencing token does not match the highest committed lease.

## CRDT-Shaped Control Plane (Posture)

s3e-mirror uses **state-based CRDTs with delta-state optimization** for its *control plane*, not for SlateDB's database state. Plans, job acknowledgements, copy-ledger entries, repair observations, progress facts, and per-replica counters are append-only mergeable facts stored as immutable objects under `mirror_state/`. The target manifest sequence remains strictly linear and is committed with `PutMode::Create` plus an expected predecessor.

CRDTs help many workers and coordinators agree on **what has happened**; they do not replace the two-phase rule, the lease, or the single-publisher-per-manifest-ID invariant. The lease arbitrates *publication*; CRDT facts are *observable* by any replica without holding the lease. HLCs (Hybrid Logical Clocks) are used for tiebreaking within a single CRDT element; ordering of database operations still comes from the manifest sequence, never from clocks.

This posture preserves the two-phase rule and enables multi-coordinator HA, multi-target fan-out, and self-healing anti-entropy repair as incremental extensions. The full treatment, including compaction with a proven join law, schema versioning, IAM scoping, and active-active research, lives in [ideas/crdts.md](ideas/crdts.md).

Explicit non-goals: CRDTs are **not** used to merge SlateDB manifests, bypass SlateDB's single-writer model, or make the target writable while mirroring is active. Active-active RockLake is a separate future design requiring RockLake-level changes (globally unique snapshot IDs, schema conflict policy). Wall-clock timestamps are never used for ordering — only HLCs for tiebreaks and the manifest sequence for true order.

## Source-Side Safety

The mirror never writes to the source store. The strongest action it takes is calling `Admin::create_detached_checkpoint`, `Admin::refresh_checkpoint`, and `Admin::delete_checkpoint` to maintain a small retention pin. The source's own GC, compactor, and writer continue to operate exactly as they would without a mirror. The only operational signal we send the source is the checkpoint TTL we have chosen.

## Target-Side Safety

The mirror writes only to its own target root, and within that root it writes to fully-namespaced subdirectories. No SlateDB writer may be opened against the target while the mirror is active. No external GC may be run against the target. The mirror's own optional mirror-aware GC, defined in [§ Mirror-Aware Target GC](#mirror-aware-target-gc), is the only thing that ever deletes files from the target.

## Failure Matrix

| Failure                                       | Detection                                  | Effect                                  | Recovery |
| --------------------------------------------- | ------------------------------------------ | --------------------------------------- | -------- |
| Worker crash mid-copy                         | claim marker stale beyond worker TTL       | none on target (no manifest committed)  | another worker claims the same job, idempotent destination name converges to same bytes |
| Worker partially uploads SST then dies        | object missing or wrong digest             | none on target                          | restart copy; deterministic dest name overwrites OK |
| Coordinator crash mid-snapshot                | lease expires                              | none on target (no manifest committed)  | next coordinator takes lease, replays plan |
| Coordinator publishes lease, never commits    | manifest ID gap visible to monitor         | none                                    | next coordinator resumes from last committed manifest |
| Source GC pressure exceeds checkpoint TTL     | mirror detects missing source file         | snapshot aborts                         | retry from a fresh checkpoint with longer TTL |
| Source GC boundary advances during enumeration | post-list boundary read crosses required file | snapshot aborts; target unchanged   | retry from fresh checkpoint; alert if checkpoint should have protected it |
| Target GC running externally                  | mirror's pre-flight check fails            | mirror refuses to start                 | operator removes external GC |
| Source rolled back to older manifest          | new candidate manifest ID < last mirrored  | snapshot aborts; alarm raised            | operator confirms intent; mirror restarts from new tip |
| Manifest schema version mismatch              | parser returns version > `manifest_max_version` | mirror halts before planning       | upgrade mirror or downgrade source; fail closed |
| Two mirrors targeting same target             | second mirror's lease create-if-absent fails | second mirror exits                    | operator picks one; revoke other |
| Cross-region partial failure mid-copy         | per-object verification fails              | retry with backoff                      | adaptive concurrency throttles |
| Workers in wrong region (cost escalation)     | observability alerts on egress             | alarm; no correctness issue             | operator moves workers |
| Manifest published referencing missing file   | **impossible by construction** (two-phase rule) | n/a                                | n/a |
| Two coordinators race to commit same manifest bytes (future HA) | one wins `PutMode::Create`; other verifies bytes match | none on target | both verify and continue |
| Two coordinators compute different bytes for same manifest ID | losing create fails; bytes mismatch on read | mirror halts; alarm | operator investigates determinism bug |
| Forged ack from compromised worker            | coordinator HEAD-verifies pre-commit       | snapshot aborts                         | revoke worker creds; audit `mirror_state/audit/` |
| CRDT control-plane schema-version skew        | document carries unknown `schema_version`  | document isolated; metrics rise         | upgrade replicas |
| CRDT state listing too slow                   | `s3e_mirror_crdt_state_object_count` exceeds threshold | latency degrades              | trigger compaction; refresh `mirror_state/CHECKPOINT_TIP` |
| Checkpoint pointer race (two compactors)       | `PutMode::Update(UpdateVersion)` fails on `CHECKPOINT_TIP` | losing compactor discards its checkpoint; no visible effect | retry from step 1; verify winning checkpoint is at least as recent |
| Checkpoint pointer regresses to older snapshot | coordinator reads newer than pointer       | cold restart reads longer tail than necessary | compactor detects and corrects on next run; correctness unaffected |
| Conditional write conflict (`AlreadyExists` / `Precondition`) | `object_store` error taxonomy             | expected race, no corruption            | read existing object, compare digest/metadata; continue if same bytes, halt if mismatch |
| Conditional write transient (`429`, `503`, provider `409`) | provider error code; retry budget          | commit delayed, target remains valid     | exponential/AIMD backoff; restart multipart uploads where provider requires it |
| Unknown SlateDB manifest schema version        | manifest parser returns version > known    | mirror halts before planning             | upgrade mirror or downgrade source; no guessed translation |
| Promotion verification fails                   | `verify --level 3` reports missing/mismatch | target remains read-only mirror          | repair or abort promotion; `promoted.marker` is not written |
| Event channel drops or reorders messages        | polling observes source tip independently  | freshness degrades only                  | fall back to polling; alert on stale event stream |
| WAL fence missing at target during promotion   | `verify-fences` finds missing zero-byte WAL | promotion aborts                         | repair by copying fence; re-run promotion verify |

---

# Part V — Architecture

## In Plain English

A small **coordinator** process sits in front of the source database and watches its manifest. When something changes, the coordinator computes a **plan**: the set of files that the target needs in order to catch up to the new source state. It writes that plan into a small **queue** that lives in the target store itself (no separate database is needed). A pool of **worker** processes — possibly running in entirely different machines, possibly in different regions — pull jobs from the queue, copy the named files from source to target, and acknowledge completion. When all of a plan's jobs are acknowledged, the coordinator translates the source manifest into a target manifest and commits it with create-if-absent. That commit is the only step that changes the *observable* state of the target. Everything before that commit is invisible to a target reader.

```
                 ┌────────────────────────────────────────────┐
                 │  Source object store (read-only access)    │
                 │  s3://prod-db/                             │
                 │  ├── manifest/                             │
                 │  ├── wal/   (possibly different store)     │
                 │  ├── compacted/                            │
                 │  ├── checkpoints/   (mirror keeps one)     │
                 │  └── compactions/                          │
                 └─────────────────┬──────────────────────────┘
                                   │ list + GET only
                                   ▼
                       ┌───────────────────────┐
                       │     Coordinator       │
                       │  - holds mirror lease │
                       │  - polls + subscribes │
                       │  - plans snapshots    │
                       │  - commits manifests  │
                       └───┬──────────────┬────┘
                           │ enqueues     │ commits
                           ▼              ▼
              ┌────────────────────┐   ┌─────────────────────────────┐
              │  Queue (in target) │   │  Target object store        │
              │  mirror_state/     │   │  az://dr-db/                │
              │  ├── lease/        │   │  ├── manifest/  ← commit pt │
              │  ├── plans/        │   │  ├── wal/                   │
              │  └── jobs/         │   │  ├── compacted/             │
              └──────┬─────────────┘   │  └── mirror_state/          │
                     │ claim+ack       └─────────────────────────────┘
                     ▼                              ▲
          ┌────────────────────┐                    │
          │   Worker pool      │  copies bytes      │
          │  - claim job       ├────────────────────┘
          │  - copy & digest   │
          │  - ack             │
          └────────────────────┘
```

## Coordinator

The coordinator is the process that holds the lease and publishes the next target manifest. It polls the source manifest (and optionally subscribes to source object-store events), decides when a new snapshot is worth materializing, computes a deterministic plan, enqueues jobs as CRDT facts, watches acks, translates the manifest, and commits.

**v1 invariant:** one active publisher per target root, gated by the lease; optional read-only standby coordinators may observe and validate but never write to `manifest/`. **Future invariant (roadmap):** multiple coordinator replicas may race to commit the *same* deterministic target manifest bytes; `PutMode::Create` arbitrates publication, and same-bytes-on-conflict verifies safety. A mismatch in computed bytes for the same manifest ID is a determinism violation and halts the mirror with an alarm. Horizontal *capacity* always comes from workers, not coordinators.

## Workers

Workers are stateless, restartable, and horizontally scalable. They each maintain a small in-memory cache of source and target `ObjectStore` clients, claim jobs from the CRDT-shaped state, stream bytes from source to target while computing a content digest in flight, write the result to a deterministic path, write an ack fact (and a `CopyVerified` ledger event), and exit the loop. A crashed worker leaves at most a stale claim attempt and, possibly, a partially-uploaded object — both of which are handled by deterministic naming, reclaim-after-TTL, and the OR-Set semantics of claim attempts.

## CRDT-Shaped Mirror State

All mirror coordination lives in the target object store under `mirror_state/` as a collection of typed, versioned, mergeable documents. There is no external coordination service. Every file is immutable and written with `PutMode::Create`; current state is derived by listing and joining facts.

File layout, with the CRDT type backing each subtree:

- `mirror_state/epochs/{target_root_hash}/{epoch_id}.json` — **G-Set** of mirror epochs (one continuous history of `(source_root, target_root)`); epochs do not auto-merge across rewinds.
- `mirror_state/plans/{source_epoch}/{candidate_manifest_id}/{plan_digest}.json` — **G-Set** of deterministic plans; two plans with the same `(source_epoch, candidate_manifest_id)` but different `plan_digest` is a determinism violation.
- `mirror_state/jobs/planned/{plan_id}/{job_id}.json` — **G-Set** of planned jobs.
- `mirror_state/jobs/claims/{plan_id}/{job_id}/{claim_id}.json` — **G-Set of claim attempts**; effective owner derived as `argmax by (fencing_token, hlc, worker_id)` filtered by TTL.
- `mirror_state/jobs/acks/{plan_id}/{job_id}/{ack_id}.json` — **G-Set** of verified completions.
- `mirror_state/jobs/failures/{plan_id}/{job_id}/{failure_id}.json` — **G-Set** of failure observations.
- `mirror_state/copy_ledger/by_file/{file_hash_prefix}/{file_hash}/{target_id}/{event_id}.json` — **OR-Map** from `(file, target)` to copy-event sets.
- `mirror_state/copy_ledger/checkpoints/{checkpoint_id}.json` — compaction summaries that satisfy `join(compact(S), Δ) = join(S, Δ)` for all future deltas Δ.
- `mirror_state/progress/{replica_id}/{event_id}.json` — **version-vector** of progress facts keyed by `(source_epoch, target_id)`; merge is pointwise max within epoch, no merge across epochs.
- `mirror_state/progress/checkpoints/{checkpoint_id}.json` — progress compaction summaries.
- `mirror_state/repairs/observations/{target_id}/{file_hash}/{event_id}.json` — **OR-Map** of repair observations from auditors; workers consume these as ordinary copy jobs.
- `mirror_state/counters/{counter_id}/{replica_id}.json` — **G-Counter** / **PN-Counter** slots; global value is the sum across replicas.
- `mirror_state/audit/{event_id}.json` — **G-Set** of operator and compaction audit events; never compacted within retention window.
- `mirror_state/CHECKPOINT_TIP` — mutable pointer to the most recent compaction checkpoint. Updated by the compactor with `PutMode::Update(UpdateVersion { e_tag, version })` after the checkpoint is verified by a second replica, preventing two compactors from racing to an older pointer. Staleness is safe; a lagging pointer means reading a longer tail, not wrong answers.
- `mirror_state/schemas/v{N}.json` — JSON-Schema documents per CRDT schema version; new fields are additive within a version, breaking changes bump `N`.
- `mirror_state/lease/000…N.lease` — sequenced single-publisher lease (unchanged from earlier description).

Legacy v1 paths (`jobs/pending/`, `jobs/claimed/`, `jobs/acked/`) are accepted by the reader during transition. See [ideas/crdts.md](ideas/crdts.md) for the formal merge laws, compaction proof obligation, and IAM scoping per subtree.

## Data Model

```rust
enum StoreRole { Main, Wal }

enum FileKind {
    Manifest { id: u64 },
    Wal { id: u64, is_fence: bool },
    CompactedSst { ulid: Ulid },
    Compactions { id: u64 },        // optional, not needed for read-only target
    GcBoundary { id: u64 },
}

struct FileRef {
    role: StoreRole,
    kind: FileKind,
    source_root: String,         // resolved if external_db
    expected_size: Option<u64>,  // hint only
    expected_digest: Option<Digest>, // mirror's own xxh3-128 / blake3-256
}

struct Plan {
    plan_id: Ulid,
    based_on_target_manifest: Option<u64>,
    candidate_source_manifest_id: u64,
    candidate_source_digest: Digest,
    files: Vec<FileRef>,
    created_at_ms: u64,
    fencing_token: u64,
}

enum Digest {
    Xxh3_128([u8; 16]),
    Blake3_256([u8; 32]),
}

// --- CRDT-shaped control-plane records ---
//
// All carry `schema_version: u32` so old replicas can isolate documents at
// unknown versions and keep operating. See ideas/crdts.md for full semantics.

struct Hlc { physical_ms: u64, logical: u32, replica_id: String }

struct SourceEpochId { source_root_id: String, epoch_id: Ulid }

struct MirrorEpoch {
    schema_version: u32,
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

struct CopyLedgerKey { file_identity_hash: Digest, target_id: String }

enum CopyEvent {
    Observed         { hlc: Hlc, observer_id: String },
    CopyStarted      { hlc: Hlc, worker_id: String, claim_id: Ulid },
    CopyVerified     { hlc: Hlc, worker_id: String, digest: Digest, size: u64 },
    ReferencedByCommit { hlc: Hlc, manifest_id: u64 },
    RepairNeeded     { hlc: Hlc, reason: String },
    RepairVerified   { hlc: Hlc, worker_id: String },
}

struct CopyLedgerEntry {
    schema_version: u32,
    key: CopyLedgerKey,
    events: std::collections::BTreeSet<CopyEvent>,
}

struct ProgressVector {
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

struct GCounterSlot {
    schema_version: u32,
    counter_id: String,
    replica_id: String,
    value: u64,
    hlc: Hlc,
}
```

---

# Part VI — Algorithms

## Enumerating the Source Snapshot

When the coordinator decides to take a new snapshot, it does so in four ordered steps:

1. **Create a fresh detached checkpoint** on the source via `Admin::create_detached_checkpoint`. Record the manifest ID `N` it pins and its digest. From this moment, every source file referenced by manifest `N` is protected from source-side GC for the checkpoint TTL.
2. **Fetch manifest `N`** via `Admin::read_manifest`. Parse every field. Resolve every `external_dbs` reference into an absolute `(root, file)` pair. Detect collisions across external DBs (two external roots claiming the same logical file with different content). Reject the snapshot on collision.
3. **Compute the live WAL interval**. `(replay_after_wal_id, next_wal_sst_id)` exclusive on both ends. List `wal/` on the source WAL store within that range, including zero-byte fence files. Reject the snapshot if any expected ID in the range is missing (this can only happen if checkpoint TTL is being violated, which should also raise an alarm).
4. **Diff against the last committed target manifest** (if any). Compute the set of files present in the new snapshot but absent from the previous one. That set, plus the new manifest itself, is the plan's input.

## Source Checkpoint Protocol

The mirror holds at most two checkpoints on the source at any time: a **stable_checkpoint** pinning the manifest version we have already mirrored to the target, and a **candidate_checkpoint** pinning the version we are currently mirroring. As soon as we commit a candidate to the target, we promote candidate to stable and refresh; once promotion is safely durable, we delete the previous stable. This double-buffering means a coordinator crash at any moment leaves the source with at most one extra checkpoint, never zero.

Checkpoint TTL is a safety budget, not a performance hint. Default:

```text
checkpoint_ttl_secs = max(3600, p99_snapshot_copy_seconds * 5)
```

The mirror refreshes the candidate checkpoint when age reaches 50% of TTL and alerts at 80%. If checkpoint age reaches 100%, the mirror stops planning new snapshots, refreshes the checkpoint first, and only resumes once the source confirms the pin. If a source file is missing while a checkpoint claims to protect it, the mirror treats this as a source-side GC contract violation and aborts the snapshot; it never papers over the gap by committing a partial target manifest.

## Bulk Copy Algorithm (Cold Start)

A cold start is the special case where no target manifest exists yet. The algorithm is the same as a snapshot delta, but every file in the snapshot is in the diff. We additionally:

- Refuse to start if the target root contains any pre-existing SlateDB files other than `mirror_state/` (operator must consent with `--allow-non-empty-target`).
- Materialize an empty initial manifest as `manifest/00000000000000000000.manifest` before any data lands, only as a marker that the root is now owned by this mirror; this manifest references nothing and is immediately superseded by the first real one. *(Optional; gated behind upstream API for direct manifest construction; see [Part XI](#part-xi--how-slatedb-could-help).)*

## Continuous Tail Algorithm

```text
loop {
    let event = wait_for_source_event(poll_interval, event_subscription);
    let candidate = source.read_latest_manifest()?;
    let last_source_id = target_metadata.last_committed_source_manifest_id;
    if candidate.id <  last_source_id { raise_rewind_alarm(); abort; }
    if candidate.id == last_source_id { continue; }
    let plan = plan_delta(target_metadata.last_source_manifest, candidate)?;
    enqueue(plan);
    wait_all_jobs_acked(plan)?;
    verify_target_has_all(plan)?;          // belt + suspenders
    commit_next_target_manifest(translate(candidate), source_manifest_id = candidate.id)?;
    advance_checkpoints();
}
```

The `wait_for_source_event` is the union of (a) a periodic poll bounded by `manifest_poll_interval`, and (b) an event subscription, when configured, that wakes the coordinator the moment a new `manifest/*.manifest` lands. See [§ Event-Driven Tail](#event-driven-tail).

## Plan Coalescing and Backpressure

When the source publishes manifests faster than the mirror can copy, the mirror does not blindly create one target manifest per source manifest forever. It uses a deterministic coalescing policy:

```text
lag_versions = latest_source_manifest_id - last_committed_source_manifest_id
backlog_bytes = sum(bytes for jobs not yet copied)

if lag_versions <= coalesce_threshold_versions
   AND backlog_bytes <= coalesce_threshold_bytes:
    mirror the next source manifest 1:1
else:
    coalesce: plan directly from last committed target manifest to latest source manifest
```

Coalescing preserves the latest database state but intentionally skips intermediate target manifests. This is safe because the target manifest remains a valid prefix endpoint: all files referenced by the coalesced target manifest are copied before commit. It is also deterministic because the decision uses only source manifest IDs, target manifest IDs, and backlog bytes derived from CRDT state.

Rules:

- The mirror records `coalesced_source_range = (first_skipped_manifest_id, latest_source_manifest_id)` in the plan and target manifest metadata.
- Coalescing is disabled during promotion drain unless the operator passes `promote --force --accept-coalescing`.
- If a user requires full target revision history, set `coalesce_threshold_versions` to a very large value and size workers/checkpoint TTL accordingly.
- Metrics: `s3e_mirror_manifests_coalesced_total` and `s3e_mirror_plan_churn_ratio`.

## WAL Prefetch Mode

WAL files are the fastest-changing part of the database and the freshest data. In WAL prefetch mode, workers also subscribe to the source `wal/` prefix (via event subscription or fast polling) and proactively copy newly-appearing WAL SSTs into the target before any manifest mentions them. This is purely an optimization: those files will not be referenced by a target manifest until the corresponding source manifest is mirrored, but having them already present means the manifest commit completes immediately.

## Target Manifest Translation

Translation is the moment where we transform a source manifest into a target manifest. The transformation rules are:

- All file IDs and ULIDs are preserved verbatim.
- `external_dbs` are resolved into local files in the target root by default (the mirror copies external DB content into the target). For very large shared bases, an operator can opt into preserving `external_dbs` pointers to *target-local* mirrors of those bases, but never to source-store URLs that could disappear.
- `wal_object_store_uri` (V1 only) is rewritten to the operator-configured target WAL store URI.
- Sequence-number fields (`recent_snapshot_min_seq`, `last_l0_seq`, `last_l0_clock_tick`, `sequence_tracker`) are preserved verbatim.
- Checkpoints from the source are **dropped** by default; the target manages its own checkpoint set. Operators can opt in to preserving named checkpoints for compatibility with downstream tooling.
- The manifest version (V1 / V2) of the source is preserved on the target.

We need a public SlateDB API to *build* and *publish* a manifest object byte-for-byte. We define the API we want in [§ Replication Snapshot and Import APIs](#replication-snapshot-and-import-apis).

```rust
struct ImportManifestOptions {
    expected_target_predecessor: Option<u64>, // commit only if target's tip == this
    preserve_external_dbs: ExternalDbStrategy,
    wal_store_remap: HashMap<String, String>,
    drop_source_checkpoints: bool,
}
```

## Manifest Schema and Encoding Contract

SlateDB manifest schemas are part of the mirror's safety boundary. The mirror does **not** guess how to translate an unknown manifest version.

Rules:

1. **Version detection first.** Read the manifest version before decoding any version-specific fields.
2. **Known version required.** If `source_manifest_version > manifest_max_version`, the mirror halts with a non-retryable schema error. Operator choices are: upgrade the mirror, downgrade the source, or disable mirroring.
3. **Same version out.** Target manifests preserve the source manifest version. A V1 source produces a V1 target; a V2 source produces a V2 target.
4. **No unknown-field promises.** FlatBuffers do not give the mirror a portable way to parse and re-emit fields from a future schema it does not know. Therefore `tolerate_unknown_fields = false` by default for manifests. Forward compatibility for CRDT control-plane JSON is separate and does allow unknown-field pass-through.
5. **Golden-byte corpus.** For every supported SlateDB minor version, the repository stores a corpus of source manifests and expected translated target manifests. Any change to the vendored schema, FlatBuffer builder, path translation, or default-field handling must pass this corpus before release.
6. **Rolling upgrade safety.** A running coordinator writes `s3e-mirror-version` and `s3e-config-digest` into plan and manifest metadata. During a rolling upgrade, old and new coordinators may both observe and plan, but only manifests whose metadata matches the active config digest may be committed. A config/schema mismatch is a determinism violation, not a retryable conflict.

Schema tests run in both directions: mirror V_old source with V_new mirror, and mirror V_new source with V_old mirror. The expected V_new -> V_old outcome is fail-closed before planning, not partial mirroring.

---

# Part VII — Object Copy Internals

## Provider-Native Fast Paths

When source and target are colocated (same provider, same region, same account, possibly different bucket), the `object_store` crate exposes `copy` and `copy_if_not_exists`, which on S3 map to `CopyObject`, on Azure to `Copy Blob`, and on GCS to `rewriteObject`. These bypass the worker's data path entirely and are dramatically cheaper. The mirror uses them whenever the source and target stores are introspectable as the same backend.

When source and target are different (the interesting case for cross-cloud mirroring), workers stream bytes through themselves. We use `ObjectStore::get_opts` to stream the source object, compute the content digest in flight, and use `ObjectStore::put_opts` with `PutMode::Create` to write the target object. The destination path is deterministic, so repeated workers attempting the same job converge.

## Conditional Writes Everywhere

All three major clouds now support create-if-absent at the object-store API level: S3 since November 2024 supports `If-None-Match: *`, Azure has supported it for years, and GCS has supported `ifGenerationMatch: 0` for years. The `object_store` crate exposes this as `PutMode::Create` portably. The design uses this primitive **everywhere a uniqueness invariant matters**: lease files, claim markers, ack markers, manifest commits, plan files, and individual SST writes that name files we believe are not yet there. We never use ETags to enforce mutual exclusion across backends; we use the upload semantics themselves.

Provider-specific semantics matter enough to state explicitly:

| Provider | Create-if-absent primitive | Update-if-current primitive | Expected conflict | Retryable transients | Notes |
| --- | --- | --- | --- | --- | --- |
| S3 | `If-None-Match: *` on PUT / CopyObject / CompleteMultipartUpload | `If-Match` on current ETag | 412 Precondition Failed; sometimes 409 Conflict around concurrent deletes | 409 (retry; restart multipart if needed), 429/503 with backoff | S3 ETag is a version token here, not a content hash; content integrity uses mirror digests. |
| Azure Blob | `If-None-Match: *` / blob create conditions | `If-Match` on current blob ETag | 412 Precondition Failed | 429/503 with backoff | Azure ETag is an opaque version identifier updated on writes. |
| GCS | `ifGenerationMatch=0` | generation/metageneration match; `object_store` preserves both through `UpdateVersion` | 412 Precondition Failed | 429/503 with backoff | Prefer generation/metageneration over ETags; GCS ETags are not the strongest version token across APIs. |
| Local/InMemory | atomic create in adapter | adapter-specific `UpdateVersion` | `AlreadyExists` / `Precondition` | n/a | Test adapters must simulate conflict ordering and retryable errors. |

The mirror classifies conditional-write results into exactly three buckets:

1. **Expected conflict** (`AlreadyExists` / `Precondition` / provider 412): read the existing object, compare digest and metadata. If bytes and required metadata match, continue. If they differ, halt with determinism violation.
2. **Retryable transient** (429, 503, provider-specific 409, network timeout before response): retry with exponential/AIMD backoff. For multipart completion conflicts, abort and restart the multipart upload, because providers may not let an already-started multipart commit be made conditional after the fact.
3. **Fatal provider error** (auth failure, unsupported precondition, malformed request, unknown schema): halt and require operator action.

The mirror never treats a conditional-write failure as success unless it has read the existing object and proven same bytes plus same required metadata.

## Optimistic Locking Protocol

Beyond `PutMode::Create`, three operations use `PutMode::Update(UpdateVersion { e_tag, version })` conditional writes to close races that create-if-absent alone cannot prevent. `UpdateVersion` preserves both ETag and provider version because stores use different combinations: S3 and Azure rely on ETags, while GCS is strongest with generation/metageneration.

### 1. Checkpoint pointer updates (UpdateVersion)

`mirror_state/CHECKPOINT_TIP` is a mutable pointer updated after every compaction. Without a conditional write, two compactors could interleave:

```
compactor_1 writes checkpoint A → (no pointer update yet)
compactor_2 writes checkpoint B → updates CHECKPOINT_TIP to B
compactor_1 updates CHECKPOINT_TIP to A   ← older checkpoint wins; B is orphaned
```

The correct protocol:

```rust
// Step 1: read current pointer and provider version token.
let tip = read_checkpoint_tip_with_update_version().await?;

// Step 2: write and verify compaction checkpoint (unchanged)
write_and_verify_checkpoint(checkpoint_id).await?;

// Step 3: update pointer — succeeds only if no one else updated it
object_store
    .put_opts(
        "mirror_state/CHECKPOINT_TIP",
        Bytes::from(checkpoint_id.to_string()),
        PutOptions { mode: PutMode::Update(tip.update_version), .. },
    )
    .await
    .map_err(|_| CheckpointRace)?;  // retry from step 1 on conflict
```

On conflict, the losing compactor re-reads the pointer, verifies the winning checkpoint is at least as recent as its own, and discards its candidate. A checkpoint can never regress because the compactor verifies the winning checkpoint's `hlc` before accepting defeat.

### 2. Plan digest in manifest metadata (defensive)

When the coordinator commits a target manifest, it embeds the plan digest used to produce it as object metadata:

```rust
object_store.put_opts(
    manifest_path,
    manifest_bytes.clone(),
    PutOptions {
        mode: PutMode::Create,
        attributes: Attributes::from_iter([
            (Attribute::Metadata("plan-digest".into()),
             AttributeValue::from(plan_digest.to_hex())),
        ]),
        ..
    },
).await?;
```

On a coordinator takeover or same-bytes race, the incoming coordinator reads the existing manifest, recomputes its own plan digest, and compares:

```rust
if existing_metadata["plan-digest"] != computed_plan_digest {
    halt!("determinism violation: plan digests differ for manifest {manifest_id}");
}
```

This catches determinism bugs one write earlier than the current same-bytes-on-conflict check, and records forensic evidence (which plan produced which manifest) in a way that survives process restarts.

### 3. Compaction checkpoint immutability

Checkpoint files themselves (`mirror_state/.../checkpoints/{checkpoint_id}.json`) are written with `PutMode::Create` and never overwritten. A compaction bug that tries to write a different checkpoint under the same ID fails immediately. Old checkpoints are deleted only after `CHECKPOINT_TIP` has been atomically advanced to a newer one (step 3 above) and the deletion is logged in `mirror_state/audit/`.

### What we deliberately do not use optimistic locking for

CRDT facts (acks, copy-ledger events, repair observations, progress, counters) use **no locking at all**. Their merge operators are commutative and idempotent, so every write is safe regardless of order. Adding If-Match or version checks there would be counterproductive — it would turn a lock-free design into a contested write for no correctness benefit.

## Multipart and Streaming

Single-worker multipart uploads use the `object_store` crate's `put_multipart` for objects above a configurable size threshold (default 64 MiB). Streams are buffered with bounded backpressure. Digests are computed incrementally during the stream so that there is never a separate read-to-hash pass.

Cross-worker multipart — where one worker initiates a multipart upload, multiple workers upload parts, and one worker completes it — is **not** in the v1 design because it is not portable through the `object_store` trait. A backend-specific variant for S3 (and only S3) is planned for Phase 6; it is gated behind a feature flag and is only useful for objects above some threshold (~5 GiB) where parallel part upload meaningfully shrinks tail latency.

## Verification

The mirror verifies each copy at three levels:

1. **In-flight digest**: while streaming, compute xxh3-128 (default) or blake3-256 (when configured); reject any copy whose computed digest does not equal the source's recomputed digest. *(For objects copied via provider-native server-side copy, we cannot see bytes; we rely on the provider's own integrity guarantees and optionally re-list-and-spot-check.)*
2. **Post-copy HEAD**: after the put completes, HEAD the target object and verify size matches expected size from the source listing.
3. **Optional periodic deep verify**: a background "auditor" job, off by default, occasionally re-reads a random subset of target SSTs and re-computes their digests against a stored expected-digest map at `mirror_state/digests/`.

### Anti-Entropy Repair

Verification levels 2 and 3 (and any future external auditor) emit **repair observations** as CRDT facts under `mirror_state/repairs/observations/{target_id}/{file_hash}/{event_id}.json`. Observations are typed: `Missing`, `SizeMismatch`, `DigestMismatch`, or `Present`. Workers consume any non-`Present` observation as an ordinary copy job, retry the affected file using the deterministic destination name, and append `CopyVerified` + `RepairVerified` events to the copy ledger on success.

No special coordinator path is required; repair is just another worker workflow gated by the same lease, fencing, and two-phase rule as initial copy. The auditor writes facts; the worker pool drains them. This makes self-healing of cosmic-ray byte flips, cross-region replication oddities, and partial deletes a built-in property rather than a separate operations playbook.

---

# Part VIII — Performance and Cost

## In Plain English

The performance of a mirror is bounded by three things: how fast the source produces new bytes, how fast the network between source and target can move those bytes, and how fast we can notice that there are new bytes. The first is a property of the workload, not the mirror. The second is a property of the network and the worker placement, and we have a lot to say about it below. The third — *how fast we notice* — is what dominates freshness for low-write-rate databases, and the default of polling once every few seconds is dramatically improvable with object-store event subscriptions.

## Throughput Model

For the dominant cross-cloud case, throughput is bounded by per-worker stream bandwidth times worker count, capped by either the source's egress bandwidth or the target's ingress rate-limits, whichever is tighter. We expect:

- Per-worker sustained throughput: ~200 MiB/s on a modest cloud VM with a single `get_opts → put_opts` stream and tuned `bytes` buffering.
- Per-worker IOPS for many small WAL SSTs: limited by HTTP request RTT, typically 50-200 ops/s; mitigated by `buffer_unordered` of many small streams.
- Wall-clock cold-start time for a 1 TB compacted set across ~10k SSTs with 16 workers: roughly an hour on commodity instances, dominated by per-object HTTP RTT for the small tail.

## Cost Model

The dominant cost of cross-cloud mirroring is **source-side egress**. Object reads from S3 to an EC2 instance in the same region are free; from S3 to anywhere else they are billed at roughly 5 cents per GiB (varies by destination). For mirrors between large clouds, this is the line item that matters. A 1 TB database is ~$50 to mirror once; an ongoing 100 GiB/day of WAL is ~$5/day on egress. Per-request costs are typically negligible for SST sizes but can dominate for short-lived databases with many tiny WAL files.

## Placement and Cost

The cheapest placement is to **run workers in the source region**. Source-region workers pull data over free intra-region links, then push outbound exactly once. Placing workers in the target region instead causes the source provider to bill egress to a destination that is "external" (sometimes the same line item, sometimes more expensive), and then the bytes still have to cross to the target. Place the *coordinator* near the *target* (so manifest commits, which are the only correctness-critical writes, have low latency to the target store), and place *workers* near the *source* (where bandwidth is free or cheap to pull). This is configurable and is the documented default.

For mirrors within the same provider, place workers anywhere — provider-native `CopyObject` does all the work server-side at the source.

## Adaptive Concurrency

The mirror starts at a low concurrency (8 in-flight requests per worker), measures error rates and latency, and adapts upward until it sees an inflection (the AIMD pattern: additive increase, multiplicative decrease). On per-prefix throttling (S3 503 SlowDown, Azure 503 ServerBusy, GCS 429), the concurrency is halved for that prefix and recovers over a configurable backoff. The mirror also **partitions listing** of the `compacted/` directory by ULID prefix (first hex character) when it exceeds 1M entries, to avoid hammering one prefix.

## Event-Driven Tail

Polling-only freshness is bounded by `manifest_poll_interval` (default 5 seconds). For applications that want sub-second mirror freshness, the mirror supports event-driven notifications:

- **AWS S3 → EventBridge → SQS**: configured on the source bucket, filter for `s3:ObjectCreated:*` on the `manifest/` prefix and (optionally) the `wal/` prefix. The coordinator consumes the SQS queue and reacts immediately.
- **Azure Blob → Event Grid → Storage Queue or Webhook**: same pattern; filter by prefix.
- **GCS → Pub/Sub**: same pattern via Pub/Sub notifications on the bucket.

Events are **purely a wake-up signal**. The coordinator always reads the actual manifest list to determine truth; events that arrive out of order or get duplicated cannot affect correctness, only latency. Polling remains on as a fallback so that a misconfigured or temporarily-broken event channel only degrades freshness, never breaks the mirror.

### Event Delivery Contract

Source events are at-least-once and unordered. The mirror assumes duplicates, delays, loss, and provider-specific batching.

Rules:

1. **Truth comes from the source store, never the event payload.** On every event or poll tick, the coordinator reads the source tip by `manifest/CURRENT` if available (future upstream request), otherwise by listing `manifest/` and selecting the highest valid manifest.
2. **Deduplication is an optimization only.** The event loop keeps a five-minute HLC window keyed by `(provider_event_id, object_path, object_generation)` and drops duplicates, but correctness is unchanged if dedupe is disabled.
3. **WAL events never commit manifests.** WAL events may wake prefetch workers, but only manifest enumeration can create a plan. A WAL event that arrives before its manifest simply prefetches bytes.
4. **Queue backpressure falls back to polling.** If event backlog exceeds `queue_depth_poll_fallback` (default 1000), the coordinator drains or discards event payloads and switches to polling-only until the backlog recovers. This preserves correctness and bounds memory.
5. **Stale stream alerts.** If no event arrives for `stale_event_timeout_secs` and polling is still healthy, emit an advisory alert. If polling also fails, emit a critical alert because the coordinator has lost both wake-up paths.

This contract gives sub-second freshness when the event channel is healthy, but does not make correctness depend on any event provider's ordering or delivery guarantees.

## Sustained-Latency Mode

For applications that need *consistently* low end-to-end lag — say, "the target is never more than 2 seconds behind the source" — the mirror runs in a mode that combines: event-driven manifest wake-up, WAL prefetch always on, workers permanently held warm (no spin-down), and an upper bound on snapshot batching so that large bursts of WAL files cannot starve manifest commits.

---

# Part IX — Operations

## Configuration

```toml
[source.main]
provider = "s3"
bucket   = "prod-db"
prefix   = ""
region   = "us-east-1"

[source.wal]                       # Optional; omit if WAL shares main store
provider = "s3"
bucket   = "prod-db-wal"
region   = "us-east-1"

[source.events]                    # Optional; enables event-driven tail
type     = "aws_sqs"
queue_url = "https://sqs.us-east-1.amazonaws.com/123/source-events"
prefixes  = ["manifest/", "wal/"]

[target.main]
provider = "azure"
account  = "drcompany"
container = "dr-db"
prefix   = ""

[target.wal]
provider = "azure"
account  = "drcompany"
container = "dr-db-wal"

[mirror]
mode                 = "continuous"   # snapshot | continuous | failover | verify
checkpoint_ttl_secs  = 3600
manifest_poll_interval_ms = 5000
wal_prefetch         = true
external_dbs_policy  = "inline"       # inline | preserve_remapped
digest_algo          = "xxh3-128"     # xxh3-128 | blake3-256
allow_non_empty_target = false
coalesce_threshold_versions = 128      # when behind, skip intermediate manifests
coalesce_threshold_bytes    = 1073741824 # or coalesce once backlog exceeds 1 GiB

[worker]
count                = 16
in_flight_per_worker = 8              # initial; adaptive
multipart_threshold_bytes = 67108864
multipart_part_bytes      = 16777216
placement_hint       = "source_region"

[target_gc]                          # Mirror-aware GC; off by default
enabled = false
retain_window_hours = 24
retain_manifest_count = 1024
find_orphans = false                  # expensive deep scan, opt-in

[schema]
manifest_max_version = 2              # source manifests above this fail closed
tolerate_unknown_fields = false       # FlatBuffers: unknown manifest fields are not preserved by parsing

[events]
stale_event_timeout_secs = 300
queue_depth_poll_fallback = 1000
dedupe_window_secs = 300

[observability]
metrics_endpoint = "0.0.0.0:9090"
log_level        = "info"
redact_manifest_uris = true
```

## CLI

```text
s3e-mirror init     --config s3e-mirror.toml
s3e-mirror plan     --config s3e-mirror.toml     # dry-run; emits planned bytes & cost
s3e-mirror run      --config s3e-mirror.toml
s3e-mirror status   --config s3e-mirror.toml     # shows lag, queue depth, last commit
s3e-mirror verify   --config s3e-mirror.toml --level 1|2|3|4|5
s3e-mirror verify-fences --config s3e-mirror.toml
s3e-mirror verify-gc-fencing --config s3e-mirror.toml
s3e-mirror repair   --config s3e-mirror.toml --priority newest|oldest|smallest
s3e-mirror corruption-audit --config s3e-mirror.toml
s3e-mirror gc       --config s3e-mirror.toml --dry-run [--include-orphans]
s3e-mirror promote  --config s3e-mirror.toml     # safe cutover; see § Promotion
s3e-mirror promotion-status --config s3e-mirror.toml
s3e-mirror promotion-abort  --config s3e-mirror.toml
s3e-mirror recover  --config s3e-mirror.toml     # idempotent restart helper
```

## Observability

The mirror exposes Prometheus-style metrics:

- `s3e_mirror_source_manifest_id` (gauge)
- `s3e_mirror_target_manifest_id` (gauge)
- `s3e_mirror_lag_seconds` (gauge; wall-clock between source manifest timestamp and target commit)
- `s3e_mirror_lag_manifest_versions` (gauge)
- `s3e_mirror_bytes_copied_total` (counter, labeled by `role`)
- `s3e_mirror_objects_copied_total` (counter, labeled by `kind`)
- `s3e_mirror_copy_errors_total` (counter, labeled by `phase`)
- `s3e_mirror_worker_in_flight` (gauge per worker)
- `s3e_mirror_queue_depth` (gauge)
- `s3e_mirror_checkpoint_age_seconds` (gauge for source-side checkpoint)
- `s3e_mirror_egress_bytes_total` (counter; equals the source cloud's bill line item)
- `s3e_mirror_progress_vector_source_epoch{replica}` (gauge; current source epoch ID per replica)
- `s3e_mirror_progress_vector_manifest_seen{replica,source_epoch}` (gauge)
- `s3e_mirror_progress_vector_manifest_committed{target}` (gauge)
- `s3e_mirror_progress_vector_rocklake_snapshot_committed{target}` (gauge)
- `s3e_mirror_crdt_schema_skew_total{type}` (counter; documents at unknown schema versions)
- `s3e_mirror_crdt_determinism_violation_total` (counter; should remain zero)
- `s3e_mirror_crdt_compaction_seconds` (histogram, labeled `type`)
- `s3e_mirror_crdt_state_object_count{type}` (gauge; pre- and post-compaction)
- `s3e_mirror_repair_backlog{target}` (gauge; pending repair observations)
- `s3e_mirror_repair_bytes_total` (counter; bytes re-copied by repair workers)
- `s3e_mirror_determinism_check_failures_total` (counter; immediate page)
- `s3e_mirror_manifest_schema_version{version}` (gauge; current source/target manifest version)
- `s3e_mirror_manifest_schema_mismatch_total` (counter; source version above mirror support)
- `s3e_mirror_conditional_write_conflicts_total{provider,operation}` (counter; expected optimistic conflicts)
- `s3e_mirror_conditional_write_transients_total{provider,status}` (counter; retryable provider throttles/conflicts)
- `s3e_mirror_event_queue_depth` (gauge; event backlog)
- `s3e_mirror_event_deduplicated_total` (counter; duplicate wake-up events ignored)
- `s3e_mirror_event_stream_stale_seconds` (gauge; seconds since last event)
- `s3e_mirror_source_checkpoint_ttl_seconds` (gauge)
- `s3e_mirror_source_checkpoint_refreshes_total` (counter)
- `s3e_mirror_source_gc_detected_missing_file_total` (counter; non-zero means operator action)
- `s3e_mirror_wal_fence_files_copied_total` (counter)
- `s3e_mirror_plans_created_total` (counter)
- `s3e_mirror_manifests_coalesced_total` (counter; intermediate source manifests intentionally skipped)
- `s3e_mirror_plan_churn_ratio` (gauge; plans created / source manifests observed)

Logs are structured JSON, never include manifest payloads verbatim (the manifest may contain URIs with credentials in older V1 schemas), and are redacted by default.

Counters whose `_total` form is named above are reported as per-replica G-Counter slots under `mirror_state/counters/`; the metrics endpoint sums them. This avoids losing counts when workers come and go and removes the need for a central aggregator.

## Promotion and Cutover

Promotion turns the mirror copy into the active read-write database. It is a state machine, not a loose checklist. The durable state lives under `mirror_state/promotion/{promotion_id}/` and every transition is written with `PutMode::Create`.

```rust
enum PromotionState {
    PrePromotion,
    DrainInProgress,
    DrainTimedOut,
    VerificationInProgress,
    VerificationFailed,
    LeaseReleasing,
    Promoted,
    Aborted,
}
```

Allowed transitions:

| From | Event | To | Notes |
| --- | --- | --- | --- |
| `PrePromotion` | `s3e-mirror promote` | `DrainInProgress` | writes `promotion_started` fact; refuses if another active promotion exists |
| `DrainInProgress` | target catches source | `VerificationInProgress` | exact source/target manifest IDs recorded |
| `DrainInProgress` | timeout | `DrainTimedOut` | operator chooses retry, abort, or `promote --force --accept-lag=N` |
| `DrainTimedOut` | force accepted | `VerificationInProgress` | records accepted source lag and operator ID |
| `VerificationInProgress` | level-3 verify passes + fences pass | `LeaseReleasing` | target still read-only |
| `VerificationInProgress` | verify fails | `VerificationFailed` | no `promoted.marker`; mirror may repair and retry |
| `VerificationFailed` | `promotion-abort` | `Aborted` | mirror resumes from previous lease state |
| `VerificationFailed` | repair complete | `VerificationInProgress` | re-run verification from scratch |
| `LeaseReleasing` | promoted marker created | `Promoted` | writes `mirror_state/promoted.marker` with promotion ID and target manifest digest |

`mirror_state/promoted.marker` is the point of no return for this mirror epoch. Any later `s3e-mirror run` invocation refuses to start unless the operator explicitly runs `s3e-mirror reverse` or `s3e-mirror restart --new-epoch`. This prevents resurrecting a mirror against a target that may now have accepted writes.

Promotion commands:

```text
s3e-mirror promote           --config s3e-mirror.toml [--wait] [--force --accept-lag=N]
s3e-mirror promotion-status  --config s3e-mirror.toml
s3e-mirror promotion-abort   --config s3e-mirror.toml
s3e-mirror verify-fences     --config s3e-mirror.toml
```

Promotion checks before `Promoted`:

1. Target latest manifest opens cleanly.
2. Every referenced object exists and passes level-3 verification.
3. Every zero-byte WAL fence in the live WAL interval exists on the target with size 0.
4. `s3e-plan-digest`, `s3e-target-manifest-digest`, and `s3e-config-digest` metadata match recomputed values.
5. No pending `RepairNeeded` facts affect the latest target manifest.
6. No downstream mirror has declared `mirror_state/chain/predecessor_source` pointing at this target, unless operator passes `--acknowledge-chain-break`.

If any check fails, promotion remains in `VerificationFailed`; the target stays read-only and the worker pool may continue repair.

## Corruption Detection and Recovery Runbook

Verification failures are actionable facts, not vague alarms.

| Symptom | Likely cause | Automated action | Operator action |
| --- | --- | --- | --- |
| Level 1 missing object | worker never finished copy, target object deleted manually, or provider list/read inconsistency | write `Missing` repair fact | wait for repair; investigate if repeated |
| Level 2 size mismatch | partial upload or wrong destination object | write `SizeMismatch` repair fact | revoke/inspect worker if repeated |
| Level 3 digest mismatch | target corruption, source object changed unexpectedly, or digest bug | re-read both sides; write `DigestMismatch` repair fact | do not promote until latest manifest verifies clean |
| Repair fact not draining | worker pool down, claim stuck, credentials broken | claim TTL expires and job re-enters pool | restart workers; check IAM |

Promotion rule with known corruption:

```text
if corruption affects latest target manifest:
    do not promote; repair and re-run verify --level 3
else if corruption affects only older retained manifests:
    verify latest target manifest explicitly;
    if latest is clean, promotion may proceed with an audit warning
else:
    promotion may proceed
```

Repair commands:

```text
s3e-mirror verify --config s3e-mirror.toml --level 3 --scope manifest:<id>
s3e-mirror repair --config s3e-mirror.toml --priority newest|oldest|smallest --max-files N
s3e-mirror corruption-audit --config s3e-mirror.toml
```

`corruption-audit` reports oldest affected manifest, latest affected manifest, files awaiting repair, files successfully repaired, and estimated time to clean at current worker throughput.

## Operational Runbooks

These are the first actions an on-call operator should take. They are intentionally short; the CLI prints the same decision tree with current values filled in.

| Incident | Trigger | First checks | Safe action |
| --- | --- | --- | --- |
| Mirror lag rising | `s3e_mirror_lag_seconds` above alert threshold | checkpoint age vs TTL, worker count, target throttling, source write rate | if checkpoint age > 80% TTL, increase `checkpoint_ttl_secs`; then add workers or enable coalescing |
| Source rewind detected | candidate manifest ID/digest behind or divergent from last mirrored | confirm operator restore vs source bug | if expected, run `s3e-mirror restart --after-source-rewind --new-epoch`; if not, stop and inspect source |
| Coordinator lease stuck | no manifest commits; lease expired or renewing late | promoted marker, current holder ID, process health | if no promoted marker, start standby coordinator; stale holder cannot commit with old fencing token |
| Worker pool down | `s3e_mirror_worker_in_flight == 0` and `s3e_mirror_queue_depth > 0` | worker logs: auth, network, resource pressure | restart workers; claims expire automatically; do not touch manifests |
| Dual mirror conflict | second mirror exits on lease create failure | which source/target pair is intended | revoke credentials for wrong mirror, verify survivor, keep target read-only until quiet |
| Conditional-write storm | conflict/transient counters spike | same-bytes vs mismatch, provider status, recent deploy | same-bytes conflicts are benign; mismatch halts; transients trigger backoff and provider quota check |

No runbook step asks an operator to edit manifests by hand. Manual object-store surgery is limited to revoking credentials, increasing TTLs, restarting processes, or deleting stale temp uploads after `verify --find-orphans` confirms they are unreachable.

## CRDT State Cold-Restart Performance

Without mitigation, a replica starting from nothing reconstructs state by listing every object under `mirror_state/`. At scale (millions of tracked file-target pairs) that is tens of seconds at best and minutes at worst. Five complementary techniques keep it well under 10 seconds in practice:

**1. Checkpoint-then-tail (primary mitigation).**
Every 15 minutes (configurable; also triggered when `s3e_mirror_crdt_state_object_count` crosses a threshold), the coordinator writes a compacted checkpoint: one JSON document per CRDT type that represents the merged state up to that point. `CHECKPOINT_TIP` is updated atomically after a second replica verifies the checkpoint is correct. A cold-starting replica reads:

```
1. GET mirror_state/CHECKPOINT_TIP           (~1 ms)
2. GET mirror_state/.../checkpoints/{id}.json  (~10-50 ms per type; ~5 files)
3. LIST mirror_state/.../  filtered to  hlc > checkpoint.hlc  (~1-2 s; small tail)
4. GET the small tail of new events            (~100-500 ms)
```

With 15-minute checkpoints, the tail is bounded: a busy mirror produces at most a few thousand events per 15-minute window, regardless of how many total facts exist.

**2. Parallel LIST across sharded prefixes.**
Every subtree under `mirror_state/` is sharded on the first N hex digits of the fact's natural key (file hash, job ID, etc.). This serves two purposes: it distributes load across S3 partition boundaries (avoiding hot-key throttling), and it lets cold-restart LIST all shards concurrently. Step 3 above fans out across all shards simultaneously rather than scanning one prefix sequentially.

**3. Streaming state reconstruction.**
A replica does not have to wait for the full state before it starts operating. Workers can start claiming jobs as soon as the `claims` and `acks` subtrees are loaded. The coordinator can start planning once `plans` and `progress` are loaded. Reading `copy_ledger` (the largest subtree) overlaps with the coordinator's first planning pass. The cold-restart "pause" is only as long as the minimum state needed for the first operation.

**4. Compaction-triggered checkpoint writes.**
Every compaction run writes a new checkpoint and updates `CHECKPOINT_TIP` immediately. This means the pointer is always fresh (within one compaction cycle, not within one wall-clock hour). The tail that accumulates between checkpoints is proportional to compaction frequency, not to system uptime.

**5. On-demand compaction when the tail grows.**
If `s3e_mirror_crdt_state_object_count{type}` rises above a configurable high-water mark (default: 50k per type), the coordinator triggers an out-of-schedule compaction run. This is the safety valve: even if the regular schedule is missed (coordinator was down), the first healthy coordinator trims the tail before the next restart would be slow.

**Expected cold-restart times with all five applied:**

| Scale | Without mitigations | With mitigations |
|---|---|---|
| 100k file-target pairs | ~5s | ~1s |
| 1M file-target pairs | ~60s | ~3s |
| 10M file-target pairs | ~10 min | ~5-8s |

New metrics for observability:
- `s3e_mirror_cold_restart_seconds` (histogram; measured on startup, labeled by what triggered cold-start)
- `s3e_mirror_crdt_tail_object_count{type}` (gauge; objects written since last checkpoint — the "tail size")
- `s3e_mirror_checkpoint_tip_age_seconds` (gauge; time since `CHECKPOINT_TIP` was last updated)
- `s3e_mirror_checkpoint_write_seconds` (histogram; time to compact and write a new checkpoint)

## Mirror-Aware Target GC

The target accumulates files; eventually we want to delete files that are no longer referenced by any committed target manifest and that have not been referenced for some configurable retention window. This is **off by default**. When enabled, the mirror-aware GC has a narrow contract:

- Reads only committed target manifests.
- Computes the union of referenced files across both `retain_window_hours` and `retain_manifest_count`; an object is protected if either rule keeps it.
- Honors the same mirror lease as the coordinator; refuses to run if no current lease exists.
- Refuses to run while promotion is in progress.
- Refuses to delete a file named by any pending plan, active claim, uncommitted copy-ledger entry, or repair observation.
- Logs every candidate and every deletion as an immutable audit event, with dry-run mode on by default for the first run.
- Never deletes `mirror_state/` except archived plans, ack summaries already reflected in checkpoints, expired claim attempts older than `lease_ttl * 100`, and stale compaction candidates whose checkpoint never became reachable from `CHECKPOINT_TIP`.

Eligible data objects:

```text
eligible(file) =
    not referenced by retained target manifests
    AND not referenced by in-flight mirror state
    AND object_mtime < now - retain_window_hours
    AND object path is under wal/, compacted/, or provider-specific temp upload prefix
```

Orphan detection is separate because it requires scanning the target prefix, not just manifests:

```text
s3e-mirror verify --config s3e-mirror.toml --find-orphans
s3e-mirror gc     --config s3e-mirror.toml --dry-run --include-orphans
```

For RockLake, the same rules apply to Parquet data files under the configured data prefix, except `excise` policy may intentionally retain data on the target when `preserve_excise = false`.

---

# Part X — Testing

## Unit Tests

All path-resolution, manifest-parsing, plan-diffing, and digest logic is deterministic and tested with property-based tests. We test that `plan_delta(M_n, M_{n+1})` is correct against synthesized manifests covering: WAL window expansion, WAL window contraction (compaction), L0 promotion, deep-level compaction, external_db addition, checkpoint addition/removal, V1↔V2 schema differences, and rewind detection.

Additional mandatory unit/property tests:

- `plan_bytes_are_order_independent`: randomized object-store listing order, worker order, hash-map insertion order, and event arrival order all produce identical `Plan` bytes.
- `manifest_translation_is_golden`: every corpus source manifest translates to expected target manifest bytes, with `target_manifest_digest` unchanged across platforms.
- `manifest_unknown_version_fails_closed`: source version above `manifest_max_version` fails before planning.
- `hlc_only_in_crdt_layer`: plan and target manifest bytes are unchanged when HLC values differ.
- `crdt_compaction_preserves_join`: `join(compact(S), Δ) == join(S, Δ)` for every CRDT type and randomized future delta.
- `checkpoint_tip_update_is_monotonic`: any interleaving of two compactors leaves `CHECKPOINT_TIP` pointing to the newest verified checkpoint or to an older safe checkpoint that will be advanced on the next run; never to invalid bytes.
- `coalescing_is_deterministic`: replicas with the same source/target/progress state make the same coalesce-vs-1:1 decision and produce the same `coalesced_source_range`.

## Integration Tests

Against the `object_store` `InMemory` and local-disk backends, we run end-to-end mirroring scenarios: cold start, steady-state tail, worker crash, coordinator crash, lease takeover, source-rewind, target-GC-collision, and dual-mirror conflict. Each scenario asserts both correctness (target opens cleanly with expected keys/values) and invariants (no missing files, no orphan files).

Integration tests also cover promotion state transitions: drain success, drain timeout with abort, verification failure with repair, concurrent promotion attempt, promoted-marker refusal on restart, and `verify-fences` rejecting a missing zero-byte WAL fence.

## Cross-Provider Tests

A CI matrix runs the same scenarios against real S3, Azure Blob, GCS, and MinIO. These are gated on credentials and run nightly.

The cross-provider matrix has a special conditional-write suite:

- `PutMode::Create` conflict returns the expected `AlreadyExists`/`Precondition` class and never overwrites.
- `PutMode::Update(UpdateVersion)` succeeds with the current version token and fails after a concurrent update.
- Multipart conditional completion conflict is retried correctly; on S3, the failed multipart is aborted and restarted.
- Existing-object same-bytes conflict is accepted only after byte and metadata comparison.
- Existing-object different-bytes conflict halts with determinism violation.

## DST (Deterministic Simulation Testing)

We integrate with SlateDB's own `slatedb-dst` harness: a seeded scheduler interleaves source writes, mirror worker steps, and injected faults (kill -9, network drop, clock jump, object-store throttle). Each seed must end in a state where the target opens to a manifest whose contents are a valid prefix of the source's history.

DST must include at least these schedules before Phase 6/7 features are enabled:

- two coordinators compute the same plan while source manifests continue to advance;
- two coordinators compute different bytes for the same target manifest ID (synthetic fault) and both halt;
- compactor writes checkpoint while a reader is mid-LIST and a worker writes new ack facts;
- event stream drops every manifest event for 10 minutes while polling remains healthy;
- source checkpoint TTL approaches expiry while workers are throttled;
- source GC boundary advances between file listing and boundary validation;
- promotion verify fails halfway through and then succeeds after repair;
- HLC clocks jump backward and forward while CRDT facts continue to merge.

## Schema-Compatibility Tests

For each SlateDB minor release, we run a compatibility suite: open a known-good source database, mirror it, and verify that opening the target with the same SlateDB version yields byte-identical query results. We also test against the previous and next SlateDB version to catch schema drift early.

For a source version newer than the mirror supports, the expected result is fail-closed with `s3e_mirror_manifest_schema_mismatch_total` incremented. There is no fallback translation mode.

## Chaos Drills

Periodic operational drills: pause workers for 10 minutes, kill the coordinator, induce target throttling, simulate region outage on source. The runbook is tested by running it.

---

# Part XI — How SlateDB Could Help

This section is a wish list directed at SlateDB upstream. None of the items below are blockers for shipping s3e-mirror, but each one removes either a category of duplicated code, a category of risk, or a category of polling cost. They are grouped by the goal they serve.

## Goal 1 — Reduce Mirror Latency

### 1. Public manifest-change subscription over IPC

SlateDB already has `Db::subscribe() -> watch::Receiver<DbStatus>` that fires on every manifest commit and durable-seq advance. Today it is only useful in-process. If SlateDB shipped a thin sidecar binary (or a documented IPC contract — a Unix socket emitting a versioned newline-delimited JSON stream of `{manifest_id, durable_seq, timestamp_ms}` events), then a co-located mirror coordinator could react in milliseconds to manifest commits without involving the object store at all. This collapses the cold-poll path entirely on the happy path and reduces lag from "seconds" to "single-digit milliseconds."

### 2. Manifest "tip pointer" file

Today, finding the current manifest requires listing the `manifest/` prefix and picking the highest. Even though SlateDB's `gc/` boundary files reduce the list cost, it is still a `LIST` per poll. A periodically-rewritten `manifest/CURRENT` file (a single small object containing the current manifest's filename) would reduce the read-cost of the steady-state poll loop from a `LIST` to a `GET` and would also reduce per-request cost. Race-conditions on the pointer are tolerable because the manifest itself is the source of truth — the pointer is only ever a hint that is verified by the actual manifest read.

### 3. Optional object-store event-notification adapter

SlateDB could ship a small adapter crate that translates AWS S3 EventBridge, Azure Event Grid, and GCS Pub/Sub notifications into the same `watch::Receiver<DbStatus>` stream that in-process callers see. This is a tiny amount of code and would unify the event-driven path in [§ Event-Driven Tail](#event-driven-tail) across providers without each downstream tool reinventing it.

### 4. Faster manifest publication knob

SlateDB's writer batches manifest updates for efficiency. Exposing a `manifest_min_publish_interval_ms` knob (already partially configurable internally) at the public API level would let operators trade write amplification for replication latency explicitly. A mirror tuned for low lag would request 100 ms publication; a mirror tuned for low cost would request several seconds.

## Goal 2 — Higher Throughput

### 5. Streaming WAL push API

A long-poll endpoint or an in-process `WalReader::stream()` that yields newly-flushed WAL SSTs the instant they become durable, without the mirror needing to list-and-diff, would remove the steady-state `LIST` on `wal/`. Combined with item 1, the WAL fast path becomes purely push-driven.

### 6. Bulk multi-file GET hint

Object stores do not generally support bulk GET, but `WalReader::list_with_metadata` returning byte ranges and pre-computed digests in one round-trip would let the mirror issue many parallel range-GETs without separate HEADs. Helpful when there are thousands of tiny WAL SSTs.

### 7. Per-SST content digest in the manifest

If the manifest carried a content digest (xxh3-128 or blake3-256) for every SST, the mirror would not have to compute its own digest in flight, and verification across providers becomes a metadata-only check. This is also valuable to SlateDB itself for detecting silent corruption on read.

### 8. Snapshot bundles

A `Db::export_snapshot(checkpoint_id) -> Stream<(Path, Bytes)>` API that internally walks the manifest and yields every referenced object in a single coherent stream, with a single end-to-end digest, would make cold-start mirroring a single operation rather than thousands of object copies orchestrated externally. Implementations could optimize this server-side (server-side copy within a provider).

### 9. Pre-packaged compaction outputs

The compactor already knows when it has written a new SST. A hook that pushes `(sst_path, size, digest)` to subscribers as compaction completes — same shape as the in-process subscribe — would let mirror workers prefetch compacted SSTs before they are referenced by the manifest, just as WAL prefetch already does for WAL files.

## Goal 3 — Cleaner Replication APIs

### 10. First-class replication slots

Today, the mirror manages "checkpoint that we promise to advance" entirely with `Admin::{create,refresh,delete}_detached_checkpoint` and bookkeeping in its own state. A first-class `ReplicationSlot { slot_id, holder_id, last_acked_manifest_id, last_acked_durable_seq, ttl }` concept in SlateDB itself would centralize this contract, would let SlateDB's own dashboards show "this database has 3 active replication consumers, the laggiest is at manifest 12345," and would prevent operator error (deleting the wrong checkpoint and stranding a mirror).

### 11. Replication snapshot API

```rust
impl Admin {
    pub fn create_replication_snapshot(opts) -> Result<ReplicationSnapshot>;
}

pub struct ReplicationSnapshot {
    pub manifest_id: u64,
    pub manifest_bytes: Bytes,         // verbatim FlatBuffer
    pub manifest_version: ManifestVersion,
    pub manifest_digest: Digest,
    pub wal_files: Vec<WalFileRef>,
    pub compacted_files: Vec<CompactedFileRef>,
    pub external_dbs: Vec<ResolvedExternalDb>,
    pub source_root: ResolvedRoot,
    pub source_wal_root: ResolvedRoot,
}
```

This is the API the mirror *wants* to call. It encapsulates everything in [§ Enumerating the Source Snapshot](#enumerating-the-source-snapshot) behind a stable contract and shields the mirror from internal changes.

### 12. Import-manifest API

```rust
impl Admin {
    pub fn import_manifest(bytes: Bytes, opts: ImportManifestOptions) -> Result<u64>;
}
```

This is the API the target side wants. Today we either need to reach into private types or duplicate the FlatBuffer codec. A public `import_manifest` with `expected_target_predecessor` semantics (commit only if target's tip matches the given ID) gives us the two-phase commit primitive we need without per-mirror reimplementation.

### 13. Public path and format crate

Split the path conventions, FlatBuffer schemas, and codec into a published, semver-stable `slatedb-format` crate. Mirrors and other replication tools can then depend on a small, stable surface without pulling in the entire database runtime.

### 14. Persisted object-store descriptors

Manifest V2 dropped `wal_object_store_uri`. That was the right decision for many reasons, but it leaves the mirror with no in-band way to describe "this database has a separate WAL store." A typed, redacted descriptor — schema like `{provider, region, bucket_or_container, prefix, requires_credentials: true}` with no secrets embedded — would let mirrors discover this from the source automatically rather than relying on operator-supplied config that can drift.

## Goal 4 — Verification and Safety

### 15. Mirror-aware GC handshake

A documented contract by which the source's GC defers to declared replication slots, and the target's GC defers to a declared mirror lease, would prevent the entire class of "GC raced the mirror" failures by construction. Today the mirror enforces this with operator discipline; making it a SlateDB-level contract would be much stronger.

### 16. Compaction-output visibility

`Compactor::recent_outputs() -> Vec<(Ulid, Size, Digest)>` would let a mirror's WAL-prefetch-style optimization extend to compacted SSTs immediately as they land, rather than waiting for the next manifest publication that references them.

### 17. Native cross-store clone with verification

Today `CloneBuilder` operates within one store. A cross-store variant that internally performed the equivalent of s3e-mirror's bulk copy with verification, exposed as a single SlateDB operation, would absorb the cold-start half of the mirror's job into the database itself and leave only the continuous-tail layer to external tools.

---

# Part XII — Roadmap

## Two Implementation Tracks

We ship in two tracks simultaneously:

- **Track A — Public-API-only** is the path that ships against released SlateDB versions without forking the crate. It uses `Admin`, `WalReader`, `VersionedManifest`, and direct `object_store` access, and reimplements path resolution and FlatBuffer round-tripping in our own `s3e-mirror-slate` crate against a vendored copy of the schemas. It is what lets us ship v1.
- **Track B — Upstream-API** lands the requests in [Part XI](#part-xi--how-slatedb-could-help) into SlateDB releases over time. As each lands, the corresponding code in `s3e-mirror-slate` is deleted in favor of the upstream call.

## Phases

- **Phase 0 — Upstream API requests.** File issues against SlateDB for items 1, 2, 7, 11, 12, 13, 15. These are conversations in parallel with implementation, not blockers.
- **Phase 1 — Read-only source enumeration.** Implement `s3e-mirror-slate` against the public APIs. Enumerate the source. Produce a plan in dry-run mode. No writes anywhere.
- **Phase 2 — Cold start to local target.** Implement workers, queue, and the bulk-copy path against an `InMemory` and local-disk target. End-to-end tests pass: source database opens, target database opens with identical keys.
- **Phase 3 — Continuous tail.** Implement the coordinator's polling loop, plan diffing, and incremental commit. Source-rewind detection. Lease and fencing.
- **Phase 4 — Cross-cloud.** Real S3 and Azure backends. WAL prefetch. Adaptive concurrency. Cost-model instrumentation.
- **Phase 5 — Event-driven tail + operations.** EventBridge/Event Grid/Pub-Sub integration. Promotion CLI. Mirror-aware GC. Verification levels. **Copy ledger** (OR-Map by `(file, target)`) and **anti-entropy repair** (worker pool drains repair observations) — see [ideas/crdts.md](ideas/crdts.md).
- **Phase 6 — Advanced.** Per-backend distributed multipart for S3. Provider-native fast paths fully wired. Pluggable digest algorithms. Snapshot-bundle ingest once SlateDB ships item 8. **Progress vectors** persisted as CRDT facts; **coordinator standby** that reconstructs state and dry-runs deterministic plans for fast failover.
- **Phase 7 — Multi-coordinator HA and multi-target fan-out.** Two or more coordinator replicas race to commit *identical* deterministic target manifest bytes; `PutMode::Create` picks the winner and same-bytes-on-conflict verification guarantees safety. One source checkpoint protects copies to multiple independent targets, each with its own target manifest commit and progress vector — enabled by the copy-ledger's `(file, target)` keying.
- **Phase 8 — Research.** Active-active RockLake (out of scope for the one-way mirror; requires upstream RockLake changes — globally unique snapshot/operation IDs, schema conflict policy). Tracked separately.

## Crate Layout

```
s3e-mirror-core      Plan/job types, digest, ObjectStore wiring, lease, fencing
s3e-mirror-slate     SlateDB layout: path resolution, manifest codec (vendored
                     from SlateDB schemas), enumeration, manifest translation
s3e-mirror-copy      Per-object copy primitive: streaming, multipart, digest,
                     provider-native fast paths, verification
s3e-mirror-queue     Object-store-backed queue: plans, jobs, claims, acks, recovery
s3e-mirror-coord     Coordinator: lease, source poll loop, source events,
                     snapshot planning, commit, GC handshake
s3e-mirror-worker    Worker: claim, copy, ack
s3e-mirror-cli       `s3e-mirror` binary with subcommands
```

---

# Part XIII — Open Questions

These are the questions we do not yet have a confident answer to. Each one is a design conversation we expect to have with the SlateDB maintainers, with our own users, and with ourselves once we have some operational experience.

1. **Manifest commit ordering vs source pace.** Default proposal is now explicit: mirror 1:1 while lag is below `coalesce_threshold_versions`; coalesce to the newest source manifest once lag exceeds the version or byte threshold. The remaining question is whether any users require *full historical 1:1 target history* even when badly behind.
2. **Long-term snapshot archival beyond default target retention.** The default retention is last 24 hours or 1024 manifests, whichever protects more. The open question is whether to add an explicit archive tier for old target manifests before mirror-aware GC removes their unreferenced files.
3. **Mirror-of-mirror.** Can a target serve as the source for *another* mirror? In principle yes — the target is a valid SlateDB database — but the chained replication slot semantics need more thought.
4. **Promotion symmetry.** When the target is promoted, do we expect operators to reverse the mirror automatically (so the old source becomes the new mirror's target), or is that always a manual decision? Default proposal: manual, with `s3e-mirror reverse` as a documented convenience.
5. **Tenancy.** Can one s3e-mirror coordinator manage multiple source/target pairs, or is the unit always one process per pair? Default proposal: one process per pair for v1, multi-tenant a Phase 7 concern.
6. **Network egress observability.** Should we ship an opinionated "this is what your cloud bill will look like next month" estimator? It is genuinely useful and operators will ask, but it is also a maintenance burden as cloud pricing drifts. Default proposal: ship a coarse model with caveats, link to the provider's calculator for ground truth.
7. **Schema-change notification from SlateDB upstream.** The mirror now fails closed on unknown manifest versions and carries a golden corpus. The open question is process: how do we get alerted within hours when SlateDB lands a manifest schema change so the corpus and vendored codec can be updated before users upgrade sources?
8. **CRDT compaction cadence.** Default proposal: **every 15 minutes**, plus on-demand when `s3e_mirror_crdt_state_object_count` exceeds 50k per type. Target invariant: `s3e_mirror_crdt_tail_object_count` stays below 5k under normal operation, so cold-restart tail LIST completes in under 2 seconds. See [§ CRDT State Cold-Restart Performance](#crdt-state-cold-restart-performance) and [ideas/crdts.md § Part VII Operational Concerns](ideas/crdts.md).
9. **IAM scoping policy.** Should the default deployment ship IAM templates that scope worker credentials to `mirror_state/jobs/claims/` + `jobs/acks/` + target object paths only, leaving manifest commits to coordinator credentials? Strongly leaning yes; documented in the security appendix of [ideas/crdts.md](ideas/crdts.md).

---

## Bottom Line

We can build a correct, fool-proof, cross-cloud SlateDB mirror against SlateDB v0.13.0 today, using only public APIs and a vendored copy of the manifest schema. The design above is conservative on safety — the two-phase rule and create-if-absent everywhere make worker crashes and lease takeover safe by construction — and aggressive on the operational details that catch real production mirrors out: source-side GC handshake, target-side GC prohibition, manifest-version skew, source rewinds, cross-region cost, adaptive concurrency, and event-driven freshness. The upstream wishlist in [Part XI](#part-xi--how-slatedb-could-help) is genuinely additive: each item makes the mirror smaller, faster, or safer, but none of them is required to ship.
