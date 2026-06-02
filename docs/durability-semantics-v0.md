---
title: TraceDB Durability Semantics v0
tags:
  - tracedb
  - durability
  - idempotency
  - wal
status: contract-freeze
type: durability-boundary
updated: 2026-06-02
---

# TraceDB Durability Semantics v0

This document defines the current durability boundary for TraceDB v0. It is
referenced by `docs/platform-contract-v0.md` and applies to the local-engine
path described by `docs/api/v1-http.md`. Managed-cloud durability semantics
are future work.

## Idempotency Keys

TraceDB supports `Idempotency-Key` on mutation, admin, and polymorphic native
operation routes. The idempotency authority is local-engine-only and scoped to
the same data directory.

### Behavior

- Same key plus same method, path, and raw body replays the first successful
  response.
- Same key with a different raw body returns `409 Conflict`.
- Replay survives a clean engine reopen from the same data directory.

### Routes That Accept Idempotency-Key

| Route | Category |
| --- | --- |
| `POST /v1/schema/apply` | mutation |
| `POST /v1/records/put` | mutation |
| `POST /v1/records/put-batch` | mutation |
| `POST /v1/records/patch` | mutation |
| `POST /v1/records/delete` | mutation |
| `POST /v1/traceql` (mutating commands) | polymorphic native |
| `POST /v1/graphql` (mutating root fields) | polymorphic native |
| `POST /v1/admin/compact` | admin |
| `POST /v1/admin/snapshot` | admin |
| `POST /v1/admin/restore` | admin |

## WAL Behavior

The Write-Ahead Log (WAL) is the primary durability mechanism. Every mutation
and schema change is appended to the WAL before it becomes visible to reads.

### Guarantees

- Writes are durable once the WAL entry is flushed to disk and the engine
  acknowledges the epoch.
- On a clean reopen from the same data directory, all committed WAL entries
  are replayed, restoring the last known state.
- Idempotency receipts are backed by WAL/checkpoint state and survive replays.

### Limitations

- Not cross-replica: this is single-instance local durability.
- Not crash-atomic exactly-once: a crash during WAL flush may leave the last
  entry in an indeterminate state (committed or lost, never partial).
- Not a managed-cloud exactly-once guarantee: this is local data-dir replay
  from WAL/checkpoint-backed idempotency receipts only.

## Snapshot Semantics

Snapshots are local filesystem operations controlled by the admin routes:

- `POST /v1/admin/snapshot` creates a point-in-time snapshot of the current
  data directory state at the given target path.
- `POST /v1/admin/restore` restores from a previously created snapshot into
  a target data directory, optionally verifying a specific record.

### Guarantees

- A snapshot captures all WAL-committed state visible at the time of the
  snapshot call.
- Restore produces a data directory that passes the same product-regression
  gate as the original.

### Limitations

- Snapshots are local filesystem artifacts. They are not encrypted at rest
  unless the data directory itself is TDE-protected.
- Cross-machine snapshot portability is not guaranteed; snapshots are tied
  to the engine version and platform that created them.
- No incremental snapshot or streaming backup protocol exists.

## Lock-File Semantics

The engine uses a lock file in the data directory to prevent concurrent access
from multiple engine instances. The durability-faults gate validates stale-lock
recovery behavior.

## TDE (Transparent Data Encryption)

When configured, TDE protects local artifacts (WAL, manifest, checkpoints,
snapshots) at rest. The current scope is `local_artifacts_when_configured`.
Key management and rotation are not yet part of the contract.

## Acknowledged Limitations

The following are explicitly **not** part of the v0 durability contract:

1. **Cross-replica replication.** No replica failover, leader election, or
   consensus protocol exists.
2. **Crash-atomic exactly-once semantics.** The idempotency boundary is
   replay-from-receipt, not hardware-level atomicity.
3. **Managed-cloud durability.** Cloud storage backends, geo-replication,
   and managed backup/restore are not implemented.
4. **Point-in-time recovery.** No WAL archiving or time-travel recovery
   beyond the last snapshot exists.
5. **Durable job API.** Admin jobs (compaction, indexing) are visible as
   queue state but do not have a durable job execution contract.

## Verification

The local durability evidence gate:

```bash
cargo run -p tracedb-cli -- durability-faults
```

Covers: wrong/missing master key, torn WAL tail, manifest/checkpoint
corruption, stale-lock recovery, encrypted snapshot restore, and WAL
idempotency replay after reopen. This is local durability evidence, not
managed-cloud backup/DR evidence.
