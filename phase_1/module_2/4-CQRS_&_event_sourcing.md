## 2.4 CQRS and Event Sourcing

CQRS (Command Query Responsibility Segregation) and Event Sourcing are advanced architectural patterns that solve specific problems. They're NOT for every application, but when you need them, they're powerful.

### What is CQRS?

**CQRS** separates read operations (queries) from write operations (commands).

#### Traditional Approach (No CQRS)

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
      projectId
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

1. **Read queries become complex** - Lots of joins, aggregations
2. **Different optimization needs** - Reads need speed, writes need consistency
3. **Contention** - Read and write operations compete for same resources
4. **Scaling challenges** - Can't scale reads and writes independently

#### CQRS Approach

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
   - Changes in write model propagate to read model via events
   - Read model might be slightly behind write model

### CQRS Implementation Example

#### Write Side (Commands)

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
      command.userId
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
      new ProjectCreatedEvent(tenantId, project.id, name, userId, new Date())
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
        new Date()
      )
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
      ]
    );
  }
}
```

#### Read Side (Queries)

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
      query.projectId
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
      [tenantId, projectId]
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
      [tenantId, filters.status]
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
      }
    );
  }
}
```

#### Controller (uses both command and query)

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
        req.body
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
        req.params.projectId
      );

      const dashboard = await this.queryBus.execute(query);

      res.json(dashboard);
    } catch (error) {
      res.status(500).json({ error: error.message });
    }
  }
}
```

### CQRS Database Design

#### Write Database (Normalized)

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

#### Read Database (Denormalized)

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

### Benefits of CQRS

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

### When to Use CQRS

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

---

### What is Event Sourcing?

**Event Sourcing** stores all changes as a sequence of events, rather than just the current state.

#### Traditional State Storage

```
Database:
┌────────────────────────────────┐
│ projects table                 │
├────┬──────────┬─────────┬──────┤
│ id │   name   │ status  │budget│
├────┼──────────┼─────────┼──────┤
│ 1  │Website   │completed│50000 │  ← Current state only
└────┴──────────┴─────────┴──────┘

History lost! How did we get here?
- When was status changed?
- Who changed the budget?
- What was the previous name?
```

#### Event Sourcing Approach

```
Event Store:
┌───────────────────────────────────────────────────────┐
│ events table                                          │
├────┬───────────────────────┬──────────────┬───────────┤
│ id │       event_type      │    data      │timestamp  │
├────┼───────────────────────┼──────────────┼───────────┤
│ 1  │ProjectCreated         │{name:Website}│2024-01-01 │
│ 2  │BudgetSet              │{budget:40000}│2024-01-05 │
│ 3  │StatusChanged          │{from:active, │2024-01-15 │
│    │                       │ to:review}   │           │
│ 4  │BudgetIncreased        │{from:40000,  │2024-01-20 │
│    │                       │ to:50000}    │           │
│ 5  │StatusChanged          │{from:review, │2024-02-01 │
│    │                       │ to:completed}│           │
└────┴───────────────────────┴──────────────┴───────────┘

Complete history! Can replay to any point in time.
```

### Event Sourcing Implementation

#### 1. Define Events

```javascript
// Base Event
class DomainEvent {
  constructor(aggregateId, aggregateType, eventType, data, metadata = {}) {
    this.eventId = uuid();
    this.aggregateId = aggregateId;
    this.aggregateType = aggregateType;
    this.eventType = eventType;
    this.data = data;
    this.metadata = {
      ...metadata,
      timestamp: new Date(),
      version: 1,
    };
  }
}

// Specific Events
class ProjectCreatedEvent extends DomainEvent {
  constructor(projectId, tenantId, name, createdBy) {
    super(
      projectId,
      "Project",
      "ProjectCreated",
      {
        tenantId,
        name,
        createdBy,
      },
      { userId: createdBy }
    );
  }
}

class ProjectBudgetChangedEvent extends DomainEvent {
  constructor(projectId, oldBudget, newBudget, changedBy) {
    super(
      projectId,
      "Project",
      "ProjectBudgetChanged",
      {
        oldBudget,
        newBudget,
        changedBy,
      },
      { userId: changedBy }
    );
  }
}

class ProjectStatusChangedEvent extends DomainEvent {
  constructor(projectId, oldStatus, newStatus, changedBy) {
    super(
      projectId,
      "Project",
      "ProjectStatusChanged",
      {
        oldStatus,
        newStatus,
        changedBy,
      },
      { userId: changedBy }
    );
  }
}
```

#### 2. Aggregate that Produces Events

```javascript
class Project {
  constructor() {
    this.id = null;
    this.tenantId = null;
    this.name = null;
    this.status = null;
    this.budget = null;
    this.createdBy = null;
    this.version = 0;
    this.uncommittedEvents = [];
  }

  // Factory method - produces ProjectCreated event
  static create(projectId, tenantId, name, budget, userId) {
    const project = new Project();

    const event = new ProjectCreatedEvent(projectId, tenantId, name, userId);

    project.applyEvent(event);
    project.uncommittedEvents.push(event);

    return project;
  }

  // Command - produces BudgetChanged event
  changeBudget(newBudget, userId) {
    if (newBudget < 0) {
      throw new ValidationError("Budget cannot be negative");
    }

    const event = new ProjectBudgetChangedEvent(
      this.id,
      this.budget,
      newBudget,
      userId
    );

    this.applyEvent(event);
    this.uncommittedEvents.push(event);
  }

  // Command - produces StatusChanged event
  changeStatus(newStatus, userId) {
    if (!this.canChangeStatus(newStatus)) {
      throw new ValidationError(
        `Cannot change from ${this.status} to ${newStatus}`
      );
    }

    const event = new ProjectStatusChangedEvent(
      this.id,
      this.status,
      newStatus,
      userId
    );

    this.applyEvent(event);
    this.uncommittedEvents.push(event);
  }

  // Apply event to change state
  applyEvent(event) {
    switch (event.eventType) {
      case "ProjectCreated":
        this.id = event.aggregateId;
        this.tenantId = event.data.tenantId;
        this.name = event.data.name;
        this.status = "active";
        this.createdBy = event.data.createdBy;
        break;

      case "ProjectBudgetChanged":
        this.budget = event.data.newBudget;
        break;

      case "ProjectStatusChanged":
        this.status = event.data.newStatus;
        break;

      default:
        throw new Error(`Unknown event type: ${event.eventType}`);
    }

    this.version++;
  }

  // Reconstruct from events
  static fromEvents(events) {
    const project = new Project();

    for (const event of events) {
      project.applyEvent(event);
    }

    return project;
  }

  getUncommittedEvents() {
    return [...this.uncommittedEvents];
  }

  clearUncommittedEvents() {
    this.uncommittedEvents = [];
  }

  canChangeStatus(newStatus) {
    const validTransitions = {
      active: ["on_hold", "completed", "cancelled"],
      on_hold: ["active", "cancelled"],
      completed: ["archived"],
      cancelled: ["archived"],
      archived: [],
    };

    return validTransitions[this.status]?.includes(newStatus) || false;
  }
}
```

#### 3. Event Store

```javascript
class EventStore {
  constructor(db) {
    this.db = db;
  }

  // Save events
  async save(aggregateId, events, expectedVersion) {
    await this.db.transaction(async (trx) => {
      // Check for concurrency conflicts
      const currentVersion = await this.getCurrentVersion(trx, aggregateId);

      if (currentVersion !== expectedVersion) {
        throw new ConcurrencyError(
          `Expected version ${expectedVersion}, but current version is ${currentVersion}`
        );
      }

      // Insert events
      for (const event of events) {
        await trx.query(
          `
          INSERT INTO events (
            event_id,
            aggregate_id,
            aggregate_type,
            event_type,
            data,
            metadata,
            version,
            created_at
          ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
        `,
          [
            event.eventId,
            event.aggregateId,
            event.aggregateType,
            event.eventType,
            JSON.stringify(event.data),
            JSON.stringify(event.metadata),
            expectedVersion + 1,
            event.metadata.timestamp,
          ]
        );

        expectedVersion++;
      }
    });
  }

  // Load events for an aggregate
  async getEventsForAggregate(aggregateId) {
    const result = await this.db.query(
      `
      SELECT * FROM events
      WHERE aggregate_id = $1
      ORDER BY version ASC
    `,
      [aggregateId]
    );

    return result.rows.map((row) => this.deserializeEvent(row));
  }

  // Get all events (for projections)
  async getAllEvents(fromVersion = 0) {
    const result = await this.db.query(
      `
      SELECT * FROM events
      WHERE version > $1
      ORDER BY version ASC
    `,
      [fromVersion]
    );

    return result.rows.map((row) => this.deserializeEvent(row));
  }

  async getCurrentVersion(trx, aggregateId) {
    const result = await trx.query(
      `
      SELECT MAX(version) as version
      FROM events
      WHERE aggregate_id = $1
    `,
      [aggregateId]
    );

    return result.rows[0].version || 0;
  }

  deserializeEvent(row) {
    const EventClass = this.getEventClass(row.event_type);

    const event = Object.create(EventClass.prototype);
    event.eventId = row.event_id;
    event.aggregateId = row.aggregate_id;
    event.aggregateType = row.aggregate_type;
    event.eventType = row.event_type;
    event.data = JSON.parse(row.data);
    event.metadata = JSON.parse(row.metadata);

    return event;
  }

  getEventClass(eventType) {
    const eventClasses = {
      ProjectCreated: ProjectCreatedEvent,
      ProjectBudgetChanged: ProjectBudgetChangedEvent,
      ProjectStatusChanged: ProjectStatusChangedEvent,
    };

    return eventClasses[eventType] || DomainEvent;
  }
}
```

#### 4. Repository using Event Store

```javascript
class EventSourcedProjectRepository {
  constructor(eventStore) {
    this.eventStore = eventStore;
  }

  // Load aggregate by replaying events
  async findById(projectId) {
    const events = await this.eventStore.getEventsForAggregate(projectId);

    if (events.length === 0) {
      return null;
    }

    // Reconstruct aggregate from events
    return Project.fromEvents(events);
  }

  // Save aggregate by storing new events
  async save(project) {
    const events = project.getUncommittedEvents();

    if (events.length === 0) {
      return; // No changes
    }

    const expectedVersion = project.version - events.length;

    await this.eventStore.save(project.id, events, expectedVersion);

    project.clearUncommittedEvents();
  }
}
```

#### 5. Usage Example

```javascript
class ProjectService {
  constructor(projectRepository, eventStore) {
    this.projectRepo = projectRepository;
    this.eventStore = eventStore;
  }

  async createProject(tenantId, userId, projectData) {
    // Create new project (produces ProjectCreated event)
    const project = Project.create(
      uuid(),
      tenantId,
      projectData.name,
      projectData.budget,
      userId
    );

    // Save (stores events)
    await this.projectRepo.save(project);

    return project.id;
  }

  async changeBudget(projectId, newBudget, userId) {
    // Load project (by replaying events)
    const project = await this.projectRepo.findById(projectId);

    if (!project) {
      throw new NotFoundError("Project not found");
    }

    // Execute command (produces event)
    project.changeBudget(newBudget, userId);

    // Save (stores new events)
    await this.projectRepo.save(project);
  }

  async changeStatus(projectId, newStatus, userId) {
    const project = await this.projectRepo.findById(projectId);

    if (!project) {
      throw new NotFoundError("Project not found");
    }

    project.changeStatus(newStatus, userId);
    await this.projectRepo.save(project);
  }

  // Time travel: get project state at specific point in time
  async getProjectAtTime(projectId, timestamp) {
    const allEvents = await this.eventStore.getEventsForAggregate(projectId);
    
    // Filter events up to the timestamp
    const eventsUntilTime = allEvents.filter(
      event => event.metadata.timestamp <= timestamp
    );

    if (eventsUntilTime.length === 0) {
      return null;
    }

    return Project.fromEvents(eventsUntilTime);
  }
}
```

---

## Advanced Event Sourcing Patterns

### 6. Projections (Read Models)

Projections build optimized read models from events for efficient querying.

```javascript
class ProjectProjection {
  constructor(db) {
    this.db = db;
  }

  // Project events into read model
  async project(event) {
    switch (event.eventType) {
      case "ProjectCreated":
        await this.handleProjectCreated(event);
        break;
      case "ProjectBudgetChanged":
        await this.handleBudgetChanged(event);
        break;
      case "ProjectStatusChanged":
        await this.handleStatusChanged(event);
        break;
    }
  }

  async handleProjectCreated(event) {
    await this.db.query(
      `
      INSERT INTO project_read_model (
        id, tenant_id, name, status, budget, created_by, created_at
      ) VALUES ($1, $2, $3, $4, $5, $6, $7)
    `,
      [
        event.aggregateId,
        event.data.tenantId,
        event.data.name,
        "active",
        0,
        event.data.createdBy,
        event.metadata.timestamp,
      ]
    );
  }

  async handleBudgetChanged(event) {
    await this.db.query(
      `
      UPDATE project_read_model
      SET budget = $1, updated_at = $2
      WHERE id = $3
    `,
      [
        event.data.newBudget,
        event.metadata.timestamp,
        event.aggregateId,
      ]
    );
  }

  async handleStatusChanged(event) {
    await this.db.query(
      `
      UPDATE project_read_model
      SET status = $1, updated_at = $2
      WHERE id = $3
    `,
      [
        event.data.newStatus,
        event.metadata.timestamp,
        event.aggregateId,
      ]
    );
  }

  // Query methods
  async findByTenant(tenantId) {
    const result = await this.db.query(
      `SELECT * FROM project_read_model WHERE tenant_id = $1`,
      [tenantId]
    );
    return result.rows;
  }

  async findActiveProjects() {
    const result = await this.db.query(
      `SELECT * FROM project_read_model WHERE status = 'active'`
    );
    return result.rows;
  }
}
```

### 7. Event Processor for Multiple Projections

```javascript
class EventProcessor {
  constructor(eventStore, projections) {
    this.eventStore = eventStore;
    this.projections = projections;
    this.lastProcessedVersion = 0;
  }

  async start() {
    // Initial catch-up
    await this.catchUp();

    // Poll for new events
    this.interval = setInterval(() => this.processNewEvents(), 1000);
  }

  async catchUp() {
    const events = await this.eventStore.getAllEvents(
      this.lastProcessedVersion
    );

    for (const event of events) {
      await this.processEvent(event);
      this.lastProcessedVersion = event.metadata.version;
    }
  }

  async processNewEvents() {
    const events = await this.eventStore.getAllEvents(
      this.lastProcessedVersion
    );

    for (const event of events) {
      await this.processEvent(event);
      this.lastProcessedVersion = event.metadata.version;
    }
  }

  async processEvent(event) {
    // Send event to all projections
    for (const projection of this.projections) {
      try {
        await projection.project(event);
      } catch (error) {
        console.error(`Error in projection: ${error.message}`);
        // Handle projection errors (retry, dead letter queue, etc.)
      }
    }
  }

  stop() {
    if (this.interval) {
      clearInterval(this.interval);
    }
  }
}
```

### 8. Snapshots for Performance

For aggregates with many events, snapshots improve load performance.

```javascript
class SnapshotStore {
  constructor(db) {
    this.db = db;
  }

  async saveSnapshot(aggregateId, snapshot, version) {
    await this.db.query(
      `
      INSERT INTO snapshots (aggregate_id, data, version, created_at)
      VALUES ($1, $2, $3, $4)
      ON CONFLICT (aggregate_id) DO UPDATE
      SET data = $2, version = $3, created_at = $4
    `,
      [aggregateId, JSON.stringify(snapshot), version, new Date()]
    );
  }

  async getSnapshot(aggregateId) {
    const result = await this.db.query(
      `SELECT * FROM snapshots WHERE aggregate_id = $1`,
      [aggregateId]
    );

    if (result.rows.length === 0) {
      return null;
    }

    return {
      data: JSON.parse(result.rows[0].data),
      version: result.rows[0].version,
    };
  }
}

// Enhanced Repository with Snapshots
class SnapshotProjectRepository extends EventSourcedProjectRepository {
  constructor(eventStore, snapshotStore) {
    super(eventStore);
    this.snapshotStore = snapshotStore;
    this.snapshotFrequency = 50; // Snapshot every 50 events
  }

  async findById(projectId) {
    // Try to load from snapshot
    const snapshot = await this.snapshotStore.getSnapshot(projectId);

    let project;
    let fromVersion = 0;

    if (snapshot) {
      // Reconstruct from snapshot
      project = new Project();
      Object.assign(project, snapshot.data);
      fromVersion = snapshot.version;
    } else {
      project = new Project();
    }

    // Load events since snapshot
    const events = await this.eventStore.getEventsForAggregate(projectId);
    const newEvents = events.filter(e => e.metadata.version > fromVersion);

    // Apply new events
    for (const event of newEvents) {
      project.applyEvent(event);
    }

    return project.id ? project : null;
  }

  async save(project) {
    await super.save(project);

    // Create snapshot if needed
    if (project.version % this.snapshotFrequency === 0) {
      const snapshot = {
        id: project.id,
        tenantId: project.tenantId,
        name: project.name,
        status: project.status,
        budget: project.budget,
        createdBy: project.createdBy,
        version: project.version,
      };

      await this.snapshotStore.saveSnapshot(
        project.id,
        snapshot,
        project.version
      );
    }
  }
}
```

### 9. Event Upcasting (Versioning)

Handle event schema evolution over time.

```javascript
class EventUpcaster {
  upcast(event) {
    const upcasters = {
      ProjectCreated: this.upcastProjectCreated,
      ProjectBudgetChanged: this.upcastProjectBudgetChanged,
    };

    const upcaster = upcasters[event.eventType];
    return upcaster ? upcaster(event) : event;
  }

  upcastProjectCreated(event) {
    // V1 -> V2: Add default deadline field
    if (event.metadata.version === 1) {
      return {
        ...event,
        data: {
          ...event.data,
          deadline: null, // New field in V2
        },
        metadata: {
          ...event.metadata,
          version: 2,
          upcastedFrom: 1,
        },
      };
    }

    return event;
  }

  upcastProjectBudgetChanged(event) {
    // V1 -> V2: Add reason field
    if (event.metadata.version === 1) {
      return {
        ...event,
        data: {
          ...event.data,
          reason: "Not specified", // New required field in V2
        },
        metadata: {
          ...event.metadata,
          version: 2,
          upcastedFrom: 1,
        },
      };
    }

    return event;
  }
}

// Use in EventStore
class EventStoreWithUpcasting extends EventStore {
  constructor(db, upcaster) {
    super(db);
    this.upcaster = upcaster;
  }

  deserializeEvent(row) {
    const event = super.deserializeEvent(row);
    return this.upcaster.upcast(event);
  }
}
```

---

## Benefits and Trade-offs

### Benefits

**Complete Audit Trail**: Every change is recorded with who, what, when, and why.

**Time Travel**: Query system state at any point in history.

**Event Replay**: Rebuild state, fix bugs retroactively, create new projections from historical data.

**Business Intelligence**: Analyze patterns, user behavior, and trends over time.

**Debugging**: Reproduce bugs by replaying exact sequence of events.

**Temporal Queries**: "Show me all projects that were active on January 15th."

**Integration**: Events can be published to other systems for real-time integration.

### Trade-offs

**Complexity**: More complex than CRUD, requires careful event design.

**Storage**: Events accumulate over time (mitigated by snapshots and archiving).

**Query Performance**: Rebuilding state from events can be slow (solved with projections).

**Eventual Consistency**: Read models may lag behind write model.

**Event Schema Evolution**: Changing event structure requires upcasting or versioning strategies.

### When to Use Event Sourcing

**Good fit**:
- Domain with complex business rules and important history
- Audit requirements (financial, healthcare, legal)
- Need for temporal queries
- Complex workflows requiring replay capability
- High-value data where every change matters

**Poor fit**:
- Simple CRUD applications
- Systems where only current state matters
- Performance-critical real-time systems with simple logic
- Small projects with limited resources

---

## Database Schema

```sql
-- Events table
CREATE TABLE events (
  event_id UUID PRIMARY KEY,
  aggregate_id UUID NOT NULL,
  aggregate_type VARCHAR(255) NOT NULL,
  event_type VARCHAR(255) NOT NULL,
  data JSONB NOT NULL,
  metadata JSONB NOT NULL,
  version INTEGER NOT NULL,
  created_at TIMESTAMP NOT NULL,
  
  UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_events_aggregate ON events(aggregate_id);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);

-- Snapshots table
CREATE TABLE snapshots (
  aggregate_id UUID PRIMARY KEY,
  data JSONB NOT NULL,
  version INTEGER NOT NULL,
  created_at TIMESTAMP NOT NULL
);

-- Read model (projection)
CREATE TABLE project_read_model (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  name VARCHAR(255) NOT NULL,
  status VARCHAR(50) NOT NULL,
  budget DECIMAL(12, 2),
  created_by UUID NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP
);

CREATE INDEX idx_projects_tenant ON project_read_model(tenant_id);
CREATE INDEX idx_projects_status ON project_read_model(status);
```

---

## Key Principles

1. **Events are immutable**: Never modify or delete events, only append new ones.

2. **Events are facts**: Past tense, describe what happened: "OrderPlaced", not "PlaceOrder".

3. **Single source of truth**: The event store is the source of truth, projections are derived.

4. **Domain events only**: Only store domain-significant events, not every property change.

5. **Complete information**: Events should contain all information needed to understand what happened.

6. **Idempotent projections**: Replaying events should produce same result.

