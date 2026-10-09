# Diagram 11: Final Architecture One-Page Blueprint

```mermaid
graph TD
    subgraph TrafficFlow ["10,000 Customers Flash-Sale Pipeline"]
        Users["10,000 Concurrent Customers"] --> Traffic["Traffic Management<br/>(CDN / WAF / Virtual Waiting Room)"]
        Traffic --> LB["Load Balancer"]
        LB --> APIGW["API Gateway"]
        APIGW --> Checkout["Sale / Checkout Service"]
        Checkout --> ResEngine["Inventory / Reservation Service"]
        
        ResEngine -->|"Atomic Decrement"| StockCheck{"Stock Check<br/>Initial = 100"}
        
        StockCheck -->|"First 100"| SuccessRes["100 RESERVED LEASES<br/>(10-Minute Hold)"]
        StockCheck -->|"Remaining 9,900"| SoldOut["9,900 SOLD OUT<br/>(Instant 409 Conflict)"]
        
        SuccessRes --> Payment["Payment Service<br/>(Idempotent Gateway Call)"]
        Payment --> Order["Order Service<br/>(State: CONFIRMED)"]
        Order --> Dispatch["Shipment & Notification<br/>(Carrier Tracking & SMS)"]
    end

    subgraph Infrastructure ["Supporting Infrastructure"]
        CacheStore[("Redis / Cache<br/>• Atomic Counter: 100<br/>• User Duplicate Locks")]
        SQLStore[("SQL Database<br/>• Master Inventory<br/>• Orders & Payments")]
        EventBus{{"Message Queue<br/>• Durable Event Buffering<br/>• 30s Outage Recovery"}}
        Telemetry["Monitoring & Tracing<br/>(Prometheus / Jaeger / Logs)"]
    end

    ResEngine <--> CacheStore
    ResEngine -.-> SQLStore
    Payment --> SQLStore
    Payment --> EventBus
    EventBus --> Order
    Order --> SQLStore
    EventBus --> Dispatch
```

### Key Highlights
* **One-Way Pipeline:** Clean progression from traffic ingress through atomic reservation, payment, and order generation.
* **Invariant Enforced:** Exactly 100 units transition to `RESERVED`; 9,900 requests fail fast at the cache layer with zero database contention.
* **Decoupled by Queue:** If downstream fulfillment or Order Service stalls, the Message Queue buffers events with zero data loss.
* **Full Observability:** Telemetry tracks latency, queue backlog, and guarantees `available + reserved + sold == 100` at all times.
