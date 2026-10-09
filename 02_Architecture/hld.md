# Diagram 2: High-Level Architecture (HLD)

```mermaid
graph TB
    subgraph Users ["10,000 Simultaneous Customers"]
        Clients["Shopper Web & Mobile Apps"]
    end

    subgraph Edge ["Edge & Ingress Tier"]
        CDN["CDN / WAF (DDoS Shield & Static Cache)"]
        LB["Load Balancer"]
        Gateway["API Gateway (Rate Limiter & Auth)"]
    end

    subgraph Services ["Core Stateless Microservices"]
        ProductSvc["Product Service"]
        SaleSvc["Cart / Sale Service"]
        CheckoutSvc["Checkout Service"]
        ResSvc["Inventory / Reservation Service"]
        PaymentSvc["Payment Service"]
        OrderSvc["Order Service"]
        ShipmentSvc["Shipment / Notification Service"]
    end

    subgraph DataTier ["Data & Streaming Tier"]
        Cache[("Redis / Cache<br/>(Atomic Stock Counter = 100)")]
        DB[("SQL Database<br/>(Master Inventory & Orders)")]
        Queue{{"Message Queue<br/>(payment.completed)"}}
    end

    subgraph Observability ["Telemetry"]
        Obs["Monitoring / Logging / Tracing"]
    end

    Clients --> CDN
    CDN --> LB
    LB --> Gateway

    Gateway --> ProductSvc
    Gateway --> SaleSvc
    Gateway --> CheckoutSvc
    Gateway --> ResSvc
    Gateway --> PaymentSvc
    Gateway --> OrderSvc

    ResSvc <-->|"Atomic Stock Decrement"| Cache
    ResSvc -.->|"Async Sync"| DB
    PaymentSvc -->|"Record Payment"| DB
    PaymentSvc -->|"Publish PaymentCompleted"| Queue

    Queue -->|"Consume Event"| OrderSvc
    OrderSvc -->|"Create Confirmed Order"| DB
    OrderSvc -->|"Publish OrderConfirmed"| Queue
    Queue -->|"Consume Dispatch"| ShipmentSvc
```

### Key Highlights
* **Traffic Ingress:** CDN and WAF filter and cache discovery traffic, forwarding only active Buy requests to the API Gateway.
* **Fast-Path Reservation:** The Reservation Service uses in-memory Cache for atomic decrement in $< 2\text{ms}$, shielding the database.
* **Asynchronous Handoff:** Payment completion emits an event to the Message Queue, decoupling Payment from Order creation.
* **Resilient Order Creation:** The Order Service consumes messages from the queue to create orders; if Order Service goes down, messages buffer safely.
