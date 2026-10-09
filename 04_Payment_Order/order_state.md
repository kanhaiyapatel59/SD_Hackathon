# Order State Machine Diagram

```mermaid
stateDiagram-v2
    [*] --> CREATED : Checkout session initiated

    CREATED --> PAYMENT_PENDING : User submits card details
    CREATED --> CANCELLED : 10-minute hold expired

    PAYMENT_PENDING --> CONFIRMED : PaymentCapturedEvent received
    PAYMENT_PENDING --> CANCELLED : Card declined / funds insufficient

    CONFIRMED --> PROCESSING : Warehouse picking slip generated
    CONFIRMED --> CANCELLED : User requested refund before dispatch

    PROCESSING --> SHIPPED : Carrier picked up parcel
    
    SHIPPED --> OUT_FOR_DELIVERY : Arrived at local courier hub
    
    OUT_FOR_DELIVERY --> DELIVERED : Package handed to customer

    DELIVERED --> [*]
    CANCELLED --> [*]
```

### Key Highlights
* **Order Flow:** Tracks orders from creation through payment, warehouse packing, and final delivery.
* **Payment Gate:** Orders can only enter `CONFIRMED` upon receiving a verified payment event.
* **Cancellation Guard:** Once an order transitions to `SHIPPED`, self-service cancellation is disabled to prevent package loss.
