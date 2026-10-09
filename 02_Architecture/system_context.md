# Diagram 1: System Context Diagram

```mermaid
graph TD
    Customer["👤 Customer / Shopper"]
    Admin["👨‍💼 Platform Admin"]

    subgraph SALESTORM ["SALESTORM Flash-Sale Platform"]
        CoreEngine["SALESTORM Core System<br/>(Catalog, Reservation, Checkout, Orders)"]
    end

    PSP["💳 Payment Gateway<br/>(Stripe / Adyen)"]
    Carrier["🚚 Shipping Partner<br/>(FedEx / DHL)"]
    Notification["📱 Notification Service<br/>(Twilio / SendGrid)"]

    Customer -->|"1. Browse & Buy Now (HTTPS)"| CoreEngine
    Admin -->|"Manage flash-sale inventory"| CoreEngine
    CoreEngine -->|"2. Authorize & Capture Charge"| PSP
    CoreEngine -->|"3. Dispatch package manifest"| Carrier
    CoreEngine -->|"4. Send SMS & Email alerts"| Notification
```

### Key Highlights
* **Customer Interface:** Customers interact via HTTPS to browse catalogs and initiate flash reservations.
* **Payment Isolation:** All card transactions route to external PCI-compliant Payment Gateways.
* **Fulfillment & Logistics:** Carrier integration is asynchronous; shipping manifests dispatch post-payment.
* **Notification Decoupling:** SMS/Email notifications are non-blocking; failure never delays or cancels an order.
