# SALESTORM — Design Patterns & SOLID

## 1. Design Patterns

```
+--------------------+--------------------------------+------------------------------------------------+
| Pattern            | Where Used                     | Problem & Benefit (1-2 sentences)              |
+--------------------+--------------------------------+------------------------------------------------+
| **Strategy**       | Payment Provider Selection     | Allows switching payment methods at runtime.   |
| **Factory**        | Payment Gateway Creation       | Instantiates appropriate gateway dynamically.  |
| **State**          | Order & Reservation Lifecycle  | Eliminates nested if-statements for states.   |
| **Repository**     | Database Access Layer          | Decouples domain models from SQL queries.      |
| **Adapter**        | External Payment Gateway (PSP) | Translates vendor APIs into uniform interface. |
| **Observer/Event** | Notifications & Fulfillment    | Decouples downstream services via queue events.|
| **Facade**         | Checkout Workflow              | Simplifies multi-step checkout into one API.   |
| **Circuit Breaker**| External Gateway Calls         | Prevents thread exhaustion when PSP is down.   |
+--------------------+--------------------------------+------------------------------------------------+
```

---

## 2. SOLID Principles in SALESTORM

* **S (Single Responsibility):** `ReservationService` handles stock allocation; background workers handle 10-minute lease expiry.
* **O (Open/Closed):** Adding a new payment gateway (e.g. Apple Pay) extends `PaymentGateway` without modifying existing code.
* **L (Liskov Substitution):** All gateway adapters (`StripeAdapter`, `AdyenAdapter`) can substitute `PaymentGateway` seamlessly.
* **I (Interface Segregation):** Catalog reads use `IInventoryReader`; reservation writes use `IInventoryModifier`.
* **D (Dependency Inversion):** Services depend on repository and message publisher abstractions, not raw database or Kafka drivers.
