# 2.3 Domain-Driven Design (DDD) Essentials

Domain-Driven Design is an approach to software development that focuses on the business domain and domain logic. It's about building software that reflects the real-world business concepts and processes.

## Key Goals of DDD

- Align software with real business problems
- Improve communication between developers and domain experts
- Reduce complexity by organizing code around business concepts
- Facilitate maintainability and scalability

## Strategic vs Tactical DDD

Before diving into concepts, it helps to know that DDD is really made of two halves, and this chapter is organized around that split:

- **Strategic DDD** is about *deciding where the boundaries go*: what is the domain, how it splits into subdomains, where one Bounded Context ends and another begins, and how those contexts talk to each other.
- **Tactical DDD** is about *how you design and code inside one boundary*: Entities, Value Objects, Aggregates, Domain Services, Domain Events, Repositories.

Most tutorials jump straight to the tactical patterns because they look like familiar object-oriented code. But the strategic half is what makes DDD different from "just good OOP" — it's the part that deals with language, meaning, and team boundaries, not just class design. Keep this distinction in mind: everything in **Part 1** is strategic, everything in **Part 2** is tactical.

---

# Part 1 — Strategic DDD

### 1. Domain

A **domain** is the specific subject area or problem space that a software addresses. It's the entire universe of discourse around the business problem.

```
DOMAIN: E-Commerce Platform
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   CORE          │  │   SUPPORTING    │  │   GENERIC   │ │
│  │   SUBDOMAINS    │  │   SUBDOMAINS    │  │   SUBDOMAINS│ │
│  │                 │  │                 │  │             │ │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │ ┌─────────┐ │ │
│  │  │  Order    │  │  │  │ Inventory │  │  │ │ Auth    │ │ │
│  │  │ Management│  │  │  │ Management│  │  │ │ System  │ │ │
│  │  └───────────┘  │  │  └───────────┘  │  │ └─────────┘ │ │
│  │                 │  │                 │  │             │ │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │ ┌─────────┐ │ │
│  │  │  Product  │  │  │  │ Shipping  │  │  │ │ Payment │ │ │
│  │  │ Catalog   │  │  │  │ Calculator│  │  │ │ Gateway │ │ │
│  │  └───────────┘  │  │  └───────────┘  │  │ └─────────┘ │ │
│  │                 │  │                 │  │             │ │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │ ┌─────────┐ │ │
│  │  │ Pricing   │  │  │  │ Customer  │  │  │ │ Email   │ │ │
│  │  │ Engine    │  │  │  │ Support   │  │  │ │ Service │ │ │
│  │  └───────────┘  │  │  └───────────┘  │  │ └─────────┘ │ │
│  └─────────────────┘  └─────────────────┘  └─────────────┘ │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**The idea**

- A **domain** is divided into subdomains.
- **Core subdomains** relate to the strategic value of the business — this is where you compete, and where investing in DDD pays off the most.
- **Supporting subdomains** are non-strategic but are business-related parts that assist the core subdomains.
- **Generic subdomains** are common across many systems and are not specific to this business — they're good candidates for off-the-shelf solutions rather than custom modeling.

### 2. Ubiquitous Language

The **Ubiquitous Language** is a shared vocabulary built jointly by developers and domain experts, and used consistently everywhere: in conversations, in documentation, and — crucially — in the code itself.

The idea is simple but easy to underestimate: if a domain expert says "we **ship** an order," the code should have `order.ship()`, not `order.updateStatus(3)`. If they talk about a "**draft** invoice" becoming "**issued**," those exact words should appear as states and method names in the domain model — not translated into generic technical jargon along the way.

Why this matters:

- **It removes translation errors.** Every time a developer silently translates business words into technical words ("cancelled" becomes `status = 4`), a small gap opens between what the business thinks the software does and what it actually does. Over time these gaps compound into serious misunderstandings.
- **It makes the domain experts' knowledge directly checkable in the code.** A domain expert who doesn't code can still read `project.archive()` or `invoice.markAsPaid()` and confirm it matches their mental model.
- **It is context-specific, not global.** The same word can — and often should — mean different things in different parts of the system. This naturally leads to the next concept: Bounded Contexts.

### 3. Bounded Contexts

A **bounded context** is a logical boundary within which a particular domain model, and a particular Ubiquitous Language, applies.

This directly follows from the previous section: if "Ubiquitous Language" only means the language is consistent *within* a boundary, then different boundaries are free — and expected — to use the same word with different meanings, because each one is home to its own Ubiquitous Language.

**Example: E-commerce SaaS**

```
┌─────────────────────────┐
│  Catalog Context        		│
│  - Product              		│  "Product" means catalog item
│  - Category             		│  with description, images
│  - Price                		│
└─────────────────────────┘

┌─────────────────────────┐
│  Shopping Context       		│
│  - Product              		│  "Product" means item in cart
│  - Cart                 		│  with quantity, selected options
│  - Checkout             		│
└─────────────────────────┘

┌─────────────────────────┐
│  Fulfillment Context    		│
│  - Product              		│  "Product" means physical item
│  - Order                		│  with SKU, warehouse location
│  - Shipment             		│
└─────────────────────────┘
```

Same word ("Product"), different meaning in each context — because each context has grown its own Ubiquitous Language around its own concerns.

**In Our SaaS Projects:**

**Java Project (EnterpriseFlow):**

```
┌─────────────────────────┐
│  Project Management     │
│  - Project              │
│  - Task                 │
│  - Milestone            │
└─────────────────────────┘

┌─────────────────────────┐
│  CRM                    │
│  - Client               │
│  - Deal                 │
│  - Contact              │
└─────────────────────────┘

┌─────────────────────────┐
│  Billing                │
│  - Invoice              │
│  - Payment              │
│  - Subscription         │
└─────────────────────────┘

┌─────────────────────────┐
│  Time Tracking          │
│  - TimeEntry            │
│  - Timesheet            │
└─────────────────────────┘
```

**Why Bounded Contexts Matter:**

1. **Reduces complexity** — Each context has its own simpler model.
2. **Enables independent evolution** — Contexts can change independently.
3. **Clear ownership** — Different teams can own different contexts.
4. **Easier to reason about** — Smaller, focused models.

### 4. Context Mapping

Bounded Contexts don't live in isolation — a "Project" from Project Management eventually needs to be billed in the Billing context, and a "Deal" won in the CRM might need to create a Project. **Context Mapping** describes the relationships between Bounded Contexts and how they exchange information without collapsing into a single, tangled model.

The most common patterns:

| Pattern | What it means |
|---|---|
| **Shared Kernel** | Two contexts deliberately share a small piece of model (and code) between them. Cheap but creates coupling — changes must be coordinated by both teams. |
| **Customer-Supplier** | One context (the supplier) produces something another context (the customer) depends on. The supplier's team takes the customer's needs into account when planning changes. |
| **Conformist** | The downstream context has no negotiating power (e.g. a third-party API) and simply conforms to the upstream model as-is, with no translation layer. |
| **Anticorruption Layer (ACL)** | The downstream context builds a translation layer that converts the upstream model into its own model, protecting its own Ubiquitous Language from being polluted by someone else's. |
| **Open Host Service / Published Language** | A context exposes a well-documented, stable protocol (often via API) that any number of other contexts can consume, instead of negotiating one-off integrations. |

**Example mapping in EnterpriseFlow:**

```
CRM  ──(Customer-Supplier)──▶  Project Management
     "Deal Won" event triggers project creation

Project Management ──(Anticorruption Layer)──▶  Billing
     Billing doesn't consume Project's internal model directly;
     it translates "completed Task" into its own "Billable Item"

Billing ──(Open Host Service)──▶  Payment Gateway (external, generic subdomain)
     Billing exposes/consumes a stable, documented API contract
```

Choosing the right pattern for each relationship is itself a design decision — it's not something you do once and forget, since the coupling it creates (or avoids) shapes how easily each team can change their context later.

---

# Part 2 — Tactical DDD

### 5. Entities

**Entities** are objects with a unique identity that persists over time.

```javascript
// Entity: Has identity (id), can change over time
class Project {
  constructor(id, name, status) {
    this.id = id; // Identity - what makes it unique
    this.name = name; // Can change
    this.status = status; // Can change
  }

  // Same project, even if name changes
  equals(other) {
    return this.id === other.id;
  }
}

const project1 = new Project("123", "Website Redesign", "active");
const project2 = new Project("123", "Website Redesign v2", "active");

console.log(project1.equals(project2)); // true - same entity!
```

### 6. Value Objects

**Value Objects** have no identity — they're defined by their attributes.

```javascript
// Value Object: No identity, defined by its values
class Money {
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }

  equals(other) {
    return this.amount === other.amount && this.currency === other.currency;
  }

  add(other) {
    if (this.currency !== other.currency) {
      throw new Error("Cannot add different currencies");
    }
    return new Money(this.amount + other.amount, this.currency);
  }
}

const price1 = new Money(100, "USD");
const price2 = new Money(100, "USD");

console.log(price1.equals(price2)); // true - same value!
console.log(price1 === price2); // false - different objects

// Value objects are immutable
const total = price1.add(price2); // Returns NEW object
console.log(total.amount); // 200
console.log(price1.amount); // Still 100 (unchanged)
```

**More Value Object Examples:**

```javascript
class Email {
  constructor(address) {
    if (!this.isValid(address)) {
      throw new Error("Invalid email address");
    }
    this.address = address;
  }

  isValid(email) {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  equals(other) {
    return this.address === other.address;
  }
}

class DateRange {
  constructor(startDate, endDate) {
    if (endDate < startDate) {
      throw new Error("End date must be after start date");
    }
    this.startDate = startDate;
    this.endDate = endDate;
  }

  getDuration() {
    return this.endDate - this.startDate;
  }

  overlaps(other) {
    return this.startDate <= other.endDate && this.endDate >= other.startDate;
  }
}
```

### 7. Anemic vs Rich Domain Model

Look back at the `Money` and `Email` examples: the validation and business rules (`isValid`, "currency mismatch") live *inside* the object, not in a separate helper that manipulates plain data. This is a deliberate choice, and it has a name.

- An **Anemic Domain Model** is one where domain objects are little more than data bags (just getters/setters, or plain fields), and all the actual business logic lives in external "service" classes that read and write those fields.
- A **Rich Domain Model** puts the business rules on the domain objects themselves — the object protects its own invariants and exposes behavior, not just data.

```javascript
// Anemic style — logic lives outside the object
class MoneyData {
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }
}

function addMoney(a, b) {
  if (a.currency !== b.currency) throw new Error("Currency mismatch");
  return new MoneyData(a.amount + b.amount, a.currency);
}

// Rich style — the object enforces its own rule
class Money {
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }
  add(other) {
    if (this.currency !== other.currency) throw new Error("Currency mismatch");
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

The anemic version isn't "wrong" for a simple CRUD app, but it defeats the purpose of DDD: nothing stops another part of the codebase from adding two different currencies together by mistake, because the rule lives outside the data instead of being attached to it. Every `Order`, `Project`, and `Money` example in this chapter follows the rich model on purpose.

### 8. Aggregates

An **aggregate** is a cluster of entities and value objects that form a consistency boundary.

**Rules:**

1. One entity is the **aggregate root** (entry point).
2. External objects can only reference the aggregate root.
3. Transactions should not cross aggregate boundaries.

**Example:**

```javascript
// Order is the Aggregate Root
class Order {
  constructor(id, customerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = []; // Order owns OrderItems
    this.status = "pending";
  }

  // Only way to add items is through the aggregate root
  addItem(productId, quantity, price) {
    // Business rule enforcement
    if (this.status === "shipped") {
      throw new Error("Cannot modify shipped order");
    }

    const existingItem = this.items.find((i) => i.productId === productId);
    if (existingItem) {
      existingItem.increaseQuantity(quantity);
    } else {
      this.items.push(new OrderItem(productId, quantity, price));
    }
  }

  // Business rules enforced at aggregate level
  calculateTotal() {
    return this.items.reduce((sum, item) => sum + item.getSubtotal(), 0);
  }

  canBeCancelled() {
    return this.status === "pending" || this.status === "confirmed";
  }

  cancel() {
    if (!this.canBeCancelled()) {
      throw new Error(`Cannot cancel order with status: ${this.status}`);
    }
    this.status = "cancelled";
  }
}

// OrderItem is part of the Order aggregate
class OrderItem {
  constructor(productId, quantity, price) {
    this.productId = productId;
    this.quantity = quantity;
    this.price = price;
  }

  increaseQuantity(amount) {
    this.quantity += amount;
  }

  getSubtotal() {
    return this.quantity * this.price;
  }
}

// Usage
const order = new Order("order-123", "customer-456");
order.addItem("product-1", 2, 29.99); // Goes through aggregate root
order.addItem("product-2", 1, 49.99);
```

#### Why the boundary exists: invariants

An **invariant** is a business rule that must always hold true, no exception, no matter which code path touches the data. In the example above, "you cannot add items to a shipped order" is an invariant. The aggregate root is the *only* gatekeeper for that invariant — which is exactly why rule #2 exists (external code can't reach `OrderItem` directly) and why rule #3 exists (a single transaction should save or reject the whole aggregate together, so it never ends up half-updated in an invalid state).

Here's a concrete example of what rule #2 is protecting against — not just a comment, but code that actually breaks the invariant:

```javascript
// ❌ BAD: bypassing the aggregate root
const shippedOrder = new Order("order-999", "customer-1");
shippedOrder.status = "shipped";

// Nothing stops this if OrderItem is touched directly:
const rogueItem = new OrderItem("product-3", 5, 10);
shippedOrder.items.push(rogueItem); // invariant silently broken!

console.log(shippedOrder.status); // still "shipped"
console.log(shippedOrder.items.length); // item was added anyway — bug

// ✅ GOOD: same attempt through the aggregate root
shippedOrder.addItem("product-3", 5, 10);
// throws Error: "Cannot modify shipped order" — invariant protected
```

Between *different* aggregates (e.g. an `Order` and a `Customer`), the rule is different: you don't force them into the same transaction. Instead, you accept that they become consistent with each other slightly after the fact — this is called **eventual consistency**, and it's exactly what Domain Events (below) are for.

#### Sizing an aggregate

A common beginner mistake is to make aggregates too large (e.g. one giant `Customer` aggregate holding every order they've ever placed). Some practical heuristics (popularized by Vernon):

- **Keep aggregates small.** Model true invariants only — if two pieces of data don't need to change together atomically, they probably don't belong in the same aggregate.
- **Reference other aggregates by ID, not by object.** An `Order` should store `customerId`, not a full `Customer` object — this keeps aggregates decoupled and avoids accidentally loading (or locking) far more data than a transaction needs.
- **One transaction, one aggregate.** If you find yourself needing to update two aggregates atomically, that's usually a sign either the boundary is wrong, or you need a Domain Event to coordinate them asynchronously instead.

**Aggregate Design in Our Projects:**

```javascript
// Project Aggregate (Java Project - EnterpriseFlow)
class Project {
  constructor(id, tenantId, name) {
    this.id = id;
    this.tenantId = tenantId;
    this.name = name;
    this.tasks = []; // Project owns Tasks
    this.milestones = []; // Project owns Milestones
  }

  addTask(taskData) {
    // Validate business rules
    if (this.status === "archived") {
      throw new Error("Cannot add tasks to archived project");
    }

    const task = new Task(uuid(), this.id, taskData);
    this.tasks.push(task);
    return task;
  }

  completeTask(taskId) {
    const task = this.tasks.find((t) => t.id === taskId);
    if (!task) {
      throw new Error("Task not found");
    }

    task.complete();

    // Check if all tasks are complete
    if (this.allTasksComplete()) {
      this.status = "ready_for_review";
    }
  }

  allTasksComplete() {
    return this.tasks.every((t) => t.status === "completed");
  }
}

class Task {
  constructor(id, projectId, data) {
    this.id = id;
    this.projectId = projectId;
    this.title = data.title;
    this.status = "todo";
  }

  complete() {
    if (this.status === "completed") {
      throw new Error("Task already completed");
    }
    this.status = "completed";
    this.completedAt = new Date();
  }
}
```

### 9. Domain Services

Some business logic doesn't naturally belong to any single Entity or Value Object — usually because it involves *several* aggregates at once. Forcing that logic onto one of them would be arbitrary and would blur its responsibility. That's what a **Domain Service** is for: a stateless object, named with a verb from the Ubiquitous Language, that holds logic which spans multiple domain objects.

```javascript
// Domain Service — logic doesn't belong to Account alone,
// it inherently involves two accounts at once
class MoneyTransferService {
  transfer(fromAccount, toAccount, amount) {
    if (fromAccount.balance.amount < amount.amount) {
      throw new Error("Insufficient funds");
    }
    fromAccount.withdraw(amount);
    toAccount.deposit(amount);
  }
}
```

**Domain Service vs Application Service — don't confuse the two:**

- A **Domain Service** contains *business rules* (e.g. "a transfer requires sufficient funds"). It belongs to the domain layer and knows nothing about HTTP, databases, or transactions.
- An **Application Service** (like `ProjectService` later in this chapter) *orchestrates*: it loads aggregates from repositories, calls domain logic, saves the result, and publishes events. It coordinates, but it should not itself contain business rules.

### 10. Factories

You've already seen a Factory in this chapter without the name attached to it: `Project.create(tenantId, name, createdBy)`, used later in the Domain Events section, is a **Factory** — a piece of code whose only job is to construct a valid aggregate.

Why not just use `new Project(...)` everywhere? Two reasons:

- **Complex construction logic.** Creating a valid aggregate sometimes requires several steps, defaults, or validations that don't belong in a bare constructor.
- **Guaranteeing a valid starting state.** A Factory ensures an aggregate is never observed in an incomplete or invalid state — including raising the "creation" Domain Event as part of the same step, so nothing can create a `Project` without also recording that it was created.

```javascript
class Project {
  constructor(id, tenantId, name) {
    this.id = id;
    this.tenantId = tenantId;
    this.name = name;
    this.domainEvents = [];
  }

  // Factory method
  static create(tenantId, name, createdBy) {
    const project = new Project(uuid(), tenantId, name);
    project.addDomainEvent(
      new ProjectCreatedEvent(tenantId, project.id, createdBy, new Date())
    );
    return project;
  }

  addDomainEvent(event) {
    this.domainEvents.push(event);
  }
}
```

### 11. Domain Events

**Domain events** represent something that happened in the domain.

```javascript
// Domain Event
class ProjectCreatedEvent {
  constructor(tenantId, projectId, createdBy, timestamp) {
    this.tenantId = tenantId;
    this.projectId = projectId;
    this.createdBy = createdBy;
    this.timestamp = timestamp;
  }
}

class TaskCompletedEvent {
  constructor(tenantId, projectId, taskId, completedBy, timestamp) {
    this.tenantId = tenantId;
    this.projectId = projectId;
    this.taskId = taskId;
    this.completedBy = completedBy;
    this.timestamp = timestamp;
  }
}

// Aggregate raises events
class Project {
  constructor(id, tenantId, name) {
    this.id = id;
    this.tenantId = tenantId;
    this.name = name;
    this.domainEvents = []; // Collect events
  }

  static create(tenantId, name, createdBy) {
    const project = new Project(uuid(), tenantId, name);

    // Raise domain event
    project.addDomainEvent(
      new ProjectCreatedEvent(tenantId, project.id, createdBy, new Date())
    );

    return project;
  }

  completeTask(taskId, userId) {
    const task = this.tasks.find((t) => t.id === taskId);
    task.complete();

    // Raise domain event
    this.addDomainEvent(
      new TaskCompletedEvent(this.tenantId, this.id, taskId, userId, new Date())
    );
  }

  addDomainEvent(event) {
    this.domainEvents.push(event);
  }

  getDomainEvents() {
    return [...this.domainEvents];
  }

  clearDomainEvents() {
    this.domainEvents = [];
  }
}

// Service publishes events after successful save
class ProjectService {
  async createProject(tenantId, userId, data) {
    const project = Project.create(tenantId, data.name, userId);

    // Save to database
    await this.projectRepo.save(project);

    // Publish domain events
    const events = project.getDomainEvents();
    for (const event of events) {
      await this.eventBus.publish(event);
    }
    project.clearDomainEvents();

    return project;
  }
}

// Event handlers react to events
class ProjectCreatedEventHandler {
  async handle(event) {
    // Send notification
    await notificationService.send(event.tenantId, {
      type: "project_created",
      projectId: event.projectId,
    });

    // Record analytics
    await analyticsService.track(event.tenantId, "project_created", {
      projectId: event.projectId,
      timestamp: event.timestamp,
    });

    // Update dashboard stats
    await dashboardService.incrementProjectCount(event.tenantId);
  }
}
```

**The missing link with Aggregates:** recall that a single transaction should touch only one aggregate. So how does a `BudgetExceededEvent` on a `Project` end up notifying, say, a separate `Billing` aggregate or context? It doesn't do it directly and synchronously — it publishes a Domain Event, and whoever needs to react (another aggregate, another Bounded Context, a notification system) does so afterward, in its own transaction. This is precisely how **eventual consistency** between aggregates is achieved in practice.

### 12. Repositories (DDD Pattern)

In DDD, **repositories** provide collection-like interfaces for aggregates.

```javascript
// Repository interface (what services expect)
class IProjectRepository {
  async findById(tenantId, projectId) {}
  async findAll(tenantId, criteria) {}
  async save(project) {}
  async delete(tenantId, projectId) {}
}

// Implementation
class ProjectRepository {
  constructor(db) {
    this.db = db;
  }

  // Find aggregate with all its parts
  async findById(tenantId, projectId) {
    const projectData = await this.db.query(
      `
      SELECT * FROM projects
      WHERE id = $1 AND tenant_id = $2
    `,
      [projectId, tenantId]
    );

    if (!projectData.rows[0]) return null;

    // Load tasks (part of aggregate)
    const tasksData = await this.db.query(
      `
      SELECT * FROM tasks
      WHERE project_id = $1
    `,
      [projectId]
    );

    // Reconstruct aggregate
    const project = new Project(
      projectData.rows[0].id,
      projectData.rows[0].tenant_id,
      projectData.rows[0].name
    );

    project.tasks = tasksData.rows.map((t) => new Task(t.id, t.project_id, t));

    return project;
  }

  // Save entire aggregate
  async save(project) {
    await this.db.transaction(async (trx) => {
      // Save root
      await trx.query(
        `
        INSERT INTO projects (id, tenant_id, name, status)
        VALUES ($1, $2, $3, $4)
        ON CONFLICT (id) DO UPDATE
        SET name = $3, status = $4, updated_at = NOW()
      `,
        [project.id, project.tenantId, project.name, project.status]
      );

      // Save all tasks
      for (const task of project.tasks) {
        await trx.query(
          `
          INSERT INTO tasks (id, project_id, title, status)
          VALUES ($1, $2, $3, $4)
          ON CONFLICT (id) DO UPDATE
          SET title = $3, status = $4, updated_at = NOW()
        `,
          [task.id, project.id, task.title, task.status]
        );
      }
    });
  }
}
```

**DDD Repository vs generic CRUD Repository — a common source of confusion:**

A repository in the DDD sense is not just "a class with `findById`/`save`" — it is specifically about persisting and reconstructing a *whole aggregate* while respecting its invariants: notice above how `save()` writes the `Project` root *and* its `Task` children together, inside one transaction, because they're the same aggregate. A generic CRUD repository, by contrast, typically maps one repository to one database table with no notion of aggregate boundaries or consistency — which works fine for simple data, but silently breaks invariants the moment a "repository" per table is used to persist parts of an aggregate independently.

---

# Part 3 — Putting It All Together

### 13. DDD in Practice: Example Flow

Let's see how all DDD concepts work together:

```javascript
// 1. Domain Layer - Entities, Value Objects, Aggregates

class Money {
  // Value Object
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }

  add(other) {
    if (this.currency !== other.currency) {
      throw new Error("Currency mismatch");
    }
    return new Money(this.amount + other.amount, this.currency);
  }
}

class Project {
  // Aggregate Root
  constructor(id, tenantId, name, budget) {
    this.id = id;
    this.tenantId = tenantId;
    this.name = name;
    this.budget = budget; // Money value object
    this.expenses = [];
    this.domainEvents = [];
  }

  addExpense(description, amount) {
    const expense = new Expense(uuid(), description, amount);
    this.expenses.push(expense);

    // Business rule: Check if over budget
    const totalExpenses = this.calculateTotalExpenses();
    if (totalExpenses.amount > this.budget.amount) {
      this.addDomainEvent(
        new BudgetExceededEvent(
          this.tenantId,
          this.id,
          totalExpenses,
          this.budget
        )
      );
    }

    return expense;
  }

  calculateTotalExpenses() {
    return this.expenses.reduce(
      (total, exp) => total.add(exp.amount),
      new Money(0, this.budget.currency)
    );
  }

  addDomainEvent(event) {
    this.domainEvents.push(event);
  }
}

class Expense {
  // Entity (part of Project aggregate)
  constructor(id, description, amount) {
    this.id = id;
    this.description = description;
    this.amount = amount; // Money value object
    this.createdAt = new Date();
  }
}

// 2. Domain Events

class BudgetExceededEvent {
  constructor(tenantId, projectId, actualExpenses, budget) {
    this.tenantId = tenantId;
    this.projectId = projectId;
    this.actualExpenses = actualExpenses;
    this.budget = budget;
    this.timestamp = new Date();
  }
}

// 3. Repository

class ProjectRepository {
  async findById(tenantId, projectId) {
    // Load from database and reconstruct aggregate
    const data = await this.db.query(/* ... */);
    return this.toDomain(data);
  }

  async save(project) {
    // Save entire aggregate
    await this.db.transaction(async (trx) => {
      await this.saveProject(trx, project);
      await this.saveExpenses(trx, project.expenses);
    });
  }

  toDomain(data) {
    // Convert database records to domain objects
    const project = new Project(
      data.id,
      data.tenant_id,
      data.name,
      new Money(data.budget_amount, data.budget_currency)
    );

    project.expenses = data.expenses.map(
      (e) => new Expense(e.id, e.description, new Money(e.amount, e.currency))
    );

    return project;
  }
}

// 4. Application Service

class ProjectService {
  constructor(projectRepo, eventBus) {
    this.projectRepo = projectRepo;
    this.eventBus = eventBus;
  }

  async addExpenseToProject(tenantId, projectId, expenseData) {
    // 1. Load aggregate
    const project = await this.projectRepo.findById(tenantId, projectId);
    if (!project) {
      throw new NotFoundError("Project not found");
    }

    // 2. Execute business logic (on domain object)
    const amount = new Money(expenseData.amount, expenseData.currency);
    const expense = project.addExpense(expenseData.description, amount);

    // 3. Save aggregate
    await this.projectRepo.save(project);

    // 4. Publish domain events
    for (const event of project.domainEvents) {
      await this.eventBus.publish(event);
    }
    project.domainEvents = [];

    return expense;
  }
}

// 5. Event Handler

class BudgetExceededEventHandler {
  async handle(event) {
    // Send notification to project owner
    await notificationService.send(event.tenantId, {
      type: "budget_exceeded",
      projectId: event.projectId,
      message: `Project expenses (${event.actualExpenses.amount}) exceeded budget (${event.budget.amount})`,
    });

    // Log for audit
    await auditService.log(event.tenantId, {
      type: "budget_exceeded",
      projectId: event.projectId,
      timestamp: event.timestamp,
    });
  }
}

// 6. Controller

class ProjectController {
  async addExpense(req, res) {
    try {
      const expense = await projectService.addExpenseToProject(
        req.tenant.id,
        req.params.projectId,
        req.body
      );

      res.status(201).json(expense);
    } catch (error) {
      // Error handling
      res.status(500).json({ error: error.message });
    }
  }
}
```

### 14. Where DDD Fits in the Architecture

Look at the flow of the example above: **Controller → Application Service → Domain (Aggregates, Value Objects, Events) → Repository → Database**. This is a **layered architecture**, and specifically follows the spirit of **Hexagonal Architecture** (also called **Ports & Adapters**):

- The **Domain layer** (`Project`, `Money`, `Expense`, the events) depends on nothing else in the system — no database, no web framework, no external library. It's pure business logic and can be tested in complete isolation.
- The **Repository** is a *port*: an interface the Domain/Application layer depends on, with a concrete database *adapter* behind it. The Domain never talks to SQL directly.
- The **Controller** is another *adapter*: it translates an HTTP request into a call to the Application Service, and translates the result back into an HTTP response.

The dependency rule is always one-directional: **everything depends on the Domain, the Domain depends on nothing.** This is what makes it possible to swap the database, the web framework, or the message bus without touching a single business rule — and it's also why the Domain Services introduced earlier must stay free of any infrastructure concern.

### 15. When to Use DDD

**✅ Use DDD When:**

- Complex business logic
- Domain experts available
- Long-term project (worth the upfront investment)
- Team familiar with DDD
- Multiple teams working on same system

**Examples:**

- Financial systems (complex rules)
- Healthcare (complex workflows)
- E-commerce (complex business logic)
- Enterprise SaaS with sophisticated features

**❌ Don't Use DDD When:**

- Simple CRUD operations
- Data-driven applications with minimal logic
- Tight deadlines (DDD has a learning curve)
- Small team unfamiliar with DDD
- Prototype/MVP phase

**Examples:**

- Simple todo app
- Content management system
- Basic forms/surveys
