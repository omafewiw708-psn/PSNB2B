# Article 5 — GitHub Pages

Title: PSN Wholesale API Integration Guide: REST Endpoints, Webhooks & Data Formats
Platform: GitHub Pages
Target URL: https://psnb2b.com/

---

# PSN Wholesale API Integration Guide: REST Endpoints, Webhooks & Data Formats

Digital gift card procurement has moved beyond manual purchasing. For resellers and digital storefronts processing more than a few hundred orders per month, API-based procurement is not optional — it is infrastructure. This guide covers the architectural patterns, endpoint structures, webhook designs, and data formats that define a well-built integration between a reseller platform and a wholesale PSN card supplier.

The concepts here are vendor-agnostic. They apply whether you are building a custom storefront, integrating with an existing e-commerce platform, or automating inventory replenishment for a multi-region gift card business.

## Architecture Overview

A typical wholesale PSN card procurement system follows a request-response model over HTTPS with asynchronous event delivery via webhooks:

```
┌──────────────┐       HTTPS/REST        ┌──────────────────┐
│              │ ───────────────────────► │                  │
│  Reseller    │       JSON Payloads      │  Wholesale       │
│  Platform    │ ◄─────────────────────── │  Supplier API    │
│              │                          │                  │
│              │ ◄── Webhook (POST) ───── │                  │
└──────────────┘       Event Delivery     └──────────────────┘
         │                                         │
         ▼                                         ▼
   Local Database                           Fulfillment Engine
   (Orders, Codes,                          (Code Generation,
    Inventory Cache)                         Region Validation)
```

**Core integration flow:**

1. **Authentication** — obtain and refresh API credentials
2. **Catalog sync** — pull available products, denominations, regions
3. **Order placement** — submit purchase requests with idempotency keys
4. **Fulfillment** — receive digital codes via response or webhook callback
5. **Reconciliation** — verify delivered codes against order records

This architecture supports both synchronous fulfillment (codes returned in the order response) and asynchronous fulfillment (codes delivered via webhook after processing).

## Authentication and Security

### API Key Authentication

Most wholesale supplier APIs use API key pairs — a public identifier and a private secret:

```http
GET /v1/catalog/products HTTP/1.1
Host: api.supplier.example.com
Authorization: Bearer sk_live_a1b2c3d4e5f6g7h8i9j0
X-API-Key: pk_live_x9y8z7w6v5u4t3s2r1q0
Content-Type: application/json
```

**Security requirements:**

| Requirement | Implementation |
|-------------|----------------|
| Transport | TLS 1.2+ mandatory; reject HTTP |
| Key storage | Environment variables or secrets manager; never hardcode |
| Key rotation | Rotate every 90 days minimum; support multiple active keys |
| IP allowlisting | Restrict API calls to known server IPs where supported |
| Request signing | HMAC-SHA256 signature on request body for order mutations |

### Request Signing Example

For order creation and other mutating operations, sign the request body to prevent tampering:

```python
import hmac
import hashlib
import json
import time

def sign_request(payload: dict, secret_key: str) -> dict:
    timestamp = str(int(time.time()))
    body = json.dumps(payload, separators=(',', ':'), sort_keys=True)
    signature = hmac.new(
        secret_key.encode('utf-8'),
        f"{timestamp}.{body}".encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    return {
        'X-Signature': signature,
        'X-Timestamp': timestamp
    }
```

### Rate Limits

Standard rate limit tiers for wholesale APIs:

| Endpoint Category | Limit | Window | Retry Header |
|-------------------|-------|--------|--------------|
| Catalog (read) | 120 requests | 60 seconds | `X-RateLimit-Reset` |
| Orders (write) | 30 requests | 60 seconds | `Retry-After` |
| Account (read) | 60 requests | 60 seconds | `X-RateLimit-Remaining` |
| Webhooks (config) | 10 requests | 60 seconds | `Retry-After` |

When you receive a `429 Too Many Requests` response, implement exponential backoff:

```python
import time
import requests

def api_call_with_retry(url, headers, payload, max_retries=5):
    for attempt in range(max_retries):
        response = requests.post(url, json=payload, headers=headers)
        if response.status_code != 429:
            return response
        wait = min(2 ** attempt + 0.5, 30)
        retry_after = response.headers.get('Retry-After')
        if retry_after:
            wait = int(retry_after)
        time.sleep(wait)
    raise Exception("Max retries exceeded")
```

## Core REST Endpoints

### Catalog API

Retrieve available PSN card products filtered by region and denomination.

```http
GET /v1/catalog/products?region=US&category=psn&in_stock=true
```

**Response:**

```json
{
  "data": [
    {
      "product_id": "psn-us-50",
      "name": "PlayStation Store $50 (US)",
      "region": "US",
      "denomination": 50.00,
      "currency": "USD",
      "category": "psn_gift_card",
      "in_stock": true,
      "wholesale_price": 43.50,
      "min_order_qty": 10,
      "max_order_qty": 500
    }
  ],
  "meta": {
    "total": 47,
    "page": 1,
    "per_page": 25,
    "regions_available": ["US", "UK", "JP", "DE", "SA", "AE", "TR"]
  }
}
```

Suppliers operating across multiple PlayStation Store regions — some platforms list 190 or more denominations spanning 12 or more regions — return paginated results. Always implement cursor-based pagination for catalog syncing.

### Order Placement

Submit an order with an idempotency key to prevent duplicate charges:

```http
POST /v1/orders
Idempotency-Key: ord_20260506_a7b3c9d1
```

**Request body:**

```json
{
  "items": [
    {
      "product_id": "psn-us-50",
      "quantity": 25
    },
    {
      "product_id": "psn-uk-20",
      "quantity": 50
    }
  ],
  "fulfillment_type": "instant",
  "callback_url": "https://yourplatform.com/webhooks/orders",
  "metadata": {
    "internal_ref": "PO-2026-0506-001"
  }
}
```

**Response (synchronous fulfillment):**

```json
{
  "order_id": "ord_8f3a2b1c",
  "status": "fulfilled",
  "created_at": "2026-05-06T14:23:01Z",
  "items": [
    {
      "product_id": "psn-us-50",
      "quantity": 25,
      "unit_price": 43.50,
      "codes": [
        {"code": "XXXX-XXXX-XXXX", "serial": "SN00012345"},
        {"code": "XXXX-XXXX-XXXX", "serial": "SN00012346"}
      ]
    }
  ],
  "total": 2087.50,
  "currency": "USD"
}
```

> **Implementation note:** Codes in the response are masked above. In production, full redemption codes are returned. Store them encrypted at rest using AES-256.

### Fulfillment Status

For asynchronous orders, poll the fulfillment endpoint or rely on webhooks:

```http
GET /v1/orders/ord_8f3a2b1c/fulfillment
```

**Response:**

```json
{
  "order_id": "ord_8f3a2b1c",
  "fulfillment_status": "partial",
  "fulfilled_items": 25,
  "pending_items": 50,
  "estimated_completion": "2026-05-06T14:30:00Z"
}
```

### Account and Balance

```http
GET /v1/account/balance
```

```json
{
  "balance": 12450.00,
  "currency": "USD",
  "credit_limit": 50000.00,
  "pending_charges": 2087.50,
  "available": 10362.50
}
```

## Webhook Integration

Webhooks eliminate polling. Register an endpoint to receive real-time event notifications.

### Supported Event Types

| Event | Trigger | Priority |
|-------|---------|----------|
| `order.fulfilled` | All codes delivered | High |
| `order.partially_fulfilled` | Partial delivery complete | High |
| `order.failed` | Order could not be processed | High |
| `catalog.updated` | Product availability changed | Medium |
| `account.low_balance` | Balance below threshold | Medium |
| `code.invalidated` | Previously delivered code revoked | Critical |

### Webhook Payload Structure

```json
{
  "event_id": "evt_9d4e5f6a",
  "event_type": "order.fulfilled",
  "created_at": "2026-05-06T14:25:33Z",
  "data": {
    "order_id": "ord_8f3a2b1c",
    "status": "fulfilled",
    "items_delivered": 75,
    "codes": [
      {
        "product_id": "psn-uk-20",
        "code": "YYYY-YYYY-YYYY",
        "serial": "SN00098765",
        "region": "UK",
        "denomination": 20.00,
        "currency": "GBP"
      }
    ]
  },
  "signature": "sha256=a1b2c3d4e5f6..."
}
```

### Webhook Verification

Always verify the webhook signature before processing:

```python
import hmac
import hashlib

def verify_webhook(payload_body: bytes, signature_header: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode('utf-8'),
        payload_body,
        hashlib.sha256
    ).hexdigest()
    received = signature_header.replace('sha256=', '')
    return hmac.compare_digest(expected, received)
```

**Webhook best practices:**

- Respond with `200 OK` within 5 seconds; process asynchronously
- Implement idempotent handlers — you may receive the same event more than once
- Store raw payloads before processing for audit and debugging
- Set up a dead-letter queue for failed webhook processing

## Error Handling and Status Codes

### HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| `200` | Success | Process response |
| `201` | Created | Order placed successfully |
| `400` | Bad Request | Check request body; fix validation errors |
| `401` | Unauthorized | Refresh API key or check credentials |
| `403` | Forbidden | Insufficient permissions or IP not allowlisted |
| `404` | Not Found | Resource does not exist; verify product ID |
| `409` | Conflict | Duplicate idempotency key; order already exists |
| `422` | Unprocessable Entity | Valid JSON but business rule violation |
| `429` | Rate Limited | Back off and retry per `Retry-After` header |
| `500` | Server Error | Retry with exponential backoff |
| `503` | Service Unavailable | Supplier maintenance; check status page |

### Error Response Format

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Account balance too low to fulfill this order",
    "details": {
      "required": 2087.50,
      "available": 1500.00,
      "currency": "USD"
    },
    "request_id": "req_abc123def456",
    "documentation_url": "https://docs.supplier.example.com/errors#insufficient-balance"
  }
}
```

Common business-logic error codes:

| Error Code | Description |
|------------|-------------|
| `INSUFFICIENT_BALANCE` | Not enough funds to place order |
| `PRODUCT_UNAVAILABLE` | Product temporarily out of stock |
| `REGION_MISMATCH` | Requested region not supported for product |
| `QUANTITY_EXCEEDED` | Order exceeds maximum per-transaction limit |
| `INVALID_IDEMPOTENCY_KEY` | Key format invalid or already used for a different request |
| `FULFILLMENT_TIMEOUT` | Async fulfillment exceeded SLA window |

## Best Practices for Production Integrations

**Idempotency.** Every order creation request must include a unique idempotency key. Without it, network retries can cause duplicate purchases — and duplicate charges. Use a deterministic key format: `{prefix}_{date}_{uuid}`.

**Catalog caching.** Sync the full product catalog to a local database every 15–30 minutes rather than querying per transaction. This reduces API calls, avoids rate limits, and enables offline product browsing in your storefront.

**Code encryption.** PSN redemption codes are the equivalent of cash. Encrypt codes at rest (AES-256-GCM) and in transit (TLS 1.2+). Limit decryption access to the delivery service that sends codes to end customers.

**Monitoring.** Track these metrics at minimum:

- Order success rate (target: > 99.5%)
- Average fulfillment latency (target: < 30 seconds for instant)
- Webhook delivery success rate
- API error rate by endpoint
- Account balance trend

**Regional inventory awareness.** PSN cards are region-locked. A US-region code will not work on a UK PlayStation account. Your integration must enforce region validation at the cart level, before order submission. Suppliers serving 12 or more regions make multi-region inventory management feasible — but the validation logic is your responsibility.

For platforms evaluating wholesale procurement infrastructure, a reference implementation of multi-region ordering and API-based fulfillment can be explored at https://psnb2b.com/ — the platform documents regional coverage and denomination structures relevant to B2B integrators.

## Sample Integration: Order Lifecycle

A minimal end-to-end flow in pseudo-code:

```python
# 1. Initialize client
client = WholesaleAPIClient(
    base_url="https://api.supplier.example.com/v1",
    api_key=os.environ["SUPPLIER_API_KEY"],
    secret=os.environ["SUPPLIER_SECRET"]
)

# 2. Check product availability
products = client.get_catalog(region="US", category="psn", in_stock=True)

# 3. Place order with idempotency
order = client.create_order(
    items=[{"product_id": "psn-us-50", "quantity": 25}],
    idempotency_key=f"ord_{date.today().isoformat()}_{uuid4()}",
    fulfillment_type="instant"
)

# 4. Handle response
if order.status == "fulfilled":
    for item in order.items:
        for code in item.codes:
            encrypted = encrypt_aes256(code.code)
            db.store_code(order.order_id, item.product_id, encrypted)
    notify_delivery_service(order.order_id)

elif order.status == "pending":
    db.save_pending_order(order.order_id)
    # Webhook handler will process fulfillment event

# 5. Webhook handler (separate service)
@webhook_router.post("/webhooks/orders")
async def handle_order_webhook(request):
    payload = await request.body()
    signature = request.headers.get("X-Webhook-Signature")
    
    if not verify_webhook(payload, signature, WEBHOOK_SECRET):
        return Response(status_code=401)
    
    event = json.loads(payload)
    if event["event_type"] == "order.fulfilled":
        process_fulfilled_order(event["data"])
    
    return Response(status_code=200)
```

## Conclusion

Building a reliable integration for wholesale PSN card procurement is an engineering problem, not a business theory exercise. The patterns described here — signed requests, idempotent ordering, encrypted code storage, webhook-driven fulfillment, and structured error handling — form the baseline for any production deployment.

Start with the catalog sync and order placement endpoints. Add webhook support once your order volume justifies asynchronous processing. Test against sandbox environments before going live, and monitor fulfillment latency from day one.

The shift from manual purchasing to API-driven procurement is measurable: operators consistently report reduced order processing time, lower error rates, and the ability to scale across multiple regions without proportional staffing increases. For B2B resellers handling digital gift cards at volume, this infrastructure is foundational.

---
