# ADR-001: Record-Locking Pattern

| Field     | Value                  |
|-----------|------------------------|
| Status    | Proposed               |
| Date      | 2026-05-14             |
| Deciders  | hatice                 |

---

## Context

Multiple clients or background workers may attempt to read and write the same record concurrently. Without a coordinated locking strategy, last-write-wins semantics can silently overwrite intermediate changes, causing data loss and inconsistency. A locking pattern must be chosen that is correct under concurrent access, performs acceptably at scale, and fits the constraints of the current infrastructure.

Two mainstream approaches were evaluated: optimistic locking (version field) and pessimistic locking (SELECT FOR UPDATE).

---

## Decision

Adopt **optimistic locking via a `version` integer field** on every mutable record.

Rules:

1. Every write request must supply the `version` value it read.
2. The UPDATE statement includes a `WHERE version = :expected_version` predicate and increments the field (`version = version + 1`) atomically.
3. If the predicate matches zero rows, the record was modified by another writer since it was last read; the application rejects the write with **HTTP 409 Conflict** and returns the current state so the caller can reconcile and retry.
4. A matched update is committed normally.

No database-level locks are held between the read and the write. Conflict detection happens at commit time.

---

## Alternatives considered

### Pessimistic locking (SELECT FOR UPDATE)

Acquires a row-level exclusive lock at read time and holds it until the transaction commits or rolls back.

Rejected because:

- **Connection pressure** — held locks require open transactions, which tie up database connections for the full read-write round trip, reducing concurrency under load.
- **Deadlock risk** — multiple transactions locking rows in different orders can deadlock; detection and retry logic adds complexity.
- **Latency coupling** — slow application code (validation, external calls) inside the transaction delays lock release and blocks other writers.
- **YAGNI** — write conflicts in this system are expected to be infrequent; paying the overhead of pessimistic locking is unnecessary at current scale.

---

## Consequences

### Positive

- No held locks between read and write; database connections are released promptly.
- Scales horizontally without coordination overhead.
- Conflict detection is explicit and surfaced to the caller, enabling informed retry or merge logic.
- Simple to implement: a single integer column and a predicate in the UPDATE.

### Negative

- Callers must handle 409 responses and implement retry or conflict-resolution logic.
- High-contention records (rare in this system) may experience repeated retries before a write succeeds.

### Risks

- If client code omits the `version` field from a write request, the guard is bypassed. Mitigation: enforce the field as required at the API schema layer and add a server-side check that rejects writes with a missing version.

---

## Principles applied

- **Optimistic Concurrency Control (OCC)** — assume conflicts are rare; detect and reject them at commit time rather than preventing them upfront.
- **YAGNI** — do not introduce pessimistic locking complexity until contention is observed and measured.
- **Explicit over implicit** — surface conflicts to callers as a first-class HTTP status (409) rather than silently retrying or overwriting.
- **Fail fast** — reject a stale write immediately rather than proceeding with potentially corrupted state.
