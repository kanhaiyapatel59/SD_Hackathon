# Diagram 3: The Critical 10,000 → 100 Architecture Diagram

```mermaid
graph TD
    Requests["10,000 CONCURRENT BUY REQUESTS"] --> Gateway["API Gateway"]
    Gateway --> ResSvc["Reservation Service"]
    ResSvc --> StockCheck[("Inventory Stock<br/>Available = 100 Units")]
    
    StockCheck -->|"Atomic Decrement<br/>Single-Threaded / DB Row Lock"| Decision{"Is Stock > 0?"}

    Decision -->|"YES (Requests 1 to 100)"| Winners["FIRST 100 REQUESTS<br/>Status: RESERVED (10-min lease)"]
    Decision -->|"NO (Requests 101 to 10,000)"| Losers["REMAINING 9,900 REQUESTS<br/>Status: SOLD OUT (Fast Rejection)"]

    Winners --> Payment["Payment Processing<br/>(Idempotent Card Authorization)"]
    Payment --> Order["Order Confirmed<br/>(Stock Becomes SOLD)"]
    
    subgraph InvariantRule ["NON-NEGOTIABLE INVARIANT"]
        Rule["SUCCESSFUL_RESERVATIONS <= AVAILABLE_STOCK<br/>SUCCESSFUL_RESERVATIONS <= 100<br/>RESULT: ZERO OVERSELLING"]
    end
```

### Key Highlights
* **Strict Invariant Enforced:** `SUCCESSFUL_RESERVATIONS <= 100`. Available inventory can never drop below zero.
* **Instant Separation:** The first 100 requests decrement the counter atomically from 100 to 0 and claim temporary leases.
* **Fast Failure for 9,900:** The remaining 9,900 requests evaluate `stock == 0` and receive an immediate `SOLD_OUT` response in $< 15\text{ms}$.
* **Zero Database Overload:** Rejected requests never touch the SQL database, shielding backend connections from collapse.
