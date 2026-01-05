# Resilient Order Orchestration & Status API (ROOSA)

## Objective

Build a production‑grade MuleSoft integration project using Anypoint Code Builder that handles order orchestration across multiple downstream systems with status tracking and failure handling.

---

## Applications to Build

You will create **six Mule applications**:

1. **order-exp-api** – Experience API
2. **order-proc-api** – Process API
3. **inventory-sys-api** – System API (mock)
4. **payment-sys-api** – System API (mock)
5. **shipping-sys-api** – System API (mock)
6. **order-status-store** – Shared Object Store configuration

---

## High-Level Architecture

```
Client
  |
  v
order-exp-api
  |
  v
order-proc-api
  |
  +--> inventory-sys-api
  |
  +--> payment-sys-api
  |
  +--> shipping-sys-api

(All status persisted in Object Store)
```

---

## Order Lifecycle Status Values

Persist **only** the following states:

```
RECEIVED
INVENTORY_CHECK_STARTED
INVENTORY_CONFIRMED
INVENTORY_FAILED
PAYMENT_COMPLETED
PAYMENT_FAILED
SHIPPING_REQUESTED
SHIPPED
SHIPPING_FAILED
```

---

## order-exp-api

### Endpoints

#### POST /orders

**Flow**

1. Receive order request
2. Generate `orderId`
3. Persist status = `RECEIVED`
4. Invoke `order-proc-api` asynchronously
5. Respond with `202 Accepted` and `orderId`

---

#### GET /orders/{orderId}/status

**Flow**

1. Read order status from Object Store
2. Return current status

---

## order-proc-api

### Flow: process-order

```
Start
  |
  v
Update status -> INVENTORY_CHECK_STARTED
  |
  v
Call inventory-sys-api
  |
  +--> failure -> INVENTORY_FAILED -> END
  |
  v
Update status -> INVENTORY_CONFIRMED
  |
  v
Call payment-sys-api
  |
  +--> failure -> PAYMENT_FAILED -> END
  |
  v
Update status -> PAYMENT_COMPLETED
  |
  v
Call shipping-sys-api (async)
  |
  +--> failure -> SHIPPING_FAILED -> END
  |
  v
Update status -> SHIPPED
END
```

### Mandatory Implementation Requirements

* Retry scope on inventory and payment calls
* Timeout on all outbound HTTP calls
* Correlation ID propagated to all system APIs
* Status update after **every** step

---

## inventory-sys-api (Mock)

### Endpoint

```
POST /inventory/check
```

### Behavior

* 70% success response
* 30% failure response
* Random latency between 0.5–2 seconds

---

## payment-sys-api (Mock)

### Endpoint

```
POST /payment/charge
```

### Behavior

* 60% success response
* 40% failure response
* Occasional timeout simulation

---

## shipping-sys-api (Mock)

### Endpoint

```
POST /shipping/create
```

### Behavior

* Asynchronous processing
* Random failure rate
* Latency between 1–4 seconds

---

## Data Persistence

* Use Mule Object Store
* Key: `orderId`
* Value: current order status
* Status must be overwritten at every state transition

---

## Non‑Negotiable Constraints

* Separate Mule application per API layer
* No business logic in Experience API
* No direct System API calls from Experience API
* Shipping call must be asynchronous
* No generic or vague order states

---

## Deliverables

* RAML definitions for all APIs
* Mule flows implementing the above logic
* Configurable failure rates via properties
* Environment‑based configuration (dev/test)

---

## Completion Criteria

The project is considered complete when:

* Orders can be created and tracked end‑to‑end
* Partial failures are reflected accurately in status
* System outages do not crash the platform
* Order status is always queryable
