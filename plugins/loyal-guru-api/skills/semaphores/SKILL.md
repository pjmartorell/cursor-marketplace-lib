---
name: semaphores
description: Work with the Semaphore concurrency control system for Sidekiq workers. Use when debugging jammed semaphores, orphaned waiting entries, duplicate semaphore entries, releasing stuck processes, or when the user mentions semaphores, red/green light, waiting list, in_process, or SemaphoreControl.
---

# Semaphores — Worker Concurrency Control

Despite the name, this system is closer to a **cooperative job scheduler with mutual exclusion** than a classical semaphore. It controls which Sidekiq workers can run concurrently per company.

## Key files

| Area | Path |
|------|------|
| Worker mixin (DSL + lifecycle) | `app/workers/semaphore.rb` |
| Core state machine | `app/models/semaphore_control.rb` |
| State storage | `app/models/company.rb` → `semaphores` JSON column |
| Orphan detector | `app/services/semaphores/idle_detector.rb` |
| Detector worker (hourly, report-only) | `app/workers/idle_semaphore_detector_worker.rb` |
| Specs | `spec/workers/semaphore_spec.rb` |

## Architecture

### Storage

State lives in `company.semaphores`, a JSON column with two lists:

```json
{
  "in_process": [ { "slug": "pump", "process_key": "...", "parameters": [...], ... } ],
  "waiting":    [ { "slug": "pump", "process_key": "...", "parameters": [...], ... } ]
}
```

### Worker DSL

Workers `include Semaphore` and configure behavior via class macros:

```ruby
class PumpWorker < BaseWorker
  include Semaphore
  semaphore_slug :pump                    # which semaphore "lane"
  semaphore_red_lights :pump, :campaign   # block if any of these slugs are in_process
  semaphore_unique :params                # dedup by parameter content (not just slug)
end
```

| Macro | Purpose |
|-------|---------|
| `semaphore_slug` | Identifies which concurrency lane this worker belongs to |
| `semaphore_red_lights` | List of slugs that block this worker when in_process |
| `semaphore_unique` | `true` = one per slug, `:params` = one per slug+params combo |
| `sempahore_error_on_red_light` | Raise instead of retry on red light (note typo in method name) |
| `semaphore_options` | `on_background: true` for background-flagged processes |

### Lifecycle flow

```
perform_with_semaphore(args)
  ├── process_key_args(args)     → appends label-... and prokey-UUID
  ├── validate_uniqueness(args)  → checks if logically duplicate job exists
  ├── set_waiting(...)           → adds to waiting list
  └── perform_async(args)        → enqueues Sidekiq job
        └── perform(args)        → instance method, called by Sidekiq
              ├── set_waiting(...)
              ├── green_light?(args)
              │     ├── GREEN → move to in_process, run perform_semaphore
              │     └── RED   → perform_in(5.minutes, args)  (retry with polling)
              └── release()   → remove from in_process when done
```

### Uniqueness (`validate_uniqueness`)

Two modes controlled by `semaphore_unique`:

- **`true`** — compares by `slug|slug_alias` only. One job per worker type per company.
- **`:params`** — compares by `slug|slug_alias|normalized_params`. Allows multiple jobs of the same type if params differ (e.g., different BigqueryJob IDs).

Parameter normalization (after CDP-11242 fix):
```ruby
stripped.map { |p| p.is_a?(Hash) || p.is_a?(Array) ? p.to_json : p.to_s }
  .join.gsub('"', '').gsub(':', '')
```

### Process key vs logical identity

Each semaphore entry has a `process_key` that includes a UUID (`prokey`), making it globally unique. But **logical identity** (for deduplication) strips `prokey` and `label` prefixes, comparing only the core parameters.

## Common issues and debugging

### Orphaned semaphores (no matching Sidekiq job)

Detect with `IdleDetector`:
```ruby
in_process_idle, waiting_idle = Services::Semaphores::IdleDetector.new.run
```

This only **reports** orphans. To release them, see the [Monitoring Jammed Semaphores](https://loyal-guru.atlassian.net/wiki/spaces/CDP/pages/2916614146/Monitoring+Jammed+semaphores) Confluence doc.

### Duplicate waiting entries

Duplicates accumulate when `validate_uniqueness` fails to match (e.g., the Ruby 3.4 `Hash#to_s` regression — CDP-11242) or when `add_to_waiting_list` only checks `process_key` (which includes a unique UUID).

To scan for duplicates:
```ruby
Company.switch!('some_company')
company = Company.current
waiting = company.semaphores['waiting']

groups = waiting.group_by do |entry|
  klass = entry['klass']
  params = entry['parameters']&.reject { |p| p.nil? || p.to_s.start_with?('label-') || p.to_s.start_with?('prokey') } || []
  "#{klass}|#{params}"
end

groups.select { |_, v| v.size > 1 }.each do |key, entries|
  puts "#{key}: #{entries.size} duplicates"
end
```

### Manually blocking/unblocking a semaphore

```ruby
Company.switch!('demo1')
company = Company.current
sc = SemaphoreControl.new(company)

# Block all pump workers
sc.hold_by_slug('pump')

# Unblock
sc.release_by_slug('pump')

# Block everything (master hold)
sc.hold_master

# Release master hold
sc.release_master
```

### Inspecting a company's semaphore state

```ruby
Company.switch!('some_company')
c = Company.current
puts "in_process: #{c.semaphores['in_process'].size}"
puts "waiting: #{c.semaphores['waiting'].size}"

# By slug
c.actions_running_and_waiting_by(:slug)

# By params (for uniqueness debugging)
c.actions_running_and_waiting_by(:params)
```

### Emergency: clear all waiting entries

```ruby
Company.switch!('some_company')
company = Company.current
company.with_lock do
  company.semaphores['waiting'] = []
  company.save(validate: false)  # bypass validations for data-only fix
end
```

Use `save(validate: false)` when Company records may have stale validation errors (e.g., missing `status` on test companies).

## Known pitfalls

- **`save!` in `SemaphoreControl`**: All state mutations use `company.save!`, which triggers full model validations. Test companies with invalid data will raise `ActiveRecord::RecordInvalid`. Use `save(validate: false)` for manual console operations.
- **Polling, not signaling**: Jobs retry every 5 minutes via `perform_in` — there's no wake-up mechanism when a semaphore is released. This can cause up to 5 minutes of unnecessary delay.
- **Hash params and Ruby version**: The `validate_uniqueness` normalization relies on consistent Hash serialization. Any future Ruby `Hash#to_s` changes could break it again. The `.to_json` fix (CDP-11242) is version-safe.

## Related documentation

- [Monitoring Sidekiq](https://loyal-guru.atlassian.net/wiki/spaces/CDP/pages/4325769233/Monitoring+Sidekiq) — operational scripts for queue cleanup, deduplication
- [Monitoring Jammed Semaphores](https://loyal-guru.atlassian.net/wiki/spaces/CDP/pages/2916614146/Monitoring+Jammed+semaphores) — how to release orphaned semaphores
