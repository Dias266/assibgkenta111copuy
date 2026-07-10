# 01 — Analysis

> Shipping on the Air — Assignment #01, Software Architecture and Platforms, a.y. 2025-2026

## 1. Business Context

The business idea is an online drone-based package delivery service. A user submits a
package (with a given weight) to be delivered from an origin to a destination, within a
requested time window (which may be "as soon as possible"). The user can track the
delivery in real time: current position, remaining time, and status.

From this rough idea, three recurring concerns emerge, which later drive the bounded
context split in [02-design.md](./02-design.md):

1. **Booking** — capturing what the customer wants shipped, where, and when.
2. **Dispatching** — deciding which drone flies it, and by what route.
3. **Observing** — knowing where the package is right now and when it will arrive.

These three concerns have different rates of change, different data shapes, and
different consumers — which is the analytical justification for treating them as
separate domains rather than one monolithic "shipment" model.

## 2. Functional Requirements

| ID | Requirement | Realised by |
|----|-------------|-------------|
| FR-1 | A user can submit a shipment request specifying origin, destination, package weight/fragility, and a desired time window (including "now"). | `order-service` — `POST /shipments` |
| FR-2 | The system validates a shipment request and rejects incomplete/invalid submissions. | `order-service` |
| FR-3 | A user can list and inspect their shipments and their current status. | `order-service` — `GET /shipments`, `GET /shipments/:id` |
| FR-4 | A user can cancel a pending shipment. | `order-service` — `DELETE /shipments/:id` |
| FR-5 | The system automatically selects an available drone capable of carrying the package, and computes a flight route (waypoints) from origin to destination. | `mission-service` — `POST /missions` |
| FR-6 | The system can abort an in-progress mission (return-to-base) or mark it completed, releasing the drone back to the available fleet. | `mission-service` — `PATCH /missions/:id/abort`, `PATCH /missions/:id/complete` |
| FR-7 | A user can track a shipment: current location, ETA, delivery progress (%), and a chronological event log (order placed → drone assigned → in flight → waypoint reached → delivered). | `tracking-service` — `GET /track/:shipmentId`, `GET /track/:shipmentId/events` |
| FR-8 | A user can query just the remaining time (ETA) for a shipment, without pulling the full event history. | `tracking-service` — `GET /track/:shipmentId/eta` |
| FR-9 | The system fleet (drones and their status/battery/payload capacity) can be inspected. | `mission-service` — `GET /drones` |

## 3. Non-Functional Requirements

These are the requirements that most directly motivate the *architectural* decisions
documented in `02-design.md` (DDD bounded contexts + microservices), rather than being
satisfied by any single piece of business logic.

| ID | Requirement | Architectural consequence |
|----|-------------|----------------------------|
| NFR-1 | **Independent scalability.** Tracking generates high-frequency, write-heavy updates (drone telemetry, event ingestion) that scale independently of the low-frequency read/write pattern of booking a shipment. | Tracking is split into its own service so it can be scaled (replicas, storage) without scaling booking. |
| NFR-2 | **Independent deployability.** Booking logic, dispatch logic, and tracking logic are expected to change at different rates and owned conceptually by different concerns. | Each concern is a separate deployable microservice with its own codebase, `Dockerfile`, and versioning, rather than modules inside one deployable. |
| NFR-3 | **Partial-failure tolerance.** A user should still be able to see an existing shipment's last known tracking data even if the mission-dispatch logic is temporarily unavailable, and vice versa. | Services do not share a database or an in-process call stack; each holds its own state and degrades independently. |
| NFR-4 | **Eventual consistency over shared state.** Because state is not shared, the "truth" about a shipment is spread across services (order status in `order-service`, position/ETA in `tracking-service`, route/drone in `mission-service`) and only needs to converge, not be transactionally atomic. | Justifies event-oriented integration (`ShipmentPlaced`, `MissionStarted`, `WaypointReached`, ...) instead of a shared-database transaction across contexts. Current prototype simulates this with direct REST calls and `console.log` markers where a message broker would sit in production (see §5 of `02-design.md`). |
| NFR-5 | **Explicit, stable contracts between services.** Since services are independently deployed, they can only rely on their published API, not on each other's internals. | Captured formally in `openapi.yaml`, served interactively via Swagger UI (`docker-compose.yml`), so the contract is documented and versioned alongside the code. |
| NFR-6 | **Observability of core flow.** For a delivery-critical system, it must always be possible to answer "where is my package and when will it arrive" without depending on the availability of the booking or dispatch logic. | `tracking-service` is deliberately the single source of truth for position/ETA/event history, decoupled from `order-service` and `mission-service`. |
| NFR-7 | **Local runnability / reproducibility.** The whole distributed system should be startable with a single command for demoing and grading. | `docker-compose.yml` orchestrates all services + frontend + API docs together. |

## 4. Out of Scope (for this prototype)

To keep the focus on architecturally relevant aspects (per the assignment brief), the
following are intentionally simplified or omitted:

- Authentication/authorization and multi-tenant user accounts.
- Persistent storage (all three services use in-memory state; the code marks where a
  real database — PostgreSQL, TimescaleDB — would be introduced).
- A real message broker (Kafka) for event propagation between contexts; direct
  synchronous REST calls are used instead, with the intended production event names
  already present as comments/log markers in the code.
- Real drone hardware/telemetry integration; drone position and events are simulated.
- Payment/billing.

These are called out explicitly here because they represent conscious architectural
trade-offs for a prototype, not gaps discovered by accident.
