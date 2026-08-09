# Inventory Management System

## Architecture Specification v1.0

**Status:** Proposed — Pending Architecture Review
**Project Type:** Multi-tenant Inventory Management System
**Primary Goal:** Provide a reliable, configurable, mobile-first inventory system for small and medium businesses.

---

# 1. Vision

The system is a configurable inventory management platform designed for businesses such as:

* Bakeries
* Burger businesses
* Restaurants
* Food production businesses
* Other businesses that manage ingredients, materials, recipes, and stock

The system must solve real inventory problems without unnecessary features or business-specific hard-coding.

The same application must be able to support different businesses through configuration and data.

Example:

### Bakery

* Flour
* Sugar
* Yeast
* Butter
* Milk

### Burger Business

* Burger buns
* Beef patties
* Mayonnaise
* Cheese
* Lettuce

The application must not contain business-specific logic such as:

```text
if business == "Bakery"
```

Instead, businesses configure their own ingredients, recipes, suppliers, units, branches, and inventory.

---

# 2. Core Architectural Principles

The system follows these principles:

1. **Multi-tenant by design**
2. **Mobile-first**
3. **Offline-capable**
4. **Cloud-backed**
5. **Auditability**
6. **Transaction-based inventory**
7. **Server-side business rules**
8. **Configuration over hard-coding**
9. **Modular and feature-oriented architecture**
10. **Extensible without unnecessary complexity**

Inventory data is business-critical and must prioritize correctness over convenience.

---

# 3. High-Level Architecture

```text
                         USERS
                           │
             ┌─────────────┴─────────────┐
             │                           │
          Mobile                       Desktop
             │                           │
             └─────────────┬─────────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │   Next.js PWA   │
                  │    Frontend     │
                  └────────┬────────┘
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
      Local Device DB              Spring Boot API
       (Offline Data)                    │
             │                           │
             │                           ▼
             │                    Business Logic
             │                           │
             │                           ▼
             │                     PostgreSQL
             │                      / Supabase
             │                           │
             └──────────── Sync ─────────┘
                                         │
                                         ▼
                                       n8n
                                   Automations

                         Future Integrations
                                │
                         ┌──────┴──────┐
                         │             │
                      Node-RED        AI
```

---

# 4. Technology Stack

## Frontend

* Next.js
* React
* TypeScript
* Responsive UI
* Progressive Web App (PWA)
* Local browser database for offline capability

## Backend

* Java
* Spring Boot
* REST API
* Server-side business rules
* Validation
* Authorization
* Transaction management

## Database / Cloud

* Supabase
* PostgreSQL

Supabase provides infrastructure around the PostgreSQL database and may also provide authentication, storage, realtime capabilities, and other supporting services where appropriate.

## Automation

* n8n

## Future Integration

* Node-RED

## Future AI

* AI services for insights and natural-language interaction

AI must not become the source of truth for inventory.

---

# 5. Multi-Tenant Architecture

The system supports multiple independent businesses.

The fundamental hierarchy is:

```text
Organization
│
├── Users
│
├── Roles
│
├── Devices
│
├── Branches
│
├── Suppliers
│
├── Ingredients
│
├── Recipes
│
└── Inventory
```

Each organization must be isolated from every other organization.

A user belonging to Organization A must never be able to access Organization B's data.

Tenant isolation must be enforced server-side and at the database/security layer where appropriate.

---

# 6. Branch Architecture

An organization may have one or multiple branches.

Example:

```text
Minute Burger
│
├── Branch 1
│   ├── Main Storage
│   └── Kitchen
│
├── Branch 2
│   ├── Main Storage
│   └── Kitchen
│
└── Branch 3
    ├── Main Storage
    └── Kitchen
```

The system must support both:

```text
Business → One Branch
```

and:

```text
Business → Multiple Branches
```

without requiring separate applications.

---

# 7. Inventory Locations

A branch may contain multiple inventory locations.

Examples:

* Main Storage
* Kitchen
* Freezer
* Refrigerator
* Warehouse

The system must distinguish:

```text
Supplier Location
```

from:

```text
Business Branch
```

and:

```text
Inventory Location
```

Future inventory transfers between branches and locations should be possible without requiring a major architectural redesign.

---

# 8. Users and Roles

The system supports:

* Owner
* Manager
* Staff

Not every organization must use every role.

For example:

```text
Small Business
├── Owner
└── Staff
```

Another organization may use:

```text
Larger Business
├── Owner
├── Manager
└── Staff
```

Unused roles may be hidden from the organization's UI.

Roles should control permissions.

Permissions may also be restricted by branch.

Example:

```text
Staff A
→ Branch 1

Staff B
→ Branch 2

Manager
→ Branch 1 + Branch 2 + Branch 3

Owner
→ Entire organization
```

---

# 9. Authentication

Authentication is required.

The system must identify the authenticated user for important actions.

A device does not represent a user.

Example:

```text
Tablet #01

Morning:
Maria logs in

Afternoon:
John logs in

Evening:
Pedro logs in
```

Actions must be attributed to:

* User
* Organization
* Branch
* Device
* Timestamp

Authentication must remain secure even when offline capabilities are introduced.

Offline authentication behavior must be explicitly designed and tested.

---

# 10. Device Tracking

Devices should have a unique device identity.

Example:

```text
Device:
TABLET-01

Organization:
Minute Burger

Branch:
Branch 1
```

A device may be shared by multiple users.

Therefore:

```text
Device ID ≠ User ID
```

Both must be recorded when appropriate.

---

# 11. Inventory Model

Inventory must be transaction-based.

The system must not rely solely on overwriting a single current quantity.

Example:

```text
Initial stock       +25 kg
Purchase            +10 kg
Production           -5 kg
Manual adjustment    -1 kg
--------------------------------
Current stock        29 kg
```

Inventory movements should be represented as transactions.

Possible transaction types include:

* Opening balance
* Purchase
* Production consumption
* Stock adjustment
* Stock correction
* Transfer
* Return
* Waste
* Other controlled inventory events

Transaction types should be extensible.

---

# 12. Inventory Adjustments

Manual corrections are allowed.

However, important corrections require a reason.

Example:

```text
System:
Flour = 5 kg

Actual:
Flour = 4 kg
```

The user selects:

```text
Adjust Stock
```

and provides:

```text
New quantity:
4 kg

Reason:
Incorrect quantity entered
```

The system records:

```text
User
Organization
Branch
Device
Ingredient
Previous quantity
New quantity
Difference
Reason
Timestamp
```

No approval is required for normal manual corrections.

The reason is mandatory.

---

# 13. Audit Trail

Important business actions must be auditable.

The system should answer:

> Who changed it?

> What changed?

> When did it change?

> Where did it happen?

> On which device?

> Why was it changed?

Audit information should include where applicable:

```text
user_id
organization_id
branch_id
device_id
action
entity
entity_id
previous_value
new_value
reason
timestamp
```

Important historical transactions should not be physically deleted.

Instead, use controlled cancellation/voiding where appropriate.

---

# 14. Units

The system must support flexible units.

Examples:

* Piece
* Gram
* Kilogram
* Milliliter
* Liter
* Box
* Bag
* Tub
* Bottle
* Pack
* Tray

Units must not be hard-coded into business logic.

Businesses may configure appropriate units.

---

# 15. Unit Conversion

Purchase units and consumption units may be different.

Example:

```text
Mayonnaise

1 Tub = 5 kg
```

Recipe:

```text
20 g mayonnaise / burger
```

Inventory internally needs a reliable representation that allows:

```text
2 tubs
=
10 kg
=
10,000 g
```

Producing:

```text
100 burgers
```

consumes:

```text
2,000 g
```

Remaining:

```text
8,000 g
```

The system must support explicit conversion relationships.

Non-convertible units such as:

```text
Box
Pack
Piece
```

must require an explicit business-defined relationship before conversion.

---

# 16. Ingredients

An ingredient represents an inventory-controlled item.

Example:

```text
Ingredient:
Mayonnaise

Base measurement:
Gram

Purchase units:
Tub

Suppliers:
Supplier A
Supplier B
```

Ingredients should support:

* Name
* Description
* Category
* Unit configuration
* Minimum stock level
* Maximum stock level if needed
* Active/inactive state
* Supplier relationships
* Branch/location availability

---

# 17. Suppliers

Suppliers belong to an organization.

A supplier may provide multiple ingredients.

An ingredient may have multiple suppliers.

Supplier information may include:

* Name
* Contact information
* Address/location
* Pickup/delivery information
* Notes
* Active/inactive status

Supplier-item relationships may include:

* Supplier
* Ingredient
* Purchase unit
* Unit conversion
* Price
* Effective date
* Supplier SKU/reference

Supplier price history should be preserved where appropriate.

---

# 18. Purchases

The system should support purchase records.

Example:

```text
Supplier:
ABC Supplier

Date:
August 9, 2026

Items:

3 bags Flour
2 tubs Mayonnaise
```

Purchases should create inventory transactions.

Purchase records should preserve:

* Supplier
* Branch/location
* User
* Device
* Date/time
* Items
* Quantities
* Units
* Prices
* Total
* Reference number if applicable

---

# 19. Recipes

Recipes are configurable business data.

Example:

```text
Classic Burger

1 × Bun
1 × Beef Patty
20 g Mayonnaise
1 × Cheese Slice
30 g Lettuce
```

A bakery may instead define:

```text
Bread

500 g Flour
50 g Sugar
5 g Yeast
20 g Butter
250 ml Water
```

Recipes must support:

* Create
* Edit
* Activate/deactivate
* Ingredients
* Quantities
* Consumption units
* Recipe versions/history where necessary

Recipes must not be hard-coded.

---

# 20. Production

Production connects recipes to inventory.

Example:

```text
Produce:
50 Classic Burgers
```

The system calculates required ingredients.

Example:

```text
50 Buns
50 Beef Patties
1,000 g Mayonnaise
50 Cheese Slices
1,500 g Lettuce
```

The backend validates inventory availability and creates appropriate inventory transactions.

Production must be atomic where appropriate.

A partially completed production transaction should not leave inventory in an inconsistent state.

---

# 21. Inventory Search and Filtering

Inventory must support:

* Live search while typing
* Sorting
* Filtering
* Supplier filtering
* Branch filtering
* Location filtering
* Low-stock filtering
* Unit filtering
* Status filtering

Desktop may use tables.

Mobile should use a responsive list/card representation where appropriate.

---

# 22. Mobile-First Design

The system must prioritize:

* Android phones
* iPhones
* Tablets

Desktop/laptop support is also required.

The system should not simply shrink a desktop interface onto a phone.

Mobile workflows should prioritize:

* Fast searching
* Quick stock checks
* Quick adjustments
* Production entry
* Purchase entry
* Clear status indicators
* Minimal unnecessary navigation

---

# 23. Progressive Web App

The frontend should be designed as a PWA.

Users should eventually be able to install the application on supported devices.

Example:

```text
Phone
 ↓
Open Web Application
 ↓
Install/Add to Home Screen
 ↓
Inventory Application
```

The PWA should support offline operation where technically appropriate.

---

# 24. Offline Architecture

The system must continue operating during temporary internet outages.

The device should maintain local application data required for offline workflows.

Conceptually:

```text
Next.js PWA
│
├── Online
│    └── Spring Boot API
│
└── Offline
     └── Local Database
          └── Pending Operations
```

Offline operations may include:

* Viewing cached inventory
* Stock adjustments
* Production
* Other approved inventory operations

The exact offline feature set must be defined during implementation.

---

# 25. Synchronization

Offline operations must be queued.

Example:

```text
Pending Operations

001  STOCK_ADJUSTMENT  Pending
002  PRODUCTION        Pending
003  PURCHASE          Pending
```

When connectivity returns:

```text
Local Queue
    ↓
Synchronization
    ↓
Spring Boot
    ↓
Validation
    ↓
PostgreSQL
    ↓
Synchronization Result
```

Each operation should have a globally unique transaction/operation identifier.

Synchronization must support:

* Idempotency
* Retry
* Duplicate prevention
* Partial failure handling
* Server validation
* Conflict detection
* Conflict resolution

The system must not simply overwrite server inventory with a stale local quantity.

Inventory changes should synchronize as business events/transactions where possible.

---

# 26. Offline Conflict Example

Example:

Initial stock:

```text
25 kg
```

Device A offline:

```text
Consumes 5 kg
```

Device B offline:

```text
Receives 10 kg
```

When both reconnect, the system should process the operations safely:

```text
25
-5
+10
----
30 kg
```

It must not perform:

```text
Device A current = 20
Device B current = 35

Last device wins
```

because this could destroy legitimate inventory activity.

Conflict handling rules must be explicitly documented and tested.

---

# 27. Backend Responsibilities

Spring Boot is responsible for:

* Business rules
* Inventory calculations
* Unit conversion validation
* Production calculations
* Inventory transaction creation
* Authorization
* Validation
* Concurrency handling
* Audit creation
* Synchronization validation
* API contracts

Critical business rules must not depend solely on frontend validation.

---

# 28. Frontend Responsibilities

Next.js is responsible for:

* User interface
* Responsive design
* PWA behavior
* User interactions
* Local offline storage
* Sync status
* API communication
* Client-side validation
* Presentation

The frontend should not be the final authority for business-critical calculations.

---

# 29. Supabase Responsibilities

Supabase/PostgreSQL is responsible for persistent cloud data storage and supporting infrastructure.

Potential capabilities:

* PostgreSQL
* Authentication where appropriate
* Row Level Security
* Storage if required
* Realtime capabilities where useful

The final architecture must clearly define the boundary between:

```text
Next.js
Spring Boot
Supabase
```

Business-critical inventory operations should remain controlled by the backend/database architecture rather than being freely writable from the frontend.

---

# 30. n8n Responsibilities

n8n is an automation/integration layer.

Potential workflows:

```text
Low stock
   ↓
n8n
   ↓
Notification
```

```text
Scheduled report
   ↓
n8n
   ↓
Owner notification
```

```text
Supplier reminder
   ↓
n8n
```

n8n must not become the source of truth for inventory.

---

# 31. Node-RED Responsibilities

Node-RED is optional and future-facing.

Potential integrations:

* Digital weighing scales
* Sensors
* Barcode scanners
* IoT devices
* Hardware systems

Node-RED should connect through defined integration/API boundaries.

It is not required for the core application.

---

# 32. AI Responsibilities

AI is optional and future-facing.

Potential capabilities:

* Inventory summaries
* Natural-language inventory questions
* Purchasing suggestions
* Trend explanations
* Anomaly explanations

AI must not directly determine authoritative inventory quantities.

For example:

```text
AI:
"You may need more flour next week."
```

is acceptable.

But:

```text
AI:
"Set flour stock to 50 kg."
```

must not bypass deterministic business rules and authorization.

---

# 33. Suggested Domain Boundaries

The backend should be organized around business capabilities rather than only technical layers.

Potential domains:

```text
authentication
organization
user
role
branch
device
ingredient
unit
supplier
purchase
recipe
production
inventory
audit
synchronization
```

This list may be refined during detailed design.

---

# 34. Backend Folder Structure

The backend should use a feature/domain-oriented structure.

Conceptually:

```text
backend/
└── src/
    └── main/
        └── java/
            └── com/
                └── application/
                    │
                    ├── authentication/
                    ├── organization/
                    ├── user/
                    ├── role/
                    ├── branch/
                    ├── device/
                    ├── ingredient/
                    ├── unit/
                    ├── supplier/
                    ├── purchase/
                    ├── recipe/
                    ├── production/
                    ├── inventory/
                    ├── audit/
                    ├── synchronization/
                    │
                    └── common/
```

Each domain may contain appropriate:

```text
controller
service
repository
domain/model
dto
mapper
validation
```

The exact structure should be refined during implementation.

---

# 35. Frontend Folder Structure

The frontend should also use feature-oriented organization.

Conceptually:

```text
frontend/
├── app/
├── features/
│   ├── authentication/
│   ├── inventory/
│   ├── ingredients/
│   ├── suppliers/
│   ├── purchases/
│   ├── recipes/
│   ├── production/
│   ├── branches/
│   └── users/
│
├── components/
├── lib/
│   ├── api/
│   ├── database/
│   ├── synchronization/
│   └── validation/
│
├── hooks/
├── types/
└── public/
```

The final Next.js structure should follow the current framework conventions chosen during implementation.

---

# 36. API Boundary

The system should expose business-oriented REST APIs.

Potential areas:

```text
/api/auth
/api/organizations
/api/users
/api/branches
/api/devices
/api/ingredients
/api/units
/api/suppliers
/api/purchases
/api/recipes
/api/production
/api/inventory
/api/audit
/api/synchronization
```

Exact endpoints should be defined during API design.

---

# 37. Security Requirements

Security must include:

* Authentication
* Authorization
* Tenant isolation
* Branch access control
* Server-side validation
* Database constraints
* Secure secrets
* API protection
* Audit logging
* Secure device identification
* Safe offline authentication
* Protection against duplicate/replayed synchronization requests

No secret keys should be committed to Git.

Environment variables must be used for credentials and secrets.

---

# 38. Testing Requirements

The system should eventually include:

### Backend

* Unit tests
* Service tests
* Repository/integration tests
* API tests
* Transaction tests
* Concurrency tests

### Frontend

* Component tests
* Form validation tests
* Integration tests
* Offline behavior tests

### End-to-End

Test important workflows:

```text
Login
→ Select branch
→ View inventory
→ Purchase stock
→ Produce product
→ Stock decreases
→ Adjust stock
→ Enter reason
→ Audit recorded
```

### Synchronization

Test:

* Offline operation
* Reconnection
* Retry
* Duplicate submission
* Concurrent changes
* Conflict handling
* Partial synchronization

---

# 39. MVP

The first usable version should focus on the core inventory problem.

## MVP

### Authentication

* Login
* Users
* Roles
* Branch access

### Organization

* Business
* Branches
* Devices

### Inventory

* Ingredients
* Units
* Unit conversion
* Stock
* Inventory transactions
* Manual adjustments
* Audit trail
* Search
* Sorting
* Filtering

### Suppliers

* Supplier management
* Supplier items
* Supplier pricing
* Purchase records

### Recipes

* Create recipes
* Add ingredients
* Define quantities
* Production
* Automatic inventory consumption

### PWA

* Responsive UI
* Basic offline foundation

The exact offline transaction scope should be finalized during implementation.

---

# 40. Phase 2

Potential features:

* Full offline synchronization
* Advanced inventory transfers
* Advanced reports
* Purchase recommendations
* Stock forecasting
* More detailed supplier management
* Advanced permissions
* Notifications
* n8n automation

---

# 41. Future

Potential future capabilities:

* AI inventory assistant
* Demand forecasting
* Natural-language reports
* Automated purchasing suggestions
* Barcode scanning
* Digital scales
* IoT integrations
* Node-RED integrations
* Advanced analytics

These must not compromise the reliability of the core inventory system.

---

# 42. Important Architectural Rule

The system should follow this principle:

```text
USER
 ↓
Next.js PWA
 ↓
Spring Boot
 ↓
Business Rules
 ↓
Inventory Transactions
 ↓
PostgreSQL / Supabase
```

While:

```text
n8n
```

handles automation,

```text
Node-RED
```

handles future hardware/integration requirements,

and:

```text
AI
```

provides optional intelligence and recommendations.

The core inventory system must remain deterministic and auditable.

---

# 43. Architecture Review Checklist

Before implementation begins, verify:

* [ ] Multi-tenant isolation is defined
* [ ] Branch access is defined
* [ ] Roles and permissions are defined
* [ ] Device tracking is defined
* [ ] Inventory transaction model is defined
* [ ] Unit conversion is defined
* [ ] Supplier model is defined
* [ ] Purchase model is defined
* [ ] Recipe model is defined
* [ ] Production model is defined
* [ ] Audit model is defined
* [ ] Offline model is defined
* [ ] Synchronization strategy is defined
* [ ] Conflict resolution is defined
* [ ] Authentication is defined
* [ ] Authorization is defined
* [ ] API boundaries are defined
* [ ] Database indexes/constraints are defined
* [ ] Security model is defined
* [ ] Testing strategy is defined
* [ ] MVP scope is defined

---

# 44. Architecture Status

**Status: Proposed — Pending Technical Review**

This document represents the agreed product and architectural direction.

Before implementation, the architecture should be reviewed for:

* Database normalization
* Multi-tenant security
* Offline synchronization correctness
* Concurrency
* Authentication
* Authorization
* Supabase/Spring Boot responsibility boundaries
* Scalability
* Complexity
* Maintainability

Once these areas have been reviewed and approved, this document becomes:

**Architecture Specification v1.0 — Approved**

Only after approval should implementation begin.
