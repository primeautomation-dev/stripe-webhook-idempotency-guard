# Stripe Webhook Idempotency Guard

A minimal, production-safe reference implementation for handling Stripe webhooks
without duplicate charges or side-effects caused by retries and partial failures.

This project demonstrates why naive, AI-generated payment code often breaks in
production — and how to fix it with proper idempotency and transaction boundaries.

## The Problem

Stripe webhooks are delivered with an at-least-once guarantee.
Naive implementations — especially AI-generated ones — assume a single delivery
and often cause duplicate charges, duplicate emails, or inconsistent state
when retries or partial failures occur.

## The Approach

This repository uses event-level idempotency enforced at the database layer.
Each webhook event is locked and processed exactly once at the business level,
with all critical state changes committed atomically before acknowledging Stripe.


## Guarantees

- Exactly-once business effects (no duplicate charges or state changes)
- Safe handling of retries and concurrent deliveries
- Clear separation between critical and best-effort operations

Non-goals:
- Not a framework
- Not a full payment system
- Not a SaaS product

