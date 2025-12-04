

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

## Pattern 7.3: Using Existing ORMs and Query Builders

**Instead of building everything from scratch**, you can use mature ORMs that already implement Repository and Specification patterns!

### Option 1: TypeORM (Node.js/TypeScript)

**TypeORM** is a mature ORM with built-in Repository pattern and query builder.

#### Installation
```bash
npm install typeorm reflect-metadata pg
```

#### Entity Definition
```typescript
// domain/entities/Project.entity.ts
import {
  Entity,
  PrimaryColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
} from 'typeorm';

@Entity('projects')
export class ProjectEntity {
  @PrimaryColumn('uuid')
  id: string;
  
  @Column('uuid')
  tenantId: string;
  
  @Column('varchar', { length: 255 })
  name: string;
  
  @Column('text')
  description: string;
  
  @Column('varchar', { length: 50 })
  status: 'active' | 'completed' | 'archived';
  
  @Column('decimal', { nullable: true })
  budget: number;
  
  @Column('timestamp', { nullable: true })
  startDate: Date;
  
  @Column('timestamp', { nullable: true })
  endDate: Date;
  
  @Column('uuid')
  createdBy: string;
  
  @CreateDateColumn()
  createdAt: Date;
  
  @UpdateDateColumn()
  updatedAt: Date;
}
```

#### Repository with TypeORM
```typescript
// infrastructure/repositories/TypeORMProjectRepository.ts
import { Repository, DataSource } from 'typeorm';
import { ProjectEntity } from '../../domain/entities/Project.entity';

export class TypeORMProjectRepository {
  private repository: Repository<ProjectEntity>;
  
  constructor(dataSource: DataSource) {
    this.repository = dataSource.getRepository(ProjectEntity);
  }
  
  // Simple queries
  async findById(tenantId: string, projectId: string): Promise<ProjectEntity | null> {
    return this.repository.findOne({
      where: { id: projectId, tenantId }
    });
  }
  
  async findAll(tenantId: string): Promise<ProjectEntity[]> {
    return this.repository.find({
      where: { tenantId },
      order: { createdAt: 'DESC' }
    });
  }
  
  async save(project: ProjectEntity): Promise<ProjectEntity> {
    return this.repository.save(project);
  }
  
  async delete(tenantId: string, projectId: string): Promise<boolean> {
    const result = await this.repository.delete({ id: projectId, tenantId });
    return result.affected > 0;
  }
  
  // Complex queries with QueryBuilder
  async findBySpecification(
    tenantId: string,
    filters: {
      status?: string;
      minBudget?: number;
      maxBudget?: number;
      createdAfter?: Date;
      createdBy?: string;
      search?: string;
    }
  ): Promise<ProjectEntity[]> {
    const qb = this.repository
      .createQueryBuilder('project')
      .where('project.tenantId = :tenantId', { tenantId });
    
    if (filters.status) {
      qb.andWhere('project.status = :status', { status: filters.status });
    }
    
    if (filters.minBudget !== undefined) {
      qb.andWhere('project.budget >= :minBudget', { minBudget: filters.minBudget });
    }
    
    if (filters.maxBudget !== undefined) {
      qb.andWhere('project.budget <= :maxBudget', { maxBudget: filters.maxBudget });
    }
    
    if (filters.createdAfter) {
      qb.andWhere('project.createdAt >= :createdAfter', { createdAfter: filters.createdAfter });
    }
    
    if (filters.createdBy) {
      qb.andWhere('project.createdBy = :createdBy', { createdBy: filters.createdBy });
    }
    
    if (filters.search) {
      qb.andWhere(
        '(project.name ILIKE :search OR project.description ILIKE :search)',
        { search: `%${filters.search}%` }
      );
    }
    
    return qb.orderBy('project.createdAt', 'DESC').getMany();
  }
  
  // Pagination
  async findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: any
  ) {
    const qb = this.repository
      .createQueryBuilder('project')
      .where('project.tenantId = :tenantId', { tenantId });
    
    // Apply filters...
    
    const [projects, total] = await qb
      .skip((page - 1) * limit)
      .take(limit)
      .getManyAndCount();
    
    return {
      projects,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit)
    };
  }
}
```

---

### Option 2: Prisma (Node.js/TypeScript)

**Prisma** is a modern ORM with excellent TypeScript support and auto-generated types.

#### Installation
```bash
npm install prisma @prisma/client
npx prisma init
```

#### Schema Definition
```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Project {
  id          String    @id @default(uuid())
  tenantId    String    @map("tenant_id")
  name        String    @db.VarChar(255)
  description String    @db.Text
  status      String    @db.VarChar(50)
  budget      Decimal?  @db.Decimal(15, 2)
  startDate   DateTime? @map("start_date")
  endDate     DateTime? @map("end_date")
  createdBy   String    @map("created_by")
  createdAt   DateTime  @default(now()) @map("created_at")
  updatedAt   DateTime  @updatedAt @map("updated_at")
  
  @@index([tenantId])
  @@index([tenantId, status])
  @@map("projects")
}
```

#### Repository with Prisma
```typescript
// infrastructure/repositories/PrismaProjectRepository.ts
import { PrismaClient, Prisma } from '@prisma/client';

export class PrismaProjectRepository {
  constructor(private prisma: PrismaClient) {}
  
  async findById(tenantId: string, projectId: string) {
    return this.prisma.project.findFirst({
      where: {
        id: projectId,
        tenantId: tenantId
      }
    });
  }
  
  async findAll(tenantId: string) {
    return this.prisma.project.findMany({
      where: { tenantId },
      orderBy: { createdAt: 'desc' }
    });
  }
  
  async save(project: Prisma.ProjectCreateInput) {
    return this.prisma.project.upsert({
      where: { id: project.id },
      update: project,
      create: project
    });
  }
  
  async delete(tenantId: string, projectId: string) {
    const result = await this.prisma.project.deleteMany({
      where: {
        id: projectId,
        tenantId: tenantId
      }
    });
    return result.count > 0;
  }
  
  // Complex queries
  async findBySpecification(
    tenantId: string,
    filters: {
      status?: string;
      minBudget?: number;
      maxBudget?: number;
      createdAfter?: Date;
      createdBy?: string;
      search?: string;
    }
  ) {
    const where: Prisma.ProjectWhereInput = {
      tenantId,
      ...(filters.status && { status: filters.status }),
      ...(filters.createdBy && { createdBy: filters.createdBy }),
      ...(filters.createdAfter && { 
        createdAt: { gte: filters.createdAfter } 
      }),
      ...(filters.minBudget !== undefined && { 
        budget: { gte: filters.minBudget } 
      }),
      ...(filters.maxBudget !== undefined && { 
        budget: { lte: filters.maxBudget } 
      }),
      ...(filters.search && {
        OR: [
          { name: { contains: filters.search, mode: 'insensitive' } },
          { description: { contains: filters.search, mode: 'insensitive' } }
        ]
      })
    };
    
    return this.prisma.project.findMany({
      where,
      orderBy: { createdAt: 'desc' }
    });
  }
  
  // Pagination
  async findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: any
  ) {
    const where = { tenantId, ...filters };
    
    const [projects, total] = await Promise.all([
      this.prisma.project.findMany({
        where,
        skip: (page - 1) * limit,
        take: limit,
        orderBy: { createdAt: 'desc' }
      }),
      this.prisma.project.count({ where })
    ]);
    
    return {
      projects,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit)
    };
  }
  
  // Transactions (Unit of Work pattern)
  async createProjectWithTasks(
    project: Prisma.ProjectCreateInput,
    tasks: Prisma.TaskCreateInput[]
  ) {
    return this.prisma.$transaction(async (tx) => {
      const createdProject = await tx.project.create({ data: project });
      
      const createdTasks = await tx.task.createMany({
        data: tasks.map(task => ({
          ...task,
          projectId: createdProject.id
        }))
      });
      
      return { project: createdProject, tasks: createdTasks };
    });
  }
}
```

---

### Option 3: Sequelize (Node.js)

**Sequelize** is a mature, feature-rich ORM for Node.js.

#### Installation
```bash
npm install sequelize pg pg-hstore
```

#### Model Definition
```typescript
// models/Project.model.ts
import { DataTypes, Model, Sequelize } from 'sequelize';

export class Project extends Model {
  public id!: string;
  public tenantId!: string;
  public name!: string;
  public description!: string;
  public status!: string;
  public budget!: number | null;
  public startDate!: Date | null;
  public endDate!: Date | null;
  public createdBy!: string;
  public readonly createdAt!: Date;
  public readonly updatedAt!: Date;
}

export function initProjectModel(sequelize: Sequelize) {
  Project.init(
    {
      id: {
        type: DataTypes.UUID,
        primaryKey: true,
        defaultValue: DataTypes.UUIDV4
      },
      tenantId: {
        type: DataTypes.UUID,
        allowNull: false,
        field: 'tenant_id'
      },
      name: {
        type: DataTypes.STRING(255),
        allowNull: false
      },
      description: {
        type: DataTypes.TEXT,
        allowNull: false
      },
      status: {
        type: DataTypes.STRING(50),
        allowNull: false
      },
      budget: {
        type: DataTypes.DECIMAL(15, 2),
        allowNull: true
      },
      startDate: {
        type: DataTypes.DATE,
        allowNull: true,
        field: 'start_date'
      },
      endDate: {
        type: DataTypes.DATE,
        allowNull: true,
        field: 'end_date'
      },
      createdBy: {
        type: DataTypes.UUID,
        allowNull: false,
        field: 'created_by'
      }
    },
    {
      sequelize,
      tableName: 'projects',
      timestamps: true,
      underscored: true
    }
  );
  
  return Project;
}
```

#### Repository with Sequelize
```typescript
// infrastructure/repositories/SequelizeProjectRepository.ts
import { Op } from 'sequelize';
import { Project } from '../../models/Project.model';

export class SequelizeProjectRepository {
  async findById(tenantId: string, projectId: string) {
    return Project.findOne({
      where: {
        id: projectId,
        tenantId
      }
    });
  }
  
  async findAll(tenantId: string) {
    return Project.findAll({
      where: { tenantId },
      order: [['createdAt', 'DESC']]
    });
  }
  
  async save(projectData: any) {
    const [project, created] = await Project.upsert(projectData);
    return project;
  }
  
  async delete(tenantId: string, projectId: string) {
    const result = await Project.destroy({
      where: {
        id: projectId,
        tenantId
      }
    });
    return result > 0;
  }
  
  // Complex queries with Op
  async findBySpecification(
    tenantId: string,
    filters: {
      status?: string;
      minBudget?: number;
      maxBudget?: number;
      createdAfter?: Date;
      createdBy?: string;
      search?: string;
    }
  ) {
    const where: any = { tenantId };
    
    if (filters.status) {
      where.status = filters.status;
    }
    
    if (filters.minBudget !== undefined || filters.maxBudget !== undefined) {
      where.budget = {};
      if (filters.minBudget !== undefined) {
        where.budget[Op.gte] = filters.minBudget;
      }
      if (filters.maxBudget !== undefined) {
        where.budget[Op.lte] = filters.maxBudget;
      }
    }
    
    if (filters.createdAfter) {
      where.createdAt = { [Op.gte]: filters.createdAfter };
    }
    
    if (filters.createdBy) {
      where.createdBy = filters.createdBy;
    }
    
    if (filters.search) {
      where[Op.or] = [
        { name: { [Op.iLike]: `%${filters.search}%` } },
        { description: { [Op.iLike]: `%${filters.search}%` } }
      ];
    }
    
    return Project.findAll({
      where,
      order: [['createdAt', 'DESC']]
    });
  }
  
  // Pagination
  async findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: any
  ) {
    const where = { tenantId, ...filters };
    
    const { count, rows } = await Project.findAndCountAll({
      where,
      limit,
      offset: (page - 1) * limit,
      order: [['createdAt', 'DESC']]
    });
    
    return {
      projects: rows,
      total: count,
      page,
      limit,
      totalPages: Math.ceil(count / limit)
    };
  }
}
```

---

### Option 4: Spring Data JPA (Java)

**Spring Data JPA** provides Repository pattern out of the box with zero boilerplate!

#### Entity Definition
```java
// domain/entities/Project.java
@Entity
@Table(name = "projects", indexes = {
    @Index(name = "idx_tenant", columnList = "tenant_id"),
    @Index(name = "idx_tenant_status", columnList = "tenant_id,status")
})
public class Project {
    @Id
    @GeneratedValue
    private UUID id;
    
    @Column(name = "tenant_id", nullable = false)
    private UUID tenantId;
    
    @Column(nullable = false, length = 255)
    private String name;
    
    @Column(columnDefinition = "TEXT")
    private String description;
    
    @Column(length = 50)
    private String status;
    
    @Column(precision = 15, scale = 2)
    private BigDecimal budget;
    
    @Column(name = "start_date")
    private LocalDateTime startDate;
    
    @Column(name = "end_date")
    private LocalDateTime endDate;
    
    @Column(name = "created_by", nullable = false)
    private UUID createdBy;
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
    
    // Getters and setters...
}
```

#### Repository with Spring Data JPA
```java
// infrastructure/repositories/ProjectRepository.java
@Repository
public interface ProjectRepository extends JpaRepository<Project, UUID>, 
                                           JpaSpecificationExecutor<Project> {
    
    // Simple queries - just method names!
    Optional<Project> findByIdAndTenantId(UUID id, UUID tenantId);
    
    List<Project> findByTenantIdOrderByCreatedAtDesc(UUID tenantId);
    
    List<Project> findByTenantIdAndStatus(UUID tenantId, String status);
    
    List<Project> findByTenantIdAndCreatedBy(UUID tenantId, UUID createdBy);
    
    Long countByTenantId(UUID tenantId);
    
    Long countByTenantIdAndStatus(UUID tenantId, String status);
    
    void deleteByIdAndTenantId(UUID id, UUID tenantId);
    
    // Custom query with @Query
    @Query("SELECT p FROM Project p WHERE p.tenantId = :tenantId " +
           "AND (LOWER(p.name) LIKE LOWER(CONCAT('%', :search, '%')) " +
           "OR LOWER(p.description) LIKE LOWER(CONCAT('%', :search, '%')))")
    List<Project> search(@Param("tenantId") UUID tenantId, 
                        @Param("search") String search);
    
    // Pagination
    Page<Project> findByTenantId(UUID tenantId, Pageable pageable);
}
```

#### Using Specifications (JPA Specification Pattern)
```java
// specifications/ProjectSpecifications.java
public class ProjectSpecifications {
    
    public static Specification<Project> hasTenantId(UUID tenantId) {
        return (root, query, cb) -> cb.equal(root.get("tenantId"), tenantId);
    }
    
    public static Specification<Project> hasStatus(String status) {
        return (root, query, cb) -> cb.equal(root.get("status"), status);
    }
    
    public static Specification<Project> budgetGreaterThan(BigDecimal minBudget) {
        return (root, query, cb) -> cb.greaterThanOrEqualTo(root.get("budget"), minBudget);
    }
    
    public static Specification<Project> budgetLessThan(BigDecimal maxBudget) {
        return (root, query, cb) -> cb.lessThanOrEqualTo(root.get("budget"), maxBudget);
    }
    
    public static Specification<Project> createdAfter(LocalDateTime date) {
        return (root, query, cb) -> cb.greaterThanOrEqualTo(root.get("createdAt"), date);
    }
    
    public static Specification<Project> createdBy(UUID userId) {
        return (root, query, cb) -> cb.equal(root.get("createdBy"), userId);
    }
    
    public static Specification<Project> nameOrDescriptionContains(String search) {
        return (root, query, cb) -> {
            String pattern = "%" + search.toLowerCase() + "%";
            return cb.or(
                cb.like(cb.lower(root.get("name")), pattern),
                cb.like(cb.lower(root.get("description")), pattern)
            );
        };
    }
}

// Usage in Service
@Service
public class ProjectService {
    @Autowired
    private ProjectRepository projectRepository;
    
    public List<Project> findComplexQuery(UUID tenantId, ProjectFilters filters) {
        Specification<Project> spec = Specification.where(
            ProjectSpecifications.hasTenantId(tenantId)
        );
        
        if (filters.getStatus() != null) {
            spec = spec.and(ProjectSpecifications.hasStatus(filters.getStatus()));
        }
        
        if (filters.getMinBudget() != null) {
            spec = spec.and(ProjectSpecifications.budgetGreaterThan(filters.getMinBudget()));
        }
        
        if (filters.getMaxBudget() != null) {
            spec = spec.and(ProjectSpecifications.budgetLessThan(filters.getMaxBudget()));
        }
        
        if (filters.getSearch() != null) {
            spec = spec.and(ProjectSpecifications.nameOrDescriptionContains(filters.getSearch()));
        }
        
        return projectRepository.findAll(spec);
    }
}
```

---

### Option 5: GORM (Go)

**GORM** is the most popular ORM for Go.

#### Installation
```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/postgres
```

#### Model Definition
```go
// domain/entities/project.go
type Project struct {
    ID          string     `gorm:"type:uuid;primary_key"`
    TenantID    string     `gorm:"type:uuid;not null;index"`
    Name        string     `gorm:"size:255;not null"`
    Description string     `gorm:"type:text"`
    Status      string     `gorm:"size:50"`
    Budget      *float64   `gorm:"type:decimal(15,2)"`
    StartDate   *time.Time
    EndDate     *time.Time
    CreatedBy   string     `gorm:"type:uuid;not null"`
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

func (Project) TableName() string {
    return "projects"
}
```

#### Repository with GORM
```go
// infrastructure/repositories/gorm_project_repository.go
package repositories

import (
    "context"
    "gorm.io/gorm"
    "yourapp/domain/entities"
)

type GormProjectRepository struct {
    db *gorm.DB
}

func NewGormProjectRepository(db *gorm.DB) *GormProjectRepository {
    return &GormProjectRepository{db: db}
}

func (r *GormProjectRepository) FindByID(ctx context.Context, tenantID, projectID string) (*entities.Project, error) {
    var project entities.Project
    err := r.db.WithContext(ctx).
        Where("id = ? AND tenant_id = ?", projectID, tenantID).
        First(&project).Error
    
    if err == gorm.ErrRecordNotFound {
        return nil, nil
    }
    return &project, err
}

func (r *GormProjectRepository) FindAll(ctx context.Context, tenantID string) ([]*entities.Project, error) {
    var projects []*entities.Project
    err := r.db.WithContext(ctx).
        Where("tenant_id = ?", tenantID).
        Order("created_at DESC").
        Find(&projects).Error
    
    return projects, err
}

func (r *GormProjectRepository) Save(ctx context.Context, project *entities.Project) error {
    return r.db.WithContext(ctx).Save(project).Error
}

func (r *GormProjectRepository) Delete(ctx context.Context, tenantID, projectID string) error {
    return r.db.WithContext(ctx).
        Where("id = ? AND tenant_id = ?", projectID, tenantID).
        Delete(&entities.Project{}).Error
}

// Complex queries with GORM
func (r *GormProjectRepository) FindBySpecification(
    ctx context.Context,
    tenantID string,
    filters map[string]interface{},
) ([]*entities.Project, error) {
    query := r.db.WithContext(ctx).Where("tenant_id = ?", tenantID)
    
    if status, ok := filters["status"].(string); ok {
        query = query.Where("status = ?", status)
    }
    
    if minBudget, ok := filters["minBudget"].(float64); ok {
        query = query.Where("budget >= ?", minBudget)
    }
    
    if maxBudget, ok := filters["maxBudget"].(float64); ok {
        query = query.Where("budget <= ?", maxBudget)
    }
    
    if createdAfter, ok := filters["createdAfter"].(time.Time); ok {
        query = query.Where("created_at >= ?", createdAfter)
    }
    
    if createdBy, ok := filters["createdBy"].(string); ok {
        query = query.Where("created_by = ?", createdBy)
    }
    
    if search, ok := filters["search"].(string); ok {
        query = query.Where(
            "name ILIKE ? OR description ILIKE ?",
            "%"+search+"%",
            "%"+search+"%",
        )
    }
    
    var projects []*entities.Project
    err := query.Order("created_at DESC").Find(&projects).Error
    
    return projects, err
}

// Pagination
func (r *GormProjectRepository) FindWithPagination(
    ctx context.Context,
    tenantID string,
    page, limit int,
    filters map[string]interface{},
) (*PaginatedResult, error) {
    query := r.db.WithContext(ctx).Where("tenant_id = ?", tenantID)
    
    // Apply filters (same as above)...
    
    var total int64
    query.Model(&entities.Project{}).Count(&total)
    
    var projects []*entities.Project
    err := query.
        Offset((page - 1) * limit).
        Limit(limit).
        Order("created_at DESC").
        Find(&projects).Error
    
    if err != nil {
        return nil, err
    }
    
    return &PaginatedResult{
        Projects:   projects,
        Total:      int(total),
        Page:       page,
        Limit:      limit,
        TotalPages: (int(total) + limit - 1) / limit,
    }, nil
}
```

---

### Comparison Table: Which ORM to Choose?

| Feature | TypeORM | Prisma | Sequelize | Spring Data JPA | GORM |
|---------|---------|--------|-----------|-----------------|------|
| **Language** | TypeScript | TypeScript | JavaScript/TS | Java | Go |
| **Type Safety** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Learning Curve** | Moderate | Easy | Moderate | Steep | Easy |
| **Performance** | Good | Excellent | Good | Excellent | Excellent |
| **Query Builder** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Migrations** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Flyway/Liquibase | ✅ AutoMigrate |
| **Raw SQL Support** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Specification Pattern** | Manual | Manual | Manual | ✅ Built-in | Manual |
| **Repository Pattern** | ✅ Built-in | Manual | Manual | ✅ Built-in | Manual |
| **Transactions** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Community** | Large | Growing | Very Large | Huge | Large |
| **Best For** | General use | Modern TS projects | Legacy projects | Enterprise Java | Go projects |


### Recommendations

#### For Node.js/TypeScript:
1. **Starting new project?** → Use **Prisma** (best DX, modern, type-safe)
2. **Need advanced features?** → Use **TypeORM** (more mature, more features)
3. **Legacy codebase?** → Use **Sequelize** (most established)

#### For Java:
1. **Any Spring Boot project** → Use **Spring Data JPA** (no-brainer, zero boilerplate)

#### For Go:
1. **Most projects** → Use **GORM** (most popular, well-tested)
2. **Need more control?** → Use **sqlx** (lightweight, closer to SQL)

### Key Takeaway

**You DON'T need to build everything from scratch!**

- Use these mature ORMs for 90% of your needs
- Implement custom Repository pattern when you need:
  - Swappable data sources
  - Additional abstraction layer
  - Domain-driven design separation
  - Testing with mocks

**Next:** Should I continue with Pattern 7.3 (Unit of Work) or would you like to see more examples with these ORMs?—

## Pattern 7.4: Unit of Work Pattern

### The Problem

When working with multiple repositories, you need to ensure **atomicity** - all operations succeed together or all fail together.

```typescript
// ❌ BAD: No transaction coordination
async function createProjectWithTasks(projectData, tasksData) {
  // Step 1: Create project
  const project = await projectRepository.save(projectData);
  
  // Step 2: Create tasks
  for (const taskData of tasksData) {
    await taskRepository.save({
      ...taskData,
      projectId: project.id
    });
  }
  
  // Step 3: Log activity
  await activityRepository.create({
    type: 'project_created',
    projectId: project.id
  });
  
  // PROBLEM: What if step 2 or 3 fails?
  // Project is created but tasks/activity are not!
  // Database is in inconsistent state!
}
```

**Problems:**
1. **Partial Updates** - Some changes saved, others failed
2. **Inconsistent State** - Database integrity compromised
3. **Hard to Rollback** - Need manual cleanup
4. **No Transaction Management** - Each repository commits independently

### The Solution: Unit of Work Pattern

The **Unit of Work** pattern maintains a list of objects affected by a business transaction and coordinates the writing of changes as a single transaction.

**Key Concepts:**
- **Track Changes** - Remember all modifications
- **Single Commit** - Write all changes at once
- **Automatic Rollback** - Rollback all if any fails
- **Transaction Boundary** - Clear start and end

### Implementation: TypeScript/Node.js

#### Step 1: Define Unit of Work Interface

```typescript
// domain/uow/IUnitOfWork.ts
export interface IUnitOfWork {
  // Access repositories
  projects: IProjectRepository;
  tasks: ITaskRepository;
  activities: IActivityRepository;
  users: IUserRepository;
  
  // Transaction control
  beginTransaction(): Promise<void>;
  commit(): Promise<void>;
  rollback(): Promise<void>;
  
  // Helper for automatic transaction management
  execute<T>(work: (uow: IUnitOfWork) => Promise<T>): Promise<T>;
}
```

#### Step 2: Implement Unit of Work

```typescript
// infrastructure/uow/PostgresUnitOfWork.ts
import { Pool, PoolClient } from 'pg';
import { IUnitOfWork } from '../../domain/uow/IUnitOfWork';
import { PostgresProjectRepository } from '../repositories/PostgresProjectRepository';
import { PostgresTaskRepository } from '../repositories/PostgresTaskRepository';
import { PostgresActivityRepository } from '../repositories/PostgresActivityRepository';
import { PostgresUserRepository } from '../repositories/PostgresUserRepository';

export class PostgresUnitOfWork implements IUnitOfWork {
  private client: PoolClient | null = null;
  private isInTransaction = false;
  
  // Repository instances
  public projects: PostgresProjectRepository;
  public tasks: PostgresTaskRepository;
  public activities: PostgresActivityRepository;
  public users: PostgresUserRepository;
  
  constructor(private pool: Pool) {
    // Initialize repositories with the pool
    // They'll use the transaction client when available
    this.projects = new PostgresProjectRepository(this);
    this.tasks = new PostgresTaskRepository(this);
    this.activities = new PostgresActivityRepository(this);
    this.users = new PostgresUserRepository(this);
  }
  
  // Get database client (transaction client if in transaction, otherwise pool)
  getClient(): PoolClient | Pool {
    return this.client || this.pool;
  }
  
  async beginTransaction(): Promise<void> {
    if (this.isInTransaction) {
      throw new Error('Transaction already started');
    }
    
    this.client = await this.pool.connect();
    await this.client.query('BEGIN');
    this.isInTransaction = true;
  }
  
  async commit(): Promise<void> {
    if (!this.isInTransaction || !this.client) {
      throw new Error('No active transaction');
    }
    
    try {
      await this.client.query('COMMIT');
    } finally {
      this.client.release();
      this.client = null;
      this.isInTransaction = false;
    }
  }
  
  async rollback(): Promise<void> {
    if (!this.isInTransaction || !this.client) {
      throw new Error('No active transaction');
    }
    
    try {
      await this.client.query('ROLLBACK');
    } finally {
      this.client.release();
      this.client = null;
      this.isInTransaction = false;
    }
  }
  
  // Helper method for automatic transaction management
  async execute<T>(work: (uow: IUnitOfWork) => Promise<T>): Promise<T> {
    await this.beginTransaction();
    
    try {
      const result = await work(this);
      await this.commit();
      return result;
    } catch (error) {
      await this.rollback();
      throw error;
    }
  }
}
```

#### Step 3: Update Repositories to Use Unit of Work

```typescript
// infrastructure/repositories/PostgresProjectRepository.ts
export class PostgresProjectRepository implements IProjectRepository {
  constructor(private uow: PostgresUnitOfWork) {}
  
  private get db() {
    return this.uow.getClient();
  }
  
  async save(project: Project): Promise<Project> {
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
    
    return this.mapToDomain(result.rows[0]);
  }
  
  // ... other methods use this.db
}

// Similar for TaskRepository, ActivityRepository, etc.
```

#### Step 4: Using Unit of Work in Service Layer

```typescript
// application/services/ProjectService.ts
export class ProjectService {
  constructor(private uowFactory: () => IUnitOfWork) {}
  
  // Example 1: Simple transaction
  async createProjectWithTasks(
    tenantId: string,
    userId: string,
    projectData: CreateProjectDTO,
    tasksData: CreateTaskDTO[]
  ): Promise<{ project: Project; tasks: Task[] }> {
    const uow = this.uowFactory();
    
    return uow.execute(async (uow) => {
      // Step 1: Create project
      const project = new Project(
        uuid(),
        tenantId,
        projectData.name,
        projectData.description,
        'active',
        projectData.budget || null,
        projectData.startDate || null,
        projectData.endDate || null,
        userId,
        new Date(),
        new Date()
      );
      
      await uow.projects.save(project);
      
      // Step 2: Create tasks
      const tasks: Task[] = [];
      for (const taskData of tasksData) {
        const task = new Task(
          uuid(),
          tenantId,
          project.id,
          taskData.title,
          taskData.description,
          'todo',
          taskData.assignedTo || null,
          userId,
          new Date(),
          new Date()
        );
        
        await uow.tasks.save(task);
        tasks.push(task);
      }
      
      // Step 3: Log activity
      await uow.activities.create({
        tenantId,
        type: 'project_created',
        entityType: 'project',
        entityId: project.id,
        userId,
        metadata: {
          projectName: project.name,
          taskCount: tasks.length
        }
      });
      
      // All operations committed together!
      return { project, tasks };
    });
  }
  
  // Example 2: Complex business transaction
  async transferProjectOwnership(
    tenantId: string,
    projectId: string,
    fromUserId: string,
    toUserId: string
  ): Promise<void> {
    const uow = this.uowFactory();
    
    return uow.execute(async (uow) => {
      // Load project
      const project = await uow.projects.findById(tenantId, projectId);
      if (!project) {
        throw new Error('Project not found');
      }
      
      // Verify current owner
      if (project.createdBy !== fromUserId) {
        throw new Error('Only project owner can transfer ownership');
      }
      
      // Verify new owner exists
      const newOwner = await uow.users.findById(tenantId, toUserId);
      if (!newOwner) {
        throw new Error('New owner not found');
      }
      
      // Update project
      project.createdBy = toUserId;
      project.updatedAt = new Date();
      await uow.projects.save(project);
      
      // Update all tasks owned by old owner
      const tasks = await uow.tasks.findByProject(tenantId, projectId);
      for (const task of tasks) {
        if (task.createdBy === fromUserId) {
          task.createdBy = toUserId;
          task.updatedAt = new Date();
          await uow.tasks.save(task);
        }
      }
      
      // Log activity
      await uow.activities.create({
        tenantId,
        type: 'project_ownership_transferred',
        entityType: 'project',
        entityId: projectId,
        userId: fromUserId,
        metadata: {
          fromUserId,
          toUserId,
          projectName: project.name
        }
      });
      
      // All or nothing!
    });
  }
  
  // Example 3: Bulk operations
  async archiveMultipleProjects(
    tenantId: string,
    projectIds: string[],
    userId: string
  ): Promise<number> {
    const uow = this.uowFactory();
    
    return uow.execute(async (uow) => {
      let archivedCount = 0;
      
      for (const projectId of projectIds) {
        const project = await uow.projects.findById(tenantId, projectId);
        
        if (project && project.canBeArchived()) {
          project.archive();
          await uow.projects.save(project);
          
          // Log each archival
          await uow.activities.create({
            tenantId,
            type: 'project_archived',
            entityType: 'project',
            entityId: projectId,
            userId,
            metadata: { projectName: project.name }
          });
          
          archivedCount++;
        }
      }
      
      // All archived together or none
      return archivedCount;
    });
  }
  
  // Example 4: Nested transactions (savepoints)
  async complexWorkflow(tenantId: string, data: any): Promise<void> {
    const uow = this.uowFactory();
    
    await uow.beginTransaction();
    
    try {
      // Step 1: Create project
      const project = await uow.projects.save(data.project);
      
      // Step 2: Try to create tasks (might partially fail)
      try {
        for (const taskData of data.tasks) {
          await uow.tasks.save(taskData);
        }
      } catch (error) {
        // Tasks failed, but we still want to keep the project
        // In PostgreSQL, we'd use SAVEPOINT here
        console.log('Some tasks failed, but project is saved');
      }
      
      // Step 3: Always log activity
      await uow.activities.create({
        tenantId,
        type: 'workflow_completed',
        entityType: 'project',
        entityId: project.id,
        userId: data.userId
      });
      
      await uow.commit();
    } catch (error) {
      await uow.rollback();
      throw error;
    }
  }
}
```

#### Step 5: Dependency Injection Setup

```typescript
// infrastructure/di/container.ts
import { Pool } from 'pg';
import { PostgresUnitOfWork } from '../uow/PostgresUnitOfWork';
import { ProjectService } from '../../application/services/ProjectService';

export class DIContainer {
  private pool: Pool;
  
  constructor(databaseConfig: any) {
    this.pool = new Pool(databaseConfig);
  }
  
  createUnitOfWork(): PostgresUnitOfWork {
    return new PostgresUnitOfWork(this.pool);
  }
  
  createProjectService(): ProjectService {
    return new ProjectService(() => this.createUnitOfWork());
  }
}

// Usage in application
const container = new DIContainer({
  host: 'localhost',
  port: 5432,
  database: 'saas_db',
  user: 'postgres',
  password: 'password'
});

const projectService = container.createProjectService();
```

---

### Implementation: Java (Spring Boot)

Spring Boot has **built-in transaction management** with `@Transactional` annotation!

```java
// application/services/ProjectService.java
@Service
public class ProjectService {
    
    @Autowired
    private ProjectRepository projectRepository;
    
    @Autowired
    private TaskRepository taskRepository;
    
    @Autowired
    private ActivityRepository activityRepository;
    
    // Simple transaction with @Transactional
    @Transactional
    public ProjectWithTasks createProjectWithTasks(
        UUID tenantId,
        UUID userId,
        CreateProjectDTO projectData,
        List<CreateTaskDTO> tasksData
    ) {
        // Step 1: Create project
        Project project = new Project();
        project.setId(UUID.randomUUID());
        project.setTenantId(tenantId);
        project.setName(projectData.getName());
        project.setDescription(projectData.getDescription());
        project.setStatus("active");
        project.setCreatedBy(userId);
        project.setCreatedAt(LocalDateTime.now());
        project.setUpdatedAt(LocalDateTime.now());
        
        project = projectRepository.save(project);
        
        // Step 2: Create tasks
        List<Task> tasks = new ArrayList<>();
        for (CreateTaskDTO taskData : tasksData) {
            Task task = new Task();
            task.setId(UUID.randomUUID());
            task.setTenantId(tenantId);
            task.setProjectId(project.getId());
            task.setTitle(taskData.getTitle());
            task.setDescription(taskData.getDescription());
            task.setStatus("todo");
            task.setCreatedBy(userId);
            task.setCreatedAt(LocalDateTime.now());
            task.setUpdatedAt(LocalDateTime.now());
            
            tasks.add(taskRepository.save(task));
        }
        
        // Step 3: Log activity
        Activity activity = new Activity();
        activity.setTenantId(tenantId);
        activity.setType("project_created");
        activity.setEntityType("project");
        activity.setEntityId(project.getId());
        activity.setUserId(userId);
        activityRepository.save(activity);
        
        // All operations committed together automatically!
        // If any step throws exception, everything rolls back!
        
        return new ProjectWithTasks(project, tasks);
    }
    
    // Transaction with manual rollback
    @Transactional
    public void transferProjectOwnership(
        UUID tenantId,
        UUID projectId,
        UUID fromUserId,
        UUID toUserId
    ) {
        // Load project
        Project project = projectRepository
            .findByIdAndTenantId(projectId, tenantId)
            .orElseThrow(() -> new NotFoundException("Project not found"));
        
        // Verify current owner
        if (!project.getCreatedBy().equals(fromUserId)) {
            throw new UnauthorizedException("Only owner can transfer ownership");
        }
        
        // Verify new owner exists
        User newOwner = userRepository
            .findByIdAndTenantId(toUserId, tenantId)
            .orElseThrow(() -> new NotFoundException("New owner not found"));
        
        // Update project
        project.setCreatedBy(toUserId);
        project.setUpdatedAt(LocalDateTime.now());
        projectRepository.save(project);
        
        // Update tasks
        List<Task> tasks = taskRepository.findByProjectId(tenantId, projectId);
        for (Task task : tasks) {
            if (task.getCreatedBy().equals(fromUserId)) {
                task.setCreatedBy(toUserId);
                task.setUpdatedAt(LocalDateTime.now());
                taskRepository.save(task);
            }
        }
        
        // Log activity
        Activity activity = new Activity();
        activity.setTenantId(tenantId);
        activity.setType("project_ownership_transferred");
        activity.setEntityType("project");
        activity.setEntityId(projectId);
        activity.setUserId(fromUserId);
        activityRepository.save(activity);
        
        // Transaction automatically commits if no exception thrown
        // Rolls back if any exception occurs
    }
    
    // Transaction with programmatic control
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public void complexWorkflowWithManualControl(UUID tenantId, WorkflowData data) {
        transactionTemplate.execute(status -> {
            try {
                // Step 1
                Project project = projectRepository.save(data.getProject());
                
                // Step 2
                for (TaskData taskData : data.getTasks()) {
                    taskRepository.save(taskData.toEntity());
                }
                
                // Step 3
                activityRepository.save(data.getActivity());
                
                return null;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
    
    // Transaction with isolation level
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void criticalOperation(UUID tenantId, UUID projectId) {
        // This transaction has the highest isolation level
        // Prevents concurrent modifications
    }
    
    // Transaction with timeout
    @Transactional(timeout = 30)
    public void longRunningOperation(UUID tenantId) {
        // Transaction will rollback if it takes more than 30 seconds
    }
    
    // Read-only transaction (optimization)
    @Transactional(readOnly = true)
    public List<Project> getProjects(UUID tenantId) {
        // Database can optimize for read-only access
        return projectRepository.findByTenantId(tenantId);
    }
}
```

---

### Implementation: Go

```go
// infrastructure/uow/unit_of_work.go
package uow

import (
    "context"
    "database/sql"
    "yourapp/infrastructure/repositories"
)

type UnitOfWork struct {
    db  *sql.DB
    tx  *sql.Tx
    
    // Repositories
    Projects   *repositories.ProjectRepository
    Tasks      *repositories.TaskRepository
    Activities *repositories.ActivityRepository
}

func NewUnitOfWork(db *sql.DB) *UnitOfWork {
    uow := &UnitOfWork{
        db: db,
    }
    return uow
}

func (uow *UnitOfWork) Begin(ctx context.Context) error {
    tx, err := uow.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    
    uow.tx = tx
    
    // Initialize repositories with transaction
    uow.Projects = repositories.NewProjectRepository(tx)
    uow.Tasks = repositories.NewTaskRepository(tx)
    uow.Activities = repositories.NewActivityRepository(tx)
    
    return nil
}

func (uow *UnitOfWork) Commit() error {
    if uow.tx == nil {
        return fmt.Errorf("no active transaction")
    }
    
    err := uow.tx.Commit()
    uow.tx = nil
    return err
}

func (uow *UnitOfWork) Rollback() error {
    if uow.tx == nil {
        return fmt.Errorf("no active transaction")
    }
    
    err := uow.tx.Rollback()
    uow.tx = nil
    return err
}

// Helper for automatic transaction management
func (uow *UnitOfWork) Execute(ctx context.Context, fn func(*UnitOfWork) error) error {
    if err := uow.Begin(ctx); err != nil {
        return err
    }
    
    defer func() {
        if p := recover(); p != nil {
            uow.Rollback()
            panic(p)
        }
    }()
    
    if err := fn(uow); err != nil {
        uow.Rollback()
        return err
    }
    
    return uow.Commit()
}

// application/services/project_service.go
type ProjectService struct {
    dbFactory func() *UnitOfWork
}

func NewProjectService(dbFactory func() *UnitOfWork) *ProjectService {
    return &ProjectService{
        dbFactory: dbFactory,
    }
}

// Example 1: Create project with tasks
func (s *ProjectService) CreateProjectWithTasks(
    ctx context.Context,
    tenantID string,
    userID string,
    projectData *CreateProjectDTO,
    tasksData []*CreateTaskDTO,
) (*Project, []*Task, error) {
    uow := s.dbFactory()
    
    var project *Project
    var tasks []*Task
    
    err := uow.Execute(ctx, func(uow *UnitOfWork) error {
        // Step 1: Create project
        project = &Project{
            ID:          uuid.New().String(),
            TenantID:    tenantID,
            Name:        projectData.Name,
            Description: projectData.Description,
            Status:      "active",
            CreatedBy:   userID,
            CreatedAt:   time.Now(),
            UpdatedAt:   time.Now(),
        }
        
        if err := uow.Projects.Save(ctx, project); err != nil {
            return err
        }
        
        // Step 2: Create tasks
        for _, taskData := range tasksData {
            task := &Task{
                ID:          uuid.New().String(),
                TenantID:    tenantID,
                ProjectID:   project.ID,
                Title:       taskData.Title,
                Description: taskData.Description,
                Status:      "todo",
                CreatedBy:   userID,
                CreatedAt:   time.Now(),
                UpdatedAt:   time.Now(),
            }
            
            if err := uow.Tasks.Save(ctx, task); err != nil {
                return err
            }
            
            tasks = append(tasks, task)
        }
        
        // Step 3: Log activity
        activity := &Activity{
            TenantID:   tenantID,
            Type:       "project_created",
            EntityType: "project",
            EntityID:   project.ID,
            UserID:     userID,
            CreatedAt:  time.Now(),
        }
        
        if err := uow.Activities.Create(ctx, activity); err != nil {
            return err
        }
        
        // All operations succeed or all fail
        return nil
    })
    
    return project, tasks, err
}

// Example 2: Transfer ownership
func (s *ProjectService) TransferProjectOwnership(
    ctx context.Context,
    tenantID, projectID, fromUserID, toUserID string,
) error {
    uow := s.dbFactory()
    
    return uow.Execute(ctx, func(uow *UnitOfWork) error {
        // Load project
        project, err := uow.Projects.FindByID(ctx, tenantID, projectID)
        if err != nil {
            return err
        }
        if project == nil {
            return fmt.Errorf("project not found")
        }
        
        // Verify current owner
        if project.CreatedBy != fromUserID {
            return fmt.Errorf("only owner can transfer")
        }
        
        // Update project
        project.CreatedBy = toUserID
        project.UpdatedAt = time.Now()
        if err := uow.Projects.Save(ctx, project); err != nil {
            return err
        }
        
        // Update tasks
        tasks, err := uow.Tasks.FindByProject(ctx, tenantID, projectID)
        if err != nil {
            return err
        }
        
        for _, task := range tasks {
            if task.CreatedBy == fromUserID {
                task.CreatedBy = toUserID
                task.UpdatedAt = time.Now()
                if err := uow.Tasks.Save(ctx, task); err != nil {
                    return err
                }
            }
        }
        
        // Log activity
        activity := &Activity{
            TenantID:   tenantID,
            Type:       "ownership_transferred",
            EntityType: "project",
            EntityID:   projectID,
            UserID:     fromUserID,
            CreatedAt:  time.Now(),
        }
        
        return uow.Activities.Create(ctx, activity)
    })
}
```

---

### Benefits of Unit of Work Pattern

✅ **Atomicity** - All operations succeed or fail together  
✅ **Consistency** - Database always in valid state  
✅ **Cleaner Code** - Transaction logic separate from business logic  
✅ **Testable** - Easy to test with mock UoW  
✅ **Prevents Partial Updates** - No inconsistent data  
✅ **Better Performance** - Single commit instead of many  

### When to Use Unit of Work

**✅ Use When:**
- Operations span multiple repositories
- Need ACID guarantees
- Complex business transactions
- Data consistency critical

**❌ Don't Use When:**
- Simple single-repository operations
- Read-only operations
- Operations don't need atomicity
- Using ORM with built-in transaction management (like Spring Data JPA)

### Real-World SaaS Use Cases

1. **User Registration** - Create user + create tenant + assign role + send welcome email
2. **Order Processing** - Create order + update inventory + charge payment + send confirmation
3. **Project Creation** - Create project + create tasks + invite team + log activity
4. **Data Migration** - Delete old data + insert new data + update references
5. **Subscription Change** - Cancel old subscription + create new + prorate charges + notify user

---

### Summary: Unit of Work Pattern

**Key Points:**
- Coordinates changes across multiple repositories
- Ensures all-or-nothing transactional behavior
- Cleaner separation of concerns
- Built-in to many ORMs (Spring, Entity Framework)
- Manual implementation needed for Node.js/Go

**Next:** Should I continue with Pattern 7.5 (Cached Repository) or move to next structural pattern?**2. Reporting:**
```typescript
// Get all high-value projects from last quarter
const lastQuarter = getLastQuarterDate();
const spec = new HighBudgetProjectSpecification(100000)
  .and(new CreatedAfterSpecification(lastQuarter));
```

---

## Pattern 7.5: Cached Repository

### The Problem

Database queries are slow and expensive, especially when:

* Same data is requested repeatedly
* Complex queries with joins and aggregations
* High traffic with many concurrent users
* Data doesn't change frequently

```typescript
// ❌ BAD: Every request hits the database
async function getUserProfile(userId: string) {
  // Database query every single time (50-100ms)
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  return user;
}

// With 1000 requests per second:
// - 1000 database queries per second
// - Database becomes bottleneck
// - Slow response times
// - High database costs
```

### The Solution: Cached Repository

Add a **caching layer** between your application and database:

* First request → Database → Cache
* Subsequent requests → Cache (1-5ms instead of 50-100ms)
* Cache invalidation when data changes

**Key Benefits:**

* 10-100x faster reads
* Reduced database load
* Lower costs
* Better scalability

***

### Implementation Strategy

#### Cache Decorator Pattern

We'll use the **Decorator Pattern** to wrap existing repositories with caching:

```
Application
    ↓
CachedProjectRepository (decorator)
    ↓ (cache miss)
ProjectRepository (actual implementation)
    ↓
Database
```

Benefits:

* Existing repositories unchanged
* Caching added transparently
* Easy to disable caching
* Testable

***

### Complete Implementation: TypeScript/Node.js

#### Step 1: Setup Redis

```bash
npm install redis
npm install @types/redis --save-dev
```

```typescript
// infrastructure/cache/RedisClient.ts
import { createClient } from 'redis';

export class RedisClient {
  private client: ReturnType<typeof createClient>;
  private isConnected = false;
  
  constructor(config: { host: string; port: number; password?: string }) {
    this.client = createClient({
      socket: {
        host: config.host,
        port: config.port
      },
      password: config.password
    });
    
    this.client.on('error', (err) => console.error('Redis error:', err));
    this.client.on('connect', () => console.log('Redis connected'));
  }
  
  async connect(): Promise<void> {
    if (!this.isConnected) {
      await this.client.connect();
      this.isConnected = true;
    }
  }
  
  async disconnect(): Promise<void> {
    if (this.isConnected) {
      await this.client.quit();
      this.isConnected = false;
    }
  }
  
  async get(key: string): Promise<string | null> {
    return this.client.get(key);
  }
  
  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    if (ttlSeconds) {
      await this.client.setEx(key, ttlSeconds, value);
    } else {
      await this.client.set(key, value);
    }
  }
  
  async del(key: string): Promise<void> {
    await this.client.del(key);
  }
  
  async delPattern(pattern: string): Promise<void> {
    const keys = await this.client.keys(pattern);
    if (keys.length > 0) {
      await this.client.del(keys);
    }
  }
  
  async exists(key: string): Promise<boolean> {
    const result = await this.client.exists(key);
    return result === 1;
  }
}
```

#### Step 2: Create Cache Key Strategy

```typescript
// infrastructure/cache/CacheKeyGenerator.ts
export class CacheKeyGenerator {
  private prefix: string;
  
  constructor(prefix: string = 'saas') {
    this.prefix = prefix;
  }
  
  // Generate keys with consistent format
  project(tenantId: string, projectId: string): string {
    return `${this.prefix}:tenant:${tenantId}:project:${projectId}`;
  }
  
  projectList(tenantId: string): string {
    return `${this.prefix}:tenant:${tenantId}:projects:list`;
  }
  
  projectsByStatus(tenantId: string, status: string): string {
    return `${this.prefix}:tenant:${tenantId}:projects:status:${status}`;
  }
  
  projectSearch(tenantId: string, searchTerm: string): string {
    // Normalize search term for cache key
    const normalized = searchTerm.toLowerCase().trim();
    return `${this.prefix}:tenant:${tenantId}:projects:search:${normalized}`;
  }
  
  // Pattern for invalidating all project caches for a tenant
  projectPattern(tenantId: string): string {
    return `${this.prefix}:tenant:${tenantId}:project*`;
  }
  
  // User-related keys
  user(tenantId: string, userId: string): string {
    return `${this.prefix}:tenant:${tenantId}:user:${userId}`;
  }
  
  userPattern(tenantId: string): string {
    return `${this.prefix}:tenant:${tenantId}:user*`;
  }
}
```

#### Step 3: Implement Cached Repository Decorator

```typescript
// infrastructure/repositories/CachedProjectRepository.ts
import { IProjectRepository } from '../../domain/repositories/IProjectRepository';
import { Project } from '../../domain/entities/Project';
import { RedisClient } from '../cache/RedisClient';
import { CacheKeyGenerator } from '../cache/CacheKeyGenerator';

export class CachedProjectRepository implements IProjectRepository {
  private ttl = {
    single: 300,      // 5 minutes for single project
    list: 60,         // 1 minute for lists
    search: 180       // 3 minutes for search results
  };
  
  constructor(
    private baseRepository: IProjectRepository,
    private cache: RedisClient,
    private keyGen: CacheKeyGenerator
  ) {}
  
  async findById(tenantId: string, projectId: string): Promise<Project | null> {
    const cacheKey = this.keyGen.project(tenantId, projectId);
    
    // Try cache first
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      console.log('Cache HIT:', cacheKey);
      return JSON.parse(cached) as Project;
    }
    
    console.log('Cache MISS:', cacheKey);
    
    // Cache miss - fetch from database
    const project = await this.baseRepository.findById(tenantId, projectId);
    
    // Store in cache
    if (project) {
      await this.cache.set(
        cacheKey,
        JSON.stringify(project),
        this.ttl.single
      );
    }
    
    return project;
  }
  
  async findAll(tenantId: string): Promise<Project[]> {
    const cacheKey = this.keyGen.projectList(tenantId);
    
    // Try cache
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      console.log('Cache HIT:', cacheKey);
      return JSON.parse(cached) as Project[];
    }
    
    console.log('Cache MISS:', cacheKey);
    
    // Fetch from database
    const projects = await this.baseRepository.findAll(tenantId);
    
    // Store in cache
    await this.cache.set(
      cacheKey,
      JSON.stringify(projects),
      this.ttl.list
    );
    
    return projects;
  }
  
  async save(project: Project): Promise<Project> {
    // Save to database first
    const saved = await this.baseRepository.save(project);
    
    // Invalidate all caches for this tenant's projects
    await this.invalidateProjectCaches(project.tenantId);
    
    // Update single project cache
    const cacheKey = this.keyGen.project(project.tenantId, project.id);
    await this.cache.set(
      cacheKey,
      JSON.stringify(saved),
      this.ttl.single
    );
    
    return saved;
  }
  
  async delete(tenantId: string, projectId: string): Promise<boolean> {
    // Delete from database
    const deleted = await this.baseRepository.delete(tenantId, projectId);
    
    if (deleted) {
      // Invalidate caches
      await this.invalidateProjectCaches(tenantId);
      
      // Delete specific cache entry
      const cacheKey = this.keyGen.project(tenantId, projectId);
      await this.cache.del(cacheKey);
    }
    
    return deleted;
  }
  
  async findByStatus(tenantId: string, status: string): Promise<Project[]> {
    const cacheKey = this.keyGen.projectsByStatus(tenantId, status);
    
    // Try cache
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      console.log('Cache HIT:', cacheKey);
      return JSON.parse(cached) as Project[];
    }
    
    console.log('Cache MISS:', cacheKey);
    
    // Fetch from database
    const projects = await this.baseRepository.findByStatus(tenantId, status);
    
    // Store in cache
    await this.cache.set(
      cacheKey,
      JSON.stringify(projects),
      this.ttl.list
    );
    
    return projects;
  }
  
  async search(tenantId: string, searchTerm: string): Promise<Project[]> {
    const cacheKey = this.keyGen.projectSearch(tenantId, searchTerm);
    
    // Try cache
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      console.log('Cache HIT:', cacheKey);
      return JSON.parse(cached) as Project[];
    }
    
    console.log('Cache MISS:', cacheKey);
    
    // Fetch from database
    const projects = await this.baseRepository.search(tenantId, searchTerm);
    
    // Store in cache
    await this.cache.set(
      cacheKey,
      JSON.stringify(projects),
      this.ttl.search
    );
    
    return projects;
  }
  
  async findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: any
  ): Promise<any> {
    // For pagination, we might not cache or use shorter TTL
    // because results change frequently
    
    // Option 1: Don't cache pagination (simpler)
    return this.baseRepository.findWithPagination(tenantId, page, limit, filters);
    
    // Option 2: Cache with shorter TTL
    // const cacheKey = `${this.keyGen.projectList(tenantId)}:page:${page}:limit:${limit}`;
    // const cached = await this.cache.get(cacheKey);
    // if (cached) return JSON.parse(cached);
    // 
    // const result = await this.baseRepository.findWithPagination(tenantId, page, limit, filters);
    // await this.cache.set(cacheKey, JSON.stringify(result), 30); // 30 seconds
    // return result;
  }
  
  async count(tenantId: string): Promise<number> {
    // Counts can be cached
    const cacheKey = `${this.keyGen.projectList(tenantId)}:count`;
    
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      return parseInt(cached);
    }
    
    const count = await this.baseRepository.count(tenantId);
    await this.cache.set(cacheKey, count.toString(), this.ttl.list);
    
    return count;
  }
  
  async countByStatus(tenantId: string, status: string): Promise<number> {
    const cacheKey = `${this.keyGen.projectsByStatus(tenantId, status)}:count`;
    
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      return parseInt(cached);
    }
    
    const count = await this.baseRepository.countByStatus(tenantId, status);
    await this.cache.set(cacheKey, count.toString(), this.ttl.list);
    
    return count;
  }
  
  async exists(tenantId: string, projectId: string): Promise<boolean> {
    // Check cache first
    const cacheKey = this.keyGen.project(tenantId, projectId);
    const inCache = await this.cache.exists(cacheKey);
    
    if (inCache) {
      return true;
    }
    
    // Check database
    return this.baseRepository.exists(tenantId, projectId);
  }
  
  async existsByName(tenantId: string, name: string): Promise<boolean> {
    // Usually not cached - names are unique checks
    return this.baseRepository.existsByName(tenantId, name);
  }
  
  // Helper: Invalidate all project caches for a tenant
  private async invalidateProjectCaches(tenantId: string): Promise<void> {
    const pattern = this.keyGen.projectPattern(tenantId);
    await this.cache.delPattern(pattern);
    console.log('Invalidated cache pattern:', pattern);
  }
  
  // Manual cache invalidation (useful for testing or admin operations)
  async invalidateCache(tenantId: string, projectId?: string): Promise<void> {
    if (projectId) {
      // Invalidate specific project
      const cacheKey = this.keyGen.project(tenantId, projectId);
      await this.cache.del(cacheKey);
    } else {
      // Invalidate all projects for tenant
      await this.invalidateProjectCaches(tenantId);
    }
  }
}
```

#### Step 4: Setup and Usage

```typescript
// infrastructure/di/container.ts
import { Pool } from 'pg';
import { RedisClient } from '../cache/RedisClient';
import { CacheKeyGenerator } from '../cache/CacheKeyGenerator';
import { PostgresProjectRepository } from '../repositories/PostgresProjectRepository';
import { CachedProjectRepository } from '../repositories/CachedProjectRepository';
import { ProjectService } from '../../application/services/ProjectService';

export class DIContainer {
  private pool: Pool;
  private redis: RedisClient;
  private keyGen: CacheKeyGenerator;
  
  constructor(dbConfig: any, redisConfig: any) {
    this.pool = new Pool(dbConfig);
    this.redis = new RedisClient(redisConfig);
    this.keyGen = new CacheKeyGenerator('saas');
  }
  
  async initialize(): Promise<void> {
    await this.redis.connect();
  }
  
  createProjectRepository(): IProjectRepository {
    const baseRepo = new PostgresProjectRepository(this.pool);
    
    // Wrap with caching
    return new CachedProjectRepository(baseRepo, this.redis, this.keyGen);
  }
  
  createProjectService(): ProjectService {
    return new ProjectService(
      this.createProjectRepository(),
      this.createEventBus()
    );
  }
}

// Usage in application
const container = new DIContainer(
  {
    host: 'localhost',
    port: 5432,
    database: 'saas_db',
    user: 'postgres',
    password: 'password'
  },
  {
    host: 'localhost',
    port: 6379,
    password: undefined
  }
);

await container.initialize();

const projectService = container.createProjectService();

// Now all repository operations are automatically cached!
const project = await projectService.getProject(tenantId, projectId);
// First call: Cache MISS → Database
// Second call: Cache HIT → Redis (10-100x faster)
```

#### Step 5: Configuration and Monitoring

```typescript
// infrastructure/cache/CacheConfig.ts
export interface CacheConfig {
  enabled: boolean;
  ttl: {
    default: number;
    short: number;
    medium: number;
    long: number;
  };
  keyPrefix: string;
}

export const cacheConfig: CacheConfig = {
  enabled: process.env.NODE_ENV !== 'test', // Disable in tests
  ttl: {
    default: 300,    // 5 minutes
    short: 60,       // 1 minute
    medium: 300,     // 5 minutes
    long: 3600       // 1 hour
  },
  keyPrefix: process.env.CACHE_PREFIX || 'saas'
};

// infrastructure/cache/CacheMonitor.ts
export class CacheMonitor {
  private hits = 0;
  private misses = 0;
  
  recordHit(): void {
    this.hits++;
  }
  
  recordMiss(): void {
    this.misses++;
  }
  
  getHitRate(): number {
    const total = this.hits + this.misses;
    return total === 0 ? 0 : (this.hits / total) * 100;
  }
  
  getStats(): { hits: number; misses: number; hitRate: number } {
    return {
      hits: this.hits,
      misses: this.misses,
      hitRate: this.getHitRate()
    };
  }
  
  reset(): void {
    this.hits = 0;
    this.misses = 0;
  }
}

// Add to repository
export class CachedProjectRepository implements IProjectRepository {
  constructor(
    private baseRepository: IProjectRepository,
    private cache: RedisClient,
    private keyGen: CacheKeyGenerator,
    private monitor?: CacheMonitor
  ) {}
  
  async findById(tenantId: string, projectId: string): Promise<Project | null> {
    const cacheKey = this.keyGen.project(tenantId, projectId);
    
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      this.monitor?.recordHit();
      return JSON.parse(cached) as Project;
    }
    
    this.monitor?.recordMiss();
    
    const project = await this.baseRepository.findById(tenantId, projectId);
    
    if (project) {
      await this.cache.set(cacheKey, JSON.stringify(project), this.ttl.single);
    }
    
    return project;
  }
  
  // ... other methods
}

// Monitor cache performance
const monitor = new CacheMonitor();
const cachedRepo = new CachedProjectRepository(baseRepo, redis, keyGen, monitor);

// After some operations
const stats = monitor.getStats();
console.log(`Cache hit rate: ${stats.hitRate.toFixed(2)}%`);
console.log(`Hits: ${stats.hits}, Misses: ${stats.misses}`);
```

***

### Cache Invalidation Strategies

#### Strategy 1: Time-Based (TTL)

**Simplest approach** - Let cache expire automatically:

```typescript
// Short TTL for frequently changing data
await cache.set(key, value, 60); // 1 minute

// Long TTL for stable data
await cache.set(key, value, 3600); // 1 hour
```

**Pros:** Simple, no invalidation logic needed\
**Cons:** Stale data until expiration

#### Strategy 2: Write-Through Invalidation

**Invalidate on every write:**

```typescript
async save(project: Project): Promise<Project> {
  // Save to database
  const saved = await this.baseRepository.save(project);
  
  // Invalidate related caches
  await this.invalidateProjectCaches(project.tenantId);
  
  // Update cache with fresh data
  const cacheKey = this.keyGen.project(project.tenantId, project.id);
  await this.cache.set(cacheKey, JSON.stringify(saved), this.ttl.single);
  
  return saved;
}
```

**Pros:** Always fresh data\
**Cons:** More cache operations

#### Strategy 3: Selective Invalidation

**Only invalidate what changed:**

```typescript
async updateProject(project: Project, changes: any): Promise<Project> {
  const updated = await this.baseRepository.save(project);
  
  // Invalidate specific caches
  const specificKey = this.keyGen.project(project.tenantId, project.id);
  await this.cache.del(specificKey);
  
  // Only invalidate list caches if status changed
  if (changes.status) {
    const oldStatusKey = this.keyGen.projectsByStatus(project.tenantId, project.status);
    const newStatusKey = this.keyGen.projectsByStatus(project.tenantId, changes.status);
    await this.cache.del(oldStatusKey);
    await this.cache.del(newStatusKey);
  }
  
  // Don't invalidate search caches (they expire naturally)
  
  return updated;
}
```

**Pros:** Efficient, minimal cache operations\
**Cons:** Complex logic, easy to miss cases

#### Strategy 4: Tag-Based Invalidation

**Group related cache entries with tags:**

```typescript
class TaggedCache {
  async setWithTags(key: string, value: string, tags: string[], ttl: number): Promise<void> {
    // Store value
    await this.cache.set(key, value, ttl);
    
    // Store key in each tag set
    for (const tag of tags) {
      await this.cache.sAdd(`tag:${tag}`, key);
    }
  }
  
  async invalidateByTag(tag: string): Promise<void> {
    // Get all keys with this tag
    const keys = await this.cache.sMembers(`tag:${tag}`);
    
    // Delete all keys
    if (keys.length > 0) {
      await this.cache.del(keys);
    }
    
    // Delete tag set
    await this.cache.del(`tag:${tag}`);
  }
}

// Usage
await taggedCache.setWithTags(
  cacheKey,
  JSON.stringify(project),
  ['tenant:123', 'project:456', 'status:active'],
  300
);

// Invalidate all caches for tenant
await taggedCache.invalidateByTag('tenant:123');
```

***

### Advanced Caching Patterns

#### Pattern 1: Cache Stampede Prevention

**Problem:** When cache expires, many requests hit database simultaneously.

**Solution:** Use locking or "cache warming":

```typescript
class CacheStampedeProtection {
  private locks = new Map<string, Promise<any>>();
  
  async getOrCompute<T>(
    key: string,
    computer: () => Promise<T>,
    ttl: number
  ): Promise<T> {
    // Try cache
    const cached = await this.cache.get(key);
    if (cached) {
      return JSON.parse(cached) as T;
    }
    
    // Check if another request is already computing
    if (this.locks.has(key)) {
      return this.locks.get(key)!;
    }
    
    // Compute and store
    const promise = (async () => {
      try {
        const value = await computer();
        await this.cache.set(key, JSON.stringify(value), ttl);
        return value;
      } finally {
        this.locks.delete(key);
      }
    })();
    
    this.locks.set(key, promise);
    return promise;
  }
}

// Usage
const value = await stampedeProtection.getOrCompute(
  cacheKey,
  () => repository.findById(tenantId, projectId),
  300
);
```

#### Pattern 2: Cache Aside with Refresh

**Keep cache fresh with background refresh:**

```typescript
class RefreshingCache {
  async get<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl: number,
    refreshThreshold: number = 0.8
  ): Promise<T> {
    const cached = await this.cache.get(key);
    
    if (cached) {
      const data = JSON.parse(cached) as { value: T; cachedAt: number };
      const age = Date.now() - data.cachedAt;
      
      // If cache is old (80% of TTL), refresh in background
      if (age > ttl * refreshThreshold * 1000) {
        // Don't await - refresh in background
        this.refreshCache(key, fetcher, ttl);
      }
      
      return data.value;
    }
    
    // Cache miss - fetch and store
    const value = await fetcher();
    await this.storeWithTimestamp(key, value, ttl);
    return value;
  }
  
  private async refreshCache<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl: number
  ): Promise<void> {
    try {
      const value = await fetcher();
      await this.storeWithTimestamp(key, value, ttl);
    } catch (error) {
      console.error('Background refresh failed:', error);
    }
  }
  
  private async storeWithTimestamp<T>(
    key: string,
    value: T,
    ttl: number
  ): Promise<void> {
    const data = {
      value,
      cachedAt: Date.now()
    };
    await this.cache.set(key, JSON.stringify(data), ttl);
  }
}
```

#### Pattern 3: Multi-Level Caching

**Cache at multiple levels:**

```
Request
  ↓
L1: In-Memory Cache (fastest, small)
  ↓ (miss)
L2: Redis Cache (fast, larger)
  ↓ (miss)
Database (slow)
```

```typescript
class MultiLevelCache {
  private memoryCache = new Map<string, { value: any; expiresAt: number }>();
  
  async get(key: string): Promise<string | null> {
    // L1: Memory cache
    const memoryCached = this.memoryCache.get(key);
    if (memoryCached && memoryCached.expiresAt > Date.now()) {
      return memoryCached.value;
    }
    
    // L2: Redis cache
    const redisCached = await this.redis.get(key);
    if (redisCached) {
      // Store in memory cache
      this.memoryCache.set(key, {
        value: redisCached,
        expiresAt: Date.now() + 60000 // 1 minute in memory
      });
      return redisCached;
    }
    
    return null;
  }
  
  async set(key: string, value: string, ttl: number): Promise<void> {
    // Store in Redis
    await this.redis.set(key, value, ttl);
    
    // Store in memory (shorter TTL)
    this.memoryCache.set(key, {
      value,
      expiresAt: Date.now() + Math.min(ttl * 1000, 60000)
    });
  }
}
```

***

### Testing Cached Repository

```typescript
// tests/repositories/CachedProjectRepository.test.ts
describe('CachedProjectRepository', () => {
  let baseRepo: jest.Mocked<IProjectRepository>;
  let redis: jest.Mocked<RedisClient>;
  let keyGen: CacheKeyGenerator;
  let cachedRepo: CachedProjectRepository;
  
  beforeEach(() => {
    baseRepo = {
      findById: jest.fn(),
      save: jest.fn(),
      delete: jest.fn(),
      // ... other methods
    } as any;
    
    redis = {
      get: jest.fn(),
      set: jest.fn(),
      del: jest.fn(),
      delPattern: jest.fn()
    } as any;
    
    keyGen = new CacheKeyGenerator('test');
    cachedRepo = new CachedProjectRepository(baseRepo, redis, keyGen);
  });
  
  describe('findById', () => {
    it('should return cached value on cache hit', async () => {
      const project = { id: '1', name: 'Test' };
      redis.get.mockResolvedValue(JSON.stringify(project));
      
      const result = await cachedRepo.findById('tenant-1', '1');
      
      expect(result).toEqual(project);
      expect(redis.get).toHaveBeenCalledWith('test:tenant:tenant-1:project:1');
      expect(baseRepo.findById).not.toHaveBeenCalled(); // Database not hit
    });
    
    it('should fetch from database on cache miss', async () => {
      const project = { id: '1', name: 'Test' };
      redis.get.mockResolvedValue(null); // Cache miss
      baseRepo.findById.mockResolvedValue(project as any);
      
      const result = await cachedRepo.findById('tenant-1', '1');
      
      expect(result).toEqual(project);
      expect(baseRepo.findById).toHaveBeenCalledWith('tenant-1', '1');
      expect(redis.set).toHaveBeenCalled(); // Cached for next time
    });
  });
  
  describe('save', () => {
    it('should invalidate cache on save', async () => {
      const project = { id: '1', tenantId: 'tenant-1', name: 'Test' };
      baseRepo.save.mockResolvedValue(project as any);
      
      await cachedRepo.save(project as any);
      
      expect(baseRepo.save).toHaveBeenCalledWith(project);
      expect(redis.delPattern).toHaveBeenCalledWith('test:tenant:tenant-1:project*');
    });
  });
});
```

***

### Performance Comparison

**Without Caching:**

```
10,000 requests for same project
- Database: 50ms per query
- Total time: 500 seconds (8.3 minutes)
- Database load: 10,000 queries
```

**With Caching:**

```
10,000 requests for same project
- First request: 50ms (cache miss)
- Next 9,999 requests: 2ms each (cache hits)
- Total time: 50ms + (9,999 × 2ms) = 20 seconds
- Database load: 1 query
- Speed improvement: 25x faster
```

***

### Best Practices

✅ **DO:**

* Cache frequently accessed data
* Use appropriate TTLs (short for changing data, long for stable)
* Invalidate on writes
* Monitor cache hit rates
* Use consistent key naming
* Handle cache failures gracefully

❌ **DON'T:**

* Cache everything (memory cost)
* Use very long TTLs for changing data
* Forget to invalidate on updates
* Cache user-specific sensitive data without encryption
* Ignore cache stampede problem
* Cache large objects (>1MB)

***

### Summary

**Cached Repository Pattern:**

* Wraps existing repository with caching
* Transparent to application code
* 10-100x performance improvement
* Reduces database load significantly
* Critical for high-traffic SaaS

**Next:** Pattern 7.6 (Read/Write Separation) or move to next structural pattern?

## Pattern 7.6: Read/Write Repository Separation

### The Problem

Most applications have different requirements for reads vs writes:

**Reads (Queries):**

* High volume (90% of operations)
* Complex joins and aggregations
* Can use denormalized data
* Can tolerate slight staleness
* Need to be fast

**Writes (Commands):**

* Lower volume (10% of operations)
* Need ACID guarantees
* Must be consistent immediately
* Need validation and business rules
* Can be slower

**Single Repository Problem:**

```typescript
// ❌ BAD: One repository trying to serve both needs
class ProjectRepository {
  // Write operations need normalized data
  async save(project: Project) {
    await db.query('INSERT INTO projects...');
    await db.query('INSERT INTO project_stats...');
    await db.query('UPDATE tenant_aggregates...');
  }
  
  // Read operations want denormalized data
  async findWithDetails(projectId: string) {
    // Complex joins to get everything
    return db.query(`
      SELECT p.*, u.name as creator_name, 
             COUNT(t.id) as task_count,
             AVG(t.completion) as avg_completion
      FROM projects p
      JOIN users u ON p.created_by = u.id
      LEFT JOIN tasks t ON t.project_id = p.id
      WHERE p.id = ?
      GROUP BY p.id, u.name
    `);
  }
}
```

### The Solution: CQRS-Lite (Command Query Responsibility Segregation)

Separate repositories for reads and writes:

```
Write Operations (Commands)
     ↓
Write Model (Normalized)
     ↓
Main Database
     ↓ (sync)
Read Model (Denormalized)
     ↑
Read Operations (Queries)
```

**Benefits:**

* Optimize each side independently
* Scale reads and writes separately
* Simpler code (each repository has one job)
* Better performance

**Note:** This is "CQRS-Lite" because we're using the same database. Full CQRS uses separate databases.

***

### Implementation: TypeScript/Node.js

#### Step 1: Define Separate Interfaces

```typescript
// domain/repositories/IProjectWriteRepository.ts
export interface IProjectWriteRepository {
  // Commands - modify state
  create(project: Project): Promise<Project>;
  update(project: Project): Promise<Project>;
  delete(tenantId: string, projectId: string): Promise<boolean>;
  
  // Batch operations
  createMany(projects: Project[]): Promise<Project[]>;
  updateMany(projects: Project[]): Promise<Project[]>;
  deleteMany(tenantId: string, projectIds: string[]): Promise<number>;
}

// domain/repositories/IProjectReadRepository.ts
export interface IProjectReadRepository {
  // Queries - no state modification
  findById(tenantId: string, projectId: string): Promise<ProjectReadModel | null>;
  findAll(tenantId: string): Promise<ProjectReadModel[]>;
  findByStatus(tenantId: string, status: string): Promise<ProjectReadModel[]>;
  search(tenantId: string, query: string): Promise<ProjectReadModel[]>;
  
  // Complex queries with aggregations
  findWithStats(tenantId: string, projectId: string): Promise<ProjectWithStats | null>;
  findDashboardData(tenantId: string): Promise<DashboardData>;
  findByDateRange(tenantId: string, start: Date, end: Date): Promise<ProjectReadModel[]>;
  
  // Pagination
  findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: any
  ): Promise<PaginatedResult<ProjectReadModel>>;
  
  // Counts and aggregations
  count(tenantId: string): Promise<number>;
  countByStatus(tenantId: string): Promise<{ status: string; count: number }[]>;
  getTotalBudget(tenantId: string): Promise<number>;
}
```

#### Step 2: Define Read Models

Read models can be different from write models - optimized for display:

```typescript
// domain/models/ProjectReadModel.ts

// Simple read model
export interface ProjectReadModel {
  id: string;
  tenantId: string;
  name: string;
  description: string;
  status: string;
  budget: number | null;
  startDate: Date | null;
  endDate: Date | null;
  
  // Denormalized data for display
  creatorName: string;
  creatorEmail: string;
  taskCount: number;
  completedTaskCount: number;
  teamMemberCount: number;
  
  // Computed fields
  completionPercentage: number;
  isOverdue: boolean;
  daysRemaining: number | null;
  
  // Timestamps
  createdAt: Date;
  updatedAt: Date;
}

// Complex read model with stats
export interface ProjectWithStats {
  project: ProjectReadModel;
  stats: {
    totalTasks: number;
    completedTasks: number;
    inProgressTasks: number;
    todoTasks: number;
    overdueTasks: number;
    avgTaskCompletionTime: number;
    budgetUsed: number;
    budgetRemaining: number;
  };
  recentActivity: Activity[];
  teamMembers: TeamMember[];
}

// Dashboard read model
export interface DashboardData {
  projectCount: number;
  activeProjectCount: number;
  completedProjectCount: number;
  totalBudget: number;
  totalBudgetUsed: number;
  overdueProjectCount: number;
  recentProjects: ProjectReadModel[];
  projectsByStatus: { status: string; count: number }[];
  upcomingDeadlines: { projectId: string; projectName: string; deadline: Date }[];
}
```

#### Step 3: Implement Write Repository

```typescript
// infrastructure/repositories/ProjectWriteRepository.ts
import { Pool } from 'pg';
import { IProjectWriteRepository } from '../../domain/repositories/IProjectWriteRepository';
import { Project } from '../../domain/entities/Project';

export class ProjectWriteRepository implements IProjectWriteRepository {
  constructor(private db: Pool) {}
  
  async create(project: Project): Promise<Project> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');
      
      // Insert into normalized projects table
      await client.query(
        `INSERT INTO projects (
          id, tenant_id, name, description, status, budget,
          start_date, end_date, created_by, created_at, updated_at
        )
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)`,
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
      
      // Update denormalized read model
      await this.updateReadModel(client, project);
      
      // Update aggregates
      await client.query(
        `INSERT INTO tenant_project_stats (tenant_id, project_count, last_updated)
         VALUES ($1, 1, NOW())
         ON CONFLICT (tenant_id)
         DO UPDATE SET 
           project_count = tenant_project_stats.project_count + 1,
           last_updated = NOW()`,
        [project.tenantId]
      );
      
      await client.query('COMMIT');
      return project;
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  async update(project: Project): Promise<Project> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');
      
      // Update normalized table
      await client.query(
        `UPDATE projects
         SET name = $1, description = $2, status = $3,
             budget = $4, start_date = $5, end_date = $6,
             updated_at = $7
         WHERE id = $8 AND tenant_id = $9`,
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
      
      // Update read model
      await this.updateReadModel(client, project);
      
      await client.query('COMMIT');
      return project;
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  async delete(tenantId: string, projectId: string): Promise<boolean> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');
      
      // Delete from normalized table
      const result = await client.query(
        `DELETE FROM projects
         WHERE id = $1 AND tenant_id = $2
         RETURNING id`,
        [projectId, tenantId]
      );
      
      if (result.rows.length === 0) {
        await client.query('ROLLBACK');
        return false;
      }
      
      // Delete from read model
      await client.query(
        `DELETE FROM projects_read_model
         WHERE id = $1 AND tenant_id = $2`,
        [projectId, tenantId]
      );
      
      // Update aggregates
      await client.query(
        `UPDATE tenant_project_stats
         SET project_count = project_count - 1,
             last_updated = NOW()
         WHERE tenant_id = $1`,
        [tenantId]
      );
      
      await client.query('COMMIT');
      return true;
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  async createMany(projects: Project[]): Promise<Project[]> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');
      
      for (const project of projects) {
        await client.query(
          `INSERT INTO projects (
            id, tenant_id, name, description, status, budget,
            start_date, end_date, created_by, created_at, updated_at
          )
          VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)`,
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
        
        await this.updateReadModel(client, project);
      }
      
      await client.query('COMMIT');
      return projects;
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  // Helper: Update denormalized read model
  private async updateReadModel(client: any, project: Project): Promise<void> {
    // Get additional data for read model
    const userData = await client.query(
      `SELECT name, email FROM users WHERE id = $1`,
      [project.createdBy]
    );
    
    const taskStats = await client.query(
      `SELECT 
         COUNT(*) as total_tasks,
         COUNT(*) FILTER (WHERE status = 'completed') as completed_tasks
       FROM tasks
       WHERE project_id = $1`,
      [project.id]
    );
    
    const teamCount = await client.query(
      `SELECT COUNT(DISTINCT user_id) as team_count
       FROM project_members
       WHERE project_id = $1`,
      [project.id]
    );
    
    const user = userData.rows[0] || { name: 'Unknown', email: '' };
    const stats = taskStats.rows[0] || { total_tasks: 0, completed_tasks: 0 };
    const team = teamCount.rows[0] || { team_count: 0 };
    
    // Calculate computed fields
    const completionPercentage = stats.total_tasks > 0
      ? Math.round((stats.completed_tasks / stats.total_tasks) * 100)
      : 0;
    
    const isOverdue = project.endDate
      ? new Date() > project.endDate && project.status === 'active'
      : false;
    
    const daysRemaining = project.endDate
      ? Math.ceil((project.endDate.getTime() - Date.now()) / (1000 * 60 * 60 * 24))
      : null;
    
    // Upsert into read model table
    await client.query(
      `INSERT INTO projects_read_model (
        id, tenant_id, name, description, status, budget,
        start_date, end_date, creator_name, creator_email,
        task_count, completed_task_count, team_member_count,
        completion_percentage, is_overdue, days_remaining,
        created_at, updated_at
      )
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16, $17, $18)
      ON CONFLICT (id)
      DO UPDATE SET
        name = $3, description = $4, status = $5, budget = $6,
        start_date = $7, end_date = $8, creator_name = $9, creator_email = $10,
        task_count = $11, completed_task_count = $12, team_member_count = $13,
        completion_percentage = $14, is_overdue = $15, days_remaining = $16,
        updated_at = $18`,
      [
        project.id,
        project.tenantId,
        project.name,
        project.description,
        project.status,
        project.budget,
        project.startDate,
        project.endDate,
        user.name,
        user.email,
        stats.total_tasks,
        stats.completed_tasks,
        team.team_count,
        completionPercentage,
        isOverdue,
        daysRemaining,
        project.createdAt,
        project.updatedAt
      ]
    );
  }
}
```

#### Step 4: Implement Read Repository

```typescript
// infrastructure/repositories/ProjectReadRepository.ts
import { Pool } from 'pg';
import { IProjectReadRepository } from '../../domain/repositories/IProjectReadRepository';
import {
  ProjectReadModel,
  ProjectWithStats,
  DashboardData
} from '../../domain/models/ProjectReadModel';

export class ProjectReadRepository implements IProjectReadRepository {
  constructor(private db: Pool) {}
  
  async findById(tenantId: string, projectId: string): Promise<ProjectReadModel | null> {
    // Simple query from denormalized read model
    const result = await this.db.query(
      `SELECT * FROM projects_read_model
       WHERE id = $1 AND tenant_id = $2`,
      [projectId, tenantId]
    );
    
    if (result.rows.length === 0) {
      return null;
    }
    
    return this.mapToReadModel(result.rows[0]);
  }
  
  async findAll(tenantId: string): Promise<ProjectReadModel[]> {
    const result = await this.db.query(
      `SELECT * FROM projects_read_model
       WHERE tenant_id = $1
       ORDER BY created_at DESC`,
      [tenantId]
    );
    
    return result.rows.map(row => this.mapToReadModel(row));
  }
  
  async findByStatus(tenantId: string, status: string): Promise<ProjectReadModel[]> {
    const result = await this.db.query(
      `SELECT * FROM projects_read_model
       WHERE tenant_id = $1 AND status = $2
       ORDER BY created_at DESC`,
      [tenantId, status]
    );
    
    return result.rows.map(row => this.mapToReadModel(row));
  }
  
  async search(tenantId: string, query: string): Promise<ProjectReadModel[]> {
    const result = await this.db.query(
      `SELECT * FROM projects_read_model
       WHERE tenant_id = $1
         AND (
           name ILIKE $2
           OR description ILIKE $2
           OR creator_name ILIKE $2
         )
       ORDER BY created_at DESC`,
      [tenantId, `%${query}%`]
    );
    
    return result.rows.map(row => this.mapToReadModel(row));
  }
  
  async findWithStats(
    tenantId: string,
    projectId: string
  ): Promise<ProjectWithStats | null> {
    // Complex query joining multiple tables
    const projectResult = await this.db.query(
      `SELECT * FROM projects_read_model
       WHERE id = $1 AND tenant_id = $2`,
      [projectId, tenantId]
    );
    
    if (projectResult.rows.length === 0) {
      return null;
    }
    
    // Get detailed stats
    const statsResult = await this.db.query(
      `SELECT
         COUNT(*) as total_tasks,
         COUNT(*) FILTER (WHERE status = 'completed') as completed_tasks,
         COUNT(*) FILTER (WHERE status = 'in_progress') as in_progress_tasks,
         COUNT(*) FILTER (WHERE status = 'todo') as todo_tasks,
         COUNT(*) FILTER (WHERE status != 'completed' AND due_date < NOW()) as overdue_tasks,
         AVG(EXTRACT(EPOCH FROM (completed_at - created_at))/3600) as avg_completion_hours
       FROM tasks
       WHERE project_id = $1`,
      [projectId]
    );
    
    // Get budget usage
    const budgetResult = await this.db.query(
      `SELECT COALESCE(SUM(amount), 0) as budget_used
       FROM project_expenses
       WHERE project_id = $1`,
      [projectId]
    );
    
    // Get recent activity
    const activityResult = await this.db.query(
      `SELECT * FROM activities
       WHERE entity_type = 'project'
         AND entity_id = $1
       ORDER BY created_at DESC
       LIMIT 10`,
      [projectId]
    );
    
    // Get team members
    const teamResult = await this.db.query(
      `SELECT u.id, u.name, u.email, u.avatar_url, pm.role
       FROM project_members pm
       JOIN users u ON pm.user_id = u.id
       WHERE pm.project_id = $1`,
      [projectId]
    );
    
    const project = this.mapToReadModel(projectResult.rows[0]);
    const stats = statsResult.rows[0];
    const budgetData = budgetResult.rows[0];
    
    return {
      project,
      stats: {
        totalTasks: parseInt(stats.total_tasks),
        completedTasks: parseInt(stats.completed_tasks),
        inProgressTasks: parseInt(stats.in_progress_tasks),
        todoTasks: parseInt(stats.todo_tasks),
        overdueTasks: parseInt(stats.overdue_tasks),
        avgTaskCompletionTime: parseFloat(stats.avg_completion_hours) || 0,
        budgetUsed: parseFloat(budgetData.budget_used),
        budgetRemaining: (project.budget || 0) - parseFloat(budgetData.budget_used)
      },
      recentActivity: activityResult.rows,
      teamMembers: teamResult.rows
    };
  }
  
  async findDashboardData(tenantId: string): Promise<DashboardData> {
    // Multiple queries for dashboard (could be one complex query)
    const [countsResult, budgetResult, recentResult, statusResult, deadlinesResult] = await Promise.all([
      // Project counts
      this.db.query(
        `SELECT
           COUNT(*) as project_count,
           COUNT(*) FILTER (WHERE status = 'active') as active_count,
           COUNT(*) FILTER (WHERE status = 'completed') as completed_count,
           COUNT(*) FILTER (WHERE is_overdue = true) as overdue_count
         FROM projects_read_model
         WHERE tenant_id = $1`,
        [tenantId]
      ),
      
      // Budget totals
      this.db.query(
        `SELECT
           COALESCE(SUM(budget), 0) as total_budget,
           COALESCE(SUM(pe.amount), 0) as total_used
         FROM projects_read_model p
         LEFT JOIN project_expenses pe ON p.id = pe.project_id
         WHERE p.tenant_id = $1`,
        [tenantId]
      ),
      
      // Recent projects
      this.db.query(
        `SELECT * FROM projects_read_model
         WHERE tenant_id = $1
         ORDER BY created_at DESC
         LIMIT 5`,
        [tenantId]
      ),
      
      // Projects by status
      this.db.query(
        `SELECT status, COUNT(*) as count
         FROM projects_read_model
         WHERE tenant_id = $1
         GROUP BY status`,
        [tenantId]
      ),
      
      // Upcoming deadlines
      this.db.query(
        `SELECT id, name, end_date
         FROM projects_read_model
         WHERE tenant_id = $1
           AND status = 'active'
           AND end_date IS NOT NULL
           AND end_date > NOW()
           AND end_date < NOW() + INTERVAL '7 days'
         ORDER BY end_date ASC`,
        [tenantId]
      )
    ]);
    
    const counts = countsResult.rows[0];
    const budget = budgetResult.rows[0];
    
    return {
      projectCount: parseInt(counts.project_count),
      activeProjectCount: parseInt(counts.active_count),
      completedProjectCount: parseInt(counts.completed_count),
      totalBudget: parseFloat(budget.total_budget),
      totalBudgetUsed: parseFloat(budget.total_used),
      overdueProjectCount: parseInt(counts.overdue_count),
      recentProjects: recentResult.rows.map(row => this.mapToReadModel(row)),
      projectsByStatus: statusResult.rows.map(row => ({
        status: row.status,
        count: parseInt(row.count)
      })),
      upcomingDeadlines: deadlinesResult.rows.map(row => ({
        projectId: row.id,
        projectName: row.name,
        deadline: row.end_date
      }))
    };
  }
  
  async findWithPagination(
    tenantId: string,
    page: number,
    limit: number,
    filters?: any
  ): Promise<PaginatedResult<ProjectReadModel>> {
    let whereClause = 'WHERE tenant_id = $1';
    const params: any[] = [tenantId];
    let paramIndex = 2;
    
    if (filters?.status) {
      whereClause += ` AND status = $${paramIndex}`;
      params.push(filters.status);
      paramIndex++;
    }
    
    if (filters?.search) {
      whereClause += ` AND (name ILIKE $${paramIndex} OR description ILIKE $${paramIndex})`;
      params.push(`%${filters.search}%`);
      paramIndex++;
    }
    
    // Count total
    const countResult = await this.db.query(
      `SELECT COUNT(*) FROM projects_read_model ${whereClause}`,
      params
    );
    
    // Get page
    const offset = (page - 1) * limit;
    const dataResult = await this.db.query(
      `SELECT * FROM projects_read_model
       ${whereClause}
       ORDER BY created_at DESC
       LIMIT $${paramIndex} OFFSET $${paramIndex + 1}`,
      [...params, limit, offset]
    );
    
    const total = parseInt(countResult.rows[0].count);
    
    return {
      data: dataResult.rows.map(row => this.mapToReadModel(row)),
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit)
    };
  }
  
  async count(tenantId: string): Promise<number> {
    const result = await this.db.query(
      `SELECT COUNT(*) as count FROM projects_read_model
       WHERE tenant_id = $1`,
      [tenantId]
    );
    
    return parseInt(result.rows[0].count);
  }
  
  async countByStatus(tenantId: string): Promise<{ status: string; count: number }[]> {
    const result = await this.db.query(
      `SELECT status, COUNT(*) as count
       FROM projects_read_model
       WHERE tenant_id = $1
       GROUP BY status
       ORDER BY count DESC`,
      [tenantId]
    );
    
    return result.rows.map(row => ({
      status: row.status,
      count: parseInt(row.count)
    }));
  }
  
  async getTotalBudget(tenantId: string): Promise<number> {
    const result = await this.db.query(
      `SELECT COALESCE(SUM(budget), 0) as total
       FROM projects_read_model
       WHERE tenant_id = $1`,
      [tenantId]
    );
    
    return parseFloat(result.rows[0].total);
  }
  
  private mapToReadModel(row: any): ProjectReadModel {
    return {
      id: row.id,
      tenantId: row.tenant_id,
      name: row.name,
      description: row.description,
      status: row.status,
      budget: row.budget,
      startDate: row.start_date,
      endDate: row.end_date,
      creatorName: row.creator_name,
      creatorEmail: row.creator_email,
      taskCount: row.task_count,
      completedTaskCount: row.completed_task_count,
      teamMemberCount: row.team_member_count,
      completionPercentage: row.completion_percentage,
      isOverdue: row.is_overdue,
      daysRemaining: row.days_remaining,
      createdAt: row.created_at,
      updatedAt: row.updated_at
    };
  }
}
```

#### Step 5: Database Schema

```sql
-- Write model (normalized)
CREATE TABLE projects (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  status VARCHAR(50) NOT NULL,
  budget DECIMAL(15, 2),
  start_date TIMESTAMP,
  end_date TIMESTAMP,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  
  CONSTRAINT check_status CHECK (status IN ('active', 'completed', 'archived'))
);

CREATE INDEX idx_projects_tenant ON projects(tenant_id);
CREATE INDEX idx_projects_status ON projects(tenant_id, status);

-- Read model (denormalized)
CREATE TABLE projects_read_model (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  status VARCHAR(50) NOT NULL,
  budget DECIMAL(15, 2),
  start_date TIMESTAMP,
  end_date TIMESTAMP,
  
  -- Denormalized user data
  creator_name VARCHAR(255),
  creator_email VARCHAR(255),
  
  -- Denormalized stats
  task_count INTEGER DEFAULT 0,
  completed_task_count INTEGER DEFAULT 0,
  team_member_count INTEGER DEFAULT 0,
  
  -- Computed fields
  completion_percentage INTEGER DEFAULT 0,
  is_overdue BOOLEAN DEFAULT FALSE,
  days_remaining INTEGER,
  
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);

CREATE INDEX idx_projects_read_tenant ON projects_read_model(tenant_id);
CREATE INDEX idx_projects_read_status ON projects_read_model(tenant_id, status);
CREATE INDEX idx_projects_read_overdue ON projects_read_model(tenant_id, is_overdue);
CREATE INDEX idx_projects_read_search ON projects_read_model 
  USING gin(to_tsvector('english', name || ' ' || description));

-- Aggregate table
CREATE TABLE tenant_project_stats (
  tenant_id UUID PRIMARY KEY REFERENCES tenants(id),
  project_count INTEGER DEFAULT 0,
  active_project_count INTEGER DEFAULT 0,
  completed_project_count INTEGER DEFAULT 0,
  total_budget DECIMAL(20, 2) DEFAULT 0,
  last_updated TIMESTAMP NOT NULL DEFAULT NOW()
);
```

#### Step 6: Using Read/Write Repositories in Service

```typescript
// application/services/ProjectService.ts
export class ProjectService {
  constructor(
    private writeRepo: IProjectWriteRepository,
    private readRepo: IProjectReadRepository,
    private eventBus: IEventBus
  ) {}
  
  // Command - uses write repository
  async createProject(
    tenantId: string,
    userId: string,
    data: CreateProjectDTO
  ): Promise<ProjectReadModel> {
    // Create domain entity
    const project = new Project(
      uuid(),
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
    
    // Write to database
    await this.writeRepo.update(project);
    
    // Emit event
    await this.eventBus.emit('project.updated', {
      tenantId,
      projectId,
      updates
    });
    
    // Return read model
    const readModel = await this.readRepo.findById(tenantId, projectId);
    if (!readModel) {
      throw new Error('Failed to read updated project');
    }
    
    return readModel;
  }
  
  // Command - uses write repository
  async deleteProject(tenantId: string, projectId: string): Promise<void> {
    const deleted = await this.writeRepo.delete(tenantId, projectId);
    
    if (!deleted) {
      throw new Error('Project not found');
    }
    
    await this.eventBus.emit('project.deleted', {
      tenantId,
      projectId
    });
  }
  
  // Query - uses read repository
  async getProject(tenantId: string, projectId: string): Promise<ProjectReadModel> {
    const project = await this.readRepo.findById(tenantId, projectId);
    
    if (!project) {
      throw new Error('Project not found');
    }
    
    return project;
  }
  
  // Query - uses read repository
  async getProjectWithStats(
    tenantId: string,
    projectId: string
  ): Promise<ProjectWithStats> {
    const projectWithStats = await this.readRepo.findWithStats(tenantId, projectId);
    
    if (!projectWithStats) {
      throw new Error('Project not found');
    }
    
    return projectWithStats;
  }
  
  // Query - uses read repository
  async listProjects(
    tenantId: string,
    options: {
      page?: number;
      limit?: number;
      status?: string;
      search?: string;
    } = {}
  ): Promise<PaginatedResult<ProjectReadModel>> {
    const page = options.page || 1;
    const limit = options.limit || 20;
    
    return this.readRepo.findWithPagination(tenantId, page, limit, {
      status: options.status,
      search: options.search
    });
  }
  
  // Query - uses read repository
  async searchProjects(tenantId: string, query: string): Promise<ProjectReadModel[]> {
    return this.readRepo.search(tenantId, query);
  }
  
  // Query - uses read repository
  async getDashboard(tenantId: string): Promise<DashboardData> {
    return this.readRepo.findDashboardData(tenantId);
  }
  
  // Query - uses read repository
  async getProjectsByStatus(tenantId: string, status: string): Promise<ProjectReadModel[]> {
    return this.readRepo.findByStatus(tenantId, status);
  }
}
```

***

### Advanced: Background Synchronization

For true CQRS with separate databases, use background sync:

```typescript
// infrastructure/sync/ReadModelSynchronizer.ts
export class ReadModelSynchronizer {
  constructor(
    private eventBus: IEventBus,
    private readRepo: IProjectReadRepository
  ) {
    this.setupEventHandlers();
  }
  
  private setupEventHandlers(): void {
    // Listen to domain events and update read model
    
    this.eventBus.on('project.created', async (event) => {
      // Read model is already updated in write repository
      // But if using separate database, update here:
      // await this.readRepo.updateFromEvent(event);
      console.log('Read model synchronized for project.created');
    });
    
    this.eventBus.on('project.updated', async (event) => {
      // Update read model
      console.log('Read model synchronized for project.updated');
    });
    
    this.eventBus.on('task.completed', async (event) => {
      // Update project stats in read model
      await this.updateProjectTaskStats(event.projectId);
    });
    
    this.eventBus.on('team_member.added', async (event) => {
      // Update team member count in read model
      await this.updateProjectTeamCount(event.projectId);
    });
  }
  
  private async updateProjectTaskStats(projectId: string): Promise<void> {
    // Recalculate task statistics and update read model
    // This keeps denormalized data in sync
  }
  
  private async updateProjectTeamCount(projectId: string): Promise<void> {
    // Recalculate team member count and update read model
  }
}
```

***

### Benefits of Read/Write Separation

#### Performance Benefits

**Read Performance:**

```
Without separation:
- Complex JOIN on 5 tables: 200ms
- Must compute aggregates: +50ms
- Total: 250ms per request

With separation:
- Simple SELECT from denormalized table: 5ms
- Pre-computed aggregates: 0ms
- Total: 5ms per request
- 50x faster!
```

**Write Performance:**

```
Without separation:
- Update normalized data: 20ms
- Update aggregates: 10ms
- Total: 30ms

With separation:
- Update normalized data: 20ms
- Update read model (async): 15ms
- Total: 35ms (slightly slower, but acceptable)
- Reads are 50x faster, so overall better
```

#### Scalability Benefits

```
Traditional:
- 1 database serves reads and writes
- Reads (90% traffic) compete with writes
- Database becomes bottleneck

Read/Write Separation:
- Write database: Optimized for writes (SSD, high IOPS)
- Read database: Optimized for reads (can be replica, cached)
- Can add read replicas easily
- Scale reads independently from writes
```

***

### When to Use Read/Write Separation

#### ✅ Use When:

1. **High Read/Write Ratio**

   * 90%+ reads, <10% writes
   * Dashboard-heavy applications
   * Reporting systems

2. **Complex Read Queries**

   * Multiple JOINs
   * Heavy aggregations
   * Real-time dashboards

3. **Different Scaling Needs**

   * Reads need to scale horizontally
   * Writes need transactional guarantees

4. **Performance Critical**

   * Sub-second response times required
   * High traffic (1000+ req/sec)

#### ❌ Don't Use When:

1. **Simple CRUD**

   * No complex queries
   * Simple data model
   * Low traffic

2. **Immediate Consistency Required**

   * Financial transactions
   * Inventory management
   * Real-time bidding

3. **Small Team**

   * Added complexity
   * More code to maintain
   * Increased testing burden

4. **Early Stage MVP**

   * Premature optimization
   * Start simple, add later if needed

***

### Eventual Consistency Considerations

#### The Challenge

With read/write separation, there's a delay between write and read:

```typescript
// Write
await writeRepo.create(project);

// Read immediately
const project = await readRepo.findById(projectId);
// Might not be there yet if async sync!
```

#### Solution 1: Synchronous Updates

Update both models in same transaction (what we did above):

```typescript
async create(project: Project): Promise<Project> {
  await client.query('BEGIN');
  
  // Update write model
  await client.query('INSERT INTO projects...');
  
  // Update read model (same transaction)
  await client.query('INSERT INTO projects_read_model...');
  
  await client.query('COMMIT');
}
```

**Pros:** Immediate consistency\
**Cons:** Slower writes, tight coupling

#### Solution 2: Return Write Model

Return data from write operation:

```typescript
async createProject(data): Promise<ProjectReadModel> {
  // Write
  const project = await writeRepo.create(data);
  
  // Convert write model to read model
  return this.toReadModel(project);
  // Don't wait for read model to update
}
```

**Pros:** Fast, no waiting\
**Cons:** Returned data might differ from subsequent reads

#### Solution 3: Eventual Consistency with Polling

Accept eventual consistency, retry if needed:

```typescript
async createProjectWithRetry(data): Promise<ProjectReadModel> {
  // Write
  const project = await writeRepo.create(data);
  
  // Poll read model until available
  for (let i = 0; i < 10; i++) {
    const readModel = await readRepo.findById(project.id);
    if (readModel) {
      return readModel;
    }
    await sleep(100); // Wait 100ms
  }
  
  throw new Error('Read model not synchronized');
}
```

**Pros:** Eventually consistent\
**Cons:** Added latency, complexity

***

### Real-World Example: E-commerce Order

```typescript
// Write Model (Normalized)
class Order {
  id: string;
  customerId: string;
  status: string;
  createdAt: Date;
}

class OrderItem {
  id: string;
  orderId: string;
  productId: string;
  quantity: number;
  price: number;
}

// Read Model (Denormalized)
interface OrderReadModel {
  id: string;
  customerId: string;
  customerName: string;
  customerEmail: string;
  status: string;
  
  // Denormalized items
  items: {
    productId: string;
    productName: string;
    productImage: string;
    quantity: number;
    price: number;
    subtotal: number;
  }[];
  
  // Computed fields
  itemCount: number;
  subtotal: number;
  tax: number;
  shipping: number;
  total: number;
  
  // Timestamps
  createdAt: Date;
  estimatedDelivery: Date;
}

// Write Repository
class OrderWriteRepository {
  async create(order: Order, items: OrderItem[]): Promise<Order> {
    await db.transaction(async (tx) => {
      // Insert order
      await tx.query('INSERT INTO orders...');
      
      // Insert items
      for (const item of items) {
        await tx.query('INSERT INTO order_items...');
      }
      
      // Update read model
      await this.updateOrderReadModel(tx, order.id);
    });
    
    return order;
  }
  
  private async updateOrderReadModel(tx, orderId): Promise<void> {
    // Join all necessary tables
    const data = await tx.query(`
      SELECT 
        o.id, o.customer_id, o.status, o.created_at,
        c.name as customer_name, c.email as customer_email,
        json_agg(json_build_object(
          'productId', oi.product_id,
          'productName', p.name,
          'productImage', p.image_url,
          'quantity', oi.quantity,
          'price', oi.price,
          'subtotal', oi.quantity * oi.price
        )) as items
      FROM orders o
      JOIN customers c ON o.customer_id = c.id
      JOIN order_items oi ON o.id = oi.order_id
      JOIN products p ON oi.product_id = p.id
      WHERE o.id = $1
      GROUP BY o.id, c.name, c.email
    `, [orderId]);
    
    const orderData = data.rows[0];
    
    // Calculate totals
    const itemsData = JSON.parse(orderData.items);
    const subtotal = itemsData.reduce((sum, item) => sum + item.subtotal, 0);
    const tax = subtotal * 0.1; // 10% tax
    const shipping = subtotal > 100 ? 0 : 10; // Free shipping over $100
    const total = subtotal + tax + shipping;
    
    // Insert/update read model
    await tx.query(`
      INSERT INTO orders_read_model (
        id, customer_id, customer_name, customer_email,
        status, items, item_count, subtotal, tax, shipping, total,
        created_at, estimated_delivery
      )
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13)
      ON CONFLICT (id) DO UPDATE SET
        status = $5, items = $6, item_count = $7,
        subtotal = $8, tax = $9, shipping = $10, total = $11
    `, [
      orderData.id,
      orderData.customer_id,
      orderData.customer_name,
      orderData.customer_email,
      orderData.status,
      JSON.stringify(itemsData),
      itemsData.length,
      subtotal,
      tax,
      shipping,
      total,
      orderData.created_at,
      new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days
    ]);
  }
}

// Read Repository
class OrderReadRepository {
  async findById(orderId: string): Promise<OrderReadModel> {
    // Simple, fast query from denormalized table
    const result = await db.query(
      'SELECT * FROM orders_read_model WHERE id = $1',
      [orderId]
    );
    
    return result.rows[0];
  }
  
  async findCustomerOrders(customerId: string): Promise<OrderReadModel[]> {
    // Fast query, no JOINs needed
    const result = await db.query(
      'SELECT * FROM orders_read_model WHERE customer_id = $1 ORDER BY created_at DESC',
      [customerId]
    );
    
    return result.rows;
  }
}
```

***

### Migration Strategy

#### Step 1: Start with Single Repository

```typescript
// Initial implementation
class ProjectRepository {
  async findById(id): Promise<Project> { }
  async save(project): Promise<Project> { }
}
```

#### Step 2: Identify Read/Write Split

```typescript
// Split into interfaces
interface IProjectReadRepository { }
interface IProjectWriteRepository { }

// Same implementation class implements both
class ProjectRepository implements IProjectReadRepository, IProjectWriteRepository {
  // All methods
}
```

#### Step 3: Create Read Model Table

```sql
-- Add denormalized read table
CREATE TABLE projects_read_model AS
SELECT 
  p.*,
  u.name as creator_name,
  COUNT(t.id) as task_count
FROM projects p
JOIN users u ON p.created_by = u.id
LEFT JOIN tasks t ON t.project_id = p.id
GROUP BY p.id, u.name;
```

#### Step 4: Separate Implementations

```typescript
// Separate classes
class ProjectWriteRepository implements IProjectWriteRepository {
  // Write operations + update read model
}

class ProjectReadRepository implements IProjectReadRepository {
  // Read from denormalized table
}
```

#### Step 5: Update Service

```typescript
// Use both repositories
class ProjectService {
  constructor(
    private writeRepo: IProjectWriteRepository,
    private readRepo: IProjectReadRepository
  ) {}
}
```

***

### Summary

#### Key Takeaways

**Read/Write Separation (CQRS-Lite):**

* Separate repositories for reads and writes
* Denormalized read models for performance
* Optimized queries for each side
* 10-50x performance improvement for reads

**When to Use:**

* High read/write ratio (90/10)
* Complex read queries
* Performance critical
* Need to scale reads independently

**Trade-offs:**

* More complex code
* Potential eventual consistency
* More storage (duplicated data)
* Worth it for high-traffic SaaS

**Implementation:**

* Same database (CQRS-Lite) or separate databases (full CQRS)
* Synchronous updates (immediate consistency)
* Async updates (eventual consistency)
* Event-driven sync (most flexible)

***
