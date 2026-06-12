# Saga & Kafka Event Flow

## Pattern

Choreography-based saga — no central orchestrator. Kafka drives the workflow, services react to events. Order Service is the **saga state holder**: it owns the order record and keeps its status in sync by consuming events from the other participants.

## Topics

| Topic | Publisher | Consumers | Events |
|---|---|---|---|
| `orders` | Order Service | Payment Service, Notification Service | OrderCreated |
| `payments` | Payment Service | Order Service, Restaurant Service, Notification Service | PaymentAuthorized, PaymentFailed, PaymentRefunded |
| `restaurants` | Restaurant Service | Courier Service, Order Service, Notification Service, Payment Service | RestaurantAccepted, RestaurantRejected |
| `couriers` | Courier Service | Order Service, Notification Service, Payment Service | CourierAssigned, CourierAssignmentFailed |
| `deliveries` | Courier Service | Order Service, Notification Service | DeliveryCompleted |

## Happy Path

```
Order Service  →  orders        →  Payment Service
               →  payments      →  Restaurant Service
               →  restaurants   →  Courier Service
               →  couriers/deliveries
```

Each hop updates the order status in Order Service:

| Event | Order status |
|---|---|
| OrderCreated | `PENDING` |
| PaymentAuthorized | `PAID` |
| RestaurantAccepted | `PREPARING` |
| CourierAssigned | `OUT_FOR_DELIVERY` |
| DeliveryCompleted | `DELIVERED` |

## Order Status Transitions

| Event | Transition |
|---|---|
| OrderCreated | → `PENDING` |
| PaymentAuthorized | `PENDING` → `PAID` |
| PaymentFailed | any → `CANCELLED` |
| RestaurantAccepted | `PAID` → `PREPARING` |
| RestaurantRejected | `PAID` → `CANCELLED` |
| CourierAssigned | `PREPARING` → `OUT_FOR_DELIVERY` |
| CourierAssignmentFailed | `PREPARING` → `CANCELLED` |
| DeliveryCompleted | `OUT_FOR_DELIVERY` → `DELIVERED` |
| PaymentRefunded | any non-terminal → `CANCELLED` |

## Failure Paths

### PaymentFailed
Payment Service publishes `PaymentFailed` to the `payments` topic. Order Service sets status to `CANCELLED`. No further saga steps.

### RestaurantRejected (compensating action)
Restaurant staff rejects a paid order via REST. Restaurant Service publishes `RestaurantRejected` to `restaurants`.
```
RestaurantRejected (restaurants topic)
  → Payment Service: AUTHORIZED → REFUNDED, publishes PaymentRefunded
  → Order Service: PAID → CANCELLED
  → Notification Service: notifies customer
```

### CourierAssignmentFailed (compensating action)
No couriers available. Courier Service publishes `CourierAssignmentFailed` to `couriers`.
```
CourierAssignmentFailed (couriers topic)
  → Payment Service: AUTHORIZED → REFUNDED, publishes PaymentRefunded
  → Order Service: PREPARING → CANCELLED
  → Notification Service: notifies customer
```

### PaymentRefunded (compensating follow-up)
Payment Service refunds after RestaurantRejected or CourierAssignmentFailed and publishes `PaymentRefunded` to `payments`.
```
PaymentRefunded (payments topic)
  → Order Service: → CANCELLED (if not already)
  → Notification Service: notifies customer of refund
```

## Restaurant Human-in-the-Loop

Restaurant Service no longer auto-accepts orders. On `PaymentAuthorized`, it saves a `PendingOrder`. Restaurant staff use REST endpoints to accept or reject:

```
PaymentAuthorized → restaurant-service saves PendingOrder
GET  /api/v2/restaurants/orders/pending              → list pending orders
POST /api/v2/restaurants/orders/{orderId}/accept      → RestaurantAccepted (saga continues)
POST /api/v2/restaurants/orders/{orderId}/reject      → RestaurantRejected (saga compensates)
```

## Outside the Kafka Flow

- **AI Service** — called synchronously by Courier Service for ETA and assignment scoring
- **User Profile Service** — REST/GraphQL only, no event-driven responsibilities
- **Notification Service** — consumer only, subscribes to all topics and pushes notifications to users at each saga step

## Diagram

See [kafka-event-flow.png](diagrams/kafka-event-flow.png) for a visual overview.
