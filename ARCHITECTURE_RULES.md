# Architecture Rules for AI Code Generation

You are assisting a Principal Software Engineer who follows **volatility-based decomposition** architecture. All code you generate must adhere to these principles.

---

## Core Principle

**Decompose by volatility, not by function.** Group components that change together. Separate components that change for different reasons. This minimizes the ripple effect of change.

---

## The Five Layers

All code must be organized into these layers. Calls flow downward: 1 → 2 → 3 → 4. Layer 5 (Utilities) can be called from any layer.

| Layer | Components | Responsibility | Volatility |
|-------|------------|----------------|------------|
| 1. CLIENT | UI, APIs, Controllers, Notifications | Present to humans & external systems | HIGH |
| 2. MANAGER | Use Case Orchestrators | WHAT is being executed | MEDIUM |
| ↳ ENGINE | Business Logic Units | HOW part of use case works | MEDIUM |
| 3. RESOURCE ACCESS | Data Access Components | Atomic business verbs on resources | LOW-MED |
| 4. RESOURCE | DB, Cache, Queue, Files | Physical storage & infrastructure | LOW |
| 5. UTILITIES | Logging, Security, Config, Diagnostics | Cross-cutting concerns (vertical) | LOW |

### Layer Rules

- **Never skip layers.** Client must not call ResourceAccess or Resource directly.
- **Dependencies point downward only.** A Manager can call ResourceAccess, but ResourceAccess cannot call Manager.
- **Utilities are the exception.** They can be called from any layer.

---

## Manager vs Engine

This is the critical distinction. Get this right.

| Aspect | Manager | Engine |
|--------|---------|--------|
| Answers | WHAT use case executes | HOW logic is implemented |
| Orchestrates | Yes — calls Engines and ResourceAccess | No — is called by Manager |
| Contains | Workflow rules, sequencing | Domain logic, calculations, algorithms |
| Reusable | No — specific to one use case | Yes — shared across use cases |
| Example | `OrderManager`, `CheckoutManager` | `PricingEngine`, `TaxEngine`, `ValidationEngine` |

### Decision Rule

- "What happens when a user clicks X?" → **Manager**
- "How do we calculate Y?" → **Engine**

---

## Naming Conventions

Always use these suffixes:

| Layer | Suffix | Examples |
|-------|--------|----------|
| Client | `-Controller`, `-API`, `-View`, `-Handler` | `OrderController`, `PaymentAPI` |
| Manager | `-Manager` | `OrderManager`, `CustomerManager` |
| Engine | `-Engine` | `PricingEngine`, `ValidationEngine` |
| Resource Access | `-Access`, `-Repository` | `OrderAccess`, `CustomerRepository` |
| Utility | `-Service`, `-Provider`, `-Helper` | `LoggingService`, `ConfigProvider` |

---

## Resource Access Patterns

### 1. Atomic Business Verbs

Methods must express domain operations, not CRUD:

```
❌ BAD:  getCustomer(id)
✅ GOOD: findActiveCustomerById(id)

❌ BAD:  updateOrder(order)
✅ GOOD: markOrderAsShipped(orderId, trackingNumber)

❌ BAD:  delete(id)
✅ GOOD: cancelPendingOrder(orderId, reason)
```

### 2. Domain Type Returns

Never leak infrastructure types through the contract:

```
❌ BAD:  ResultSet executeQuery(String sql)
✅ GOOD: List<Order> findPendingOrders()

❌ BAD:  JsonNode getResponse()
✅ GOOD: CustomerProfile fetchCustomerProfile(customerId)
```

### 3. Transaction Boundaries

- ResourceAccess owns transaction scope
- Managers coordinate but don't manage connections
- Never pass connection/session objects between layers

### 4. Caching Strategy

- Cache at ResourceAccess level
- Invalidate based on write operations
- Manager should not know about cache existence

---

## Architecture Smells to Avoid

When generating code, check for these anti-patterns:

| Smell | Symptom | Fix |
|-------|---------|-----|
| Layer Skip | Client calls Resource directly | Add Manager and/or ResourceAccess layer |
| Fat Manager | Manager has 500+ lines or too many dependencies | Extract Engines for reusable logic |
| Anemic Engine | Engine just passes through to ResourceAccess | Merge into Manager or delete |
| Leaky Abstraction | DB types (ResultSet, Entity) in Manager | ResourceAccess must return domain types |
| Utility Creep | Business logic in Utility class | Move to appropriate Engine |
| Circular Dependencies | Managers calling each other | Extract shared logic into Engine |
| God ResourceAccess | One RA class handles multiple resources | Split by resource/aggregate |

---

## DDD Alignment

When working with Domain-Driven Design concepts, map them as follows:

| DDD Concept | Maps To |
|-------------|---------|
| Bounded Context | One or more Managers |
| Aggregate | Engine (logic) + ResourceAccess (persistence) |
| Domain Event | Flows between Managers via Utilities (event bus) |
| Anti-Corruption Layer | ResourceAccess component |
| Ubiquitous Language | Enforced at Manager contract level |
| Value Object | Immutable data classes, used across all layers |
| Entity | Domain object with identity, managed by ResourceAccess |

---

## Security by Layer

Apply security controls at the appropriate layer:

| Layer | Security Concerns |
|-------|-------------------|
| Client | Input validation, AuthN/AuthZ, Rate limiting, CORS, CSP headers |
| Manager | Business rule authorization, Audit logging, Sensitive data masking |
| Engine | Algorithm security, Side-channel prevention, Secure defaults |
| Resource Access | SQL injection prevention, Connection encryption, Least privilege |
| Resource | Encryption at rest, Access controls, Backup security |
| Utilities | Secret management, Log sanitization, Secure configuration |

---

## Code Style Preferences

When generating code, follow these preferences:

### General

- Prefer **immutable data structures**
- Write **functions without side effects** where possible
- Use **domain types** over primitives (e.g., `OrderId` not `String`)
- Favor **composition over inheritance**

### Python

- Use `uv` for dependency management and builds
- Use `@dataclass(frozen=True)` for immutable value objects
- Use `Protocol` for interfaces/contracts
- Type hints are mandatory

### Java

- Design for **Docker environments**
- Use records for value objects (Java 16+)
- Use interfaces for layer contracts
- Constructor injection for dependencies

### Scala

- Use **ZIO** for effects
- Prefer `case class` for domain objects
- Use `trait` for layer contracts

---

## Decomposition Checklist

Before generating a new component, verify:

1. ☐ Which layer does this belong to?
2. ☐ Is it a Manager (WHAT) or Engine (HOW)?
3. ☐ Does it follow the naming convention?
4. ☐ Does it only depend on layers below it?
5. ☐ Does ResourceAccess return domain types?
6. ☐ Is there a single reason for this component to change?
7. ☐ Are cross-cutting concerns in Utilities?
8. ☐ Is security applied at the correct layer?

---

## Example Structure

```
src/
├── client/                     # Layer 1: Client
│   ├── api/
│   │   └── OrderController.py
│   └── handlers/
│       └── WebhookHandler.py
├── manager/                    # Layer 2: Manager
│   ├── OrderManager.py
│   └── CustomerManager.py
├── engine/                     # Layer 2: Engine (sub-layer)
│   ├── PricingEngine.py
│   ├── TaxEngine.py
│   └── ValidationEngine.py
├── resource_access/            # Layer 3: Resource Access
│   ├── OrderAccess.py
│   ├── CustomerRepository.py
│   └── PaymentGatewayAccess.py
├── resource/                   # Layer 4: Resource (config/schemas)
│   ├── database/
│   └── cache/
├── utilities/                  # Layer 5: Utilities
│   ├── LoggingService.py
│   ├── ConfigProvider.py
│   └── SecurityService.py
└── domain/                     # Shared domain types
    ├── entities/
    ├── value_objects/
    └── events/
```

---

## Quick Reference

When asked to create something, map it:

| Request | Create | Layer |
|---------|--------|-------|
| "Create an API endpoint" | `*Controller` or `*API` | Client |
| "Implement user registration" | `RegistrationManager` | Manager |
| "Calculate shipping cost" | `ShippingEngine` | Engine |
| "Fetch orders from database" | `OrderAccess` | Resource Access |
| "Add logging" | `LoggingService` | Utilities |
| "Create a data model" | Domain class | Domain (shared) |

---

## Reference

Based on *Righting Software* by Juval Löwy (The Method) and Domain-Driven Design principles by Eric Evans.
