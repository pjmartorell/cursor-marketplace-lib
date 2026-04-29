---
name: loyalty-engine-smart-policy-effects
description: Smart policy flag true. loyalty/effect_engine.go EffectType constants EffectTypePercentage Ratio DirectValue Strikethrough ComboPrice Formula Games. Wire new types register evaluator defaultEvaluators api/service/loyalty.go. ApplicationBuilder stackability skill.
---

# Smart policy: effects

Flag **`le.loyalty.new_smart_policy_loyalty_engine` = true**. Errors → `loyalty-engine-smart-policy`. Evaluators → `loyalty-engine-smart-policy-evaluators`. Stackability → `loyalty-engine-smart-policy-stackability`.

## Where

- Logic: [`loyalty/effect_engine.go`](../../../loyalty/effect_engine.go) — `EffectTypePercentage`, `EffectTypeDirectValue`, `EffectTypeStrikethrough`, `EffectTypeComboPrice`, `EffectTypeRatio`, `EffectTypeFormula`, `EffectTypeGames`, …
- Apply after groups: [`api/service/application_builder.go`](../../../api/service/application_builder.go) `Apply` (limits, ratio, line occupation — coordinate with stackability skill).

## New effect type

1. Constants/types in `loyalty/` matching JSON.
2. Wire in `effect_engine.go`.
3. Evaluator for `PromotionRule.Effect.Type` → register in [`api/service/loyalty.go`](../../../api/service/loyalty.go) `defaultEvaluators`.
4. If stackability/limits/lines change → `loyalty-engine-smart-policy-stackability`.
5. Unit + integration (flag **true**) when behaviour changes.
6. Business behaviour changes → flag at API/service, not inside `loyalty/`.
