# Functional & Non-Functional Requirements

## Functional Requirements

### Authentication & Identity

- Users can register and log in via Keycloak (OAuth2/OIDC)
- Users receive a JWT on login, used for all subsequent requests
- Roles are enforced per service (customer, restaurant owner, courier, admin)

### User Profiles

- Users can create and update their profile (display name, phone, addresses, preferences)
- Addresses can be saved and reused at checkout

### Restaurants

- Restaurant owners can manage their restaurant (name, description, opening hours, availability)
- Restaurant owners can manage their menu (add, update, delete items and categories)
- Customers can browse restaurants and view menus
- Restaurant owners can accept or reject incoming orders

### Ordering

- Customers can place an order from a single restaurant
- Customers can view live order status throughout the saga lifecycle
- Customers can cancel an order before it has been accepted by the restaurant
- Orders are automatically cancelled with a full refund if no courier picks up the order
- The system maintains order history per customer

### Payments

- The system authorises payment when an order is placed
- Payment is idempotent — placing the same order twice does not result in a double charge
- On payment failure, the order is cancelled and the customer is notified
- Refunds are issued automatically on valid cancellations

### Courier & Delivery

- Couriers can register and toggle their availability
- The system automatically assigns an available courier when a restaurant accepts an order
- Couriers can update delivery status (picked up, delivered)

### Notifications

- Customers receive real-time notifications at each stage of the order lifecycle
- Notifications are delivered via WebSocket (in-app) and push/email

---

## Non-Functional Requirements

### Scalability

Each microservice scales independently. Kafka decouples producers and consumers, allowing high-throughput event processing without bottlenecks at any single service.

### Performance

- Order placement and status queries must have low latency
- Real-time status updates must be delivered with minimal delay via WebSocket

### Security

- All external requests are authenticated at the API Gateway (JWT validation via OAuth2/JWT)
- Services enforce authorisation via shared RBAC libraries — no service trusts the caller blindly
- User data is encrypted and handled in compliance with GDPR

### Fault Tolerance

- Saga compensation handles failures — e.g. payment failure triggers order cancellation
- Services are stateless where possible; state lives in their own databases

### Observability

- Prometheus collects metrics from all services
- Grafana provides dashboards and alerting for production monitoring

### Maintainability

- Shared CI/CD workflows enforce consistent quality gates across all services
- Shared auth libraries keep RBAC logic in one place
- SemVer enforced across all services via CI

### Data Consistency

The system uses eventual consistency via choreography-based saga. Each service owns its data — no shared databases across service boundaries.
