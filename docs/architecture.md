# Inventory Management System — Architecture Specification v1.0

**Status:** Approved for Implementation Planning
**Project Type:** Multi-tenant, offline-capable inventory management system
**Stack:** Next.js (PWA) · Spring Boot · PostgreSQL/Supabase · Supabase Auth · n8n (automation, future)

This document is the final Architecture Specification v1.0, incorporating all architectural decisions made during review: current-stock strategy, tenant isolation, authentication, offline MVP scope, recipe versioning, stock-level projection consistency, session/refresh/revocation design, RLS testing discipline, conflict review process, ingredient deactivation behavior, and currency/locale approach. It is the implementation-ready baseline.

---

## 1. Core Principles

1. Multi-tenant by design — every tenant-scoped row carries `organization_id`.
2. The `inventory_transactions` ledger is the single source of truth for stock. `stock_levels` is a cache, updated synchronously in the same DB transaction as the ledger insert, never authoritative history on its own.
3. Spring Boot is the primary authorization and business-rule enforcement layer. PostgreSQL Row Level Security (RLS) is defense-in-depth, not the primary control, and is proven via mandatory cross-tenant test coverage.
4. Supabase Auth is the identity provider. Spring Boot owns roles, permissions, organization/branch access, and issues its own short-lived, revocable session tokens.
5. Offline-first for a defined, practical operation set — not "everything, eventually." Genuine sync conflicts are flagged for human review, not silently resolved.
6. Recipes are versioned; production always references the recipe version active at the time. Ingredient deactivation is a soft flag that never breaks historical records.
7. Pricing is currency-agnostic in the schema (decimal + ISO currency code), defaulting to PHP for MVP without assuming a single currency long-term.
8. AI/n8n/Node-RED are integration/automation layers, never sources of truth for inventory.

---

## 2. High-Level Architecture

```
                         USERS (mobile / tablet / desktop)
                                     │
                                     ▼
                          Next.js PWA (Frontend)
                          ┌─────────┴─────────┐
                          │                   │
                     Local IndexedDB     Online API calls
                    (outbox + cache)          │
                          │                   ▼
                          │           Spring Boot API
                          │        (business rules, authZ)
                          │                   │
                          │                   ▼
                          │            PostgreSQL (Supabase)
                          │           + Row Level Security
                          │                   │
                          └──────Sync─────────┘
                                     │
                                     ▼
                              Supabase Auth
                          (identity / login / tokens)
                                     │
                                     ▼
                                    n8n
                            (automation, notifications)
                                     │
                          ┌──────────┴──────────┐
                          ▼                     ▼
                      Node-RED                 AI
                    (future, IoT)      (future, insights only)
```

**Responsibility boundary:**
- **Supabase Auth** — issues identity tokens only. Does not own roles/permissions.
- **Spring Boot** — validates the Supabase-issued token, resolves org/branch/role context, enforces all business rules, is the only writer of `inventory_transactions`.
- **PostgreSQL/Supabase** — storage + RLS as a second line of defense against cross-tenant leaks (e.g., a missed `WHERE org_id = ?` in a query still can't return another tenant's rows).
- **Next.js** — UI, offline cache, outbox queue. Never a source of truth for business calculations.

---

## 3. Domain Model

```
Organization (tenant root)
 ├─ Branch (1..N)
 │   └─ InventoryLocation (1..N)          [Main Storage, Kitchen, Freezer...]
 ├─ Device (registered per branch)
 ├─ User (identity = Supabase Auth user id)
 │   └─ UserBranchAccess (user, branch, role)
 ├─ Role (Owner / Manager / Staff — org-scoped, some roles hidden per org config)
 ├─ Ingredient
 │   ├─ UnitConversion (ingredient, from_unit, to_unit, factor)
 │   └─ SupplierIngredient (ingredient, supplier, price, unit, effective_from)
 ├─ Supplier
 ├─ Recipe (versioned)
 │   └─ RecipeVersion
 │        └─ RecipeIngredient (recipe_version, ingredient, qty, unit)
 ├─ Purchase
 │   └─ PurchaseLine (purchase, ingredient, qty, unit, price)
 ├─ ProductionRun (references a specific RecipeVersion, not just Recipe)
 │   └─ ProductionConsumption (production_run, ingredient, qty_consumed)
 ├─ InventoryTransaction (append-only ledger — source of truth)
 ├─ StockLevel (cached projection, derived only)
 └─ AuditLog (generic, references any entity)
```

**Key modeling decision — Recipe versioning:** `recipes` holds identity/metadata (name, org, active flag). Each edit creates a new `recipe_versions` row. `production_runs.recipe_version_id` is fixed at production time, so a later recipe edit never rewrites history — production records always show exactly what was consumed and why.

---

## 4. Entity-Relationship Diagram (Full)

```
organizations
  id PK
  name
  created_at

branches
  id PK
  organization_id FK -> organizations.id
  name
  UNIQUE (organization_id, name)

inventory_locations
  id PK
  branch_id FK -> branches.id
  name
  UNIQUE (branch_id, name)

users
  id PK                      -- matches Supabase Auth user id
  organization_id FK -> organizations.id
  full_name
  status (active/disabled)
  created_at

roles
  id PK
  organization_id FK -> organizations.id
  name (Owner / Manager / Staff / custom)
  is_visible boolean          -- org can hide unused roles

user_branch_access
  id PK
  user_id FK -> users.id
  branch_id FK -> branches.id
  role_id FK -> roles.id
  UNIQUE (user_id, branch_id)

devices
  id PK
  organization_id FK -> organizations.id
  branch_id FK -> branches.id
  device_key UNIQUE            -- server-issued credential, not client-generated
  label
  registered_at
  last_seen_at

ingredients
  id PK
  organization_id FK -> organizations.id
  name
  base_unit                    -- canonical unit for internal storage (e.g. gram)
  min_stock_level nullable
  max_stock_level nullable
  status (active/inactive)
  UNIQUE (organization_id, name)

unit_conversions
  id PK
  ingredient_id FK -> ingredients.id
  unit
  factor_to_base                -- e.g. "tub" -> 5000 (grams)
  UNIQUE (ingredient_id, unit)

suppliers
  id PK
  organization_id FK -> organizations.id
  name
  contact_info
  status (active/inactive)

supplier_ingredients
  id PK
  supplier_id FK -> suppliers.id
  ingredient_id FK -> ingredients.id
  unit
  price                          -- stored as decimal, currency-agnostic
  currency_code                  -- ISO 4217, default 'PHP' for MVP
  effective_from date
  supplier_sku nullable
  INDEX (ingredient_id, effective_from DESC)   -- latest price lookup

recipes
  id PK
  organization_id FK -> organizations.id
  name
  status (active/inactive)
  current_version_id FK -> recipe_versions.id nullable

recipe_versions
  id PK
  recipe_id FK -> recipes.id
  version_number
  created_at
  created_by FK -> users.id
  UNIQUE (recipe_id, version_number)

recipe_ingredients
  id PK
  recipe_version_id FK -> recipe_versions.id
  ingredient_id FK -> ingredients.id
  quantity
  unit

purchases
  id PK
  organization_id FK -> organizations.id
  branch_id FK -> branches.id
  supplier_id FK -> suppliers.id
  user_id FK -> users.id
  device_id FK -> devices.id nullable
  reference_number nullable
  currency_code                  -- ISO 4217, default 'PHP' for MVP
  created_at

purchase_lines
  id PK
  purchase_id FK -> purchases.id
  ingredient_id FK -> ingredients.id
  quantity
  unit
  unit_price                     -- decimal, in purchases.currency_code

production_runs
  id PK
  organization_id FK -> organizations.id
  branch_id FK -> branches.id
  recipe_version_id FK -> recipe_versions.id     -- fixed snapshot reference
  quantity_produced
  user_id FK -> users.id
  device_id FK -> devices.id nullable
  status (completed / failed_insufficient_stock)
  created_at

production_consumptions
  id PK
  production_run_id FK -> production_runs.id
  ingredient_id FK -> ingredients.id
  quantity_consumed
  unit

inventory_transactions          -- APPEND-ONLY LEDGER, SOURCE OF TRUTH
  id PK
  organization_id FK -> organizations.id
  branch_id FK -> branches.id
  location_id FK -> inventory_locations.id
  ingredient_id FK -> ingredients.id
  type (purchase / production_consumption / adjustment / correction /
        transfer / return / waste / opening_balance)
  quantity_delta                 -- signed, in base_unit
  reference_type nullable        -- 'purchase' / 'production_run' / 'adjustment' etc.
  reference_id nullable
  user_id FK -> users.id
  device_id FK -> devices.id nullable
  reason nullable                -- required for manual adjustment/correction
  sync_op_id UUID nullable       -- client-generated, for idempotency
  status (applied / flagged_for_review)
  created_at
  UNIQUE (organization_id, sync_op_id)   -- enforces idempotent sync
  INDEX (organization_id, location_id, ingredient_id, created_at)

stock_levels                     -- CACHE ONLY, derived from ledger
  organization_id FK -> organizations.id
  location_id FK -> inventory_locations.id
  ingredient_id FK -> ingredients.id
  quantity_cached
  last_transaction_id FK -> inventory_transactions.id
  updated_at
  PRIMARY KEY (organization_id, location_id, ingredient_id)

audit_logs
  id PK
  organization_id FK -> organizations.id
  user_id FK -> users.id nullable
  entity
  entity_id
  action
  previous_value jsonb nullable
  new_value jsonb nullable
  reason nullable
  created_at
  INDEX (organization_id, entity, entity_id, created_at)
```

**Constraints worth calling out explicitly:**
- `inventory_transactions` is **insert-only** at the application layer — no UPDATE/DELETE grants for the app's DB role. Corrections are new rows referencing the original (`reference_type = 'correction'`, `reference_id = original_transaction_id`).
- `stock_levels` is recomputable at any time by replaying `inventory_transactions` — treat it as a materialized view conceptually, even if implemented as a real table for write performance.
- Every tenant-scoped table has `organization_id` directly (not just via join) so RLS policies can filter without multi-hop joins.
- `ingredients.status = 'inactive'` is a **soft flag**, never a delete. Historical rows in `recipe_ingredients`, `purchase_lines`, `production_consumptions`, and `inventory_transactions` keep their FK to the ingredient regardless of its current status — historical reads are unaffected. Application-layer validation (not a DB constraint) blocks *new* recipe versions, purchases, or production runs from selecting an inactive ingredient, unless explicitly reactivated first.
- Pricing (`supplier_ingredients.price`, `purchase_lines.unit_price`) is stored as a plain decimal plus an explicit `currency_code` column, rather than assuming a single currency. MVP defaults every org to `PHP`, but nothing in the ledger or inventory model assumes a currency — multi-currency later means adding per-org currency configuration and FX-aware reporting, not restructuring `inventory_transactions` or `stock_levels` (which never store money, only quantities).

---

## 5. Authentication & Authorization Flow

**Identity provider:** Supabase Auth (handles login, password/OAuth, token issuance, session refresh).
**Authorization owner:** Spring Boot (roles, permissions, org/branch access, all business rules).

### 5.1 Online login flow

```
1. User submits credentials to Supabase Auth (via Next.js)
2. Supabase Auth returns a signed JWT (contains Supabase user id, email)
3. Next.js calls Spring Boot with the Supabase JWT in Authorization header
4. Spring Boot verifies the JWT signature against Supabase's public JWKS
5. Spring Boot looks up `users` by Supabase user id -> resolves organization_id
6. Spring Boot looks up `user_branch_access` -> resolves role(s) per branch
7. Spring Boot issues its own short-lived application session (see 5.1.1)
8. Client uses the Spring-issued access token for all subsequent API calls
```

Rationale for step 7: embedding org/branch/role directly in a Supabase-controlled token would require Spring Boot to trust claims it didn't issue for authorization decisions. Re-issuing a Spring-signed token keeps Spring Boot as the sole authority over *authorization* claims while Supabase remains sole authority over *identity*.

### 5.1.1 Session, refresh, and revocation design

Spring Boot issues **two** tokens at step 7, mirroring the standard access/refresh pattern:

```
access_token
  - Spring-signed JWT
  - short-lived: 10–15 minutes
  - claims: user_id, organization_id, branch_roles[], token_version
  - used on every API request, verified statelessly (signature + expiry)

refresh_token
  - opaque, random, high-entropy string (not a JWT)
  - stored server-side in a `refresh_tokens` table:
      id, user_id, token_hash, device_id nullable, issued_at,
      expires_at (e.g. 7–30 days), revoked_at nullable
  - stored client-side only in a secure, httpOnly-equivalent location
    (secure storage on mobile/PWA context)
```

**Refresh flow:**
```
Client -> POST /api/session/refresh { refresh_token }
Spring Boot -> hash incoming token, look up refresh_tokens row
  - not found / revoked / expired -> 401, client forced to re-login
  - valid -> re-check users.status = 'active' and current
    user_branch_access (roles may have changed since last issue)
  - issue new access_token (fresh 10–15 min window, current claims)
  - rotate refresh_token: revoke old row, insert new row
    (rotation on every use — detects token theft: a reused old
    refresh_token after rotation is treated as a compromise signal
    and revokes the entire token family for that user)
```

**Why short-lived access + rotating refresh, instead of one long-lived token:** a 10–15 minute access token means a disabled user's existing token expires and is *not* renewed (refresh re-checks `users.status` every time) — so authorization goes stale within minutes, not hours or days, without requiring an active revocation list to be checked on every single request.

**Revocation (explicit, not just expiry):**
```
Admin disables a user -> users.status = 'disabled'
  -> next refresh attempt for that user is rejected (checked in refresh flow above)
  -> existing access_token remains valid only until its own short expiry lapses
     (max exposure window = access token lifetime, e.g. 15 minutes)

Admin revokes a specific session/device -> mark that refresh_tokens row revoked_at
  -> that device can no longer refresh; other devices/sessions for the
     same user are unaffected unless explicitly revoked too

"Revoke all sessions" (e.g. suspected compromise) -> bulk revoke all
  refresh_tokens rows for that user_id
```

**Sensitive operations re-check server state, not just the token:** for high-impact actions — inventory adjustments/corrections, production runs, purchases, role changes — Spring Boot re-reads `users.status` and current `user_branch_access` from the database at request time rather than relying solely on the branch-roles claim embedded in the access token. This closes the gap where a role change or user disablement happens mid-way through a still-valid 15-minute access token window. Low-sensitivity reads (e.g., viewing cached inventory) may rely on the token claims alone for performance.

### 5.2 Device registration flow

```
1. On first use, device requests registration: POST /api/devices/register
   (authenticated user + branch context required)
2. Spring Boot generates a server-issued device_key, stores it against
   organization_id + branch_id
3. device_key is stored locally on the device (not user-guessable, not
   client-generated) and attached to all subsequent writes from that device
```

This prevents device spoofing — a device identity can't be fabricated client-side.

### 5.3 Offline authentication

```
- On successful online login, Spring Boot also issues an offline-capable
  token with a longer expiry (e.g. 12–24h, configurable per org) scoped
  to read + queue-writes only — this token cannot itself authorize
  synchronization; sync requests are re-validated fully once online.
- While offline, the device uses the cached token to permit local UI
  actions (view cached inventory, queue adjustments/production).
- No new user can log in on a device while offline unless that user has
  a previously cached valid offline token issued for that device.
- All offline-queued writes carry the user_id from the token that was
  valid when the action was performed — this is validated again at
  sync time, and rejected if that token/session has since been revoked.
```

---

## 6. Multi-Tenant Isolation Strategy

**Primary enforcement — Spring Boot:**
- Every repository query is scoped by `organization_id` derived from the authenticated session context, never from client-supplied input.
- A shared base repository/interceptor pattern injects `organization_id` into every query automatically, so individual developers can't "forget" the filter.

**Defense-in-depth — PostgreSQL RLS:**
- RLS policies enabled on every tenant-scoped table: `USING (organization_id = current_setting('app.current_org_id')::uuid)`.
- Spring Boot sets `app.current_org_id` as a session-local Postgres variable at the start of each request-scoped transaction.
- RLS is **not** used to make authorization decisions (e.g., role checks) — only tenant boundary enforcement. Role/permission logic stays entirely in Spring Boot to avoid the two-systems-drift problem.
- This means: even if a Spring Boot query has a bug and omits the org filter, the database itself refuses to return cross-tenant rows.

**Explicitly not doing:** Supabase RLS-based *authorization* (role-based row access) — that logic lives only in Spring Boot, so there is one place, not two, where "can this user do X" is decided.

### 6.1 RLS Testing (mandatory)

RLS is only a real defense if it's proven to work, so it ships with a dedicated, required test suite — not an optional nice-to-have:

```
For every tenant-scoped table:
  1. Seed two organizations (Org A, Org B) with equivalent rows.
  2. Open a DB session as Org A (SET app.current_org_id = <org_a_id>).
  3. Attempt SELECT * FROM <table> — assert ONLY Org A rows returned,
     never Org B rows, even with no WHERE clause at all.
  4. Attempt UPDATE/DELETE targeting an Org B row's known id directly
     — assert 0 rows affected (not an error — RLS silently excludes it).
  5. Attempt INSERT with organization_id = Org B while session is
     scoped to Org A — assert rejected by the RLS WITH CHECK clause.
```

This suite runs against every table in §4 as part of CI (using a real Postgres test container, not mocks — RLS policies cannot be validated against an in-memory or mocked database). A schema migration that adds a new tenant-scoped table without an accompanying RLS test is treated as incomplete, not merely "pending."

---

## 7. Offline Synchronization

### 7.1 Scope (MVP)

Included offline:
- View cached inventory (last-synced snapshot)
- Stock adjustments/corrections
- Production runs
- Purchases (queued)

Explicitly deferred to Phase 2:
- Automatic conflict resolution beyond delta-summing
- Multi-branch transfer operations while offline
- Real-time cross-device inventory visibility while offline

### 7.2 Sequence — Online write

```
Client -> POST /api/inventory/adjustments (sync_op_id generated client-side)

Spring Boot, within a SINGLE database transaction:
  1. validate business rules
  2. INSERT inventory_transactions (status=applied)
  3. UPSERT stock_levels (quantity_cached += delta, last_transaction_id = new id)
  4. INSERT audit_logs
  -> COMMIT (all four succeed together, or ALL roll back)

Spring Boot -> 200 OK { transaction_id, status: applied }
```

**Consistency rule:** steps 2–4 above execute inside one DB transaction (`@Transactional` at the service layer). If the `stock_levels` upsert fails for any reason, the `inventory_transactions` insert is rolled back too — the ledger and its cache can never diverge. Because `stock_levels` is fully recomputable from the ledger, this is also the recovery path: if the cache is ever suspected to be wrong, it can be safely rebuilt with `SELECT SUM(quantity_delta) ... GROUP BY organization_id, location_id, ingredient_id` without touching the ledger.

### 7.3 Sequence — Offline write, then reconnect

```
[OFFLINE]
Client generates sync_op_id (UUID) at time of action
Client writes to local outbox (IndexedDB), status = pending
UI shows: 🟡 Pending synchronization

[RECONNECT]
Client -> POST /api/sync  { operations: [ {sync_op_id, type, payload}, ... ] }
Spring Boot, per operation, in received order:
  1. Check UNIQUE(organization_id, sync_op_id) in inventory_transactions
     -> if exists: return { sync_op_id, status: already_applied } (idempotent no-op)
  2. Re-validate business rules against CURRENT server state
     (e.g., would this drive stock negative)
  3. If valid: INSERT inventory_transactions, update stock_levels
     -> return { sync_op_id, status: applied }
  4. If it's a correction whose "previous_quantity" no longer matches
     current server state: do NOT apply
     -> INSERT as status=flagged_for_review
     -> return { sync_op_id, status: conflict_flagged }
Client updates local outbox per response:
  applied -> 🟢 Synchronized
  already_applied -> 🟢 Synchronized (dedup, no user-visible change)
  conflict_flagged -> 🔴 Needs review (surfaced to Manager/Owner)
```

### 7.4 Conflict Handling Rules

| Scenario | Rule |
|---|---|
| Two devices offline, both record **delta** operations (purchase, production, waste) against the same ingredient | Both apply — deltas are commutative, sum correctly regardless of order. No conflict. |
| Two devices offline, both record a **manual correction** (absolute value set) against the same ingredient | Second one to sync is compared against the server's `previous_quantity` at that point. If mismatched, flagged for review — never silently applied. |
| Offline production would drive stock negative once server-side deltas are applied | Rejected at sync time (`status: failed_insufficient_stock`), surfaced to the user, not silently clamped to zero. |
| Same `sync_op_id` submitted twice (retry) | No-op, returns `already_applied`. |
| Device was offline long enough that its cached auth token expired | Sync request rejected with re-auth required; queued items remain pending locally until the device re-authenticates. |

### 7.5 Conflict Review Process (MVP)

Flagged-for-review items appear in a dedicated review queue, visible only to Manager/Owner roles, showing:
- Both conflicting values (server's current state vs. the rejected offline correction)
- Who performed each, on which device, at what time
- The ingredient/location context

Resolution flow:
```
Manager/Owner opens the flagged item -> chooses the correct final quantity
  (may match either side, or be a new value entirely)
Spring Boot -> INSERT a new inventory_transactions row
  (type = 'correction', reference_type = 'conflict_resolution',
   reference_id = the flagged transaction's id, reason = required text)
Spring Boot -> mark the original flagged_for_review row's status
  = 'resolved' (the flagged row itself is never edited or deleted —
   it remains as a permanent record that a conflict occurred)
Spring Boot -> INSERT audit_logs entry for the resolution action
```

This keeps the ledger's insert-only guarantee intact even for conflict resolution — nothing is ever rewritten, only appended to. There is currently no enforced SLA on how quickly a flagged item must be resolved; it remains visible in the queue indefinitely until acted on (tracked as an open item in §14).

---

## 8. API Boundaries

```
Auth
  (Supabase Auth handles login directly — Next.js talks to Supabase SDK)
  POST /api/session/exchange        -- exchange Supabase JWT for Spring access+refresh tokens
  POST /api/session/refresh         -- rotate refresh token, issue new access token
  POST /api/session/revoke          -- revoke current device session
  POST /api/session/revoke-all      -- revoke all sessions for the authenticated user

Devices
  POST /api/devices/register

Organizations / Branches / Locations
  GET/POST/PUT   /api/organizations/{id}
  GET/POST/PUT   /api/branches
  GET/POST/PUT   /api/branches/{id}/locations

Users / Roles / Access
  GET/POST/PUT   /api/users
  GET/POST/PUT   /api/roles
  GET/POST/PUT   /api/users/{id}/branch-access

Ingredients / Units
  GET/POST/PUT   /api/ingredients
  PUT            /api/ingredients/{id}/status        -- activate/deactivate (soft flag only)
  GET/POST/PUT   /api/ingredients/{id}/conversions

Suppliers
  GET/POST/PUT   /api/suppliers
  GET/POST/PUT   /api/suppliers/{id}/ingredients

Purchases
  GET/POST       /api/purchases

Recipes
  GET/POST       /api/recipes
  POST           /api/recipes/{id}/versions        -- creates new version
  GET            /api/recipes/{id}/versions/{v}

Production
  POST           /api/production                    -- validates stock, creates run
  GET            /api/production

Inventory
  GET            /api/inventory/stock                -- reads stock_levels
  GET            /api/inventory/transactions          -- ledger, audit view
  POST           /api/inventory/adjustments            -- manual correction, reason required

Sync
  POST           /api/sync                            -- batched offline replay
  GET            /api/sync/conflicts                  -- flagged-for-review queue (Manager/Owner)
  POST           /api/sync/conflicts/{id}/resolve      -- resolve, creates new ledger entry

Audit
  GET            /api/audit
```

Every endpoint requires the Spring-issued session token; org/branch context resolved server-side, never trusted from request body/query params.

---

## 9. Spring Boot Package Structure

```
backend/
└── src/main/java/com/application/
    ├── auth/                 -- Supabase token verification, session issuance
    ├── organization/
    ├── branch/
    ├── device/
    ├── user/
    ├── role/
    ├── ingredient/
    ├── unit/
    ├── supplier/
    ├── purchase/
    ├── recipe/                -- recipes + recipe_versions
    ├── production/
    ├── inventory/              -- ledger, stock_levels projection, adjustments
    ├── sync/                   -- idempotency, conflict flagging, /api/sync
    ├── audit/
    └── common/
        ├── security/           -- RLS session var setter, tenant context filter
        ├── exception/
        └── validation/
```

Each domain module: `controller/`, `service/`, `repository/`, `domain/`, `dto/`, `mapper/`.

---

## 10. Next.js Folder Structure

```
frontend/
├── app/
├── features/
│   ├── auth/
│   ├── inventory/
│   ├── ingredients/
│   ├── suppliers/
│   ├── purchases/
│   ├── recipes/
│   ├── production/
│   ├── branches/
│   └── users/
├── components/
├── lib/
│   ├── api/                  -- Spring Boot API client
│   ├── supabase/              -- Supabase Auth client
│   ├── local-db/               -- IndexedDB/Dexie schema (cache + outbox)
│   ├── sync/                    -- outbox queue, retry, status tracking
│   └── validation/
├── hooks/
├── types/
└── public/
```

---

## 11. MVP vs Phase 2

### MVP
- Auth (Supabase Auth + Spring session exchange), roles, branch access
- Organizations, branches, locations, device registration
- Ingredients, units, unit conversion
- Suppliers, supplier pricing, purchases
- Recipes (versioned), production with stock validation
- Ledger-based inventory transactions + stock_levels projection
- Manual adjustments/corrections with mandatory reason
- Audit trail
- Offline: outbox + idempotent sync for adjustments/production/purchases, conflict flagging (not auto-resolution)
- Search/filter/sort on inventory
- Responsive UI, installable PWA

### Phase 2
- Automatic conflict resolution beyond delta-summing
- Inventory transfers between branches/locations
- Advanced reporting, stock forecasting
- Notifications, n8n automation workflows
- Advanced permission granularity

### Future
- AI insights/natural-language queries (recommend-only, never authoritative)
- Node-RED / IoT / barcode / digital scale integrations

---

## 12. Security Considerations

- No secrets in source control; environment variables / secret manager for Supabase keys, DB credentials, JWT signing keys.
- `inventory_transactions` insert-only at the DB role level (no UPDATE/DELETE grants).
- RLS enabled on all tenant tables as defense-in-depth (see §6), with mandatory cross-tenant-access test coverage (see §6.1) — no new tenant-scoped table ships without it.
- Access tokens are short-lived (10–15 min); refresh tokens are rotated on every use and revocable server-side, so a disabled user's authorization lapses within one access-token window at most (see §5.1.1).
- Sensitive write operations (adjustments, production, purchases, role changes) re-check `users.status` and current role/branch access from the database, not just token claims.
- Device credentials are server-issued, not client-generated.
- Idempotency key (`sync_op_id`) required on every write-through-sync endpoint to prevent replay-driven duplication.
- Offline tokens are scoped (read + queue-only) and time-limited; full validation happens again at sync time regardless of what the offline token permitted locally.
- Rate limiting on `/api/sync` and `/api/session/*` endpoints.
- All audit log entries immutable (insert-only), same as the inventory ledger.
- Conflict resolution never edits historical rows — it appends a new ledger entry and marks the original flagged row `resolved`, preserving a full record of what happened.

---

## 13. Testing Strategy

**Backend:** unit tests per service (especially unit-conversion math and stock-sufficiency checks), repository/integration tests against a real Postgres test container, API contract tests, concurrency tests specifically for simultaneous writes to the same `(organization_id, location_id, ingredient_id)`.

**RLS suite (mandatory, see §6.1):** cross-tenant SELECT/UPDATE/DELETE/INSERT attempts against every tenant-scoped table, run in CI against a real Postgres container.

**Session/auth suite:** access token expiry enforcement, refresh token rotation and reuse-detection (stolen-token scenario), revocation propagation (disabled user rejected on next refresh; explicit session revoke blocks that device only; revoke-all blocks every device), sensitive-operation re-check behavior (role changed mid-session denies the stale-claim action).

**Frontend:** component tests, form validation tests, offline-mode tests (outbox queuing, sync status transitions).

**End-to-end:** login → select branch → view inventory → purchase → produce → stock decreases → adjust with reason → audit recorded.

**Synchronization-specific (critical path, needs dedicated suite):** offline queue → reconnect → idempotent replay of duplicate `sync_op_id` → simultaneous corrections from two devices → production that would go negative once server deltas applied → expired offline token attempting sync.

---

## 14. Resolution Status and Remaining Open Items

All eleven architectural decisions raised across this review (the original five, plus the six follow-ups) are now incorporated:

| # | Item | Status |
|---|---|---|
| 1 | Current-stock strategy | ✅ Resolved — ledger source of truth, cached projection |
| 2 | Tenant isolation ownership | ✅ Resolved — Spring Boot primary, RLS defense-in-depth |
| 3 | Authentication provider | ✅ Resolved — Supabase Auth (identity) + Spring Boot (authorization) |
| 4 | Offline MVP scope | ✅ Resolved — outbox + idempotent sync, conflict flagging only |
| 5 | Recipe versioning | ✅ Resolved — `recipe_versions`, production pins a version |
| 6 | Stock-level projection consistency | ✅ Resolved — synchronous, single-transaction, rollback-safe (§7.2) |
| 7 | Session/refresh/revocation design | ✅ Resolved — short-lived access + rotating refresh (§5.1.1) |
| 8 | RLS testing | ✅ Resolved — mandatory cross-tenant test suite, CI-enforced (§6.1) |
| 9 | Conflict review ownership | ✅ Resolved — Manager/Owner queue, append-only resolution (§7.5) |
| 10 | Ingredient deactivation | ✅ Resolved — soft flag, historical reads unaffected (§4 constraints) |
| 11 | Currency/locale | ✅ Resolved — PHP default, currency-agnostic schema (§4 constraints) |

### Remaining items (minor, non-blocking)

These are worth deciding early in implementation but don't block starting:

1. **Conflict-review SLA** — no enforced time limit on resolving flagged items yet. Recommend a simple dashboard indicator (e.g., "3 items pending review, oldest 2 days") rather than a hard SLA for MVP; revisit if it proves to be a real operational problem.
2. **Refresh token storage on the client** — the exact secure-storage mechanism on each target platform (web PWA vs. installed/mobile-wrapped context) needs a platform-specific decision during frontend implementation; the server-side design (§5.1.1) is unaffected by that choice.
3. **RLS policy maintenance process** — as new tenant-scoped tables are added post-MVP, confirm the team has a checklist/PR template step ("added RLS policy + test") so §6.1's discipline doesn't erode over time. Process, not architecture — worth a line in the contribution guide.

### Architecture Status: **READY FOR IMPLEMENTATION**

No remaining architectural blockers. The three items above are implementation-detail/process items to track, not open design questions that would change the schema, API boundaries, or core flows described in this document.
