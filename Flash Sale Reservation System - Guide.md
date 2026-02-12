# Assessment & Evaluation Guide (Internal)

## Purpose of This Exercise

This take-home assignment evaluates a senior backend engineer's ability to design **correct, scalable, and resilient backend systems** under high concurrency.

The handout **does not explicitly prescribe** an event-driven or eventually consistent architecture. Instead, the **Performance & Reliability Expectations** section establishes constraints — low-latency p99 targets, non-blocking request handling, and resilience under contention — that **strongly imply** one. A strong candidate should recognize that meeting these expectations naturally leads to asynchronous, event-driven design. Arriving at this architecture through reasoning (rather than being told) is itself a signal of senior-level design judgment.

The scope is intentionally constrained. Candidates are expected to demonstrate judgment, not completeness.

This guide outlines what interviewers should look for when reviewing submissions and discussing trade-offs with candidates.

---

## Key Scope Clarification: Authentication & Trust Boundary

Per the candidate handout, **authentication and authorization are explicitly out of scope**. The service is assumed to be a **private internal service** sitting behind a trusted gateway that already authenticates users and injects verified identity information into requests.

### What This Means for Evaluation

You are **not** evaluating:

* Login flows
* Token issuance or verification
* Permission models
* Identity persistence

You **are** evaluating:

* Whether the candidate correctly models and respects the trust boundary
* Whether identity is treated as an input, not a responsibility
* Whether the design avoids leaking auth concerns into core domain logic

🚩 **Red flag:** Reintroducing auth, JWT validation, or user management
🚩 **Positive signal:** Explicitly stating assumptions about trusted upstream identity

---

## Core Architectural Signals

### 1. SLA Awareness and Architectural Reasoning

The handout's **Performance & Reliability Expectations** are the primary design driver. They are intentionally abstract — no specific technology is prescribed — but the constraints make it very difficult to satisfy them without event-driven, eventually consistent architecture.

Interviewers should evaluate whether the candidate:

* **Connects the SLAs to their architecture.** A strong candidate will explicitly reason: "To meet p99 < 200ms under high concurrency, I cannot block API requests on downstream processing or DB lock contention, so I need to decouple writes from side effects using events/queues."
* **Separates the acknowledge path from the processing path.** The API should accept the user's intent (e.g., reserve an item) and return quickly, while downstream effects (promotion, notifications, payment link generation) happen asynchronously.
* **Identifies contention as the primary latency risk.** Under flash sale load, naive designs (e.g., row-level locks held during synchronous orchestration) cause lock contention that cascades into p99 spikes and timeouts.

🚩 **Red flag:** No mention of the SLA constraints, or a design that cannot plausibly meet them
🚩 **Red flag:** Synchronous request-response flows that perform multi-step orchestration before responding
✅ **Strong signal:** Explicit reasoning from SLA constraints to architectural decisions
✅ **Strong signal:** Clear separation between the "fast path" (acknowledge + persist) and "slow path" (async processing)

---

### 2. Event-Driven and Eventually Consistent by Design

Given the performance constraints in the handout, an **event-driven, eventually consistent architecture** is the expected outcome — even though it is not explicitly required.

Strong candidates will:

* Represent meaningful state transitions as events
* Avoid synchronous orchestration across multiple subsystems
* Accept that read models may lag behind writes
* Design idempotent consumers and handlers

🚩 **Red flag:** Transaction scripts coordinating multiple subsystems synchronously
🚩 **Red flag:** Expecting strong consistency across async boundaries

---

## 3. Queue vs Stream: Correct Tooling for the Job

A central evaluation axis is whether the candidate can distinguish between **job queues** and **event streams**, and apply them appropriately.

### Streams (e.g. Kafka)

Expected use cases:

* Order lifecycle transitions
* Reservation creation, expiration, cancellation
* Inventory state changes

These represent **immutable facts** about the system.

### Queues (e.g. SQS, RabbitMQ)

Expected use cases:

* Time-based actions (reservation expiration)
* Retryable tasks
* Payment-timeout enforcement
* Generating payment-link jobs

These represent **commands or work to be done**.

🚩 **Red flag:** Using a stream as a delayed job system
🚩 **Red flag:** Modeling mutable state inside a queue
✅ **Strong signal:** Clear explanation of why each async mechanism exists

---

## 4. CDC Preferred Over Application-Level Outbox

When a relational database is used as the system of record, **Change Data Capture (CDC)** is preferred over manual outbox tables.

Strong candidates will:

* Treat the database as the source of truth
* Emit domain events via CDC (e.g. Debezium)
* Avoid dual-write consistency problems
* Understand ordering and replay semantics

Reference:
[https://docs.confluent.io/cloud/current/connectors/cc-postgresql-cdc-source-v2-debezium/cc-postgresql-cdc-source-v2-debezium.html](https://docs.confluent.io/cloud/current/connectors/cc-postgresql-cdc-source-v2-debezium/cc-postgresql-cdc-source-v2-debezium.html)

🚩 **Red flag:** Outbox pattern with no acknowledgment of trade-offs
🟡 **Acceptable:** Outbox used deliberately with a clear justification

---

## 5. Order & Reservation Lifecycle Modeling

The lifecycle described in the handout is intentionally rich and stateful.

Interviewers should look for:

* Explicit order and reservation states
* Clear transition rules
* Well-defined invariants (e.g. inventory never goes negative)
* Safe handling of duplicate or late events

Strong designs will:

* Use state machines or explicit status transitions
* Be resilient to retries and replay
* Avoid "check-then-act" race conditions

🚩 **Red flag:** Implicit lifecycle hidden in application logic
🚩 **Red flag:** No strategy for duplicate events or retries

---

## 6. High-Concurrency Inventory Management

This is one of the most important evaluation dimensions.

Candidates should demonstrate:

* Awareness of race conditions under load
* Atomic inventory allocation strategies
* Correct promotion of waiting orders
* Idempotent reservation creation

Look for discussion of:

* Optimistic locking or conditional updates
* Deterministic ordering for promotion
* Avoidance of in-memory counters as source of truth

🚩 **Red flag:** "We just decrement inventory in memory"
🚩 **Red flag:** Serializing everything behind a global lock

---

## 7. Payment Timeout Handling (Follow-Up Exercise)

Although full timeout enforcement may be discussed in an in-person follow-up, the take-home design should clearly *support* it.

Strong candidates will:

* Associate expiration timestamps with reservations
* Trigger expiration asynchronously
* Cancel reservations deterministically
* Promote the next eligible order
* Emit events or enqueue jobs for downstream processing

🚩 **Red flag:** Cron jobs scanning the entire table
✅ **Strong signal:** Per-reservation delayed jobs or indexed expiration handling

---

## 8. NoSQL and Alternative Data Stores

NoSQL data stores are acceptable **if and only if** the candidate can clearly justify their choice.

You should expect answers to:

* How ordering guarantees are preserved
* How concurrency constraints are enforced
* How eventual consistency is handled
* How events are produced reliably

🚩 **Red flag:** "NoSQL is faster" without deeper reasoning
✅ **Strong signal:** Conditional writes, versioning, or transactional semantics

---

## What to Prioritize in Evaluation

| Dimension      | What You're Assessing                                         |
| -------------- | ------------------------------------------------------------- |
| SLA reasoning  | Connects performance constraints to architectural choices     |
| Architecture   | Separation of concerns, async boundaries                      |
| Data modeling  | Explicit lifecycle, invariants                                |
| Concurrency    | Correctness under contention without blocking the API         |
| Async design   | Proper use of queues vs streams                               |
| Assumptions    | Clear trust boundary and scope control                        |
| Communication  | Ability to explain trade-offs clearly                         |

---

## Final Interviewer Note

This exercise is most valuable when followed by a **design discussion**, not a code review.

The strongest candidates will:

* Explain what they deliberately did *not* build
* Be honest about trade-offs
* Show comfort with ambiguity
* Treat eventual consistency as a feature, not a bug
* Trace their architecture back to the performance expectations in the handout
