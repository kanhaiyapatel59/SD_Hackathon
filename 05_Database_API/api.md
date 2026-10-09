# SALESTORM — API Specification

## 1. Overview
This specification details the **6 critical REST APIs** powering the SALESTORM flash-sale pipeline.

---

## 2. API Endpoints

### 1. `POST /cart`
* **Purpose:** Add a standard catalog item to the user's cart.
* **Idempotency:** None required.
* **Request:**
  ```json
  { "customer_id": "cust_123", "product_id": "prod_555", "quantity": 1 }
  ```
* **Success Response (200 OK):**
  ```json
  { "cart_id": "cart_888", "total_items": 1, "subtotal": 499.00 }
  ```
* **Failure Response (400 Bad Request):**
  ```json
  { "error": "INVALID_QUANTITY", "message": "Quantity must be greater than 0" }
  ```

---

### 2. `POST /reservations` (CRITICAL CONCURRENCY PATH)
* **Purpose:** Atomically reserve 1 unit of flash-sale stock for 10 minutes.
* **Idempotency:** **MANDATORY** via `X-Idempotency-Key` header.
* **Request:**
  ```json
  { "customer_id": "cust_123", "product_id": "prod_flash_16", "quantity": 1 }
  ```
* **Success Response (201 Created — Winning 100):**
  ```json
  {
    "reservation_id": "res_999",
    "status": "RESERVED",
    "expires_at": "2026-10-05T12:10:00Z",
    "ttl_seconds": 600
  }
  ```
* **Failure Response (409 Conflict — Losing 9,900):**
  ```json
  { "error": "SOLD_OUT", "message": "All flash sale units have been reserved." }
  ```

---

### 3. `POST /checkout`
* **Purpose:** Create checkout session and apply shipping address and discounts.
* **Idempotency:** Recommended via `X-Idempotency-Key`.
* **Request:**
  ```json
  { "reservation_id": "res_999", "shipping_address": "100 Market St, SF, CA" }
  ```
* **Success Response (200 OK):**
  ```json
  { "checkout_id": "chk_777", "total_amount": 499.00, "status": "READY_FOR_PAYMENT" }
  ```
* **Failure Response (410 Gone):**
  ```json
  { "error": "RESERVATION_EXPIRED", "message": "Your 10-minute hold has expired." }
  ```

---

### 4. `POST /payments`
* **Purpose:** Authorize and capture payment with external gateway.
* **Idempotency:** **MANDATORY** via `X-Idempotency-Key` header.
* **Request:**
  ```json
  { "reservation_id": "res_999", "amount": 499.00, "payment_token": "tok_visa_4242" }
  ```
* **Success Response (200 OK):**
  ```json
  { "payment_id": "pay_555", "transaction_ref": "TX_9876", "status": "SUCCESS" }
  ```
* **Failure Response (402 Payment Required):**
  ```json
  { "error": "CARD_DECLINED", "message": "Insufficient funds on card." }
  ```

---

### 5. `POST /orders`
* **Purpose:** Finalize confirmed order from payment.
* **Idempotency:** **MANDATORY** via `X-Idempotency-Key`.
* **Request:**
  ```json
  { "reservation_id": "res_999", "payment_id": "pay_555" }
  ```
* **Success Response (201 Created):**
  ```json
  { "order_id": "ord_101", "status": "CONFIRMED", "total_amount": 499.00 }
  ```
* **Failure Response (400 Bad Request):**
  ```json
  { "error": "INVALID_PAYMENT", "message": "Payment not verified." }
  ```

---

### 6. `GET /orders/{id}`
* **Purpose:** Fetch current status and tracking details for an order.
* **Idempotency:** None (Idempotent by HTTP GET specification).
* **Success Response (200 OK):**
  ```json
  {
    "order_id": "ord_101",
    "status": "CONFIRMED",
    "tracking_number": "FX-889102",
    "estimated_delivery": "2026-10-07"
  }
  ```
* **Failure Response (404 Not Found):**
  ```json
  { "error": "ORDER_NOT_FOUND", "message": "Order does not exist." }
  ```
