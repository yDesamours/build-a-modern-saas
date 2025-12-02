## 1.5 Architectural Principles for SaaS

Now that we understand the business context, let's translate this into architectural principles that will guide every technical decision we make.

### Principle 1: Design for Multi-Tenancy from Day 1

**Why:**
- Retrofitting multi-tenancy is extremely painful
- Even if you start with one customer, plan for multiple

**How:**
```javascript
// ❌ BAD: Global queries
const allProjects = await db.query('SELECT * FROM projects');

// ✅ GOOD: Always include tenant context
const projects = await db.query(
  'SELECT * FROM projects WHERE tenant_id = $1',
  [req.tenant.id]
);

// ✅ BETTER: Middleware that enforces tenant isolation
app.use((req, res, next) => {
  if (req.user) {
    req.tenant = req.user.tenant;
    
    // Set tenant context for ORM
    db.setTenantContext(req.tenant.id);
  }
  next();
});
```

**Checklist:**
- [ ] Every table has tenant_id (or equivalent)
- [ ] All queries filter by tenant
- [ ] No cross-tenant data access (unless explicitly admin)
- [ ] Tenant context in logs for debugging
- [ ] Test with multiple tenants from day 1

---

### Principle 2: Build for Scale, Deploy for Current Size

**The Balance:**
- Don't over-engineer for millions of users on day 1
- But don't make choices that prevent future scaling

**Architecture Evolution:**

**Phase 1: 0-1,000 users (Months 0-6)**
```
Architecture:
- Monolithic application
- Single database (shared schema multi-tenant)
- Single server
- Basic caching (in-memory)

Focus: Speed of development, feature iteration
```

**Phase 2: 1,000-10,000 users (Months 6-18)**
```
Architecture:
- Still monolithic (but well-structured)
- Database read replicas
- Load balancer + multiple app servers
- Redis for caching and sessions
- Background job queue
- CDN for static assets

Focus: Performance optimization, reliability
```

**Phase 3: 10,000-100,000 users (Months 18-36)**
```
Architecture:
- Begin service extraction (most critical components)
- Database sharding (if needed)
- Microservices for specific domains
- Message queue (Kafka/RabbitMQ)
- Separate schema per tenant (for larger tenants)
- Advanced caching strategies
- Multiple data centers

Focus: Scalability, service isolation
```

**Phase 4: 100,000-1,000,000+ users (Year 3+)**
```
Architecture:
- Full microservices (where beneficial)
- Polyglot persistence (right database for each service)
- Event-driven architecture
- Service mesh
- Global distribution
- Auto-scaling infrastructure
- Separate databases for enterprise tenants

Focus: Global scale, cost optimization
```

**How to Build for Future Scale:**

```javascript
// ✅ GOOD: Structured so you can extract services later
// file: services/ProjectService.js
class ProjectService {
  async createProject(tenantId, data) {
    // Business logic here
    const project = await this.projectRepo.create(tenantId, data);
    
    // Emit event (can move to message queue later)
    await this.eventBus.emit('project.created', { tenantId, projectId: project.id });
    
    return project;
  }
}

// Later, this becomes a microservice with the same interface
// The event bus becomes Kafka, but the service interface stays the same
```

```javascript
// ❌ BAD: Tightly coupled, hard to extract
app.post('/projects', async (req, res) => {
  const project = await db.query('INSERT INTO projects...');
  await db.query('INSERT INTO activities...');
  await db.query('UPDATE user_stats...');
  res.json(project);
});
```

---

### Principle 3: Embrace Asynchronous Processing

**Why:**
- Improves response times
- Enables scaling of CPU-intensive operations
- Provides resilience (retries, error handling)

**What Should Be Async:**

1. **Email/Notifications**
   ```javascript
   // ❌ BAD: Send email in request handler (slow!)
   app.post('/signup', async (req, res) => {
     const user = await createUser(req.body);
     await emailService.sendWelcomeEmail(user.email); // Blocks for 2-3 seconds!
     res.json({ success: true });
   });
   
   // ✅ GOOD: Queue email sending
   app.post('/signup', async (req, res) => {
     const user = await createUser(req.body);
     await jobQueue.add('send-welcome-email', { userId: user.id });
     res.json({ success: true }); // Fast response!
   });
   ```

2. **Data Processing**
   ```javascript
   // File upload → thumbnail generation
   app.post('/upload', async (req, res) => {
     const file = await storage.save(req.file);
     
     // Don't block on thumbnail generation
     await jobQueue.add('generate-thumbnail', { 
       fileId: file.id,
       sizes: ['small', 'medium', 'large']
     });
     
     res.json({ fileId: file.id, status: 'processing' });
   });
   ```

3. **External API Calls**
   ```javascript
   // Integration webhooks
   app.post('/webhooks/stripe', async (req, res) => {
     // Acknowledge immediately
     res.json({ received: true });
     
     // Process in background
     await jobQueue.add('process-stripe-webhook', { 
       payload: req.body 
     });
   });
   ```

4. **Reports and Exports**
   ```javascript
   app.post('/reports/generate', async (req, res) => {
     const reportId = uuidv4();
     
     await jobQueue.add('generate-report', {
       reportId,
       tenantId: req.tenant.id,
       filters: req.body.filters
     });
     
     res.json({ 
       reportId, 
       status: 'processing',
       statusUrl: `/reports/${reportId}/status`
     });
   });
   ```

---

### Principle 4: Design for Failure

**Murphy's Law of SaaS:** Anything that can fail, will fail. And it will fail at the worst possible time.

**What Will Fail:**
- Database connections
- External API calls
- Network requests
- Disk space
- Memory
- Third-party services

**How to Handle:**

#### 4.1 Circuit Breaker Pattern

**Definition:** A circuit breaker is a design pattern that prevents your application from repeatedly trying to execute an operation that's likely to fail.


```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000; // 1 minute
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failures = 0;
    this.nextAttempt = Date.now();
  }
  
  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failures++;
    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
    }
  }
}

// Usage
const stripeCircuitBreaker = new CircuitBreaker({
  failureThreshold: 5,
  resetTimeout: 60000
});

async function chargeCustomer(customerId, amount) {
  try {
    return await stripeCircuitBreaker.execute(async () => {
      return await stripe.charges.create({ customer: customerId, amount });
    });
  } catch (error) {
    // Fallback: queue for retry or notify admin
    await jobQueue.add('retry-charge', { customerId, amount });
    throw new Error('Payment processing temporarily unavailable');
  }
}
```

**More Reading** [learn more on microsoft learn]https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker

#### 4.2 Retry with Exponential Backoff

**Definition:** Retry with exponential backoff is a strategy for handling failed requests by waiting for a short, then progressively longer, amount of time before retrying.

```javascript
async function retryWithBackoff(fn, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries - 1) throw error;
      
      const delay = Math.pow(2, attempt) * 1000; // 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Usage
await retryWithBackoff(async () => {
  return await externalAPI.getData();
});
```

#### 4.3 Graceful Degradation

**Definition:** Graceful degradation is a design and engineering strategy used to make systems remain usable even when parts of them fail. If something breaks, the system continues to work in a reduced but acceptable way, instead of crashing completely.

```javascript
// Feature flags for graceful degradation
class FeatureManager {
  async isFeatureEnabled(featureName) {
    try {
      // Check if feature is enabled (from config/database)
      return await this.getFeatureFlag(featureName);
    } catch (error) {
      // If feature flag service is down, fail safely
      return false; // Disable non-critical features
    }
  }
}

// In your application
app.get('/dashboard', async (req, res) => {
  const data = await getDashboardData(req.tenant.id);
  
  // Try to load recommendations, but don't fail if service is down
  let recommendations = [];
  try {
    if (await featureManager.isFeatureEnabled('recommendations')) {
      recommendations = await recommendationService.get(req.user.id);
    }
  } catch (error) {
    logger.error('Recommendation service failed', error);
    // Continue without recommendations
  }
  
  res.json({ data, recommendations });
});
```

#### 4.4 Health Checks

**Definition:** A health check is a mechanism where a service exposes an endpoint or signal saying

```javascript
// Health check endpoint
app.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    checks: {}
  };
  
  // Database check
  try {
    await db.query('SELECT 1');
    health.checks.database = 'healthy';
  } catch (error) {
    health.checks.database = 'unhealthy';
    health.status = 'unhealthy';
  }
  
  // Redis check
  try {
    await redis.ping();
    health.checks.redis = 'healthy';
  } catch (error) {
    health.checks.redis = 'unhealthy';
    health.status = 'degraded'; // Can operate without redis
  }
  
  // Job queue check
  try {
    const queueHealth = await jobQueue.checkHealth();
    health.checks.jobQueue = queueHealth;
  } catch (error) {
    health.checks.jobQueue = 'unhealthy';
  }
  
  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
});
```

---

### Principle 5: Observability is Not Optional

**The Three Pillars:**

1. **Logs** - What happened
2. **Metrics** - How much/how many
3. **Traces** - Where time was spent

#### 5.1 Structured Logging

```javascript
// ❌ BAD: Unstructured logs
console.log('User logged in');
console.log('Error: ' + error.message);

// ✅ GOOD: Structured logs
logger.info('user_logged_in', {
  userId: user.id,
  tenantId: user.tenantId,
  ipAddress: req.ip,
  userAgent: req.headers['user-agent']
});

logger.error('payment_failed', {
  tenantId: tenant.id,
  amount: amount,
  currency: 'USD',
  errorCode: error.code,
  errorMessage: error.message,
  stripeChargeId: chargeId
});
```

**Why Structured Logs Matter**
- Easy to search and filter
- Machine-readable & predictable
- Better for distributed systems**

#### 5.2 Correlation IDs

**Definition:** A correlation ID is a unique identifier added to every request which facilitate following that request across all the services, logs, and systems it touches.

```javascript
// Middleware to add correlation ID to all requests
app.use((req, res, next) => {
  req.correlationId = req.headers['x-correlation-id'] || uuidv4();
  res.setHeader('X-Correlation-ID', req.correlationId);
  
  // Add to logger context
  req.logger = logger.child({ correlationId: req.correlationId });
  
  next();
});

// Now all logs for a request are traceable
app.post('/api/projects', async (req, res) => {
  req.logger.info('create_project_started', { tenantId: req.tenant.id });
  
  try {
    const project = await createProject(req.tenant.id, req.body);
    req.logger.info('create_project_success', { projectId: project.id });
    res.json(project);
  } catch (error) {
    req.logger.error('create_project_failed', { error: error.message });
    res.status(500).json({ error: 'Failed to create project' });
  }
});
```

#### 5.3 Key Metrics to Track

```javascript
class MetricsCollector {
  // Request metrics
  recordRequest(endpoint, method, statusCode, duration) {
    metrics.increment('http.requests', {
      endpoint,
      method,
      status: statusCode
    });
    
    metrics.histogram('http.request.duration', duration, {
      endpoint,
      method
    });
  }
  
  // Business metrics
  recordSignup(tenantId, plan) {
    metrics.increment('signups', { plan });
  }
  
  recordSubscriptionChange(tenantId, fromPlan, toPlan) {
    metrics.increment('subscription.changed', {
      from: fromPlan,
      to: toPlan,
      direction: this.getDirection(fromPlan, toPlan)
    });
  }
  
  // Database metrics
  recordQueryDuration(query, duration) {
    metrics.histogram('db.query.duration', duration, {
      query_type: this.classifyQuery(query)
    });
  }
  
  // Queue metrics
  recordJobProcessed(jobType, duration, status) {
    metrics.increment('jobs.processed', { job_type: jobType, status });
    metrics.histogram('jobs.duration', duration, { job_type: jobType });
  }
}
```

---

### Principle 6: Security by Default

**Assume Breach:** Design as if attackers are already inside

#### 6.1 Tenant Isolation

```javascript
// ✅ GOOD: Use ORM/query builder that enforces tenant isolation
class TenantAwareRepository {
  constructor(tenantId) {
    this.tenantId = tenantId;
  }
  
  async findAll() {
    return db.query(
      'SELECT * FROM projects WHERE tenant_id = $1',
      [this.tenantId]
    );
  }
  
  async findById(id) {
    return db.queryOne(
      'SELECT * FROM projects WHERE id = $1 AND tenant_id = $2',
      [id, this.tenantId]
    );
  }
}

// Middleware that creates tenant-scoped repositories
app.use((req, res, next) => {
  if (req.tenant) {
    req.repos = {
      projects: new TenantAwareRepository(req.tenant.id),
      tasks: new TaskRepository(req.tenant.id),
      users: new UserRepository(req.tenant.id)
    };
  }
  next();
});
```

#### 6.2 Input Validation

```javascript
// Use validation library (e.g., Joi, Yup, Zod)
const projectSchema = Joi.object({
  name: Joi.string().min(1).max(200).required(),
  description: Joi.string().max(5000).optional(),
  startDate: Joi.date().optional(),
  budget: Joi.number().positive().optional(),
  tags: Joi.array().items(Joi.string()).max(10).optional()
});

app.post('/api/projects', async (req, res) => {
  // Validate input
  const { error, value } = projectSchema.validate(req.body);
  if (error) {
    return res.status(400).json({ error: error.details[0].message });
  }
  
  // Use validated data
  const project = await createProject(req.tenant.id, value);
  res.json(project);
});
```

#### 6.3 Rate Limiting

```javascript
// Prevent abuse
const rateLimit = require('express-rate-limit');

const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per window
  message: 'Too many requests from this IP',
  standardHeaders: true,
  legacyHeaders: false,
  
  // Different limits per plan
  keyGenerator: (req) => req.tenant?.id || req.ip,
  max: (req) => {
    const limits = {
      free: 100,
      pro: 1000,
      enterprise: 10000
    };
    return limits[req.tenant?.plan] || 100;
  }
});

app.use('/api/', apiLimiter);
```

---

### Principle 7: Cost Optimization from Day 1

**Cost-Aware Architecture Decisions:**

1. **Database:**
   - Start with single DB (cheapest)
   - Use connection pooling (fewer connections = lower cost)
   - Archive old data (storage costs add up)
   
2. **Caching:**
   - Aggressive caching reduces database load
   - CDN for static assets (cheaper than serving from your servers)
   - Cache expensive computations
   
3. **Storage:**
   - Use object storage (S3) instead of server disk
   - Implement lifecycle policies (move old files to cheaper storage)
   - Compress files before storing
   
4. **Compute:**
   - Auto-scaling (don't pay for unused capacity)
   - Background jobs on cheaper instances
   - Use serverless for spiky workloads

**Example: Cost-Aware File Storage**

```javascript
class StorageService {
  async uploadFile(file, options = {}) {
    // Compress images before uploading
    if (file.mimetype.startsWith('image/')) {
      file.buffer = await this.compressImage(file.buffer, {
        quality: 85,
        maxWidth: 2000
      });
    }
    
    // Determine storage class based on expected access
    const storageClass = options.frequentAccess 
      ? 'STANDARD'  // More expensive, fast retrieval
      : 'STANDARD_IA';  // Cheaper, slower retrieval
    
    // Upload to S3 with lifecycle policy
    await s3.upload({
      Bucket: process.env.S3_BUCKET,
      Key: file.key,
      Body: file.buffer,
      StorageClass: storageClass,
      
      // Metadata for lifecycle management
      Tagging: `access-frequency=${options.frequentAccess ? 'high' : 'low'}`
    });
    
    // Set lifecycle policy to move to cheaper storage after 90 days
    if (!options.frequentAccess) {
      await this.setLifecyclePolicy(file.key, {
        transitionToGlacierAfterDays: 90
      });
    }
  }
}
```

---