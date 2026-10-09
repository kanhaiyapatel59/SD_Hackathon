# SALESTORM — Order System & Failure Recovery Design

## 1. Overview
The Order Service manages customer purchase records and fulfillment lifecycles. It is designed around **asynchronous event consumption** and **idempotent event processing** so that it can recover gracefully from crashes.

---

## 2. The 30-Second Order Service Outage Recovery (Deep-Dive)

```
+---------------------------------------------------------------------------------------------------+
|                                 ORDER SERVICE RECOVERY TIMELINE                                   |
+---------+-----------------------+-----------------------------------------------------------------+
| Time    | Component             | Event / State                                                   |
+---------+-----------------------+-----------------------------------------------------------------+
| T = 0s  | Payment Gateway       | Card charged successfully ($499.00).                            |
| T = 1s  | Payment Service       | Publishes PaymentCompletedEvent to Message Queue (Topic).       |
| T = 2s  | Message Queue         | Persists event durably to disk across 3 broker nodes.           |
| T = 3s  | Order Service         | CRASHES due to hardware glitch / OOM. Offline for 30 seconds.   |
| T = 4..32s | Message Queue      | Holds the message safely in queue. Zero messages dropped.       |
| T = 33s | Order Service         | Restarts and reconnects to Message Queue.                       |
| T = 34s | Order Service         | Reads buffered PaymentCompletedEvent from last committed offset.|
| T = 35s | Order Service         | Idempotently creates Confirmed Order in SQL database.           |
| T = 36s | Whole System          | Full recovery completed. 100% orders preserved.                 |
+---------+-----------------------+-----------------------------------------------------------------+
```

---

## 3. Idempotent Consumer Mechanism

When the Order Service consumes a message from the queue, network retries or partition rebalances could cause the same message to be delivered more than once.

To prevent duplicate orders:
1. The `orders` table enforces a unique constraint on `payment_id`:
   ```sql
   CREATE UNIQUE INDEX uq_order_payment_id ON orders (payment_id);
   ```
2. The consumer executes:
   ```java
   public void handlePaymentCompleted(PaymentCompletedEvent event) {
       // Check if order already created for this payment
       if (orderRepository.existsByPaymentId(event.getPaymentId())) {
           log.info("Duplicate event detected. Order already exists. Acknowledging message.");
           return; // Safe no-op
       }
       
       // Otherwise, create order
       Order order = new Order(event.getCustomerId(), event.getPaymentId(), "CONFIRMED");
       orderRepository.save(order);
   }
   ```
3. **Result:** Even if Kafka delivers the same event 5 times, **exactly 1 order** is created in the database.
