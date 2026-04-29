---
name: add-config-driven-form
description: Add or extend a config-driven form using QuestionBase and QuestionControlService in the Loyal Guru owner app. Use when creating or editing forms with dynamic fields, multiselects, or when the user mentions app-question, QuestionControlService, or dynamic form.
---

# Add Config-Driven Form

## Pattern

1. **Define inputs**: array of `QuestionBase` subclasses (`TextboxQuestion`, `MultiSelectQuestion`, `TextareaQuestion`, `CheckboxQuestion`, etc.) from `src/app/shared/models/forms/`.
2. **Build form**: `this.form = this.qcs.toFormGroup(this.inputs)` where `qcs` is `QuestionControlService`.
3. **Resolve config in template**: `getInputConfig(key)` that returns `qcs.getInputCfgByKey(this.inputs, key)`.
4. **Template**: `<app-question [question]="getInputConfig('key')" [form]="form" />` inside a `<form [formGroup]="form">`.

## Multiselect

- Use `MultiSelectQuestion` with `dataSource` (service implementing `MultiselectDataSourceable`), `selectedIds`, and optional `settings` / `filters`.
- Handle `(multiselectChanged)` on `app-question` when selection drives other state.

## Validation and errors

- Set `question.required`, `question.min`, `question.max`, or `question.customValidators`.
- Paint API errors with `qcs.paintErrorsInForm(inputs, form, errors)`.

## Reference

- [docs/forms-multiselect.md](../../docs/forms-multiselect.md)
- `src/app/shared/services/question-control.service.ts`
- `src/app/shared/components/dynamic-form-question/`
