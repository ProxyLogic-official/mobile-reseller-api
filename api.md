# ProxyLogic Mobile API reseller guide


## Connection details

| Item | Value |
| --- | --- |
| API base URL | `https://api.mobile.proxylogic.org` |
| Authentication | `X-API-Key: YOUR_API_KEY` |
| Request/response format | JSON, except requests with no body |
| Proxy hostname returned in orders | `mobile.proxylogic.org` |
| Proxy ports | Always returned as an integer array |

Keep the API key on your website backend. Do not expose it in browser JavaScript, mobile apps,
public repositories, frontend environment variables, or screenshots.

## Common headers

Every request:

```http
X-API-Key: pl_live_REPLACE_ME
```

Every `POST` and `PATCH` request:

```http
Content-Type: application/json
Idempotency-Key: your-unique-operation-key
```

The idempotency key must be 8â€“100 characters. Generate one stable key for each checkout or
user action. Retrying the same request with the same key is safe. Reusing a key with a changed
body returns `409 IDEMPOTENCY_CONFLICT`.

Every response includes `X-Request-ID`. Store it with failed requests so ProxyLogic support can
trace the request.

## Asynchronous operations

Purchases, extensions, cancellations, and settings changes normally return `202 Accepted`:

```json
{
  "operation": {
    "id": "2dd44902-e94e-4463-bd67-f881d75f4668",
    "kind": "PURCHASE",
    "status": "PENDING",
    "result": null,
    "error_code": null,
    "error_message": null,
    "created_at": "2026-07-15T12:00:00Z",
    "completed_at": null
  },
  "message": "The operation is still processing. Poll the operation URL for its result."
}
```

Poll `GET /v1/operations/{operation.id}` until the status is final. Suggested polling delays
are 1, 2, 4, 8, then 10 seconds with small random jitter. Stop after `SUCCEEDED` or `FAILED`.
Do not submit a second purchase with a new idempotency key while the first one is pending.

Operation statuses:

| Status | Meaning | Client action |
| --- | --- | --- |
| `PENDING` | Accepted and waiting/processing | Continue polling |
| `UNCERTAIN` | Delayed; automatic recovery is running | Continue polling |
| `NEEDS_REVIEW` | Support review is required | Stop retrying and contact support |
| `SUCCEEDED` | Finished successfully | Read `result` |
| `FAILED` | Finished unsuccessfully | Read `error_code` and `error_message` |

An idempotent replay of an already completed mutation can return its final result directly with
`200 OK`.

## Endpoint summary

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/v1/account` | Balance and current mobile prices |
| `GET` | `/v1/account/ledger` | Money ledger |
| `GET` | `/v1/mobile/availability/locations` | Location availability |
| `GET` | `/v1/mobile/availability/carriers` | Carrier availability |
| `POST` | `/v1/mobile/orders` | Purchase mobile ports |
| `GET` | `/v1/mobile/orders` | List your orders |
| `GET` | `/v1/mobile/orders/{order_id}` | Get one order |
| `POST` | `/v1/mobile/orders/{order_id}/extend` | Extend an order |
| `POST` | `/v1/mobile/orders/{order_id}/cancel` | Cancel an eligible order |
| `PATCH` | `/v1/mobile/orders/{order_id}/settings/whitelist` | Change whitelist IP |
| `PATCH` | `/v1/mobile/orders/{order_id}/settings/protocol` | Change protocol |
| `PATCH` | `/v1/mobile/orders/{order_id}/settings/rotation` | Change rotation |
| `PATCH` | `/v1/mobile/orders/{order_id}/settings/location` | Change location |
| `GET` | `/v1/operations/{operation_id}` | Poll an operation |

## Account balance and pricing

```bash
curl -sS https://api.mobile.proxylogic.org/v1/account \
  -H "X-API-Key: CUSTOMER_API_KEY"
```

Response â€” `200 OK`:

```json
{
  "balance": "350.0000",
  "currency": "USD",
  "pricing": {
    "mobile_per_port_day": "2.5000"
  }
}
```

Purchase charge:

```text
ports Ã— days Ã— current per-port/day price
```

The complete charge is reserved when a purchase or extension is accepted.

## Account ledger

`limit` is optional, defaults to `50`, and accepts `1`â€“`200`.

```bash
curl -sS "https://api.mobile.proxylogic.org/v1/account/ledger?limit=50" \
  -H "X-API-Key: CUSTOMER_API_KEY"
```

Response â€” `200 OK`:

```json
{
  "entries": [
    {
      "id": 42,
      "order_id": "9cb0876f-d73d-49bb-8237-b30ab7b09872",
      "type": "CANCELLATION_REFUND",
      "amount": "75.0000",
      "balance_after": "425.0000",
      "created_at": "2026-07-15T12:10:00Z"
    }
  ]
}
```

Negative amounts are debits/reservations; positive amounts are credits/releases/refunds.

## Availability

Locations:

```bash
curl -sS https://api.mobile.proxylogic.org/v1/mobile/availability/locations \
  -H "X-API-Key: CUSTOMER_API_KEY"
```

```json
{
  "locations": [
    {
      "name": "California",
      "code": "CA",
      "available_ports": 30
    }
  ]
}
```

Carriers:

```bash
curl -sS https://api.mobile.proxylogic.org/v1/mobile/availability/carriers \
  -H "X-API-Key: CUSTOMER_API_KEY"
```

```json
{
  "carriers": [
    {
      "carrier": "TMOBILE",
      "available_ports": 12,
      "protocols": {"https": 5, "socks5": 7}
    }
  ]
}
```

Availability changes continuously and is not a reservation.

## Purchase mobile ports

```bash
curl -sS -X POST https://api.mobile.proxylogic.org/v1/mobile/orders \
  -H "X-API-Key: CUSTOMER_API_KEY" \
  -H "Idempotency-Key: checkout-84721" \
  -H "Content-Type: application/json" \
  -d '{
    "ports":2,
    "days":30,
    "protocol":"SOCKS5",
    "whitelist_ip":"203.0.113.25",
    "ip_rotation":true,
    "location":"CA"
  }'
```

Request fields:

| Field | Accepted values |
| --- | --- |
| `ports` | Integer `1`â€“`25` |
| `days` | Integer `1`â€“`31` |
| `protocol` | `HTTPS` or `SOCKS5` |
| `whitelist_ip` | Public IPv4 address used to access the ports |
| `ip_rotation` | Boolean; `true` selects normal rotation |
| `location` | `RANDOM` or two-letter US state code |

Every purchase is 4G mobile with random-carrier allocation. `connection_type` and `carrier` are
internal fixed settings and must not be sent in the reseller request. A specific location can be
selected during purchase. Location
can be changed after activation at minimum 30-minute intervals; carrier cannot be changed after
placement.

Poll the returned operation. A successful purchase has an order object in `result`:

```json
{
  "id": "9cb0876f-d73d-49bb-8237-b30ab7b09872",
  "status": "ACTIVE",
  "connection_type": "4G_MOBILE",
  "ports": 2,
  "original_days": 30,
  "protocol": "SOCKS5",
  "whitelist_ip": "203.0.113.25",
  "rotation": "TRUE",
  "rotation_time": "30",
  "location": "CA",
  "carrier": "RANDOM",
  "original_charge": "150.0000",
  "extension_charges": "0.0000",
  "refund_total": "0.0000",
  "date_created": "2026-07-15T12:00:03Z",
  "date_end": "2026-08-14T12:00:03Z",
  "proxy": {
    "host": "mobile.proxylogic.org",
    "ports": [22001, 22002]
  }
}
```

Use every returned port. Never assume ports are consecutive or returned as a single value.

## List orders

Optional query parameters:

- `status`: for example `ACTIVE` or `CANCELED`.
- `limit`: `1`â€“`200`, default `100`.

```bash
curl -sS "https://api.mobile.proxylogic.org/v1/mobile/orders?status=ACTIVE&limit=100" \
  -H "X-API-Key: CUSTOMER_API_KEY"
```

Response â€” `200 OK`: a JSON array of order objects. An account with no matching orders returns
`[]`.

## Get one order

```bash
curl -sS https://api.mobile.proxylogic.org/v1/mobile/orders/ORDER_ID \
  -H "X-API-Key: CUSTOMER_API_KEY"
```

Add `?refresh=true` to request a live refresh. If live refresh is temporarily unavailable, the
last stored order is returned with `X-Data-Stale: true`.

## Extend an order

```bash
curl -sS -X POST https://api.mobile.proxylogic.org/v1/mobile/orders/ORDER_ID/extend \
  -H "X-API-Key: CUSTOMER_API_KEY" \
  -H "Idempotency-Key: extend-ORDER_ID-7-days" \
  -H "Content-Type: application/json" \
  -d '{"days":7}'
```

`days` accepts `1`â€“`31`. The current applicable price is used. Successful operation result:

```json
{
  "order_id": "9cb0876f-d73d-49bb-8237-b30ab7b09872",
  "days_added": 7,
  "charged": "35.0000",
  "new_date_end": "2026-08-21T12:00:03Z"
}
```

## Cancel an order

```bash
curl -sS -X POST https://api.mobile.proxylogic.org/v1/mobile/orders/ORDER_ID/cancel \
  -H "X-API-Key: CUSTOMER_API_KEY" \
  -H "Idempotency-Key: cancel-ORDER_ID"
```

Cancellation rules:

- The original order length must be 30 or 31 days.
- Cancellation must be requested during the first 24 hours.
- A confirmed cancellation credits 50% of the original purchase charge.
- Extension charges are not refunded.
- Replaying the same cancellation cannot credit the refund twice.

Successful operation result:

```json
{
  "order_id": "9cb0876f-d73d-49bb-8237-b30ab7b09872",
  "status": "CANCELED",
  "refund_credited": "75.0000"
}
```

## Update whitelist IPv4

```bash
curl -sS -X PATCH \
  https://api.mobile.proxylogic.org/v1/mobile/orders/ORDER_ID/settings/whitelist \
  -H "X-API-Key: CUSTOMER_API_KEY" \
  -H "Idempotency-Key: whitelist-ORDER_ID-2" \
  -H "Content-Type: application/json" \
  -d '{"whitelist_ip":"203.0.113.26"}'
```

Successful result:

```json
{
  "order_id": "9cb0876f-d73d-49bb-8237-b30ab7b09872",
  "whitelist_ip": "203.0.113.26",
  "locked_until": "2026-07-15T12:30:00Z"
}
```

## Update protocol

```bash
curl -sS -X PATCH \
  https://api.mobile.proxylogic.org/v1/mobile/orders/ORDER_ID/settings/protocol \
  -H "X-API-Key: CUSTOMER_API_KEY" \
  -H "Idempotency-Key: protocol-ORDER_ID-2" \
  -H "Content-Type: application/json" \
  -d '{"protocol":"HTTPS"}'
```

Successful result:

```json
{
  "order_id": "9cb0876f-d73d-49bb-8237-b30ab7b09872",
  "protocol": "HTTPS"
}
```

Protocol can normally be changed only once per order.

## Update rotation

```bash
curl -sS -X PATCH \
  https://api.mobile.proxylogic.org/v1/mobile/orders/ORDER_ID/settings/rotation \
  -H "X-API-Key: CUSTOMER_API_KEY" \
  -H "Idempotency-Key: rotation-ORDER_ID-2" \
  -H "Content-Type: application/json" \
  -d '{"rotation":"10"}'
```

`rotation` accepts only `TRUE` or `FALSE`. `TRUE` means standard 30-minute rotation; `FALSE`
means extended rotation.

```json
{
  "order_id": "9cb0876f-d73d-49bb-8237-b30ab7b09872",
  "rotation": "10",
  "next_rotation": "2026-07-15T12:10:00Z"
}
```

## Update location

```bash
curl -sS -X PATCH \
  https://api.mobile.proxylogic.org/v1/mobile/orders/ORDER_ID/settings/location \
  -H "X-API-Key: CUSTOMER_API_KEY" \
  -H "Idempotency-Key: location-ORDER_ID-2" \
  -H "Content-Type: application/json" \
  -d '{"location":"FL"}'
```

Use `RANDOM` or a two-letter US state code.

```json
{
  "order_id": "9cb0876f-d73d-49bb-8237-b30ab7b09872",
  "location": "FL",
  "locked_until": "2026-07-15T12:30:00Z"
}
```

Whitelist and location changes may be temporarily locked. Wait until `locked_until` before
submitting another change.

## Poll an operation

```bash
curl -sS https://api.mobile.proxylogic.org/v1/operations/OPERATION_ID \
  -H "X-API-Key: CUSTOMER_API_KEY"
```

Successful purchase operation example:

```json
{
  "id": "2dd44902-e94e-4463-bd67-f881d75f4668",
  "kind": "PURCHASE",
  "status": "SUCCEEDED",
  "result": {"id":"9cb0876f-d73d-49bb-8237-b30ab7b09872","proxy":{"host":"mobile.proxylogic.org","ports":[22001,22002]}},
  "error_code": null,
  "error_message": null,
  "created_at": "2026-07-15T12:00:00Z",
  "completed_at": "2026-07-15T12:00:03Z"
}
```

## Errors

Error envelope:

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Insufficient account balance.",
    "request_id": "f8c77149029ea4129ac97811"
  }
}
```

| HTTP | Common codes | Action |
| --- | --- | --- |
| `400` | `IDEMPOTENCY_KEY_REQUIRED`, `INVALID_IDEMPOTENCY_KEY` | Add/fix the idempotency header |
| `401` | `INVALID_API_KEY` | Check/rotate the API key |
| `402` | `INSUFFICIENT_BALANCE` | Add funds or reduce the order |
| `403` | `ACCOUNT_DISABLED` | Contact ProxyLogic support |
| `404` | `ORDER_NOT_FOUND`, `OPERATION_NOT_FOUND` | Check the public ID and API key |
| `409` | `IDEMPOTENCY_CONFLICT`, `ORDER_OPERATION_PENDING`, `SETTING_LOCKED`, `CANCEL_NOT_ALLOWED` | Fix the request state; do not blindly retry |
| `422` | `VALIDATION_ERROR`, `LOCATION_UNAVAILABLE` | Correct request fields |
| `429` | `RATE_LIMITED` | Honor `Retry-After` and use jitter |
| `503` | `QUEUE_SATURATED`, `SERVICE_UNAVAILABLE`, `AVAILABILITY_UNAVAILABLE`, `ORDER_PROCESSING_ERROR` | Honor `Retry-After`; retry with backoff |
| `500` | `INTERNAL_ERROR` | Retry once, then contact support with `X-Request-ID` |

Example processing failure:

```json
{
  "error": {
    "code": "ORDER_PROCESSING_ERROR",
    "message": "We couldn't complete your request. Contact ProxyLogic support.",
    "request_id": "f8c77149029ea4129ac97811"
  }
}
```

Example validation failure:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more request fields are invalid.",
    "request_id": "f8c77149029ea4129ac97811",
    "fields": [
      {"field":"ports","message":"Input should be less than or equal to 25"}
    ]
  }
}
```

## Minimal website integration flow

1. Fetch `/v1/account` and availability from your backend.
2. Calculate/display the current customer charge.
3. Generate one idempotency key for checkout.
4. Submit the purchase once.
5. Save the operation ID and idempotency key in your database.
6. Poll the operation from a background job.
7. When successful, save the public order ID, hostname, and every returned port.
8. On retryable errors, apply exponential backoff with jitter.
