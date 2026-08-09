# 01 — Spring Boot Backend Setup

## 1. Goal

Create the initial backend project for the Inventory Management System.

The backend will eventually provide the REST API and business logic
used by the frontend application.

At this stage, we are only creating the project foundation.
We are not implementing inventory features yet.

---

# 2. What Are We Building?

The project is an inventory management system intended to support
small food businesses such as bakeries, burger shops, and other
businesses that need to track ingredients and stock.

The system will eventually support features such as:

- Ingredients
- Stock tracking
- Stock adjustments
- Suppliers
- Recipes
- Users and roles
- Multiple branches
- Audit history
- Offline operation and synchronization
- Other features defined in the project architecture

The backend will be responsible for the application's business logic
and communication with the database.

The frontend will communicate with the backend through APIs.

Basic architecture:

Frontend
    ↓
HTTP / REST API
    ↓
Spring Boot Backend
    ↓
PostgreSQL Database

---

# 3. Why Are We Using Spring Boot?

## What?

Spring Boot is a Java framework used to build backend applications.

It provides tools and conventions that make it easier to create
web applications and REST APIs using Java.

## Why?

We need a backend that can:

- Receive requests from the frontend
- Process business logic
- Validate data
- Communicate with the database
- Handle authentication and authorization
- Return responses to the frontend

Spring Boot provides the foundation for these responsibilities.

## How?

The frontend will eventually send requests such as:

GET /api/ingredients

or:

POST /api/inventory/adjustments

Spring Boot will receive these requests, process them, and return
the appropriate response.

Example:

Frontend
    ↓
GET /api/ingredients
    ↓
Spring Boot
    ↓
Business Logic
    ↓
Database
    ↓
Response
    ↓
Frontend

---

# 4. Project Configuration

The initial Spring Boot project was generated with the following
configuration.

| Setting | Selected Value |
|---|---|
| Project | Maven |
| Language | Java |
| Spring Boot | 4.1.0 |
| Group | com.inventory.system |
| Artifact | inventory-system-backend |
| Package | com.inventory.system |
| Packaging | JAR |
| Configuration | Properties |
| Java | 17 |

---

# 5. Why Java 17?

## What?

Java 17 is the Java version used by this project.

## Why?

Java 17 is an LTS (Long-Term Support) version of Java.

We chose it because:

- It is stable
- It is widely supported
- It is suitable for Spring Boot development
- It provides a stable foundation for learning

We do not need to use the newest Java version simply because it
exists.

For this project, stability and learning are more important than
using the newest available version.

## How?

Maven and Spring Boot use the selected Java version when compiling
and running the application.

---

# 6. Why Maven?

## What?

Maven is a build and dependency management tool for Java projects.

The main Maven configuration file is:

pom.xml

## Why?

Our application will use many external libraries.

Instead of manually downloading and managing every library, Maven
allows us to declare the dependencies that the project needs.

For example:

Spring Web
Spring Data JPA
Spring Security
PostgreSQL Driver

Maven manages these dependencies and helps build the application.

## How?

The project contains:

pom.xml

The pom.xml file describes:

- Project information
- Spring Boot configuration
- Dependencies
- Build configuration

When Maven builds the project, it resolves the required dependencies.

---

# 7. Why JAR Packaging?

## What?

The project is packaged as a JAR (Java Archive).

## Why?

Spring Boot applications commonly run as executable JAR files.

The application can contain the code and required components needed
to run the backend.

This is appropriate for our Spring Boot backend.

## How?

After building the project, Maven can produce a JAR file that can
be executed by Java.

---

# 8. Why These Dependencies?

The initial project contains five main dependencies.

- Spring Web
- Spring Data JPA
- PostgreSQL Driver
- Spring Security
- Validation

Each dependency exists because it solves a specific problem.

---

## 8.1 Spring Web

### What?

Spring Web provides functionality for building web applications
and REST APIs.

### Why?

Our Next.js frontend needs a way to communicate with the backend.

The backend will expose API endpoints that the frontend can call.

### How?

Example:

Frontend
    ↓
HTTP Request
    ↓
Spring Web
    ↓
Controller
    ↓
Business Logic

Spring Web gives us the tools needed to create REST controllers
and handle HTTP requests and responses.

---

## 8.2 Spring Data JPA

### What?

Spring Data JPA provides a way to work with relational database
data using Java objects and repositories.

JPA stands for Java Persistence API.

Spring Data JPA commonly works with Hibernate as the JPA implementation.

### Why?

Our inventory system will contain many types of persistent data,
such as:

- Ingredients
- Suppliers
- Users
- Branches
- Recipes
- Inventory transactions

We need a way for our Java application to work with this data.

### How?

A Java class can represent a database entity.

Example concept:

Java Object
    ↓
JPA / Hibernate
    ↓
Database Table

Important:

JPA does not mean that SQL is no longer important.

We still need to understand SQL and PostgreSQL because the database
is an important part of the system.

---

## 8.3 PostgreSQL Driver

### What?

The PostgreSQL Driver allows the Java application to communicate
with a PostgreSQL database.

### Why?

Our application will use PostgreSQL as its relational database.

The Java application needs a database driver to establish
communication with PostgreSQL.

### How?

The basic communication path is:

Spring Boot
    ↓
Spring Data JPA / Hibernate
    ↓
PostgreSQL Driver
    ↓
PostgreSQL

The driver acts as the connection layer between the Java application
and PostgreSQL.

---

## 8.4 Spring Security

### What?

Spring Security provides security features for Spring applications.

It supports functionality related to authentication and
authorization.

### Why?

Our inventory system will have users with different roles.

Examples:

- Owner
- Manager
- Staff

Different users may have different permissions.

For example:

Owner
    → Full access

Manager
    → Management access

Staff
    → Operational access

The exact permissions will be defined later in the architecture
and implementation.

### How?

A request can pass through the security layer before reaching
the application's business logic.

Request
    ↓
Security
    ↓
Authentication / Authorization
    ↓
Business Logic
    ↓
Response

Important distinction:

Authentication:
    "Who is this user?"

Authorization:
    "What is this user allowed to do?"

---

## 8.5 Validation

### What?

Validation provides mechanisms for checking whether incoming data
meets defined rules.

### Why?

Inventory systems depend on accurate data.

For example, a request containing:

Quantity = -50

may not be valid for a normal stock operation.

Validation helps prevent invalid input from entering the application.

### How?

Validation rules can be placed on request data.

Conceptual example:

Name
    → Must not be blank

Quantity
    → Must be zero or greater

The backend should validate data even if the frontend already
performs validation.

The frontend improves user experience.

The backend protects the application's data and rules.

---

# 9. Why These Five Dependencies?

The dependencies were intentionally kept limited during the initial
project setup.

| Dependency | Main Responsibility |
|---|---|
| Spring Web | REST API and HTTP communication |
| Spring Data JPA | Database persistence |
| PostgreSQL Driver | PostgreSQL communication |
| Spring Security | Authentication and authorization |
| Validation | Input validation |

The principle is:

> Only add a dependency when it solves a real problem.

We are intentionally avoiding unnecessary dependencies at the
beginning of the project.

Additional dependencies can be introduced later when the system
actually requires them.

---

# 10. Package Naming

The project uses:

com.inventory.system

as its base Java package.

## Why?

The package represents the application's domain rather than
repeating the project artifact name.

The artifact is:

inventory-system-backend

while the package is:

com.inventory.system

These serve different purposes.

### Artifact

Identifies the generated project/build artifact.

### Package

Provides the namespace for Java source code.

The package therefore remains:

com.inventory.system

instead of:

com.inventory.system.inventory-system-backend

---

# 11. Initial Project Structure

After generating the project, Spring Boot creates a basic project
structure.

The generated structure will be documented in the next phase after
we inspect the project files.

Expected high-level structure:

src/
├── main/
│   ├── java/
│   └── resources/
│
└── test/

The exact purpose of these folders will be documented separately.

---

# 12. What We Have Done

At this stage we have:

- Created the Spring Boot project
- Selected Java 17
- Selected Maven
- Configured the project package
- Selected JAR packaging
- Added Spring Web
- Added Spring Data JPA
- Added PostgreSQL Driver
- Added Spring Security
- Added Validation

No inventory business logic has been implemented yet.

---

# 13. What I Learned

At the end of this phase, I should understand:

- What Spring Boot is
- What Maven does
- What a dependency is
- Why Spring Web is needed
- What Spring Data JPA does
- Why a PostgreSQL Driver is needed
- What Spring Security is responsible for
- The difference between authentication and authorization
- Why backend validation is necessary
- The difference between a project artifact and Java package
- Why we should avoid unnecessary dependencies

---

# 14. Problems / Decisions

## Package Name

### Initial package

com.inventory.system.inventory-system-backend

### Final package

com.inventory.system

### Reason

The original package unnecessarily repeated the artifact name and
contained a hyphenated project name.

The final package is cleaner and provides a better namespace for
the application's Java code.

---

# 15. Verification

Before moving to the next phase:

- [ ] Spring Boot project generated successfully
- [ ] Project opens successfully in the IDE
- [ ] Maven dependencies are downloaded
- [ ] Application starts successfully
- [ ] No startup errors
- [ ] Base package is com.inventory.system
- [ ] Java version is 17
- [ ] Spring Boot version is 4.1.0

---

# 16. Next Step

The next phase is to inspect and understand the generated
Spring Boot project structure.

Before writing application features, we will understand:

- pom.xml
- src/main/java
- src/main/resources
- application.properties
- src/test
- The main Spring Boot application class

The goal is to understand what Spring Boot generated for us before
we begin writing our own code.
