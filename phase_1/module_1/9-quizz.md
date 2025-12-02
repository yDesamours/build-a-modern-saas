
## Self-Assessment Quiz

Test your understanding before moving to Module 2:

### Questions

1. **What are the three main multi-tenancy patterns?** Describe each and when to use them.

2. **Calculate MRR:** 
   - 20 customers at $50/month
   - 5 customers at $200/month
   - 100 customers at $10/month

3. **True or False:** In a shared schema multi-tenancy pattern, you should always include tenant_id in every query. Why?

4. **Which technology would you choose:**
   - Real-time chat application: _______
   - Complex financial transaction processing: _______
   - High-performance API gateway: _______

5. **What is the difference between MRR and ARR?** When would ARR be more relevant than MRR?

6. **Name three things that should be processed asynchronously in a SaaS application and explain why.**

7. **What is "graceful degradation" and why is it important?**

8. **If your CAC is $200 and your monthly subscription is $20, what's the minimum number of months a customer must stay to break even?**

9. **What are the three pillars of observability?**

10. **Design question:** You're building a SaaS app that will have 50 customers in 6 months, but potentially 10,000 in 3 years. What multi-tenancy pattern would you choose and why? How would you handle the transition if needed?

### Answers

**Answer Key:**

1. 
   - **Shared Database, Shared Schema:** All tenants share same tables with tenant_id discriminator. Use when: starting out, cost-sensitive, B2C SaaS.
   - **Shared Database, Separate Schema:** Each tenant gets own schema in one database. Use when: need better isolation, 100-1000 tenants, B2B SaaS.
   - **Separate Database per Tenant:** Each tenant has own database. Use when: enterprise customers, compliance requirements, high-value customers.

2. **MRR = (20 × $50) + (5 × $200) + (100 × $10) = $1,000 + $1,000 + $1,000 = $3,000**

3. **True.** Forgetting tenant_id in queries is the #1 cause of data leakage between tenants. EVERY query must filter by tenant_id to ensure data isolation.

4. 
   - Real-time chat: **Node.js** (excellent WebSocket support, event-driven)
   - Financial transactions: **Java/Spring Boot** (transaction management, ACID guarantees, mature ecosystem)
   - API gateway: **Go** (high performance, low latency, concurrent request handling)

5. **MRR = Monthly Recurring Revenue, ARR = Annual Recurring Revenue (MRR × 12).** ARR is more relevant when you have annual contracts, as it shows committed revenue and is the standard metric for enterprise SaaS businesses.

6. Examples:
   - **Email sending:** Slow, can fail, shouldn't block user request
   - **Report generation:** CPU-intensive, can take minutes
   - **Image processing:** CPU/memory intensive, user doesn't need to wait
   - **Webhook delivery:** External API call, unreliable, needs retries
   - **Data export:** Large dataset, long-running operation

7. **Graceful degradation** means your application continues to function (with reduced capabilities) when components fail. Example: If recommendation service fails, show the page without recommendations rather than failing completely. Important because: reduces impact of failures, improves reliability, better user experience.

8. **10 months.** $200 CAC ÷ $20 MRR = 10 months to break even (ignoring other costs).

9. 
   - **Logs:** What happened (events, errors)
   - **Metrics:** How much/many (counters, gauges)
   - **Traces:** Where time was spent (distributed tracing)

10. **Sample Answer:** 
   - Start with **Shared Database, Shared Schema** (simplest, fastest to build, adequate for 50-1000 customers)
   - Structure code in a tenant-aware way from day 1 (middleware, repositories)
   - Monitor performance per tenant to identify "noisy neighbors"
   - At ~1000-2000 customers, evaluate if needed:
     - Move largest tenants to separate schemas or databases
     - Keep small tenants on shared schema
   - Transition plan:
     - Build migration tool to move tenant data
     - Test thoroughly with one pilot tenant
     - Move enterprise customers first (they'll appreciate dedicated resources)
     - Gradually migrate based on size/value
