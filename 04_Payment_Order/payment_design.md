# SALESTORM — Payment System Design

## 1. Overview
The Payment Service coordinates card charges with external Payment Gateways (Stripe/Adyen). It enforces financial idempotency to prevent duplicate charges and decouples payment completion from order fulfillment.

---

## 2. The Five Critical Payment Scenarios

```
+---------------------------------------------------------------------------------------------------+
|                                    FIVE PAYMENT SCENARIOS MATRIX                                  |
+---+-----------------------------+------------------------------------+----------------------------+
| # | Scenario                    | Cause / Condition                  | Handling & Resolution      |
+---+-----------------------------+------------------------------------+----------------------------+
| A | Payment Succeeds            | Card valid, funds sufficient       | Order succeeds via event   |
| B | Payment Fails               | Card declined / fraud flag         | Release reservation back   |
| C | Payment Times Out           | Gateway drops network socket       | Status inquiry & reconciler|
| D | Duplicate Payment Request   | User clicks "Pay" twice            | Idempotency key stops char |
| E | Order Service Unavailable   | Order Service down for 30 seconds  | Message Queue recovers ord |
+---+-----------------------------+------------------------------------+----------------------------+
```

---

### Case A: Payment Succeeds $\to$ Order Succeeds (Happy Path)
1. Customer submits payment with `Idempotency-Key` and `reservation_id`.
2. Payment Service calls the Payment Gateway.
3. Gateway authorizes and returns `status = SUCCEEDED`.
4. Payment Service records `status = SUCCESS` in SQL database and publishes `PaymentCompletedEvent` to the **Message Queue**.
5. Customer receives immediate payment confirmation.

---

### Case B: Payment Fails $\to$ Reservation Released
1. Payment Gateway returns a card decline (e.g. `insufficient_funds`).
2. Payment Service records `status = FAILED` in the payment ledger.
3. Emits `PaymentFailedEvent` to the Message Queue.
4. Reservation Service consumes this event and immediately releases the reservation, adding +1 back to `available_quantity`.
5. Customer is notified to try another card.

---

### Case C: Payment Times Out $\to$ Retry / Reconciliation
* **The Danger:** A network timeout occurs after 3 seconds. The application does not know if the customer's card was charged or not. Blindly charging again causes double charges!
* **The Solution:**
  1. The Payment Service generates a unique transaction reference: `TX_REF = "TX-" + Idempotency-Key`.
  2. When a timeout occurs, the service **does NOT recharge**.
  3. Instead, it queries the Payment Gateway: `GET /charges?reference=TX_REF`.
  4. If the gateway says "Charge Succeeded", the system marks the payment successful.
  5. If the gateway says "Not Found", it is safe to retry or fail the payment.

---

### Case D: Duplicate Payment Request $\to$ Idempotency Prevents Double Charge
* **The Scenario:** Customer double-clicks "Pay Now" or their phone resends the request.
* **The Solution:**
  1. The SQL database enforces a unique constraint on `idempotency_key`:
     ```sql
     CREATE UNIQUE INDEX uq_payment_idempotency ON payment (idempotency_key);
     ```
  2. When the second request arrives with the same key:
     - The database rejects a second insert.
     - The Payment Service looks up the existing record and returns the **previous payment confirmation**.
  3. Result: The customer is charged **EXACTLY ONCE**.

---

### Case E: Payment Succeeds but Order Service is Temporarily Unavailable (30s Outage)
* **The Scenario:** Customer pays successfully. Immediately after, the Order Service crashes for 30 seconds.
* **The Flow:**
  ```
  Payment Gateway
        |
        v
  Payment SUCCESS
        |
        v
  Event / Durable Message (Published to Message Queue with acks=all)
        |
        v
  Order Service Temporarily Unavailable (Crashed / Rebooting for 30s)
        |
        v
  Message Remains Available (Safely buffered in Message Queue on disk)
        |
        v
  Order Service Restarts & Retries Consumer Loop
        |
        v
  Order Created Idempotently (100% Data Preserved, Zero Lost Orders)
  ```
