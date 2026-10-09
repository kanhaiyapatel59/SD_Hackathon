# Diagram 7: Payment & Order Failure Recovery Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant PaySvc as Payment Service
    participant PSP as Payment Gateway
    participant Queue as Message Queue
    participant OrderSvc as Order Service

    Note over Customer, OrderSvc: Scenario 1: Duplicate Payment Request (Idempotency Key)
    Customer->>PaySvc: Click "Pay" (Idempotency-Key: KEY-1)
    Customer->>PaySvc: Double Click "Pay" (Same Idempotency-Key: KEY-1)
    PaySvc->>PaySvc: Check Key: First in-flight, Second returns Cached Result (No Double Charge)

    Note over Customer, OrderSvc: Scenario 2: Payment Timeout & Retry
    PaySvc->>PSP: Authorize Payment (Timeout after 3s)
    PSP--xPaySvc: Socket Timeout (Unknown state)
    PaySvc->>PSP: Query Transaction Status by Unique Ref
    PSP-->>PaySvc: Return "Charge Succeeded" (Recovered)

    Note over Customer, OrderSvc: Scenario 3: Order Service Downtime for 30 Seconds
    PaySvc->>Queue: Publish PaymentCompletedEvent
    Queue-->>PaySvc: Event Persisted to Disk (ACK)
    PaySvc-->>Customer: HTTP 200 "Payment Successful"

    rect rgb(255, 235, 235)
        Note over OrderSvc: Order Service is DOWN / CRASHED (30 Seconds)
        Note over Queue: Message stays safely buffered in queue log
    end

    rect rgb(240, 248, 255)
        Note over OrderSvc: T = 31s: Order Service Restarts & Connects
        Queue->>OrderSvc: Deliver buffered PaymentCompletedEvent
        OrderSvc->>OrderSvc: Deduplicate against DB & Insert Confirmed Order
        OrderSvc->>Queue: Commit Offset (Order Created with Zero Loss!)
    end
```

### Key Highlights
* **Duplicate Protection:** Duplicate clicks carrying the same `Idempotency-Key` are resolved from the cache/database with zero duplicate charges.
* **Timeout Reconciliation:** Payment timeouts query the gateway status by unique reference before deciding to charge or abort.
* **Durable Queue Buffering:** If the Order Service crashes for 30 seconds post-payment, events buffer safely on message queue disks.
* **Automatic Consumer Catch-up:** Upon reboot, the Order Service consumes all pending messages and materializes orders with 100% data preservation.
