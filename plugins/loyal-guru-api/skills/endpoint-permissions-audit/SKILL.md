---
name: endpoint-permissions-audit
description: Audit API endpoint permissions and authorization security. Use when reviewing endpoint security, checking for missing authorize calls, auditing customer access, or when the user mentions permission audit, security review, endpoint authorization, or access control audit.
---

# Endpoint Permissions Audit

Security audit for Grape API endpoints. Analyzes authorization coverage, customer access, and permission gaps.

## When Triggered

The user provides one or more resource names (e.g. `recipes`, `scores`, `rewards`). Run the full audit for each.

## Arquitectura Multi-Tenant (contexto crítico)

Esta plataforma usa **Apartment** con **PostgreSQL schemas**. Cada company tiene su propio schema, por lo que **en la misma tabla no conviven registros de distintas compañías** — salvo los modelos excluidos.

### Modelos con tabla compartida entre compañías (`excluded_models`)

Definidos en `config/initializers/apartament.rb`:

```
Company, User, UserLocation, UserCountry, Admin, Jobtitude,
SystemStatus, SystemStatusLog, PasswordHistory, CompanyConfiguration,
TriggerType, TriggerAction, ScheduledExecution
```

Todo modelo que **no** esté en esta lista es **tenant-specific**: sus registros ya están aislados por schema, por lo que usar `authorize ModelClass, :action?` no supone riesgo de cross-tenant entre compañías.

### Aislamiento a nivel de customer (dentro del mismo tenant)

Aunque el schema aísla entre compañías, dentro del mismo tenant los registros de distintos customers **sí conviven en la misma tabla**. Modelos como `Profile`, `Reward`, `Score`, etc. tienen un campo `customer_id` (o `profile_id`) que los vincula a un customer concreto.

Por tanto, para endpoints accesibles por customers, usar `authorize ModelClass, :action?` en una acción mutante **sí es un riesgo**: el customer podría operar sobre registros de otro customer del mismo tenant si no se valida ownership vía la instancia.

### Regla de evaluación: clase vs instancia

| Tipo de modelo | Acción del endpoint | Customer puede llamarlo | ¿Clase OK? |
|---|---|---|---|
| Tenant-specific sin ownership de customer (`customer_id`/`profile_id`) | Cualquiera | No importa | ✅ Sin riesgo |
| Tenant-specific con `customer_id`/`profile_id` | No mutante (GET) | Sí | ✅ OK si el scope filtra por customer |
| Tenant-specific con `customer_id`/`profile_id` | Mutante (POST/PATCH/DELETE) | Sí | ⚠️ Riesgo — necesita instancia para verificar ownership |
| Shared model (`excluded_models`) | Cualquier mutación | Cualquiera | ⚠️ Riesgo cross-tenant |

## Audit Workflow

For each resource:

```
Checklist:
- [ ] 1. Read the endpoint file
- [ ] 2. Read the policy file
- [ ] 3. Check Role.permissions_table
- [ ] 4. Cross-reference and build the audit table
- [ ] 5. Report security observations
```

### Step 1: Read the endpoint file

Read `app/api/v1/<resource>.rb`. Extract every route block — identify:
- HTTP method (`get`, `post`, `patch`, `put`, `delete`)
- Route path (resource name + any nested `route_param` or named sub-routes)
- Description (`desc` string)
- Whether `authorize` is called, and if so:
  - What class/instance is authorized (e.g. `Recipe` vs `recipe`)
  - What action is checked (e.g. `:index?`, `:show?`, `:create?`)
  - **If `authorize` is called with a class** (e.g. `authorize Recipe, :create?`): consult the "Regla de evaluación: clase vs instancia" table in the "Arquitectura Multi-Tenant" section above to decide if this is a real problem or not. Do NOT flag it automatically — check: (1) is the model in `excluded_models`? (2) does the model have `customer_id`/`profile_id` and is it accessible by customers?
- **IDOR analysis** (see below): for endpoints accessible by customers, check if IDs from `params` are used to load records of other users without ownership validation

**Key red flag**: any route block that does NOT call `authorize` at all.

### Step 2: Read the policy file

Read `app/policies/<resource_singular>_policy.rb`. Extract:
- `ACTIONS` array — actions available to internal users (admin/user)
- `CUSTOMER_ACTIONS` array — actions available to customers
- Any explicit method overrides (e.g. `def create?`, `def show?`)
- Whether `allowed?` is called in overrides (ownership check)

If the policy file doesn't exist, flag it as a **critical security issue**.

### Step 3: Check Role.permissions_table

Search `app/models/role.rb` for entries matching the resource's model name (e.g. `'Recipe@'`). Note which actions have role restrictions and which roles are assigned.

### Step 4: Build the audit table

Generate a markdown table with this format:

```markdown
## Auditoría de permisos: <Resource>

| Endpoint | Método | Descripción | Tiene `authorize` | Llamable por cliente | Nombre del permiso | Observaciones de seguridad |
|----------|--------|-------------|-------------------|----------------------|--------------------|---------------------------|
| /resource | GET | All items | ✅ | ✅ | Resource@index | — |
| /resource/:id | GET | Get item | ✅ | ❌ | Resource@show | — |
| /resource | POST | Create item | ❌ | — | — | ⚠️ SIN AUTORIZACIÓN |
```

Column definitions:
- **Endpoint**: route path (e.g. `/recipes`, `/recipes/:id`)
- **Método**: HTTP verb
- **Descripción**: from the `desc` block
- **Tiene `authorize`**: ✅ if `authorize` is called, ❌ if missing
- **Llamable por cliente**: ✅ if the action is in `CUSTOMER_ACTIONS`, ❌ if not, `—` if no authorize
- **Nombre del permiso**: `ModelName@action` as it appears in `Role.permissions_table`, or `—` if not found
- **Observaciones de seguridad**: see detection rules below

### Step 5: Security observations

Flag these issues in the observations column:

| Code | Meaning | Severity |
|------|---------|----------|
| ⚠️ SIN AUTORIZACIÓN | Endpoint has no `authorize` call | Critical |
| ⚠️ SIN POLICY | No policy file exists for this resource | Critical |
| ⚠️ AUTHORIZE CON CLASE (modelo compartido) | Mutating action (create/update/delete) on a **shared model** (listed in `excluded_models` in `config/initializers/apartament.rb`) uses `authorize ModelClass` instead of instance — `allowed?` cannot check cross-tenant ownership | High |
| ⚠️ AUTHORIZE CON CLASE (customer mutable) | Mutating action on a tenant-specific model with `customer_id`/`profile_id` uses `authorize ModelClass` and the action is in `CUSTOMER_ACTIONS` — a customer could mutate another customer's record without ownership validation | High |
| ⚠️ SIN `allowed?` EN CUSTOMER_ACTION MUTABLE | Customer can call a mutating action but the policy has no explicit method with `allowed?` | High |
| ⚠️ NO EN permissions_table | Action exists in ACTIONS but has no entry in `Role.permissions_table` — all users pass `permitted?` | Medium |
| ⚠️ ACCIÓN EN POLICY SIN ENDPOINT | Action is in ACTIONS/CUSTOMER_ACTIONS but no corresponding endpoint exists | Low |
| ⚠️ ENDPOINT SIN ACCIÓN EN POLICY | Endpoint calls authorize with an action not listed in ACTIONS | Medium |
| ⚠️ POSIBLE IDOR | Endpoint accessible by customers uses an ID from `params` to load a record of another model without verifying it belongs to `current_user`. A customer could access or manipulate data of other customers | High |

#### IDOR Detection (Insecure Direct Object Reference)

When an endpoint's action is in `CUSTOMER_ACTIONS`, scan the endpoint body for this pattern:

1. A record is loaded using an ID from `params` (e.g. `Profile.find(params[:profile_id])`, `Customer.find(params[:customer_id])`)
2. That loaded record is used to query or return data
3. There is **no check** that the loaded record belongs to `current_user`

Valid ownership checks that neutralize IDOR:
- `authorize instance, :action?` where the policy's method calls `allowed?` (checks `customer_id == @user.id`)
- Explicit guard like `current_user.id == params[:customer_id]` or `record.customer_id == current_user.id`
- Scoping through `current_user` association (e.g. `current_user.profiles.find(...)`)

**Vulnerable example:**

```ruby
authorize Reward, :index?
profile = Profile.find(params[:profile_id])  # any customer can pass any profile_id
rewards = Reward.available_for_profile_and_user(profile, current_user)
```

The `authorize` passes (customer has `:index?`), but `profile_id` comes from untrusted input — a customer can query rewards for another customer's profile.

**Safe example:**

```ruby
authorize Reward, :index?
profile = current_user.profiles.find(params[:profile_id])  # scoped to current_user
rewards = Reward.available_for_profile_and_user(profile, current_user)
```

**Key principle**: any value from `params` is untrusted external input. When a customer-accessible endpoint uses `params[:*_id]` to load a record via `.find()` or `.where()`, and that record is not validated against `current_user`, flag it as IDOR.

## Output Format

For each finding, include:
- **Título**: nombre del problema, endpoint afectado y nivel de severidad (🔴 Critical / 🟠 High / 🟡 Medium / 🔵 Low)
- **Código**: extracto exacto del código donde se detecta el problema
- **Descripción**: explicación del hallazgo y por qué es un riesgo

After the table, add a summary section:

```markdown
### Resumen

- **Total endpoints**: X
- **Sin autorización**: X (list them)
- **Accesibles por cliente**: X
- **Problemas críticos**: X
- **Problemas altos**: X
- **Problemas medios**: X
```

If auditing multiple resources, produce one table per resource and a final consolidated summary.

## Reference

- Endpoints live in `app/api/v1/<resource>.rb`
- Policies live in `app/policies/<model_singular>_policy.rb`
- Role permissions are in `app/models/role.rb` → `self.permissions_table`
- For deeper understanding of the authorization architecture, read the [authorization skill](../authorization/SKILL.md)
- `PolicyBase` is at `app/policies/policy_base.rb`
