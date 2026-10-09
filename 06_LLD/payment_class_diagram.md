# Diagram 10B: Payment LLD Class Diagram

```mermaid
classDiagram
    class Payment {
        -UUID paymentId
        -UUID orderId
        -BigDecimal amount
        -String status
        -String transactionReference
        -String idempotencyKey
        +markSuccess(String txRef) void
        +markFailed(String reason) void
    }

    class PaymentStrategy {
        <<interface>>
        +executePayment(PaymentRequest request) PaymentResponse
    }

    class CreditCardStrategy {
        -PaymentGateway gateway
        +executePayment(PaymentRequest request) PaymentResponse
    }

    class PaymentGateway {
        <<interface>>
        +charge(String token, BigDecimal amount, String ref) GatewayResult
        +queryStatus(String ref) GatewayResult
    }

    class PaymentRepository {
        <<interface>>
        +save(Payment payment) void
        +findByIdempotencyKey(String key) Payment
    }

    class PaymentService {
        -PaymentRepository paymentRepo
        -PaymentStrategy strategy
        +processPayment(PaymentRequest req, String key) PaymentResponse
    }

    PaymentService --> PaymentRepository : uses
    PaymentService --> PaymentStrategy : delegates to
    CreditCardStrategy ..|> PaymentStrategy : implements
    CreditCardStrategy --> PaymentGateway : calls
    PaymentService --> Payment : creates
```

### Key Highlights
* **PaymentService:** Coordinates payment requests and checks `PaymentRepository` to guarantee idempotency.
* **PaymentStrategy:** Strategy Pattern allowing pluggable payment methods without altering core business rules.
* **PaymentGateway:** Adapter Pattern isolating external payment gateway APIs (Stripe, Adyen).
