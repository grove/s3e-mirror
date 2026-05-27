# s3e-mirror

> A continuously-updated, byte-faithful copy of your [SlateDB](https://slatedb.io) database — in another bucket, another region, or another cloud.

**Project status:** Design / RFC. The system described here is being implemented in the open; this repository currently contains the design document and will gain code over the coming weeks. If you are looking for the deep technical design, it lives in [DESIGN.md](DESIGN.md). This README is for everyone else: a plain-English explanation of what we are building, why it matters, and what it does and does not do.

---

## What is this, in one sentence?

s3e-mirror keeps a working copy of your SlateDB database in a second object store — possibly a completely different cloud provider — so that if anything ever happens to the original, you already have a database you can read from and, if you choose, promote to be the new primary.

## Why would you want that?

Most modern databases that store their data in object storage — SlateDB included — are extraordinarily durable as long as the object store itself is healthy. Cloud object stores rarely lose bytes. But they can become *unavailable*, and unavailability is what actually hurts businesses. A region can go down. An account can be suspended for a billing dispute. A vendor can be subject to a regulatory action that affects your access. A misconfigured IAM policy can lock you out of your own data for hours. A ransomware incident can encrypt the wrong S3 bucket. None of these scenarios involve the object store "losing" your data, and yet in every one of them your database is effectively offline.

The standard answer to this problem is to keep a copy of your data somewhere else — somewhere with a different blast radius. A different region is a start; a different cloud provider is much better; a different account in a different cloud, with different administrative credentials, is better still. The trouble is that "keep a copy" is much harder than it sounds for a live database. The database is changing continuously. Files are being added, files are being rewritten in summary form, garbage collection is reclaiming files you might still need to copy, and the *meaning* of the database — which files together constitute "the current state" — is itself recorded in a file that is also being rewritten.

s3e-mirror exists to do that copying correctly, continuously, and without anyone having to think about it after it has been set up.

## How does it work, conceptually?

SlateDB has a property that makes correct mirroring possible at all: it never edits files in place. Every piece of data it writes to object storage is written once and never modified. New writes go into new files. When older files are consolidated, the consolidation produces new files; the originals stay there until garbage collection reclaims them. The database's "current state" is described by a small index file (the *manifest*) that itself is just one in a numbered sequence — manifest number 100 supersedes manifest number 99, but it does not overwrite it.

Because of this immutability, mirroring is fundamentally a *copy* problem rather than a *synchronization* problem. There is nothing to diff and nothing to merge. The mirror only ever needs to do two things: copy the data files that the source's current manifest refers to, and then, *only once that is done*, publish a translated copy of the manifest to the destination. The order matters enormously. If you publish the manifest first and the data files are not yet there, the destination is corrupt. If you publish the manifest last, the destination is always coherent — even if the mirror crashes in the middle.

That single ordering rule — "data first, then manifest" — is the heart of the design. Everything else is plumbing: how to discover what files need copying, how to coordinate work across many machines so the copying goes fast, how to avoid asking the source to slow down, how to react quickly when new data shows up, and how to make sure nothing in the system ever silently does the wrong thing.

## A picture that might help

Imagine the source database as a warehouse that only ever receives new boxes. Boxes are never opened and edited; they are stacked on top of existing boxes, and occasionally a forklift consolidates many small boxes into one large box. There is a clipboard at the front door listing exactly which boxes constitute the current official inventory. s3e-mirror is a fleet of small delivery vans that watches the clipboard, copies every new box to a second warehouse on the other side of the country, and only updates the clipboard at the destination warehouse once every box mentioned on the new version of the clipboard has physically arrived. If a van breaks down halfway through a delivery, the destination warehouse is unaffected: its clipboard still describes a coherent inventory, just a slightly older one. Another van will pick up the delivery and try again. Nothing ever leaves the destination warehouse in a half-updated state.

## What s3e-mirror is not

It is worth being clear about what this tool is not, because the temptation to assume it can do more is real.

**It is not two-way synchronization.** s3e-mirror copies from a source to a target. If you write to the target while the mirror is active, the mirror will refuse to continue, and rightly so — there is no sensible meaning of "merge these conflicting writes" for a database. If you want a writable target, you first stop the mirror, then open the target for writing. We provide a documented promotion procedure for exactly this.

**It is not a replacement for your primary database.** The target database is read-openable at any time — you can run analytics or backups against it — but it is intentionally not the active write path while mirroring is on.

**It is not a backup-rotation tool.** It maintains a single, continuously-updated copy. If you also want point-in-time snapshots for "what did the database look like at 3pm yesterday," that is a related but separate concern, and we have ideas about it in the design document.

**It is not magic.** Copying data across clouds costs money, mostly in egress charges from the source. The mirror is designed to make this cost predictable and to minimize it where possible (worker placement, server-side copy when source and target share a provider, batching), but the bytes still have to traverse the network. The [DESIGN.md](DESIGN.md) document includes a cost model section that estimates the bill.

## Who is this for?

The primary audience is operations and platform teams who are responsible for the resilience of a SlateDB-backed system and who have either been asked by their security or compliance team, or have decided themselves, that single-cloud durability is not the same as availability. Concretely:

- **Operations and SRE teams** running a SlateDB database in production who want a documented disaster-recovery story that does not rely on the goodwill of their primary cloud vendor.
- **Platform engineers** designing multi-region architectures who want a low-friction way to keep a database close to users in multiple regions for read traffic.
- **Compliance and security teams** facing requirements that data must be retrievable independently of any single vendor account.
- **Anyone** who has ever lost an afternoon to a region outage and decided, never again.

The tool is intentionally something you set up once and then mostly forget about. The day-to-day cost is the cloud egress bill and a small amount of object-store request volume. The day-of-disaster value is that you already have a database somewhere safe.

## What is the relationship to SlateDB?

s3e-mirror is an independent tool that builds *on top of* SlateDB. It uses SlateDB's public administrative APIs to read the source database safely, and it writes to the target using ordinary object-store operations. It does not require any patches to SlateDB to work; it does, however, identify a number of improvements to SlateDB that would make replication faster, lower-latency, and simpler — those proposals are in the design document and we intend to contribute them upstream over time.

If you are a SlateDB contributor or maintainer reading this, the [DESIGN.md](DESIGN.md) section *How SlateDB Could Help* is the part you will want to read; it is a focused wish list grouped by goal (lower latency, higher throughput, cleaner replication APIs, better safety).

## Project status and roadmap

We are early. This repository currently contains the design document and this README. The implementation will land in phases roughly matching the roadmap in DESIGN.md: read-only enumeration first, then cold-start copy, then continuous tail, then cross-cloud, then event-driven low-latency mode and operational tooling. Each phase will have tests and documentation before the next one begins.

If you are interested in this project, the most useful things you can do right now are: read the design document and tell us what is wrong with it; tell us about the disaster-recovery requirements your organization has that we are not addressing; and, if you work on SlateDB itself, weigh in on the upstream API proposals.

## How to follow along

This is a public repository. Watch it for releases, open issues for questions, and open pull requests for design feedback on [DESIGN.md](DESIGN.md). The design is intentionally a living document during this phase — substantial revisions are expected as we learn more.

## License

To be decided. Likely Apache-2.0 to match the SlateDB ecosystem.

## Further reading

- **[DESIGN.md](DESIGN.md)** — the full technical design, written to be read by engineers but with a substantial accessible introduction that does not require deep SlateDB knowledge.
- **[SlateDB documentation](https://slatedb.io/docs)** — for understanding the underlying database this tool mirrors.
