## 1.2 Multi-Tenancy Concepts

Multi-tenancy is THE defining architectural characteristic of most SaaS applications. Understanding this concept is fundamental.

### What is Multi-Tenancy?

**Multi-tenancy** is an architecture where a single instance of the software serves multiple customers (tenants). Each tenant's data is isolated and invisible to other tenants, but they all share the same application instance and infrastructure.

**Think of it like an apartment building:**
- One building (application instance)
- Multiple apartments (tenants)
- Shared infrastructure (plumbing, electricity = database, servers)
- Private spaces (each apartment is isolated)
- Shared amenities (gym, lobby = shared application features)

### Why Multi-Tenancy Matters

**Cost Efficiency:**
- One server can handle multiple customers
- Shared infrastructure costs (database, caching, load balancers)
- Lower per-customer cost = more competitive pricing

**Maintenance:**
- Deploy once, all tenants get the update
- Bug fixes benefit everyone immediately
- No need to coordinate upgrades with customers

**Scaling:**
- Add new tenants without deploying new infrastructure (initially)
- Optimize once, benefit all tenants

### Multi-Tenancy Patterns (High-Level Overview)

#### 1. **Shared Database, Shared Schema (Row-Level Isolation)**

```
Database: saas_app
├── Table: users
│   ├── id | tenant_id | name | email
│   ├── 1  | tenant_a  | John | john@a.com
│   ├── 2  | tenant_b  | Jane | jane@b.com
│
├── Table: projects
│   ├── id | tenant_id | name | owner_id
│   ├── 1  | tenant_a  | Project X | 1
│   ├── 2  | tenant_b  | Project Y | 2
```

**How it works:**
- Every table has a `tenant_id` column
- Application filters ALL queries by tenant_id
- Simplest to implement initially

**Pros:**
- Easiest to implement and maintain
- Most cost-effective (one database)
- Easiest to optimize queries across all tenants
- Simple backups

**Cons:**
- Risk of data leakage (if you forget to filter by tenant_id)
- One tenant's large dataset can slow down others
- Difficult to offer tenant-specific customizations
- Compliance challenges (data residency requirements)
- Complex migrations as tenant count grows

**Best for:** 
- Starting out (0-1000 tenants)
- B2C SaaS with simple data models
- When tenants have similar data sizes
- Cost-sensitive applications

**Examples:** 
- Basecamp (early days)
- Simple project management tools
- Form builders
- Survey tools

#### 2. **Shared Database, Separate Schemas**

```
Database: saas_app
├── Schema: tenant_a
│   ├── Table: users (id, name, email)
│   ├── Table: projects (id, name, owner_id)
│
├── Schema: tenant_b
│   ├── Table: users (id, name, email)
│   ├── Table: projects (id, name, owner_id)
```

**How it works:**
- Each tenant gets their own database schema (namespace)
- Same table structures, but isolated at schema level
- Application switches schema based on tenant context

**Pros:**
- Better isolation than shared schema
- Can restore individual tenant data
- Can customize schema per tenant if needed
- Performance isolation (indexes per tenant)
- Easier to move tenant to different database later

**Cons:**
- More complex than shared schema
- Migrations must run for EACH schema
- Database connection pooling more complex
- Limited by database's schema limit (e.g., PostgreSQL recommends < 100 schemas)

**Best for:**
- Medium-sized B2B SaaS (100-1000 tenants)
- When some tenants need customization
- When data isolation is important but separate DBs are too expensive
- Industries with compliance requirements

**Examples:**
- Salesforce (uses similar approach with their multi-tenant architecture)
- Enterprise CRM systems
- Healthcare SaaS platforms

#### 3. **Separate Databases per Tenant**

```
Database: tenant_a_db
├── Table: users
├── Table: projects

Database: tenant_b_db
├── Table: users
├── Table: projects

Database: tenant_c_db
├── Table: users
├── Table: projects
```

**How it works:**
- Each tenant has a completely separate database
- Application routes to the correct database based on tenant
- Maximum isolation

**Pros:**
- Strongest data isolation and security
- Easy to customize per tenant
- Easy to back up/restore individual tenants
- Can optimize database per tenant (indexes, configuration)
- Easy to move tenants to different servers
- Compliance-friendly (data residency)
- One tenant's issues don't affect others

**Cons:**
- Most expensive (database resources per tenant)
- Complex operational overhead (monitoring hundreds of databases)
- Difficult to run queries across all tenants
- Migrations must run for EACH database
- Connection pooling challenges

**Best for:**
- Enterprise SaaS with large customers
- High-value customers who demand isolation
- Compliance-heavy industries (finance, healthcare)
- When customers have vastly different sizes
- When offering "bring your own database" options

**Examples:**
- Slack (enterprise customers can get dedicated infrastructure)
- GitHub Enterprise
- Enterprise versions of many SaaS products

#### 4. **Hybrid Approaches**

Real-world SaaS often combines approaches:

**Tiered Multi-Tenancy:**
```
Small customers → Shared database, shared schema
Medium customers → Shared database, separate schemas  
Enterprise customers → Separate databases
```

This is the most pragmatic approach for growing SaaS businesses.

**Example Decision Tree:**
- Free/Starter tier: Shared schema (cost optimization)
- Professional tier: Separate schema (better isolation)
- Enterprise tier: Separate database (maximum control)

### Multi-Tenancy Anti-Patterns to Avoid

❌ **Forgetting tenant_id in queries** - The #1 cause of data leakage
❌ **Global caching without tenant context** - Can leak data between tenants
❌ **Not planning for tenant migration** - Gets harder as you grow
❌ **Overcomplicating from day 1** - Start simple (shared schema), evolve
❌ **Ignoring "noisy neighbor" problem** - One tenant shouldn't affect others' performance

### When NOT to Use Multi-Tenancy

Sometimes, you DON'T want multi-tenancy:

1. **Single-tenant SaaS** (each customer gets their own instance)
   - Common in highly regulated industries
   - "Managed service" model
   - Examples: Gitlab self-hosted, Confluence Data Center

2. **Consumer applications** where everyone is the same "tenant"
   - Social media platforms (Twitter, Instagram)
   - Consumer tools (Grammarly)

3. **Extremely high-security requirements**
   - Defense contractors
   - Financial trading platforms

---
