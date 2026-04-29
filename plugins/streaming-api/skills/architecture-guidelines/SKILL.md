---
name: architecture-guidelines
description: SOLID + Hexagonal for new code. Domain loyalty/ no I/O; ports in streaming or beside use case; adapters postgres/redis/feature. sqlbuilder in postgres/; Kallax legacy only. New use case injects ports; handlers thin. Refactor priorities table below.
---

# Architecture: SOLID + Hexagonal

Domain and application depend on **interfaces**; adapters implement them.

## SOLID (short)

| Principle | Practice |
|-----------|----------|
| SRP | One use case = one flow; one adapter = one I/O concern. |
| OCP | Extend via new evaluators/adapters/use cases. |
| LSP | Implementations substitutable for interfaces. |
| ISP | Narrow ports; avoid passing full `Stores` when one port suffices. |
| DIP | Domain/application → abstractions only. |

## Hexagonal roles

| Role | This repo |
|------|-----------|
| Domain | `loyalty/` — no `postgres`, `feature`, HTTP, `queue`. |
| Application | `api/service/...` — orchestration via ports. |
| Ports | `streaming` or next to use case. |
| Driven adapters | `postgres/`, `redis_cache/`, `feature/launchdarkly/`. |
| Driving | `api/` handlers → parse, use case, map response. |

### SQL

- **New**: `postgres/` + **sqlbuilder**; follow `postgres/score_store.go`, `postgres/redemption_service.go`.
- **No** new Kallax for greenfield. **Legacy** Kallax: edit model → `make kallax`, commit generated; CI `make validate`.

### Dependency rule

Inward: adapters → ports ← application → domain. Domain must not import `api`, `postgres`, `redis_cache`, `feature`, `queue`, `job`. Application must not import concrete adapters.

## New use case

1. Type with **port interfaces** only.
2. Load, validate, effects, persist — no HTTP or LD client inside.
3. Wire in `main.go`. Handler: parse → `Execute` → map.

## New port

Define in domain/app or `streaming` (`RuleRepository`, `FeatureFlagChecker`). Implement in `postgres/`, `feature/launchdarkly/`.

## New domain behaviour

In `loyalty/` without I/O. Evaluators implement `Evaluator`; register in composition root.

## Feature flags in new code

Port `FeatureFlagChecker.Check` at adapter or use-case start; pass **values** into domain. Domain never imports the flag client.

## Package naming

**Avoid**: `utils`, `common`, `shared`, `base`, `core`, `infrastructure`, `infra`, `runtime`, `ports`, `helpers`, `types`, `misc`, `application`.

**Prefer**: `cache`, `queue`, `loyalty`, `discount`, `ticket`, `feature`.

## Refactor priorities (when touching existing code)

| Priority | Area | Change |
|----------|------|--------|
| High | Root | Move `AppServices` / infra out of domain. |
| High | Stores | Narrow port interfaces per use case vs full `Stores`. |
| High | Flags | `FeatureFlagChecker` port + adapter + inject. |
| Medium | Use cases | Extract from `api/service/loyalty.go` + handlers. |
| Medium | Handlers | Thin: validate → use case → map. |
| Medium | service/ | New flows as use cases with ports. |
| Low | Loyalty | Keep `loyalty/` I/O-free. |
