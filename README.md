# DLS-2

## Project

We are building a food delivery platform as a distributed system of microservices. The domain covers the full lifecycle of a food order: browsing restaurants, placing and paying for an order, restaurant preparation, courier assignment, and delivery — with real-time notifications at each step.

---

## Technology Stack

| Layer                   | Choice                         | Reason                                                                                                                                               |
| ----------------------- | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Backend languages       | Python, Java                   | Python for FastAPI services, Java (Spring Boot / Quarkus) for services where it fits                                                                 |
| Frontend                | Angular or React               | Not yet decided                                                                                                                                      |
| Service frameworks      | FastAPI, Spring Boot / Quarkus | FastAPI for lightweight Python services, Spring Boot / Quarkus for Java services                                                                     |
| Message broker          | Kafka                          | Seems like the industry standard for event-driven microservices + we have experience with it and want to get better at it, as it is relevant at work |
| Primary databases       | PostgreSQL                     | Relational — fits the structured, transactional data in Order, Payment, Courier, and User Profile services                                           |
| Document store          | MongoDB                        | Schema flexibility for Restaurant Service (menus, hours, availability change shape frequently)                                                       |
| Cache / real-time store | Redis                          | Used by Notification Service for message queuing                                                                                                     |
| Identity provider       | Keycloak                       | Handles OAuth2/OIDC, JWT issuance, and role management — keeps auth out of the services themselves                                                   |
| CI/CD                   | GitHub Actions                 | Works well with GitHub actions - we can have shared workflows across all repos                                                                       |
| Container registry      | GHCR                           | Works well with GitHub actions and it is free                                                                                                        |
| Cloud provider          | Azure                          | We have student credtis :D                                                                                                                           |
| Logging & monitoring    | Prometheus + Grafana           | Industry standard for metrics collection and dashboarding                                                                                            |

---

## System Architecture

The system is structured as independent microservices behind an API Gateway, with Kafka as the message broker and Keycloak as the identity provider. See [system architecture diagram](diagrams/first-version-no-infra.png) for a visual overview.

**Services:**

- **API Gateway** — single entry point that validates JWTs and routes requests via REST, GraphQL, and WebSocket
- **Order Service** — manages the order lifecycle and holds the saga state
- **Payment Service** — handles payment authorization, capture, and refunds (simulated ofc)
- **Restaurant Service** — manages menus, hours, and availability
- **Courier Service** — handles courier assignment and delivery tracking
- **User Profile Service** — stores domain user data (addresses, preferences) linked to Keycloak user IDs
- **Notification Service** — subscribes to all Kafka topics and pushes notifications to users
- **AI Service** — We still need to figure out what exactly this will do

**Frontend interaction:**
The frontend communicates with the API Gateway over REST and GraphQL for standard operations, and WebSocket for real-time delivery tracking and notifications. The frontend authenticates directly with Keycloak (OIDC) and passes the resulting JWT to the gateway on every request.

**Communication channels:**

- External (client ↔ system): REST, GraphQL, WebSocket via API Gateway
- Internal async (service ↔ service): Kafka events

**Repository structure:**
Each microservice lives in its own GitHub repository under the DLS-soft2 organisation. We chose this over a monorepo to mirror a production-like environment — easier to manage versioning of each service, and also to have isolated CI/CD pipelines

---

## Architectural Patterns

**Choreography-based saga** — Services react to Kafka events and the Order Service maintains the state. See [saga-events.md](saga-events.md) for the full event flow.

**CQRS** — under consideration but will be implemented for services with a high read-to-write ratio (e.g. customers frequently checking order status vs. fewer writes).

**Tombstone / Snapshot pattern** — under consideration - would make sense for maintaining historical data without hard deletes, particularly for order and payment history.

**Shared RBAC libraries** — authorization logic (role decorators, JWT claims parsing) lives in shared libraries (`auth-lib-python`, `auth-lib-java`) used as build-time dependencies. The API Gateway handles authentication, services handle authorization. This way we don't need to implement auth logic in each service, and we can enforce consistent RBAC across the system.

---

## Project Management Strategy

**Branching strategy:** GitHub flow — short-lived feature branches, PRs to `main`, CI gate on every PR before merge.

---

## CI/CD Pipeline

All Python services share a common pipeline defined in the `shared-workflows` repository and composed via `workflow_call`.

**On pull request to `main` — Tollgate:**

- Pylint linting across all `.py` files (fails below 6/10)
- Pytest run if a `tests/` or `app/tests/` directory exists
- Version bump check — `pyproject.toml` version must be higher than the latest git tag
- A summary comment is posted automatically on the PR with linting score, test results, and version status

**On merge to `main` — Build:**

- Docker image built and pushed to GHCR as `ghcr.io/dls-soft2/<service>:<version>` and `:latest`
- Git tag and GitHub Release created automatically from the `pyproject.toml` version

This will be extended to Java services as well, with a similar structure but using Java specific tools for linting, testing, versioning and so on.

---

## Deployment Strategy

Cloud provider: **Azure**.

- We have student credits and it integrates well with GitHub Actions for deployment, if we choose to deploy the project.

---

## Versioning Strategy

**Code:** Semantic versioning via `pyproject.toml` for Python services. The CI pipeline enforces that the version is bumped on every merge to `main` and automatically creates a matching git tag and GitHub Release.

**APIs:** URI versioning (e.g. `/v1/`, `/v2/`) is introduced only on breaking changes.

**Databases:** Not sure yet.

---

## Logging & Monitoring

Prometheus for metrics collection and Grafana for dashboards and alerting.

---

## Documentation Strategy

README per repository. API documentation via Swagger. Possibly docosaurus for complete system documentation if we have time.

---

## Functional & Non-Functional Requirements

See [functional-reqs.md](functional-reqs.md).

---
