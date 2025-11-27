---
marp: true
theme: default
paginate: true
backgroundColor: #fff
style: |
  section {
    font-family: 'Arial', sans-serif;
  }
  h1 {
    color: #2c3e50;
  }
  h2 {
    color: #34495e;
  }
  code {
    background-color: #f0f0f0;
    padding: 2px 5px;
    border-radius: 4px;
  }
---

# Project Architecture Overview
## Ecommerce Backend (Go)

---

# Introduction

- **Project**: Ecommerce Backend Server
- **Language**: Golang 1.22+
- **Core Framework**: Fiber (HTTP), Gorm (ORM)
- **Goal**: A scalable, modular, and highly configurable platform for ecommerce.

---

# High-Level Architecture

- **Platform-Based**: The application runs as a "Platform" that hosts specific "Systems".
- **Service-Oriented**: Systems are composed of loosely coupled services.
- **Hybrid Design**: Modular monolith core with integration for external gRPC microservices.

![bg right:40% fit](https://upload.wikimedia.org/wikipedia/commons/thumb/2/23/Golang.png/480px-Golang.png)

---

# The Core Philosophy: Flexibility

The system is designed to be **configuration-driven**. 

- **Dynamic Composition**: Services are loaded and wired together based on YAML configuration.
- **No Hard Dependencies**: Implementations can be swapped without changing core logic.
- **Versioning**: Run different versions of services (v1, v2) side-by-side.

---

# Configuration Example

The entire system behavior is defined in `.yaml` files:

```yaml
id: "f59b5250-b8f5-46fc-94cd-7f7347714665"
name: "ecommerce_v2"
services:
  - name: "db"
    package: "db_gorm"
    config:
      DB_HOST: "localhost"
      DB_PORT: "5432"
  - name: "automigration"
    package: "automigration/v2"
    dependencies:
      db: "db"
```

---

# Key Architectural Components

1.  **Platform Service**: The kernel that reads config and initializes the system.
2.  **Ecommerce API**: REST API layer for client interactions.
3.  **Ecommerce Router**: Handles HTTP routing and middleware.
4.  **DB Services**: Manages data persistence and schema.

---

# Interface-Based Design

Services expose `interface.go` to define contracts. Implementations are hidden.

**Benefits:**
- Easy to mock for testing.
- Swap implementations (e.g., change Payment Gateway from Braintree to Stripe) just by changing the config.

---

# External Integrations (gRPC)

The core backend integrates with specialized microservices via gRPC:

- **Communications Service**: Handles SMS and Email (Twilio, CustomerIO).
- **Payments Service**: Secure payment processing (Braintree).
- **Week Service**: Time/Date management utilities.

---

# Developer Experience

- **Standardized API Responses**:
  ```json
  {
      "success": true,
      "data": { ... }
  }
  ```
- **Tooling**: `Makefile` for schema updates and proto generation.
- **Documentation**: Comprehensive `dev_docs/` covering platform, infrastructure, and setup.

---

# Deployment & Infrastructure

- **Containerization**: Fully Dockerized (`Dockerfile`, `docker-compose`).
- **Secrets Management**: Integrated with **Vault** for secure credential storage.
- **CI/CD**: Automated pipelines via GitHub Actions.
- **Scalability**: Stateless architecture allows easy horizontal scaling.

---

# Conclusion

This architecture provides a **future-proof foundation**.

- **Adaptable**: Ready for changing business requirements via config.
- **Maintainable**: Clear separation of concerns and modular services.
- **Scalable**: Built on high-performance Go and cloud-native principles.

---

# Domain-Driven Design (DDD) & Abstraction

The architecture strictly separates **Business Logic** from **Infrastructure**.

- **High-Level Logic**: Developers write business rules without worrying about SQL or database specifics.
- **Abstraction**: Services use interfaces to interact with the database.

**Example: Creating an Order**

```go
// admin_order.go (Controller)
// The developer calls a high-level business method.
// No SQL here!
customerOrderInfo, err := s.api.CreateOrderForCustomer(ctx, token, data)
```

---

# Abstraction in Action

The `API Service` holds a reference to the `MainDB` interface, not a concrete SQL implementation.

```go
// api.go (Service Layer)
type Service struct {
    // Abstract interface for database operations
    mainDB main_db_service.Service 
}

// The business logic uses methods like:
// s.mainDB.CreateAddressCtx(...)
```

---

# Low-Level Infrastructure

Database operations are encapsulated in specialized services.

```go
// main_db_service/v3/interface.go (Repository Pattern)
type RepoAddress interface {
    // Concrete DB operations are defined here
    CreateAddressCtx(ctx base.Context, address *schema.Address) error
    FindAddressCtx(ctx base.Context, query interface{}) (*schema.Address, error)
}
```

---

# Lifecycle & Versioning

The platform supports **Evolutionary Architecture**.

- **Multi-Version Support**: You can run `v2` and `v3` of a service simultaneously.
  - `ecommerce_api/v2`
  - `ecommerce_api/v3`
- **Smooth Migration**: 
  - Deploy new functionality in a new service version.
  - Switch traffic gradually via configuration.
  - Deprecate old versions without breaking existing clients.

---

# System Module Tree (Generated from Config)

The system is composed of **interdependent services**.

```
+------------------+         +--------------------+
|  Router (HTTP)   |         | Background Workers |
+--------+---------+         +----------+---------+
         |                              |
         +--------------+---------------+
                        |
                        v
         +------------------------------+
         | API Service (Business Logic) |
         +--------------+---------------+
                        |
      +-----------------+-----------------+
      |                 |                 |
      v                 v                 v
+----------+     +-------------+     +----------+
| Payments |     |  Shipments  |     |   Runs   |
+-----+----+     +------+------+     +-----+----+
      |                 |                  |
      +--------+--------+------------------+
               |
               v
    +----------------------+
    |  Main DB (Postgres)  |
    +----------------------+
```

*Note: This is a simplified view based on `.config.server.yaml`.*


