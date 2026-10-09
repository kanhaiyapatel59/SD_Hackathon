# Diagram 10A: Inventory LLD Class Diagram

```mermaid
classDiagram
    class Inventory {
        -UUID productId
        -int availableQuantity
        -int reservedQuantity
        -int soldQuantity
        -long version
        +reserve(int qty) boolean
        +release(int qty) void
        +confirmSale(int qty) void
    }

    class Reservation {
        -UUID reservationId
        -UUID customerId
        -UUID productId
        -int quantity
        -String status
        -DateTime expiresAt
        -String idempotencyKey
        +isExpired() boolean
    }

    class InventoryService {
        -InventoryRepository inventoryRepo
        +checkStock(UUID productId) int
    }

    class ReservationService {
        -InventoryRepository inventoryRepo
        -CacheClient cache
        +reserveStock(UUID customerId, UUID productId, String key) ReservationResult
        +releaseReservation(UUID reservationId) void
    }

    class InventoryRepository {
        <<interface>>
        +findByProductId(UUID productId) Inventory
        +save(Inventory inventory) void
        +atomicDecrement(UUID productId, int qty) boolean
    }

    Inventory "1" --> "*" Reservation : tracks
    InventoryService --> InventoryRepository : uses
    ReservationService --> InventoryRepository : uses
    ReservationService --> Inventory : updates
```

### Key Highlights
* **Inventory Aggregate:** Enforces invariant `available + reserved + sold == total` within domain boundaries.
* **ReservationService:** Coordinates fast in-memory checks with cache and records durable reservations.
* **InventoryRepository:** Encapsulates atomic SQL updates (`UPDATE inventory SET available = available - 1 WHERE available > 0`).
