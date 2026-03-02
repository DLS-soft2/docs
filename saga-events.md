# Saga & Kafka Event Flow

## Pattern

Choreography-based saga — no central orchestrator. Kafka drives the workflow, services react to events. Order Service is the **saga state holder**: it owns the order record and keeps its status in sync by consuming events from the other participants.

## Topics

| Topic | Publisher | Consumers |
|---|---|---|
| `orders` | Order Service | Payment Service, Notification Service |
| `payments` | Payment Service | Restaurant Service, Order Service, Notification Service |
| `restaurants` | Restaurant Service | Courier Service, Order Service, Notification Service |
| `couriers` | Courier Service | Order Service, Notification Service |
| `deliveries` | Courier Service | Order Service, Notification Service |

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

## Failure Path

If payment fails, Payment Service publishes `PaymentFailed` to the `payments` topic. Order Service consumes it and sets status to `CANCELLED`. No further saga steps are triggered.

## Outside the Kafka Flow

- **AI Service** — called synchronously by Courier Service for ETA and assignment scoring
- **User Profile Service** — REST/GraphQL only, no event-driven responsibilities
- **Notification Service** — consumer only, subscribes to all topics and pushes notifications to users at each saga step

## Diagram

See kafka-event-flow diagram for a visual overview.
