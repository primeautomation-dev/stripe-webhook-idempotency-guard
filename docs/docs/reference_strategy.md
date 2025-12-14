# Reference Strategy: Minimal Idempotency Guard

This document describes a minimal, production-safe strategy for handling
Stripe webhook retries and duplicates without introducing a full framework
or complex infrastructure.

## Core Principle

**Idempotency must be enforced before any business logic runs.**

Webhook handlers should be treated as distributed event processors,
not simple HTTP endpoints.

---

## High-Level Flow

Webhook Request
↓
Acquire Idempotency Lock (atomic)
↓
Execute Critical Business Logic (transaction)
↓
Mark Event Completed
↓
Return 200 OK

yaml
Kodu kopyala

Duplicate or concurrent deliveries must exit early without reapplying
business effects.

---

## Idempotency Lock

### Where It Lives
- A database table keyed by `event_id`
- Enforced via a UNIQUE constraint
- No cache-based locks (Redis TTL expiry can cause data corruption)

### Acquisition
- Attempt an atomic insert using `event_id`
- If the insert succeeds → processing continues
- If it fails:
  - If status is `completed` → return 200 immediately
  - If status is `processing` → return 200 and let the active worker finish

This guarantees only one worker processes the event.

---

## Transaction Boundary

All **critical state changes** must be executed in a single transaction:

- Subscription creation
- Payment recording
- User access updates
- Event completion marker

The event is marked as completed **inside the same transaction**
as the business logic.

This guarantees:
- No successful business logic without completion marking
- No completion marking without successful business logic

---

## Handling Partial Failures

Operations are split into two categories:

### Critical Operations
- Must be atomic
- Must be idempotent
- Must succeed exactly once

These run **inside the transaction**.

### Best-Effort Operations
- Emails
- Analytics
- Notifications

These run **after the transaction commits**.
Failures here must not cause webhook retries.

---

## Retry Behavior

- Failures before transaction commit → return non-200, Stripe retries
- Failures after commit → return 200, no retry
- Duplicate deliveries → return 200 immediately

Retries are safe because business effects are idempotent.

---

## Guarantees

This strategy guarantees:
- Exactly-once business effects
- Safe handling of retries and duplicate deliveries
- No duplicate charges or state corruption

It does **not** guarantee:
- Exactly-once execution of handler code
- Delivery order
- Guaranteed execution of best-effort operations

---

## Scope

This is intentionally a **reference implementation**:
- Not a framework
- Not a full payment system
- Not tied to a specific language or ORM

It is designed to be adapted to existing production systems.
