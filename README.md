# Flash-Sale E-commerce System: SRS & Engineering Guide

---

## 1. Purpose
Build a microservice backend that can handle **flash-sale traffic spikes** while ensuring:
* **No overselling**
* **Reliable order processing** even with retries/failures
* **Async processing** via RabbitMQ + workers
* **Observability** (metrics/logs/tracing)
* **Autoscaling** APIs + workers as traffic grows

---

## 2. Scope

### In-scope
* Limited stock product “Drop” (flash sale)
* High traffic “place order” endpoint
* Order lifecycle (pending → reserved → paid → completed / cancelled)
* Inventory reservation with TTL
* Asynchronous payment simulation
* Notification (email mock)
* Analytics events (eventually consistent)
* Monitoring and autoscaling

### Out-of-scope (for demo)
* Real payment gateway integration (use mock)
* Real shipping carrier integration
* Full admin UI (use basic APIs)
* Multi-warehouse inventory

---

## 3. System Actors
* **Customer**: browses product, places order, checks status
* **Admin**: creates drop, sets stock, monitors sale
* **System Workers**: process inventory/payment/notifications asynchronously

---

## 4. High-Level Architecture

### 4.1 Services (Microservices)
1.  **API Gateway / BFF**
    * Auth validation, request routing, rate limiting, request IDs
2.  **Auth Service**
    * JWT issuance/validation (simple)
3.  **Catalog Service**
    * Product info, drop schedule
4.  **Order Service (Saga Orchestrator)**
    * Owns order state machine
    * Publishes events via outbox
5.  **Inventory Service**
    * Owns stock and reservations (strong consistency)
6.  **Payment Service (Mock async)**
    * Simulate success/failure/latency
7.  **Notification Service**
    * Send email/SMS mock
8.  **Analytics Service**
    * Consume events, aggregate metrics

### 4.2 Data Ownership
* Each service has its **own database** (no shared DB).
* Only communicate via **HTTP/gRPC** + **RabbitMQ events**.

### 4.3 Message Broker
* **RabbitMQ** for:
  * Event-driven flows
  * Load buffering during spikes
  * Worker fan-out

---

## 5. Functional Requirements

### 5.1 Catalog
* **FR-C1**: Admin can create/update a product
* **FR-C2**: Admin can schedule a flash drop (start/end time)
* **FR-C3**: Customer can view product + drop timer

### 5.2 Orders
* **FR-O1**: Customer can place an order for product quantity (default 1)
* **FR-O2**: System must prevent overselling
* **FR-O3**: Customer can query order status
* **FR-O4**: Orders must be idempotent (client retry safe)
* **FR-O5**: Order states:
  * `PENDING` → `RESERVE_REQUESTED` → `RESERVED` → `PAYMENT_REQUESTED` → `PAID` → `COMPLETED`
  * **Failure paths**:
    * `RESERVE_FAILED` → `CANCELLED`
    * `PAYMENT_FAILED` → `CANCELLED` (+ release reservation)

### 5.3 Inventory
* **FR-I1**: Inventory must support **reservation with TTL** (e.g., 2 minutes)
* **FR-I2**: Reservation must be idempotent (same request processed twice = same outcome)
* **FR-I3**: Inventory “commit” happens only after payment success
* **FR-I4**: Expired reservations are released by a worker

### 5.4 Payment (Mock)
* **FR-P1**: Accept payment capture requests asynchronously
* **FR-P2**: Simulate failures (e.g., 5–15%), random latency
* **FR-P3**: Emit `payment.succeeded` or `payment.failed`

### 5.5 Notification
* **FR-N1**: Send notification on order completion/cancellation
* **FR-N2**: Must be async, retriable, DLQ on failure

### 5.6 Analytics
* **FR-A1**: Consume order/payment events
* **FR-A2**: Store aggregates (orders/min, success rate, failures)
* **FR-A3**: Eventual consistency is acceptable

---

## 6. Non-Functional Requirements (Real-World Goals)

### 6.1 Performance / Capacity Targets (demo but realistic)
* **NFR-P1**: Handle **peak 1,000–5,000 req/s** on `POST /orders` in local/perf environment (k6)
* **NFR-P2**: P95 latency for `POST /orders` under load: **< 200ms** (because it only enqueues work + minimal DB)
* **NFR-P3**: Redirect reads (GET product/order status) should remain stable under burst

### 6.2 Reliability
* **NFR-R1**: **No oversell** (hard requirement)
* **NFR-R2**: At-least-once delivery tolerated; consumers must be idempotent
* **NFR-R3**: Support retries with backoff + DLQ for poison messages
* **NFR-R4**: Outbox pattern to guarantee “DB write + event publish” reliability

### 6.3 Consistency Model
* Orders/inventory/payment are coordinated via **Saga**
* Inventory stock is strongly consistent **within Inventory Service**
* Analytics is **eventually consistent**

### 6.4 Security
* JWT-based auth
* Rate limiting on order placement
* Audit logs for admin actions

### 6.5 Observability (must-have)
* **Metrics**: Prometheus
* **Dashboards**: Grafana
* **Tracing**: OpenTelemetry + Jaeger
* **Logs**: structured JSON logs with correlation IDs

---

## 7. Core Flows

### 7.1 Happy Path: Place Order
1. Client → `POST /orders` (idempotency key)
2. **Order Service**:
   * Create order `PENDING`
   * Publish `inventory.reserve.requested` (via outbox)
3. **Inventory worker**:
   * Reserve stock (atomic)
   * Publish `inventory.reserved`
4. **Order Service**:
   * Set status `RESERVED`
   * Publish `payment.capture.requested`
5. **Payment worker**:
   * Simulate capture
   * Publish `payment.succeeded`
6. **Order Service**:
   * Mark `PAID` → `COMPLETED`
   * Publish `notification.send` + analytics events
7. **Notification worker** sends message

### 7.2 Failure Path: Payment Failed
* Payment emits `payment.failed`
* Order Service marks `CANCELLED`
* Publish `inventory.release.requested`

### 7.3 Reservation Expiry
* Inventory has TTL on reservations
* A scheduled worker releases expired reservations
* Emits `inventory.reservation.expired` (optional)

---

## 8. API Specification (minimal set)

### Gateway / BFF
* `POST /auth/login`
* `GET /products/:id`
* `POST /orders`
  * Headers: `Idempotency-Key`
  * Body: `{ productId, quantity }`
* `GET /orders/:id`

### Admin
* `POST /admin/products`
* `POST /admin/drops`
* `GET /admin/metrics` (optional shortcut, real metrics is Grafana)

---

## 9. Messaging Design (RabbitMQ)

### Exchanges/Queues
Use a topic exchange: `ecom.events`

**Routing keys (examples):**
* `order.created`
* `inventory.reserve.requested`
* `inventory.reserved`
* `inventory.rejected`
* `inventory.release.requested`
* `payment.capture.requested`
* `payment.succeeded`
* `payment.failed`
* `notification.send`
* `analytics.event`

**Rules:**
* Every message includes:
  * `eventId` (UUID)
  * `correlationId` (trace)
  * `idempotencyKey` or `requestId`
  * `occurredAt`
  * `schemaVersion`

**DLQ:**
* For each queue, configure `.dlq` + retry strategy (x-death / delayed exchange)

---

## 10. Data Model (important parts)

### Order DB (Order Service)
* `orders(id, userId, productId, qty, status, totalAmount, createdAt, updatedAt)`
* `order_events(id, orderId, type, payload, createdAt)` (optional)
* `outbox(id, aggregateId, eventType, payload, status, createdAt)` ✅

### Inventory DB
* `stock(productId, available, reserved, updatedAt)`
* `reservations(id, orderId, productId, qty, status, expiresAt, createdAt)`
  * Unique constraint on `orderId` to enforce idempotency

### Payment DB (optional)
* `payments(id, orderId, status, amount, createdAt)`

---

## 11. Implementation Plan (phased, “real-world”)

### Phase 0 — Foundations
**Goal:** local dev environment + basic service skeleton
* Docker Compose: RabbitMQ + Postgres per service (or schemas)
* Common libraries:
  * logging (JSON)
  * correlationId middleware
  * config (env-based)
* API Gateway routes to services
* **Done when:** all services start + health checks work

### Phase 1 — “Hot Path” Order Creation (fast + safe)
**Goal:** `POST /orders` is fast and idempotent
* Implement Order Service:
  * Validate drop window + basic checks
  * Save order `PENDING`
  * Write outbox event `inventory.reserve.requested`
* Implement Outbox Publisher worker (within Order service)
* **Done when:** you can create orders and see messages reliably published

### Phase 2 — Inventory Reservation (no oversell)
**Goal:** guarantee stock correctness
* Inventory Service:
  * Atomic reserve:
    * If `available >= qty` then decrement available, create reservation
  * Idempotency on `orderId`
* Inventory publishes `inventory.reserved` / `inventory.rejected`
* **Done when:** load test shows **0 oversells**

### Phase 3 — Saga Orchestration + Payment Async
**Goal:** complete/ cancel orders through events
* Order consumes inventory events
* Payment Service consumes capture requests and emits result
* Order handles payment result + triggers compensation
* **Done when:** orders complete end-to-end and failures compensate correctly

### Phase 4 — Retries, DLQ, Poison Handling
**Goal:** production-grade message processing
* Retry with backoff (delayed exchange or retry queues)
* DLQ + alerting dashboard
* Consumer idempotency for every event
* **Done when:** killing workers / duplicate messages doesn’t corrupt state

### Phase 5 — Monitoring + Tracing
**Goal:** you can *see* what’s happening
* Prometheus metrics:
  * request latency, error rate
  * queue depth, consumer lag
  * saga step durations
* Grafana dashboards
* OpenTelemetry traces across gateway → services → worker handlers
* **Done when:** you can trace one order across all services

### Phase 6 — Autoscaling (K8s recommended)
**Goal:** scale APIs + workers when traffic increases
* Deploy to Kubernetes
* HPA for API services (CPU/RPS)
* KEDA for RabbitMQ consumers (queue length)
* **Done when:** increasing k6 load causes worker replicas to grow, queue lag stays controlled

---

## 12. “Real-World” Goals Checklist
✅ No overselling under burst traffic
✅ Idempotency on:
* `POST /orders`
* reservation
* payment result handling
✅ Outbox pattern implemented
✅ DLQ + retries + dashboards
✅ Correlation IDs + distributed tracing
✅ Autoscaling for both API and workers
✅ Clear SLOs:
* order API p95 latency
* queue lag thresholds
* failure rate alerts

---

## 13. Suggested Tech Stack (simple + common)
* **Services**: Java/Spring Boot **or** Node/NestJS **or** Go/Fiber
* **DB**: Postgres
* **Broker**: RabbitMQ
* **Observability**: Prometheus + Grafana + Jaeger (OpenTelemetry)
* **Load test**: k6
* **Deployment**: Docker Compose → Kubernetes + KEDA

---
---

# Engineering Deep-Dive

I like this question. You’re right — there *is* a difference between:
* **Developer** → writes features that work
* **Software Engineer** → designs systems that survive reality

If you’re building a Flash-Sale high-traffic microservice system and want to think like a **software engineer**, here’s the theory stack you should master.

---

## 1. Core Foundations (Non-Negotiable)
These are engineering fundamentals.

### 1.1 Data Structures & Algorithms
**Why?**
* Prevent O(n²) disasters under load
* Understand memory/CPU tradeoffs

**Focus on:**
* Hash maps
* Queues
* Heaps
* Concurrency-safe structures
* Complexity analysis (Big-O)

### 1.2 Operating Systems
**You must understand:**
* Threads vs processes
* Context switching
* Memory models
* File descriptors
* TCP sockets
* How Linux handles connections

**Why?**
Because “5,000 req/sec” is an OS + network problem before it's a code problem.

### 1.3 Computer Networking
**Critical for distributed systems:**
* TCP vs UDP
* HTTP lifecycle
* Connection pooling
* TLS overhead
* Load balancing (L4 vs L7)
* Reverse proxies
* DNS behavior
* CAP theorem (distributed consistency tradeoffs)

Without networking theory, microservices are guesswork.

---

## 2. Distributed Systems Theory (Very Important)
This is where developers become engineers.

### 2.1 CAP Theorem
Consistency vs Availability vs Partition tolerance.
**Flash-sale system example:**
* Inventory must prefer **Consistency**
* Analytics can prefer **Availability**

### 2.2 Consistency Models
* Strong consistency
* Eventual consistency
* Read-after-write
* Monotonic reads
You must know when to choose which.

### 2.3 Distributed Transactions
* 2PC (Two Phase Commit)
* Sagas (orchestration vs choreography)
* Compensation transactions
**Your flash-sale system uses:**
> Saga pattern + compensation

### 2.4 Idempotency & Exactly-Once Illusion
* At-least-once delivery
* At-most-once delivery
* Idempotent consumers
* Deduplication strategies
**RabbitMQ gives you:**
> At-least-once
You must design for duplicates.

### 2.5 The Fallacies of Distributed Systems
**You must deeply understand:**
* The network is reliable ❌
* Latency is zero ❌
* Bandwidth is infinite ❌
* Topology doesn’t change ❌
Production systems fail because people ignore these.

---

## 3. Architecture & System Design

### 3.1 Monolith vs Microservices
**Tradeoffs:**
* Operational complexity
* Deployment complexity
* Debug difficulty
* Data ownership
Microservices are not “cool” — they are expensive.

### 3.2 Domain-Driven Design (DDD)
**Learn:**
* Bounded contexts
* Aggregates
* Entities vs Value Objects
* Domain events
Inventory and Order should be separate bounded contexts.

### 3.3 Event-Driven Architecture
* Pub/Sub
* Event sourcing
* Outbox pattern
* CQRS (Command Query Responsibility Segregation)

### 3.4 Scalability Patterns
* Horizontal scaling
* Stateless services
* Caching strategies
* Load shedding
* Backpressure
* Rate limiting
* Circuit breakers

---

## 4. Reliability Engineering
Now we enter **real engineering** territory.

### 4.1 Fault Tolerance
* Retry strategies
* Exponential backoff
* Dead-letter queues
* Timeout strategies

### 4.2 Observability
Difference between logging and observability.
**You must understand:**
* Metrics
* Logs
* Traces
* SLO / SLA / SLIs

**Engineering mindset:**
> If you can't measure it, you don’t understand it.

### 4.3 Chaos Engineering
* Inject latency
* Kill pods
* Drop messages
* Simulate partition
See how your system behaves.

### 4.4 Resilience Patterns
* Circuit breaker
* Bulkhead
* Rate limiter
* Timeout + fallback

---

## 5. Database Theory
Critical for flash-sale system.

### 5.1 ACID
* Atomicity
* Consistency
* Isolation
* Durability

### 5.2 Isolation Levels
* Read committed
* Repeatable read
* Serializable
Overselling happens when isolation is misunderstood.

### 5.3 Locking & Concurrency Control
* Optimistic locking
* Pessimistic locking
* Row-level locks
* Deadlocks

### 5.4 Indexing & Query Planning
Under load, bad indexes = system collapse.

---

## 6. Cloud & Infrastructure Engineering

### 6.1 Containers
* Docker networking
* Layer caching
* Resource limits

### 6.2 Kubernetes
* Pods
* ReplicaSets
* HPA
* KEDA
* Liveness vs Readiness probes

### 6.3 Load Testing
* Throughput vs latency
* p95 vs average
* Queue depth metrics

---

## 7. Security Engineering
Often ignored.
* JWT design
* OAuth basics
* Rate limiting
* Abuse prevention
* Replay attack prevention
* Secure headers
* Input validation
Flash-sale systems get attacked.

---

## 8. Performance Engineering
**Understand:**
* CPU-bound vs IO-bound
* Async vs blocking
* Thread pools
* Connection pools
* GC behavior (if JVM)
* Memory leaks under load

---

## 9. Engineering Thinking (The Real Difference)

**Developer asks:**
> "How do I implement this feature?"

**Engineer asks:**
> "What happens when 10,000 users hit this at once?"
> * What fails first?
> * How do we observe failure?
> * How do we recover?
> * What is the blast radius?
> * What is the rollback strategy?
> * What is the capacity limit?
> * What is the cost?

---

## Final Path Recommendation

### Phase 1 — Foundations
* OS basics
* Networking basics
* Database internals
* Concurrency

### Phase 2 — Distributed Systems
* CAP
* Consistency models
* Sagas
* Event-driven architecture
* Idempotency

### Phase 3 — Scalability & Reliability
* Resilience patterns
* Observability
* Autoscaling
* Load testing

### Phase 4 — Production Engineering
* Kubernetes
* CI/CD
* Incident response
* Monitoring dashboards

---

## Final Advice
To become a **software engineer**, you must learn to think in:
* Failure modes
* Tradeoffs
* Constraints
* Data consistency
* Performance envelopes
* Observability

Not just:
* Controllers
* CRUD
* Endpoints
* Framework features
