## 1.3 SaaS Business Models and Technical Implications

### Common SaaS Pricing Models

#### 1. **Flat-Rate Pricing**

**Model:** Everyone pays the same price for full access

**Example:** Basecamp ($99/month for unlimited users and projects)

**Technical Implications:**
- ✅ Simpler implementation (no usage tracking needed)
- ✅ No complex billing logic
- ✅ Predictable resource usage per customer
- ⚠️ Must prevent abuse (rate limiting still needed)
- ⚠️ Power users cost you more but pay the same

**When to use:**
- Simple products with predictable costs
- When you want to simplify decision-making for customers
- When usage tracking is difficult or expensive

**Technical Requirements:**
- Basic subscription management
- Payment processing (Stripe, PayPal)
- Access control (active subscription = full access)

---

#### 2. **Tiered Pricing**

**Model:** Different feature sets at different price points

**Example:** 
- Starter: $10/month (basic features)
- Professional: $50/month (advanced features)
- Enterprise: $200/month (all features + support)

**Technical Implications:**
- 📊 Feature flags needed for tier-based access
- 📊 Database must store plan information per tenant
- 📊 Upgrade/downgrade flows required
- 📊 May need usage limits per tier
- 📊 Proration calculations for mid-cycle changes

**When to use:**
- When you have distinct customer segments
- When features have clear value tiers
- Most common SaaS model

**Technical Requirements:**

Features flags (Feature gating) are crucial. They help control who can use which features.

```javascript
// Example: Feature access check
class FeatureAccess {
  canAccessAdvancedReports(tenant) {
    return tenant.plan in ['professional', 'enterprise'];
  }
  
  canAccessAPI(tenant) {
    return tenant.plan === 'enterprise';
  }
  
  getProjectLimit(tenant) {
    return {
      'starter': 5,
      'professional': 50,
      'enterprise': Infinity
    }[tenant.plan];
  }
}
```

**Database Schema:**
```sql
-- tenants table
CREATE TABLE tenants (
  id UUID PRIMARY KEY,
  name VARCHAR(255),
  plan VARCHAR(50), -- 'starter', 'professional', 'enterprise'
  plan_started_at TIMESTAMP,
  subscription_status VARCHAR(50) -- 'active', 'past_due', 'canceled'
);

-- features table (optional, for dynamic features)
CREATE TABLE plan_features (
  plan VARCHAR(50),
  feature_key VARCHAR(100),
  enabled BOOLEAN,
  limit_value INTEGER,
  PRIMARY KEY (plan, feature_key)
);
```

---

#### 3. **Per-User Pricing (Seat-Based)**

**Model:** Price scales with number of users

**Example:** Slack ($8/user/month), Jira ($7.75/user/month)

**Technical Implications:**
- 👥 Must track active users accurately
- 👥 Need user invitation/removal flows
- 👥 Billing updates when users added/removed
- 👥 Proration calculations
- 👥 Consider "what is a user?" definition (active monthly users vs all users)

**When to use:**
- Team collaboration tools
- When value scales with team size
- When costs scale with users (support, infrastructure)

**Technical Requirements:**

```sql
-- Track users per tenant
CREATE TABLE tenant_users (
  id UUID PRIMARY KEY,
  tenant_id UUID REFERENCES tenants(id),
  user_id UUID REFERENCES users(id),
  role VARCHAR(50),
  added_at TIMESTAMP,
  removed_at TIMESTAMP, -- NULL if active
  billable BOOLEAN -- some users might not be billable (guests)
);

-- Function to calculate current billable users
CREATE FUNCTION get_billable_user_count(tenant_uuid UUID) 
RETURNS INTEGER AS $$
  SELECT COUNT(*) 
  FROM tenant_users 
  WHERE tenant_id = tenant_uuid 
    AND removed_at IS NULL 
    AND billable = TRUE;
$$ LANGUAGE SQL;
```

**Billing Logic:**
```javascript
// Calculate monthly cost
function calculateMonthlyCost(tenant) {
  const billableUsers = tenant.getBillableUserCount();
  const pricePerUser = tenant.plan.pricePerUser;
  return billableUsers * pricePerUser;
}

// Handle mid-cycle user additions
function handleUserAdded(tenant, newUser) {
  const daysRemainingInCycle = tenant.getDaysUntilNextBilling();
  const daysInCycle = 30;
  const proratedAmount = (tenant.plan.pricePerUser / daysInCycle) * daysRemainingInCycle;
  
  // Charge prorated amount or add to next invoice
  billingSystem.charge(tenant, proratedAmount);
}
```

---

#### 4. **Usage-Based Pricing (Pay-As-You-Go)**

**Model:** Price based on actual usage metrics

**Examples:** 
- AWS (per GB storage, per API call)
- SendGrid (per email sent)
- Twilio (per SMS/call)
- Stripe (per transaction)

**Technical Implications:**
- 📈 Must track ALL usage accurately
- 📈 Need usage aggregation systems
- 📈 Real-time or near-real-time metering
- 📈 Complex billing calculations
- 📈 Usage dashboards for transparency
- 📈 Prevent bill shock (spending limits)

**When to use:**
- When usage varies dramatically between customers
- When your costs directly correlate with usage
- API services, infrastructure services

**Technical Requirements:**

This is the most technically complex pricing model.

```sql
-- Usage tracking table
CREATE TABLE usage_events (
  id UUID PRIMARY KEY,
  tenant_id UUID REFERENCES tenants(id),
  event_type VARCHAR(100), -- 'api_call', 'email_sent', 'storage_gb_hour'
  quantity DECIMAL(10,2),
  metadata JSONB, -- additional context
  created_at TIMESTAMP,
  processed BOOLEAN DEFAULT FALSE,
  billing_cycle VARCHAR(50) -- '2024-01' for January 2024
);

-- Create index for fast aggregation
CREATE INDEX idx_usage_tenant_cycle 
ON usage_events(tenant_id, billing_cycle, event_type);

-- Aggregated usage for billing
CREATE TABLE usage_aggregates (
  id UUID PRIMARY KEY,
  tenant_id UUID REFERENCES tenants(id),
  billing_cycle VARCHAR(50),
  event_type VARCHAR(100),
  total_quantity DECIMAL(15,2),
  unit_price DECIMAL(10,4),
  total_amount DECIMAL(15,2),
  last_updated TIMESTAMP
);
```

**Usage Tracking Implementation:**

```javascript
// Usage tracking service
class UsageTracker {
  // Track an event
  async trackUsage(tenantId, eventType, quantity, metadata = {}) {
    const billingCycle = this.getCurrentBillingCycle(); // e.g., '2024-01'
    
    await db.query(`
      INSERT INTO usage_events 
        (id, tenant_id, event_type, quantity, metadata, billing_cycle, created_at)
      VALUES 
        ($1, $2, $3, $4, $5, $6, NOW())
    `, [
      uuidv4(),
      tenantId,
      eventType,
      quantity,
      metadata,
      billingCycle
    ]);
    
    // Update aggregate in background
    await this.updateAggregate(tenantId, eventType, billingCycle);
  }
  
  // Aggregate usage for billing
  async updateAggregate(tenantId, eventType, billingCycle) {
    await db.query(`
      INSERT INTO usage_aggregates 
        (id, tenant_id, billing_cycle, event_type, total_quantity, last_updated)
      SELECT 
        $1,
        tenant_id,
        billing_cycle,
        event_type,
        SUM(quantity),
        NOW()
      FROM usage_events
      WHERE tenant_id = $2 
        AND event_type = $3 
        AND billing_cycle = $4
        AND processed = FALSE
      GROUP BY tenant_id, billing_cycle, event_type
      ON CONFLICT (tenant_id, billing_cycle, event_type)
      DO UPDATE SET
        total_quantity = usage_aggregates.total_quantity + EXCLUDED.total_quantity,
        last_updated = NOW()
    `, [uuidv4(), tenantId, eventType, billingCycle]);
    
    // Mark events as processed
    await db.query(`
      UPDATE usage_events 
      SET processed = TRUE
      WHERE tenant_id = $1 
        AND event_type = $2 
        AND billing_cycle = $3
        AND processed = FALSE
    `, [tenantId, eventType, billingCycle]);
  }
  
  // Calculate monthly bill
  async calculateMonthlyBill(tenantId, billingCycle) {
    const usage = await db.query(`
      SELECT 
        event_type,
        total_quantity,
        unit_price,
        (total_quantity * unit_price) as amount
      FROM usage_aggregates ua
      JOIN pricing p ON ua.event_type = p.event_type
      WHERE tenant_id = $1 AND billing_cycle = $2
    `, [tenantId, billingCycle]);
    
    const totalAmount = usage.rows.reduce((sum, item) => sum + item.amount, 0);
    
    return {
      tenantId,
      billingCycle,
      lineItems: usage.rows,
      total: totalAmount
    };
  }
}
```

**Example: Tracking API calls**

```javascript
// In your API middleware
app.use(async (req, res, next) => {
  const tenantId = req.tenant.id;
  
  // Track the API call
  await usageTracker.trackUsage(tenantId, 'api_call', 1, {
    endpoint: req.path,
    method: req.method
  });
  
  next();
});
```

**Preventing Bill Shock:**

```javascript
class SpendingLimitService {
  async checkSpendingLimit(tenantId) {
    const tenant = await getTenant(tenantId);
    const currentSpend = await this.getCurrentMonthSpend(tenantId);
    
    if (tenant.spendingLimit && currentSpend >= tenant.spendingLimit) {
      throw new Error('Spending limit reached');
    }
    
    // Warn at 80%
    if (tenant.spendingLimit && currentSpend >= tenant.spendingLimit * 0.8) {
      await notificationService.send(tenant, 'SPENDING_LIMIT_WARNING', {
        currentSpend,
        limit: tenant.spendingLimit,
        percentage: (currentSpend / tenant.spendingLimit) * 100
      });
    }
  }
}
```

---

#### 5. **Freemium Model**

**Model:** Free tier with limited features/usage, paid tiers for more

**Examples:** 
- Dropbox (2GB free, pay for more)
- Mailchimp (free for up to 500 subscribers)
- GitHub (free public repos, paid for private)

**Technical Implications:**
- 🆓 Must enforce usage limits strictly
- 🆓 Need upgrade prompts and conversion funnels
- 🆓 May need to handle high volume of free users
- 🆓 Must prevent abuse on free tier
- 🆓 Balance between generous free tier and paid conversion

**When to use:**
- Viral growth potential
- When free users add value (network effects)
- Low marginal cost per free user
- Need large user base

**Technical Requirements:**

```javascript
class UsageLimitService {
  async checkLimit(tenant, resourceType) {
    const limit = this.getLimit(tenant.plan, resourceType);
    const current = await this.getCurrentUsage(tenant, resourceType);
    
    if (current >= limit) {
      throw new LimitExceededError(
        `${resourceType} limit reached. Upgrade to continue.`,
        { current, limit, upgradeUrl: '/upgrade' }
      );
    }
    
    // Warn when approaching limit
    if (current >= limit * 0.9) {
      return {
        allowed: true,
        warning: `You're at ${current}/${limit} ${resourceType}. Consider upgrading.`
      };
    }
    
    return { allowed: true };
  }
  
  getLimit(plan, resourceType) {
    const limits = {
      free: {
        projects: 3,
        storage_mb: 100,
        api_calls_per_day: 1000
      },
      pro: {
        projects: 50,
        storage_mb: 10000,
        api_calls_per_day: 100000
      },
      enterprise: {
        projects: Infinity,
        storage_mb: Infinity,
        api_calls_per_day: Infinity
      }
    };
    
    return limits[plan][resourceType];
  }
}
```

---

#### 6. **Hybrid Models**

Most successful SaaS products combine multiple models:

**Example 1: Tiered + Per-User + Usage**
- Base tier price: $50/month
- Additional users: $10/user/month
- Extra API calls: $0.01 per 1000 calls over limit

**Example 2: Freemium + Tiered + Usage**
- Free: Up to 100 API calls/day
- Starter ($10/month): Up to 10,000 API calls/day
- Pro ($50/month): Up to 100,000 API calls/day
- Overage: $0.01 per 1000 calls

**Technical Challenge:**
Combining models increases complexity significantly. You need:
- Multiple tracking systems
- Complex billing calculations
- Clear usage visibility for customers
- Careful testing of edge cases

---
