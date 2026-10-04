# 🏛️ Enterprise Backend Architecture, Standards & Protocols Guide
## KubeOps-Aegis Microservices Engineering Blueprint

This document specifies the architectural patterns, communication protocols, logging standards, and enterprise design principles implemented across the **KubeOps-Aegis** application services.

---

## 📑 Table of Contents
1. [Inversion of Control (IoC) & Event-Driven Request Lifecycle](#1-inversion-of-control-ioc--event-driven-request-lifecycle)
2. [FastAPI Dependency Injection (DI) Standards](#2-fastapi-dependency-injection-di-standards)
3. [Database Architecture: AWS Lambda vs Kubernetes Containers](#3-database-architecture-aws-lambda-vs-kubernetes-containers)
4. [Singleton Logging & Observability Protocols](#4-singleton-logging--observability-protocols)
5. [Middleware vs Business Logic Separation](#5-middleware-vs-business-logic-separation)
6. [Uvicorn Production Server Configuration Standards](#6-uvicorn-production-server-configuration-standards)
7. [Azure Log Analytics & Kusto (KQL) Query Playbook](#7-azure-log-analytics--kusto-kql-query-playbook)

---

## 1. Inversion of Control (IoC) & Event-Driven Request Lifecycle

### What is IoC in FastAPI?
In traditional procedural programming, application code drives execution by explicitly calling functions in sequence. In **Inversion of Control (IoC)** (the *Hollywood Principle*: *"Don't call us, we'll call you"*):
1. **The Server Owns the Process**: Uvicorn initializes the Python process, sets up an asynchronous event loop (`asyncio`), and binds a non-blocking TCP socket to `0.0.0.0:8000`.
2. **Registration, Not Invocation**: In `main.py` and routers, route handlers (`@router.post("")`, `@router.get("")`) are not executed by you. They are **registered** in FastAPI's internal URL routing table.
3. **Reactive Dispatch**: When a network packet arrives from an external client, Uvicorn translates the HTTP byte stream into an ASGI scope dictionary and passes it to `app`. FastAPI matches the URL path, resolves all required dependencies, and reactively invokes your async route handler (`await create_order(...)`).

```
Client (Browser / Ingress)
        |
        |  HTTP POST /api/v1/orders
        v
+-----------------------------------------------------------+
| Uvicorn Server (Async Event Loop on Port 8000)            |
|   - Reads bytes from OS TCP Socket                        |
|   - Constructs ASGI Scope                                 |
+-----------------------------------------------------------+
        |
        v
+-----------------------------------------------------------+
| FastAPI Application (IoC Controller)                      |
|   1. Executes Middleware (X-Request-ID, Latency Timer)    |
|   2. Matches Path in Routing Table                        |
|   3. Resolves Dependencies (DB, Auth, Blob Service)       |
|   4. Invokes Route Handler: await create_order(...)       |
+-----------------------------------------------------------+
        |
        v
+-----------------------------------------------------------+
| OrderService (Business Domain Logic)                      |
|   - Validates business rules                              |
|   - Calculates pricing & persists data                    |
+-----------------------------------------------------------+
```

---

## 2. FastAPI Dependency Injection (DI) Standards

### Why Dependency Injection?
Hardcoding global instances (e.g. `ORDERS_DB = []` or directly importing static singletons) tightly couples code, prevents parallel testing, and violates the **Dependency Inversion Principle (the "D" in SOLID)**.

### Our Standard (`backend.app.dependencies`):
All external infrastructure, security credentials, and business services are injected into router endpoints via FastAPI's `Depends()`:

```python
@router.post("", response_model=Order, status_code=status.HTTP_201_CREATED)
async def create_order(
    request: CreateOrderRequest,
    current_user: dict = Depends(get_current_user),          # 💉 Injected Auth
    order_service: OrderService = Depends(get_order_service),  # 💉 Injected Service
    request_id: str = Depends(get_request_id)                # 💉 Injected Correlation ID
):
    return await order_service.create_order(
        user_id=current_user["id"],
        items=request.items,
        shipping_address=request.shipping_address,
        request_id=request_id
    )
```

### Benefits for Enterprise Quality & Portfolios:
1. **Decoupled Architecture**: Routers act purely as HTTP adapters; business logic resides strictly inside services.
2. **Instant Unit Testability**: In automated test suites, cloud dependencies (Azure PostgreSQL, Azure Blob Storage) are swapped with mocks in a single line without editing production code:
   ```python
   app.dependency_overrides[get_blob_service] = get_test_blob_mock
   ```

---

## 3. Database Architecture: AWS Lambda vs Kubernetes Containers

A major architectural difference exists between Serverless (AWS Lambda) and Long-Running Containerized APIs (FastAPI on AKS):

| Attribute | AWS Lambda (Serverless) | FastAPI on AKS (Kubernetes Container) |
| :--- | :--- | :--- |
| **Lifecycle** | Ephemeral, short-lived instances (seconds/minutes). | Long-running container process (days/weeks). |
| **Concurrency Model** | 1 concurrent request per container execution environment. | **Hundreds of concurrent async requests** processed by a single event loop. |
| **DB Connection Strategy** | **Singleton Connection**: A single database connection is instantiated outside the handler and reused across warm invocations. | **Singleton Connection Pool (`asyncpg.Pool`)**: A single connection would cause severe deadlocks! A connection pool is initialized once during startup. |
| **Request Handling** | Uses the single cached connection. | **Leased Connection**: Each HTTP request leases an active connection from the pool, executes queries, and returns it to the pool immediately upon completion (`Depends(get_db_connection)`). |

### Our Implementation:
```python
# Startup: Lifespan initializes Singleton Connection Pool (min=2, max=10)
pool = await asyncpg.create_pool(dsn=settings.DATABASE_URL, min_size=2, max_size=10)

# Per-Request: Leases connection and guarantees release
async with pool.acquire() as connection:
    yield connection
```

---

## 4. Singleton Logging & Observability Protocols

### Are Loggers Singletons?
**YES.** In Python, `logging.getLogger("name")` is fundamentally designed as a **Singleton Pattern**. Python's internal `logging.Manager` maintains a registry of logger instances:
* Calling `logging.getLogger("backend.orders")` anywhere in the codebase returns the exact same logger instance.
* All loggers inherit configurations, formatting, and handlers from the root logger.

### Structured Logging Standard:
Every log line emitted by the application follows a structured, machine-parseable format:
```text
YYYY-MM-DD HH:MM:SS [LEVEL] logger.name: [req-xxxxxx] MESSAGE
```
* **Correlation ID (`X-Request-ID`)**: Every incoming request is tagged with a unique tracing UUID. This ID is passed to all downstream service calls and database queries, allowing a distributed request to be traced from start to finish.
* **Audit Tags**: High-value security and business events use explicit prefixes:
  * `AUDIT_AUTH`: User registration, login attempts, authentication failures.
  * `AUDIT_ORDERS`: Checkout processing, payment validation, order ID generation.
  * `AUDIT_CATALOG`: Catalog search and inventory lookups.
  * `AUDIT_UPLOADS`: Blob storage file transmissions.

---

## 5. Middleware vs Business Logic Separation

To maintain clean code and prevent cognitive overload, our codebase enforces strict boundary separation:

```
+---------------------------------------------------------------------------------+
| 1. HTTP MIDDLEWARE LAYER (Cross-Cutting Concerns)                               |
|   - Request ID Generation & Header Tagging (`X-Request-ID`)                     |
|   - Response Latency Measurement (`X-Response-Time: 12.4ms`)                    |
|   - Prometheus RED Telemetry (HTTP_REQUESTS_TOTAL, LATENCY HISTOGRAM)           |
|   - Global Exception Handling (Converts 500 crashes into clean JSON payloads)   |
|   - CORS Header Management                                                      |
+---------------------------------------------------------------------------------+
                                       |
                                       v
+---------------------------------------------------------------------------------+
| 2. CONTROLLER / ROUTER LAYER (HTTP Protocol Adapters)                            |
|   - Request Body Validation (Pydantic Models)                                   |
|   - Query Parameter Parsing                                                     |
|   - HTTP Status Code Assignment (201 Created, 400 Bad Request, 404 Not Found)   |
|   - Calls Injected Service                                                      |
+---------------------------------------------------------------------------------+
                                       |
                                       v
+---------------------------------------------------------------------------------+
| 3. SERVICE LAYER (Pure Domain Business Logic)                                   |
|   - Price calculation, discounts, and item tax aggregation                      |
|   - Formatting receipt URLs                                                     |
|   - Interacting with PostgreSQL Connection & Azure Blob Storage                 |
|   - Zero knowledge of HTTP headers, cookies, or status codes                    |
+---------------------------------------------------------------------------------+
```

---

## 6. Uvicorn Production Server Configuration Standards

Configured in `apps/backend/Dockerfile`:

```dockerfile
ENTRYPOINT ["uvicorn", "backend.app.main:app", \
            "--host", "0.0.0.0", \
            "--port", "8000", \
            "--timeout-keep-alive", "75", \
            "--timeout-graceful-shutdown", "30", \
            "--limit-concurrency", "200", \
            "--backlog", "2048", \
            "--access-log"]
```

### Engineering Rationale:
1. **`--timeout-keep-alive 75`**:
   * Azure Load Balancer TCP idle timeout is **240 seconds**.
   * NGINX Ingress keep-alive timeout is **65–75 seconds**.
   * By setting Uvicorn to **75 seconds**, upstream proxies close idle sockets cleanly before Uvicorn drops them, completely eliminating intermittent **`502 Bad Gateway / TCP RST`** errors.
2. **`--timeout-graceful-shutdown 30`**:
   * When Kubernetes rolls out an update or scales down a pod, kubelet sends a `SIGTERM` signal followed by `terminationGracePeriodSeconds` (default 30s).
   * Uvicorn will stop accepting new connections while giving existing in-flight transactions 30 seconds to finish without dropping customer orders.
3. **`--limit-concurrency 200`**:
   * Prevents container thread exhaustion and memory starvation during traffic spikes.

---

## 7. Azure Log Analytics & Kusto (KQL) Query Playbook

All logs printed to `stdout` and `stderr` by Uvicorn are captured by the Azure Kubernetes Service Container Insights agent.

### Query 1: Trace a Specific User Request Across the Entire Cluster
```kql
ContainerLogV2
| where LogMessage contains "req-abc12345"
| project TimeGenerated, PodName, LogMessage
| order by TimeGenerated asc
```

### Query 2: Audit All Customer Orders & Payments
```kql
ContainerLogV2
| where LogMessage contains "AUDIT_ORDERS"
| project TimeGenerated, PodName, LogMessage
| order by TimeGenerated desc
```

### Query 3: Real-Time Error Spike Detection (HTTP 500 & Unhandled Exceptions)
```kql
ContainerLogV2
| where LogLevel == "error" or LogMessage contains "Unhandled Exception"
| summarize ErrorCount = count() by bin(TimeGenerated, 5m), PodName
| render timechart
```
