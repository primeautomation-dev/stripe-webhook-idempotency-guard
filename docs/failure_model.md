# Failure Model: Stripe Webhook Retries

This document describes the real-world failure modes that break naive
Stripe webhook implementations in production, especially those generated
by AI without retry awareness.

## Core Problem

Stripe delivers webhooks with an **at-least-once** guarantee.
This means the same event can be delivered multiple times due to:
- Network timeouts
- Partial failures
- Slow acknowledgements
- Process crashes

Naive implementations assume:
- One event → one execution
- Events arrive once and in order
- Returning HTTP 200 means the system is consistent

All of these assumptions are false in production.

## Critical Failure Scenarios

### 1. Duplicate Event Delivery
The same `event_id` is delivered multiple times.
If not handled correctly, this causes:
- Double charges
- Duplicate subscriptions
- Duplicate emails or side-effects

### 2. Retry After Partial Failure
A webhook handler crashes after committing some changes but before completion.
Stripe retries the event, causing:
- Inconsistent state
- Duplicate database writes
- Replayed side-effects

### 3. Concurrent Retries
Multiple retries of the same event are processed simultaneously.
Without atomic locking, this leads to:
- Race conditions
- Deadlocks
- Duplicate processing

## Why AI-Generated Code Fails Here

Most AI-generated webhook code:
- Treats webhooks as simple HTTP endpoints
- Lacks event-level idempotency
- Does not separate critical and best-effort operations
- Commits side-effects before establishing processing guarantees

## Design Requirement

A production-safe webhook handler must:
- Enforce idempotency **before** business logic
- Commit critical state changes atomically
- Safely ignore duplicate and concurrent deliveries
- Allow retries without reapplying business effects
