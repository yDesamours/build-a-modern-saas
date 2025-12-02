## 2.2 Layered Architecture

Layered architecture is about **separation of concerns** - keeping different types of code in different places.

### The Classic Layers

```
┌─────────────────────────────────────┐
│     Presentation Layer              │  ← HTTP controllers, GraphQL resolvers
│     (Controllers, API)              │
└──────────────┬──────────────────────┘
               ↓
┌──────────────┴──────────────────────┐
│     Application Layer               │  ← Use cases, orchestration
│     (Services, Use Cases)           │
└──────────────┬──────────────────────┘
               ↓
┌──────────────┴──────────────────────┐
│     Domain Layer                    │  ← Business logic, entities
│     (Business Logic, Entities)      │
└──────────────┬──────────────────────┘
               ↓
┌──────────────┴──────────────────────┐
│     Data Access Layer               │  ← Databases, external APIs
│     (Repositories, DAOs)            │
└──────────────┬──────────────────────┘
               ↓
┌──────────────┴──────────────────────┐
│     Infrastructure Layer            │  ← Database, file system, email
│     (Database, External Services)   │
└─────────────────────────────────────┘
```

### Layer Responsibilities

#### Layer 1: Presentation Layer (Controllers)

**Responsibility:** Handle HTTP requests/responses, input validation, authentication

**What it does:**
- Parse incoming requests
- Validate input format (not business rules)
- Call application services
- Format responses
- Handle HTTP-specific concerns (status codes, headers)

**What it does NOT do:**
- Business logic
- Direct database access
- Complex computations

**Example (Node.js/Express):**

```javascript
// controllers/ProjectController.js
class ProjectController {
  constructor(projectService) {
    this.projectService = projectService;
  }
  
  async create(req, res) {
    try {
      // 1. Validate input (format, not business rules)
      const { error, value } = createProjectSchema.validate(req.body);
      if (error) {
        return res.status(400).json({ 
          error: 'Invalid input', 
          details: error.details 
        });
      }
      
      // 2. Extract context
      const tenantId = req.tenant.id;
      const userId = req.user.id;
      
      // 3. Call service (business logic layer)
      const project = await this.projectService.createProject(
        tenantId,
        userId,
        value
      );
      
      // 4. Format response
      res.status(201).json({
        data: project,
        message: 'Project created successfully'
      });
      
    } catch (error) {
      // 5. Handle errors
      if (error instanceof ValidationError) {
        return res.status(400).json({ error: error.message });
      }
      if (error instanceof UnauthorizedError) {
        return res.status(403).json({ error: error.message });
      }
      
      // Unknown error
      console.error('Error creating project:', error);
      res.status(500).json({ error: 'Internal server error' });
    }
  }
  
  async list(req, res) {
    try {
      const tenantId = req.tenant.id;
      const { page = 1, limit = 20, status, search } = req.query;
      
      const result = await this.projectService.listProjects(tenantId, {
        page: parseInt(page),
        limit: parseInt(limit),
        status,
        search
      });
      
      res.json({
        data: result.projects,
        pagination: {
          page: result.page,
          limit: result.limit,
          total: result.total,
          pages: Math.ceil(result.total / result.limit)
        }
      });
      
    } catch (error) {
      console.error('Error listing projects:', error);
      res.status(500).json({ error: 'Internal server error' });
    }
  }
}

module.exports = ProjectController;
```

#### Layer 2: Application Layer (Services/Use Cases)

**Responsibility:** Orchestrate business logic, coordinate between entities

**What it does:**
- Implement use cases
- Orchestrate operations across multiple entities
- Transaction management
- Call repositories
- Emit events
- Business rule validation

**What it does NOT do:**
- HTTP handling
- Direct database queries (use repositories)
- Complex business logic (delegate to domain entities)

**Example (Node.js):**

```javascript
// services/ProjectService.js
class ProjectService {
  constructor(projectRepository, userRepository, activityRepository, eventBus) {
    this.projectRepo = projectRepository;
    this.userRepo = userRepository;
    this.activityRepo = activityRepository;
    this.eventBus = eventBus;
  }
  
  async createProject(tenantId, userId, projectData) {
    // 1. Validate business rules
    const user = await this.userRepo.findById(tenantId, userId);
    if (!user.canCreateProject()) {
      throw new UnauthorizedError('User cannot create projects');
    }
    
    // Check plan limits
    const projectCount = await this.projectRepo.countByTenant(tenantId);
    const limit = user.tenant.getProjectLimit();
    if (projectCount >= limit) {
      throw new ValidationError(
        `Project limit reached. Upgrade to create more projects.`
      );
    }
    
    // 2. Create project (repository handles database)
    const project = await this.projectRepo.create(tenantId, {
      ...projectData,
      createdBy: userId,
      status: 'active'
    });
    
    // 3. Record activity
    await this.activityRepo.create(tenantId, {
      type: 'project_created',
      entityType: 'project',
      entityId: project.id,
      userId,
      metadata: {
        projectName: project.name
      }
    });
    
    // 4. Emit event (for notifications, analytics, etc.)
    await this.eventBus.emit('project.created', {
      tenantId,
      projectId: project.id,
      userId,
      timestamp: new Date()
    });
    
    return project;
  }
  
  async listProjects(tenantId, options) {
    // Business logic: Apply filters based on user role
    const filters = {
      tenantId,
      status: options.status,
      search: options.search
    };
    
    // Repository handles the actual query
    const projects = await this.projectRepo.findAll(filters, {
      page: options.page,
      limit: options.limit,
      orderBy: 'created_at',
      order: 'DESC'
    });
    
    const total = await this.projectRepo.count(filters);
    
    return {
      projects,
      total,
      page: options.page,
      limit: options.limit
    };
  }
  
  async updateProject(tenantId, projectId, userId, updates) {
    // 1. Check permissions
    const project = await this.projectRepo.findById(tenantId, projectId);
    if (!project) {
      throw new NotFoundError('Project not found');
    }
    
    const user = await this.userRepo.findById(tenantId, userId);
    if (!user.canEditProject(project)) {
      throw new UnauthorizedError('Cannot edit this project');
    }
    
    // 2. Business rule: Can't change status directly if project has pending tasks
    if (updates.status && updates.status === 'archived') {
      const hasPendingTasks = await project.hasPendingTasks();
      if (hasPendingTasks) {
        throw new ValidationError(
          'Cannot archive project with pending tasks'
        );
      }
    }
    
    // 3. Update project
    const updatedProject = await this.projectRepo.update(
      tenantId,
      projectId,
      updates
    );
    
    // 4. Record activity
    await this.activityRepo.create(tenantId, {
      type: 'project_updated',
      entityType: 'project',
      entityId: projectId,
      userId,
      metadata: { changes: updates }
    });
    
    // 5. Emit event
    await this.eventBus.emit('project.updated', {
      tenantId,
      projectId,
      userId,
      changes: updates
    });
    
    return updatedProject;
  }
}

module.exports = ProjectService;
```

#### Layer 3: Domain Layer (Business Entities)

**Responsibility:** Core business logic, business rules, domain entities

**What it does:**
- Encapsulate business rules
- Validate business constraints
- Domain calculations
- Domain events

**What it does NOT do:**
- Database access
- HTTP handling
- External service calls

**Example (Node.js):**

```javascript
// models/Project.js
class Project {
  constructor(data) {
    this.id = data.id;
    this.tenantId = data.tenantId;
    this.name = data.name;
    this.description = data.description;
    this.status = data.status;
    this.budget = data.budget;
    this.startDate = data.startDate;
    this.endDate = data.endDate;
    this.createdBy = data.createdBy;
    this.createdAt = data.createdAt;
    this.updatedAt = data.updatedAt;
  }
  
  // Business logic: Is project overdue?
  isOverdue() {
    if (this.status === 'completed' || this.status === 'archived') {
      return false;
    }
    return this.endDate && new Date() > this.endDate;
  }
  
  // Business logic: Can project be archived?
  canBeArchived() {
    return this.status === 'completed' || this.status === 'cancelled';
  }
  
  // Business logic: Calculate completion percentage
  calculateCompletion(tasks) {
    if (!tasks || tasks.length === 0) return 0;
    
    const completedTasks = tasks.filter(t => t.status === 'completed');
    return Math.round((completedTasks.length / tasks.length) * 100);
  }
  
  // Business logic: Is budget exceeded?
  isBudgetExceeded(actualSpend) {
    if (!this.budget) return false;
    return actualSpend > this.budget;
  }
  
  // Business rule: Validate project dates
  validateDates() {
    if (this.startDate && this.endDate) {
      if (this.endDate <= this.startDate) {
        throw new ValidationError('End date must be after start date');
      }
    }
  }
  
  // Business rule: Can user edit this project?
  canBeEditedBy(user) {
    // Business rules for editing
    if (this.status === 'archived') return false;
    if (user.role === 'admin') return true;
    if (user.id === this.createdBy) return true;
    return false;
  }
}

module.exports = Project;
```

#### Layer 4: Data Access Layer (Repositories)

**Responsibility:** Abstract database access, query construction

**What it does:**
- CRUD operations
- Complex queries
- Database-specific logic
- Data mapping (DB ↔ Domain objects)

**What it does NOT do:**
- Business logic
- Business rule validation
- Complex orchestration

**Example (Node.js with PostgreSQL):**

```javascript
// repositories/ProjectRepository.js
class ProjectRepository {
  constructor(db) {
    this.db = db;
  }
  
  async create(tenantId, projectData) {
    const result = await this.db.query(`
      INSERT INTO projects (
        id, tenant_id, name, description, status, budget,
        start_date, end_date, created_by, created_at, updated_at
      )
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, NOW(), NOW())
      RETURNING *
    `, [
      uuidv4(),
      tenantId,
      projectData.name,
      projectData.description,
      projectData.status,
      projectData.budget,
      projectData.startDate,
      projectData.endDate,
      projectData.createdBy
    ]);
    
    return new Project(result.rows[0]);
  }
  
  async findById(tenantId, projectId) {
    const result = await this.db.query(`
      SELECT * FROM projects
      WHERE id = $1 AND tenant_id = $2
    `, [projectId, tenantId]);
    
    if (result.rows.length === 0) {
      return null;
    }
    
    return new Project(result.rows[0]);
  }
  
  async findAll(filters, options = {}) {
    // Build query dynamically based on filters
    let query = `SELECT * FROM projects WHERE tenant_id = $1`;
    const params = [filters.tenantId];
    let paramIndex = 2;
    
    if (filters.status) {
      query += ` AND status = $${paramIndex}`;
      params.push(filters.status);
      paramIndex++;
    }
    
    if (filters.search) {
      query += ` AND (name ILIKE $${paramIndex} OR description ILIKE $${paramIndex})`;
      params.push(`%${filters.search}%`);
      paramIndex++;
    }
    
    // Ordering
    query += ` ORDER BY ${options.orderBy || 'created_at'} ${options.order || 'DESC'}`;
    
    // Pagination
    if (options.limit) {
      query += ` LIMIT $${paramIndex}`;
      params.push(options.limit);
      paramIndex++;
    }
    
    if (options.page && options.limit) {
      const offset = (options.page - 1) * options.limit;
      query += ` OFFSET $${paramIndex}`;
      params.push(offset);
    }
    
    const result = await this.db.query(query, params);
    return result.rows.map(row => new Project(row));
  }
  
  async count(filters) {
    let query = `SELECT COUNT(*) FROM projects WHERE tenant_id = $1`;
    const params = [filters.tenantId];
    let paramIndex = 2;
    
    if (filters.status) {
      query += ` AND status = $${paramIndex}`;
      params.push(filters.status);
      paramIndex++;
    }
    
    if (filters.search) {
      query += ` AND (name ILIKE $${paramIndex} OR description ILIKE $${paramIndex})`;
      params.push(`%${filters.search}%`);
      paramIndex++;
    }
    
    const result = await this.db.query(query, params);
    return parseInt(result.rows[0].count);
  }
  
  async update(tenantId, projectId, updates) {
    const fields = [];
    const params = [projectId, tenantId];
    let paramIndex = 3;
    
    Object.keys(updates).forEach(key => {
      fields.push(`${key} = ${paramIndex}`);
      params.push(updates[key]);
      paramIndex++;
    });
    
    fields.push(`updated_at = NOW()`);
    
    const query = `
      UPDATE projects
      SET ${fields.join(', ')}
      WHERE id = $1 AND tenant_id = $2
      RETURNING *
    `;
    
    const result = await this.db.query(query, params);
    return result.rows.length > 0 ? new Project(result.rows[0]) : null;
  }
  
  async delete(tenantId, projectId) {
    const result = await this.db.query(`
      DELETE FROM projects
      WHERE id = $1 AND tenant_id = $2
      RETURNING id
    `, [projectId, tenantId]);
    
    return result.rows.length > 0;
  }
  
  async countByTenant(tenantId) {
    const result = await this.db.query(`
      SELECT COUNT(*) FROM projects
      WHERE tenant_id = $1
    `, [tenantId]);
    
    return parseInt(result.rows[0].count);
  }
}

module.exports = ProjectRepository;
```

#### Layer 5: Infrastructure Layer

**Responsibility:** External services, file system, email, database connections

**What it does:**
- Database connection management
- Email service integration
- File storage (S3, local)
- External API clients
- Message queues
- Caching

**Example:**

```javascript
// infrastructure/email/EmailService.js
class EmailService {
  constructor(config) {
    this.transporter = nodemailer.createTransport(config);
  }
  
  async sendWelcomeEmail(user) {
    await this.transporter.sendMail({
      from: 'noreply@app.com',
      to: user.email,
      subject: 'Welcome to Our App!',
      html: this.renderWelcomeTemplate(user)
    });
  }
  
  async sendPasswordReset(user, token) {
    await this.transporter.sendMail({
      from: 'noreply@app.com',
      to: user.email,
      subject: 'Reset Your Password',
      html: this.renderResetTemplate(user, token)
    });
  }
  
  renderWelcomeTemplate(user) {
    return `
      <h1>Welcome, ${user.name}!</h1>
      <p>Thank you for signing up...</p>
    `;
  }
  
  renderResetTemplate(user, token) {
    return `
      <h1>Reset Your Password</h1>
      <p>Click here to reset: ${process.env.APP_URL}/reset/${token}</p>
    `;
  }
}

module.exports = EmailService;
```

---

### Benefits of Layered Architecture

**1. Separation of Concerns**
- Each layer has a single responsibility
- Easy to understand and maintain
- Changes in one layer don't affect others

**2. Testability**
- Can test each layer independently
- Mock dependencies easily

```javascript
// Testing service layer without database
describe('ProjectService', () => {
  it('should create project with valid data', async () => {
    // Mock repository
    const mockRepo = {
      create: jest.fn().mockResolvedValue(mockProject),
      countByTenant: jest.fn().mockResolvedValue(5)
    };
    
    const service = new ProjectService(mockRepo, ...);
    const result = await service.createProject(tenantId, userId, projectData);
    
    expect(mockRepo.create).toHaveBeenCalled();
    expect(result).toEqual(mockProject);
  });
});
```

**3. Reusability**
- Services can be reused by different controllers (REST API, GraphQL, CLI)
- Repositories can be reused by different services

**4. Flexibility**
- Easy to swap implementations (e.g., switch from PostgreSQL to MongoDB)
- Just change the repository layer

**5. Team Collaboration**
- Different developers can work on different layers
- Clear interfaces between layers

---

### Anti-Patterns to Avoid

#### ❌ Anti-Pattern 1: Anemic Domain Model

**Problem:** Domain objects have no behavior, just getters/setters

```javascript
// BAD: Anemic domain model
class Project {
  constructor(data) {
    this.id = data.id;
    this.name = data.name;
    this.status = data.status;
  }
  
  // Just getters and setters, no behavior
  getName() { return this.name; }
  setName(name) { this.name = name; }
  getStatus() { return this.status; }
  setStatus(status) { this.status = status; }
}

// All business logic ends up in service layer
class ProjectService {
  updateStatus(project, newStatus) {
    // Business logic shouldn't be here!
    if (project.getStatus() === 'archived') {
      throw new Error('Cannot update archived project');
    }
    if (newStatus === 'completed' && project.getTasks().some(t => t.status !== 'done')) {
      throw new Error('Cannot complete project with incomplete tasks');
    }
    project.setStatus(newStatus);
  }
}
```

**Solution:** Put business logic in domain objects

```javascript
// GOOD: Rich domain model
class Project {
  constructor(data) {
    this.id = data.id;
    this.name = data.name;
    this.status = data.status;
  }
  
  // Business logic lives in the domain
  canChangeStatus(newStatus) {
    if (this.status === 'archived') {
      return { allowed: false, reason: 'Cannot update archived project' };
    }
    return { allowed: true };
  }
  
  canBeCompleted(tasks) {
    const incompleteTasks = tasks.filter(t => t.status !== 'done');
    if (incompleteTasks.length > 0) {
      return { 
        allowed: false, 
        reason: `${incompleteTasks.length} tasks still incomplete` 
      };
    }
    return { allowed: true };
  }
  
  complete() {
    this.status = 'completed';
    this.completedAt = new Date();
  }
}

// Service orchestrates, domain enforces rules
class ProjectService {
  async completeProject(tenantId, projectId) {
    const project = await this.projectRepo.findById(tenantId, projectId);
    const tasks = await this.taskRepo.findByProject(tenantId, projectId);
    
    // Domain object decides if completion is allowed
    const check = project.canBeCompleted(tasks);
    if (!check.allowed) {
      throw new ValidationError(check.reason);
    }
    
    // Domain object handles the completion
    project.complete();
    
    await this.projectRepo.update(tenantId, projectId, project);
  }
}
```

#### ❌ Anti-Pattern 2: God Service

**Problem:** One service does everything

```javascript
// BAD: God service
class ApplicationService {
  async createProject(tenantId, userId, data) { ... }
  async updateProject(tenantId, projectId, data) { ... }
  async deleteProject(tenantId, projectId) { ... }
  async createTask(tenantId, projectId, data) { ... }
  async updateTask(tenantId, taskId, data) { ... }
  async assignTask(tenantId, taskId, userId) { ... }
  async createUser(tenantId, data) { ... }
  async updateUserRole(tenantId, userId, role) { ... }
  async sendNotification(tenantId, userId, message) { ... }
  async generateReport(tenantId, reportType) { ... }
  // ... 50 more methods
}
```

**Solution:** Separate services by domain

```javascript
// GOOD: Focused services
class ProjectService {
  async createProject(...) { ... }
  async updateProject(...) { ... }
  async deleteProject(...) { ... }
}

class TaskService {
  async createTask(...) { ... }
  async updateTask(...) { ... }
  async assignTask(...) { ... }
}

class UserService {
  async createUser(...) { ... }
  async updateUserRole(...) { ... }
}

class NotificationService {
  async send(...) { ... }
}

class ReportService {
  async generate(...) { ... }
}
```

#### ❌ Anti-Pattern 3: Layer Skipping

**Problem:** Controllers directly access repositories

```javascript
// BAD: Controller accessing repository directly
class ProjectController {
  async create(req, res) {
    // Skipping service layer!
    const project = await projectRepository.create(req.tenant.id, req.body);
    
    // Business logic in controller (wrong layer!)
    if (project.budget > 1000000) {
      await notificationRepository.create({
        type: 'high_budget_project',
        projectId: project.id
      });
    }
    
    res.json(project);
  }
}
```

**Solution:** Always go through the appropriate layer

```javascript
// GOOD: Proper layer flow
class ProjectController {
  async create(req, res) {
    // Controller → Service (correct!)
    const project = await projectService.createProject(
      req.tenant.id,
      req.user.id,
      req.body
    );
    
    res.json(project);
  }
}

class ProjectService {
  async createProject(tenantId, userId, data) {
    // Service → Repository (correct!)
    const project = await projectRepository.create(tenantId, data);
    
    // Business logic in service layer
    if (project.budget > 1000000) {
      await notificationService.notifyHighBudgetProject(tenantId, project);
    }
    
    return project;
  }
}
```

---
