# Saleor System Design Study

## Goal

Understand Saleor 3.23 by running the platform, exercising its GraphQL API, tracing the backend code, and completing an order from product discovery to fulfillment.

## Repositories

- `saleor-study` contains `saleor-platform`, which orchestrates the complete system with Docker Compose.
- `saleor-backend-source` contains the Saleor Python/Django backend source for code study.
- The platform runs prebuilt Saleor API and Dashboard images tagged `3.23`.

## Runtime Architecture

| Service | Port | Responsibility |
| --- | ---: | --- |
| Dashboard | 9000 | Staff interface for products, orders, and fulfillment |
| API | 8000 | GraphQL endpoint and commerce business logic |
| PostgreSQL | 5432 | Persistent commerce data |
| Valkey | 6379 | Cache and Celery task broker |
| Worker | None | Background and scheduled tasks |
| Mailpit | 8025 | Local email capture and inspection |
| Jaeger | 16686 | Distributed request tracing |

Containers communicate using Docker service names such as `db`, `cache`, `mailpit`, and `jaeger`, rather than `localhost`.

## GraphQL Fundamentals

Saleor primarily exposes commerce through one endpoint:

```text
http://localhost:8000/graphql/
```

- A query reads data.
- A mutation validates and changes data.
- Clients request only the fields they need.
- `edges` and `node` are Relay-style pagination structures.
- GraphQL IDs are opaque global identifiers and should be passed back unchanged.
- `requestedQueryCost` measures query complexity, not execution time.

The request path is:

```text
GraphQL operation
-> schema field
-> resolver and business rules
-> Django ORM
-> PostgreSQL
-> GraphQL response
```

## Commerce Model

```text
Product
-> Product variant
-> Channel-specific price
-> Stock record
-> Warehouse
```

- A product contains shared presentation data such as name and slug.
- A variant is the purchasable item, such as a particular shirt size.
- An SKU identifies a variant for inventory and operations.
- Prices and availability can differ by sales channel.
- Stock belongs to variants in warehouses, not directly to products.
- `quantityAvailable` is customer-facing and may be capped by the maximum quantity allowed per checkout. It intentionally does not always reveal exact warehouse stock.

## Checkout Lifecycle Tested

1. Queried products, variants, prices, and inventory.
2. Created a checkout containing one small Monospace Tee.
3. Added Canadian shipping and billing addresses.
4. Retrieved delivery methods based on the address and channel.
5. Selected EMS shipping.
6. Created a `$54.20 USD` payment with the local dummy gateway.
7. Completed the checkout and created paid order `21`.
8. Confirmed payment and fulfillment are separate states.
9. Fulfilled the order through the Dashboard with a fictional tracking number.

The checkout calculation was:

```text
Product subtotal: $20.00 USD
EMS shipping:     $34.20 USD
Order total:      $54.20 USD
```

Checkout completion is an important transactional boundary. Saleor revalidates pricing, addresses, delivery, payment, and inventory before converting the temporary checkout into a permanent order.

## Reliability Concepts

- Checkout writes use database transactions so related changes succeed together or roll back together.
- Order creation allocates inventory; fulfillment records the physical movement of stock.
- Payment and fulfillment have separate statuses because charging a customer does not mean an item has shipped.
- Valkey supports caching and queues background work for Celery workers.
- Webhooks allow external systems to react to commerce events.
- Jaeger traces show total request duration and nested spans, helping identify errors, slow operations, and repeated database work.
- Saleor DataLoaders batch related database reads to reduce GraphQL N+1 query problems.

## Important Backend Files

These paths are relative to the separate `saleor-backend-source` folder:

| File | Purpose |
| --- | --- |
| `saleor/graphql/product/schema.py` | Product GraphQL fields and top-level resolvers |
| `saleor/graphql/product/resolvers.py` | Product query construction and visibility rules |
| `saleor/graphql/product/types/products.py` | Product and variant GraphQL types |
| `saleor/product/models.py` | Django product models stored in PostgreSQL |
| `saleor/graphql/checkout/mutations/checkout_create.py` | Checkout creation and validation |
| `saleor/checkout/models.py` | Checkout and checkout-line persistence models |
| `saleor/graphql/checkout/mutations/checkout_complete.py` | Checkout-to-order transition |

## Saleor Compared With Medusa

| Area | Saleor | Medusa |
| --- | --- | --- |
| Primary API | GraphQL | REST and SDK |
| Backend stack | Python and Django | TypeScript and Node.js |
| Business organization | Schemas, resolvers, services, and Django models | Modules, services, and workflows |
| Response shape | Client-selected nested fields | Endpoint responses with selected expansions |
| Strong fit | Complex multichannel GraphQL commerce | TypeScript teams and modular custom commerce |

Both platforms use the same fundamental ecommerce concepts: products, variants, prices, inventory, carts or checkouts, payments, orders, and fulfillment.

## Useful Commands

Run these from the `saleor-study` folder:

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=30 worker
docker compose stop
```

- `up -d` starts the stack in the background.
- `ps` shows container status and port mappings.
- `logs` helps diagnose a service.
- `stop` pauses containers without deleting their data.

## Decision Rule

```text
Standard store               -> hosted platform such as Shopify
Custom TypeScript commerce   -> consider Medusa
Complex multichannel GraphQL -> consider Saleor
```

The final decision should be tested against the business's hardest requirement, such as regional pricing, wholesale rules, subscriptions, or fulfillment integrations.
