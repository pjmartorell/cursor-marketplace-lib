---
name: loyalty-engine-legacy
description: Flag le.loyalty.new_smart_policy_loyalty_engine false. vouchers.Engine ApplyDiscounts api/tickets PolicyApplier scoreTicketService AddScores challenges GiveAwardChallenge service/loyalty triggerLegacyEngine rule creator validation. NOT loyalty PromotionEngine for ticket line discounts.
---

# Loyalty engine (legacy)

**Flag:** `le.loyalty.new_smart_policy_loyalty_engine` = **false**.

This path does **not** use `loyalty/` `PromotionEngine` / `Evaluator` / `effect_engine` for **ticket line discounts** (that is flag **true** — `loyalty-engine-smart-policy` + topic skills).

## Ticket discounts

- [`service/vouchers/engine.go`](../../../service/vouchers/engine.go) `ApplyDiscounts(ctx, company, ticket, opts)`.
- Entry [`api/tickets.go`](../../../api/tickets.go) when flag false — vouchers on `*streaming.Ticket`, not `ticket.ToMap` + rules engine.
- Validators: `service/vouchers/` (`ticket_validators`, `line_validators`, `voucher_validators`).

## Ticket points (scores)

- `loyalty_policies` → `PolicyApplier.Apply` → `scoreTicketService.AddScores`.
- [`api/tickets.go`](../../../api/tickets.go) legacy branch.

## Challenges

- `PolicyApplier` → winners → [`service/loyalty/engine.go`](../../../service/loyalty/engine.go) `GiveAwardChallenge`.
- [`api/challenges.go`](../../../api/challenges.go) `challengesLegacyEngine`.

`service/loyalty` = **awards** (post-policy), not root [`loyalty/`](../../../loyalty/) promotion package.

## HTTP trigger

[`api/internal/trigger.go`](../../../api/internal/trigger.go) `triggerLegacyEngine` when flag false.

## Rule creation / validation

[`service/rule/model.go`](../../../service/rule/model.go) `useNewSmartPolicyEngine`; [`service/rule/creator.go`](../../../service/rule/creator.go) branches (e.g. static voucher codes starting with `0` when flag off — legacy-only validation rules).

## Not this skill

`loyalty.Evaluator`, `PromotionEngine`, `effect_engine`, `ApplicationBuilder` for **ticket discounts** → smart policy (flag true).

## Tests

`ff.SetFlag("le.loyalty.new_smart_policy_loyalty_engine", false)`. `feature.NewStrictStub(t)` + every flag on path; fixtures: **`integration-testing-fixtures`** (double load).
