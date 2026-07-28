# 2.4 CQRS (Command Query Responsibility Segregation)

CQRS is an advanced architectural pattern that solves a specific problem. It's NOT for every application, but when you need it, it's powerful.

CQRS and Event Sourcing are two **independent** patterns, even though they're often mentioned in the same breath. You can use CQRS without ever touching Event Sourcing — most real systems do exactly that. This chapter covers CQRS on its own. Event Sourcing gets its own chapter, and there it's presented as one possible (advanced) way to implement the write side of a CQRS system — not a requirement.

## What is CQRS?

**CQRS** separates read operations (queries) from write operations (commands).

### Traditional Approach (No CQRS)

```
Client Request
      ↓
  Controller
      ↓
   Service ←→ Same Model ←→ Single Database
      ↓
   Response
```

Both reads and writes use the same:

- Data model
- Database tables
- Business logic layer

**Example (Traditional):**

```javascript
// Single model for both reads and writes
class ProjectService {
  // Write operation
  async createProject(tenantId, data) {
    return await this.projectRepo.create(tenantId, data);
  }

  // Read operation
  async getProject(tenantId, projectId) {
    return await this.projectRepo.findById(tenantId, projectId);
  }

  // Complex read operation
  async getProjectDashboard(tenantId, projectId) {
    const project = await this.projectRepo.findById(tenantId, projectId);
    const tasks = await this.taskRepo.findByProject(tenantId, projectId);
    const users = await this.userRepo.findByProject(tenantId, projectId);
    const activities = await this.activityRepo.findByProject(
      tenantId,
      projectId,
    );

    // Multiple queries, joins, aggregations...
    return {
      project,
      tasks,
      users,
      activities,
      stats: this.calculateStats(tasks),
    };
  }
}
```

**Problems with Traditional Approach:**

1. **Read queries become complex** — Lots of joins, aggregations
2. **Different optimization needs** — Reads need speed, writes need consistency
3. **Contention** — Read and write operations compete for the same resources
4. **Scaling challenges** — Can't scale reads and writes independently

## Command vs Query vs Event

Before looking at any CQRS code, it's worth putting these three words side by side, because the whole pattern is built on keeping them separate:

|             | Tense      | Purpose                                                     | Can it be rejected?                               | Has side effects?    |
| ----------- | ---------- | ----------------------------------------------------------- | ------------------------------------------------- | -------------------- |
| **Command** | Imperative | Expresses an _intent_ to change something ("CreateProject") | Yes — validation can refuse it                    | Yes, if accepted     |
| **Query**   | Present    | Asks for data ("GetProjectDashboard")                       | Usually just fails or returns empty               | No                   |
| **Event**   | Past       | States a _fact_ that already happened ("ProjectCreated")    | No — it already happened, it can't be un-happened | It's a record of one |

A Command is a request that might be turned down. An Event is the immutable record of what actually occurred once a Command was accepted. A Query never changes anything — it only looks. Keeping these three separate in your code (separate classes, separate handlers) is what makes CQRS readable: every operation in the system is unambiguously one of the three.

## The CQRS Approach

```
Write Side (Commands)              Read Side (Queries)
      ↓                                   ↓
Write Model ←→ Write DB           Read Model ←→ Read DB
      ↓                                   ↑
      └────────── Events ─────────────────┘
```

**Key Principles:**

1. **Separate Models:**
   - Write Model: Optimized for business logic and validation
   - Read Model: Optimized for querying and display

2. **Separate Databases (optional but common):**
   - Write DB: Normalized, ACID transactions
   - Read DB: Denormalized, optimized for queries

3. **Eventual Consistency:**
   - Changes in the write model propagate to the read model via events
   - The read model might be slightly behind the write model

## Maturity Levels of CQRS

CQRS is not all-or-nothing — it's a spectrum, and most real systems don't need to go all the way to the end of it:

- **Level 1 — Same database, same tables.** Only the code is split: separate `Command`/`CommandHandler` classes from `Query`/`QueryHandler` classes, purely for readability and single-responsibility. No infrastructure change at all.
- **Level 2 — Same database, different models.** Writes still go through normalized tables, but reads are served from dedicated views or denormalized tables inside the _same_ database (e.g. a SQL view like `project_dashboard_view`).
- **Level 3 — Separate databases.** The read side lives in its own database (or even a different database technology), kept in sync with the write side via events.
- **Level 4 — Event Sourcing on the write side.** Instead of storing current state, the write side stores the full sequence of events, and every read model is a projection built by replaying them. This is covered in the next chapter — it's an option, not a requirement of CQRS.

Most production systems stop at Level 2 or Level 3. Jumping straight to "two databases plus an event bus" (as the examples below will show) is useful to understand the full pattern, but don't treat it as the only valid way to "do CQRS" — plenty of systems get real benefit from Level 1 or 2 alone.

## CQRS Implementation Example

### Write Side (Commands)

```javascript
// Command - Represents intent to change state
class CreateProjectCommand {
  constructor(tenantId, userId, data) {
    this.tenantId = tenantId;
    this.userId = userId;
    this.name = data.name;
    this.description = data.description;
    this.budget = data.budget;
  }
}

class UpdateProjectStatusCommand {
  constructor(tenantId, projectId, userId, newStatus) {
    this.tenantId = tenantId;
    this.projectId = projectId;
    this.userId = userId;
    this.newStatus = newStatus;
  }
}

// Command Handler - Executes business logic
class CreateProjectCommandHandler {
  constructor(projectRepository, eventBus) {
    this.projectRepo = projectRepository;
    this.eventBus = eventBus;
  }

  async handle(command) {
    // 1. Validate business rules
    await this.validateProjectLimit(command.tenantId);

    // 2. Create aggregate
    const project = Project.create(
      command.tenantId,
      command.name,
      command.description,
      command.userId,
    );

    // 3. Save to write database
    await this.projectRepo.save(project);

    // 4. Publish events (for read model update)
    const events = project.getDomainEvents();
    for (const event of events) {
      await this.eventBus.publish(event);
    }

    return project.id;
  }

  async validateProjectLimit(tenantId) {
    const count = await this.projectRepo.countByTenant(tenantId);
    const tenant = await this.tenantRepo.findById(tenantId);

    if (count >= tenant.getProjectLimit()) {
      throw new ValidationError("Project limit reached");
    }
  }
}

// Write Model (Domain Model)
class Project {
  constructor(id, tenantId, name, description, createdBy) {
    this.id = id;
    this.tenantId = tenantId;
    this.name = name;
    this.description = description;
    this.status = "active";
    this.createdBy = createdBy;
    this.createdAt = new Date();
    this.domainEvents = [];
  }

  static create(tenantId, name, description, userId) {
    const project = new Project(uuid(), tenantId, name, description, userId);

    project.addDomainEvent(
      new ProjectCreatedEvent(tenantId, project.id, name, userId, new Date()),
    );

    return project;
  }

  updateStatus(newStatus, userId) {
    if (!this.canChangeStatus(newStatus)) {
      throw new ValidationError(`Cannot change status to ${newStatus}`);
    }

    const oldStatus = this.status;
    this.status = newStatus;

    this.addDomainEvent(
      new ProjectStatusChangedEvent(
        this.tenantId,
        this.id,
        oldStatus,
        newStatus,
        userId,
        new Date(),
      ),
    );
  }

  canChangeStatus(newStatus) {
    const validTransitions = {
      active: ["on_hold", "completed", "cancelled"],
      on_hold: ["active", "cancelled"],
      completed: ["archived"],
      cancelled: ["archived"],
      archived: [],
    };

    return validTransitions[this.status].includes(newStatus);
  }

  addDomainEvent(event) {
    this.domainEvents.push(event);
  }

  getDomainEvents() {
    return [...this.domainEvents];
  }
}

// Write Repository (saves to normalized database)
class ProjectWriteRepository {
  constructor(writeDb) {
    this.db = writeDb;
  }

  async save(project) {
    await this.db.query(
      `
      INSERT INTO projects (
        id, tenant_id, name, description, status, created_by, created_at
      ) VALUES ($1, $2, $3, $4, $5, $6, $7)
      ON CONFLICT (id) DO UPDATE SET
        name = $3,
        description = $4,
        status = $5,
        updated_at = NOW()
    `,
      [
        project.id,
        project.tenantId,
        project.name,
        project.description,
        project.status,
        project.createdBy,
        project.createdAt,
      ],
    );
  }
}
```

### Read Side (Queries)

```javascript
// Query - Represents request for data
class GetProjectDashboardQuery {
  constructor(tenantId, projectId) {
    this.tenantId = tenantId;
    this.projectId = projectId;
  }
}

class ListProjectsQuery {
  constructor(tenantId, filters) {
    this.tenantId = tenantId;
    this.filters = filters;
  }
}

// Query Handler - Returns data (no business logic)
class GetProjectDashboardQueryHandler {
  constructor(projectReadRepository) {
    this.projectReadRepo = projectReadRepository;
  }

  async handle(query) {
    // Simple query from denormalized read model
    return await this.projectReadRepo.getDashboard(
      query.tenantId,
      query.projectId,
    );
  }
}

// Read Model (denormalized, optimized for queries)
class ProjectDashboardReadModel {
  constructor(data) {
    // Everything needed for dashboard in one place
    this.projectId = data.projectId;
    this.projectName = data.projectName;
    this.projectStatus = data.projectStatus;
    this.createdBy = data.createdBy;
    this.createdByName = data.createdByName; // Denormalized!
    this.totalTasks = data.totalTasks;
    this.completedTasks = data.completedTasks;
    this.completionPercentage = data.completionPercentage;
    this.teamMembers = data.teamMembers; // Denormalized array
    this.recentActivities = data.recentActivities; // Denormalized
    this.budget = data.budget;
    this.actualSpend = data.actualSpend;
    this.lastUpdated = data.lastUpdated;
  }
}

// Read Repository (queries denormalized database)
class ProjectReadRepository {
  constructor(readDb) {
    this.db = readDb;
  }

  async getDashboard(tenantId, projectId) {
    // Single query from denormalized table - super fast!
    const result = await this.db.query(
      `
      SELECT * FROM project_dashboard_view
      WHERE tenant_id = $1 AND project_id = $2
    `,
      [tenantId, projectId],
    );

    if (!result.rows[0]) return null;

    return new ProjectDashboardReadModel(result.rows[0]);
  }

  async list(tenantId, filters) {
    // Optimized query with pre-calculated values
    const result = await this.db.query(
      `
      SELECT 
        project_id,
        project_name,
        project_status,
        total_tasks,
        completed_tasks,
        completion_percentage
      FROM project_list_view
      WHERE tenant_id = $1
        AND status = COALESCE($2, status)
      ORDER BY created_at DESC
    `,
      [tenantId, filters.status],
    );

    return result.rows;
  }
}

// Event Handler - Updates read model when events occur
class ProjectCreatedEventHandler {
  constructor(projectReadRepository) {
    this.projectReadRepo = projectReadRepository;
  }

  async handle(event) {
    // Update read model (denormalized)
    await this.projectReadRepo.createDashboardView({
      projectId: event.projectId,
      projectName: event.projectName,
      projectStatus: "active",
      tenantId: event.tenantId,
      createdBy: event.userId,
      createdByName: await this.getUserName(event.userId), // Denormalize
      totalTasks: 0,
      completedTasks: 0,
      completionPercentage: 0,
      teamMembers: [event.userId],
      recentActivities: [],
      lastUpdated: event.timestamp,
    });
  }

  async getUserName(userId) {
    // Fetch and denormalize user name
    const user = await this.userRepo.findById(userId);
    return user.name;
  }
}

class TaskCompletedEventHandler {
  constructor(projectReadRepository) {
    this.projectReadRepo = projectReadRepository;
  }

  async handle(event) {
    // Update read model with new stats
    await this.projectReadRepo.updateDashboardStats(
      event.tenantId,
      event.projectId,
      {
        completedTasks: "INCREMENT", // Increment count
        completionPercentage: "RECALCULATE",
        lastUpdated: event.timestamp,
      },
    );
  }
}
```

### Controller (uses both Command Bus and Query Bus)

```javascript
class ProjectController {
  constructor(commandBus, queryBus) {
    this.commandBus = commandBus;
    this.queryBus = queryBus;
  }

  // Write endpoint
  async createProject(req, res) {
    try {
      const command = new CreateProjectCommand(
        req.tenant.id,
        req.user.id,
        req.body,
      );

      const projectId = await this.commandBus.execute(command);

      res.status(201).json({
        projectId,
        message: "Project created successfully",
      });
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }

  // Read endpoint
  async getProjectDashboard(req, res) {
    try {
      const query = new GetProjectDashboardQuery(
        req.tenant.id,
        req.params.projectId,
      );

      const dashboard = await this.queryBus.execute(query);

      res.json(dashboard);
    } catch (error) {
      res.status(500).json({ error: error.message });
    }
  }
}
```

**What is a Command Bus / Query Bus, exactly?** It's a small dispatcher that takes a Command (or Query) object and routes it to whichever handler is registered for that object's type — usually just by matching the class name (`CreateProjectCommand` → `CreateProjectCommandHandler`). The controller never needs to know which handler class to call directly; it just hands the bus a command and gets a result back.

This indirection is worth the extra step for two reasons:

- **Decoupling.** Controllers depend only on the bus, not on every individual handler class — handlers can be added, removed, or replaced without touching the controllers.
- **Cross-cutting middleware in one place.** Logging, authorization checks, validation, or metrics can be applied once at the bus level (e.g. "log every command before dispatching it") instead of being repeated inside every single handler.

## How Events Actually Get From Write Side to Read Side

Every example above calls `eventBus.publish(event)` as if it were a simple, guaranteed operation — but the mechanics behind that call matter a lot in practice.

**Synchronous vs asynchronous delivery.** The event bus can call read-model handlers directly, in-process, before the write request even returns (simple, low latency, but couples the read model's availability to the write path). Or it can push events onto a message broker (Kafka, RabbitMQ, SQS, etc.) and let handlers consume them independently (more resilient and scalable, but the read model now lags behind by however long delivery takes).

**The dual-write problem.** Step 3 saves the aggregate to the write database; step 4 publishes events. What happens if step 3 succeeds but the process crashes before step 4 runs? The write happened, but the read model never finds out about it — silently going stale. This is a well-known failure mode, usually called the **dual-write problem**, and it's the reason production systems often reach for a **transactional outbox**: events are written to an "outbox" table in the _same_ database transaction as the aggregate, and a separate process reliably relays outbox rows to the event bus afterward. The example code above simplifies past this problem — worth knowing it exists before shipping something similar.

**Idempotency of event handlers.** Most message brokers only guarantee "at-least-once" delivery, meaning a handler can receive the same event twice (e.g. after a retry following a timeout). If `TaskCompletedEventHandler.handle()` blindly increments `completedTasks` again on a duplicate delivery, the read model's numbers become wrong. The fix is to make handlers **idempotent**: before applying an event, check whether it (or its version/sequence number) has already been applied to this read row, and skip it if so. This is the practical meaning behind the often glossed-over principle "replaying events should produce the same result" — it isn't just a nice property, it's a requirement for correctness once delivery isn't perfectly exactly-once.

## CQRS Database Design

### Write Database (Normalized)

```sql
-- Write side: Normalized for consistency
CREATE TABLE projects (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  status VARCHAR(50),
  created_by UUID NOT NULL,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

CREATE TABLE tasks (
  id UUID PRIMARY KEY,
  project_id UUID REFERENCES projects(id),
  title VARCHAR(255),
  status VARCHAR(50),
  assigned_to UUID,
  created_at TIMESTAMP
);

CREATE TABLE users (
  id UUID PRIMARY KEY,
  tenant_id UUID,
  name VARCHAR(255),
  email VARCHAR(255)
);
```

### Read Database (Denormalized)

```sql
-- Read side: Denormalized for speed
CREATE TABLE project_dashboard_view (
  project_id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  project_name VARCHAR(255),
  project_status VARCHAR(50),
  created_by UUID,
  created_by_name VARCHAR(255),  -- Denormalized!
  total_tasks INTEGER,
  completed_tasks INTEGER,
  completion_percentage INTEGER,
  team_members JSONB,  -- Array of users
  recent_activities JSONB,  -- Pre-calculated
  budget DECIMAL,
  actual_spend DECIMAL,
  last_updated TIMESTAMP,

  -- Indexes for fast queries
  INDEX idx_tenant_status (tenant_id, project_status)
);

CREATE TABLE project_list_view (
  project_id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  project_name VARCHAR(255),
  project_status VARCHAR(50),
  created_at TIMESTAMP,
  total_tasks INTEGER,
  completed_tasks INTEGER,
  completion_percentage INTEGER,

  INDEX idx_tenant_created (tenant_id, created_at DESC)
);
```

## Benefits of CQRS

**1. Performance Optimization**

```javascript
// Without CQRS: Complex query with multiple joins
async function getProjectDashboard(tenantId, projectId) {
  const result = await db.query(`
    SELECT 
      p.*,
      u.name as created_by_name,
      COUNT(t.id) as total_tasks,
      COUNT(t.id) FILTER (WHERE t.status = 'completed') as completed_tasks,
      json_agg(DISTINCT team.*) as team_members,
      json_agg(a.* ORDER BY a.created_at DESC) as activities
    FROM projects p
    LEFT JOIN users u ON p.created_by = u.id
    LEFT JOIN tasks t ON t.project_id = p.id
    LEFT JOIN project_members pm ON pm.project_id = p.id
    LEFT JOIN users team ON pm.user_id = team.id
    LEFT JOIN activities a ON a.project_id = p.id
    WHERE p.id = $1 AND p.tenant_id = $2
    GROUP BY p.id, u.name
  `);
  // Slow! Multiple joins, aggregations
}

// With CQRS: Simple query from pre-calculated view
async function getProjectDashboard(tenantId, projectId) {
  const result = await readDb.query(`
    SELECT * FROM project_dashboard_view
    WHERE project_id = $1 AND tenant_id = $2
  `);
  // Fast! Single table, no joins
}
```

**2. Independent Scaling**

```
Write Side:                    Read Side:
2 servers                      10 servers (read-heavy workload)
PostgreSQL (ACID)              PostgreSQL + Redis cache
Master database                Read replicas
```

**3. Different Database Technologies**

```
Write Side:                    Read Side:
PostgreSQL                     Elasticsearch (for search)
(ACID, consistency)            MongoDB (for flexible queries)
                              Redis (for fast lookups)
```

**4. Optimized for Use Case**

```javascript
// Write model: Rich with business logic
class Project {
  updateBudget(newBudget, userId) {
    if (!this.canUpdateBudget(userId)) {
      throw new UnauthorizedError();
    }
    if (newBudget < this.actualSpend) {
      throw new ValidationError("Budget cannot be less than actual spend");
    }
    // Complex business rules
  }
}

// Read model: Simple DTO for display
class ProjectListItemReadModel {
  constructor(data) {
    this.id = data.id;
    this.name = data.name;
    this.status = data.status;
    this.completionPercentage = data.completionPercentage;
    // Just data, no behavior
  }
}
```

**5. Multiple Read Models**

One write model can power multiple read models:

```
Write Model (Project)
        ↓ events
        ├──→ Dashboard Read Model (denormalized for dashboard)
        ├──→ List Read Model (optimized for lists)
        ├──→ Search Read Model (Elasticsearch for search)
        ├──→ Analytics Read Model (time-series for charts)
        └──→ Report Read Model (optimized for exports)
```

## When to Use CQRS

**✅ Use CQRS When:**

1. **Read and write have vastly different loads**
   - Example: 1000 reads per second, 10 writes per second
2. **Complex queries slow down writes**
   - Lots of joins, aggregations in read queries
3. **Different scaling requirements**
   - Need to scale reads independently
4. **Multiple views of same data**
   - Dashboard, lists, search, reports all need different formats
5. **High-performance requirements**
   - Sub-second response times critical
6. **Audit and analytics are important**
   - All changes captured as events

**Use Cases:**

- E-commerce platforms (complex product catalogs, simple writes)
- Social media (read-heavy, billions of views, fewer writes)
- Analytics dashboards (complex aggregations, occasional updates)
- Collaboration tools (complex views, moderate writes)

**❌ Don't Use CQRS When:**

1. **Simple CRUD operations**
   - Just basic create/read/update/delete
2. **Similar read and write complexity**
   - No performance benefit
3. **Small team unfamiliar with CQRS**
   - Adds complexity, learning curve
4. **Strong consistency required everywhere**
   - Eventual consistency not acceptable
5. **Starting an MVP**
   - Premature optimization

**Use Cases:**

- Simple blog
- Todo app
- Basic CRM with simple queries
- Internal tools
