# Diagram 10C: Order LLD Class Diagram

```mermaid
classDiagram
    class Order {
        -UUID orderId
        -UUID customerId
        -UUID paymentId
        -BigDecimal totalAmount
        -OrderState currentState
        +transitionTo(OrderState nextState) void
        +canCancel() boolean
    }

    class OrderState {
        <<interface>>
        +confirm(Order order) void
        +process(Order order) void
        +ship(Order order) void
        +deliver(Order order) void
        +cancel(Order order) void
    }

    class ConfirmedState {
        +confirm(Order order) void
        +process(Order order) void
        +cancel(Order order) void
    }

    class ShippedState {
        +deliver(Order order) void
    }

    class OrderRepository {
        <<interface>>
        +save(Order order) void
        +findById(UUID orderId) Order
        +existsByPaymentId(UUID paymentId) boolean
    }

    class OrderService {
        -OrderRepository orderRepo
        +createOrder(PaymentEvent event) Order
        +cancelOrder(UUID orderId) void
    }

    Order --> OrderState : maintains
    ConfirmedState ..|> OrderState : implements
    ShippedState ..|> OrderState : implements
    OrderService --> OrderRepository : uses
    OrderService --> Order : creates
```

### Key Highlights
* **Order Aggregate:** Encapsulates order identity and items; state changes pass through `transitionTo()`.
* **State Pattern:** `OrderState` prevents illegal state jumps (e.g. canceling an order already shipped).
* **Deduplication Check:** `OrderRepository.existsByPaymentId()` prevents duplicate orders during consumer retries.
