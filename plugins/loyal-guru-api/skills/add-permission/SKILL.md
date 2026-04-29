---
name: add-permission
description: Create a new permission and open a standalone PR from master following the loyal-guru-api convention. Covers role.rb (old roles system), i18n locale files, role_spec.rb count updates, and rake task documentation (new roles system). Use when the user asks to add a permission, create a permissions PR, register a new Resource@action, or run permissions:create_permission rake tasks.
---

# Add Permission

## Overview

A permission PR always branches from `master` (never from a feature branch), follows the pattern established by PRs like #7932, and is separate from the feature that uses the permission.

The two systems involved:
- **Old roles system**: `role.rb` `permissions_table` hash — updated in code.
- **New roles system**: `Permission` records in the database — created via rake tasks post-deploy, documented in the PR description.

## Workflow

### 1. Create branch from master

```bash
git fetch origin master
git checkout -b PJM-<TICKET>-<descriptor> origin/master
```

### 2. Add to `app/models/role.rb`

Find the right alphabetical position in `permissions_table` and add:

```ruby
'Resource@action' => [:admin, :internal, :owner],  # adjust roles as needed
```

Common role sets:
- Admin-only: `[:admin, :internal]`
- Owner-facing: `[:admin, :internal, :owner]`
- Broad: `[:admin, :internal, :owner, :manager]`

### 3. Add i18n translations

In all 6 locale files (`en`, `es`, `fr`, `hr`, `it`, `ro`) under `config/locales/`, add the permission keys inside the `permissions:` block. The i18n key is derived from the identifier:

`Resource@action` → `permissions.<resource_downcased>.<action>.name` (and `.description` if `description=true` in the rake task)

Find the alphabetical insertion point (e.g. `usersecurity` goes after `usergroup`):

```yaml
    <resourcedowncased>:
      <action>:
        name: Human Readable Name
        description: What this permission allows.  # only when description=true
```

For non-English locales, copy the English strings — translations are done separately.

### 4. Update `spec/models/role_spec.rb` counts

For every role that receives the new permission, increment the `total_count` assertion:

```bash
grep -n "total_count\|starters\[" spec/models/role_spec.rb | grep "size.*eq"
```

Add the number of new permissions per role (usually +1 per permission added).

Verify counts:

```bash
bundle exec rspec spec/models/role_spec.rb -e "total_count"
```

All examples must pass before committing.

### 5. Commit and push

```bash
git add app/models/role.rb spec/models/role_spec.rb config/locales/*.yml
git commit -m "feat(<TICKET>): add <Resource@action> permission"
git push -u origin <branch>
```

### 6. Create the PR

Use the PR template. The PR description **must** include the rake tasks under a dedicated block:

```markdown
**New roles system** — run post-deploy against the target environment:

\`\`\`
rake 'permissions:create_permission[Resource@action,<group_slug>,<basic>,<visible>,<description>,role1,role2,...]'
\`\`\`
```

Rake task argument reference (from `lib/tasks/permissions.rake`):

| Arg | Description |
|-----|-------------|
| `identifier` | `Resource@action` format |
| `group_slug` | Existing `PermissionGroup` slug (e.g. `configuration_user_security`, `customers`) — leave empty to skip |
| `basic` | `true`/`false` — included in basic plans |
| `visible` | `true`/`false` — shown in permissions UI |
| `description` | `true`/`false` — auto-generates i18n description key |
| `role1,role2,...` | `GlobalRole` names (e.g. `admin_legacy`, `super_admin_lg`, `owner_legacy`, `owner`, `internal_legacy`, `internal_lg`) |

Create a draft PR; mark ready and assign a `deploy_staging*` label only after CircleCI passes.

## Key files

| File | Purpose |
|------|---------|
| `app/models/role.rb` | Old roles system permissions table |
| `config/locales/{en,es,fr,hr,it,ro}.yml` | Permission i18n names/descriptions |
| `spec/models/role_spec.rb` | Role permission count assertions |
| `lib/tasks/permissions.rake` | `permissions:create_permission` task definition |

## Common mistakes

- **Never** create the permissions branch from a feature branch — always from `master`.
- **Never** skip updating the spec counts; CI will fail.
- The rake task runs in `Apartment::Tenant.switch('public')` — it targets the shared schema, not a tenant.
- If `group_slug` is passed, the `PermissionGroup` must already exist in the DB; otherwise the task aborts.
