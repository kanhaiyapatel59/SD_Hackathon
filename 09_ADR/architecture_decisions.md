# SALESTORM — Architecture Decision Records (ADRs)

## Decision 1: SQL Database vs. NoSQL
* **Decision:** Use a **Relational SQL Database** (PostgreSQL) as the master system of record.
* **Reason:** Declarative `CHECK (available_quantity >= 0)` constraints provide a mathematical guarantee against overselling.
* **Trade-off:** Single primary writer requires connection pooling.
* **Alternative Rejected:** NoSQL (DynamoDB/MongoDB) lacked engine-level constraint enforcement across concurrent updates.

---

## Decision 2: Atomic In-Memory Decrement vs. Relational Locking
* **Decision:** Use an **In-Memory Atomic Decrement** (Cache) backed by SQL conditional updates.
* **Reason:** Redis single-threaded execution evaluates 10,000 requests in $< 50\text{ms}$, shedding 9,900 losers without database load.
* **Trade-off:** In-memory state requires periodic reconciliation with SQL master.
* **Alternative Rejected:** Pessimistic locking (`SELECT FOR UPDATE`) serialized 10,000 queries and exhausted DB connection pools.

---

## Decision 3: Synchronous Ingress vs. Asynchronous Order Processing
* **Decision:** **Synchronous** for Reservation & Payment; **Asynchronous** for Order fulfillment.
* **Reason:** Users demand instant feedback on whether they won the item and completed payment. Decoupling order creation via events protects orders if Order Service crashes.
* **Trade-off:** Order state is eventually consistent by a few hundred milliseconds.
* **Alternative Rejected:** Pure synchronous chain crashed payments whenever downstream order services degraded.

---

## Decision 4: Multi-Tier Caching Strategy
* **Decision:** Three-tier cache: Edge CDN (static catalog) + Gateway Cache (Sold-Out flag) + In-Memory Store (Stock counter).
* **Reason:** Absorbs 450,000 RPS at the CDN edge and rejects sold-out attempts at the gateway in $< 0.1\text{ms}$.
* **Trade-off:** Requires cache invalidation webhooks on flash-sale completion.
* **Alternative Rejected:** Direct database reads would collapse the database under normal traffic.

---

## Decision 5: Message Queue for Event Decoupling
* **Decision:** Deploy a durable **Message Queue** (e.g. Kafka) for payment and order events.
* **Reason:** Durable disk buffering allows the Order Service to recover from a 30-second outage with zero data loss.
* **Trade-off:** Requires idempotent consumer deduplication logic.
* **Alternative Rejected:** Direct HTTP microservice calls caused cascading failures during outages.
