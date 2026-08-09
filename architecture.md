# Inventory Management System — Architecture & Technical Design

You are acting as a **senior software architect and technical lead** helping me design a production-quality, multi-tenant inventory management system.

Do **NOT write application code yet**.

Your job in this phase is to analyze the requirements, challenge weak assumptions, identify architectural risks, and produce a complete **Architecture Specification v1.0** that can later be used as the foundation for implementation.

---

# 1. Product Vision

I want to build a configurable inventory management system that can be used by different types of businesses.

Examples:

* Bakeries
* Burger/food businesses
* Restaurants
* Small food manufacturers
* Other businesses that need ingredient/material inventory

The system should solve the actual inventory problem rather than becoming overloaded with unnecessary features.

A bakery might have:

* Flour
* Sugar
* Yeast
* Butter
* Milk

A burger business might have:

* Burger buns
* Beef
* Mayonnaise
* Cheese
* Lettuce
* Tomatoes

The same application should support both businesses without hard-coded business-specific logic.

The system must be **configuration/data driven**.

---

# 2. Technology Direction

The planned technology stack is:

### Frontend

* Next.js
* Responsive/mobile-first UI
* PWA capabilities
* Android/iPhone/tablet/laptop support

### Backend

* Java
* Spring Boot
* REST API
* Business logic should primarily live in the backend

### Database / Cloud

* Supabase
* PostgreSQL

### Offline

* Browser local database/storage, potentially IndexedDB/Dexie or an equivalent appropriate solution
* Offline-first capabilities
* Synchronization with the cloud when connectivity returns

### Automation

* n8n

### Future integrations

* Node-RED for hardware/IoT/integration use cases

### Future AI

AI may later be added for:

* Inventory insights
* Purchasing suggestions
* Natural-language queries
* Reports/analysis

However, AI must NOT be responsible for critical inventory calculations or directly become the source of truth for inventory.

---

# 3. Multi-Tenant Architecture

The system must support multiple businesses using the same application.

For example:

Business A:

* Bakery
* 1 branch
* 5 users

Business B:

* Burger business
* 3 branches
* 10 users

The architecture must prevent data from one organization/business from being exposed to another.

A business should be able to configure its own:

* Branches
* Users
* Roles
* Ingredients
* Units
* Suppliers
* Recipes
* Inventory locations
* Purchases
* Production

Do not hard-code any business-specific ingredients or workflows.

---

# 4. Branches and Locations

Businesses can have multiple branches.

Example:

Minute Burger:

* Branch 1
* Branch 2
* Branch 3

Each branch may have inventory locations such as:

* Main storage
* Kitchen
* Freezer
* Refrigerator

Clearly distinguish:

**Supplier location**

from:

**Business branch**

from:

**Inventory storage location**

The architecture should allow a business to use one branch or many branches without requiring a different application.

---

# 5. Users, Roles and Authentication

Authentication is required.

The system should support:

* Owner
* Manager
* Staff

Some businesses may only use Owner + Staff.

Unused roles should be configurable/hidden from the UI rather than requiring every business to use every role.

Users must be associated with an organization/business.

Users may also have access restrictions by branch.

Example:

A staff member assigned to Branch 2 should not automatically be able to modify Branch 1 inventory unless authorized.

Design an appropriate role/permission model.

---

# 6. Device Tracking

The system must track devices.

This is important because a small business may have only ONE physical device.

For example:

Minute Burger Branch 1:

* One Android tablet

Different users may log into that same tablet:

Morning:

* Maria

Afternoon:

* John

Evening:

* Pedro

Therefore:

**Device identity must NOT be used as the identity of the person performing an action.**

Important inventory actions must record at minimum:

* Organization
* Branch
* User
* Device
* Timestamp
* Transaction/action ID

---

# 7. Audit Trail and Corrections

This is a major requirement.

Inventory changes must be traceable.

The system should NOT silently overwrite important inventory history.

For example:

System says:

5 kg flour

User realizes it should have been:

4 kg flour

The system should allow a stock adjustment, but the user must provide a reason.

Example:

Previous:
5 kg

New:
4 kg

Reason:
"Incorrect quantity entered"

The system records:

* User
* Branch
* Device
* Date/time
* Item
* Previous quantity
* New quantity
* Difference
* Reason

No approval is required for normal corrections.

However, the reason is mandatory.

For important transactions, prefer **void/cancellation** over physical deletion.

Never destroy important historical inventory transactions simply to make the current state look correct.

Explain which operations require audit records.

---

# 8. Inventory Transactions

Do NOT design the system around simply storing and overwriting:

current_stock = 25

Instead, design an inventory movement/transaction model.

Examples:

+25 kg — Purchase
-5 kg — Production
-1 kg — Manual adjustment
+2 kg — Stock correction

The current quantity can be derived from valid inventory movements or maintained as a controlled projection/cache.

The architecture should explain which approach is safest and most performant.

Inventory transaction types should be extensible.

---

# 9. Units and Measurements

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
* Other configurable units

Do NOT hard-code units into the application.

Businesses should be able to define appropriate units where necessary.

---

# 10. Purchase Unit vs Consumption Unit

This is very important.

An ingredient may be purchased using one unit but consumed using another.

Example:

Mayonnaise:

Purchase:
1 tub = 5 kg

Recipe:

20 g mayonnaise per burger

If the business buys:

2 tubs

The system should understand:

2 tubs = 10 kg = 10,000 g

Then if 100 burgers are produced:

100 × 20 g = 2,000 g consumed

Remaining:

8,000 g

The UI may display quantities using a human-friendly unit.

Design a robust unit conversion model.

Explain how to handle units that cannot safely be converted, such as:

* Box
* Pack
* Piece

unless the business explicitly defines the relationship.

---

# 11. Suppliers

The system should track suppliers.

Supplier information may include:

* Supplier name
* Contact information
* Location
* Pickup/delivery information
* Items supplied
* Supplier-specific price
* Purchase history
* Notes
* Active/inactive status

The architecture should support one ingredient being supplied by multiple suppliers.

Example:

Flour:

Supplier A:
₱1,200 / 25 kg

Supplier B:
₱1,250 / 25 kg

The system should be able to maintain supplier-specific purchasing information without assuming that an ingredient has only one supplier.

---

# 12. Purchases

The system should support actual purchase records.

Example:

Purchase:

Supplier:
ABC Supplier

Date:
August 9, 2026

Items:

3 bags flour
₱1,200 each

2 tubs mayonnaise
₱450 each

Purchases should create appropriate inventory transactions.

Explain how purchase price/history should be modeled.

---

# 13. Recipes

The system must support configurable recipes.

Example:

Classic Burger:

1 bun
1 beef patty
20 g mayonnaise
1 slice cheese
30 g lettuce

A bakery could instead define:

Bread:

500 g flour
50 g sugar
5 g yeast
20 g butter
250 ml water

Recipes should not be hard-coded.

Businesses must be able to:

* Create recipe
* Edit recipe
* Add ingredients
* Remove ingredients
* Define quantity
* Define consumption unit
* Activate/deactivate recipe

---

# 14. Production

Recipes should connect to production.

Example:

Produce:

50 burgers

The system calculates the required ingredients and creates inventory movements.

For example:

50 buns
50 beef patties
1,000 g mayonnaise
50 cheese slices
1,500 g lettuce

The backend should validate inventory availability before completing production.

Explain how production should behave when inventory is insufficient.

---

# 15. Inventory Locations

Inventory should be location-aware.

Example:

Branch 1:

* Main Storage
* Kitchen
* Freezer

The system should know where stock exists.

Consider whether stock transfers between locations/branches should be part of the MVP or a later phase, but design the architecture so that it can be supported later without major restructuring.

---

# 16. Search, Filtering and Sorting

The inventory UI should support:

* Live search while typing
* Column filtering
* Sorting
* Supplier filtering
* Branch filtering
* Location filtering
* Low-stock filtering
* Unit filtering
* Status filtering

It must work well on mobile devices.

Desktop can use a traditional table.

Mobile may use cards/list views rather than forcing a large table onto a phone.

---

# 17. Offline-First Requirement

This is a major requirement.

The system should still be usable when the internet connection is temporarily unavailable.

Example:

A tablet is being used in a bakery.

Internet goes offline.

The user can still:

* View relevant inventory
* Perform allowed inventory operations
* Record production
* Make stock adjustments
* Possibly record purchases depending on the design

The actions are stored locally.

When the internet returns, they synchronize with the server.

The user should be able to see synchronization state.

Example:

🟡 Pending synchronization

then:

🟢 Synchronized

---

# 18. Offline Synchronization

Design a robust synchronization strategy.

Consider:

* Local database
* Pending operation queue
* Transaction IDs
* Device IDs
* User IDs
* Organization IDs
* Timestamps
* Idempotency
* Retry behavior
* Duplicate prevention
* Network failure
* Partial synchronization
* Conflict detection
* Conflict resolution
* Server validation
* Authentication while offline
* What happens if the same inventory item is changed on two devices while offline?

Example:

Device A:

25 kg → consumes 5 kg

Device B:

25 kg → receives 10 kg

Both operate offline.

When both reconnect, the architecture must safely reconcile the operations.

Do NOT simply overwrite one device's stock with the other.

Explain the recommended synchronization model.

---

# 19. Data Integrity

Inventory is business-critical.

The architecture should prioritize:

* Transaction integrity
* Auditability
* Idempotency
* Concurrency safety
* Validation
* Authorization
* Data isolation between tenants
* Reliable synchronization

Do not rely on frontend validation alone.

Critical business rules must be enforced server-side.

---

# 20. Backend Architecture

Design a maintainable Spring Boot architecture.

Prefer a domain/feature-oriented structure rather than putting every controller/service/repository into giant global folders.

Potential domains include:

* Authentication
* Organization
* Branch
* User
* Device
* Ingredient
* Unit
* Supplier
* Purchase
* Recipe
* Production
* Inventory
* Audit

Do not blindly accept this list. Modify it if a better domain model exists.

Explain:

* Package structure
* Domain boundaries
* DTOs
* Services
* Repositories
* Controllers
* Validation
* Exception handling
* Transactions
* Security
* Testing strategy

---

# 21. Frontend Architecture

Design a scalable Next.js architecture.

It must support:

* Mobile-first UI
* PWA
* Offline state
* Local data
* Synchronization state
* Authentication
* Permissions
* Branch selection
* Inventory
* Recipes
* Suppliers
* Purchases
* Production
* Audit history

Prefer feature/domain-oriented organization.

Explain:

* Routing
* Components
* Hooks
* State management
* API layer
* Local database layer
* Sync layer
* Authentication handling
* Form validation
* Error handling
* Loading states
* Offline states

---

# 22. Supabase

Explain which Supabase capabilities should be used.

Consider:

* PostgreSQL
* Authentication
* Row Level Security
* Storage if necessary
* Realtime if useful
* Database functions if appropriate

The architecture must clearly define what responsibility belongs to:

**Next.js**

vs

**Spring Boot**

vs

**Supabase**

Do not allow business-critical inventory logic to become scattered across all three.

---

# 23. n8n

n8n should be an automation/integration layer rather than the core inventory engine.

Potential future workflows:

* Low-stock notifications
* Supplier reminders
* Scheduled inventory reports
* Daily/weekly summaries
* Notifications when stock reaches thresholds
* Purchase reminders

Explain where n8n should connect to the system and how it should avoid becoming a source of truth.

---

# 24. Node-RED

Node-RED is optional/future.

Possible future use cases:

* Digital weighing scales
* Barcode scanners
* Sensors
* IoT devices
* Hardware integrations

Do not add Node-RED into the core architecture unless there is a concrete reason.

Design an integration boundary that allows it later.

---

# 25. AI

AI is NOT part of the core inventory calculation engine.

Potential future AI features:

* "What ingredients are running low?"
* "What should I purchase this week?"
* Inventory summaries
* Natural-language reports
* Anomaly explanations
* Business insights

AI may recommend or explain things, but deterministic backend logic remains responsible for actual inventory calculations and database changes.

---

# 26. Architecture Deliverables

Before any implementation, produce the following:

## A. Executive architecture overview

Explain the complete system in understandable language.

## B. Architecture diagram

Show:

User
→ Next.js PWA
→ Local database
→ Spring Boot
→ Supabase/PostgreSQL

Also show:

n8n

and future Node-RED/AI integrations.

## C. Multi-tenant architecture

Explain how organizations, branches, users, roles and devices relate.

## D. Database ERD

Provide a detailed ERD.

Include:

* Tables
* Primary keys
* Foreign keys
* Relationships
* Important indexes
* Important constraints

Do not create unnecessary tables.

## E. Domain model

Explain the major business concepts and how they interact.

## F. Offline synchronization architecture

This is extremely important.

Provide a sequence diagram for:

1. Online transaction
2. Offline transaction
3. Reconnection
4. Synchronization
5. Conflict
6. Retry
7. Duplicate prevention

## G. Authentication and authorization model

Explain:

* Login
* Sessions/tokens
* Roles
* Permissions
* Branch access
* Offline authentication considerations

## H. Audit architecture

Show how the system records:

* Who
* What
* When
* Where
* Device
* Previous value
* New value
* Reason

## I. API design

Provide proposed REST endpoints.

For each major endpoint explain:

* HTTP method
* URL
* Purpose
* Authentication
* Authorization
* Request
* Response
* Important validation

## J. Folder structure

Provide recommended production-quality folder structures for:

* Next.js
* Spring Boot

Explain why the structure is maintainable.

## K. Security architecture

Consider:

* Authentication
* Authorization
* Tenant isolation
* Row Level Security
* API security
* Input validation
* Rate limiting
* Audit logging
* Secrets
* Device identification
* Offline security

## L. Testing architecture

Recommend:

* Unit tests
* Integration tests
* API tests
* Database tests
* Frontend tests
* E2E tests
* Offline/synchronization tests
* Concurrency tests

## M. Deployment architecture

Recommend a practical deployment approach for:

* Next.js
* Spring Boot
* Supabase
* n8n

Keep cost and simplicity in mind.

## N. MVP scope

Clearly divide:

### MVP

What must exist for the first usable version.

### Phase 2

Useful but not required initially.

### Future

Advanced automation, AI, IoT, etc.

Avoid feature creep.

---

# 27. Critical Review

Before finalizing your proposal, challenge the requirements.

Identify:

1. Architectural risks
2. Potential overengineering
3. Missing requirements
4. Database risks
5. Offline-sync risks
6. Security risks
7. Multi-tenant risks
8. Scalability concerns
9. UX problems
10. Places where the chosen technology stack may be unnecessarily complicated

If you believe any of my technology choices are inappropriate, **say so clearly and explain why**.

Do not agree with me simply because I suggested something.

---

# 28. Important Constraints

Do NOT:

* Generate application code yet
* Generate hundreds of files
* Assume one business only
* Hard-code bakery/burger logic
* Make AI responsible for inventory calculations
* Make n8n the core business logic
* Make Node-RED a required dependency
* Physically delete important transaction history
* Allow frontend-only enforcement of critical business rules
* Treat the current stock number as the only source of truth
* Ignore offline synchronization
* Ignore auditability

---

# 29. Final Output

At the end, provide:

### Architecture Status

Choose one:

* READY FOR IMPLEMENTATION
* NEEDS REQUIREMENT CLARIFICATION
* NEEDS ARCHITECTURAL CHANGES

If clarification is required, list only the questions that genuinely block implementation.

Otherwise, produce the complete Architecture Specification v1.0.

Do not start coding.

The goal of this phase is to create an architecture that is:

* Maintainable
* Secure
* Multi-tenant
* Offline-capable
* Synchronizable
* Mobile-first
* Extensible
* Testable
* Practical
* Not unnecessarily overengineered

I want this to be a real-world system that could eventually be used by businesses, not just a tutorial/demo project.
