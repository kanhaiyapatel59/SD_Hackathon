# SALESTORM — Requirements & Invariants

## Core Scenario
* **10,000 customers** simultaneously click "Buy Now".
* Only **100 units** of product available.
* Traffic: Normal **10,000 req/sec**, Peak **500,000 req/sec**.

---

## Strict Invariants

```
                +------------------------------------------------+
                |          THE ZERO-OVERSELLING INVARIANT        |
                +------------------------------------------------+
                |                                                |
                |   SUCCESSFUL_RESERVATIONS <= AVAILABLE_STOCK   |
                |                                                |
                |   For this sale:                               |
                |   SUCCESSFUL_RESERVATIONS <= 100               |
                |                                                |
                |   AVAILABLE_QUANTITY >= 0  (At all times)      |
                |                                                |
                +------------------------------------------------+
```

* **No Overselling:** Reservations can never exceed initial stock (100).
* **Payment Idempotency:** Duplicate clicks/retries with same key produce exactly one financial charge.
* **10-Minute Lease Expiry:** Unpaid reservations release automatically after 600 seconds.
* **Zero Order Loss:** If Order Service crashes for 30s post-payment, orders recover 100% from queue.
