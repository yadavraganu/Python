# Python Architecture and Design for Senior Interviews

This guide connects core Python knowledge to larger software-design decisions. It is intentionally framework-neutral: the same principles apply to services, data systems, command-line tools, libraries, and long-running workers.

## 1. Start with requirements and constraints

Before proposing classes or modules, clarify:

- Functional behavior and domain invariants.
- Expected throughput, latency, and traffic shape.
- Data volume, consistency, ordering, and retention.
- Failure behavior, retries, idempotency, and recovery.
- Security, privacy, auditability, and authorization.
- Deployment model, operational ownership, and observability.
- Compatibility, migration, and support lifetime.

A design is a set of trade-offs, not a diagram of boxes. State assumptions and identify which requirements drive the architecture.

## 2. Organize code around responsibilities and boundaries

A maintainable Python application commonly separates:

- **Domain model:** business rules and invariants, ideally independent of infrastructure.
- **Application/use-case layer:** coordinates domain operations and transaction boundaries.
- **Adapters:** database, HTTP, message broker, filesystem, clock, and external APIs.
- **Entry points:** CLI commands, web handlers, scheduled jobs, or message consumers.
- **Composition root:** creates concrete dependencies and wires the application together.

This is a practical form of ports-and-adapters / hexagonal architecture. The important property is dependency direction: core policy should not depend on a particular database driver or web framework.

## 3. Dependency inversion and injection

Depend on a small contract where the boundary needs substitution or isolation. In Python, the contract can be a `Protocol` or an abstract base class.

```python
from typing import Protocol

class UserRepository(Protocol):
    def get_email(self, user_id: int) -> str | None: ...

class WelcomeService:
    def __init__(self, users: UserRepository) -> None:
        self._users = users

    def greeting(self, user_id: int) -> str:
        email = self._users.get_email(user_id)
        if email is None:
            raise LookupError("user not found")
        return f"Welcome, {email}"
```

A fake can be passed in tests without inheriting from the protocol:

```python
class FakeUsers:
    def get_email(self, user_id: int) -> str | None:
        return "person@example.test"

service = WelcomeService(FakeUsers())
assert service.greeting(1) == "Welcome, person@example.test"
```

Do not create an interface for every class. Introduce an abstraction when it protects a meaningful boundary, enables a real alternative, or makes a side effect controllable.

## 4. Keep side effects at the edges

Pure functions are easy to test, cache, retry, and reason about because output depends only on input and no external state is changed.

```python
def calculate_total(unit_price: int, quantity: int, discount: int = 0) -> int:
    if unit_price < 0 or quantity < 0 or discount < 0:
        raise ValueError("values must be non-negative")
    gross = unit_price * quantity
    return max(0, gross - discount)
```

A use case can orchestrate I/O while delegating decisions to pure functions. Inject clocks, random-number sources, ID generators, and external clients when deterministic tests matter.

## 5. Design data contracts explicitly

- Use dataclasses for internal records and value objects.
- Use `TypedDict` or validation models at dictionary/JSON boundaries when mapping shape matters.
- Use enums for stable finite states.
- Make mutability explicit; use frozen value objects when appropriate.
- Validate invariants in constructors or domain methods, not only in callers.
- Version long-lived external event/message schemas.
- Avoid exposing database entities directly as public API contracts.

Type annotations help tooling but do not enforce runtime validation.

## 6. Errors, retries, and idempotency

Choose error categories that callers can act on:

- Validation errors: do not retry unchanged input.
- Not-found/conflict errors: translate to expected domain/API outcomes.
- Transient dependency errors: bounded retry with backoff and jitter when safe.
- Authentication/authorization errors: do not retry blindly.
- Unexpected errors: preserve traceback, report with correlation context, and fail safely.

Retries can amplify an outage. Apply deadlines, attempt limits, backoff, jitter, and circuit-breaking/load-shedding where appropriate.

A retried operation should be idempotent or protected by an idempotency key / deduplication strategy. For example, payment or message processing must not apply a side effect twice merely because an acknowledgment was lost.

## 7. Concurrency model is an architectural choice

| Workload | Typical option | Main trade-offs |
|---|---|---|
| Blocking I/O with synchronous libraries | Thread pool | Simple integration; bound workers and connections |
| Many concurrent network I/O tasks | `asyncio` | Requires async-compatible dependencies and cancellation discipline |
| CPU-heavy work on GIL-enabled CPython | Process pool or native code | Serialization, startup, and memory overhead |
| CPU work on free-threaded build | Threads may scale | Extension compatibility and data-race discipline |
| Independent service workloads | Separate processes/services | Deployment and operational complexity |

Do not make a codebase half-sync/half-async without a clear boundary. Avoid unbounded task creation, detached background tasks without lifecycle ownership, and pools sized without regard to downstream capacity.

## 8. Reliability and backpressure

- Bound queues and worker pools.
- Define what happens when a queue is full: block, reject, drop, or persist.
- Use timeouts on network and database calls.
- Propagate cancellation and clean up resources.
- Distinguish at-most-once, at-least-once, and effectively-once processing.
- Make consumers safe under redelivery.
- Use dead-letter or quarantine handling for poison messages.
- Ensure shutdown drains or checkpoints work according to an explicit policy.

A system that accepts work faster than it can process it without bounds will eventually fail through memory growth or unacceptable latency.

## 9. Observability

Use three complementary signals:

- **Logs:** discrete events with structured fields and correlation IDs.
- **Metrics:** aggregate rates, error ratios, queue depth, saturation, and latency distributions.
- **Traces:** end-to-end spans across service/dependency boundaries.

Track p50/p95/p99 rather than averages alone. Avoid high-cardinality metric labels such as raw user IDs or request IDs. Never put secrets or sensitive payloads in telemetry.

## 10. Testing strategy for an architecture

| Test level | Purpose | Dependency strategy |
|---|---|---|
| Unit | Domain rules and pure functions | No external I/O |
| Component | A module plus its collaborators | Fakes or local implementations |
| Integration | Real database/broker/filesystem adapter | Ephemeral test infrastructure |
| Contract | Compatibility across service boundaries | Shared schemas and provider/consumer checks |
| End-to-end | Critical user workflow | Minimal set; slower and more fragile |

Avoid mocking every internal method. Test observable behavior and important boundaries. Keep a small number of end-to-end tests for high-value workflows.

## 11. Packaging and deployment considerations

- Use a `src/` layout or another deliberate import layout to avoid accidental imports from the repository root.
- Declare runtime dependencies and supported Python versions.
- Build distributable wheels and test installation outside the source checkout.
- Keep configuration separate from code; validate it at startup.
- Read secrets from a managed secret source/environment injection, not committed files.
- Use graceful shutdown hooks for workers and servers.
- Log startup metadata: Python version, build/version, and safe configuration identifiers.
- Make migrations explicit, observable, and backward-compatible during rolling deploys.

## 12. Modularity: monolith, modular monolith, or services?

A modular monolith is often a strong starting point: one deployable unit with explicit internal boundaries. Split into services when independent scaling, deployment ownership, fault isolation, regulatory boundaries, or team topology justify the distributed-systems cost.

Service extraction adds network failures, partial availability, schema evolution, observability, deployment, and consistency challenges. Do not split merely to make a diagram look modern.

## 13. Senior/architect interview answer framework

When asked to design a Python system, answer in this order:

1. Clarify scale, SLOs, data consistency, and failure requirements.
2. Draw the major components and trust boundaries.
3. Describe key data models and contracts.
4. Explain dependency direction and how infrastructure is isolated.
5. Choose sync/async/process execution based on workload and libraries.
6. Walk through one normal request and one failure path.
7. Explain idempotency, retries, backpressure, and shutdown.
8. Describe logs, metrics, traces, and operational alerts.
9. Outline unit/integration/contract/end-to-end tests.
10. State trade-offs and a path to evolve the design.

## 14. Common weak answers

- **"Use microservices for scalability."** Explain which bottleneck or ownership boundary requires a service split.
- **"Use async because it is faster."** Async helps with concurrent waiting; it does not make CPU computation faster.
- **"Retry on any error."** Retries need classification, bounds, deadlines, and idempotency.
- **"Mock the database in every test."** Keep adapter integration tests against a real compatible database.
- **"Add a cache."** Define cache key, TTL/invalidation, consistency, memory limit, and behavior on cache failure.
- **"Use a singleton/global."** Explain lifecycle, test isolation, concurrency safety, and multi-process behavior.
