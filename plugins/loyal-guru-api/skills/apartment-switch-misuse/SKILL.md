---
name: apartment-switch-misuse
description: Identify unnecessary Apartment::Tenant.switch calls in services and models that cause query cache pollution and redundant DB overhead. Use when reviewing or writing Check*ConfigService classes, services that read Configuration records, or any code that calls Apartment::Tenant.switch to read tenant data. Also applies when the user mentions Apartment, tenant switch, query cache, or Configuration service performance.
---

# Apartment::Tenant.switch — Misuse Detection

## The Core Problem

`Apartment::Tenant.switch` (block form) always clears the query cache **twice**, even when switching to the tenant that is already active:

```ruby
# ros-apartment source: abstract_adapter.rb
def switch(tenant = nil)
  previous_tenant = current
  switch!(tenant)       # → clear_query_cache #1
  yield
ensure
  switch!(previous_tenant)  # → clear_query_cache #2
end

def switch!(tenant = nil)
  run_callbacks :switch do
    connect_to_new(tenant).tap do
      Apartment.connection.clear_query_cache  # always runs
    end
  end
end
```

`clear_query_cache` wipes all queries Rails has cached for the current request. Any query that would have been a cache hit after this point now hits the DB again.

## When the Switch IS Redundant

The switch is unnecessary when the service is **only ever called via `Company.current`**.

`Company.current` guarantees `company.slug == Apartment::Tenant.current` before returning:

```ruby
def self.current
  company = CurrentThread.company
  return company if company.present? && company.slug == current_tenant
  CurrentThread.company = Company.find_by(slug: current_tenant)
end
```

So `company_slug` passed to `EnforceGlobalMaskingConfigService.new(company_slug: slug)` **is** the current tenant by definition. The switch switches to where you already are — but still pays the full cache-clearing cost.

## How to Identify Redundant Switches

Check all callers of the method that invokes the service:

```bash
grep -rn "method_name\|ServiceClassName" app/
```

If **all callers** go through `company.some_method?` where `company` is `Company.current` (or any company whose slug is guaranteed to match `Apartment::Tenant.current`), the switch is redundant.

**Pattern that IS safe without a switch:**
```ruby
# Called only as Company.current.enforce_global_masking?
# → tenant already correct, switch unnecessary
def call
  config_value = Configuration.get_configuration_value(ENTITY, SLUG)
  return DEFAULT_VALUE if config_value.nil?
  config_value
end
```

**Pattern that NEEDS a switch:**
```ruby
# Called with arbitrary company_slug (e.g. background jobs, admin operations,
# login flows before tenant is set, iterating over companies)
def call
  Apartment::Tenant.switch(company_slug) do
    config_value = Configuration.get_configuration_value(ENTITY, SLUG)
    return DEFAULT_VALUE if config_value.nil?
    config_value
  end
end
```

## Performance Impact by Call Frequency

| Call context | Switch cost |
|---|---|
| Once per login (`CheckMFASignInConfigService`) | Low — 2 cache clears, acceptable |
| Once per request (company/user representer) | Low — manageable |
| Per field per customer (representer hot path) | **High** — N customers × M fields × 2 cache clears |

Always evaluate call frequency when deciding whether a redundant switch matters in practice.

## The Memoization Fix (when keeping the switch)

If the switch is genuinely needed but the method is on a hot path, memoize on the model object. Use `defined?` not `||=` to correctly handle `false`:

```ruby
def enforce_global_masking?
  return @enforce_global_masking if defined?(@enforce_global_masking)
  @enforce_global_masking = EnforceGlobalMaskingConfigService.new(company_slug: slug).call
end
```

Since `Company.current` returns the same object for the request lifetime, this collapses N service calls into 1 DB query per request.

## Summary: Decision Tree

```
Does the service always receive a company_slug that matches Apartment::Tenant.current?
│
├─ YES → Remove the switch. Query Configuration directly.
│        Add memoization on the caller if it's a hot path.
│
└─ NO  → Keep the switch (background jobs, admin cross-tenant, login flow).
         Add memoization if the method is called more than once per request.
```
