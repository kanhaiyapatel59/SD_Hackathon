# SALESTORM — Scalability, Reliability & Failure Scenarios

## 1. Scalability Architecture (10k to 500k Requests/Sec)

* **Normal Traffic (10k RPS):** Served by stateless service instances behind the Load Balancer with database read replicas.
* **Peak Traffic (500k RPS):**
  - **Edge Caching:** CDN absorbs 450,000 RPS of static discovery reads.
  - **Rate Limiting & Backpressure:** API Gateway token bucket limits requests to 1 req/sec per user.
  - **In-Memory Reservation:** 10,000 Buy requests hit in-memory Cache; 9,900 are rejected in $< 2\text{ms}$.
  - **Queue Decoupling:** Message Queue buffers downstream payment and order writes.

---

## 2. The 8 Critical Failure Scenarios

| # | Failure Scenario | Detection | Recovery Mechanism | Final Consistency |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Inventory Reaches Zero** | Counter hits 0 in Cache/DB | Instant HTTP 409 `SOLD_OUT` returned | Available stock = 0, no overselling |
| **2** | **Duplicate Buy Request** | Duplicate `Idempotency-Key` | Database rejects duplicate insert; returns cached receipt | Exactly 1 reservation created |
| **3** | **Payment Failure** | Gateway returns card decline | Reservation marked `RELEASED`; stock incremented (+1) | Stock returned to public pool |
| **4** | **Payment Timeout** | Gateway network drops after 3s | Query gateway status by unique transaction reference | Verified before retrying or releasing |
| **5** | **Order Service Down (30s)** | Pod heartbeat lost | Message Queue buffers event; consumer catches up on reboot | 100% orders materialized |
| **6** | **Database Failure** | Connection drop | Database failover promotes replica in $< 30\text{s}$ | ACID transactions preserved |
| **7** | **Message Queue Delay** | Consumer lag metric spikes | Auto-scale consumer instances horizontally | Queue clears, events processed |
| **8** | **50x Traffic Spike** | Edge ingress > 100k RPS | Virtual Waiting Room queues excess requests at edge | Origin protected from crash |

---

## 3. Concise Security & Observability

* **Security:** HTTPS/TLS 1.3, JWT auth at API Gateway, Rate limiting, PCI-DSS compliant hosted payment fields, non-negative DB constraints.
* **Observability:** Prometheus metrics (RPS, latency, error rate, queue lag), structured JSON logs with correlation IDs, and P1 alerts if `available + reserved + sold != 100`.
