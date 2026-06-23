# Architecture Guidelines

> Decisions made here outlast the sprint. Think before you import.

---

## 1. Layered Architecture — Allowed Import Directions

```
  ┌────────────────────────────┐
  │         API / CLI          │  ← entry points, routing, serialization
  └────────────┬───────────────┘
               │ allowed
               ▼
  ┌────────────────────────────┐
  │      Application /         │  ← use-cases, orchestration
  │      Service Layer         │
  └────────────┬───────────────┘
               │ allowed
               ▼
  ┌────────────────────────────┐
  │        Domain Layer        │  ← entities, value objects, business rules
  └────────────────────────────┘
               ▲
               │ allowed (infra implements domain interfaces)
  ┌────────────────────────────┐
  │    Infrastructure Layer    │  ← DB, HTTP clients, queues, file I/O
  └────────────────────────────┘
```

- **No cross-layer imports**: domain must not import infrastructure; API must not reach directly into infrastructure.
- Dependency inversion: infrastructure depends on domain interfaces, never the reverse.

---

## 2. New External Dependencies → ADR Required

Any new runtime dependency on an **external system** (3rd-party API, SaaS, managed service) needs an Architecture Decision Record before the code is merged.

ADR lives at `docs/adr/NNNN-title.md`. Minimum content:
- Context & problem
- Options considered (≥ 2)
- Decision + rationale
- Consequences (good and bad)

---

## 3. Module & Package Boundaries

- One module = one clear responsibility. Name it after what it **does**, not what it contains.
- Modules expose a public interface (`index.ts`, `__init__.py`, etc.). Internal files are private by default.
- Circular dependencies are a build error. Configure your linter to enforce it.

---

## 4. Service Communication

```
  Service A ──── sync (HTTP/gRPC) ────▶ Service B
                                         (low latency, strong coupling)

  Service A ──── async (queue/event) ──▶ Service B
                                         (resilient, loose coupling — prefer this)
```

- Prefer async messaging for cross-service writes; sync only for reads that require immediacy.
- Never share a database across services. Own your data.
- Publish domain events, not internal implementation details.

---

## 5. Scalability & State

- Stateless services by default. Externalise all shared state (cache, DB, object store).
- Design for horizontal scale from day one. No in-process singleton caches that break under replicas.
- Idempotent handlers: processing the same message twice must produce the same outcome.

---

## 6. Observability as a First-Class Concern

Every service ships with:
- **Structured logs** — `correlation_id` threads requests end-to-end.
- **Metrics** — RED (Rate, Errors, Duration) for every external call.
- **Traces** — OpenTelemetry span per operation; propagate `traceparent`.

---

## 7. Change Management

- Breaking API changes → major version bump + deprecation notice in `CHANGELOG.md`.
- Database migrations are forward-only. No destructive `ALTER`/`DROP` without a runbook.
- Feature flags gate risky changes in production before full rollout.

---

## PR Gate Checklist

- [ ] No cross-layer imports introduced
- [ ] New external dependency has an ADR
- [ ] No new circular dependency
- [ ] Stateful logic is externalised (no new in-process singletons)
- [ ] Breaking changes are versioned and documented
