# YSQL authentication catalog follower reads

Implemented behind an experimental, default-false routing flag. Release build and
integration validation are described in [local reproduction](ysql-auth-follower-reads-local.md).
Production throughput, mixed-version, and sanitizer validation remain outstanding.

## Permanent operational restriction

**Reserving a universe for catalog follower reads permanently excludes PITR in
this release. Disabling follower-read routing does not undo the reservation.**

- Reservation requires no snapshot schedules, retained schedule snapshots, or
  schedule restorations. Deleted but not yet cleaned-up PITR state still blocks it.
- After reservation, creating PITR schedules and restoring the system catalog are
  prohibited across the universe, including YCQL schedules: the masters share one
  physical system-catalog tablet.
- Restart, leader change, and capability-AutoFlag demotion do not unlock PITR.
- There is no release/unreserve API or supported in-place conversion, with or
  without downtime. Do not opt in if this universe will need PITR later.
- Ordinary backup snapshots and restores that do not restore the system catalog
  remain available. Workflows that require PITR schedules are excluded.

The supported operator commands are:

```sh
yb-admin --master_addresses "$MASTERS" get_ysql_catalog_follower_read_reservation
yb-admin --master_addresses "$MASTERS" reserve_ysql_catalog_follower_reads \
  acknowledge_permanent_pitr_exclusion
```

The reservation command accepts exactly that one, case-sensitive acknowledgement
argument. Missing, different, or extra arguments are rejected before contacting
the cluster. The underlying `ReserveYsqlCatalogFollowerReads` leader RPC requires
`acknowledge_permanent_pitr_exclusion=true`. Repeating an acknowledged reservation
is idempotent; it neither enables routing nor changes an existing reservation.
A new universe is not reserved by default.

The read-only status command reports `reserved`, `reservation_pending`,
`pitr_admitted_in_term`, and `leader_term`. Pending reservation and PITR admission
describe only that term, not durable state. After an uncertain timeout, inspect
status and retry the acknowledged reservation. A leader change can resolve
transient uncertainty but cannot release a committed reservation.

## Why exclude restore

Normal MVCC writes preserve historical snapshots. System-catalog restore can
change the historical data visible to a read. Supporting both features requires
restore-overlap checks, durable read boundaries, recovery rules, and cross-replica
restore validation. This rollout instead prohibits the conflicting operation
before any follower reads are enabled.

## Admission and persistence

The reservation is a dedicated `SysConfigEntryPB` row in the replicated system
catalog. The snapshot coordinator loads and applies it on every master replica.
It is not part of the mutable cluster-configuration document, so unrelated admin
updates cannot clear it by omitting an unfamiliar field.

Immutability is enforced by the supported administrative APIs: the reservation
producer only writes true and there is no removal operation. Post-write callbacks
validate the stored form for diagnostics; they do not veto storage writes.
Direct system-catalog edits in emergency repair mode are outside this contract
and are not a supported way to release the reservation.

PITR admission and reservation use the snapshot coordinator's mutex. The first
admitted side blocks the conflicting side for that leader term. The durable
reservation continues to block PITR in later terms.

A client timeout does not cancel a Raft write. A timed-out or failed admission
therefore remains conservative: the conflicting mode may remain unavailable until
a new leader catches up and resolves the durable state. The new leader must be
ready before handling requests; stale leader terms cannot reset admission state.
This covers requests paused before submission as well as writes awaiting apply.

## Fresh authentication snapshot and routing

`ysql_enable_auth_catalog_follower_reads` defaults to false in masters, tservers,
and PostgreSQL/pggate. Reservation and capability promotion do not change this
flag. Enable it only after the reservation is committed and all peers are
compatible; enabling it earlier can fail eligible authentication startup.

For an eligible uncached authentication attempt:

1. PostgreSQL resets prefetch state, then asks the master leader for one fresh
   snapshot through `GetYsqlAuthCatalogReadTime`. The leader checks its routing
   flag and durable reservation, incorporates propagated hybrid time, selects
   `T = MaxGlobalNow()`, waits for lease-backed safe time at least T, and rechecks
   the leader term. A lagging safe-time value alone is not a fresh read boundary.
2. Pggate retains `ReadHybridTime::SingleTime(T)` through authentication, database
   CONNECT checks, and role settings, across individual prefetch lifetimes. The
   tserver validates the uncached, nontransactional catalog-session envelope and
   rejects writes, row locking, transactional state, and conflicting paging times.
3. Only allowlisted full catalog scans and their matching indexes are tagged for
   follower routing: current-version `template1` IDs for `pg_authid`,
   `pg_database`, `pg_auth_members`, `pg_db_role_setting`,
   `pg_yb_catalog_version`, and `pg_yb_logical_client_version`. Predicates, server
   expressions, sampling, backfill, vector requests, and unrelated relations do
   not pass the follower validator. Other permitted pure catalog reads stay
   STRONG at the same T; they do not broaden follower admission.
4. The client prefers a known nonleader, distributing requests among equally
   local followers. A serving master independently validates the strict tagged
   envelope, initialization, non-shell state, locally applied reservation, and
   safe time at least T. A tagged `CONSISTENT_PREFIX` read also requires its
   routing flag. Untagged master reads remain leader-only.
5. Follower safe-time wait is bounded by 1000 ms and half the remaining RPC
   deadline. The client bounds the follower RPC by 2000 ms and half its remaining
   budget, including unreachable followers. These are internal limits, not flags.
   Retry through leader selection explicitly switches to STRONG and waits under
   the leader lease at the same T. Neither fallback nor paging chooses a new
   snapshot. Conflicting restart/response read times fail rather than silently
   advancing T.

The snapshot ends explicitly after CONNECT checks and `pg_db_role_setting`, before
ordinary session initialization can execute SQL. Catalog-version invalidation is
deferred while T is pinned and resumes afterward. Snapshot acquisition failure is
an authentication-startup error, not permission to use an older snapshot.

### Bounded acquisition and latency

`GetYsqlAuthCatalogReadTime` runs on dedicated, on-demand worker pools rather than
holding general RPC workers during the clock/lease wait. Admission includes queued
and running tasks; excess requests receive `ServiceUnavailable`. Queue plus execution
is bounded by the RPC deadline and an internal 5000 ms budget. Expiration returns
`TimedOut`; shutdown rejects queued tasks and joins running tasks.

| Internal limit | Master | Tserver |
| --- | --- | --- |
| Workers | 32 | 16 |
| Running plus queued tasks | 128 | 64 |
| Queue plus execution budget | 5000 ms | 5000 ms |

These limits bound resource use, not guaranteed capacity. The routing switch is
the only new operator flag; deterministic tests configure limits through sync
points rather than expanding the public tuning surface.
Selecting `MaxGlobalNow()` can add approximately the clock-skew bound to each
eligible login (500 ms with the default wall clock). Freshness is not traded away
to avoid that wait. Measure connection latency, overload, and leader row/byte work
before enabling this experimentally; no throughput improvement is established.

## Excluded paths and cache behavior

The PostgreSQL entry point is restricted to direct TCP client backends. Unix
sockets, connection-manager authentication/physical/control/passthrough backends,
internal/background workers, autovacuum, initdb, and binary-upgrade paths retain
their existing behavior. Login profiles are excluded whenever
`ysql_enable_profile=true`, even before profile catalogs exist, because
authentication can write profile state.

Response-cache selection takes precedence over follower routing. Every attempt
that selects the authentication response cache, including misses and retries,
stays on the existing cache/leader path. Shared-cache freshness semantics are not
changed by this feature. To exercise follower routing locally, explicitly set
`ysql_enable_read_request_cache_for_connection_auth=false`; do not disable global
`ysql_enable_read_request_caching`. Forwarded PostgreSQL gflags are captured when
the postmaster starts, so changing only the live tserver flag is insufficient.

Later phase-3 prefetches within an opted-in attempt also bypass the shared response
cache, because its data may come from another snapshot. Per-attempt PostgreSQL
prefetch remains enabled. This can increase leader work for non-allowlisted catalogs,
so include warm/cold relcache and broad-preload configurations in performance tests.

## Observability and validation

Master metrics distinguish `ysql_auth_catalog_snapshot_acquisitions`,
`ysql_auth_catalog_follower_reads`, and `ysql_auth_catalog_leader_reads`. Read
counters increment after envelope, reservation, and safe-time checks, classified
by the actual serving replica's role. They count admissions, not completed storage
reads or successful logins. A wrong password can still cause a read admission.

The metric prefixes `master_ysql_auth_snapshot_` and `tserver_ysql_auth_snapshot_`
expose `task_limit_rejections`, `deadline_expirations`, and `outstanding_tasks`.
Monitor both layers: a full tserver queue can reject logins before the master sees
any acquisition. Read-admission counters alone do not show this overload or prove
reduced leader load.

Validation must cover CLI acknowledgement and idempotency, both PITR/reservation
admission orderings and timeouts, persistence through replay/restart/failover,
retained PITR state, and ordinary backup availability. Read-path tests must cover
fresh role/password changes, follower admissions, fixed-T leader fallback,
malformed envelopes, lag/timeout handling, cache/profile exclusions, direct-TCP
scope, and default-off behavior. The integration target is
`pg_auth_follower_reads-test`; commands are in the local reproduction document.
No performance improvement is claimed.

## Upgrade/Rollback safety

`ysql_enable_catalog_follower_read_reservation` is a `kLocalPersisted` AutoFlag,
initially false and targeting true. Promotion permits reservations; it does not
reserve a universe or enable routing. Install compatible binaries on every master
peer and participating tserver/PostgreSQL process before promotion or reservation.
Promote the capability cluster-wide through AutoFlags; promotion alone is not
proof that every master is compatible. Do not substitute per-process capability
overrides or attempt activation during a mixed-version upgrade.

The operational states differ:

- Unpromoted and unreserved: the capability is unavailable; PITR is unchanged.
- Promoted but unreserved: PITR is unchanged, but old binaries can reject the
  unknown promoted flag at startup. Explicitly demote it before older-binary
  rollback. Generic AutoFlag rollback does not roll back persisted-class flags.
- Reserved: demotion stops new reservations but does not release this universe.
  **Downgrade to binaries that do not enforce the reservation is not supported.**
  Those binaries could admit PITR. Disabling routing or demoting the capability
  is not a conversion or rollback procedure.
