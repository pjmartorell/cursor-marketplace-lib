---
name: loyalty-engine-smart-policy
description: Flag le.loyalty.new_smart_policy_loyalty_engine true. Entry api/service/loyalty.go ApplyDiscount ApplyPoints rulesEngine loyalty/ PromotionEngine application_builder. Sentinel errors table RuleErrorFormat errors.Is. notInExcludePolicy second flag points to evaluators skill. NOT legacy vouchers.
---

# Smart policy — overview

**Flag:** `le.loyalty.new_smart_policy_loyalty_engine` = **true**.

## Flow

[`api/service/loyalty.go`](../../../api/service/loyalty.go) `ApplyDiscount` / `ApplyPoints` → `rulesEngine` → [`loyalty/`](../../../loyalty/) (`PromotionEngine`, evaluators, `effect_engine`) + [`api/service/application_builder.go`](../../../api/service/application_builder.go). Tickets: `ticket.ToMap` / `SyncFromMap` ([`api/tickets.go`](../../../api/tickets.go)).

**Flag false:** `loyalty-engine-legacy`. Root [`loyalty/`](../../../loyalty/) is promotion engine; [`service/loyalty/`](../../../service/loyalty/) is legacy **awards** — do not confuse.

## Related skills

| Topic | Skill |
|-------|--------|
| Evaluators, `defaultEvaluators`, `notInExcludePolicy` | `loyalty-engine-smart-policy-evaluators` |
| `EffectType*`, `effect_engine.go` | `loyalty-engine-smart-policy-effects` |
| `RuleGroupType`, `processorsMap`, `ApplicationBuilder` | `loyalty-engine-smart-policy-stackability` |

## Sentinel errors

| Error | Defined in | When |
|-------|------------|------|
| `ErrConditionsNotMet` | `loyalty/promotion_engine.go` | Condition policy fails |
| `ErrApplicationConditionsNotMet` | `loyalty/promotion_engine.go` | Apply conditions fail |
| `ErrEffectConditionsNotMet` | `loyalty/promotion_engine.go` | Effect fails / none |
| `ErrDiscountExceedsTotal` | `loyalty/promotion_engine.go` | Discount over cap |
| `ErrPromotionRuleCreation` | `loyalty/promotion_engine.go` | `NewPromotionRule` fails |
| `ErrGamingRuleHasNoGame` | `loyalty/games_evaluator.go` | Games effect without game |
| `streaming.ErrLoyaltyRuleNotFound` | `streaming` | Rule missing |
| `streaming.ErrLoyaltyRuleWithoutCoupon` | `streaming` | Coupon required |
| `streaming.ErrLoyaltyRuleNotTargeted` | `streaming` | Not targeted to customer |

### Wrapping

```go
return nil, true, fmt.Errorf(streaming.RuleErrorFormat, pr.Rule.ID, ErrConditionsNotMet)
```

Use `errors.Is(err, ErrConditionsNotMet)` in tests and callers.

### Where returned

- **Checks** `loyalty/promotion_rule.go`: `ErrConditionsNotMet`, `ErrApplicationConditionsNotMet`
- **Evaluators**: sentinels on condition/application failure
- **Effect engine** `loyalty/effect_engine.go`: `ErrDiscountExceedsTotal`; other effect failures → `ErrEffectConditionsNotMet`
- **Policies** `loyalty/policies.go`: `ErrLoyaltyRuleNotFound`, `ErrLoyaltyRuleWithoutCoupon`, `ErrLoyaltyRuleNotTargeted`

## `notInExcludePolicy`

Second flag: `le.loyalty.promotion_conditions.enable_not_in_excludes_policy`. Path, bundle behaviour, grouped tests → **`loyalty-engine-smart-policy-evaluators`**.
