# Diagram 4: Inventory Concurrency Diagram

```mermaid
graph TD
    UserA["User 1"] --> ResSvc["Reservation Service"]
    UserB["User 2"] --> ResSvc
    UserC["User 3"] --> ResSvc
    UserDots["..."] --> ResSvc
    User9999["User 9,999"] --> ResSvc
    User10000["User 10,000"] --> ResSvc

    ResSvc --> AtomicOp["Atomic Inventory Update<br/>(Cache Lua Script / SQL Conditional Update)<br/>WHERE available_quantity > 0"]

    AtomicOp --> Branch{"Check Remaining Stock"}

    Branch -->|"Stock > 0 (First 100)"| Reserve["RESERVE UNIT<br/>(Decrement Stock by 1)"]
    Branch -->|"Stock == 0 (Remaining 9,900)"| SoldOut["SOLD OUT<br/>(Instant HTTP 409)"]

    Reserve --> UniqueCheck["Two concurrent requests CANNOT<br/>reserve the same final 100th unit<br/>due to atomic serialization."]
```

### Key Highlights
* **Atomic Serialization:** Concurrent incoming requests pass through a serialized atomic gate (in-memory single-thread or database row lock).
* **Guaranteed Unit Allocation:** Each winning request atomically decrements exactly 1 unit; two threads can never claim the same unit.
* **Conditional Gate:** The condition `available_quantity > 0` ensures that once stock hits 0, all subsequent updates return 0 affected rows.
* **Immediate Rejection:** Requests 101 to 10,000 fail immediately without waiting or blocking server threads.
