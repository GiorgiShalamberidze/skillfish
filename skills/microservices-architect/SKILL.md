---
name: microservices-architect
description: Microservices design: service decomposition, inter-service communication, saga pattern, API gateway, distributed tracing, and monolith migration.
---

# Microservices Architect

## Overview

The Microservices Architect skill provides deep expertise in designing, building, and evolving distributed systems based on microservices principles. It covers the full lifecycle from decomposing a monolith into bounded services, choosing the right communication patterns, handling distributed transactions with sagas, securing traffic through API gateways, instrumenting observability across service boundaries, managing per-service data stores, and hardening the system with resilience patterns. Use this skill whenever you need to make architectural decisions about service boundaries, distributed data consistency, or production-grade microservices infrastructure.

## Table of Contents

- [When to Use](#when-to-use)
- [1. Service Decomposition](#1-service-decomposition)
- [2. Communication Patterns](#2-communication-patterns)
- [3. Saga Pattern](#3-saga-pattern)
- [4. API Gateway](#4-api-gateway)
- [5. Distributed Tracing and Observability](#5-distributed-tracing-and-observability)
- [6. Data Management](#6-data-management)
- [7. Resilience Patterns](#7-resilience-patterns)
- [8. Migration from Monolith](#8-migration-from-monolith)
- [Output Format](#output-format)
- [Decision Cheat Sheet](#decision-cheat-sheet)

---

## When to Use

- You are decomposing a monolith into microservices and need to identify service boundaries
- You need to choose between synchronous and asynchronous communication for inter-service calls
- You are designing distributed transactions and need saga orchestration or choreography
- You want to set up an API gateway with routing, rate limiting, and authentication
- You need to instrument distributed tracing and centralized observability across services
- You are deciding on database-per-service, event sourcing, or CQRS strategies
- You need resilience patterns like circuit breakers, bulkheads, and retry policies
- You are planning a phased migration from a monolith to microservices

---

## 1. Service Decomposition

### 1.1 Domain-Driven Design Foundations

Service decomposition starts with understanding the business domain. Apply Domain-Driven Design (DDD) to discover natural boundaries.

**Core DDD Concepts for Decomposition:**

| Concept | Definition | Architectural Impact |
|---------|-----------|---------------------|
| **Domain** | The overall problem space the software addresses | Defines the system scope |
| **Subdomain** | A distinct area within the domain (core, supporting, generic) | Maps to candidate services |
| **Ubiquitous Language** | Shared vocabulary between developers and domain experts | Defines API contracts and data models |
| **Domain Model** | Code representation of business rules and entities | Lives inside one service |
| **Domain Event** | Something meaningful that happened in the domain | Drives inter-service communication |

**Subdomain Classification:**

- **Core subdomains** -- Competitive advantage. Build custom, invest heavily. These become your most critical services.
- **Supporting subdomains** -- Necessary but not differentiating. Build simpler services or use low-code.
- **Generic subdomains** -- Commodity functionality. Buy or use open-source (auth, email, payments).

### 1.2 Bounded Contexts

A bounded context is the explicit boundary within which a domain model applies. Each microservice should align with exactly one bounded context.

**Identifying Bounded Contexts:**

1. **Event Storming** -- Gather domain experts and developers. Place domain events on a timeline. Cluster related events into candidate contexts.
2. **Context Mapping** -- Document relationships between bounded contexts:
   - **Shared Kernel** -- Two contexts share a subset of the model (tight coupling, use sparingly)
   - **Customer-Supplier** -- Upstream context provides data to downstream (API contract)
   - **Anticorruption Layer (ACL)** -- Downstream context translates upstream models to protect its own model
   - **Conformist** -- Downstream context adopts upstream model as-is (low effort, high coupling)
   - **Open Host Service** -- Upstream publishes a well-defined protocol for any consumer
   - **Published Language** -- Shared schema format (protobuf, Avro, JSON Schema)

**Context Map Diagram:**

```
+-------------------+     Shared Kernel     +-------------------+
|   Order Context   |<--------------------->| Inventory Context |
|  (Core Subdomain) |                       | (Core Subdomain)  |
+-------------------+                       +-------------------+
        |                                           |
        | Customer-Supplier                         | ACL
        v                                           v
+-------------------+                       +-------------------+
| Payment Context   |                       | Shipping Context  |
| (Core Subdomain)  |                       | (Supporting)      |
+-------------------+                       +-------------------+
        |                                           |
        | Open Host Service                         | Conformist
        v                                           v
+-------------------+                       +-------------------+
| Notification Ctx  |                       | Analytics Context |
| (Generic)         |                       | (Supporting)      |
+-------------------+                       +-------------------+
```

### 1.3 Identifying Service Boundaries

**Heuristics for Good Boundaries:**

1. **Single Responsibility** -- Each service owns one business capability end-to-end
2. **Data Ownership** -- Each service owns and is the sole writer to its data store
3. **Team Alignment** -- Service boundaries should map to team boundaries (Conway's Law)
4. **Independent Deployability** -- A change in one service should not force deployment of another
5. **Failure Isolation** -- A service failure should not cascade to unrelated services

**Anti-Patterns to Avoid:**

- **Distributed Monolith** -- Services that must be deployed together, share databases, or have synchronous call chains
- **Nano-services** -- Services so small they add network overhead without meaningful encapsulation
- **Shared Database** -- Multiple services reading/writing the same tables
- **God Service** -- One service that orchestrates everything and contains too much logic

### 1.4 Data Ownership

Every entity has exactly one owning service. Other services hold read-only projections or references (IDs).

**Data Ownership Rules:**

| Rule | Description |
|------|-------------|
| **Single Writer** | Only the owning service performs writes for its entities |
| **ID References** | Other services store entity IDs, not copies of the data |
| **Eventual Consistency** | Read projections in other services may lag behind the source |
| **Events for Sync** | Owning service publishes domain events; consumers build local views |
| **No Shared Tables** | Each service has its own schema/database; no cross-service joins |

### 1.5 Strangler Fig Pattern

The Strangler Fig pattern is the primary strategy for incrementally migrating from monolith to microservices (detailed in Section 8).

**Principle:** Route traffic through a facade. Gradually replace monolith functionality with new services behind the facade. The monolith shrinks over time until it can be decommissioned.

```
                    +-------------------+
   Clients -------->|   API Gateway /   |
                    |   Facade Layer    |
                    +-------------------+
                     /        |         \
                    v         v          v
            +---------+ +---------+ +-----------+
            | New Svc | | New Svc | | Monolith  |
            |  Orders | | Users   | | (shrinking|
            +---------+ +---------+ |  over     |
                                    |  time)    |
                                    +-----------+
```

---

## 2. Communication Patterns

### 2.1 Synchronous Communication

Synchronous patterns block the caller until the callee responds. Use when the caller needs an immediate answer to proceed.

#### REST (HTTP/JSON)

**When to Use:** Public-facing APIs, CRUD operations, broad tooling/ecosystem needs.

| Aspect | Details |
|--------|---------|
| **Protocol** | HTTP/1.1 or HTTP/2 |
| **Serialization** | JSON (human-readable, larger payload) |
| **Contract** | OpenAPI / Swagger |
| **Discovery** | DNS, service registry, load balancer |
| **Strengths** | Universal tooling, easy debugging, caching with HTTP semantics |
| **Weaknesses** | Higher latency than binary protocols, no streaming (HTTP/1.1), verbose payloads |

#### gRPC

**When to Use:** Internal service-to-service calls, high-throughput low-latency paths, polyglot environments.

| Aspect | Details |
|--------|---------|
| **Protocol** | HTTP/2 with binary framing |
| **Serialization** | Protocol Buffers (compact, fast) |
| **Contract** | `.proto` files with code generation |
| **Discovery** | Service mesh sidecar, DNS, etcd |
| **Strengths** | Bidirectional streaming, strong typing, ~10x smaller payloads than JSON |
| **Weaknesses** | Harder to debug (binary), less browser support, steeper learning curve |

**gRPC Communication Modes:**

1. **Unary** -- Single request, single response (like REST)
2. **Server streaming** -- Single request, stream of responses (real-time feeds)
3. **Client streaming** -- Stream of requests, single response (batch uploads)
4. **Bidirectional streaming** -- Both sides stream simultaneously (chat, live sync)

### 2.2 Asynchronous Communication

Asynchronous patterns decouple sender and receiver in time. The sender fires a message and does not wait for a response.

#### Message Queues (Point-to-Point)

**When to Use:** Task distribution, work queues, guaranteed delivery to a single consumer.

**Technologies:** RabbitMQ, Amazon SQS, Azure Service Bus, Redis Streams.

```
+-----------+     +-------------+     +------------+
| Producer  |---->|   Queue     |---->| Consumer   |
| (Order    |     | (order-     |     | (Payment   |
|  Service) |     |  processing)|     |  Service)  |
+-----------+     +-------------+     +------------+
```

**Delivery Guarantees:**

- **At-most-once** -- Message may be lost, never duplicated (fastest)
- **At-least-once** -- Message never lost, may be duplicated (most common)
- **Exactly-once** -- Message delivered once (requires idempotent consumers or transactional outbox)

#### Event Streaming (Pub-Sub)

**When to Use:** Event-driven architectures, multiple consumers need the same event, event replay, audit logs.

**Technologies:** Apache Kafka, Amazon Kinesis, Redpanda, Pulsar.

```
+-----------+     +-----------------+     +------------+
| Publisher |---->|   Topic         |---->| Subscriber |
| (Order    |     | (order-events)  |---->| (Inventory)|
|  Service) |     |                 |---->| (Analytics)|
+-----------+     +-----------------+     +------------+
```

**Key Characteristics:**

- **Durable log** -- Events are stored on disk, replayable from any offset
- **Consumer groups** -- Parallel consumption with partition assignment
- **Ordering** -- Guaranteed within a partition (use entity ID as partition key)
- **Retention** -- Time-based or size-based; can be infinite (compacted topics)

### 2.3 Request-Reply over Messaging

Combines async transport with a synchronous-feeling interaction when needed.

```
+-----------+   request   +-------+   request   +----------+
| Service A |------------>| Queue |------------>| Service B|
|           |<------------|       |<------------|          |
+-----------+   reply     +-------+   reply     +----------+
     (correlation ID links request to response)
```

**Implementation:** Attach a `correlation_id` and `reply_to` queue/topic to the request message. The consumer processes the request and publishes the response to the reply destination with the same correlation ID.

### 2.4 Choosing Between Patterns

**Decision Matrix:**

| Factor | REST | gRPC | Message Queue | Event Streaming |
|--------|------|------|---------------|-----------------|
| **Latency requirement** | Medium | Low | Varies | Varies |
| **Coupling** | Temporal + spatial | Temporal + spatial | Temporal decoupling | Full decoupling |
| **Caller needs response** | Yes | Yes | Optional | No |
| **Multiple consumers** | No (1:1) | No (1:1) | No (1:1) | Yes (1:N) |
| **Guaranteed delivery** | Retry needed | Retry needed | Built-in | Built-in |
| **Ordering** | N/A | N/A | FIFO possible | Per-partition |
| **Replay** | No | No | No (consumed) | Yes |
| **Best for** | Public APIs, CRUD | Internal, streaming | Task queues | Event-driven arch |

---

## 3. Saga Pattern

Sagas manage distributed transactions across multiple services without a two-phase commit. Each service performs its local transaction and publishes an event or command. If any step fails, compensating transactions undo prior steps.

### 3.1 Choreography vs Orchestration

#### Choreography (Event-Driven)

Each service listens for events and decides autonomously what to do next.

```
Order Service          Payment Service        Inventory Service       Shipping Service
     |                       |                       |                       |
     |-- OrderCreated ------>|                       |                       |
     |                       |-- PaymentProcessed -->|                       |
     |                       |                       |-- InventoryReserved ->|
     |                       |                       |                       |-- ShipmentScheduled
     |<-----------------------------------------------------------------------|
     |                                  OrderCompleted
```

| Aspect | Details |
|--------|---------|
| **Coordination** | Decentralized -- each service reacts to events |
| **Coupling** | Loose -- services only know about events, not each other |
| **Complexity** | Low for simple flows; grows rapidly with branching/compensation |
| **Visibility** | Hard to trace the overall saga state without tooling |
| **Best for** | Simple, linear workflows with few participants (2-4 services) |

#### Orchestration (Command-Driven)

A central orchestrator service directs each participant step by step.

```
                    +-------------------+
                    |  Saga Orchestrator|
                    |  (Order Saga)     |
                    +-------------------+
                   /         |          \
     ProcessPayment   ReserveInventory   ScheduleShipment
                 /           |            \
    +----------+    +----------+    +----------+
    | Payment  |    | Inventory|    | Shipping |
    | Service  |    | Service  |    | Service  |
    +----------+    +----------+    +----------+
```

| Aspect | Details |
|--------|---------|
| **Coordination** | Centralized -- orchestrator controls flow |
| **Coupling** | Orchestrator depends on all participants |
| **Complexity** | Manageable even for complex flows; state machine is explicit |
| **Visibility** | High -- orchestrator holds the full saga state |
| **Best for** | Complex, branching workflows with many participants (4+ services) |

### 3.2 Compensating Transactions

When a saga step fails, all previously completed steps must be undone via compensating transactions.

**Compensation Chain Example (Order Saga Failure at Inventory):**

| Step | Forward Action | Compensating Action |
|------|---------------|-------------------|
| 1 | Create Order (status: PENDING) | Cancel Order (status: CANCELLED) |
| 2 | Process Payment (charge card) | Refund Payment (reverse charge) |
| 3 | Reserve Inventory (decrement stock) | Release Inventory (increment stock) |
| 4 | Schedule Shipment | *Step never reached -- no compensation needed* |

**Compensation Rules:**

1. Compensating transactions must be **idempotent** -- safe to execute multiple times
2. Compensations execute in **reverse order** of the forward steps
3. Compensations themselves can fail -- implement retry with exponential backoff
4. Log compensation outcomes for auditing and manual intervention

### 3.3 State Machines

Model each saga as a finite state machine to make transitions explicit and debuggable.

**Order Saga State Machine:**

```
[STARTED] --PaymentRequested--> [AWAITING_PAYMENT]
[AWAITING_PAYMENT] --PaymentSucceeded--> [AWAITING_INVENTORY]
[AWAITING_PAYMENT] --PaymentFailed--> [COMPENSATING_ORDER]
[AWAITING_INVENTORY] --InventoryReserved--> [AWAITING_SHIPMENT]
[AWAITING_INVENTORY] --InventoryFailed--> [COMPENSATING_PAYMENT]
[AWAITING_SHIPMENT] --ShipmentScheduled--> [COMPLETED]
[AWAITING_SHIPMENT] --ShipmentFailed--> [COMPENSATING_INVENTORY]
[COMPENSATING_INVENTORY] --InventoryReleased--> [COMPENSATING_PAYMENT]
[COMPENSATING_PAYMENT] --PaymentRefunded--> [COMPENSATING_ORDER]
[COMPENSATING_ORDER] --OrderCancelled--> [FAILED]
```

**Implementation Recommendations:**

- Store saga state in a durable store (Postgres, DynamoDB)
- Include `saga_id`, `current_state`, `created_at`, `updated_at`, and a `history` JSON array
- Use a state machine library (e.g., XState, Spring Statemachine, MassTransit Automatonymous)
- Set timeouts on each state to detect stuck sagas

### 3.4 Failure Handling

**Failure Categories:**

| Failure Type | Description | Handling Strategy |
|-------------|-------------|-------------------|
| **Transient** | Network timeout, temporary unavailability | Retry with exponential backoff |
| **Business** | Insufficient funds, out of stock | Compensate and mark saga as failed |
| **Poison** | Malformed message, unrecoverable bug | Dead-letter queue + alert |
| **Timeout** | Step exceeds expected duration | Timeout handler triggers compensation |

**Dead Letter Queue (DLQ) Strategy:**

- Messages that fail after max retries go to a DLQ
- Monitor DLQ depth with alerts
- Build tooling for manual inspection, replay, and disposal of DLQ messages

### 3.5 Idempotency

Every saga participant must be idempotent -- processing the same message twice produces the same result.

**Idempotency Techniques:**

1. **Idempotency Key** -- Store a unique `message_id` in a processed-messages table. Check before processing; skip if already seen.
2. **Conditional Updates** -- Use optimistic locking (`WHERE version = expected_version`) to prevent double processing.
3. **Natural Idempotency** -- Design operations to be naturally idempotent (e.g., `SET balance = 50` instead of `SET balance = balance - 10`).
4. **Deduplication Window** -- Keep idempotency keys for a bounded window (e.g., 7 days), then expire.

---

## 4. API Gateway

The API gateway is the single entry point for all client traffic. It handles cross-cutting concerns so individual services do not have to.

### 4.1 Core Responsibilities

```
                         +------------------------+
    Clients ------------>|      API Gateway        |
    (Web, Mobile,        |                        |
     Third-party)        |  - Routing             |
                         |  - Rate Limiting        |
                         |  - Authentication       |
                         |  - Request Aggregation  |
                         |  - TLS Termination      |
                         |  - Logging / Metrics    |
                         +------------------------+
                          /        |         \
                         v         v          v
                   +---------+ +---------+ +---------+
                   | Service | | Service | | Service |
                   |    A    | |    B    | |    C    |
                   +---------+ +---------+ +---------+
```

### 4.2 Routing

**Path-Based Routing:**

| Route Pattern | Target Service |
|---------------|---------------|
| `/api/v1/orders/*` | Order Service |
| `/api/v1/users/*` | User Service |
| `/api/v1/products/*` | Product Service |
| `/api/v1/payments/*` | Payment Service |

**Header-Based Routing:**

- Route by `Accept` header (e.g., `application/vnd.v2+json` for API versioning)
- Route by `X-Tenant-ID` for multi-tenant architectures
- Route by `X-Feature-Flag` for canary deployments

### 4.3 Rate Limiting

**Rate Limiting Strategies:**

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Fixed Window** | N requests per time window | Simple, predictable limits |
| **Sliding Window** | Rolling count over last N seconds | Smoother throttling |
| **Token Bucket** | Tokens replenish at fixed rate; burst allowed | Allow bursts within budget |
| **Leaky Bucket** | Requests processed at fixed rate; excess queued | Smooth outbound rate |

**Configuration Example (Kong):**

```yaml
plugins:
  - name: rate-limiting
    config:
      minute: 100
      hour: 5000
      policy: redis
      redis_host: redis.internal
      fault_tolerant: true
      hide_client_headers: false
```

### 4.4 Authentication and Authorization

**Authentication Flow:**

```
Client --> API Gateway --> Validate JWT --> Extract claims
                |                              |
                | (invalid)                    | (valid)
                v                              v
          401 Unauthorized              Forward request with
                                        X-User-ID, X-Roles headers
                                        to downstream service
```

**Common Patterns:**

- **JWT Validation** -- Gateway verifies JWT signature and expiry; passes claims downstream
- **OAuth 2.0 / OIDC** -- Gateway handles token introspection or JWKS validation
- **API Key** -- Gateway validates API key against a registry; applies rate limits per key
- **mTLS** -- Mutual TLS between gateway and internal services for zero-trust networking

### 4.5 Request Aggregation

Combine multiple downstream calls into a single client response to reduce round trips.

```
Client: GET /api/v1/dashboard

Gateway internally calls:
  1. GET /users/123          --> { name, email }
  2. GET /orders?user=123    --> [ orders ]
  3. GET /analytics?user=123 --> { metrics }

Gateway responds:
  {
    "user": { name, email },
    "orders": [ orders ],
    "analytics": { metrics }
  }
```

**Caution:** Aggregation adds latency (sequential or parallel fan-out) and increases gateway complexity. Use judiciously.

### 4.6 Backend for Frontend (BFF) Pattern

Deploy separate gateway instances tailored to each client type.

```
+----------+     +----------+     +-----------+
| Web App  |     | Mobile   |     | Third-    |
|          |     | App      |     | Party API |
+----------+     +----------+     +-----------+
      |                |                |
      v                v                v
+----------+     +----------+     +-----------+
| Web BFF  |     | Mobile   |     | Public    |
|          |     | BFF      |     | API GW    |
+----------+     +----------+     +-----------+
      \               |               /
       \              |              /
        v             v             v
   +------+ +-------+ +--------+ +-------+
   |Order | |User   | |Product | |Payment|
   |Svc   | |Svc    | |Svc     | |Svc    |
   +------+ +-------+ +--------+ +-------+
```

**Benefits:**
- Optimized payloads per client (mobile gets less data than web)
- Independent release cycles per client channel
- Client-specific authentication strategies

### 4.7 Gateway Technology Comparison

| Gateway | Type | Strengths | Best For |
|---------|------|-----------|----------|
| **Kong** | OSS / Enterprise | Plugin ecosystem, Lua extensibility, DB-less mode | General purpose, plugin-heavy |
| **Envoy** | OSS (CNCF) | L4/L7 proxy, xDS API, Wasm filters, service mesh data plane | Kubernetes, service mesh (Istio) |
| **AWS API Gateway** | Managed | Lambda integration, usage plans, WebSocket support | AWS-native, serverless |
| **NGINX** | OSS / Plus | Battle-tested, high perf, Lua scripting (OpenResty) | High-throughput, simple routing |
| **Traefik** | OSS | Auto-discovery (Docker, K8s), Let's Encrypt, middleware chains | Kubernetes, Docker Compose |
| **Apigee** | Managed (GCP) | Analytics, monetization, developer portal | Enterprise API management |

---

## 5. Distributed Tracing and Observability

### 5.1 The Three Pillars of Observability

| Pillar | What It Captures | Tools |
|--------|-----------------|-------|
| **Traces** | Request flow across services with timing | Jaeger, Zipkin, Tempo, Datadog APM |
| **Metrics** | Numeric measurements aggregated over time | Prometheus, Grafana, Datadog, CloudWatch |
| **Logs** | Discrete events with structured context | ELK Stack, Loki, Datadog Logs, CloudWatch |

### 5.2 OpenTelemetry

OpenTelemetry (OTel) is the CNCF standard for instrumenting, collecting, and exporting telemetry data.

**Architecture:**

```
+-------------------+     +-------------------+     +-------------------+
| Service A         |     | Service B         |     | Service C         |
| +---------------+ |     | +---------------+ |     | +---------------+ |
| | OTel SDK      | |     | | OTel SDK      | |     | | OTel SDK      | |
| | (auto-instr.) | |     | | (auto-instr.) | |     | | (auto-instr.) | |
| +-------+-------+ |     | +-------+-------+ |     | +-------+-------+ |
+---------|----------+     +---------|----------+     +---------|----------+
          |                          |                          |
          v                          v                          v
     +------------------------------------------------------------+
     |              OTel Collector (agent or gateway)             |
     |  - Receives traces, metrics, logs                         |
     |  - Processes (batching, sampling, enrichment)             |
     |  - Exports to backends                                    |
     +------------------------------------------------------------+
          /                    |                    \
         v                     v                     v
   +-----------+        +------------+        +-----------+
   | Jaeger /  |        | Prometheus |        | Loki /    |
   | Tempo     |        | / Grafana  |        | ELK       |
   | (traces)  |        | (metrics)  |        | (logs)    |
   +-----------+        +------------+        +-----------+
```

**OTel SDK Setup (Node.js Example):**

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector:4318/v1/traces',
  }),
  instrumentations: [getNodeAutoInstrumentations()],
  serviceName: 'order-service',
});

sdk.start();
```

### 5.3 Trace Propagation

Trace context must propagate across service boundaries so all spans in a request chain link together.

**W3C Trace Context Headers:**

```
traceparent: 00-<trace-id>-<span-id>-<trace-flags>
tracestate: vendor1=value1,vendor2=value2
```

**Propagation Methods:**

| Transport | Propagation Mechanism |
|-----------|----------------------|
| HTTP | `traceparent` / `tracestate` headers (W3C) or `X-B3-*` (Zipkin) |
| gRPC | Metadata headers |
| Kafka | Message headers |
| SQS/SNS | Message attributes |

### 5.4 Correlation IDs

A `correlation_id` ties together all operations for a single business transaction, even across retries and async flows.

**Implementation Rules:**

1. Generate a UUID at the system edge (API gateway or first service)
2. Pass it in a header (`X-Correlation-ID`) on every inter-service call
3. Include it in every log line, trace span, and message payload
4. Store it in the saga state for async correlation

### 5.5 Distributed Logging

**Structured Logging Format:**

```json
{
  "timestamp": "2026-04-04T10:15:30.123Z",
  "level": "INFO",
  "service": "order-service",
  "trace_id": "abc123def456",
  "span_id": "789ghi",
  "correlation_id": "txn-001-xyz",
  "message": "Order created successfully",
  "order_id": "ORD-5678",
  "user_id": "USR-1234",
  "duration_ms": 42
}
```

**Best Practices:**

- Use structured JSON logging -- never unstructured text in production
- Include `trace_id`, `span_id`, and `correlation_id` in every log entry
- Set log levels consistently: ERROR (alerts), WARN (needs attention), INFO (business events), DEBUG (development only)
- Aggregate logs into a central system (ELK, Loki, Datadog) with a retention policy
- Never log secrets, PII in plaintext, or full request bodies in production

### 5.6 Metrics Aggregation

**Key Metrics for Microservices (RED Method):**

| Metric | Description | Alerting Threshold (example) |
|--------|-------------|------------------------------|
| **Rate** | Requests per second per service | Sudden drop > 50% from baseline |
| **Errors** | Error rate (5xx / total) | > 1% over 5-minute window |
| **Duration** | Latency percentiles (p50, p95, p99) | p99 > 2x baseline |

**USE Method (for infrastructure):**

| Metric | Description |
|--------|-------------|
| **Utilization** | CPU, memory, disk, network usage percentage |
| **Saturation** | Queue depth, thread pool exhaustion |
| **Errors** | Hardware errors, OOM kills, disk failures |

---

## 6. Data Management

### 6.1 Database-per-Service

Each microservice owns its own database. No other service accesses it directly.

**Database Selection per Service:**

| Service Type | Recommended DB | Reasoning |
|-------------|----------------|-----------|
| Transactional (orders, payments) | PostgreSQL, MySQL | ACID guarantees, relational integrity |
| High-read catalog (products) | PostgreSQL + Redis cache | Read replicas + caching layer |
| Search-heavy (product search) | Elasticsearch, OpenSearch | Full-text search, faceted queries |
| Session / cache | Redis, Memcached | Low-latency key-value access |
| Time-series (metrics, IoT) | TimescaleDB, InfluxDB | Time-partitioned, fast aggregation |
| Document / flexible schema | MongoDB, DynamoDB | Schema-less, horizontal scaling |
| Graph (social, recommendations) | Neo4j, Neptune | Relationship traversal queries |

### 6.2 Event Sourcing

Instead of storing current state, store every state change as an immutable event.

**Event Store Structure:**

```
+------------------------------------------------------------------+
| Event Store (append-only log)                                     |
+------------------------------------------------------------------+
| event_id | aggregate_id | type              | data         | seq |
|----------|-------------|-------------------|--------------|-----|
| evt-001  | ORD-123     | OrderCreated      | {items, user}|  1  |
| evt-002  | ORD-123     | PaymentReceived   | {amount}     |  2  |
| evt-003  | ORD-123     | ItemShipped        | {tracking}   |  3  |
| evt-004  | ORD-123     | OrderCompleted     | {}           |  4  |
+------------------------------------------------------------------+
```

**Current state** is derived by replaying events for an aggregate.

**Benefits:**
- Complete audit trail -- every change is recorded
- Temporal queries -- reconstruct state at any point in time
- Event replay -- rebuild read models or fix bugs by reprocessing events

**Challenges:**
- Schema evolution -- events are immutable, so you need upcasting strategies
- Storage growth -- snapshot periodically to avoid replaying thousands of events
- Eventual consistency -- read models may lag behind the event store

### 6.3 CQRS (Command Query Responsibility Segregation)

Separate the write model (commands) from the read model (queries). Often paired with event sourcing.

```
+--------+    Command     +----------------+    Events    +----------------+
| Client |--------------->| Write Model    |------------->| Event Store    |
|        |                | (validates,    |              | (append-only)  |
|        |                |  enforces      |              +--------+-------+
|        |                |  invariants)   |                       |
|        |                +----------------+                       | Project
|        |                                                         v
|        |    Query       +----------------+              +--------+-------+
|        |<---------------| Read Model     |<-------------| Projection     |
|        |                | (optimized for |              | Builder        |
+--------+                |  queries)      |              +----------------+
                          +----------------+
```

**When to Use CQRS:**

- Read and write workloads have very different scaling requirements
- Complex domain logic on writes but simple lookups on reads
- Multiple read representations are needed (list view, dashboard, search index)
- You are already using event sourcing and need materialized views

**When to Avoid:**

- Simple CRUD with balanced read/write ratios
- Small teams that cannot maintain two models
- Strong consistency is required on every read

### 6.4 Eventual Consistency

In a microservices architecture, strong consistency across services is impractical. Embrace eventual consistency.

**Consistency Patterns:**

| Pattern | Description | Consistency Window |
|---------|-------------|-------------------|
| **Transactional Outbox** | Write event to outbox table in same DB transaction as state change; poller/CDC publishes to message broker | Seconds |
| **Change Data Capture (CDC)** | Stream DB changes (Debezium) to Kafka for downstream consumers | Seconds |
| **Polling Publisher** | Periodically poll outbox table and publish unpublished events | Seconds to minutes |
| **Event Notification** | Publish thin event; consumer calls back for full data | Seconds + call latency |

**Transactional Outbox Pattern:**

```
+-------------------+
| Service Database  |
|                   |
| +--------------+  |     +----------+     +----------+
| | orders table |  |     | Message  |     | Consumer |
| +--------------+  |     | Broker   |     | Service  |
| | outbox table |--|---->| (Kafka)  |---->|          |
| +--------------+  |     +----------+     +----------+
+-------------------+
   (single DB transaction
    writes to both tables)
```

### 6.5 Saga-Based Transactions

For operations spanning multiple services, use sagas (see Section 3) instead of distributed transactions (2PC).

**Why Not Two-Phase Commit (2PC):**

- Locks resources across services -- kills throughput
- Coordinator is a single point of failure
- Does not scale with many participants
- Not supported by most message brokers or NoSQL databases

---

## 7. Resilience Patterns

### 7.1 Circuit Breaker

Prevent cascading failures by stopping calls to a failing downstream service.

**State Machine:**

```
                    failure threshold
[CLOSED] --------------------------------> [OPEN]
   ^                                          |
   |                                          | timeout expires
   |        success threshold                 v
   +-------------------------------------- [HALF-OPEN]
   |                                          |
   |            failure in half-open          |
   +------------------------------------------+
                (back to OPEN)
```

**States:**

| State | Behavior |
|-------|----------|
| **CLOSED** | Requests pass through normally. Failures are counted. |
| **OPEN** | Requests immediately fail with a fallback response. No calls to downstream. |
| **HALF-OPEN** | A limited number of test requests are allowed through. Success resets to CLOSED; failure returns to OPEN. |

**Configuration Parameters:**

- `failure_threshold`: Number of failures before opening (e.g., 5)
- `timeout`: How long to stay OPEN before testing (e.g., 30s)
- `success_threshold`: Successes needed in HALF-OPEN to close (e.g., 3)

### 7.2 Bulkhead

Isolate resources so that a failure in one area does not exhaust resources for others.

**Thread Pool Bulkhead:**

```
+------------------------------------------------------+
| Service                                              |
|  +------------------+  +------------------+          |
|  | Order Thread Pool|  | Payment Thread   |          |
|  | (max: 20)        |  | Pool (max: 10)   |          |
|  |                  |  |                  |          |
|  | [][][][][]       |  | [][][]           |          |
|  +------------------+  +------------------+          |
|                                                      |
|  If Payment pool is exhausted, Order pool is         |
|  unaffected -- orders continue to be processed.      |
+------------------------------------------------------+
```

**Bulkhead Types:**

- **Thread pool isolation** -- Separate thread pools per downstream dependency
- **Semaphore isolation** -- Limit concurrent calls using a semaphore (lighter weight)
- **Connection pool isolation** -- Separate DB connection pools per use case
- **Pod/container isolation** -- Deploy critical paths in separate pods (Kubernetes)

### 7.3 Retry with Exponential Backoff

Retry transient failures with increasing delay to avoid thundering herd.

**Retry Formula:**

```
delay = base_delay * (2 ^ attempt) + random_jitter

Example (base_delay = 100ms):
  Attempt 1: 100ms  + jitter
  Attempt 2: 200ms  + jitter
  Attempt 3: 400ms  + jitter
  Attempt 4: 800ms  + jitter
  Attempt 5: 1600ms + jitter (max reached, give up)
```

**Retry Rules:**

1. **Only retry idempotent operations** -- GET, PUT, DELETE (not POST unless idempotency key is used)
2. **Set a max retry count** -- Typically 3-5 attempts
3. **Add jitter** -- Randomize delay to prevent synchronized retries across instances
4. **Respect `Retry-After` headers** -- If the server provides one, use it
5. **Do not retry 4xx errors** -- Client errors are not transient (except 429 Too Many Requests)

### 7.4 Timeout

Set explicit timeouts on every external call to prevent indefinite blocking.

**Timeout Strategy:**

| Call Type | Recommended Timeout | Notes |
|-----------|-------------------|-------|
| HTTP REST call | 1-5 seconds | Adjust based on downstream SLA |
| gRPC call | 500ms - 3 seconds | Tighter than REST due to binary protocol |
| Database query | 5-30 seconds | Long queries should be async jobs |
| Message publish | 1-3 seconds | Broker should be local/fast |
| External third-party API | 5-10 seconds | Protect with circuit breaker too |

**Cascade Budget:** If Service A calls Service B which calls Service C, A's timeout must exceed B's timeout plus B-to-C's timeout. Otherwise, A times out before B can complete.

```
A (timeout: 5s) --> B (timeout: 3s) --> C (timeout: 1s)
                    B needs up to 1s for C + its own work (~1s) = 2s
                    A needs up to 2s for B + its own work (~1s) = 3s < 5s (safe)
```

### 7.5 Fallback

When a dependency fails (and circuit breaker opens), return a degraded but useful response.

**Fallback Strategies:**

| Strategy | Description | Example |
|----------|-------------|---------|
| **Cached response** | Return last known good data from cache | Product catalog from Redis |
| **Default value** | Return a sensible default | Default shipping estimate "5-7 days" |
| **Graceful degradation** | Disable the feature, keep the rest working | Hide recommendations widget |
| **Queued retry** | Accept the request, process later | "Order received, confirmation pending" |
| **Alternate service** | Route to a backup or simplified service | Fallback payment processor |

---

## 8. Migration from Monolith

### 8.1 Step-by-Step Migration Workflow

Migrating from a monolith to microservices is a multi-phase effort. Never attempt a big-bang rewrite.

**Phase 1: Assess and Prepare**

1. **Map the monolith** -- Document all modules, their dependencies, database tables they access, and external integrations
2. **Identify bounded contexts** -- Apply Event Storming or domain analysis to find natural boundaries within the monolith
3. **Prioritize extraction candidates** -- Score each module on: (a) change frequency, (b) scaling needs, (c) team ownership clarity, (d) coupling to other modules
4. **Establish infrastructure** -- Set up: API gateway, CI/CD pipeline for independent service deployment, container orchestration (Kubernetes), observability stack (OTel + Grafana), message broker
5. **Define API contracts** -- Design the API surface for the first service to be extracted using OpenAPI or protobuf

**Phase 2: Strangler Fig Extraction**

6. **Deploy the API gateway** -- Route all traffic through the gateway; initially everything proxies to the monolith
7. **Extract the first service** -- Choose the lowest-risk, most self-contained module:
   - Copy the code into a new service repository
   - Set up its own database (migrate relevant tables)
   - Implement the agreed API contract
   - Write integration tests against the contract
8. **Data migration** -- Use dual-write or CDC to keep monolith and new service in sync during transition:

```
Phase 2a: Monolith writes to both old DB and new service (dual-write)
Phase 2b: Read traffic shifts to new service; monolith still writes
Phase 2c: Write traffic shifts to new service; monolith module disabled
Phase 2d: Remove old tables from monolith DB
```

9. **Route traffic** -- Update the API gateway to route the extracted module's paths to the new service instead of the monolith
10. **Validate** -- Run the new service in shadow mode (both monolith and service process requests; compare results) before cutting over

**Phase 3: Iterate and Harden**

11. **Extract the next service** -- Repeat steps 7-10 for the next highest-priority module
12. **Introduce async communication** -- Replace synchronous monolith-internal calls with events/messages between extracted services
13. **Implement sagas** -- Replace database transactions that spanned modules with saga patterns
14. **Add resilience** -- Apply circuit breakers, bulkheads, retries, and timeouts to all inter-service calls
15. **Decommission monolith modules** -- As each module is fully extracted and validated, remove the dead code from the monolith

**Phase 4: Optimize**

16. **Performance tuning** -- Profile inter-service latency; optimize hot paths with caching, gRPC, or request aggregation
17. **Consolidate observability** -- Ensure full distributed tracing, centralized logging, and dashboards cover all services
18. **Document architecture** -- Maintain a living Architecture Decision Record (ADR) log and up-to-date context map

### 8.2 Migration Anti-Patterns

| Anti-Pattern | Problem | Solution |
|-------------|---------|----------|
| **Big-bang rewrite** | Months of work with no production validation | Strangler fig, incremental extraction |
| **Shared database** | Coupling survives the split | Database-per-service from day one |
| **No API gateway** | Clients couple directly to service locations | Deploy gateway before extracting services |
| **Extracting everything at once** | Overwhelms the team, multiplies risk | One service at a time, validate, repeat |
| **Ignoring data migration** | Stale data, split-brain, lost writes | Dual-write + CDC + shadow validation |
| **No observability** | Cannot diagnose issues in distributed system | Instrument before migrating traffic |

---

## Output Format

When using this skill to produce architecture deliverables, structure output as follows:

### Architecture Decision Record (ADR)

```markdown
# ADR-NNN: [Decision Title]

## Status
Proposed | Accepted | Deprecated | Superseded by ADR-NNN

## Context
[Why this decision is needed. What problem are we solving?]

## Decision
[The decision and its rationale.]

## Consequences
### Positive
- [Benefit 1]
- [Benefit 2]

### Negative
- [Trade-off 1]
- [Trade-off 2]

### Risks
- [Risk 1 and mitigation]
```

### Service Specification

```markdown
# Service: [service-name]

## Responsibility
[Single sentence describing the service's business capability]

## Bounded Context
[Which domain context this service belongs to]

## API Contract
[OpenAPI spec or protobuf definition]

## Data Store
[Database type, key entities, ownership boundaries]

## Events Published
[List of domain events this service emits]

## Events Consumed
[List of domain events this service listens to]

## Dependencies
[Upstream and downstream services]

## SLA
[Latency targets, availability target, throughput expectations]
```

---

## Decision Cheat Sheet

| Decision | When to Choose A | When to Choose B |
|----------|-----------------|-----------------|
| **REST vs gRPC** | Public API, broad client support | Internal service-to-service, low latency |
| **Sync vs Async** | Caller needs immediate response | Fire-and-forget, multiple consumers |
| **Choreography vs Orchestration** | Simple linear saga, few services | Complex branching, many participants |
| **Event Sourcing vs State** | Audit trail needed, temporal queries | Simple CRUD, small team |
| **CQRS vs unified model** | Different read/write scaling needs | Balanced read/write, simple domain |
| **Shared DB vs DB-per-service** | Never share (use DB-per-service) | -- |
| **Circuit breaker vs retry only** | Downstream has sustained failures | Transient blips, fast recovery |
| **Monolith vs microservices** | Small team (<5), early product stage | Clear domain boundaries, scaling needs, multiple teams |
