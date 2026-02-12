# Flash Sale Reservation System

## Context

You are designing a backend service for a flash sale where a limited number of items are sold under high demand. Many customers may attempt to purchase the same item at the same time, and the system must remain correct, fair, and resilient under concurrency.

The goal of this exercise is **not** to build a fully featured e-commerce platform. Instead, it is to demonstrate how you reason about system design, data modeling, concurrency, and asynchronous workflows in a realistic but tightly scoped problem.

You are encouraged to make reasonable assumptions and clearly document them.

---

## Technical Constraints

- The service must be implemented in **TypeScript using NestJS**.
  - You are free to choose the rest of the stack and the tooling.
- No frontend or UI is required.
- External systems such as payment providers or email services should be stubbed or simulated.

---

## Performance & Reliability Expectations

The system is expected to handle large bursts of concurrent requests during a flash sale event. The following expectations should inform your design decisions:

- API endpoints should respond **within predictable, known low-latency bounds**, even during peak concurrency.
- The system must **not degrade under contention**. Concurrent reservation attempts for the same item should not cause cascading delays, timeouts, or lock-related bottlenecks that affect unrelated requests.
- It is acceptable for certain downstream effects or read models to be **eventually consistent**, as long as correctness invariants (e.g., inventory never goes negative) are preserved.
- You can assume an ideal lag-free auto-scaler maintaining the service replicas based on the number of incoming requests.

These are not formal SLAs but design-level expectations. They should guide your architectural choices and trade-offs.

---

## High-Level Problem Description

The system manages the lifecycle of customer orders during a flash sale. Inventory is limited, and customers compete to reserve items. A reservation temporarily holds inventory for a customer while they attempt to complete payment.

Reservations may expire if payment is not completed in time, at which point inventory should be released and offered to the next eligible customer.

---

## Order and Reservation Lifecycle

An order in this system progresses through a series of well-defined states. You are free to name or model these states differently, but your design should clearly support the following behaviors.

A customer begins by attempting to reserve an item. If inventory is available, the system creates a reservation and places the order into a state where payment is expected within a fixed time window. While the reservation is active, the inventory is considered unavailable to other customers.

If inventory is not available at the time of the request, the order may be placed into a waiting or pending state. When inventory becomes available again, either due to a reservation expiring or being canceled, the next eligible order should be promoted and granted a reservation.

When a customer completes payment within the allowed time window, the order transitions into a completed state and the reservation becomes final.

If payment is not completed before the reservation expires, the system must cancel the reservation, release the inventory, and trigger the appropriate follow-up actions. These actions include promoting the next waiting order (if any) and generating a job or event to initiate a new payment attempt for the newly promoted customer.

The system must be robust to retries, duplicate events, and concurrent state transitions.

---

## Authentication

This service does not need to authenticate or authorize users. Assume that it’s a private service behind a public gateway. The gateway has already authenticated the user, and the identification parameters received by this service can be trusted.

---

## Payment and Timeouts

Payments are time-bound. Each reservation has a fixed expiration time, after which it is no longer valid.

If payment completes successfully before expiration, the order is finalized.

If payment does not complete before the timeout, the reservation must be canceled automatically. Inventory must be released, and the system must asynchronously trigger whatever work is required to promote the next order and generate a new payment opportunity.

---

## Deliverables

Please submit:

1. The source code for your service.
2. A README that explains:
   - The overall architecture and the API design
   - The order and reservation lifecycle
   - How concurrency and race conditions are handled
   - The trade-offs you made and alternatives you considered
