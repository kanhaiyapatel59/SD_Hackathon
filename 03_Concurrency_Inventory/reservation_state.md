# Diagram 5: Reservation State Machine Diagram

```mermaid
stateDiagram-v2
    [*] --> AVAILABLE : Initial Stock = 100

    AVAILABLE --> RESERVED : Buy Now Clicked (Atomic Decrement)
    
    RESERVED --> PAYMENT_PENDING : User Lands on Checkout
    
    PAYMENT_PENDING --> CONFIRMED : Payment Captured Successfully
    
    CONFIRMED --> SOLD : Order Created (Permanent Sale)

    RESERVED --> TIMEOUT : 10-Minute Timeout Expires
    TIMEOUT --> RELEASED : Trigger Stock Reclamation
    
    PAYMENT_PENDING --> PAYMENT_FAILED : Card Declined
    PAYMENT_FAILED --> RELEASED : Trigger Immediate Release

    RELEASED --> AVAILABLE : Stock Incremented (+1)

    SOLD --> [*]
```

### Key Highlights
* **Happy Path:** `AVAILABLE` $\to$ `RESERVED` $\to$ `PAYMENT_PENDING` $\to$ `CONFIRMED` $\to$ `SOLD`.
* **Lease Expiry Path:** If 10 minutes elapse without payment, state moves `RESERVED` $\to$ `TIMEOUT` $\to$ `RELEASED` $\to$ `AVAILABLE`.
* **Payment Failure Path:** If the card is declined, state moves `PAYMENT_PENDING` $\to$ `PAYMENT_FAILED` $\to$ `RELEASED` $\to$ `AVAILABLE`.
* **Zero Inventory Stranded:** Any failure or timeout automatically returns the unit back to `AVAILABLE` for the next waiting customer.
