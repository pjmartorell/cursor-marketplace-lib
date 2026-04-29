---
name: loyalty-engine-smart-policy-evaluators
description: Smart policy flag true. Evaluator.Process processed bool; defaultEvaluators order api/service/loyalty.go. notInExcludePolicy flag le.loyalty.promotion_conditions.enable_not_in_excludes_policy through PromotionRule.Checks grouped not_in. BundleEvaluator Strikethrough Formula Percentage Ratio DirectValue Games.
---

# Smart policy: evaluators

Flag **`le.loyalty.new_smart_policy_loyalty_engine` = true**. Errors → `loyalty-engine-smart-policy`. Effects → `loyalty-engine-smart-policy-effects`. Stackability → `loyalty-engine-smart-policy-stackability`.

## Interface

```go
type Evaluator interface {
    Process(ctx context.Context, entityData map[string]interface{}, pr *PromotionRule, notInExcludePolicy bool) ([]PromotionGroupResult, bool, error)
}
```

- `(promotions, processed, err)` — first evaluator with `processed == true` wins.

## Add evaluator

1. `loyalty/<name>_evaluator.go`; `processed == true` only when this evaluator owns the rule.
2. Register in [`api/service/loyalty.go`](../../../api/service/loyalty.go) `defaultEvaluators` (**order matters**).
3. Wrap with `streaming.RuleErrorFormat`; `errors.Is` in tests.
4. Flags in `api/service/loyalty.go` only — **no** `feature` import in `loyalty/`.
5. Unit: `loyalty/<name>_evaluator_test.go`, `//go:build unit`; integration when flow changes.

## Registered evaluators

`BundleEvaluator`, `StrikethroughEvaluator`, `FormulaEvaluator`, `PercentageEvaluator`, `RatioEvaluator`, `DirectValueEvaluator`, `GamesEvaluator`.

## `notInExcludePolicy`

- **Flag**: `le.loyalty.promotion_conditions.enable_not_in_excludes_policy` (off = legacy exclusion behaviour; on = exclude whole rule when grouped `not_in` matches — see `feature-flags` skill table).
- **Path**: `api/service/loyalty.go` → `GroupProcessor.Process` → `PromotionEngine.Execute` → `Evaluator.Process` → `PromotionRule.Checks`. Bundle condition path uses flag; apply policy uses `false`.
- **Grouped `not_in`**: `evaluateGroupedArrayConditions` → `collectContributingLines` / `matchArrayOperator`. Non-grouped `not_in` does not use this — tests for exclusion need `"grouped": true`.
- **Tests**: `loyalty/promotion_rule_test.go` — `TestNotInExcludePolicy_Checks_LegacyVsNew`, `TestNotInExcludePolicy_Engine_Integration`.
