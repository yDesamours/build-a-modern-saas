

## Step 7: Advanced Repository Patterns

Now that you understand the basic Repository pattern, let's explore advanced patterns that solve specific problems in production SaaS applications.

### Overview of Advanced Patterns

We'll cover:

1. **Generic Repository** - Reduce code duplication across entities
2. **Specification Pattern** - Build complex queries dynamically
3. **Unit of Work Pattern** - Manage transactions across multiple repositories
4. **Cached Repository** - Add caching layer transparently
5. **Read/Write Repository Separation** - CQRS-lite pattern

***

## Pattern 7.1: Generic Repository

### The Problem

You're repeating the same CRUD code for every entity:

```typescript
// ❌ BAD: Repeating code for every entity
class ProjectRepository {
  async findById(id: string) { /* ... */ }
  async findAll() { /* ... */ }
  async save(entity) { /* ... */ }
  async delete(id: string) { /* ... */ }
}

class TaskRepository {
  async findById(id: string) { /* ... */ }  // Same code!
  async findAll() { /* ... */ }              // Same code!
  async save(entity) { /* ... */ }           // Same code!
  async delete(id: string) { /* ... */ }     // Same code!
}

class UserRepository {
  async findById(id: string) { /* ... */ }  // Same code again!
  async findAll() { /* ... */ }
  async save(entity) { /* ... */ }
  async delete(id: string) { /* ... */ }
}
```

### The Solution: Generic Base Repository

#### Implementation: TypeScript

```typescript
// infrastructure/repositories/BaseRepository.ts
import { Pool } from 'pg';

export abstract class BaseRepository<T> {
  constructor(
    protected db: Pool,
    protected tableName: string
  ) {}
  
  // Generic CRUD operations
  async findById(tenantId: string, id: string): Promise<T | null> {
    const result = await this.db.query(
      `SELECT * FROM ${this.tableName} 
       WHERE id = $1 AND tenant_id = $2`,
      [id, tenantId]
    );
    
    if (result.rows.length === 0) {
      return null;
    }
    
    return this.mapToDomain(result.rows[0]);
  }
  
  async findAll(tenantId: string): Promise<T[]> {
    const result = await this.db.query(
      `SELECT * FROM ${this.tableName} 
       WHERE tenant_id = $1 
       ORDER BY created_at DESC`,
      [tenantId]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
  
  async save(entity: any): Promise<T> {
    const exists = await this.exists(entity.tenantId, entity.id);
    
    if (exists) {
      return this.update(entity);
    } else {
      return this.insert(entity);
    }
  }
  
  async delete(tenantId: string, id: string): Promise<boolean> {
    const result = await this.db.query(
      `DELETE FROM ${this.tableName} 
       WHERE id = $1 AND tenant_id = $2`,
      [id, tenantId]
    );
    
    return result.rowCount > 0;
  }
  
  async exists(tenantId: string, id: string): Promise<boolean> {
    const result = await this.db.query(
      `SELECT EXISTS(
        SELECT 1 FROM ${this.tableName} 
        WHERE id = $1 AND tenant_id = $2
      ) as exists`,
      [id, tenantId]
    );
    
    return result.rows[0].exists;
  }
  
  async count(tenantId: string): Promise<number> {
    const result = await this.db.query(
      `SELECT COUNT(*) as count FROM ${this.tableName} 
       WHERE tenant_id = $1`,
      [tenantId]
    );
    
    return parseInt(result.rows[0].count);
  }
  
  // Subclasses must implement these
  protected abstract mapToDomain(row: any): T;
  protected abstract insert(entity: any): Promise<T>;
  protected abstract update(entity: any): Promise<T>;
}

// Now specific repositories are much simpler!
// infrastructure/repositories/PostgresProjectRepository.ts
export class PostgresProjectRepository 
  extends BaseRepository<Project> 
  implements IProjectRepository {
  
  constructor(db: Pool) {
    super(db, 'projects');
  }
  
  // Only implement entity-specific logic
  protected mapToDomain(row: any): Project {
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
  
  protected async insert(project: Project): Promise<Project> {
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
    
    return this.mapToDomain(result.rows[0]);
  }
  
  protected async update(project: Project): Promise<Project> {
    const result = await this.db.query(
      `UPDATE projects 
       SET name = $1, description = $2, status = $3, 
           budget = $4, start_date = $5, end_date = $6,
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
    
    return this.mapToDomain(result.rows[0]);
  }
  
  // Add entity-specific methods
  async findByStatus(tenantId: string, status: string): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 AND status = $2
       ORDER BY created_at DESC`,
      [tenantId, status]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
  
  async search(tenantId: string, searchTerm: string): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 
         AND (name ILIKE $2 OR description ILIKE $2)
       ORDER BY created_at DESC`,
      [tenantId, `%${searchTerm}%`]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
}

// Task repository is now much simpler too!
export class PostgresTaskRepository 
  extends BaseRepository<Task> 
  implements ITaskRepository {
  
  constructor(db: Pool) {
    super(db, 'tasks');
  }
  
  protected mapToDomain(row: any): Task {
    return new Task(
      row.id,
      row.tenant_id,
      row.project_id,
      row.title,
      row.description,
      row.status,
      row.assigned_to,
      row.created_by,
      row.created_at,
      row.updated_at
    );
  }
  
  protected async insert(task: Task): Promise<Task> {
    // Task-specific insert logic
    // ...
  }
  
  protected async update(task: Task): Promise<Task> {
    // Task-specific update logic
    // ...
  }
  
  // Task-specific methods
  async findByProject(tenantId: string, projectId: string): Promise<Task[]> {
    const result = await this.db.query(
      `SELECT * FROM tasks 
       WHERE tenant_id = $1 AND project_id = $2
       ORDER BY created_at DESC`,
      [tenantId, projectId]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
  
  async findByAssignee(tenantId: string, userId: string): Promise<Task[]> {
    const result = await this.db.query(
      `SELECT * FROM tasks 
       WHERE tenant_id = $1 AND assigned_to = $2
       ORDER BY created_at DESC`,
      [tenantId, userId]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
}
```

#### Implementation: Java (Spring Boot)

```java
// infrastructure/repositories/BaseRepository.java
package com.saas.infrastructure.repositories;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

public abstract class BaseRepository<T> {
    protected final JdbcTemplate jdbcTemplate;
    protected final String tableName;
    
    protected BaseRepository(JdbcTemplate jdbcTemplate, String tableName) {
        this.jdbcTemplate = jdbcTemplate;
        this.tableName = tableName;
    }
    
    // Generic CRUD operations
    public Optional<T> findById(UUID tenantId, UUID id) {
        String sql = String.format(
            "SELECT * FROM %s WHERE id = ? AND tenant_id = ?",
            tableName
        );
        
        List<T> results = jdbcTemplate.query(
            sql,
            getRowMapper(),
            id,
            tenantId
        );
        
        return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
    }
    
    public List<T> findAll(UUID tenantId) {
        String sql = String.format(
            "SELECT * FROM %s WHERE tenant_id = ? ORDER BY created_at DESC",
            tableName
        );
        
        return jdbcTemplate.query(sql, getRowMapper(), tenantId);
    }
    
    public T save(T entity) {
        if (exists(getTenantId(entity), getId(entity))) {
            return update(entity);
        } else {
            return insert(entity);
        }
    }
    
    public boolean delete(UUID tenantId, UUID id) {
        String sql = String.format(
            "DELETE FROM %s WHERE id = ? AND tenant_id = ?",
            tableName
        );
        
        int rowsAffected = jdbcTemplate.update(sql, id, tenantId);
        return rowsAffected > 0;
    }
    
    public boolean exists(UUID tenantId, UUID id) {
        String sql = String.format(
            "SELECT EXISTS(SELECT 1 FROM %s WHERE id = ? AND tenant_id = ?)",
            tableName
        );
        
        Boolean exists = jdbcTemplate.queryForObject(
            sql,
            Boolean.class,
            id,
            tenantId
        );
        
        return exists != null && exists;
    }
    
    public long count(UUID tenantId) {
        String sql = String.format(
            "SELECT COUNT(*) FROM %s WHERE tenant_id = ?",
            tableName
        );
        
        Long count = jdbcTemplate.queryForObject(sql, Long.class, tenantId);
        return count != null ? count : 0;
    }
    
    // Subclasses must implement these
    protected abstract RowMapper<T> getRowMapper();
    protected abstract T insert(T entity);
    protected abstract T update(T entity);
    protected abstract UUID getId(T entity);
    protected abstract UUID getTenantId(T entity);
}

// Specific repository
// infrastructure/repositories/PostgresProjectRepository.java
@Repository
public class PostgresProjectRepository 
    extends BaseRepository<Project> 
    implements IProjectRepository {
    
    public PostgresProjectRepository(JdbcTemplate jdbcTemplate) {
        super(jdbcTemplate, "projects");
    }
    
    @Override
    protected RowMapper<Project> getRowMapper() {
        return (rs, rowNum) -> new Project(
            UUID.fromString(rs.getString("id")),
            UUID.fromString(rs.getString("tenant_id")),
            rs.getString("name"),
            rs.getString("description"),
            ProjectStatus.valueOf(rs.getString("status")),
            rs.getDouble("budget"),
            rs.getTimestamp("start_date").toLocalDateTime(),
            rs.getTimestamp("end_date").toLocalDateTime(),
            UUID.fromString(rs.getString("created_by")),
            rs.getTimestamp("created_at").toLocalDateTime(),
            rs.getTimestamp("updated_at").toLocalDateTime()
        );
    }
    
    @Override
    protected Project insert(Project project) {
        String sql = """
            INSERT INTO projects (
                id, tenant_id, name, description, status, budget,
                start_date, end_date, created_by, created_at, updated_at
            )
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """;
        
        jdbcTemplate.update(
            sql,
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
    
    @Override
    protected Project update(Project project) {
        String sql = """
            UPDATE projects 
            SET name = ?, description = ?, status = ?,
                budget = ?, start_date = ?, end_date = ?,
                updated_at = ?
            WHERE id = ? AND tenant_id = ?
            """;
        
        jdbcTemplate.update(
            sql,
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
        
        return project;
    }
    
    @Override
    protected UUID getId(Project entity) {
        return entity.getId();
    }
    
    @Override
    protected UUID getTenantId(Project entity) {
        return entity.getTenantId();
    }
    
    // Entity-specific methods
    @Override
    public List<Project> findByStatus(UUID tenantId, ProjectStatus status) {
        String sql = """
            SELECT * FROM projects 
            WHERE tenant_id = ? AND status = ?
            ORDER BY created_at DESC
            """;
        
        return jdbcTemplate.query(sql, getRowMapper(), tenantId, status.name());
    }
    
    @Override
    public List<Project> search(UUID tenantId, String searchTerm) {
        String sql = """
            SELECT * FROM projects 
            WHERE tenant_id = ? 
              AND (name ILIKE ? OR description ILIKE ?)
            ORDER BY created_at DESC
            """;
        
        String pattern = "%" + searchTerm + "%";
        return jdbcTemplate.query(
            sql,
            getRowMapper(),
            tenantId,
            pattern,
            pattern
        );
    }
}
```

#### Implementation: Go

```go
// infrastructure/repositories/base_repository.go
package repositories

import (
    "context"
    "database/sql"
    "fmt"
)

// BaseRepository provides common CRUD operations
type BaseRepository struct {
    db        *sql.DB
    tableName string
}

// NewBaseRepository creates a new base repository
func NewBaseRepository(db *sql.DB, tableName string) *BaseRepository {
    return &BaseRepository{
        db:        db,
        tableName: tableName,
    }
}

// Exists checks if a record exists
func (r *BaseRepository) Exists(ctx context.Context, tenantID, id string) (bool, error) {
    query := fmt.Sprintf(`
        SELECT EXISTS(
            SELECT 1 FROM %s 
            WHERE id = $1 AND tenant_id = $2
        )
    `, r.tableName)
    
    var exists bool
    err := r.db.QueryRowContext(ctx, query, id, tenantID).Scan(&exists)
    if err != nil {
        return false, err
    }
    
    return exists, nil
}

// Count counts records for a tenant
func (r *BaseRepository) Count(ctx context.Context, tenantID string) (int, error) {
    query := fmt.Sprintf(`
        SELECT COUNT(*) FROM %s WHERE tenant_id = $1
    `, r.tableName)
    
    var count int
    err := r.db.QueryRowContext(ctx, query, tenantID).Scan(&count)
    if err != nil {
        return 0, err
    }
    
    return count, nil
}

// Delete deletes a record
func (r *BaseRepository) Delete(ctx context.Context, tenantID, id string) error {
    query := fmt.Sprintf(`
        DELETE FROM %s WHERE id = $1 AND tenant_id = $2
    `, r.tableName)
    
    result, err := r.db.ExecContext(ctx, query, id, tenantID)
    if err != nil {
        return err
    }
    
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        return fmt.Errorf("record not found")
    }
    
    return nil
}

// Specific repository embeds base repository
// infrastructure/repositories/postgres_project_repository.go
type PostgresProjectRepository struct {
    *BaseRepository
}

func NewPostgresProjectRepository(db *sql.DB) *PostgresProjectRepository {
    return &PostgresProjectRepository{
        BaseRepository: NewBaseRepository(db, "projects"),
    }
}

// FindByID finds a project by ID
func (r *PostgresProjectRepository) FindByID(
    ctx context.Context,
    tenantID, projectID string,
) (*entities.Project, error) {
    query := `
        SELECT id, tenant_id, name, description, status, budget,
               start_date, end_date, created_by, created_at, updated_at
        FROM projects
        WHERE id = $1 AND tenant_id = $2
    `
    
    var project entities.Project
    var startDate, endDate sql.NullTime
    var budget sql.NullFloat64
    
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
        return nil, fmt.Errorf("project not found")
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

// FindAll finds all projects for a tenant
func (r *PostgresProjectRepository) FindAll(
    ctx context.Context,
    tenantID string,
) ([]*entities.Project, error) {
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
        var project entities.Project
        var startDate, endDate sql.NullTime
        var budget sql.NullFloat64
        
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
        
        projects = append(projects, &project)
    }
    
    return projects, nil
}

// Save inserts or updates a project
func (r *PostgresProjectRepository) Save(
    ctx context.Context,
    project *entities.Project,
) error {
    exists, err := r.Exists(ctx, project.TenantID, project.ID)
    if err != nil {
        return err
    }
    
    if exists {
        return r.update(ctx, project)
    }
    return r.insert(ctx, project)
}

func (r *PostgresProjectRepository) insert(
    ctx context.Context,
    project *entities.Project,
) error {
    query := `
        INSERT INTO projects (
            id, tenant_id, name, description, status, budget,
            start_date, end_date, created_by, created_at, updated_at
        )
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
    `
    
    _, err := r.db.ExecContext(
        ctx,
        query,
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

func (r *PostgresProjectRepository) update(
    ctx context.Context,
    project *entities.Project,
) error {
    query := `
        UPDATE projects
        SET name = $1, description = $2, status = $3,
            budget = $4, start_date = $5, end_date = $6,
            updated_at = $7
        WHERE id = $8 AND tenant_id = $9
    `
    
    _, err := r.db.ExecContext(
        ctx,
        query,
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
    
    return err
}

// FindByStatus finds projects by status
func (r *PostgresProjectRepository) FindByStatus(
    ctx context.Context,
    tenantID, status string,
) ([]*entities.Project, error) {
    query := `
        SELECT id, tenant_id, name, description, status, budget,
               start_date, end_date, created_by, created_at, updated_at
        FROM projects
        WHERE tenant_id = $1 AND status = $2
        ORDER BY created_at DESC
    `
    
    rows, err := r.db.QueryContext(ctx, query, tenantID, status)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    // ... scan logic similar to FindAll
    return scanProjects(rows)
}

// Search searches projects
func (r *PostgresProjectRepository) Search(
    ctx context.Context,
    tenantID, searchTerm string,
) ([]*entities.Project, error) {
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
    
    return scanProjects(rows)
}

// Helper function to scan rows into projects
func scanProjects(rows *sql.Rows) ([]*entities.Project, error) {
    var projects []*entities.Project
    
    for rows.Next() {
        var project entities.Project
        var startDate, endDate sql.NullTime
        var budget sql.NullFloat64
        
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
        
        projects = append(projects, &project)
    }
    
    return projects, rows.Err()
}
```

### Benefits of Generic Repository

✅ **Reduces Code Duplication** - Write common code once\
✅ **Consistent Behavior** - All repositories work the same way\
✅ **Easier Maintenance** - Fix bugs in one place\
✅ **Faster Development** - New repositories are quick to create\
✅ **Type Safety** - Generics ensure type correctness

### When to Use Generic Repository

**✅ Use When:**

* Multiple entities share similar CRUD operations
* Want consistent repository behavior
* Team follows strict conventions

**❌ Don't Use When:**

* Entities have very different access patterns
* Over-abstraction makes code harder to understand
* Specific repositories need heavy customization

## Pattern 7.2: Specification Pattern

### The Problem

Building complex queries with multiple optional filters becomes messy:

```typescript
// ❌ BAD: Query builder gets out of control
async function findProjects(filters: any) {
  let query = 'SELECT * FROM projects WHERE tenant_id = $1';
  const params = [tenantId];
  let paramIndex = 2;
  
  if (filters.status) {
    query += ` AND status = ${paramIndex}`;
    params.push(filters.status);
    paramIndex++;
  }
  
  if (filters.minBudget) {
    query += ` AND budget >= ${paramIndex}`;
    params.push(filters.minBudget);
    paramIndex++;
  }
  
  if (filters.maxBudget) {
    query += ` AND budget <= ${paramIndex}`;
    params.push(filters.maxBudget);
    paramIndex++;
  }
  
  if (filters.createdAfter) {
    query += ` AND created_at >= ${paramIndex}`;
    params.push(filters.createdAfter);
    paramIndex++;
  }
  
  if (filters.createdBy) {
    query += ` AND created_by = ${paramIndex}`;
    params.push(filters.createdBy);
    paramIndex++;
  }
  
  if (filters.search) {
    query += ` AND (name ILIKE ${paramIndex} OR description ILIKE ${paramIndex})`;
    params.push(`%${filters.search}%`);
    paramIndex++;
  }
  
  // This gets unmanageable quickly!
  // What about combining conditions with OR?
  // What about reusing these conditions?
  
  return db.query(query, params);
}
```

### The Solution: Specification Pattern

The **Specification Pattern** encapsulates business rules into reusable, combinable objects.

**Key Concepts:**

* Each specification represents ONE business rule
* Specifications can be combined (AND, OR, NOT)
* Specifications are reusable and testable
* Clean separation of query logic

### Implementation: TypeScript

#### Step 1: Define Specification Interface

```typescript
// domain/specifications/ISpecification.ts
export interface ISpecification<T> {
  // Check if entity satisfies the specification (in-memory)
  isSatisfiedBy(entity: T): boolean;
  
  // Convert to SQL WHERE clause
  toSql(): { clause: string; params: any[] };
  
  // Combine with other specifications
  and(other: ISpecification<T>): ISpecification<T>;
  or(other: ISpecification<T>): ISpecification<T>;
  not(): ISpecification<T>;
}
```

#### Step 2: Create Base Specification Class

```typescript
// domain/specifications/BaseSpecification.ts
export abstract class BaseSpecification<T> implements ISpecification<T> {
  abstract isSatisfiedBy(entity: T): boolean;
  abstract toSql(): { clause: string; params: any[] };
  
  and(other: ISpecification<T>): ISpecification<T> {
    return new AndSpecification([this, other]);
  }
  
  or(other: ISpecification<T>): ISpecification<T> {
    return new OrSpecification([this, other]);
  }
  
  not(): ISpecification<T> {
    return new NotSpecification(this);
  }
}
```

#### Step 3: Create Composite Specifications

```typescript
// domain/specifications/CompositeSpecifications.ts

// AND Specification
export class AndSpecification<T> extends BaseSpecification<T> {
  constructor(private specifications: ISpecification<T>[]) {
    super();
  }
  
  isSatisfiedBy(entity: T): boolean {
    return this.specifications.every(spec => spec.isSatisfiedBy(entity));
  }
  
  toSql(): { clause: string; params: any[] } {
    const sqlParts = this.specifications.map(spec => spec.toSql());
    const clauses = sqlParts.map(part => `(${part.clause})`).join(' AND ');
    const params = sqlParts.flatMap(part => part.params);
    
    return { clause: clauses, params };
  }
}

// OR Specification
export class OrSpecification<T> extends BaseSpecification<T> {
  constructor(private specifications: ISpecification<T>[]) {
    super();
  }
  
  isSatisfiedBy(entity: T): boolean {
    return this.specifications.some(spec => spec.isSatisfiedBy(entity));
  }
  
  toSql(): { clause: string; params: any[] } {
    const sqlParts = this.specifications.map(spec => spec.toSql());
    const clauses = sqlParts.map(part => `(${part.clause})`).join(' OR ');
    const params = sqlParts.flatMap(part => part.params);
    
    return { clause: clauses, params };
  }
}

// NOT Specification
export class NotSpecification<T> extends BaseSpecification<T> {
  constructor(private specification: ISpecification<T>) {
    super();
  }
  
  isSatisfiedBy(entity: T): boolean {
    return !this.specification.isSatisfiedBy(entity);
  }
  
  toSql(): { clause: string; params: any[] } {
    const sql = this.specification.toSql();
    return {
      clause: `NOT (${sql.clause})`,
      params: sql.params
    };
  }
}
```

#### Step 4: Create Concrete Specifications for Projects

```typescript
// domain/specifications/project/ProjectStatusSpecification.ts
import { BaseSpecification } from '../BaseSpecification';
import { Project } from '../../entities/Project';

export class ProjectStatusSpecification extends BaseSpecification<Project> {
  constructor(private status: 'active' | 'completed' | 'archived') {
    super();
  }
  
  isSatisfiedBy(project: Project): boolean {
    return project.status === this.status;
  }
  
  toSql(): { clause: string; params: any[] } {
    return {
      clause: 'status = ?',
      params: [this.status]
    };
  }
}

// domain/specifications/project/BudgetRangeSpecification.ts
export class BudgetRangeSpecification extends BaseSpecification<Project> {
  constructor(
    private minBudget?: number,
    private maxBudget?: number
  ) {
    super();
  }
  
  isSatisfiedBy(project: Project): boolean {
    if (this.minBudget !== undefined && project.budget < this.minBudget) {
      return false;
    }
    if (this.maxBudget !== undefined && project.budget > this.maxBudget) {
      return false;
    }
    return true;
  }
  
  toSql(): { clause: string; params: any[] } {
    const clauses: string[] = [];
    const params: any[] = [];
    
    if (this.minBudget !== undefined) {
      clauses.push('budget >= ?');
      params.push(this.minBudget);
    }
    
    if (this.maxBudget !== undefined) {
      clauses.push('budget <= ?');
      params.push(this.maxBudget);
    }
    
    return {
      clause: clauses.join(' AND '),
      params
    };
  }
}

// domain/specifications/project/CreatedAfterSpecification.ts
export class CreatedAfterSpecification extends BaseSpecification<Project> {
  constructor(private date: Date) {
    super();
  }
  
  isSatisfiedBy(project: Project): boolean {
    return project.createdAt >= this.date;
  }
  
  toSql(): { clause: string; params: any[] } {
    return {
      clause: 'created_at >= ?',
      params: [this.date]
    };
  }
}

// domain/specifications/project/CreatedBySpecification.ts
export class CreatedBySpecification extends BaseSpecification<Project> {
  constructor(private userId: string) {
    super();
  }
  
  isSatisfiedBy(project: Project): boolean {
    return project.createdBy === this.userId;
  }
  
  toSql(): { clause: string; params: any[] } {
    return {
      clause: 'created_by = ?',
      params: [this.userId]
    };
  }
}

// domain/specifications/project/ProjectSearchSpecification.ts
export class ProjectSearchSpecification extends BaseSpecification<Project> {
  constructor(private searchTerm: string) {
    super();
  }
  
  isSatisfiedBy(project: Project): boolean {
    const term = this.searchTerm.toLowerCase();
    return (
      project.name.toLowerCase().includes(term) ||
      project.description.toLowerCase().includes(term)
    );
  }
  
  toSql(): { clause: string; params: any[] } {
    return {
      clause: '(name ILIKE ? OR description ILIKE ?)',
      params: [`%${this.searchTerm}%`, `%${this.searchTerm}%`]
    };
  }
}

// domain/specifications/project/OverdueProjectSpecification.ts
export class OverdueProjectSpecification extends BaseSpecification<Project> {
  isSatisfiedBy(project: Project): boolean {
    return (
      project.status === 'active' &&
      project.endDate !== null &&
      project.endDate < new Date()
    );
  }
  
  toSql(): { clause: string; params: any[] } {
    return {
      clause: "status = 'active' AND end_date < NOW()",
      params: []
    };
  }
}

// domain/specifications/project/HighBudgetProjectSpecification.ts
export class HighBudgetProjectSpecification extends BaseSpecification<Project> {
  constructor(private threshold: number = 100000) {
    super();
  }
  
  isSatisfiedBy(project: Project): boolean {
    return project.budget !== null && project.budget >= this.threshold;
  }
  
  toSql(): { clause: string; params: any[] } {
    return {
      clause: 'budget >= ?',
      params: [this.threshold]
    };
  }
}
```

#### Step 5: Update Repository to Use Specifications

```typescript
// domain/repositories/IProjectRepository.ts
export interface IProjectRepository {
  // ... existing methods ...
  
  // Add specification-based query method
  findBySpecification(
    tenantId: string,
    specification: ISpecification<Project>
  ): Promise<Project[]>;
  
  countBySpecification(
    tenantId: string,
    specification: ISpecification<Project>
  ): Promise<number>;
}

// infrastructure/repositories/PostgresProjectRepository.ts
export class PostgresProjectRepository implements IProjectRepository {
  constructor(private db: Pool) {}
  
  async findBySpecification(
    tenantId: string,
    specification: ISpecification<Project>
  ): Promise<Project[]> {
    const { clause, params } = specification.toSql();
    
    // Always include tenant_id
    const query = `
      SELECT * FROM projects
      WHERE tenant_id = $1 AND (${clause})
      ORDER BY created_at DESC
    `;
    
    // Adjust parameter placeholders ($1, $2, etc.)
    const adjustedClause = this.adjustParameterPlaceholders(clause, 1);
    const finalQuery = `
      SELECT * FROM projects
      WHERE tenant_id = $1 AND (${adjustedClause})
      ORDER BY created_at DESC
    `;
    
    const result = await this.db.query(finalQuery, [tenantId, ...params]);
    return result.rows.map(row => this.mapToDomain(row));
  }
  
  async countBySpecification(
    tenantId: string,
    specification: ISpecification<Project>
  ): Promise<number> {
    const { clause, params } = specification.toSql();
    const adjustedClause = this.adjustParameterPlaceholders(clause, 1);
    
    const query = `
      SELECT COUNT(*) as count FROM projects
      WHERE tenant_id = $1 AND (${adjustedClause})
    `;
    
    const result = await this.db.query(query, [tenantId, ...params]);
    return parseInt(result.rows[0].count);
  }
  
  // Helper to adjust ? to $1, $2, etc. for PostgreSQL
  private adjustParameterPlaceholders(clause: string, startIndex: number): string {
    let index = startIndex;
    return clause.replace(/\?/g, () => `${++index}`);
  }
  
  // ... other methods ...
}
```

#### Step 6: Using Specifications

```typescript
// application/services/ProjectService.ts

// Example 1: Simple specification
async findActiveProjects(tenantId: string): Promise<Project[]> {
  const spec = new ProjectStatusSpecification('active');
  return this.projectRepository.findBySpecification(tenantId, spec);
}

// Example 2: Combined specifications with AND
async findHighBudgetActiveProjects(tenantId: string): Promise<Project[]> {
  const spec = new ProjectStatusSpecification('active')
    .and(new HighBudgetProjectSpecification(100000));
  
  return this.projectRepository.findBySpecification(tenantId, spec);
}

// Example 3: Complex combination
async findRecentHighValueProjects(
  tenantId: string,
  userId: string
): Promise<Project[]> {
  const thirtyDaysAgo = new Date();
  thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);
  
  const spec = new AndSpecification([
    new ProjectStatusSpecification('active'),
    new CreatedAfterSpecification(thirtyDaysAgo),
    new HighBudgetProjectSpecification(50000),
    new CreatedBySpecification(userId)
  ]);
  
  return this.projectRepository.findBySpecification(tenantId, spec);
}

// Example 4: OR combination
async findProjectsNeedingAttention(tenantId: string): Promise<Project[]> {
  // Find projects that are either overdue OR high budget
  const spec = new OverdueProjectSpecification()
    .or(new HighBudgetProjectSpecification(100000));
  
  return this.projectRepository.findBySpecification(tenantId, spec);
}

// Example 5: Complex business rule
async findCriticalProjects(tenantId: string): Promise<Project[]> {
  // (High budget AND active) OR overdue
  const highBudgetAndActive = new HighBudgetProjectSpecification(100000)
    .and(new ProjectStatusSpecification('active'));
  
  const spec = highBudgetAndActive.or(new OverdueProjectSpecification());
  
  return this.projectRepository.findBySpecification(tenantId, spec);
}

// Example 6: Search with filters
async searchProjects(
  tenantId: string,
  searchTerm: string,
  filters: {
    status?: 'active' | 'completed' | 'archived';
    minBudget?: number;
    maxBudget?: number;
    createdBy?: string;
  }
): Promise<Project[]> {
  let spec: ISpecification<Project> = new ProjectSearchSpecification(searchTerm);
  
  if (filters.status) {
    spec = spec.and(new ProjectStatusSpecification(filters.status));
  }
  
  if (filters.minBudget || filters.maxBudget) {
    spec = spec.and(new BudgetRangeSpecification(filters.minBudget, filters.maxBudget));
  }
  
  if (filters.createdBy) {
    spec = spec.and(new CreatedBySpecification(filters.createdBy));
  }
  
  return this.projectRepository.findBySpecification(tenantId, spec);
}

// Example 7: NOT specification
async findNonArchivedProjects(tenantId: string): Promise<Project[]> {
  const spec = new ProjectStatusSpecification('archived').not();
  return this.projectRepository.findBySpecification(tenantId, spec);
}

// Example 8: Reusable specifications in multiple queries
private getActiveHighBudgetSpec(): ISpecification<Project> {
  return new ProjectStatusSpecification('active')
    .and(new HighBudgetProjectSpecification(100000));
}

async countActiveHighBudgetProjects(tenantId: string): Promise<number> {
  return this.projectRepository.countBySpecification(
    tenantId,
    this.getActiveHighBudgetSpec()
  );
}

async findActiveHighBudgetProjects(tenantId: string): Promise<Project[]> {
  return this.projectRepository.findBySpecification(
    tenantId,
    this.getActiveHighBudgetSpec()
  );
}
```

### Testing Specifications

```typescript
// tests/specifications/ProjectStatusSpecification.test.ts
import { ProjectStatusSpecification } from '../../domain/specifications/project/ProjectStatusSpecification';
import { Project } from '../../domain/entities/Project';

describe('ProjectStatusSpecification', () => {
  it('should satisfy active projects', () => {
    const spec = new ProjectStatusSpecification('active');
    
    const activeProject = new Project(
      '1',
      'tenant-1',
      'Project A',
      'Description',
      'active',
      null,
      null,
      null,
      'user-1',
      new Date(),
      new Date()
    );
    
    expect(spec.isSatisfiedBy(activeProject)).toBe(true);
  });
  
  it('should not satisfy completed projects when looking for active', () => {
    const spec = new ProjectStatusSpecification('active');
    
    const completedProject = new Project(
      '1',
      'tenant-1',
      'Project A',
      'Description',
      'completed',
      null,
      null,
      null,
      'user-1',
      new Date(),
      new Date()
    );
    
    expect(spec.isSatisfiedBy(completedProject)).toBe(false);
  });
  
  it('should generate correct SQL', () => {
    const spec = new ProjectStatusSpecification('active');
    const { clause, params } = spec.toSql();
    
    expect(clause).toBe('status = ?');
    expect(params).toEqual(['active']);
  });
});

// tests/specifications/CompositeSpecifications.test.ts
describe('Composite Specifications', () => {
  it('should combine specifications with AND', () => {
    const spec1 = new ProjectStatusSpecification('active');
    const spec2 = new HighBudgetProjectSpecification(10000);
    
    const combined = spec1.and(spec2);
    const { clause, params } = combined.toSql();
    
    expect(clause).toContain('status = ?');
    expect(clause).toContain('AND');
    expect(clause).toContain('budget >= ?');
    expect(params).toEqual(['active', 10000]);
  });
  
  it('should combine specifications with OR', () => {
    const spec1 = new ProjectStatusSpecification('active');
    const spec2 = new ProjectStatusSpecification('completed');
    
    const combined = spec1.or(spec2);
    const { clause, params } = combined.toSql();
    
    expect(clause).toContain('OR');
    expect(params).toEqual(['active', 'completed']);
  });
  
  it('should negate specification with NOT', () => {
    const spec = new ProjectStatusSpecification('archived');
    const negated = spec.not();
    
    const { clause, params } = negated.toSql();
    
    expect(clause).toContain('NOT');
    expect(clause).toContain('status = ?');
    expect(params).toEqual(['archived']);
  });
});
```

### Benefits of Specification Pattern

✅ **Reusable Business Rules** - Define once, use everywhere\
✅ **Composable** - Combine specifications with AND, OR, NOT\
✅ **Testable** - Each specification is independently testable\
✅ **Type-Safe** - Compile-time checking of specifications\
✅ **Readable** - Business rules are explicit and clear\
✅ **Maintainable** - Change rule in one place\
✅ **Works In-Memory** - Can filter collections without database

### When to Use Specification Pattern

**✅ Use When:**

* Complex query logic with many combinations
* Business rules need to be reused
* Need to filter both database and in-memory collections
* Query logic changes frequently
* Want to unit test query logic

**❌ Don't Use When:**

* Simple queries (overkill)
* Queries are unique and never reused
* Performance is critical (adds abstraction overhead)
* Team unfamiliar with pattern

### Real-World SaaS Use Cases

**1. Permission Checking:**

```typescript
const canEditSpec = new IsOwnerSpecification(userId)
  .or(new HasRoleSpecification('admin'))
  .or(new IsCollaboratorSpecification(userId));

if (canEditSpec.isSatisfiedBy(project)) {
  // Allow edit
}
```

\*\*2. Reporting:# Module 2.5: Repository Pattern - Step 7

**Advanced Repository Patterns**\
**Duration:** Remaining Day 2-3\
**Prerequisites:** Steps 1-6 completed (Basic Repository implementation in all three languages)

***

## Step 7: Advanced Repository Patterns

Now that you understand the basic Repository pattern, let's explore advanced patterns that solve specific problems in production SaaS applications.

### Overview of Advanced Patterns

We'll cover:

1. **Generic Repository** - Reduce code duplication across entities
2. **Specification Pattern** - Build complex queries dynamically
3. **Unit of Work Pattern** - Manage transactions across multiple repositories
4. **Cached Repository** - Add caching layer transparently
5. **Read/Write Repository Separation** - CQRS-lite pattern

***

## Pattern 7.1: Generic Repository

### The Problem

You're repeating the same CRUD code for every entity:

```typescript
// ❌ BAD: Repeating code for every entity
class ProjectRepository {
  async findById(id: string) { /* ... */ }
  async findAll() { /* ... */ }
  async save(entity) { /* ... */ }
  async delete(id: string) { /* ... */ }
}

class TaskRepository {
  async findById(id: string) { /* ... */ }  // Same code!
  async findAll() { /* ... */ }              // Same code!
  async save(entity) { /* ... */ }           // Same code!
  async delete(id: string) { /* ... */ }     // Same code!
}

class UserRepository {
  async findById(id: string) { /* ... */ }  // Same code again!
  async findAll() { /* ... */ }
  async save(entity) { /* ... */ }
  async delete(id: string) { /* ... */ }
}
```

### The Solution: Generic Base Repository

#### Implementation: TypeScript

```typescript
// infrastructure/repositories/BaseRepository.ts
import { Pool } from 'pg';

export abstract class BaseRepository<T> {
  constructor(
    protected db: Pool,
    protected tableName: string
  ) {}
  
  // Generic CRUD operations
  async findById(tenantId: string, id: string): Promise<T | null> {
    const result = await this.db.query(
      `SELECT * FROM ${this.tableName} 
       WHERE id = $1 AND tenant_id = $2`,
      [id, tenantId]
    );
    
    if (result.rows.length === 0) {
      return null;
    }
    
    return this.mapToDomain(result.rows[0]);
  }
  
  async findAll(tenantId: string): Promise<T[]> {
    const result = await this.db.query(
      `SELECT * FROM ${this.tableName} 
       WHERE tenant_id = $1 
       ORDER BY created_at DESC`,
      [tenantId]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
  
  async save(entity: any): Promise<T> {
    const exists = await this.exists(entity.tenantId, entity.id);
    
    if (exists) {
      return this.update(entity);
    } else {
      return this.insert(entity);
    }
  }
  
  async delete(tenantId: string, id: string): Promise<boolean> {
    const result = await this.db.query(
      `DELETE FROM ${this.tableName} 
       WHERE id = $1 AND tenant_id = $2`,
      [id, tenantId]
    );
    
    return result.rowCount > 0;
  }
  
  async exists(tenantId: string, id: string): Promise<boolean> {
    const result = await this.db.query(
      `SELECT EXISTS(
        SELECT 1 FROM ${this.tableName} 
        WHERE id = $1 AND tenant_id = $2
      ) as exists`,
      [id, tenantId]
    );
    
    return result.rows[0].exists;
  }
  
  async count(tenantId: string): Promise<number> {
    const result = await this.db.query(
      `SELECT COUNT(*) as count FROM ${this.tableName} 
       WHERE tenant_id = $1`,
      [tenantId]
    );
    
    return parseInt(result.rows[0].count);
  }
  
  // Subclasses must implement these
  protected abstract mapToDomain(row: any): T;
  protected abstract insert(entity: any): Promise<T>;
  protected abstract update(entity: any): Promise<T>;
}

// Now specific repositories are much simpler!
// infrastructure/repositories/PostgresProjectRepository.ts
export class PostgresProjectRepository 
  extends BaseRepository<Project> 
  implements IProjectRepository {
  
  constructor(db: Pool) {
    super(db, 'projects');
  }
  
  // Only implement entity-specific logic
  protected mapToDomain(row: any): Project {
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
  
  protected async insert(project: Project): Promise<Project> {
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
    
    return this.mapToDomain(result.rows[0]);
  }
  
  protected async update(project: Project): Promise<Project> {
    const result = await this.db.query(
      `UPDATE projects 
       SET name = $1, description = $2, status = $3, 
           budget = $4, start_date = $5, end_date = $6,
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
    
    return this.mapToDomain(result.rows[0]);
  }
  
  // Add entity-specific methods
  async findByStatus(tenantId: string, status: string): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 AND status = $2
       ORDER BY created_at DESC`,
      [tenantId, status]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
  
  async search(tenantId: string, searchTerm: string): Promise<Project[]> {
    const result = await this.db.query(
      `SELECT * FROM projects 
       WHERE tenant_id = $1 
         AND (name ILIKE $2 OR description ILIKE $2)
       ORDER BY created_at DESC`,
      [tenantId, `%${searchTerm}%`]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
}

// Task repository is now much simpler too!
export class PostgresTaskRepository 
  extends BaseRepository<Task> 
  implements ITaskRepository {
  
  constructor(db: Pool) {
    super(db, 'tasks');
  }
  
  protected mapToDomain(row: any): Task {
    return new Task(
      row.id,
      row.tenant_id,
      row.project_id,
      row.title,
      row.description,
      row.status,
      row.assigned_to,
      row.created_by,
      row.created_at,
      row.updated_at
    );
  }
  
  protected async insert(task: Task): Promise<Task> {
    // Task-specific insert logic
    // ...
  }
  
  protected async update(task: Task): Promise<Task> {
    // Task-specific update logic
    // ...
  }
  
  // Task-specific methods
  async findByProject(tenantId: string, projectId: string): Promise<Task[]> {
    const result = await this.db.query(
      `SELECT * FROM tasks 
       WHERE tenant_id = $1 AND project_id = $2
       ORDER BY created_at DESC`,
      [tenantId, projectId]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
  
  async findByAssignee(tenantId: string, userId: string): Promise<Task[]> {
    const result = await this.db.query(
      `SELECT * FROM tasks 
       WHERE tenant_id = $1 AND assigned_to = $2
       ORDER BY created_at DESC`,
      [tenantId, userId]
    );
    
    return result.rows.map(row => this.mapToDomain(row));
  }
}
```

#### Implementation: Java (Spring Boot)

```java
// infrastructure/repositories/BaseRepository.java
package com.saas.infrastructure.repositories;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

public abstract class BaseRepository<T> {
    protected final JdbcTemplate jdbcTemplate;
    protected final String tableName;
    
    protected BaseRepository(JdbcTemplate jdbcTemplate, String tableName) {
        this.jdbcTemplate = jdbcTemplate;
        this.tableName = tableName;
    }
    
    // Generic CRUD operations
    public Optional<T> findById(UUID tenantId, UUID id) {
        String sql = String.format(
            "SELECT * FROM %s WHERE id = ? AND tenant_id = ?",
            tableName
        );
        
        List<T> results = jdbcTemplate.query(
            sql,
            getRowMapper(),
            id,
            tenantId
        );
        
        return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
    }
    
    public List<T> findAll(UUID tenantId) {
        String sql = String.format(
            "SELECT * FROM %s WHERE tenant_id = ? ORDER BY created_at DESC",
            tableName
        );
        
        return jdbcTemplate.query(sql, getRowMapper(), tenantId);
    }
    
    public T save(T entity) {
        if (exists(getTenantId(entity), getId(entity))) {
            return update(entity);
        } else {
            return insert(entity);
        }
    }
    
    public boolean delete(UUID tenantId, UUID id) {
        String sql = String.format(
            "DELETE FROM %s WHERE id = ? AND tenant_id = ?",
            tableName
        );
        
        int rowsAffected = jdbcTemplate.update(sql, id, tenantId);
        return rowsAffected > 0;
    }
    
    public boolean exists(UUID tenantId, UUID id) {
        String sql = String.format(
            "SELECT EXISTS(SELECT 1 FROM %s WHERE id = ? AND tenant_id = ?)",
            tableName
        );
        
        Boolean exists = jdbcTemplate.queryForObject(
            sql,
            Boolean.class,
            id,
            tenantId
        );
        
        return exists != null && exists;
    }
    
    public long count(UUID tenantId) {
        String sql = String.format(
            "SELECT COUNT(*) FROM %s WHERE tenant_id = ?",
            tableName
        );
        
        Long count = jdbcTemplate.queryForObject(sql, Long.class, tenantId);
        return count != null ? count : 0;
    }
    
    // Subclasses must implement these
    protected abstract RowMapper<T> getRowMapper();
    protected abstract T insert(T entity);
    protected abstract T update(T entity);
    protected abstract UUID getId(T entity);
    protected abstract UUID getTenantId(T entity);
}

// Specific repository
// infrastructure/repositories/PostgresProjectRepository.java
@Repository
public class PostgresProjectRepository 
    extends BaseRepository<Project> 
    implements IProjectRepository {
    
    public PostgresProjectRepository(JdbcTemplate jdbcTemplate) {
        super(jdbcTemplate, "projects");
    }
    
    @Override
    protected RowMapper<Project> getRowMapper() {
        return (rs, rowNum) -> new Project(
            UUID.fromString(rs.getString("id")),
            UUID.fromString(rs.getString("tenant_id")),
            rs.getString("name"),
            rs.getString("description"),
            ProjectStatus.valueOf(rs.getString("status")),
            rs.getDouble("budget"),
            rs.getTimestamp("start_date").toLocalDateTime(),
            rs.getTimestamp("end_date").toLocalDateTime(),
            UUID.fromString(rs.getString("created_by")),
            rs.getTimestamp("created_at").toLocalDateTime(),
            rs.getTimestamp("updated_at").toLocalDateTime()
        );
    }
    
    @Override
    protected Project insert(Project project) {
        String sql = """
            INSERT INTO projects (
                id, tenant_id, name, description, status, budget,
                start_date, end_date, created_by, created_at, updated_at
            )
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """;
        
        jdbcTemplate.update(
            sql,
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
    
    @Override
    protected Project update(Project project) {
        String sql = """
            UPDATE projects 
            SET name = ?, description = ?, status = ?,
                budget = ?, start_date = ?, end_date = ?,
                updated_at = ?
            WHERE id = ? AND tenant_id = ?
            """;
        
        jdbcTemplate.update(
            sql,
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
        
        return project;
    }
    
    @Override
    protected UUID getId(Project entity) {
        return entity.getId();
    }
    
    @Override
    protected UUID getTenantId(Project entity) {
        return entity.getTenantId();
    }
    
    // Entity-specific methods
    @Override
    public List<Project> findByStatus(UUID tenantId, ProjectStatus status) {
        String sql = """
            SELECT * FROM projects 
            WHERE tenant_id = ? AND status = ?
            ORDER BY created_at DESC
            """;
        
        return jdbcTemplate.query(sql, getRowMapper(), tenantId, status.name());
    }
    
    @Override
    public List<Project> search(UUID tenantId, String searchTerm) {
        String sql = """
            SELECT * FROM projects 
            WHERE tenant_id = ? 
              AND (name ILIKE ? OR description ILIKE ?)
            ORDER BY created_at DESC
            """;
        
        String pattern = "%" + searchTerm + "%";
        return jdbcTemplate.query(
            sql,
            getRowMapper(),
            tenantId,
            pattern,
            pattern
        );
    }
}
```

#### Implementation: Go

```go
// infrastructure/repositories/base_repository.go
package repositories

import (
    "context"
    "database/sql"
    "fmt"
)

// BaseRepository provides common CRUD operations
type BaseRepository struct {
    db        *sql.DB
    tableName string
}

// NewBaseRepository creates a new base repository
func NewBaseRepository(db *sql.DB, tableName string) *BaseRepository {
    return &BaseRepository{
        db:        db,
        tableName: tableName,
    }
}

// Exists checks if a record exists
func (r *BaseRepository) Exists(ctx context.Context, tenantID, id string) (bool, error) {
    query := fmt.Sprintf(`
        SELECT EXISTS(
            SELECT 1 FROM %s 
            WHERE id = $1 AND tenant_id = $2
        )
    `, r.tableName)
    
    var exists bool
    err := r.db.QueryRowContext(ctx, query, id, tenantID).Scan(&exists)
    if err != nil {
        return false, err
    }
    
    return exists, nil
}

// Count counts records for a tenant
func (r *BaseRepository) Count(ctx context.Context, tenantID string) (int, error) {
    query := fmt.Sprintf(`
        SELECT COUNT(*) FROM %s WHERE tenant_id = $1
    `, r.tableName)
    
    var count int
    err := r.db.QueryRowContext(ctx, query, tenantID).Scan(&count)
    if err != nil {
        return 0, err
    }
    
    return count, nil
}

// Delete deletes a record
func (r *BaseRepository) Delete(ctx context.Context, tenantID, id string) error {
    query := fmt.Sprintf(`
        DELETE FROM %s WHERE id = $1 AND tenant_id = $2
    `, r.tableName)
    
    result, err := r.db.ExecContext(ctx, query, id, tenantID)
    if err != nil {
        return err
    }
    
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        return fmt.Errorf("record not found")
    }
    
    return nil
}

// Specific repository embeds base repository
// infrastructure/repositories/postgres_project_repository.go
type PostgresProjectRepository struct {
    *BaseRepository
}

func NewPostgresProjectRepository(db *sql.DB) *PostgresProjectRepository {
    return &PostgresProjectRepository{
        BaseRepository: NewBaseRepository(db, "projects"),
    }
}

// FindByID finds a project by ID
func (r *PostgresProjectRepository) FindByID(
    ctx context.Context,
    tenantID, projectID string,
) (*entities.Project, error) {
    query := `
        SELECT id, tenant_id, name, description, status, budget,
               start_date, end_date, created_by, created_at, updated_at
        FROM projects
        WHERE id = $1 AND tenant_id = $2
    `
    
    var project entities.Project
    var startDate, endDate sql.NullTime
    var budget sql.NullFloat64
    
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
        return nil, fmt.Errorf("project not found")
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

// FindAll finds all projects for a tenant
func (r *PostgresProjectRepository) FindAll(
    ctx context.Context,
    tenantID string,
) ([]*entities.Project, error) {
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
        var project entities.Project
        var startDate, endDate sql.NullTime
        var budget sql.NullFloat64
        
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
        
        projects = append(projects, &project)
    }
    
    return projects, nil
}

// Save inserts or updates a project
func (r *PostgresProjectRepository) Save(
    ctx context.Context,
    project *entities.Project,
) error {
    exists, err := r.Exists(ctx, project.TenantID, project.ID)
    if err != nil {
        return err
    }
    
    if exists {
        return r.update(ctx, project)
    }
    return r.insert(ctx, project)
}

func (r *PostgresProjectRepository) insert(
    ctx context.Context,
    project *entities.Project,
) error {
    query := `
        INSERT INTO projects (
            id, tenant_id, name, description, status, budget,
            start_date, end_date, created_by, created_at, updated_at
        )
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
    `
    
    _, err := r.db.ExecContext(
        ctx,
        query,
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

func (r *PostgresProjectRepository) update(
    ctx context.Context,
    project *entities.Project,
) error {
    query := `
        UPDATE projects
        SET name = $1, description = $2, status = $3,
            budget = $4, start_date = $5, end_date = $6,
            updated_at = $7
        WHERE id = $8 AND tenant_id = $9
    `
    
    _, err := r.db.ExecContext(
        ctx,
        query,
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
    
    return err
}

// FindByStatus finds projects by status
func (r *PostgresProjectRepository) FindByStatus(
    ctx context.Context,
    tenantID, status string,
) ([]*entities.Project, error) {
    query := `
        SELECT id, tenant_id, name, description, status, budget,
               start_date, end_date, created_by, created_at, updated_at
        FROM projects
        WHERE tenant_id = $1 AND status = $2
        ORDER BY created_at DESC
    `
    
    rows, err := r.db.QueryContext(ctx, query, tenantID, status)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    // ... scan logic similar to FindAll
    return scanProjects(rows)
}

// Search searches projects
func (r *PostgresProjectRepository) Search(
    ctx context.Context,
    tenantID, searchTerm string,
) ([]*entities.Project, error) {
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
    
    return scanProjects(rows)
}

// Helper function to scan rows into projects
func scanProjects(rows *sql.Rows) ([]*entities.Project, error) {
    var projects []*entities.Project
    
    for rows.Next() {
        var project entities.Project
        var startDate, endDate sql.NullTime
        var budget sql.NullFloat64
        
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
        
        projects = append(projects, &project)
    }
    
    return projects, rows.Err()
}
```

### Benefits of Generic Repository

✅ **Reduces Code Duplication** - Write common code once\
✅ **Consistent Behavior** - All repositories work the same way\
✅ **Easier Maintenance** - Fix bugs in one place\
✅ **Faster Development** - New repositories are quick to create\
✅ **Type Safety** - Generics ensure type correctness

### When to Use Generic Repository

**✅ Use When:**

* Multiple entities share similar CRUD operations
* Want consistent repository behavior
* Team follows strict conventions

**❌ Don't Use When:**

* Entities have very different access patterns
* Over-abstraction makes code harder to understand
* Specific repositories need heavy customization

***

## Next: Pattern 7.2 - Specification Pattern

Should I continue with the Specification Pattern for building complex queries dynamically?

