---
name: authorization
description: Manage Pundit policies, Role permissions, and access control. Use when creating, modifying, or debugging policies, permissions, roles, ACTIONS, CUSTOMER_ACTIONS, allowed?, or when the user mentions authorization, access control, or permissions.
---

# Authorization & Permissions

For full architecture details, see [/docs/AUTHORIZATION.md](/docs/AUTHORIZATION.md).

## Architecture

Authorization uses two systems that **must stay in sync**:

1. **`Role.permissions_table`** (`app/models/role.rb`) — maps `'Resource@action'` to roles for internal users.
2. **Pundit Policies** (`app/policies/<resource>_policy.rb`) — define access via `ACTIONS` (internal users) and `CUSTOMER_ACTIONS` (end customers). All inherit from `PolicyBase` (`app/policies/policy_base.rb`).

### How `method_missing` resolves access

`PolicyBase#method_missing` handles actions without explicit methods:
- **Admin**: allowed if action is in `ACTIONS`: Obsolete! don't use anymor for new developments
- **User**: allowed if action is in `ACTIONS` AND `permitted?(action)` (checks `Role.permissions_table`)
- **Customer**: allowed if action is in `CUSTOMER_ACTIONS`

## Adding or Modifying a Permission

Follow **all four steps** — skipping any creates security gaps:

```
Checklist:
- [ ] 1. Update `Role.permissions_table` in app/models/role.rb
- [ ] 2. Update `ACTIONS` in the policy (internal user actions)
- [ ] 3. Update `CUSTOMER_ACTIONS` in the policy (customer actions)
- [ ] 4. Add explicit policy method if ownership checks are needed
```

### Step 1: Role.permissions_table

```ruby
# app/models/role.rb -> self.permissions_table
'MyResource@create' => [:admin, :internal, :owner, :manager],
'MyResource@index'  => [:admin, :internal, :owner, :manager, :customer_service],
```

### Step 2–3: Policy constants

```ruby
class MyResourcePolicy < PolicyBase
  ACTIONS = [:index, :create, :show, :update, :delete].freeze
  CUSTOMER_ACTIONS = [:index, :show].freeze
end
```

### Step 4: Explicit method with `allowed?`

**Required** when `CUSTOMER_ACTIONS` includes mutating actions (`create`, `update`, `delete`).

```ruby
def create?
  return super unless @user.customer?
  customer_specific_checks && allowed?
end
```

`allowed?` enforces `@resource.customer_id == @user.id` for customers, preventing cross-customer access.

**Critical:** authorize with an **instance**, not the class:

```ruby
# Correct
record = MyModel.new(params)
authorize record, :create?

# Wrong — allowed? cannot check ownership
authorize MyModel, :create?
```

## Removing a Permission

Remove from **all** locations:

1. Remove endpoint from `app/api/v1/<resource>.rb`
2. Remove from `Role.permissions_table`
3. Remove from `ACTIONS` in the policy
4. Remove from `CUSTOMER_ACTIONS` in the policy (critical for security)

## Policy Patterns

### Basic policy (internal users only)

```ruby
class MyResourcePolicy < PolicyBase
  ACTIONS = [:index, :show, :create, :update].freeze
end
```

### Policy with customer access and ownership

```ruby
class MyResourcePolicy < PolicyBase
  ACTIONS = [:index, :show, :create, :update].freeze
  CUSTOMER_ACTIONS = [:index, :show, :create].freeze

  def create?
    return super unless @user.customer?
    allowed?
  end

  def show?
    super && allowed?
  end
end
```

### Policy with feature flag gating

```ruby
CUSTOMER_CREATE_FLAG = 'my_resource.customer_create'

def create?
  super && feature_enabled? && allowed?
end

private

def feature_enabled?
  Services::FeatureFlag::Client.get_flag_value(
    CUSTOMER_CREATE_FLAG, user_feature_flag, false
  )
end
```

### Policy with custom ownership (non-customer_id)

```ruby
def update?
  return super unless @user.customer?
  @resource.company_id == @user.company_id
end
```

## PolicyBase Key Methods

| Method | Purpose |
|--------|---------|
| `allowed?` | Ownership check: tenant for users, `customer_id` match for customers |
| `permitted?(action)` | Checks `Role.permissions_table` for current user |
| `in_tenant?` | Verifies `user.company_id == Company.current.id` |
| `admin?` | User is admin. Obsolete, do not use for new developments |
| `user_or_higher?` | User is admin or regular user. Obsolete, do not use for new developments  |
| `location_restricted?(location_id)` | Franchise user without access to location. Obsolete, do not use for new developments  |

## Testing

Every permission change requires updates to:

### 1. Role spec (`spec/models/role_spec.rb`)

- Update `total_count` for affected roles
- Update permission list in the relevant `context 'ModelName'` block

### 2. Policy spec (`spec/policies/<resource>_policy_spec.rb`)

```ruby
describe MyResourcePolicy do
  before do
    allow(LaunchDarkly::Client).to receive(:client)
      .and_return(double(variation: false))
  end

  permissions :create? do
    # Customer access
    include_examples 'customer access denied'
    # OR
    include_examples 'customer access allowed'

    # User access
    include_context 'user policies'
  end
end
```

### 3. Request specs — authorization is globally bypassed

`rails_helper.rb` replaces all policies with a permissive double for every request spec:

```ruby
# spec/rails_helper.rb
config.before(:each, type: :request) do
  policy = double('policy', method_missing: true)
  allow(PolicyBase).to receive(:new).and_return(policy)
end
```

This means **all `authorize` calls return `true` by default**. Both "allows access" and "denies access" tests are meaningless without opting back in to real policy logic.

To test real authorization, re-enable the specific policy with `.and_call_original`:

```ruby
context 'authorization' do
  before { allow(UserPolicy).to receive(:new).and_call_original }

  it 'allows access with the right permission' do
    role = create(:role, permissions: ['User@show'])
    user = create(:user, company: company, role: role)
    get_with_auth '/users/export_csv', user
    expect(response).to be_successful
  end

  it 'returns 401 without the right permission' do
    role = create(:role, permissions: [])
    user = create(:user, company: company, role: role)
    get_with_auth '/users/export_csv', user
    expect(response).to have_http_status(401)
  end
end
```

Prefer the specific policy class (`UserPolicy`, `SegmentPolicy`) over `PolicyBase` to avoid side effects on other policies called during the request.

### LaunchDarkly stub ordering

`let!` runs before `before` blocks defined after it. Always stub **before** any `let!` that creates users/customers:

```ruby
# Correct
before do
  allow(LaunchDarkly::Client).to receive(:client)
    .and_return(double(variation: false))
end
let!(:user) { create(:user) }

# Wrong — factory fires before stub
let!(:user) { create(:user) }
before { ... }
```

## Additional Resources

- PolicyBase source: `app/policies/policy_base.rb`
- Role permissions: `app/models/role.rb` → `self.permissions_table`
