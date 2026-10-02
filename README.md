# Acme Orders API

Order lifecycle service for the Acme commerce platform. Owns the order state
machine from checkout through fulfilment and cancellation.

- **Spec:** [`openapi.yaml`](./openapi.yaml)
- **Internal endpoint:** `https://orders.internal.acme.example/v1`
- **Owner:** Acme Platform Team

## Dependency map

This file is the human-readable source of truth for the service dependency
graph. Postman Agent Context infers edges from the OpenAPI spec, the repository
contents, and linked collections — keeping the relationships stated in plain
language here makes the graph resolvable regardless of inference strategy.

### Upstream — services that call Orders

| Caller | Operations consumed |
|---|---|
| `acme-gateway` | `POST /orders`, `GET /orders/{orderId}`, `POST /orders/{orderId}/cancel` |

### Downstream — services Orders calls

| Dependency | Operation | Purpose | Failure mode |
|---|---|---|---|
| `acme-payments-api` | `POST /v1/charges` | Authorise payment at checkout | Order not created; returns `402` |
| `acme-payments-api` | `POST /v1/charges/{chargeId}/refunds` | Refund on cancellation | Cancel fails; order stays `paid` |
| `acme-inventory-api` | `POST /v1/reservations` | Hold stock for a pending order | Order not created; returns `409` |
| `acme-inventory-api` | `DELETE /v1/reservations/{reservationId}` | Release stock on cancellation | Stock stays held; needs reconciliation job |
| SendGrid *(external)* | `POST /v3/mail/send` | Email order receipts | Non-blocking; receipt delayed |

### Data stores

| Store | Engine | Tables |
|---|---|---|
| `acme_orders` | PostgreSQL 15 | `orders`, `order_items`, `order_events` |

## Blast radius

`POST /orders` is the highest-fanout operation in the platform — a single
synchronous request touches **Payments** and **Inventory** before responding. A
breaking change to `CreateOrderRequest` cascades to:

1. `acme-gateway` (public contract — the edge forwards this body verbatim)
2. The web client checkout flow
3. Any consumer of the `order.created` event

## Running the Postman CLI here

```bash
postman init                 # generates postman/ + .postman/resources.yaml
postman workspace create     # or connect an existing workspace
git add .postman/resources.yaml && git commit -m "link workspace"
```

`postman init` adopts `openapi.yaml` at the repo root automatically. Do **not**
create the `postman/` directory by hand — `init` refuses to run if it already
contains source code.
