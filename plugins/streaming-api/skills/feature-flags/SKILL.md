---
name: feature-flags
description: LaunchDarkly; company slug as user key. Naming (le.*, snake_case), resolve only in api/service, pass bools into loyalty/, never context. Tests feature.NewStrictStub(t) + SetFlag every path. Key flags table below. Loyalty le.loyalty.new_smart_policy_loyalty_engine routes to legacy vs smart-policy skills.
---

# Feature flags

## When to use

Use a flag when a change alters **business behaviour** (eligibility, calculation, rules). Not for refactors only.

**Provider**: LaunchDarkly. Per company: `ldclient.User{Key: &company}`.

## Naming

- Dot-separated hierarchy; prefix `le.` for loyalty. **true** = new behaviour; **false** = legacy.
- Example: `le.loyalty.promotion_conditions.enable_not_in_excludes_policy`
- Last segments: **snake_case**.

## Code

1. Resolve in **service/API** only (`api/service/loyalty.go`, etc.). `loyalty/` must **not** import `feature` or `ldclient`.
2. Resolve once per request/flow; **pass as parameters** — not via `context`.
3. Parameter names describe behaviour, e.g. `notInExcludePolicy bool`.

```go
notInExcludePolicy, _ := feature.Client.Check(
    "le.loyalty.promotion_conditions.enable_not_in_excludes_policy",
    ldclient.User{Key: &company},
)
processor.Process(ctx, rules, entityData, ..., notInExcludePolicy)
```

## Tests

1. `ff := feature.NewStrictStub(t)` only — never `NewStrictStub(t, true)`. Assign to `feature.Client`.
2. `ff.SetFlag("flag.key", value)` for **every** flag read on the path.
3. Scenario values: usually `false` = legacy, `true` = new.

**Fix silent tests**: remove `true` from `NewStrictStub`, add missing `SetFlag` calls.

**New flag in code**: set default in existing tests; add at least one test with flag **true** for new behaviour.

```go
ff := feature.NewStrictStub(t)
ff.SetFlag("le.loyalty.promotion_conditions.enable_not_in_excludes_policy", false)
feature.Client = ff
```

Integration paths that touch multiple branches: set **all** flags any branch might read.

## Flags used often (business summary)

| Flag | Summary |
|------|---------|
| `le.loyalty.promotion_conditions.enable_not_in_excludes_policy` | Off: line in exclusion still gets promotion. On: whole rule excluded when `not_in` matches (grouped path). |
| `le.loyalty.smart_policy_join_highest_discounts` | Off: separate discount groups. On: single highest-discount group across types. |
| `le.loyalty.new_smart_policy_loyalty_engine` | Off: legacy vouchers/PolicyApplier/trigger. On: `loyalty/` PromotionEngine + evaluators + ApplicationBuilder. |
| `restrict.payment.methods` | Off: no payment checks. On: restrict payment methods on tickets/score rules. |
| `le.scores.multiplicator.decimal.strategy` | Off: integer precision. On: decimal for points. |
| `le.loyalty_rule.limit.scores.won` | Off: no cap. On: cap points per rule/transaction. |

When you add or remove a **business** flag, update any in-repo flag inventory the team maintains so it stays aligned with code.

**Loyalty routing**: `le.loyalty.new_smart_policy_loyalty_engine` false → `loyalty-engine-legacy`; true → `loyalty-engine-smart-policy`, then evaluators / effects / stackability as needed.
