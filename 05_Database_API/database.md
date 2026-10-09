# Diagram 9: Database Entity-Relationship (ER) Diagram

```mermaid
erDiagram
    CUSTOMER ||--o{ INVENTORY_RESERVATION : reserves
    CUSTOMER ||--o{ ORDERS : places
    CUSTOMER ||--o{ PAYMENT : makes

    PRODUCT ||--|| INVENTORY : has
    PRODUCT ||--o{ INVENTORY_RESERVATION : allocated_in
    PRODUCT ||--o{ ORDER_ITEM : ordered_as

    INVENTORY_RESERVATION ||--|| ORDERS : creates
    ORDERS ||--o{ ORDER_ITEM : contains
    ORDERS ||--|| PAYMENT : settled_by

    CUSTOMER {
        uuid customer_id PK
        string email UK
        string full_name
        timestamp created_at
    }

    PRODUCT {
        uuid product_id PK
        string sku UK
        string name
        numeric base_price
        boolean is_flash_sale
    }

    INVENTORY {
        uuid inventory_id PK
        uuid product_id FK,UK
        integer available_quantity "CHECK >= 0"
        integer reserved_quantity "CHECK >= 0"
        integer sold_quantity "CHECK >= 0"
        bigint version
        timestamp updated_at
    }

    INVENTORY_RESERVATION {
        uuid reservation_id PK
        uuid customer_id FK
        uuid product_id FK
        integer quantity
        string status
        timestamp expires_at
        string idempotency_key UK
    }

    ORDERS {
        uuid order_id PK
        uuid customer_id FK
        uuid reservation_id FK,UK
        uuid payment_id UK
        string status
        numeric total_amount
        timestamp created_at
    }

    ORDER_ITEM {
        uuid order_item_id PK
        uuid order_id FK
        uuid product_id FK
        integer quantity
        numeric unit_price
    }

    PAYMENT {
        uuid payment_id PK
        uuid order_id FK
        uuid customer_id FK
        numeric amount
        string status
        string transaction_reference
        string idempotency_key UK
    }
```

### Key Highlights
* **7 Core Entities:** Contains only the entities essential to flash sales: Customer, Product, Inventory, Reservation, Order, OrderItem, and Payment.
* **Non-Negative Constraints:** `available_quantity`, `reserved_quantity`, and `sold_quantity` have database check constraints forbidding negative numbers.
* **Idempotency Uniqueness:** Both `INVENTORY_RESERVATION` and `PAYMENT` enforce unique constraints on `idempotency_key` to prevent duplicates.
* **1-to-1 Protection:** Orders strictly require a valid, non-expired `reservation_id` to exist.
