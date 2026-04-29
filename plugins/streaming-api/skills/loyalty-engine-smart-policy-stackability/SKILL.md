---
name: loyalty-engine-smart-policy-stackability
description: Smart policy flag true. RuleGroupProcessingOrder in loyalty_rule.go root. processorsMap BestOfGroupProcessor PriorityOrderProcessor api/service/loyalty.go. New RuleGroupType streaming + order + register processor. ApplicationBuilder Apply line occupation merging promotions.
---

# Smart policy: stackability

Flag **`le.loyalty.new_smart_policy_loyalty_engine` = true**. Overview → `loyalty-engine-smart-policy`. Evaluators → `loyalty-engine-smart-policy-evaluators`. Effects → `loyalty-engine-smart-policy-effects`.

## Pipeline

- Order: `streaming.RuleGroupProcessingOrder` in [`loyalty_rule.go`](../../../loyalty_rule.go) (repo root).
- **Processors**: [`api/service/loyalty.go`](../../../api/service/loyalty.go) `processorsMap` — typically `BestOfGroupProcessor`, `PriorityOrderProcessor`.

## New `RuleGroupType`

1. Add type in `streaming`.
2. Append to `RuleGroupProcessingOrder` in [`loyalty_rule.go`](../../../loyalty_rule.go).
3. Register in `processorsMap` in [`api/service/loyalty.go`](../../../api/service/loyalty.go).

## `ApplicationBuilder`

[`api/service/application_builder.go`](../../../api/service/application_builder.go) `Apply` merges promotion results, `passLimits`, partial discount / ratio branches, occupied lines/fields, effects in `entityData`. Combine with `loyalty-engine-smart-policy-effects` when changing stacking or line consumption.
