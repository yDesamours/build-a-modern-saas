## Repository Pattern ⭐ (THE MOST CRITICAL PATTERN FOR SAAS)

### What is the Repository Pattern?

**The Repository Pattern** mediates between the domain and data mapping layers, acting like an in-memory collection of domain objects.

Think of a repository as a **collection interface**:

```typescript
// It feels like working with an in-memory collection
const repository = new ProjectRepository();
await repository.save(project);           // Like: collection.add(project)
const project = await repository.findById(id);  // Like: collection.get(id)
const all = await repository.findAll();   // Like: collection.values()
await repository.delete(id);              // Like: collection.delete(id)

// But it's actually backed by a database
```

### Why Repository Pattern is CRITICAL for SaaS

#### Problem Without Repository Pattern

```typescript
// ❌ BAD: Business logic mixed with database code
class ProjectService {
  constructor(private db: Database) {}
  
  async createProject(tenantId: string, data: any) {
    // Database-specific code in business layer
    const result = await this.db.query(`
      INSERT INTO projects (id, tenant_id, name, status, created_at)
      VALUES ($1, $2, $3, $4, NOW())
      RETURNING *
    `, [uuid(), tenantId, data.name, 'active']);
    
    return result.rows[0];
  }
  
  async getProject(tenantId: string, projectId: string) {
    const result = await this.db.query(`
      SELECT * FROM projects 
      WHERE id = $1 AND tenant_id = $2
    `, [projectId, tenantId]);
    
    return result.rows[0];
  }
}

// Problems:
// 1. Can't test without database
// 2. Can't switch databases easily
// 3. SQL scattered throughout codebase
// 4. Easy to forget tenant_id filtering
// 5. Business logic coupled to database
```

#### Solution With Repository Pattern

```typescript
// ✅ GOOD: Clean separation of concerns
class ProjectService {
  constructor(private projectRepo: IProjectRepository) {}
  
  async createProject(tenantId: string, data: CreateProjectDTO) {
    // Pure business logic
    const project = new Project(
      uuid(),
      tenantId,
      data.name,
      'active',
      new Date()
    );
    
    // Repository handles persistence
    return this.projectRepo.save(project);
  }
  
  async getProject(tenantId: string, projectId: string) {
    return this.projectRepo.findById(tenantId, projectId);
  }
}

// Benefits:
// 1. Easy to test (mock repository)
// 2. Can switch databases by swapping implementation
// 3. All database logic centralized
// 4. tenant_id filtering guaranteed in repository
// 5. Business logic independent of persistence
```

---

## Complete Repository Pattern Implementation

### Step 1: Define Domain Entity

The domain entity represents your business object with behavior.

```typescript
// domain/entities/Project.ts
export class Project {
  constructor(
    public readonly id: string,
    public readonly tenantId: string,
    public name: string,
    public description: string,
    public status: 'active' | 'completed' | 'archived',
    public budget: number | null,
    public startDate: Date | null,
    public endDate: Date | null,
    public readonly createdBy: string,
    public readonly createdAt: Date,
    public updatedAt: Date
  ) {
    this.validate();
  }
  
  // Domain validation
  private validate(): void {
    if (!this.name || this.name.trim().length === 0) {
      throw new Error('Project name is required');
    }
    
    if (this.name.length > 255) {
      throw new Error('Project name too long');
    }
    
    if (this.startDate && this.endDate && this.endDate < this.startDate) {
      throw new Error('End date must be after start date');
    }
  }
  
  // Business logic methods
  canBeArchived(): boolean {
    return this.status === 'completed';
  }
  
  archive(): void {
    if (!this.canBeArchived()) {
      throw new Error('Only completed projects can be archived');
    }
    this.status = 'archived';
    this.updatedAt = new Date();
  }
  
  complete(): void {
    if (this.status === 'archived') {
      throw new Error('Cannot complete an archived project');
    }
    this.status = 'completed';
    this.updatedAt = new Date();
  }
  
  updateDetails(name: string, description: string): void {
    if (this.status === 'archived') {
      throw new Error('Cannot update archived project');
    }
    
    this.name = name;
    this.description = description;
    this.updatedAt = new Date();
    this.validate();
  }
  
  updateBudget(budget: number | null): void {
    if (budget !== null && budget < 0) {
      throw new Error('Budget cannot be negative');
    }
    this.budget = budget;
    this.updatedAt = new Date();
  }
  
  isOverdue(): boolean {
    if (!this.endDate || this.status !== 'active') {
      return false;
    }
    return new Date() > this.endDate;
  }
}
```

### Step 2: Define Repository Interface

The interface defines what operations are available, without specifying HOW they're implemented.

```typescript
// domain/repositories/IProjectRepository.ts
import { Project } from '../entities/Project';

export interface IProjectRepository {
  // Basic CRUD operations
  findById(tenantId: string, projectId: string): Promise<Project | null>;
  findAll(tenantId: string): Promise<Project[]>;
  save(project: Project): Promise<Project>;
  delete(tenantId: string, projectId: string): Promise<boolean>;
  
  // Query operations
  findByStatus(
    tenantId: string, 
    status: 'active' | 'completed' | 'archived'
  ): Promise<Project[]>;
  
  findByCreator(tenantId: string, userId: string): Promise<Project[]>;
  
  findByDateRange(
    tenantId: string, 
    startDate: Date, 
    endDate: Date
  ): Promise<Project[]>;
  
  search(
    tenantId: string, 
    searchTerm: string
  ): Promise<Project[]>;
  
  findOverdue(tenantId: string): Promise<Project[]>;
  
  // Pagination
  findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: {
      status?: string;
      search?: string;
      createdBy?: string;
    }
  ): Promise<{
    projects: Project[];
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  }>;
  
  // Aggregations
  count(tenantId: string): Promise<number>;
  countByStatus(tenantId: string, status: string): Promise<number>;
  
  // Batch operations
  saveMany(projects: Project[]): Promise<Project[]>;
  deleteMany(tenantId: string, projectIds: string[]): Promise<number>;
  
  // Existence checks
  exists(tenantId: string, projectId: string): Promise<boolean>;
  existsByName(tenantId: string, name: string): Promise<boolean>;
}
```

### Step 3: Implement PostgreSQL Repository

```typescript
// infrastructure/repositories/PostgresProjectRepository.ts
import { Pool, PoolClient } from 'pg';
import { IProjectRepository } from '../../domain/repositories/IProjectRepository';
import { Project } from '../../domain/entities/Project';

export class PostgresProjectRepository implements IProjectRepository {
  constructor(private db: Pool) {}
  
  async findById(tenantId: string, projectId: string): Promise<Project | null> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE id = $1 AND tenant_id = $2`,
      [projectId, tenantId]
    );
    
    if (result.rows.length === 0) {
      return null;
    }
    
    return this.toDomain(result.rows[0]);
  }
  
  async findAll(tenantId: string): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 
       ORDER BY created_at DESC`,
      [tenantId]
    );
    
    return result.rows.map(row => this.toDomain(row));
  }
  
  async save(project: Project): Promise<Project> {
    // Check if exists
    const exists = await this.exists(project.tenantId, project.id);
    
    if (exists) {
      return this.update(project);
    } else {
      return this.insert(project);
    }
  }
  
  private async insert(project: Project): Promise<Project> {
    const result = await this.db.query(
      `INSERT INTO projects (
        id, tenant_id, name, description, status, budget,
        start_date, end_date, created_by, created_at, updated_at
      )
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
      RETURNING *`,
      [
        project.id,
        project.tenantId,
        project.name,
        project.description,
        project.status,
        project.budget,
        project.startDate,
        project.endDate,
        project.createdBy,
        project.createdAt,
        project.updatedAt
      ]
    );
    
    return this.toDomain(result.rows[0]);
  }
  
  private async update(project: Project): Promise<Project> {
    const result = await this.db.query(
      `UPDATE projects 
       SET name = $1, 
           description = $2, 
           status = $3, 
           budget = $4,
           start_date = $5,
           end_date = $6,
           updated_at = $7
       WHERE id = $8 AND tenant_id = $9
       RETURNING *`,
      [
        project.name,
        project.description,
        project.status,
        project.budget,
        project.startDate,
        project.endDate,
        project.updatedAt,
        project.id,
        project.tenantId
      ]
    );
    
    if (result.rows.length === 0) {
      throw new Error('Project not found or update failed');
    }
    
    return this.toDomain(result.rows[0]);
  }
  
  async delete(tenantId: string, projectId: string): Promise<boolean> {
    const result = await this.db.query(
      `DELETE FROM projects 
       WHERE id = $1 AND tenant_id = $2`,
      [projectId, tenantId]
    );
    
    return result.rowCount > 0;
  }
  
  async findByStatus(tenantId: string, status: string): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 AND status = $2
       ORDER BY created_at DESC`,
      [tenantId, status]
    );
    
    return result.rows.map(row => this.toDomain(row));
  }
  
  async findByCreator(tenantId: string, userId: string): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 AND created_by = $2
       ORDER BY created_at DESC`,
      [tenantId, userId]
    );
    
    return result.rows.map(row => this.toDomain(row));
  }
  
  async findByDateRange(
    tenantId: string, 
    startDate: Date, 
    endDate: Date
  ): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 
         AND created_at >= $2 
         AND created_at <= $3
       ORDER BY created_at DESC`,
      [tenantId, startDate, endDate]
    );
    
    return result.rows.map(row => this.toDomain(row));
  }
  
  async search(tenantId: string, searchTerm: string): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 
         AND (
           name ILIKE $2 
           OR description ILIKE $2
         )
       ORDER BY created_at DESC`,
      [tenantId, `%${searchTerm}%`]
    );
    
    return result.rows.map(row => this.toDomain(row));
  }
  
  async findOverdue(tenantId: string): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 
         AND status = 'active'
         AND end_date < NOW()
       ORDER BY end_date ASC`,
      [tenantId]
    );
    
    return result.rows.map(row => this.toDomain(row));
  }
  
  async findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: {
      status?: string;
      search?: string;
      createdBy?: string;
    }
  ): Promise<{
    projects: Project[];
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  }> {
    // Build WHERE clause dynamically
    const conditions: string[] = ['tenant_id = $1'];
    const params: any[] = [tenantId];
    let paramIndex = 2;
    
    if (filters?.status) {
      conditions.push(`status = $${paramIndex}`);
      params.push(filters.status);
      paramIndex++;
    }
    
    if (filters?.search) {
      conditions.push(`(name ILIKE $${paramIndex} OR description ILIKE $${paramIndex})`);
      params.push(`%${filters.search}%`);
      paramIndex++;
    }
    
    if (filters?.createdBy) {
      conditions.push(`created_by = $${paramIndex}`);
      params.push(filters.createdBy);
      paramIndex++;
    }
    
    const whereClause = conditions.join(' AND ');
    
    // Get total count
    const countResult = await this.db.query(
      `SELECT COUNT(*) as count FROM projects WHERE ${whereClause}`,
      params
    );
    const total = parseInt(countResult.rows[0].count);
    
    // Get paginated results
    const offset = (page - 1) * limit;
    const dataResult = await this.db.query(
      `SELECT * FROM projects 
       WHERE ${whereClause}
       ORDER BY created_at DESC
       LIMIT $${paramIndex} OFFSET $${paramIndex + 1}`,
      [...params, limit, offset]
    );
    
    const projects = dataResult.rows.map(row => this.toDomain(row));
    
    return {
      projects,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit)
    };
  }
  
  async count(tenantId: string): Promise<number> {
    const result = await this.db.query(
      `SELECT COUNT(*) as count FROM projects WHERE tenant_id = $1`,
      [tenantId]
    );
    
    return parseInt(result.rows[0].count);
  }
  
  async countByStatus(tenantId: string, status: string): Promise<number> {
    const result = await this.db.query(
      `SELECT COUNT(*) as count FROM projects 
       WHERE tenant_id = $1 AND status = $2`,
      [tenantId, status]
    );
    
    return parseInt(result.rows[0].count);
  }
  
  async saveMany(projects: Project[]): Promise<Project[]> {
    // Use transaction for batch insert
    const client = await this.db.connect();
    try {
      await client.query('BEGIN');
      
      const savedProjects: Project[] = [];
      for (const project of projects) {
        const result = await client.query(
          `INSERT INTO projects (
            id, tenant_id, name, description, status, budget,
            start_date, end_date, created_by, created_at, updated_at
          )
          VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
          RETURNING *`,
          [
            project.id,
            project.tenantId,
            project.name,
            project.description,
            project.status,
            project.budget,
            project.startDate,
            project.endDate,
            project.createdBy,
            project.createdAt,
            project.updatedAt
          ]
        );
        savedProjects.push(this.toDomain(result.rows[0]));
      }
      
      await client.query('COMMIT');
      return savedProjects;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  async deleteMany(tenantId: string, projectIds: string[]): Promise<number> {
    const result = await this.db.query(
      `DELETE FROM projects 
       WHERE tenant_id = $1 AND id = ANY($2)`,
      [tenantId, projectIds]
    );
    
    return result.rowCount;
  }
  
  async exists(tenantId: string, projectId: string): Promise<boolean> {
    const result = await this.db.query(
      `SELECT EXISTS(
        SELECT 1 FROM projects 
        WHERE id = $1 AND tenant_id = $2
      ) as exists`,
      [projectId, tenantId]
    );
    
    return result.rows[0].exists;
  }
  
  async existsByName(tenantId: string, name: string): Promise<boolean> {
    const result = await this.db.query(
      `SELECT EXISTS(
        SELECT 1 FROM projects 
        WHERE tenant_id = $1 AND name = $2
      ) as exists`,
      [tenantId, name]
    );
    
    return result.rows[0].exists;
  }
  
  // Map database row to domain entity
  private toDomain(row: any): Project {
    return new Project(
      row.id,
      row.tenant_id,
      row.name,
      row.description,
      row.status,
      row.budget,
      row.start_date,
      row.end_date,
      row.created_by,
      row.created_at,
      row.updated_at
    );
  }
}
```

### Step 4: Use Repository in Service Layer

```typescript
// application/services/ProjectService.ts
import { IProjectRepository } from '../../domain/repositories/IProjectRepository';
import { Project } from '../../domain/entities/Project';
import { IEventBus } from '../events/IEventBus';

export class ProjectService {
  constructor(
    private projectRepository: IProjectRepository,
    private eventBus: IEventBus
  ) {}
  
  async createProject(
    tenantId: string,
    userId: string,
    data: {
      name: string;
      description: string;
      budget?: number;
      startDate?: Date;
      endDate?: Date;
    }
  ): Promise<Project> {
    // Check for duplicate name
    const nameExists = await this.projectRepository.existsByName(
      tenantId, 
      data.name
    );
    
    if (nameExists) {
      throw new Error('Project with this name already exists');
    }
    
    // Create domain entity
    const project = new Project(
      this.generateId(),
      tenantId,
      data.name,
      data.description,
      'active',
      data.budget || null,
      data.startDate || null,
      data.endDate || null,
      userId,
      new Date(),
      new Date()
    );
    
    // Save via repository
    const savedProject = await this.projectRepository.save(project);
    
    // Emit domain event
    await this.eventBus.emit('project.created', {
      tenantId,
      projectId: savedProject.id,
      userId,
      timestamp: new Date()
    });
    
    return savedProject;
  }
  
  async getProject(tenantId: string, projectId: string): Promise<Project> {
    const project = await this.projectRepository.findById(tenantId, projectId);
    
    if (!project) {
      throw new Error('Project not found');
    }
    
    return project;
  }
  
  async updateProject(
    tenantId: string,
    projectId: string,
    updates: {
      name?: string;
      description?: string;
      budget?: number;
    }
  ): Promise<Project> {
    // Load project
    const project = await this.getProject(tenantId, projectId);
    
    // Apply updates (business logic in domain)
    if (updates.name && updates.description !== undefined) {
      project.updateDetails(updates.name, updates.description);
    }
    
    if (updates.budget !== undefined) {
      project.updateBudget(updates.budget);
    }
    
    // Save via repository
    const updatedProject = await this.projectRepository.save(project);
    
    // Emit event
    await this.eventBus.emit('project.updated', {
      tenantId,
      projectId: updatedProject.id,
      updates,
      timestamp: new Date()
    });
    
    return updatedProject;
  }
  
  async completeProject(tenantId: string, projectId: string): Promise<Project> {
    const project = await this.getProject(tenantId, projectId);
    
    // Business logic in domain
    project.complete();
    
    // Save via repository
    const completedProject = await this.projectRepository.save(project);
    
    await this.eventBus.emit('project.completed', {
      tenantId,
      projectId: completedProject.id,
      timestamp: new Date()
    });
    
    return completedProject;
  }
  
  async archiveProject(tenantId: string, projectId: string): Promise<Project> {
    const project = await this.getProject(tenantId, projectId);
    
    // Business logic in domain
    project.archive();
    
    // Save via repository
    const archivedProject = await this.projectRepository.save(project);
    
    await this.eventBus.emit('project.archived', {
      tenantId,
      projectId: archivedProject.id,
      timestamp: new Date()
    });
    
    return archivedProject;
  }
  
  async listProjects(
    tenantId: string,
    options: {
      page?: number;
      limit?: number;
      status?: string;
      search?: string;
      createdBy?: string;
    } = {}
  ): Promise<{
    projects: Project[];
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  }> {
    const page = options.page || 1;
    const limit = options.limit || 20;
    
    return this.projectRepository.findWithPagination(
      tenantId,
      page,
      limit,
      {
        status: options.status,
        search: options.search,
        createdBy: options.createdBy
      }
    );
  }
  
  async searchProjects(
    tenantId: string,
    searchTerm: string
  ): Promise<Project[]> {
    return this.projectRepository.search(tenantId, searchTerm);
  }
  
  async getOverdueProjects(tenantId: string): Promise<Project[]> {
    return this.projectRepository.findOverdue(tenantId);
  }
  
  async deleteProject(tenantId: string, projectId: string): Promise<void> {
    const deleted = await this.projectRepository.delete(tenantId, projectId);
    
    if (!deleted) {
      throw new Error('Project not found or already deleted');
    }
    
    await this.eventBus.emit('project.deleted', {
      tenantId,
      projectId,
      timestamp: new Date()
    });
  }
  
  private generateId(): string {
    // Use uuid or similar
    return crypto.randomUUID();
  }
}
```

### Step 5: Easy Testing with Mock Repository

```typescript
// tests/mocks/MockProjectRepository.ts
export class MockProjectRepository implements IProjectRepository {
  private projects: Map<string, Project> = new Map();
  
  async findById(tenantId: string, projectId: string): Promise<Project | null> {
    const project = this.projects.get(projectId);
    
    if (!project || project.tenantId !== tenantId) {
      return null;
    }
    
    return project;
  }
  
  async findAll(tenantId: string): Promise<Project[]> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId)
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
  }
  
  async save(project: Project): Promise<Project> {
    this.projects.set(project.id, project);
    return project;
  }
  
  async delete(tenantId: string, projectId: string): Promise<boolean> {
    const project = this.projects.get(projectId);
    
    if (!project || project.tenantId !== tenantId) {
      return false;
    }
    
    this.projects.delete(projectId);
    return true;
  }
  
  async findByStatus(tenantId: string, status: string): Promise<Project[]> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId && p.status === status);
  }
  
  async findByCreator(tenantId: string, userId: string): Promise<Project[]> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId && p.createdBy === userId);
  }
  
  async findByDateRange(
    tenantId: string,
    startDate: Date,
    endDate: Date
  ): Promise<Project[]> {
    return Array.from(this.projects.values())
      .filter(p => 
        p.tenantId === tenantId &&
        p.createdAt >= startDate &&
        p.createdAt <= endDate
      );
  }
  
  async search(tenantId: string, searchTerm: string): Promise<Project[]> {
    const term = searchTerm.toLowerCase();
    return Array.from(this.projects.values())
      .filter(p =>
        p.tenantId === tenantId &&
        (p.name.toLowerCase().includes(term) ||
         p.description.toLowerCase().includes(term))
      );
  }
  
  async findOverdue(tenantId: string): Promise<Project[]> {
    const now = new Date();
    return Array.from(this.projects.values())
      .filter(p =>
        p.tenantId === tenantId &&
        p.status === 'active' &&
        p.endDate &&
        p.endDate < now
      );
  }
  
  async findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: any
  ): Promise<any> {
    let filtered = Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId);
    
    if (filters?.status) {
      filtered = filtered.filter(p => p.status === filters.status);
    }
    
    if (filters?.search) {
      const term = filters.search.toLowerCase();
      filtered = filtered.filter(p =>
        p.name.toLowerCase().includes(term) ||
        p.description.toLowerCase().includes(term)
      );
    }
    
    if (filters?.createdBy) {
      filtered = filtered.filter(p => p.createdBy === filters.createdBy);
    }
    
    const total = filtered.length;
    const start = (page - 1) * limit;
    const projects = filtered.slice(start, start + limit);
    
    return {
      projects,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit)
    };
  }
  
  async count(tenantId: string): Promise<number> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId)
      .length;
  }
  
  async countByStatus(tenantId: string, status: string): Promise<number> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId && p.status === status)
      .length;
  }
  
  async saveMany(projects: Project[]): Promise<Project[]> {
    projects.forEach(p => this.projects.set(p.id, p));
    return projects;
  }
  
  async deleteMany(tenantId: string, projectIds: string[]): Promise<number> {
    let deleted = 0;
    for (const id of projectIds) {
      if (await this.delete(tenantId, id)) {
        deleted++;
      }
    }
    return deleted;
  }
  
  async exists(tenantId: string, projectId: string): Promise<boolean> {
    const project = this.projects.get(projectId);
    return project !== undefined && project.tenantId === tenantId;
  }
  
  async existsByName(tenantId: string, name: string): Promise<boolean> {
    return Array.from(this.projects.values())
      .some(p => p.tenantId === tenantId && p.name === name);
  }
  
  // Test helper methods
  clear(): void {
    this.projects.clear();
  }
  
  seed(projects: Project[]): void {
    projects.forEach(p => this.projects.set(p.id, p));
  }
}

// Usage in tests
describe('ProjectService', () => {
  let projectService: ProjectService;
  let mockRepository: MockProjectRepository;
  let mockEventBus: MockEventBus;
  
  beforeEach(() => {
    mockRepository = new MockProjectRepository();
    mockEventBus = new MockEventBus();
    projectService = new ProjectService(mockRepository, mockEventBus);
  });
  
  afterEach(() => {
    mockRepository.clear();
  });
  
  it('should create a project', async () => {
    const tenantId = 'tenant-123';
    const userId = 'user-456';
    
    const project = await projectService
# Module 2.5: Repository Pattern - Testing & Advanced Topics

**Continuation from Step 5**

---

## Step 5: Testing with Mock Repository (Complete)

### Creating a Mock Repository

The beauty of the Repository pattern is that you can easily create in-memory implementations for testing.

```typescript
// tests/mocks/MockProjectRepository.ts
import { IProjectRepository } from '../../domain/repositories/IProjectRepository';
import { Project } from '../../domain/entities/Project';

export class MockProjectRepository implements IProjectRepository {
  private projects: Map<string, Project> = new Map();
  
  // Basic CRUD
  async findById(tenantId: string, projectId: string): Promise<Project | null> {
    const project = this.projects.get(projectId);
    
    // Enforce tenant isolation even in tests
    if (!project || project.tenantId !== tenantId) {
      return null;
    }
    
    return project;
  }
  
  async findAll(tenantId: string): Promise<Project[]> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId)
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
  }
  
  async save(project: Project): Promise<Project> {
    // Simulate saving
    this.projects.set(project.id, project);
    return project;
  }
  
  async delete(tenantId: string, projectId: string): Promise<boolean> {
    const project = this.projects.get(projectId);
    
    if (!project || project.tenantId !== tenantId) {
      return false;
    }
    
    this.projects.delete(projectId);
    return true;
  }
  
  // Query methods
  async findByStatus(tenantId: string, status: string): Promise<Project[]> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId && p.status === status)
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
  }
  
  async findByCreator(tenantId: string, userId: string): Promise<Project[]> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId && p.createdBy === userId)
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
  }
  
  async findByDateRange(
    tenantId: string,
    startDate: Date,
    endDate: Date
  ): Promise<Project[]> {
    return Array.from(this.projects.values())
      .filter(p =>
        p.tenantId === tenantId &&
        p.createdAt >= startDate &&
        p.createdAt <= endDate
      )
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
  }
  
  async search(tenantId: string, searchTerm: string): Promise<Project[]> {
    const term = searchTerm.toLowerCase();
    return Array.from(this.projects.values())
      .filter(p =>
        p.tenantId === tenantId &&
        (p.name.toLowerCase().includes(term) ||
         p.description.toLowerCase().includes(term))
      )
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
  }
  
  async findOverdue(tenantId: string): Promise<Project[]> {
    const now = new Date();
    return Array.from(this.projects.values())
      .filter(p =>
        p.tenantId === tenantId &&
        p.status === 'active' &&
        p.endDate !== null &&
        p.endDate < now
      )
      .sort((a, b) => a.endDate!.getTime() - b.endDate!.getTime());
  }
  
  async findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: {
      status?: string;
      search?: string;
      createdBy?: string;
    }
  ): Promise<{
    projects: Project[];
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  }> {
    let filtered = Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId);
    
    // Apply filters
    if (filters?.status) {
      filtered = filtered.filter(p => p.status === filters.status);
    }
    
    if (filters?.search) {
      const term = filters.search.toLowerCase();
      filtered = filtered.filter(p =>
        p.name.toLowerCase().includes(term) ||
        p.description.toLowerCase().includes(term)
      );
    }
    
    if (filters?.createdBy) {
      filtered = filtered.filter(p => p.createdBy === filters.createdBy);
    }
    
    // Sort by creation date
    filtered.sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
    
    // Calculate pagination
    const total = filtered.length;
    const start = (page - 1) * limit;
    const projects = filtered.slice(start, start + limit);
    
    return {
      projects,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit)
    };
  }
  
  // Aggregations
  async count(tenantId: string): Promise<number> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId)
      .length;
  }
  
  async countByStatus(tenantId: string, status: string): Promise<number> {
    return Array.from(this.projects.values())
      .filter(p => p.tenantId === tenantId && p.status === status)
      .length;
  }
  
  // Batch operations
  async saveMany(projects: Project[]): Promise<Project[]> {
    projects.forEach(p => this.projects.set(p.id, p));
    return projects;
  }
  
  async deleteMany(tenantId: string, projectIds: string[]): Promise<number> {
    let deleted = 0;
    
    for (const id of projectIds) {
      if (await this.delete(tenantId, id)) {
        deleted++;
      }
    }
    
    return deleted;
  }
  
  // Existence checks
  async exists(tenantId: string, projectId: string): Promise<boolean> {
    const project = this.projects.get(projectId);
    return project !== undefined && project.tenantId === tenantId;
  }
  
  async existsByName(tenantId: string, name: string): Promise<boolean> {
    return Array.from(this.projects.values())
      .some(p => p.tenantId === tenantId && p.name === name);
  }
  
  // Test helper methods
  clear(): void {
    this.projects.clear();
  }
  
  seed(projects: Project[]): void {
    projects.forEach(p => this.projects.set(p.id, p));
  }
  
  getAll(): Project[] {
    return Array.from(this.projects.values());
  }
}
```

### Writing Tests with Mock Repository

```typescript
// tests/services/ProjectService.test.ts
import { ProjectService } from '../../application/services/ProjectService';
import { MockProjectRepository } from '../mocks/MockProjectRepository';
import { MockEventBus } from '../mocks/MockEventBus';
import { Project } from '../../domain/entities/Project';

describe('ProjectService', () => {
  let projectService: ProjectService;
  let mockRepository: MockProjectRepository;
  let mockEventBus: MockEventBus;
  
  const tenantId = 'tenant-123';
  const userId = 'user-456';
  
  beforeEach(() => {
    mockRepository = new MockProjectRepository();
    mockEventBus = new MockEventBus();
    projectService = new ProjectService(mockRepository, mockEventBus);
  });
  
  afterEach(() => {
    mockRepository.clear();
    mockEventBus.clear();
  });
  
  describe('createProject', () => {
    it('should create a project successfully', async () => {
      const projectData = {
        name: 'Test Project',
        description: 'Test Description',
        budget: 10000
      };
      
      const project = await projectService.createProject(
        tenantId,
        userId,
        projectData
      );
      
      expect(project).toBeDefined();
      expect(project.name).toBe(projectData.name);
      expect(project.description).toBe(projectData.description);
      expect(project.budget).toBe(projectData.budget);
      expect(project.tenantId).toBe(tenantId);
      expect(project.createdBy).toBe(userId);
      expect(project.status).toBe('active');
      
      // Verify it was saved to repository
      const saved = await mockRepository.findById(tenantId, project.id);
      expect(saved).toBeDefined();
      expect(saved?.id).toBe(project.id);
      
      // Verify event was emitted
      expect(mockEventBus.getEvents('project.created')).toHaveLength(1);
    });
    
    it('should throw error if project name already exists', async () => {
      const projectData = {
        name: 'Duplicate Project',
        description: 'Test'
      };
      
      // Create first project
      await projectService.createProject(tenantId, userId, projectData);
      
      // Try to create duplicate
      await expect(
        projectService.createProject(tenantId, userId, projectData)
      ).rejects.toThrow('Project with this name already exists');
    });
    
    it('should enforce tenant isolation', async () => {
      const projectData = {
        name: 'Tenant A Project',
        description: 'Test'
      };
      
      const project = await projectService.createProject(
        'tenant-a',
        userId,
        projectData
      );
      
      // Try to access from different tenant
      const retrieved = await mockRepository.findById('tenant-b', project.id);
      expect(retrieved).toBeNull();
    });
  });
  
  describe('updateProject', () => {
    it('should update project successfully', async () => {
      // Create project
      const project = await projectService.createProject(
        tenantId,
        userId,
        {
          name: 'Original Name',
          description: 'Original Description',
          budget: 5000
        }
      );
      
      // Update project
      const updates = {
        name: 'Updated Name',
        description: 'Updated Description',
        budget: 7500
      };
      
      const updated = await projectService.updateProject(
        tenantId,
        project.id,
        updates
      );
      
      expect(updated.name).toBe(updates.name);
      expect(updated.description).toBe(updates.description);
      expect(updated.budget).toBe(updates.budget);
      expect(updated.updatedAt.getTime()).toBeGreaterThan(
        project.updatedAt.getTime()
      );
      
      // Verify event was emitted
      const updateEvents = mockEventBus.getEvents('project.updated');
      expect(updateEvents).toHaveLength(1);
      expect(updateEvents[0].data.projectId).toBe(project.id);
    });
    
    it('should throw error if project not found', async () => {
      await expect(
        projectService.updateProject(tenantId, 'non-existent-id', {
          name: 'New Name'
        })
      ).rejects.toThrow('Project not found');
    });
  });
  
  describe('completeProject', () => {
    it('should complete an active project', async () => {
      const project = await projectService.createProject(
        tenantId,
        userId,
        {
          name: 'Active Project',
          description: 'Test'
        }
      );
      
      const completed = await projectService.completeProject(
        tenantId,
        project.id
      );
      
      expect(completed.status).toBe('completed');
      
      // Verify event
      const events = mockEventBus.getEvents('project.completed');
      expect(events).toHaveLength(1);
    });
    
    it('should not complete an archived project', async () => {
      const project = await projectService.createProject(
        tenantId,
        userId,
        { name: 'Test', description: 'Test' }
      );
      
      // Complete then archive
      await projectService.completeProject(tenantId, project.id);
      await projectService.archiveProject(tenantId, project.id);
      
      // Try to complete again
      await expect(
        projectService.completeProject(tenantId, project.id)
      ).rejects.toThrow('Cannot complete an archived project');
    });
  });
  
  describe('archiveProject', () => {
    it('should archive a completed project', async () => {
      const project = await projectService.createProject(
        tenantId,
        userId,
        { name: 'Test', description: 'Test' }
      );
      
      await projectService.completeProject(tenantId, project.id);
      const archived = await projectService.archiveProject(tenantId, project.id);
      
      expect(archived.status).toBe('archived');
    });
    
    it('should not archive an active project', async () => {
      const project = await projectService.createProject(
        tenantId,
        userId,
        { name: 'Test', description: 'Test' }
      );
      
      await expect(
        projectService.archiveProject(tenantId, project.id)
      ).rejects.toThrow('Only completed projects can be archived');
    });
  });
  
  describe('listProjects', () => {
    beforeEach(async () => {
      // Seed test data
      const projects = [
        { name: 'Project 1', description: 'Test 1', status: 'active' },
        { name: 'Project 2', description: 'Test 2', status: 'active' },
        { name: 'Project 3', description: 'Test 3', status: 'completed' },
        { name: 'Project 4', description: 'Test 4', status: 'active' },
        { name: 'Project 5', description: 'Test 5', status: 'archived' },
      ];
      
      for (const data of projects) {
        await projectService.createProject(tenantId, userId, data);
      }
    });
    
    it('should list all projects with pagination', async () => {
      const result = await projectService.listProjects(tenantId, {
        page: 1,
        limit: 3
      });
      
      expect(result.projects).toHaveLength(3);
      expect(result.total).toBe(5);
      expect(result.page).toBe(1);
      expect(result.limit).toBe(3);
      expect(result.totalPages).toBe(2);
    });
    
    it('should filter by status', async () => {
      const result = await projectService.listProjects(tenantId, {
        status: 'active'
      });
      
      expect(result.projects).toHaveLength(3);
      expect(result.projects.every(p => p.status === 'active')).toBe(true);
    });
    
    it('should search by name', async () => {
      const result = await projectService.listProjects(tenantId, {
        search: 'Project 2'
      });
      
      expect(result.projects).toHaveLength(1);
      expect(result.projects[0].name).toBe('Project 2');
    });
  });
  
  describe('searchProjects', () => {
    beforeEach(async () => {
      await projectService.createProject(tenantId, userId, {
        name: 'Web Development',
        description: 'Building a website'
      });
      
      await projectService.createProject(tenantId, userId, {
        name: 'Mobile App',
        description: 'Building a mobile application'
      });
      
      await projectService.createProject(tenantId, userId, {
        name: 'API Development',
        description: 'Creating REST APIs'
      });
    });
    
    it('should search by name', async () => {
      const results = await projectService.searchProjects(tenantId, 'Web');
      
      expect(results).toHaveLength(1);
      expect(results[0].name).toBe('Web Development');
    });
    
    it('should search by description', async () => {
      const results = await projectService.searchProjects(tenantId, 'mobile');
      
      expect(results).toHaveLength(1);
      expect(results[0].name).toBe('Mobile App');
    });
    
    it('should be case insensitive', async () => {
      const results = await projectService.searchProjects(tenantId, 'BUILDING');
      
      expect(results).toHaveLength(2);
    });
  });
  
  describe('getOverdueProjects', () => {
    it('should return overdue active projects', async () => {
      const yesterday = new Date();
      yesterday.setDate(yesterday.getDate() - 1);
      
      const tomorrow = new Date();
      tomorrow.setDate(tomorrow.getDate() + 1);
      
      // Create overdue project
      await projectService.createProject(tenantId, userId, {
        name: 'Overdue Project',
        description: 'Test',
        endDate: yesterday
      });
      
      // Create on-time project
      await projectService.createProject(tenantId, userId, {
        name: 'On Time Project',
        description: 'Test',
        endDate: tomorrow
      });
      
      const overdue = await projectService.getOverdueProjects(tenantId);
      
      expect(overdue).toHaveLength(1);
      expect(overdue[0].name).toBe('Overdue Project');
    });
    
    it('should not include completed projects', async () => {
      const yesterday = new Date();
      yesterday.setDate(yesterday.getDate() - 1);
      
      const project = await projectService.createProject(tenantId, userId, {
        name: 'Completed Overdue',
        description: 'Test',
        endDate: yesterday
      });
      
      await projectService.completeProject(tenantId, project.id);
      
      const overdue = await projectService.getOverdueProjects(tenantId);
      
      expect(overdue).toHaveLength(0);
    });
  });
  
  describe('deleteProject', () => {
    it('should delete project successfully', async () => {
      const project = await projectService.createProject(
        tenantId,
        userId,
        { name: 'Test', description: 'Test' }
      );
      
      await projectService.deleteProject(tenantId, project.id);
      
      const deleted = await mockRepository.findById(tenantId, project.id);
      expect(deleted).toBeNull();
      
      // Verify event
      const events = mockEventBus.getEvents('project.deleted');
      expect(events).toHaveLength(1);
    });
    
    it('should throw error if project not found', async () => {
      await expect(
        projectService.deleteProject(tenantId, 'non-existent-id')
      ).rejects.toThrow('Project not found or already deleted');
    });
  });
  
  describe('tenant isolation', () => {
    it('should not allow access to other tenant projects', async () => {
      const tenant1 = 'tenant-1';
      const tenant2 = 'tenant-2';
      
      // Create project for tenant 1
      const project1 = await projectService.createProject(
        tenant1,
        userId,
        { name: 'Tenant 1 Project', description: 'Test' }
      );
      
      // Try to access from tenant 2
      await expect(
        projectService.getProject(tenant2, project1.id)
      ).rejects.toThrow('Project not found');
    });
    
    it('should only list projects for current tenant', async () => {
      const tenant1 = 'tenant-1';
      const tenant2 = 'tenant-2';
      
      // Create projects for both tenants
      await projectService.createProject(
        tenant1,
        userId,
        { name: 'T1 Project 1', description: 'Test' }
      );
      
      await projectService.createProject(
        tenant1,
        userId,
        { name: 'T1 Project 2', description: 'Test' }
      );
      
      await projectService.createProject(
        tenant2,
        userId,
        { name: 'T2 Project 1', description: 'Test' }
      );
      
      // List for tenant 1
      const tenant1Projects = await projectService.listProjects(tenant1);
      expect(tenant1Projects.total).toBe(2);
      
      // List for tenant 2
      const tenant2Projects = await projectService.listProjects(tenant2);
      expect(tenant2Projects.total).toBe(1);
    });
  });
});
```

---

## Implementation in Java (Spring Boot)

### Domain Entity

```java
// domain/entities/Project.java
package com.example.saas.domain.entities;

import java.time.LocalDateTime;
import java.util.UUID;

public class Project {
    private final UUID id;
    private final UUID tenantId;
    private String name;
    private String description;
    private ProjectStatus status;
    private Double budget;
    private LocalDateTime startDate;
    private LocalDateTime endDate;
    private final UUID createdBy;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    public enum ProjectStatus {
        ACTIVE, COMPLETED, ARCHIVED
    }
    
    public Project(
        UUID id,
        UUID tenantId,
        String name,
        String description,
        ProjectStatus status,
        Double budget,
        LocalDateTime startDate,
        LocalDateTime endDate,
        UUID createdBy,
        LocalDateTime createdAt,
        LocalDateTime updatedAt
    ) {
        this.id = id;
        this.tenantId = tenantId;
        this.name = name;
        this.description = description;
        this.status = status;
        this.budget = budget;
        this.startDate = startDate;
        this.endDate = endDate;
        this.createdBy = createdBy;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
        
        validate();
    }
    
    private void validate() {
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("Project name is required");
        }
        
        if (name.length() > 255) {
            throw new IllegalArgumentException("Project name too long");
        }
        
        if (startDate != null && endDate != null && endDate.isBefore(startDate)) {
            throw new IllegalArgumentException("End date must be after start date");
        }
    }
    
    public boolean canBeArchived() {
        return status == ProjectStatus.COMPLETED;
    }
    
    public void archive() {
        if (!canBeArchived()) {
            throw new IllegalStateException("Only completed projects can be archived");
        }
        this.status = ProjectStatus.ARCHIVED;
        this.updatedAt = LocalDateTime.now();
    }
    
    public void complete() {
        if (status == ProjectStatus.ARCHIVED) {
            throw new IllegalStateException("Cannot complete an archived project");
        }
        this.status = ProjectStatus.COMPLETED;
        this.updatedAt = LocalDateTime.now();
    }
    
    public void updateDetails(String name, String description) {
        if (status == ProjectStatus.ARCHIVED) {
            throw new IllegalStateException("Cannot update archived project");
        }
        
        this.name = name;
        this.description = description;
        this.updatedAt = LocalDateTime.now();
        validate();
    }
    
    public void updateBudget(Double budget) {
        if (budget != null && budget < 0) {
            throw new IllegalArgumentException("Budget cannot be negative");
        }
        this.budget = budget;
        this.updatedAt = LocalDateTime.now();
    }
    
    public boolean isOverdue() {
        if (endDate == null || status != ProjectStatus.ACTIVE) {
            return false;
        }
        return LocalDateTime.now().isAfter(endDate);
    }
    
    // Getters
    public UUID getId() { return id; }
    public UUID getTenantId() { return tenantId; }
    public String getName() { return name; }
    public String getDescription() { return description; }
    public ProjectStatus getStatus() { return status; }
    public Double getBudget() { return budget; }
    public LocalDateTime getStartDate() { return startDate; }
    public LocalDateTime getEndDate() { return endDate; }
    public UUID getCreatedBy() { return createdBy; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

### Repository Interface

```java
// domain/repositories/IProjectRepository.java
package com.example.saas.domain.repositories;

import com.example.saas.domain.entities.Project;
import com.example.saas.domain.entities.Project.ProjectStatus;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface IProjectRepository {
    // Basic CRUD
    Optional<Project> findById(UUID tenantId, UUID projectId);
    List<Project> findAll(UUID tenantId);
    Project save(Project project);
    boolean delete(UUID tenantId, UUID projectId);
    
    // Query methods
    List<Project> findByStatus(UUID tenantId, ProjectStatus status);
    List<Project> findByCreator(UUID tenantId, UUID userId);
    List<Project> findByDateRange(UUID tenantId, LocalDateTime startDate, LocalDateTime endDate);
    List<Project> search(UUID tenantId, String searchTerm);
    List<Project> findOverdue(UUID tenantId);
    
    // Pagination
    PaginatedResult<Project> findWithPagination(
        UUID tenantId,
        int page,
        int limit,
        ProjectFilters filters
    );
    
    // Aggregations
    long count(UUID tenantId);
    long countByStatus(UUID tenantId, ProjectStatus status);
    
    // Batch operations
    List<Project> saveMany(List<Project> projects);
    int deleteMany(UUID tenantId, List<UUID> projectIds);
    
    // Existence checks
    boolean exists(UUID tenantId, UUID projectId);
    boolean existsByName(UUID tenantId, String name);
}

// Supporting classes
class ProjectFilters {
    private ProjectStatus status;
    private String search;
    private UUID createdBy;
    
    // Getters and setters
    public ProjectStatus getStatus() { return status; }
    public void setStatus(ProjectStatus status) { this.status = status; }
    public String getSearch() { return search; }
    public void setSearch(String search) { this.search = search; }
    public UUID getCreatedBy() { return createdBy; }
    public void setCreatedBy(UUID createdBy) { this.createdBy = createdBy; }
}

class PaginatedResult<T> {
    private final List<T> items;
    private final long total;
    private final int page;
    private final int limit;
    private final int totalPages;
    
    public PaginatedResult(List<T> items, long total, int page, int limit) {
        this.items = items;
        this.total = total;
        this.page = page;
        this.limit = limit;
        this.totalPages = (int) Math.ceil((double) total / limit);
    }
    
    // Getters
    public List<T> getItems() { return items; }
    public long getTotal() { return total; }
    public int getPage() { return page; }
    public int getLimit() { return limit; }
    public int getTotalPages() { return totalPages; }
}
```

### PostgreSQL Repository Implementation

```java
// infrastructure/repositories/PostgresProjectRepository.java
package com.saas.infrastructure.repositories;

import com.saas.domain.entities.Project;
import com.saas.domain.entities.Project.ProjectStatus;
import com.saas.domain.repositories.IProjectRepository;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Repository
public class PostgresProjectRepository implements IProjectRepository {
    
    private final JdbcTemplate jdbcTemplate;
    
    public PostgresProjectRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    
    @Override
    public Optional<Project> findById(String tenantId, String projectId) {
        String sql = "SELECT * FROM projects WHERE id = ? AND tenant_id = ?";
        
        List<Project> results = jdbcTemplate.query(
            sql,
            new Object[]{projectId, tenantId},
            new ProjectRowMapper()
        );
        
        return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
    }
    
    @Override
    public List<Project> findAll(String tenantId) {
        String sql = "SELECT * FROM projects WHERE tenant_id = ? ORDER BY created_at DESC";
        return jdbcTemplate.query(sql, new Object[]{tenantId}, new ProjectRowMapper());
    }
    
    @Override
    public Project save(Project project) {
        if (exists(project.getTenantId(), project.getId())) {
            return update(project);
        } else {
            return insert(project);
        }
    }
    
    private Project insert(Project project) {
        String sql = """
            INSERT INTO projects (
                id, tenant_id, name, description, status, budget,
                start_date, end_date, created_by, created_at, updated_at
            )
            VALUES (?, ?, ?, ?, ?::project_status, ?, ?, ?, ?, ?, ?)
            """;
        
        jdbcTemplate.update(sql,
            project.getId(),
            project.getTenantId(),
            project.getName(),
            project.getDescription(),
            project.getStatus().name(),
            project.getBudget(),
            project.getStartDate(),
            project.getEndDate(),
            project.getCreatedBy(),
            project.getCreatedAt(),
            project.getUpdatedAt()
        );
        
        return project;
    }
    
    private Project update(Project project) {
        String sql = """
            UPDATE projects 
            SET name = ?, description = ?, status = ?::project_status, 
                budget = ?, start_date = ?, end_date = ?, updated_at = ?
            WHERE id = ? AND tenant_id = ?
            """;
        
        int rows = jdbcTemplate.update(sql,
            project.getName(),
            project.getDescription(),
            project.getStatus().name(),
            project.getBudget(),
            project.getStartDate(),
            project.getEndDate(),
            project.getUpdatedAt(),
            project.getId(),
            project.getTenantId()
        );
        
        if (rows == 0) {
            throw new RuntimeException("Project not found or update failed");
        }
        
        return project;
    }
    
    @Override
    public boolean delete(String tenantId, String projectId) {
        String sql = "DELETE FROM projects WHERE id = ? AND tenant_id = ?";
        int rows = jdbcTemplate.update(sql, projectId, tenantId);
        return rows > 0;
    }
    
    @Override
    public List<Project> findByStatus(String tenantId, ProjectStatus status) {
        String sql = """
            SELECT * FROM projects 
            WHERE tenant_id = ? AND status = ?::project_status
            ORDER BY created_at DESC
            """;
        
        return jdbcTemplate.query(
            sql,
            new Object[]{tenantId, status.name()},
            new ProjectRowMapper()
        );
    }
    
    @Override
    public List<Project> search(String tenantId, String searchTerm) {
        String sql = """
            SELECT * FROM projects 
            WHERE tenant_id = ? 
              AND (name ILIKE ? OR description ILIKE ?)
            ORDER BY created_at DESC
            """;
        
        String pattern = "%" + searchTerm + "%";
        return jdbcTemplate.query(
            sql,
            new Object[]{tenantId, pattern, pattern},
            new ProjectRowMapper()
        );
    }
    
    @Override
    public long count(String tenantId) {
        String sql = "SELECT COUNT(*) FROM projects WHERE tenant_id = ?";
        Long count = jdbcTemplate.queryForObject(sql, Long.class, tenantId);
        return count != null ? count : 0;
    }
    
    @Override
    public boolean exists(String tenantId, String projectId) {
        String sql = "SELECT EXISTS(SELECT 1 FROM projects WHERE id = ? AND tenant_id = ?)";
        Boolean exists = jdbcTemplate.queryForObject(sql, Boolean.class, projectId, tenantId);
        return exists != null && exists;
    }
    
    @Override
    public boolean existsByName(String tenantId, String name) {
        String sql = "SELECT EXISTS(SELECT 1 FROM projects WHERE tenant_id = ? AND name = ?)";
        Boolean exists = jdbcTemplate.queryForObject(sql, Boolean.class, tenantId, name);
        return exists != null && exists;
    }
    
    // Row mapper to convert ResultSet to Project entity
    private static class ProjectRowMapper implements RowMapper<Project> {
        @Override
        public Project mapRow(ResultSet rs, int rowNum) throws SQLException {
            return new Project(
                rs.getString("id"),
                rs.getString("tenant_id"),
                rs.getString("name"),
                rs.getString("description"),
                ProjectStatus.valueOf(rs.getString("status")),
                rs.getDouble("budget"),
                rs.getTimestamp("start_date") != null 
                    ? rs.getTimestamp("start_date").toLocalDateTime() 
                    : null,
                rs.getTimestamp("end_date") != null 
                    ? rs.getTimestamp("end_date").toLocalDateTime() 
                    : null,
                rs.getString("created_by"),
                rs.getTimestamp("created_at").toLocalDateTime(),
                rs.getTimestamp("updated_at").toLocalDateTime()
            );
        }
    }
}
```

#### Service Layer

```java
// application/services/ProjectService.java
package com.saas.application.services;

import com.saas.domain.entities.Project;
import com.saas.domain.repositories.IProjectRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.UUID;

@Service
@Transactional
public class ProjectService {
    
    private final IProjectRepository projectRepository;
    private final IEventBus eventBus;
    
    public ProjectService(IProjectRepository projectRepository, IEventBus eventBus) {
        this.projectRepository = projectRepository;
        this.eventBus = eventBus;
    }
    
    public Project createProject(String tenantId, String userId, CreateProjectDTO data) {
        // Check for duplicate name
        if (projectRepository.existsByName(tenantId, data.getName())) {
            throw new IllegalArgumentException("Project with this name already exists");
        }
        
        // Create domain entity
        Project project = new Project(
            UUID.randomUUID().toString(),
            tenantId,
            data.getName(),
            data.getDescription(),
            Project.ProjectStatus.ACTIVE,
            data.getBudget(),
            data.getStartDate(),
            data.getEndDate(),
            userId,
            LocalDateTime.now(),
            LocalDateTime.now()
        );
        
        // Save via repository
        Project savedProject = projectRepository.save(project);
        
        // Emit event
        eventBus.emit("project.created", new ProjectCreatedEvent(
            tenantId,
            savedProject.getId(),
            userId,
            LocalDateTime.now()
        ));
        
        return savedProject;
    }
    
    public Project getProject(String tenantId, String projectId) {
        return projectRepository.findById(tenantId, projectId)
            .orElseThrow(() -> new RuntimeException("Project not found"));
    }
    
    public Project completeProject(String tenantId, String projectId) {
        Project project = getProject(tenantId, projectId);
        
        // Business logic in domain
        project.complete();
        
        // Save via repository
        Project completedProject = projectRepository.save(project);
        
        eventBus.emit("project.completed", new ProjectCompletedEvent(
            tenantId,
            completedProject.getId(),
            LocalDateTime.now()
        ));
        
        return completedProject;
    }
    
    public Project archiveProject(String tenantId, String projectId) {
        Project project = getProject(tenantId, projectId);
        
        // Business logic in domain
        project.archive();
        
        // Save via repository
        Project archivedProject = projectRepository.save(project);
        
        eventBus.emit("project.archived", new ProjectArchivedEvent(
            tenantId,
            archivedProject.getId(),
            LocalDateTime.now()
        ));
        
        return archivedProject;
    }
}

// DTOs
class CreateProjectDTO {
    private String name;
    private String description;
    private Double budget;
    private LocalDateTime startDate;
    private LocalDateTime endDate;
    
    // Getters and setters
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public Double getBudget() { return budget; }
    public void setBudget(Double budget) { this.budget = budget; }
    public LocalDateTime getStartDate() { return startDate; }
    public void setStartDate(LocalDateTime startDate) { this.startDate = startDate; }
    public LocalDateTime getEndDate() { return endDate; }
    public void setEndDate(LocalDateTime endDate) { this.endDate = endDate; }
}
```

---

## Go Implementation

### Domain Entity

```go
// domain/entities/project.go
package entities

import (
    "errors"
    "time"
)

type ProjectStatus string

const (
    StatusActive    ProjectStatus = "active"
    StatusCompleted ProjectStatus = "completed"
    StatusArchived  ProjectStatus = "archived"
)

type Project struct {
    ID          string
    TenantID    string
    Name        string
    Description string
    Status      ProjectStatus
    Budget      *float64
    StartDate   *time.Time
    EndDate     *time.Time
    CreatedBy   string
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

func NewProject(
    id, tenantID, name, description string,
    status ProjectStatus,
    budget *float64,
    startDate, endDate *time.Time,
    createdBy string,
    createdAt, updatedAt time.Time,
) (*Project, error) {
    p := &Project{
        ID:          id,
        TenantID:    tenantID,
        Name:        name,
        Description: description,
        Status:      status,
        Budget:      budget,
        StartDate:   startDate,
        EndDate:     endDate,
        CreatedBy:   createdBy,
        CreatedAt:   createdAt,
        UpdatedAt:   updatedAt,
    }
    
    if err := p.Validate(); err != nil {
        return nil, err
    }
    
    return p, nil
}

func (p *Project) Validate() error {
    if p.Name == "" {
        return errors.New("project name is required")
    }
    
    if len(p.Name) > 255 {
        return errors.New("project name too long")
    }
    
    if p.StartDate != nil && p.EndDate != nil && p.EndDate.Before(*p.StartDate) {
        return errors.New("end date must be after start date")
    }
    
    return nil
}

// Business logic
func (p *Project) CanBeArchived() bool {
    return p.Status == StatusCompleted
}

func (p *Project) Archive() error {
    if !p.CanBeArchived() {
        return errors.New("only completed projects can be archived")
    }
    p.Status = StatusArchived
    p.UpdatedAt = time.Now()
    return nil
}

func (p *Project) Complete() error {
    if p.Status == StatusArchived {
        return errors.New("cannot complete an archived project")
    }
    p.Status = StatusCompleted
    p.UpdatedAt = time.Now()
    return nil
}

func (p *Project) UpdateDetails(name, description string) error {
    if p.Status == StatusArchived {
        return errors.New("cannot update archived project")
    }
    p.Name = name
    p.Description = description
    p.UpdatedAt = time.Now()
    return p.Validate()
}

func (p *Project) IsOverdue() bool {
    if p.EndDate == nil || p.Status != StatusActive {
        return false
    }
    return time.Now().After(*p.EndDate)
}
```

### Repository Interface

```go
// domain/repositories/project_repository.go
package repositories

import (
    "context"
    "saas/domain/entities"
    "time"
)

type IProjectRepository interface {
    // Basic CRUD
    FindByID(ctx context.Context, tenantID, projectID string) (*entities.Project, error)
    FindAll(ctx context.Context, tenantID string) ([]*entities.Project, error)
    Save(ctx context.Context, project *entities.Project) error
    Delete(ctx context.Context, tenantID, projectID string) error
    
    // Query methods
    FindByStatus(ctx context.Context, tenantID string, status entities.ProjectStatus) ([]*entities.Project, error)
    FindByCreator(ctx context.Context, tenantID, userID string) ([]*entities.Project, error)
    FindByDateRange(ctx context.Context, tenantID string, startDate, endDate time.Time) ([]*entities.Project, error)
    Search(ctx context.Context, tenantID, searchTerm string) ([]*entities.Project, error)
    FindOverdue(ctx context.Context, tenantID string) ([]*entities.Project, error)
    
    // Pagination
    FindWithPagination(ctx context.Context, tenantID string, page, limit int, filters ProjectFilters) (*PageResult, error)
    
    // Aggregations
    Count(ctx context.Context, tenantID string) (int64, error)
    CountByStatus(ctx context.Context, tenantID string, status entities.ProjectStatus) (int64, error)
    
    // Existence checks
    Exists(ctx context.Context, tenantID, projectID string) (bool, error)
    ExistsByName(ctx context.Context, tenantID, name string) (bool, error)
}

type ProjectFilters struct {
    Status    *entities.ProjectStatus
    Search    string
    CreatedBy string
}

type PageResult struct {
    Projects   []*entities.Project
    Total      int64
    Page       int
    Limit      int
    TotalPages int
}
```

### PostgreSQL Repository Implementation

```go
// infrastructure/repositories/postgres_project_repository.go
package repositories

import (
    "context"
    "database/sql"
    "fmt"
    "saas/domain/entities"
    "saas/domain/repositories"
    "time"
)

type PostgresProjectRepository struct {
    db *sql.DB
}

func NewPostgresProjectRepository(db *sql.DB) *PostgresProjectRepository {
    return &PostgresProjectRepository{db: db}
}

func (r *PostgresProjectRepository) FindByID(ctx context.Context, tenantID, projectID string) (*entities.Project, error) {
    query := `
        SELECT id, tenant_id, name, description, status, budget,
               start_date, end_date, created_by, created_at, updated_at
        FROM projects
        WHERE id = $1 AND tenant_id = $2
    `
    
    var project entities.Project
    var budget sql.NullFloat64
    var startDate, endDate sql.NullTime
    
    err := r.db.QueryRowContext(ctx, query, projectID, tenantID).Scan(
        &project.ID,
        &project.TenantID,
        &project.Name,
        &project.Description,
        &project.Status,
        &budget,
        &startDate,
        &endDate,
        &project.CreatedBy,
        &project.CreatedAt,
        &project.UpdatedAt,
    )
    
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    
    if budget.Valid {
        project.Budget = &budget.Float64
    }
    if startDate.Valid {
        project.StartDate = &startDate.Time
    }
    if endDate.Valid {
        project.EndDate = &endDate.Time
    }
    
    return &project, nil
}

func (r *PostgresProjectRepository) FindAll(ctx context.Context, tenantID string) ([]*entities.Project, error) {
    query := `
        SELECT id, tenant_id, name, description, status, budget,
               start_date, end_date, created_by, created_at, updated_at
        FROM projects
        WHERE tenant_id = $1
        ORDER BY created_at DESC
    `
    
    rows, err := r.db.QueryContext(ctx, query, tenantID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var projects []*entities.Project
    for rows.Next() {
        project, err := r.scanProject(rows)
        if err != nil {
            return nil, err
        }
        projects = append(projects, project)
    }
    
    return projects, nil
}

func (r *PostgresProjectRepository) Save(ctx context.Context, project *entities.Project) error {
    exists, err := r.Exists(ctx, project.TenantID, project.ID)
    if err != nil {
        return err
    }
    
    if exists {
        return r.update(ctx, project)
    }
    return r.insert(ctx, project)
}

func (r *PostgresProjectRepository) insert(ctx context.Context, project *entities.Project) error {
    query := `
        INSERT INTO projects (
            id, tenant_id, name, description, status, budget,
            start_date, end_date, created_by, created_at, updated_at
        )
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
    `
    
    _, err := r.db.ExecContext(ctx, query,
        project.ID,
        project.TenantID,
        project.Name,
        project.Description,
        project.Status,
        project.Budget,
        project.StartDate,
        project.EndDate,
        project.CreatedBy,
        project.CreatedAt,
        project.UpdatedAt,
    )
    
    return err
}

func (r *PostgresProjectRepository) update(ctx context.Context, project *entities.Project) error {
    query := `
        UPDATE projects
        SET name = $1, description = $2, status = $3, budget = $4,
            start_date = $5, end_date = $6, updated_at = $7
        WHERE id = $8 AND tenant_id = $9
    `
    
    result, err := r.db.ExecContext(ctx, query,
        project.Name,
        project.Description,
        project.Status,
        project.Budget,
        project.StartDate,
        project.EndDate,
        project.UpdatedAt,
        project.ID,
        project.TenantID,
    )
    
    if err != nil {
        return err
    }
    
    rows, err := result.RowsAffected()
    if err != nil {
        return err
    }
    if rows == 0 {
        return fmt.Errorf("project not found or update failed")
    }
    
    return nil
}

func (r *PostgresProjectRepository) Delete(ctx context.Context, tenantID, projectID string) error {
    query := `DELETE FROM projects WHERE id = $1 AND tenant_id = $2`
    
    result, err := r.db.ExecContext(ctx, query, projectID, tenantID)
    if err != nil {
        return err
    }
    
    rows, err := result.RowsAffected()
    if err != nil {
        return err
    }
    if rows == 0 {
        return fmt.Errorf("project not found")
    }
    
    return nil
}

func (r *PostgresProjectRepository) Search(ctx context.Context, tenantID, searchTerm string) ([]*entities.Project, error) {
    query := `
        SELECT id, tenant_id, name, description, status, budget,
               start_date, end_date, created_by, created_at, updated_at
        FROM projects
        WHERE tenant_id = $1
          AND (name ILIKE $2 OR description ILIKE $2)
        ORDER BY created_at DESC
    `
    
    pattern := "%" + searchTerm + "%"
    rows, err := r.db.QueryContext(ctx, query, tenantID, pattern)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var projects []*entities.Project
    for rows.Next() {
        project, err := r.scanProject(rows)
        if err != nil {
            return nil, err
        }
        projects = append(projects, project)
    }
    
    return projects, nil
}

func (r *PostgresProjectRepository) Count(ctx context.Context, tenantID string) (int64, error) {
    query := `SELECT COUNT(*) FROM projects WHERE tenant_id = $1`
    
    var count int64
    err := r.db.QueryRowContext(ctx, query, tenantID).Scan(&count)
    return count, err
}

func (r *PostgresProjectRepository) Exists(ctx context.Context, tenantID, projectID string) (bool, error) {
    query := `SELECT EXISTS(SELECT 1 FROM projects WHERE id = $1 AND tenant_id = $2)`
    
    var exists bool
    err := r.db.QueryRowContext(ctx, query, projectID, tenantID).Scan(&exists)
    return exists, err
}

func (r *PostgresProjectRepository) ExistsByName(ctx context.Context, tenantID, name string) (bool, error) {
    query := `SELECT EXISTS(SELECT 1 FROM projects WHERE tenant_id = $1 AND name = $2)`
    
    var exists bool
    err := r.db.QueryRowContext(ctx, query, tenantID, name).Scan(&exists)
    return exists, err
}

// Helper function to scan a project from rows
func (r *PostgresProjectRepository) scanProject(rows *sql.Rows) (*entities.Project, error) {
    var project entities.Project
    var budget sql.NullFloat64
    var startDate, endDate sql.NullTime
    
    err := rows.Scan(
        &project.ID,
        &project.TenantID,
        &project.Name,
        &project.Description,
        &project.Status,
        &budget,
        &startDate,
        &endDate,
        &project.CreatedBy,
        &project.CreatedAt,
        &project.UpdatedAt,
    )
    
    if err != nil {
        return nil, err
    }
    
    if budget.Valid {
        project.Budget = &budget.Float64
    }
    if startDate.Valid {
        project.StartDate = &startDate.Time
    }
    if endDate.Valid {
        project.EndDate = &endDate.Time
    }
    
    return &project, nil
}
```


### Service Layer Implementation

```go
// application/services/project_service.go
package services

import (
    "context"
    "errors"
    "time"
    
    "github.com/google/uuid"
    "yourapp/domain/entities"
    "yourapp/domain/repositories"
    "yourapp/application/events"
)

type ProjectService struct {
    projectRepo repositories.ProjectRepository
    eventBus    events.EventBus
}

func NewProjectService(
    projectRepo repositories.ProjectRepository,
    eventBus events.EventBus,
) *ProjectService {
    return &ProjectService{
        projectRepo: projectRepo,
        eventBus:    eventBus,
    }
}

// CreateProject creates a new project with business logic validation
func (s *ProjectService) CreateProject(
    ctx context.Context,
    tenantID uuid.UUID,
    userID uuid.UUID,
    req CreateProjectRequest,
) (*entities.Project, error) {
    // Business rule: Check for duplicate name
    exists, err := s.projectRepo.ExistsByName(ctx, tenantID, req.Name)
    if err != nil {
        return nil, err
    }
    if exists {
        return nil, errors.New("project with this name already exists")
    }
    
    // Create domain entity
    project := &entities.Project{
        ID:          uuid.New(),
        TenantID:    tenantID,
        Name:        req.Name,
        Description: req.Description,
        Status:      entities.ProjectStatusActive,
        Budget:      req.Budget,
        StartDate:   req.StartDate,
        EndDate:     req.EndDate,
        CreatedBy:   userID,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    
    // Validate using domain logic
    if err := project.Validate(); err != nil {
        return nil, err
    }
    
    // Save via repository
    savedProject, err := s.projectRepo.Save(ctx, project)
    if err != nil {
        return nil, err
    }
    
    // Emit domain event
    s.eventBus.Publish(ctx, events.ProjectCreatedEvent{
        TenantID:  tenantID,
        ProjectID: savedProject.ID,
        UserID:    userID,
        Timestamp: time.Now(),
    })
    
    return savedProject, nil
}

// GetProject retrieves a project by ID
func (s *ProjectService) GetProject(
    ctx context.Context,
    tenantID uuid.UUID,
    projectID uuid.UUID,
) (*entities.Project, error) {
    project, err := s.projectRepo.FindByID(ctx, tenantID, projectID)
    if err != nil {
        return nil, err
    }
    if project == nil {
        return nil, errors.New("project not found")
    }
    return project, nil
}

// UpdateProject updates project details
func (s *ProjectService) UpdateProject(
    ctx context.Context,
    tenantID uuid.UUID,
    projectID uuid.UUID,
    updates UpdateProjectRequest,
) (*entities.Project, error) {
    // Load project
    project, err := s.GetProject(ctx, tenantID, projectID)
    if err != nil {
        return nil, err
    }
    
    // Apply updates (business logic in domain entity)
    if updates.Name != nil && updates.Description != nil {
        if err := project.UpdateDetails(*updates.Name, *updates.Description); err != nil {
            return nil, err
        }
    }
    
    if updates.Budget != nil {
        if err := project.UpdateBudget(updates.Budget); err != nil {
            return nil, err
        }
    }
    
    // Save via repository
    updatedProject, err := s.projectRepo.Save(ctx, project)
    if err != nil {
        return nil, err
    }
    
    // Emit event
    s.eventBus.Publish(ctx, events.ProjectUpdatedEvent{
        TenantID:  tenantID,
        ProjectID: projectID,
        Updates:   updates,
        Timestamp: time.Now(),
    })
    
    return updatedProject, nil
}

// CompleteProject marks a project as completed
func (s *ProjectService) CompleteProject(
    ctx context.Context,
    tenantID uuid.UUID,
    projectID uuid.UUID,
) (*entities.Project, error) {
    project, err := s.GetProject(ctx, tenantID, projectID)
    if err != nil {
        return nil, err
    }
    
    // Business logic in domain
    if err := project.Complete(); err != nil {
        return nil, err
    }
    
    // Save via repository
    completedProject, err := s.projectRepo.Save(ctx, project)
    if err != nil {
        return nil, err
    }
    
    s.eventBus.Publish(ctx, events.ProjectCompletedEvent{
        TenantID:  tenantID,
        ProjectID: projectID,
        Timestamp: time.Now(),
    })
    
    return completedProject, nil
}

// ArchiveProject archives a completed project
func (s *ProjectService) ArchiveProject(
    ctx context.Context,
    tenantID uuid.UUID,
    projectID uuid.UUID,
) (*entities.Project, error) {
    project, err := s.GetProject(ctx, tenantID, projectID)
    if err != nil {
        return nil, err
    }
    
    // Business logic in domain
    if err := project.Archive(); err != nil {
        return nil, err
    }
    
    // Save via repository
    archivedProject, err := s.projectRepo.Save(ctx, project)
    if err != nil {
        return nil, err
    }
    
    s.eventBus.Publish(ctx, events.ProjectArchivedEvent{
        TenantID:  tenantID,
        ProjectID: projectID,
        Timestamp: time.Now(),
    })
    
    return archivedProject, nil
}

// ListProjects returns paginated projects with filters
func (s *ProjectService) ListProjects(
    ctx context.Context,
    tenantID uuid.UUID,
    options ListProjectsOptions,
) (*PaginatedProjects, error) {
    page := options.Page
    if page < 1 {
        page = 1
    }
    
    limit := options.Limit
    if limit < 1 {
        limit = 20
    }
    if limit > 100 {
        limit = 100 // Max limit
    }
    
    filters := repositories.ProjectFilters{
        Status:    options.Status,
        Search:    options.Search,
        CreatedBy: options.CreatedBy,
    }
    
    return s.projectRepo.FindWithPagination(ctx, tenantID, page, limit, filters)
}

// SearchProjects searches projects by term
func (s *ProjectService) SearchProjects(
    ctx context.Context,
    tenantID uuid.UUID,
    searchTerm string,
) ([]*entities.Project, error) {
    return s.projectRepo.Search(ctx, tenantID, searchTerm)
}

// GetOverdueProjects returns all overdue projects
func (s *ProjectService) GetOverdueProjects(
    ctx context.Context,
    tenantID uuid.UUID,
) ([]*entities.Project, error) {
    return s.projectRepo.FindOverdue(ctx, tenantID)
}

// DeleteProject deletes a project
func (s *ProjectService) DeleteProject(
    ctx context.Context,
    tenantID uuid.UUID,
    projectID uuid.UUID,
) error {
    deleted, err := s.projectRepo.Delete(ctx, tenantID, projectID)
    if err != nil {
        return err
    }
    if !deleted {
        return errors.New("project not found or already deleted")
    }
    
    s.eventBus.Publish(ctx, events.ProjectDeletedEvent{
        TenantID:  tenantID,
        ProjectID: projectID,
        Timestamp: time.Now(),
    })
    
    return nil
}

// GetProjectStats returns statistics for tenant's projects
func (s *ProjectService) GetProjectStats(
    ctx context.Context,
    tenantID uuid.UUID,
) (*ProjectStats, error) {
    total, err := s.projectRepo.Count(ctx, tenantID)
    if err != nil {
        return nil, err
    }
    
    active, err := s.projectRepo.CountByStatus(ctx, tenantID, string(entities.ProjectStatusActive))
    if err != nil {
        return nil, err
    }
    
    completed, err := s.projectRepo.CountByStatus(ctx, tenantID, string(entities.ProjectStatusCompleted))
    if err != nil {
        return nil, err
    }
    
    archived, err := s.projectRepo.CountByStatus(ctx, tenantID, string(entities.ProjectStatusArchived))
    if err != nil {
        return nil, err
    }
    
    overdue, err := s.projectRepo.FindOverdue(ctx, tenantID)
    if err != nil {
        return nil, err
    }
    
    return &ProjectStats{
        Total:     total,
        Active:    active,
        Completed: completed,
        Archived:  archived,
        Overdue:   len(overdue),
    }, nil
}

// DTOs and Request/Response types
type CreateProjectRequest struct {
    Name        string
    Description string
    Budget      *float64
    StartDate   *time.Time
    EndDate     *time.Time
}

type UpdateProjectRequest struct {
    Name        *string
    Description *string
    Budget      *float64
}

type ListProjectsOptions struct {
    Page      int
    Limit     int
    Status    *string
    Search    *string
    CreatedBy *uuid.UUID
}

type PaginatedProjects struct {
    Projects   []*entities.Project
    Total      int
    Page       int
    Limit      int
    TotalPages int
}

type ProjectStats struct {
    Total     int
    Active    int
    Completed int
    Archived  int
    Overdue   int
}
```

### Mock Repository Implementation

```go
// tests/mocks/mock_project_repository.go
package mocks

import (
    "context"
    "errors"
    "strings"
    "sync"
    "time"
    
    "yourapp/domain/entities"
    "yourapp/domain/repositories"
)

// MockProjectRepository implements repositories.ProjectRepository for testing
type MockProjectRepository struct {
    projects map[string]*entities.Project
    mu       sync.RWMutex
}

// NewMockProjectRepository creates a new mock repository
func NewMockProjectRepository() *MockProjectRepository {
    return &MockProjectRepository{
        projects: make(map[string]*entities.Project),
    }
}

// FindByID finds a project by ID and tenant ID
func (m *MockProjectRepository) FindByID(ctx context.Context, tenantID, projectID string) (*entities.Project, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    project, exists := m.projects[projectID]
    if !exists {
        return nil, errors.New("project not found")
    }
    
    if project.TenantID != tenantID {
        return nil, errors.New("project not found")
    }
    
    return project, nil
}

// FindAll returns all projects for a tenant
func (m *MockProjectRepository) FindAll(ctx context.Context, tenantID string) ([]*entities.Project, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    var result []*entities.Project
    for _, project := range m.projects {
        if project.TenantID == tenantID {
            result = append(result, project)
        }
    }
    
    // Sort by created date (newest first)
    // Simple bubble sort for testing
    for i := 0; i < len(result)-1; i++ {
        for j := 0; j < len(result)-i-1; j++ {
            if result[j].CreatedAt.Before(result[j+1].CreatedAt) {
                result[j], result[j+1] = result[j+1], result[j]
            }
        }
    }
    
    return result, nil
}

// Save inserts or updates a project
func (m *MockProjectRepository) Save(ctx context.Context, project *entities.Project) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    // Create a copy to avoid external modifications
    projectCopy := *project
    m.projects[project.ID] = &projectCopy
    
    return nil
}

// Delete removes a project
func (m *MockProjectRepository) Delete(ctx context.Context, tenantID, projectID string) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    project, exists := m.projects[projectID]
    if !exists {
        return errors.New("project not found")
    }
    
    if project.TenantID != tenantID {
        return errors.New("project not found")
    }
    
    delete(m.projects, projectID)
    return nil
}

// FindByStatus finds projects by status
func (m *MockProjectRepository) FindByStatus(ctx context.Context, tenantID, status string) ([]*entities.Project, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    var result []*entities.Project
    for _, project := range m.projects {
        if project.TenantID == tenantID && project.Status == status {
            result = append(result, project)
        }
    }
    
    return result, nil
}

// FindByCreator finds projects created by a specific user
func (m *MockProjectRepository) FindByCreator(ctx context.Context, tenantID, userID string) ([]*entities.Project, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    var result []*entities.Project
    for _, project := range m.projects {
        if project.TenantID == tenantID && project.CreatedBy == userID {
            result = append(result, project)
        }
    }
    
    return result, nil
}

// FindByDateRange finds projects within a date range
func (m *MockProjectRepository) FindByDateRange(ctx context.Context, tenantID string, startDate, endDate time.Time) ([]*entities.Project, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    var result []*entities.Project
    for _, project := range m.projects {
        if project.TenantID == tenantID &&
            !project.CreatedAt.Before(startDate) &&
            !project.CreatedAt.After(endDate) {
            result = append(result, project)
        }
    }
    
    return result, nil
}

// Search searches projects by name or description
func (m *MockProjectRepository) Search(ctx context.Context, tenantID, searchTerm string) ([]*entities.Project, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    searchLower := strings.ToLower(searchTerm)
    var result []*entities.Project
    
    for _, project := range m.projects {
        if project.TenantID != tenantID {
            continue
        }
        
        nameLower := strings.ToLower(project.Name)
        descLower := strings.ToLower(project.Description)
        
        if strings.Contains(nameLower, searchLower) || strings.Contains(descLower, searchLower) {
            result = append(result, project)
        }
    }
    
    return result, nil
}

// FindOverdue finds overdue active projects
func (m *MockProjectRepository) FindOverdue(ctx context.Context, tenantID string) ([]*entities.Project, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    now := time.Now()
    var result []*entities.Project
    
    for _, project := range m.projects {
        if project.TenantID == tenantID &&
            project.Status == "active" &&
            project.EndDate != nil &&
            project.EndDate.Before(now) {
            result = append(result, project)
        }
    }
    
    return result, nil
}

// FindWithPagination finds projects with pagination
func (m *MockProjectRepository) FindWithPagination(
    ctx context.Context,
    tenantID string,
    page, limit int,
    filters map[string]interface{},
) (*repositories.PaginatedResult, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    // Filter projects
    var filtered []*entities.Project
    for _, project := range m.projects {
        if project.TenantID != tenantID {
            continue
        }
        
        // Apply status filter
        if status, ok := filters["status"].(string); ok && status != "" {
            if project.Status != status {
                continue
            }
        }
        
        // Apply search filter
        if search, ok := filters["search"].(string); ok && search != "" {
            searchLower := strings.ToLower(search)
            nameLower := strings.ToLower(project.Name)
            descLower := strings.ToLower(project.Description)
            
            if !strings.Contains(nameLower, searchLower) && !strings.Contains(descLower, searchLower) {
                continue
            }
        }
        
        // Apply creator filter
        if createdBy, ok := filters["createdBy"].(string); ok && createdBy != "" {
            if project.CreatedBy != createdBy {
                continue
            }
        }
        
        filtered = append(filtered, project)
    }
    
    // Calculate pagination
    total := len(filtered)
    start := (page - 1) * limit
    end := start + limit
    
    if start > total {
        start = total
    }
    if end > total {
        end = total
    }
    
    var paginated []*entities.Project
    if start < end {
        paginated = filtered[start:end]
    }
    
    totalPages := (total + limit - 1) / limit
    if totalPages < 1 {
        totalPages = 1
    }
    
    return &repositories.PaginatedResult{
        Projects:   paginated,
        Total:      total,
        Page:       page,
        Limit:      limit,
        TotalPages: totalPages,
    }, nil
}

// Count counts total projects for a tenant
func (m *MockProjectRepository) Count(ctx context.Context, tenantID string) (int, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    count := 0
    for _, project := range m.projects {
        if project.TenantID == tenantID {
            count++
        }
    }
    
    return count, nil
}

// CountByStatus counts projects by status
func (m *MockProjectRepository) CountByStatus(ctx context.Context, tenantID, status string) (int, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    count := 0
    for _, project := range m.projects {
        if project.TenantID == tenantID && project.Status == status {
            count++
        }
    }
    
    return count, nil
}

// SaveMany saves multiple projects
func (m *MockProjectRepository) SaveMany(ctx context.Context, projects []*entities.Project) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    for _, project := range projects {
        projectCopy := *project
        m.projects[project.ID] = &projectCopy
    }
    
    return nil
}

// DeleteMany deletes multiple projects
func (m *MockProjectRepository) DeleteMany(ctx context.Context, tenantID string, projectIDs []string) (int, error) {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    deleted := 0
    for _, projectID := range projectIDs {
        project, exists := m.projects[projectID]
        if exists && project.TenantID == tenantID {
            delete(m.projects, projectID)
            deleted++
        }
    }
    
    return deleted, nil
}

// Exists checks if a project exists
func (m *MockProjectRepository) Exists(ctx context.Context, tenantID, projectID string) (bool, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    project, exists := m.projects[projectID]
    if !exists {
        return false, nil
    }
    
    return project.TenantID == tenantID, nil
}

// ExistsByName checks if a project with the given name exists
func (m *MockProjectRepository) ExistsByName(ctx context.Context, tenantID, name string) (bool, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    for _, project := range m.projects {
        if project.TenantID == tenantID && project.Name == name {
            return true, nil
        }
    }
    
    return false, nil
}

// Test helper methods

// Clear removes all projects (useful for test cleanup)
func (m *MockProjectRepository) Clear() {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    m.projects = make(map[string]*entities.Project)
}

// Seed adds multiple projects to the mock repository
func (m *MockProjectRepository) Seed(projects []*entities.Project) {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    for _, project := range projects {
        projectCopy := *project
        m.projects[project.ID] = &projectCopy
    }
}

// GetAll returns all projects (no tenant filtering - for testing only)
func (m *MockProjectRepository) GetAll() []*entities.Project {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    var result []*entities.Project
    for _, project := range m.projects {
        result = append(result, project)
    }
    
    return result
}
```

### Test Example

```go
// application/services/project_service_test.go
package services

import (
    "context"
    "testing"
    "time"
    
    "github.com/google/uuid"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    
    "yourapp/domain/entities"
    "yourapp/tests/mocks"
)

func TestProjectService_CreateProject(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    
    // Test data
    projectData := CreateProjectInput{
        Name:        "Test Project",
        Description: "Test Description",
        Budget:      10000.0,
        StartDate:   time.Now(),
        EndDate:     time.Now().Add(30 * 24 * time.Hour),
    }
    
    // Execute
    project, err := service.CreateProject(ctx, tenantID, userID, projectData)
    
    // Assert
    require.NoError(t, err)
    assert.NotEmpty(t, project.ID)
    assert.Equal(t, tenantID, project.TenantID)
    assert.Equal(t, "Test Project", project.Name)
    assert.Equal(t, "active", project.Status)
    assert.Equal(t, userID, project.CreatedBy)
    
    // Verify it was saved
    savedProject, err := mockRepo.FindByID(ctx, tenantID, project.ID)
    require.NoError(t, err)
    assert.Equal(t, project.ID, savedProject.ID)
    
    // Verify event was emitted
    events := mockEventBus.GetEvents("project.created")
    assert.Len(t, events, 1)
}

func TestProjectService_CreateProject_DuplicateName(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    
    // Seed existing project
    existingProject := &entities.Project{
        ID:          uuid.New().String(),
        TenantID:    tenantID,
        Name:        "Existing Project",
        Description: "Existing",
        Status:      "active",
        CreatedBy:   userID,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    mockRepo.Save(ctx, existingProject)
    
    // Try to create project with same name
    projectData := CreateProjectInput{
        Name:        "Existing Project",
        Description: "Duplicate",
    }
    
    // Execute
    _, err := service.CreateProject(ctx, tenantID, userID, projectData)
    
    // Assert
    require.Error(t, err)
    assert.Contains(t, err.Error(), "already exists")
}

func TestProjectService_GetProject(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    projectID := uuid.New().String()
    
    // Seed project
    project := &entities.Project{
        ID:          projectID,
        TenantID:    tenantID,
        Name:        "Test Project",
        Description: "Test",
        Status:      "active",
        CreatedBy:   userID,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    mockRepo.Save(ctx, project)
    
    // Execute
    result, err := service.GetProject(ctx, tenantID, projectID)
    
    // Assert
    require.NoError(t, err)
    assert.Equal(t, projectID, result.ID)
    assert.Equal(t, "Test Project", result.Name)
}

func TestProjectService_GetProject_NotFound(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    projectID := uuid.New().String()
    
    // Execute
    _, err := service.GetProject(ctx, tenantID, projectID)
    
    // Assert
    require.Error(t, err)
    assert.Contains(t, err.Error(), "not found")
}

func TestProjectService_UpdateProject(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    projectID := uuid.New().String()
    
    // Seed project
    project := &entities.Project{
        ID:          projectID,
        TenantID:    tenantID,
        Name:        "Old Name",
        Description: "Old Description",
        Status:      "active",
        CreatedBy:   userID,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    mockRepo.Save(ctx, project)
    
    // Update data
    updateData := UpdateProjectInput{
        Name:        "New Name",
        Description: "New Description",
        Budget:      5000.0,
    }
    
    // Execute
    updated, err := service.UpdateProject(ctx, tenantID, projectID, updateData)
    
    // Assert
    require.NoError(t, err)
    assert.Equal(t, "New Name", updated.Name)
    assert.Equal(t, "New Description", updated.Description)
    assert.Equal(t, 5000.0, *updated.Budget)
    
    // Verify event was emitted
    events := mockEventBus.GetEvents("project.updated")
    assert.Len(t, events, 1)
}

func TestProjectService_CompleteProject(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    projectID := uuid.New().String()
    
    // Seed active project
    project := &entities.Project{
        ID:          projectID,
        TenantID:    tenantID,
        Name:        "Test Project",
        Description: "Test",
        Status:      "active",
        CreatedBy:   userID,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    mockRepo.Save(ctx, project)
    
    // Execute
    completed, err := service.CompleteProject(ctx, tenantID, projectID)
    
    // Assert
    require.NoError(t, err)
    assert.Equal(t, "completed", completed.Status)
    
    // Verify event
    events := mockEventBus.GetEvents("project.completed")
    assert.Len(t, events, 1)
}

func TestProjectService_ArchiveProject(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    projectID := uuid.New().String()
    
    // Seed completed project
    project := &entities.Project{
        ID:          projectID,
        TenantID:    tenantID,
        Name:        "Test Project",
        Description: "Test",
        Status:      "completed",
        CreatedBy:   userID,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    mockRepo.Save(ctx, project)
    
    // Execute
    archived, err := service.ArchiveProject(ctx, tenantID, projectID)
    
    // Assert
    require.NoError(t, err)
    assert.Equal(t, "archived", archived.Status)
}

func TestProjectService_ArchiveProject_CannotArchiveActive(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    projectID := uuid.New().String()
    
    // Seed active project
    project := &entities.Project{
        ID:          projectID,
        TenantID:    tenantID,
        Name:        "Test Project",
        Description: "Test",
        Status:      "active",  // Still active
        CreatedBy:   userID,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    mockRepo.Save(ctx, project)
    
    // Execute
    _, err := service.ArchiveProject(ctx, tenantID, projectID)
    
    // Assert
    require.Error(t, err)
    assert.Contains(t, err.Error(), "completed")
}

func TestProjectService_SearchProjects(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    
    // Seed multiple projects
    projects := []*entities.Project{
        {
            ID:          uuid.New().String(),
            TenantID:    tenantID,
            Name:        "Website Redesign",
            Description: "Update company website",
            Status:      "active",
            CreatedBy:   userID,
            CreatedAt:   time.Now(),
            UpdatedAt:   time.Now(),
        },
        {
            ID:          uuid.New().String(),
            TenantID:    tenantID,
            Name:        "Mobile App",
            Description: "New mobile application",
            Status:      "active",
            CreatedBy:   userID,
            CreatedAt:   time.Now(),
            UpdatedAt:   time.Now(),
        },
        {
            ID:          uuid.New().String(),
            TenantID:    tenantID,
            Name:        "Database Migration",
            Description: "Migrate to new database",
            Status:      "active",
            CreatedBy:   userID,
            CreatedAt:   time.Now(),
            UpdatedAt:   time.Now(),
        },
    }
    
    for _, p := range projects {
        mockRepo.Save(ctx, p)
    }
    
    // Execute - search for "website"
    results, err := service.SearchProjects(ctx, tenantID, "website")
    
    // Assert
    require.NoError(t, err)
    assert.Len(t, results, 1)
    assert.Equal(t, "Website Redesign", results[0].Name)
}

func TestProjectService_ListProjects_WithPagination(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    
    // Seed 25 projects
    for i := 0; i < 25; i++ {
        project := &entities.Project{
            ID:          uuid.New().String(),
            TenantID:    tenantID,
            Name:        fmt.Sprintf("Project %d", i+1),
            Description: "Test",
            Status:      "active",
            CreatedBy:   userID,
            CreatedAt:   time.Now(),
            UpdatedAt:   time.Now(),
        }
        mockRepo.Save(ctx, project)
    }
    
    // Execute - get page 1 (10 items per page)
    result, err := service.ListProjects(ctx, tenantID, ListProjectsOptions{
        Page:  1,
        Limit: 10,
    })
    
    // Assert
    require.NoError(t, err)
    assert.Len(t, result.Projects, 10)
    assert.Equal(t, 25, result.Total)
    assert.Equal(t, 1, result.Page)
    assert.Equal(t, 3, result.TotalPages) // 25 / 10 = 3 pages
}

func TestProjectService_DeleteProject(t *testing.T) {
    // Setup
    mockRepo := mocks.NewMockProjectRepository()
    mockEventBus := mocks.NewMockEventBus()
    service := NewProjectService(mockRepo, mockEventBus)
    
    ctx := context.Background()
    tenantID := uuid.New().String()
    userID := uuid.New().String()
    projectID := uuid.New().String()
    
    // Seed project
    project := &entities.Project{
        ID:          projectID,
        TenantID:    tenantID,
        Name:        "Test Project",
        Description: "Test",
        Status:      "active",
        CreatedBy:   userID,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    mockRepo.Save(ctx, project)
    
    // Execute
    err := service.DeleteProject(ctx, tenantID, projectID)
    
    // Assert
    require.NoError(t, err)
    
    // Verify it's deleted
    _, err = mockRepo.FindByID(ctx, tenantID, projectID)
    require.Error(t, err)
    
    // Verify event
    events := mockEventBus.GetEvents("project.deleted")
    assert.Len(t, events, 1)
}

// Test cleanup
func TestMain(m *testing.M) {
    // Run tests
    code := m.Run()
    
    // Exit
    os.Exit(code)
}
```

###  Running the Tests

```bash
# Run all tests
go test ./application/services/...

# Run with verbose output
go test -v ./application/services/...

# Run specific test
go test -v ./application/services -run TestProjectService_CreateProject

# Run with coverage
go test -cover ./application/services/...

# Generate coverage report
go test -coverprofile=coverage.out ./application/services/...
go tool cover -html=coverage.out
```


### Benefits of Mock Repository in Go

1. **Fast Tests** - No database connection needed
2. **Isolated Tests** - Each test is independent
3. **Easy Setup** - Simple `NewMockProjectRepository()`
4. **Thread-Safe** - Uses `sync.RWMutex` for concurrent access
5. **Test Helpers** - `Clear()`, `Seed()`, `GetAll()` methods
6. **Predictable** - No network latency or database quirks

