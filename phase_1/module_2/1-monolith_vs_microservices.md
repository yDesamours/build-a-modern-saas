## 2.1 Monolith vs Microservices

This is THE architectural decision that impacts everything else. Let's be clear from the start: **most SaaS applications should start as monoliths**.

### The Monolith-First Approach

#### What is a Monolith?

A **monolithic application** is a single, tightly coupled unified codebase where all functionality is deployed together as one unit. 

```
Monolithic Application
┌───────────────────────────────────────┐
│                                     		   	│
│  ┌──────────┐  ┌──────────┐       	   	│
│  │   API    	│  	│   Web    	 │       		│
│  │  Layer   	│  	│   UI    	 │       		│
│  └────┬─────┘  └─┬────────┘       		│
│        │            │              			│
│  ┌────┴──────────┴───────┐   	   		│
│  │   Business Logic   		│      			│
│  │   - Auth                 	│      			│
│  │   - Projects             	│      			│
│  │   - Tasks                	│      			│
│  │   - Billing              	│      			│
│  └──────────┬────────────┘	      		│
│             	│                       		│
│  ┌──────────┴───────────────┐      		│
│  │   Data Access Layer      		│   		│
│  └──────────┬───────────────┘      		│
│            	 │                       		│
└─────────────┼─────────────────────────┘
				│
        ┌──────┴──────┐
        │ Database  	 │
        └─────────────┘
```

#### Why Start with a Monolith?

**Reason 1: Speed of Development**

When you're building an MVP or starting a SaaS product, speed matters more than scalability:

```javascript
// Monolith: Simple, straightforward
async function createProject(tenantId, userId, projectData) {
  // All in one codebase, easy to reason about
  const project = await db.projects.create({
    ...projectData,
    tenantId,
    createdBy: userId
  });
  
  await db.activities.create({
    tenantId,
    type: 'project_created',
    projectId: project.id,
    userId
  });
  
  await notificationService.notify(tenantId, {
    type: 'project_created',
    projectId: project.id
  });
  
  return project;
}

// Microservices: Complex, requires coordination
async function createProject(tenantId, userId, projectData) {
  // 1. Call Projects Service
  const project = await projectsServiceClient.create({
    ...projectData,
    tenantId,
    createdBy: userId
  });
  
  // 2. Call Activity Service
  await activityServiceClient.create({
    tenantId,
    type: 'project_created',
    projectId: project.id,
    userId
  });
  
  // 3. Publish event to message queue
  await messageQueue.publish('project.created', {
    tenantId,
    projectId: project.id
  });
  
  // 4. Handle potential failures and retries
  // What if activity service fails?
  // What if notification service is down?
  // Need distributed transaction handling
  
  return project;
}
```

**Reason 2: Simpler Operations**

- **Deploy once** vs deploying 20+ services
- **One codebase** to understand vs navigating multiple repos
- **Easier debugging** - all code in one place
- **No network latency** between components
- **Simpler monitoring** - one application to monitor
- **Easier testing** - no need for contract testing between services

**Reason 3: Database Transactions Work**

```javascript
// Monolith: ACID transactions just work
async function transferFunds(fromAccount, toAccount, amount) {
  await db.transaction(async (trx) => {
    // Debit from account
    await trx('accounts')
      .where('id', fromAccount)
      .decrement('balance', amount);
    
    // Credit to account
    await trx('accounts')
      .where('id', toAccount)
      .increment('balance', amount);
    
    // Log transaction
    await trx('transactions').insert({
      fromAccount,
      toAccount,
      amount,
      timestamp: new Date()
    });
    
    // If any step fails, entire transaction rolls back
  });
}

// Microservices: Need distributed transaction patterns (Saga, etc.)
// Much more complex, error-prone
```

**Reason 4: You Don't Have Microservice Problems Yet**

Microservices solve specific problems:
- ✅ Different parts need to scale independently
- ✅ Different teams working on different services
- ✅ Need to deploy parts independently
- ✅ Different technology stacks for different services

**If you have < 100K users and < 10 engineers, you DON'T have these problems yet.**

#### The Well-Structured Monolith

The key is building a **modular monolith** that can be split later if needed.

**Definition:** The modular monolith architectural pattern structures the application into independent modules or components with well-defined boundaries. The modules are split based on logical boundaries, grouping together related functionalities.

```
src/
├── modules/
│   ├── auth/
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── models/
│   │   └── index.ts          # Public API of auth module
│   │
│   ├── projects/
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── models/
│   │   └── index.ts
│   │
│   ├── billing/
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── models/
│   │   └── index.ts
│   │
│   └── notifications/
│       ├── controllers/
│       ├── services/
│       ├── repositories/
│       ├── models/
│       └── index.ts
│
├── shared/
│   ├── database/
│   ├── middleware/
│   ├── utils/
│   └── types/
│
└── app.ts
```

**Key Principles:**

1. **Clear module boundaries** - Each module is self-contained
2. **Defined interfaces** - Modules communicate through well-defined APIs
3. **Loose coupling** - Modules don't directly access each other's internals
4. **High cohesion** - Related functionality stays together

**Example: Module Communication**

```javascript
// ❌ BAD: Direct database access across modules
// In projects/services/ProjectService.js
async function createProject(data) {
  const project = await db.projects.create(data);
  
  // DON'T DO THIS - directly accessing another module's database
  await db.notifications.create({
    type: 'project_created',
    projectId: project.id
  });
}

// ✅ GOOD: Use the other module's public API
// In projects/services/ProjectService.js
const { notificationService } = require('../../notifications');

async function createProject(data) {
  const project = await db.projects.create(data);
  
  // Use notification module's service (public API)
  await notificationService.send({
    type: 'project_created',
    projectId: project.id
  });
  
  return project;
}
```

---

### Microservices

#### What Are Microservices?

**Microservices** are an architectural style where an application is composed of small, independent services that communicate over a network. Each service owns its data and logic, and is developed, deployed, and scaled independently.

```
Microservices Architecture
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   API Gateway   │  │   Web Client    │  │   Mobile App    │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
┌────────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐
│  Auth Service   │  │  Projects       │  │  Billing        │
│                 │  │  Service        │  │  Service        │
│ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │
│ │ Business    │ │  │ │ Business    │ │  │ │ Business    │ │
│ │ Logic       │ │  │ │ Logic       │ │  │ │ Logic       │ │
│ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │
│ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │
│ │ Auth        │ │  │ │ Projects    │ │  │ │ Billing     │ │
│ │ Database    │ │  │ │ Database    │ │  │ │ Database    │ │
│ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                ┌─────────────▼─────────────┐
                │   Message Queue /         │
                │   Service Mesh            │
                │   - Service Discovery     │
                │   - Load Balancing        │
                │   - Circuit Breaker       │
                └───────────────────────────┘
```

### When to Consider Microservices

You should consider splitting into microservices when:

#### Signal 1: Team Scaling Issues

**Scenario:** Many teams and they are stepping on each other's toes:
- Merge conflicts constantly
- Can't deploy without coordinating with 5 teams
- Changes in one area break another team's code

**Solution:** Split along team boundaries (Conway's Law)

#### Signal 2: Different Scaling Requirements

**Scenario:** One part of the app needs 10x the resources of the rest:

```
Your App:
- API: 2 servers (handles 1000 req/sec)
- Background Jobs: 1 server (processes 100 jobs/sec)
- WebSocket Server: 5 servers (handles 50K concurrent connections)
- Image Processing: 10 servers (CPU-intensive)
```

With a monolith, you'd need 10 servers of EVERYTHING just to handle image processing.

With microservices:
- API: 2 servers
- Background Jobs: 1 server
- WebSocket: 5 servers
- Image Processing: 10 servers (only these scale up)

**Cost savings: Huge**

#### Signal 3: Technology Heterogeneity

**Scenario:** Different parts need different technologies:
- Main app: Node.js (great for APIs)
- ML model serving: Python (TensorFlow)
- Real-time analytics: Go (high performance)
- Search: Elasticsearch

#### Signal 4: Independent Deployment

**Scenario:** You need to deploy parts independently:
- Payment service must have 99.99% uptime
- Experimental features can have downtime
- Different release cycles for different features

#### Signal 5: Performance Bottlenecks

**Scenario:** One slow operation blocks everything else in your monolith.

---

### The Migration Path: Monolith to Microservices

When you DO need to split, do it gradually:

#### Phase 1: Start Modular
```
Monolithic Application (Well-Structured)
├── Auth Module
├── Projects Module
├── Billing Module
└── Notifications Module
```

#### Phase 2: Extract First Service

Extract the most independent module first (usually notifications or background jobs):

```
Monolithic Application               Notification Service
├── Auth Module                      (Separate Deployment)
├── Projects Module                         ↑
├── Billing Module                          │
└── Notifications Module ─────────────────┘
                          API calls or
                          Message Queue
```

#### Phase 3: Gradual Extraction

Extract services based on need:

```
API Gateway
    ↓
┌───┴────┬──────────┬──────────┬────────────┐
│        │          │          │            │
Auth   Projects  Billing  Notifications  Background
Service Service Service   Service       Jobs Service
```

#### Phase 4: Full Microservices

Only if you truly need it:

```
                  API Gateway
                      ↓
      ┌───────────────┼───────────────┐
      ↓               ↓               ↓
   Auth          Projects         Billing
   Service       Service          Service
      │               │               │
      └───────┬───────┴───────┬───────┘
              ↓               ↓
         Message Queue    Shared DB/Cache
              ↓
    ┌─────────┴─────────┐
    ↓                   ↓
Notifications      Analytics
Service            Service
```

---

### Microservices: The Hidden Costs

Before you jump to microservices, understand what you're signing up for:

**1. Distributed System Complexity**

```javascript
// Simple in monolith
async function createOrder(customerId, items) {
  await db.transaction(async (trx) => {
    const order = await trx('orders').insert({ customerId });
    await trx('items').insert(items.map(i => ({ ...i, orderId: order.id })));
    await trx('customers').where('id', customerId).increment('order_count');
  });
}

// Complex in microservices (Saga pattern needed)
async function createOrder(customerId, items) {
  const sagaId = uuid();
  
  try {
    // Step 1: Create order
    const order = await orderService.create({ customerId }, sagaId);
    
    // Step 2: Reserve inventory
    await inventoryService.reserve(items, sagaId);
    
    // Step 3: Process payment
    await paymentService.charge(customerId, order.total, sagaId);
    
    // Step 4: Update customer stats
    await customerService.incrementOrderCount(customerId, sagaId);
    
  } catch (error) {
    // COMPENSATING TRANSACTIONS (undo everything)
    await orderService.cancel(order.id, sagaId);
    await inventoryService.release(items, sagaId);
    await paymentService.refund(customerId, order.total, sagaId);
    // What if compensation fails? Need retry logic, dead letter queues...
  }
}
```

**2. Operational Overhead**

| Aspect | Monolith | 10 Microservices |
|--------|----------|------------------|
| **Deployment** | 1 pipeline | 10 pipelines |
| **Monitoring** | 1 dashboard | 10+ dashboards |
| **Logging** | 1 log stream | 10 log streams (need correlation) |
| **Debugging** | Stack trace in 1 place | Distributed tracing needed |
| **Testing** | Integration tests | Contract tests, integration, E2E |
| **Infrastructure** | 2-3 servers | 20+ servers (each service needs redundancy) |

**3. Network Calls Replace Function Calls**

```javascript
// Monolith: Function call (microseconds)
const user = await userService.getUser(userId);

// Microservices: HTTP call (milliseconds)
// 100x-1000x slower!
const user = await userServiceClient.get(`/users/${userId}`);
// What if network fails? Need retry logic, circuit breakers, timeouts...
```

**4. Data Consistency Challenges**

In a monolith, you can join tables:

```sql
-- Easy in monolith
SELECT p.*, u.name as owner_name
FROM projects p
JOIN users u ON p.owner_id = u.id
WHERE p.tenant_id = ?
```

In microservices, you need:
- Multiple API calls
- Data denormalization
- Eventual consistency
- Event-driven updates

---

### Decision Framework: Monolith or Microservices?

Use this flowchart:

```
Start Here
    ↓
Do you have < 10 engineers?
    ↓ YES
Use Monolith ✅
    
    ↓ NO
Do all parts of your app scale similarly?
    ↓ YES
Use Monolith ✅
    
    ↓ NO
Can you handle distributed system complexity?
    ↓ NO
Use Monolith ✅
    
    ↓ YES
Do you have dedicated DevOps team?
    ↓ NO
Use Monolith ✅
    
    ↓ YES
Are you comfortable with eventual consistency?
    ↓ NO
Use Monolith ✅
    
    ↓ YES
Consider Microservices ⚠️
(but start with modular monolith anyway!)
```

---

### Our Three Projects: Architecture Decision

Based on the above analysis:

#### Java Project (EnterpriseFlow): Modular Monolith
**Rationale:**
- Complex business logic benefits from transactions
- Starting with small team
- Can split later if needed
- Spring Boot makes modular monoliths easy

#### Node.js Project (CollabSpace): Modular Monolith with Service Extraction
**Rationale:**
- Most features in monolith
- WebSocket server might be separate service (different scaling)
- Real-time features benefit from isolated deployment

#### Go Project (ApiCore): Monolith Initially, Microservices-Ready
**Rationale:**
- High-performance requirements
- Go makes microservices easy (small binaries, fast startup)
- Will demonstrate migration path to microservices
- API gateway pattern naturally suited for Go

---
