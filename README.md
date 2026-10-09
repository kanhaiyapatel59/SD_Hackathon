# SALESTORM — High-Scale Flash-Sale Architecture Blueprint

## 1. Problem
* **10,000 customers** click "Buy Now" simultaneously.
* Only **100 units** of inventory are available.
* Traffic: Normal **10,000 req/sec**, Peak **500,000 req/sec**.
* **Goal:** Guarantee **zero overselling**, **idempotent payments**, and **reliable order recovery**.

---

## 2. Architecture Overview
SALESTORM uses 7 focused microservices and standard infrastructure:

```
[10,000 Customers]
        │
        ▼
   [CDN / WAF]  <--- Absorbs 90% read traffic
        │
        ▼
 [Load Balancer]
        │
        ▼
  [API Gateway]
        │
        +---> [Product / Sale Service]
        +---> [Inventory / Reservation Service] <---> [In-Memory Cache (Stock = 100)]
        +---> [Checkout Service]
        +---> [Payment Service] ────> [Message Queue]
                                             │
                                             ▼
                                      [Order Service] ────> [SQL Database]
                                             │
                                             ▼
                                 [Shipment / Notification]
```

---

## 3. The 10,000 vs. 100 Solution (No Overselling)

* **The Invariant:** $\text{Successful Reservations} \le \text{Available Inventory} \le 100$.
* **How it works:**
  1. Requests arrive at the Reservation Service.
  2. An in-memory atomic decrement executes in a single-threaded queue.
  3. The **first 100 requests** decrement the stock counter from 100 to 0 and claim 10-minute reservation leases.
  4. The **remaining 9,900 requests** evaluate `stock == 0` and are immediately rejected with `SOLD_OUT` in $< 15\text{ms}$.
  5. The SQL Database enforces `CHECK (available_quantity >= 0)` as defense-in-depth.
  6. **Result: Zero overselling. The database is never overwhelmed.**

---

## 4. Payment Reliability & Idempotency
* **Duplicate Prevention:** Client passes `Idempotency-Key`. The database enforces a unique constraint on this key. Double clicks return the original receipt without double-charging.
* **Timeout Handling:** If the payment gateway times out, the system queries transaction status by reference before retrying.

---

## 5. Order Recovery (The 30-Second Outage)
* **Scenario:** Customer pays successfully, but the Order Service crashes for 30 seconds.
* **Resolution:** Payment Service publishes `PaymentCompletedEvent` to the durable Message Queue.
* While the Order Service is down, messages buffer safely on disk.
* When the Order Service restarts, it consumes all buffered events and materializes confirmed orders with **zero data loss**.

---

## 6. Scalability Strategy
* **Horizontal Scaling:** Stateless service instances scale out behind the Load Balancer.
* **Edge Caching:** CDN absorbs 450,000 RPS of catalog browsing traffic.
* **Rate Limiting:** Token bucket limits each user to 1 reservation attempt per second.
* **Decoupled Queue:** Message Queue protects downstream fulfillment from traffic bursts.

---

## 7. Key Trade-offs
* **In-Memory Cache vs. Pure Database:** In-memory reservation provides $< 2\text{ms}$ latency and sheds 9,900 losers instantly, but requires periodic synchronization with the SQL master database.
* **Synchronous vs. Asynchronous:** Ingress reservation and payment are synchronous for immediate user feedback; order creation is asynchronous for maximum resilience.

---

## 8. Project Structure

```
SALESTORM/
├── 01_Requirements/
│   └── requirements.md
├── 02_Architecture/
│   ├── system_context.md
│   ├── hld.md
│   ├── architecture_diagram.md
│   └── deployment_diagram.md
├── 03_Concurrency_Inventory/
│   ├── inventory_design.md
│   ├── reservation_sequence.md
│   └── reservation_state.md
├── 04_Payment_Order/
│   ├── payment_design.md
│   ├── payment_sequence.md
│   ├── order_design.md
│   ├── order_sequence.md
│   └── order_state.md
├── 05_Database_API/
│   ├── database.md
│   └── api.md
├── 06_LLD/
│   ├── inventory_class_diagram.md
│   ├── payment_class_diagram.md
│   └── order_class_diagram.md
├── 07_SOLID_Patterns/
│   └── design_patterns.md
├── 08_Reliability/
│   └── scalability_reliability.md
├── 09_ADR/
│   └── architecture_decisions.md
├── 10_Presentation/
│   └── 5_minute_pitch.md
├── index.html                   <-- Interactive Visual Presentation Dashboard
└── README.md
```
