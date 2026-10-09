# SALESTORM — 5-Minute Pitch Deck (10 Slides)

## Slide 1: The Problem (0:00 - 0:30)
* **The Scenario:** 10,000 customers simultaneously click "Buy Now" at millisecond $T_0$.
* **The Constraint:** Only 100 units exist.
* **The Danger:** Database row locks serialize 10,000 queries, connection pools crash, and naive systems oversell inventory.
* **Our Mission:** Deliver sub-50ms user response with a mathematically defensible zero-overselling guarantee.

---

## Slide 2: Strict Requirements & Invariants (0:30 - 0:55)
* **Non-Negotiable Invariant:** $\text{Successful Reservations} \le \text{Available Inventory} \le 100$.
* **Payment Idempotency:** Duplicate checkout clicks result in exactly one financial charge.
* **10-Minute Lease Expiry:** Unpaid reservations automatically return to the stock pool.
* **Zero Order Loss:** 100% order recovery even during downstream service outages.

---

## Slide 3: High-Level Architecture (HLD) (0:55 - 1:30)
* **Ingress:** CDN / WAF $\to$ Load Balancer $\to$ API Gateway.
* **Core Microservices:** Product, Sale/Cart, Reservation, Checkout, Payment, Order, Shipment.
* **Data Tier:** In-Memory Cache for atomic reservations, Message Queue for durable event decoupling, and SQL Database for ACID financial ledger.

---

## Slide 4: THE CRITICAL 10,000 → 100 FLOW (THE MOST IMPORTANT SLIDE) (1:30 - 2:30)
```
10,000 CUSTOMERS CLICK BUY NOW
             │
             ▼
        API GATEWAY
             │
             ▼
    RESERVATION SERVICE
             │
      ATOMIC STOCK GATE
             │
    ┌────────┴────────┐
    ▼                 ▼
FIRST 100 WIN     REMAINING 9,900
    │                 │
    ▼                 ▼
RESERVED (Lease)   SOLD OUT (Instant 409)
    │
    ▼
 PAYMENT
    │
    ▼
  ORDER
```
* **Why only 100 succeed:** The stock counter is decremented atomically.
* **The 100 Winners:** Secure a 10-minute reservation lease in $< 2\text{ms}$ and proceed to payment.
* **The 9,900 Losers:** Read `stock == 0` and receive an immediate `SOLD_OUT` response in $< 15\text{ms}$.
* **The Result:** Zero database lock contention, zero server crashes, **ZERO OVERSELLING**.

---

## Slide 5: Inventory Concurrency Mechanism (2:30 - 3:00)
* **Comparison:** Pessimistic locking exhausts database connection pools under 10,000 queries.
* **Chosen Mechanism:** In-Memory Single-Threaded Atomic Cache Decrement backed by:
  ```sql
  UPDATE inventory
  SET available_quantity = available_quantity - 1, reserved_quantity = reserved_quantity + 1
  WHERE product_id = :id AND available_quantity > 0;
  ```
* **Defense-in-Depth:** SQL check constraint `CHECK (available_quantity >= 0)` physically prevents negative stock.

---

## Slide 6: Payment & Order Recovery (3:00 - 3:45)
* **Duplicate Protection:** `Idempotency-Key` ensures double-clicks return existing receipts without re-charging.
* **Payment Timeout:** System queries gateway status by unique reference before retrying.
* **Order Service 30s Outage:**
  - Payment completes $\to$ Event published to Message Queue.
  - Order Service is down for 30s $\to$ Event buffers safely on queue disk.
  - Order Service revives $\to$ Consumes message and creates order with zero data loss!

---

## Slide 7: LLD, SOLID & Design Patterns (3:45 - 4:15)
* **SOLID:** Single Responsibility in reservation service; Open/Closed in payment adapters.
* **Design Patterns:** Strategy for payment providers, State Pattern for order lifecycle, Circuit Breaker for payment gateway timeouts, Repository for SQL isolation.

---

## Slide 8: Scalability & Reliability (4:15 - 4:35)
* **Normal 10k RPS:** Handled smoothly by stateless horizontal instances and read replicas.
* **Peak 500k RPS:** CDN edge absorbs 90% of traffic; rate limiters and virtual waiting room smooth spikes.

---

## Slide 9: Failure Scenarios (4:35 - 4:50)
* **Stock Exhaustion:** Immediate 409 reject.
* **Card Decline:** Reservation released immediately.
* **Checkout Abandonment:** 10-minute lease expiry worker reclaims stock.
* **Queue Replay:** Idempotent consumers check `UNIQUE(payment_id)` to prevent duplicate orders.

---

## Slide 10: Final Takeaway (4:50 - 5:00)
* **Simple & Defensible:** No bloated infrastructure, no over-engineering.
* **Mathematically Sound:** Atomic inventory decrement guarantees $\text{Reservations} \le 100$.
* **Production-Grade Resilience:** Decoupled queues guarantee zero dropped payments and zero lost orders.
