## 1.4 Key SaaS Metrics (And Why Developers Should Care)

These metrics aren't just for business people - they directly impact your technical decisions.

### Financial Metrics

#### 1. **MRR (Monthly Recurring Revenue)**

**Definition:** Predictable revenue generated each month from subscriptions

```
MRR = Number of customers × Average subscription price
```

**Why developers care:**
- Higher MRR = more budget for infrastructure
- MRR growth = need to plan for scale
- MRR per customer = indicates target customer size (affects architecture)

**Technical Impact:**
- Low MRR per customer → optimize for cost efficiency (shared resources)
- High MRR per customer → can afford dedicated resources, better support

---

#### 2. **ARR (Annual Recurring Revenue)**

**Definition:** MRR × 12 (for annual contracts)

**Why developers care:**
- Annual contracts = more predictable funding for infrastructure
- ARR goals affect feature prioritization
- Enterprise ARR = need enterprise features (SSO, audit logs, SLAs)

---

#### 3. **CAC (Customer Acquisition Cost)**

**Definition:** Total sales/marketing cost ÷ Number of new customers

**Why developers care:**
- High CAC = need to retain customers longer (affects churn-reduction features)
- Low CAC (product-led growth) = need great onboarding, self-service features
- CAC affects how much you can spend on customer support infrastructure

**Technical Impact:**
If CAC is $1000 but MRR is only $50, you need customers to stay for 20+ months to break even. This means:
- Must build retention features (notifications, integrations)
- Must minimize churn (performance, reliability)
- Must have good onboarding (reduces early churn)

---

#### 4. **LTV (Lifetime Value)**

**Definition:** Average revenue per customer over their entire lifetime

```
LTV = ARPU (Average Revenue Per User) × Customer Lifetime
```

**Why developers care:**
- LTV > 3 × CAC = healthy business (can invest in growth)
- LTV affects how much you can spend on infrastructure per customer
- Low LTV = must optimize costs aggressively

**Example Decision:**
- LTV = $5000 per customer → Can afford to spend $50/month on infra per customer
- LTV = $200 per customer → Must keep infra costs under $5/month per customer

---

### Usage Metrics

#### 5. **DAU/MAU (Daily/Monthly Active Users)**

**Definition:** Unique users who use the product in a day/month

**Why developers care:**
- DAU/MAU ratio (stickiness) = how often users return
- High MAU but low DAU = engagement problem (need features to increase visits)
- Affects caching strategy (high DAU = need aggressive caching)

**Technical Impact:**
```javascript
// Example: Tracking DAU
async function recordUserActivity(userId) {
  const today = new Date().toISOString().split('T')[0]; // '2024-01-15'
  
  await redis.setex(
    `dau:${today}:${userId}`,
    86400, // 24 hours
    '1'
  );
}

// Calculate DAU
async function getDAU(date) {
  const keys = await redis.keys(`dau:${date}:*`);
  return keys.length;
}
```

---

#### 6. **Churn Rate**

**Definition:** Percentage of customers who cancel in a given period

```
Churn Rate = (Customers Lost ÷ Total Customers at Start) × 100
```

**Why developers care:**
- High churn = something is broken (performance? bugs? missing features?)
- Need to identify churn indicators early
- Technical debt increases churn (slow app, bugs)

**Technical Impact:**
Build retention features:
- Email digests (keep users engaged)
- Notifications (bring users back)
- Performance optimization (slow apps cause churn)
- Data export (ironically, easier exit reduces churn anxiety)

**Measuring Technical Indicators of Churn:**
```javascript
// Track engagement decline
class ChurnPrediction {
  async identifyAtRiskUsers() {
    // Users who haven't logged in for 14 days
    const inactiveUsers = await db.query(`
      SELECT user_id, tenant_id, last_login_at
      FROM users
      WHERE last_login_at < NOW() - INTERVAL '14 days'
        AND subscription_status = 'active'
    `);
    
    // Users with declining usage
    const decliningUsers = await db.query(`
      SELECT 
        user_id,
        COUNT(*) as actions_this_month,
        LAG(COUNT(*)) OVER (PARTITION BY user_id ORDER BY month) as actions_last_month
      FROM user_activities
      WHERE created_at >= NOW() - INTERVAL '2 months'
      GROUP BY user_id, DATE_TRUNC('month', created_at)
      HAVING COUNT(*) < LAG(COUNT(*)) OVER (...) * 0.5
    `);
    
    return { inactiveUsers, decliningUsers };
  }
}
```

---

#### 7. **NRR (Net Revenue Retention)**

**Definition:** Revenue retained from existing customers including upgrades/downgrades

```
NRR = ((Starting MRR + Expansion - Downgrades - Churn) ÷ Starting MRR) × 100
```

**Why developers care:**
- NRR > 100% = customers are upgrading (your features are valuable!)
- NRR < 100% = customers are leaving or downgrading (quality issues?)
- Affects feature prioritization (build features that drive upgrades)

**Technical Impact:**
- High NRR = invest in advanced features for existing customers
- Low NRR = focus on core reliability and basic features first

---

### Product Metrics

#### 8. **Activation Rate**

**Definition:** Percentage of signups who reach "aha moment" (first value)

**Why developers care:**
- Low activation = poor onboarding (need better UX, tutorials)
- Need to identify and optimize "aha moment" actions
- Affects architecture (fast initial setup vs feature-rich)

**Example Activation Milestones:**
- Project management tool: Create first project
- CRM: Add first contact
- Analytics tool: Install tracking code and see first data

**Technical Implementation:**
```javascript
// Track activation funnel
class ActivationTracking {
  async trackMilestone(userId, milestone) {
    await db.query(`
      INSERT INTO user_milestones (user_id, milestone, reached_at)
      VALUES ($1, $2, NOW())
    `, [userId, milestone]);
    
    // Check if user is activated
    const requiredMilestones = [
      'account_created',
      'first_project_created',
      'first_task_added',
      'invited_team_member'
    ];
    
    const completedMilestones = await this.getCompletedMilestones(userId);
    
    if (requiredMilestones.every(m => completedMilestones.includes(m))) {
      await this.markUserAsActivated(userId);
    }
  }
  
  async getActivationRate() {
    return db.query(`
      SELECT 
        COUNT(DISTINCT user_id) FILTER (WHERE activated = TRUE) * 100.0 / 
        COUNT(DISTINCT user_id) as activation_rate
      FROM users
      WHERE created_at >= NOW() - INTERVAL '30 days'
    `);
  }
}
```

---

#### 9. **Time to Value (TTV)**

**Definition:** Time from signup to first valuable outcome

**Why developers care:**
- Long TTV = users abandon before seeing value
- Need to optimize onboarding performance
- Background jobs must complete quickly during onboarding
- First-run experience is critical

**Technical Optimization:**
```javascript
// Optimize first-time user experience
class OnboardingOptimizer {
  async handleNewUser(user) {
    // Immediate: Create sample data so user sees value instantly
    await this.createSampleData(user);
    
    // Fast: Pre-populate common settings
    await this.applyDefaultSettings(user);
    
    // Background: Import/sync external data (don't block)
    await jobQueue.add('import-user-data', { userId: user.id });
    
    // Measure TTV
    await analytics.track(user.id, 'onboarding_started', {
      timestamp: Date.now()
    });
  }
  
  async createSampleData(user) {
    // For a project management tool: create a sample project
    const sampleProject = await Project.create({
      userId: user.id,
      name: "Welcome Project (Sample)",
      description: "Try out features here!",
      is_sample: true
    });
    
    // Add sample tasks so they see a populated interface
    await Task.bulkCreate([
      { projectId: sampleProject.id, title: "Click here to mark complete", status: "todo" },
      { projectId: sampleProject.id, title: "Try creating your own task", status: "todo" },
      { projectId: sampleProject.id, title: "Invite a team member", status: "todo" }
    ]);
  }
  
  async measureTTV(userId, valueEvent) {
    const user = await User.findById(userId);
    const ttv = Date.now() - user.created_at.getTime();
    
    await analytics.track(userId, 'time_to_value', {
      ttv_seconds: ttv / 1000,
      value_event: valueEvent
    });
  }
}
```

---

#### 10. **Feature Adoption Rate**

**Definition:** Percentage of users who use a specific feature

**Why developers care:**
- Low adoption of expensive features = waste of development time
- Guides feature deprecation decisions
- Informs where to invest performance optimization

**Technical Implementation:**
```javascript
class FeatureTracking {
  async trackFeatureUsage(userId, featureName) {
    await db.query(`
      INSERT INTO feature_usage (user_id, feature_name, used_at)
      VALUES ($1, $2, NOW())
      ON CONFLICT (user_id, feature_name, DATE(used_at))
      DO NOTHING -- Only count once per day
    `, [userId, featureName]);
  }
  
  async getFeatureAdoptionRate(featureName, days = 30) {
    const result = await db.query(`
      SELECT 
        COUNT(DISTINCT fu.user_id) * 100.0 / COUNT(DISTINCT u.user_id) as adoption_rate
      FROM users u
      LEFT JOIN feature_usage fu 
        ON u.id = fu.user_id 
        AND fu.feature_name = $1
        AND fu.used_at >= NOW() - INTERVAL '${days} days'
      WHERE u.created_at < NOW() - INTERVAL '${days} days'
    `, [featureName]);
    
    return result.rows[0].adoption_rate;
  }
}

// Usage in your application
app.post('/api/projects/:id/export', async (req, res) => {
  await featureTracking.trackFeatureUsage(req.user.id, 'project_export');
  
  // ... actual export logic
});
```

---
