---
title: Distributed Lock-Based Scheduler Concurrency
---

# ADR-04: Distributed Lock-Based Scheduler Concurrency Control

> 🇰🇷 [한국어 버전](/ko/adr/collector/04-distributed-lock-scheduler-concurrency.md)

| Date       | Author       | Repos     |
| ---------- | ------------ | --------- |
| 2024-12-18 | @KubrickCode | collector |

## Context

### The Multi-Instance Problem

In blue-green deployments and horizontally scaled environments, multiple scheduler instances may run simultaneously during transitions:

```
Blue-Green Deployment:
[Blue v1.0]  ────────────────────────────────►  [Shutdown]
                    ↑ overlap period ↓
             [Green v1.1]  ────────────────────────────────►

During overlap: Both instances attempt scheduled jobs → Duplicate execution
```

**Without coordination:**

- Same repository analyzed multiple times
- Duplicate tasks enqueued to worker queue
- Wasted computational resources
- Potential data inconsistencies

### Requirements

1. **Single Execution Guarantee**: Only one scheduler instance executes a scheduled job at any time
2. **Automatic Recovery**: If active instance crashes, another can take over
3. **No Manual Intervention**: Dead instances should not require operator action
4. **Minimal Infrastructure**: Avoid additional dependencies beyond existing stack

## Decision

**Implement Redis-based distributed locking with UUID token ownership and heartbeat-based lease extension.**

Core components:

1. **SETNX-based Lock Acquisition**: Atomic compare-and-set for lock creation
2. **UUID Token Ownership**: Prevents incorrect release by expired holders
3. **Lua Script Atomicity**: Ensures check-and-modify operations are race-free
4. **Heartbeat Lease Extension**: Supports long-running jobs beyond initial TTL

## Options Considered

### Option A: Redis SETNX with Token (Selected)

**Mechanism:**

```
Acquire: SETNX key=lockKey value=UUID TTL=10min
Extend:  Lua(if GET(key)==UUID then EXPIRE(key, TTL))
Release: Lua(if GET(key)==UUID then DEL(key))
```

**Pros:**

- Single dependency (Redis already used for Asynq queue)
- Millisecond-level performance
- Token verification prevents accidental release
- Automatic expiration prevents dead locks
- No clock synchronization required

**Cons:**

- Single Redis instance is a single point of failure
- Requires careful TTL/heartbeat tuning

### Option B: Zookeeper/etcd Consensus

**Mechanism:**

- Create ephemeral sequential node for leader election
- Watch for node changes to detect leader failure

**Pros:**

- Strong consistency guarantees
- Built-in leader election primitives
- Handles network partitions gracefully

**Cons:**

- Additional infrastructure (separate cluster required)
- Operational complexity (quorum maintenance)
- Overkill for single-scheduler use case
- Higher latency than Redis

### Option C: Database Advisory Locks

**Mechanism:**

- PostgreSQL `pg_advisory_lock` or `SELECT FOR UPDATE`
- Poll periodically to check/acquire lock

**Pros:**

- Uses existing PostgreSQL infrastructure
- Transactional guarantees

**Cons:**

- Polling wastes database resources
- Not designed for long-held locks
- Potential for deadlock scenarios
- Connection-bound (lost on disconnect)

## Implementation Principles

### Token-Based Ownership

The critical insight: A simple `DEL key` after job completion is unsafe.

**Problem Scenario:**

```
[0:00] Instance A acquires lock (TTL=10min)
[0:10] TTL expires (A had long GC pause)
[0:11] Instance B acquires lock
[0:12] Instance A resumes, calls Release
[0:12] Instance A deletes Instance B's lock ← BUG
```

**Solution:**

```go
// Acquire: Store unique token as value
token := uuid.New().String()
ok := redis.SetNX(key, token, ttl)
if ok {
    self.token = token  // Remember ownership
}

// Release: Only delete if still owner
if redis.Get(key) == self.token {
    redis.Del(key)
}
```

### Atomic Operations via Lua Scripts

Checking ownership and modifying state must be atomic. Redis Lua scripts execute atomically:

```lua
-- Release: Conditional delete
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end

-- Extend: Conditional TTL refresh
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("expire", KEYS[1], ARGV[2])
else
    return 0
end
```

**Why not separate GET + DEL?**

```
[T1] Instance A: GET key → "token-a" (matches)
[T2] Instance A: (context switch / network delay)
[T3] TTL expires, Instance B acquires with "token-b"
[T4] Instance A: DEL key → Deletes B's lock ← BUG
```

### Heartbeat Strategy

For jobs that may exceed initial TTL, a background heartbeat extends the lease:

```
Configuration:
├── Lock TTL: 10 minutes
├── Heartbeat Interval: 3 minutes
└── Job Timeout: 5 minutes (soft limit)

Timeline (normal case):
[0:00] Lock acquired, TTL=10min, heartbeat goroutine starts
[0:03] Heartbeat → Extend TTL to 10min from now
[0:05] Job completes, Release lock
[0:05] Heartbeat goroutine exits

Timeline (long job):
[0:00] Lock acquired
[0:03] Heartbeat → Extend
[0:06] Heartbeat → Extend
[0:08] Job completes, Release lock

Timeline (crash recovery):
[0:00] Instance A acquires lock
[0:03] Instance A crashes (no more heartbeats)
[0:10] TTL expires, lock auto-released
[0:10] Instance B can now acquire
```

**Design Rationale:**

- `Heartbeat < TTL/3`: Ensures at least 2 extension attempts before expiration
- `TTL > Job Timeout`: Normal jobs complete before any risk of expiration
- Heartbeat only runs while job executes (not between scheduled runs)

### Graceful Degradation

When lock acquisition fails (Redis unavailable), the handler logs and skips:

```go
acquired, err := lock.TryAcquire(ctx)
if err != nil {
    slog.Error("lock acquisition failed", "error", err)
    return  // Skip this cycle, retry next schedule
}
if !acquired {
    slog.Debug("skipped: another instance is running")
    return  // Normal case during overlap
}
```

**Behavior:**

- Redis down: All instances skip (no job runs until Redis recovers)
- Lock held by another: Instance silently skips
- Both are INFO/DEBUG level (not ERROR for expected conditions)

## Consequences

### Positive

**Blue-Green Deployment Safety:**

- During rolling deployments, only one instance executes scheduled jobs
- No duplicate task enqueuing during transition periods
- Zero-downtime deployments remain safe

**Automatic Failure Recovery:**

- If active scheduler crashes, lock expires after TTL
- Standby instances automatically take over
- No manual intervention or monitoring alerts required

**Resource Efficiency:**

- Reuses existing Redis connection (shared with Asynq)
- No additional infrastructure provisioning
- Minimal memory/CPU overhead for lock operations

**Observability:**

- Lock acquisition/release logged at appropriate levels
- Easy to trace which instance is active via logs
- Lock state inspectable via Redis CLI if needed

### Negative

**Redis Dependency:**

- Redis unavailability blocks all scheduled job execution
- Single point of failure (unless Redis HA configured)
- Scheduler cannot fall back to "run anyway" mode

**Time-Based Guarantees:**

- Extreme GC pauses (>TTL) could theoretically cause dual execution
- Mitigated by heartbeat, but not mathematically impossible
- Acceptable trade-off for simplicity vs. consensus protocols

**Debugging Complexity:**

- Distributed locking issues harder to reproduce locally
- Requires Redis access to inspect lock state
- Timing-dependent bugs may be intermittent

### Technical Specifications

| Parameter          | Value                         | Rationale                                |
| ------------------ | ----------------------------- | ---------------------------------------- |
| Lock Key           | `scheduler:auto-refresh:lock` | Namespaced to avoid collision            |
| TTL                | 10 min                        | > Job timeout, allows heartbeat recovery |
| Heartbeat Interval | 3 min                         | < TTL/3 for safety margin                |
| Token Format       | UUID v4                       | Globally unique, no coordination needed  |

## References

- [ADR-01: Scheduled Re-collection](./01-scheduled-recollection.md) (Uses this lock mechanism)
- [ADR-03: Graceful Shutdown](./03-graceful-shutdown.md) (Lock release on shutdown)
- [ADR-07: Worker-Scheduler Separation](./07-worker-scheduler-separation.md) (Why scheduler needs lock)
- [Redis SETNX Documentation](https://redis.io/commands/setnx/)
- [Redlock Algorithm Discussion](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
