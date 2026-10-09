# Diagram 6: Purchase Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant GW as API Gateway
    participant Sale as Sale Service
    participant Inv as Inventory Service
    participant DB as SQL Database
    participant Pay as Payment Service
    participant Order as Order Service

    Customer->>GW: 1. Click "Buy Now"
    GW->>Sale: 2. Request Purchase
    Sale->>Inv: 3. Reserve Stock

    rect rgb(240, 248, 255)
        Note over Inv, DB: Atomic Reservation Execution
        Inv->>DB: 4. Atomic Reservation (UPDATE ... WHERE stock > 0)
        alt Stock Available (First 100)
            DB-->>Inv: 5. Stock Decremented (Success)
            Inv-->>Pay: 6. Initiate Payment
            Pay-->>Order: 7. Payment Succeeded
            Order->>DB: 8. Create Confirmed Order
            Order-->>Customer: 9. Order Confirmed (HTTP 200)
        else Stock Unavailable (Remaining 9,900)
            DB-->>Inv: 5. 0 Rows Updated (Out of Stock)
            Inv-->>Sale: 6. SOLD_OUT
            Sale-->>Customer: 7. HTTP 409: Item Sold Out
        end
    end
```

### Key Highlights
* **End-to-End Flow:** Traces a purchase from customer click through gateway, sale, inventory reservation, payment, and order creation.
* **Atomic Gate:** The Database/Cache atomic reservation determines whether the flow branches to payment or immediate rejection.
* **Winning Path:** Successful stock holds immediately trigger payment processing and order confirmation.
* **Failing Path:** Requests past 100 receive an immediate `SOLD_OUT` response, terminating without invoking payment rails.
