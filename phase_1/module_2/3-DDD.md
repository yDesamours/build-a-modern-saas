## 2.3 Domain-Driven Design (DDD) Essentials

Domain-Driven Design is an approach to software development that focuses on the business domain and domain logic. It's about building software that reflects the real-world business concepts and processes.

### Key Goals of DDD

- Align software with real business problems
- Improve communication between developers and domain experts
- Reduce complexity by organizing code around business concepts
- Facilitate maintainability and scalability


### Core DDD Concepts
DDD introduces key concepts to help structure applications.

#### 1. Domain

A **domain** is the specific subject area or problem space that a software addresses. It's the entire universe of discourse around the business problem.

```
DOMAIN: E-Commerce Platform
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐ │
│  │   CORE          │  │   SUPPORTING    │  │   GENERIC   │ │
│  │   SUBDOMAINS    │  │   SUBDOMAINS    │  │   SUBDOMAINS │ │
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
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**The idea**
- **Domain** is divised in subdomain
- **Core subdomains** relates to the strategic value of the business.
-  **Supporting subdomains** are non-strategic but are business related parts which assist the core subdomains.
-  **Generic subdomains** are common across many systems and are non business related.

#### 2. Bounded Contexts

A **bounded context** is a logical boundary within which a particular domain model applies.

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

Same concept ("Product"), different meaning in each context!

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

1. **Reduces complexity** - Each context has its own simpler model
2. **Enables independent evolution** - Contexts can change independently
3. **Clear ownership** - Different teams can own different contexts
4. **Easier to reason about** - Smaller, focused models

#### 3. Entities

**Entities** are objects with a unique identity that persists over time.

```javascript
// Entity: Has identity (id), can change over time
class Project {
  constructor(id, name, status) {
    this.id = id;          // Identity - what makes it unique
    this.name = name;      // Can change
    this.status = status;  // Can change
  }
  
  // Same project, even if name changes
  equals(other) {
    return this.id === other.id;
  }
}

const project1 = new Project('123', 'Website Redesign', 'active');
const project2 = new Project('123', 'Website Redesign v2', 'active');

console.log(project1.equals(project2)); // true - same entity!
```

#### 4. Value Objects

**Value Objects** have no identity - they're defined by their attributes.

```javascript
// Value Object: No identity, defined by its values
class Money {
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }
  
  equals(other) {
    return this.amount === other.amount && 
           this.currency === other.currency;
  }
  
  add(other) {
    if (this.currency !== other.currency) {
      throw new Error('Cannot add different currencies');
    }
    return new Money(this.amount + other.amount, this.currency);
  }
}

const price1 = new Money(100, 'USD');
const price2 = new Money(100, 'USD');

console.log(price1.equals(price2)); // true - same value!
console.log(price1 === price2);     // false - different objects

// Value objects are immutable
const total = price1.add(price2);  // Returns NEW object
console.log(total.amount);         // 200
console.log(price1.amount);        // Still 100 (unchanged)
```

**More Value Object Examples:**

```javascript
class Email {
  constructor(address) {
    if (!this.isValid(address)) {
      throw new Error('Invalid email address');
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
      throw new Error('End date must be after start date');
    }
    this.startDate = startDate;
    this.endDate = endDate;
  }
  
  getDuration() {
    return this.endDate - this.startDate;
  }
  
  overlaps(other) {
    return this.startDate <= other.endDate && 
           this.endDate >= other.startDate;
  }
}
```

#### 5. Aggregates

An **aggregate** is a cluster of entities and value objects that form a consistency boundary.

**Rules:**
1. One entity is the **aggregate root** (entry point)
2. External objects can only reference the aggregate root
3. Transactions should not cross aggregate boundaries

**Example:**

```javascript
// Order is the Aggregate Root
class Order {
  constructor(id, customerId) {
    this.id = id;
    this.customerId = customerId;
    this.items = [];  // Order owns OrderItems
    this.status = 'pending';
  }
  
  // Only way to add items is through the aggregate root
  addItem(productId, quantity, price) {
    // Business rule enforcement
    if (this.status === 'shipped') {
      throw new Error('Cannot modify shipped order');
    }
    
    const existingItem = this.items.find(i => i.productId === productId);
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
    return this.status === 'pending' || this.status === 'confirmed';
  }
  
  cancel() {
    if (!this.canBeCancelled()) {
      throw new Error(`Cannot cancel order with status: ${this.status}`);
    }
    this.status = 'cancelled';
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
const order = new Order('order-123', 'customer-456');
order.addItem('product-1', 2, 29.99);  // Goes through aggregate root
order.addItem('product-2', 1, 49.99);

// ❌ BAD: Don't access OrderItem directly
// orderItem.quantity = 10;  // Bypasses business rules!

// ✅ GOOD: Always through aggregate root
order.addItem('product-1', 3, 29.99);  // Enforces business rules
```

**Aggregate Design in Our Projects:**

```javascript
// Project Aggregate (Java Project - EnterpriseFlow)
class Project {
  constructor(id, tenantId, name) {
    this.id = id;
    this.tenantId = tenantId;
    this.name = name;
    this.tasks = [];        // Project owns Tasks
    this.milestones = [];   // Project owns Milestones
  }
  
  addTask(taskData) {
    // Validate business rules
    if (this.status === 'archived') {
      throw new Error('Cannot add tasks to archived project');
    }
    
    const task = new Task(uuid(), this.id, taskData);
    this.tasks.push(task);
    return task;
  }
  
  completeTask(taskId) {
    const task = this.tasks.find(t => t.id === taskId);
    if (!task) {
      throw new Error('Task not found');
    }
    
    task.complete();
    
    // Check if all tasks are complete
    if (this.allTasksComplete()) {
      this.status = 'ready_for_review';
    }
  }
  
  allTasksComplete() {
    return this.tasks.every(t => t.status === 'completed');
  }
}

class Task {
  constructor(id, projectId, data) {
    this.id = id;
    this.projectId = projectId;
    this.title = data.title;
    this.status = 'todo';
  }
  
  complete() {
    if (this.status === 'completed') {
      throw new Error('Task already completed');
    }
    this.status = 'completed';
    this.completedAt = new Date();
  }
}
```

#### 6. Domain Events

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
    this.domainEvents = [];  // Collect events
  }
  
  static create(tenantId, name, createdBy) {
    const project = new Project(uuid(), tenantId, name);
    
    // Raise domain event
    project.addDomainEvent(new ProjectCreatedEvent(
      tenantId,
      project.id,
      createdBy,
      new Date()
    ));
    
    return project;
  }
  
  completeTask(taskId, userId) {
    const task = this.tasks.find(t => t.id === taskId);
    task.complete();
    
    // Raise domain event
    this.addDomainEvent(new TaskCompletedEvent(
      this.tenantId,
      this.id,
      taskId,
      userId,
      new Date()
    ));
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
      type: 'project_created',
      projectId: event.projectId
    });
    
    // Record analytics
    await analyticsService.track(event.tenantId, 'project_created', {
      projectId: event.projectId,
      timestamp: event.timestamp
    });
    
    // Update dashboard stats
    await dashboardService.incrementProjectCount(event.tenantId);
  }
}
```

#### 7. Repositories (DDD Pattern)

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
    const projectData = await this.db.query(`
      SELECT * FROM projects
      WHERE id = $1 AND tenant_id = $2
    `, [projectId, tenantId]);
    
    if (!projectData.rows[0]) return null;
    
    // Load tasks (part of aggregate)
    const tasksData = await this.db.query(`
      SELECT * FROM tasks
      WHERE project_id = $1
    `, [projectId]);
    
    // Reconstruct aggregate
    const project = new Project(
      projectData.rows[0].id,
      projectData.rows[0].tenant_id,
      projectData.rows[0].name
    );
    
    project.tasks = tasksData.rows.map(t => new Task(t.id, t.project_id, t));
    
    return project;
  }
  
  // Save entire aggregate
  async save(project) {
    await this.db.transaction(async (trx) => {
      // Save root
      await trx.query(`
        INSERT INTO projects (id, tenant_id, name, status)
        VALUES ($1, $2, $3, $4)
        ON CONFLICT (id) DO UPDATE
        SET name = $3, status = $4, updated_at = NOW()
      `, [project.id, project.tenantId, project.name, project.status]);
      
      // Save all tasks
      for (const task of project.tasks) {
        await trx.query(`
          INSERT INTO tasks (id, project_id, title, status)
          VALUES ($1, $2, $3, $4)
          ON CONFLICT (id) DO UPDATE
          SET title = $3, status = $4, updated_at = NOW()
        `, [task.id, project.id, task.title, task.status]);
      }
    });
  }
}
```

---

### DDD in Practice: Example Flow

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
      throw new Error('Currency mismatch');
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
    this.budget = budget;  // Money value object
    this.expenses = [];
    this.domainEvents = [];
  }
  
  addExpense(description, amount) {
    const expense = new Expense(uuid(), description, amount);
    this.expenses.push(expense);
    
    // Business rule: Check if over budget
    const totalExpenses = this.calculateTotalExpenses();
    if (totalExpenses.amount > this.budget.amount) {
      this.addDomainEvent(new BudgetExceededEvent(
        this.tenantId,
        this.id,
        totalExpenses,
        this.budget
      ));
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
    this.amount = amount;  // Money value object
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
    
    project.expenses = data.expenses.map(e => 
      new Expense(
        e.id,
        e.description,
        new Money(e.amount, e.currency)
      )
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
      throw new NotFoundError('Project not found');
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
      type: 'budget_exceeded',
      projectId: event.projectId,
      message: `Project expenses (${event.actualExpenses.amount}) exceeded budget (${event.budget.amount})`
    });
    
    // Log for audit
    await auditService.log(event.tenantId, {
      type: 'budget_exceeded',
      projectId: event.projectId,
      timestamp: event.timestamp
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

---

### When to Use DDD

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
- Tight deadlines (DDD has learning curve)
- Small team unfamiliar with DDD
- Prototype/MVP phase

**Examples:**
- Simple todo app
- Content management system
- Basic forms/surveys

---
