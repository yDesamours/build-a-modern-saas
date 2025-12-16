# Module 3.5: Version Control & Branching Strategies

**Duration:** 3 days  
**Part of:** Phase 1 - Foundation & Architecture

---

## 📚 Learning Objectives

By the end of this module, you will:

- Master advanced Git workflows for team collaboration
- Choose the right branching strategy for your team size and deployment frequency
- Implement effective code review processes
- Use Git hooks for automation and quality control
- Understand monorepo vs polyrepo trade-offs
- Handle merge conflicts effectively
- Create meaningful commit messages and PR descriptions

---

## 🎯 Why Version Control Strategy Matters

**The Problem Without Strategy:**

```
main branch
├── feature-1 (6 weeks old, conflicts everywhere)
├── feature-2 (merged without review)
├── hotfix (merged to wrong branch)
├── experimental-stuff (no one knows what this is)
└── john-local-changes (should never have been pushed)
```

**With Proper Strategy:**

```
main (always deployable)
├── develop (integration branch)
│   ├── feature/user-authentication (PR #123, reviewed)
│   ├── feature/payment-integration (PR #124, in review)
│   └── bugfix/email-validation (PR #125, approved)
└── hotfix/critical-security-patch (PR #126, merged to main & develop)
```

**Real-World Impact:**

- **Without strategy:** 2-3 days to prepare releases, frequent production bugs, unclear code history
- **With strategy:** Deploy multiple times per day, clear audit trail, fewer bugs escape to production

---

## 📖 Day 1: Git Fundamentals & Branching Strategies

### 3.5.1 Essential Git Commands Review

#### Basic Operations

```bash
# Repository initialization
git init
git clone https://github.com/company/saas-backend.git

# Check status and history
git status
git log --oneline --graph --all --decorate
git log --author="John" --since="2 weeks ago"

# Viewing changes
git diff                          # Working directory vs staging
git diff --staged                 # Staging vs last commit
git diff main..feature/auth       # Between branches
git show HEAD                     # Show last commit
git show abc123                   # Show specific commit

# Stashing changes
git stash                         # Save work temporarily
git stash list                    # List stashed changes
git stash pop                     # Apply and remove stash
git stash apply stash@{0}         # Apply specific stash
git stash drop stash@{0}          # Remove specific stash
```

#### Branch Management

```bash
# Creating branches
git branch feature/user-auth                    # Create branch
git checkout -b feature/payment-gateway         # Create and switch
git switch -c feature/notifications             # Modern syntax

# Switching branches
git checkout main
git switch develop                              # Modern syntax

# Listing branches
git branch                        # Local branches
git branch -r                     # Remote branches
git branch -a                     # All branches
git branch -vv                    # Verbose with tracking info

# Deleting branches
git branch -d feature/completed   # Safe delete (merged only)
git branch -D feature/abandoned   # Force delete
git push origin --delete feature/old-feature  # Delete remote branch

# Renaming branches
git branch -m old-name new-name
git branch -m new-name            # Rename current branch
```

#### Merging and Rebasing

```bash
# Merging
git checkout main
git merge feature/user-auth              # Regular merge
git merge --no-ff feature/payment        # Force merge commit
git merge --squash feature/small-fix     # Squash all commits

# Rebasing
git checkout feature/user-auth
git rebase main                          # Rebase onto main
git rebase -i HEAD~3                     # Interactive rebase (last 3 commits)
git rebase -i main                       # Interactive rebase onto main

# During rebase
git rebase --continue                    # After resolving conflicts
git rebase --skip                        # Skip current commit
git rebase --abort                       # Cancel rebase

# Cherry-picking
git cherry-pick abc123                   # Apply specific commit
git cherry-pick abc123 def456            # Apply multiple commits
```

#### Undoing Changes

```bash
# Discard working directory changes
git checkout -- file.txt             # Discard changes to file
git restore file.txt                 # Modern syntax
git restore .                        # Discard all changes

# Unstage files
git reset HEAD file.txt              # Unstage file
git restore --staged file.txt        # Modern syntax

# Undo commits
git reset --soft HEAD~1              # Undo commit, keep changes staged
git reset --mixed HEAD~1             # Undo commit, unstage changes (default)
git reset --hard HEAD~1              # Undo commit, discard changes

# Revert commits (safe for shared branches)
git revert HEAD                      # Revert last commit
git revert abc123                    # Revert specific commit
git revert HEAD~3..HEAD              # Revert last 3 commits

# Recover lost commits
git reflog                           # View all HEAD movements
git checkout abc123                  # Restore lost commit
git branch recovered-branch abc123   # Create branch from lost commit
```

#### Remote Operations

```bash
# Remote management
git remote -v                        # List remotes
git remote add upstream https://github.com/original/repo.git
git remote remove origin
git remote rename origin upstream

# Fetching and pulling
git fetch origin                     # Fetch all branches
git fetch origin main                # Fetch specific branch
git pull origin main                 # Fetch and merge
git pull --rebase origin main        # Fetch and rebase

# Pushing
git push origin feature/auth         # Push branch
git push -u origin feature/auth      # Push and set upstream
git push --force-with-lease          # Safe force push
git push --force                     # Dangerous force push (avoid!)
git push origin --tags               # Push tags
```

---

### 3.5.2 Git Flow Strategy

**Best For:**

- Large teams (10+ developers)
- Scheduled releases (monthly, quarterly)
- Multiple versions in production
- Enterprise SaaS with strict release cycles

**Branch Structure:**

```
main (production)
  ↓
develop (integration)
  ↓
feature/* (new features)
release/* (release preparation)
hotfix/* (production fixes)
```

#### Branch Types and Rules

| Branch Type | Created From | Merged Into        | Naming                  | Purpose            |
| ----------- | ------------ | ------------------ | ----------------------- | ------------------ |
| `main`      | -            | -                  | `main`                  | Production code    |
| `develop`   | `main`       | -                  | `develop`               | Integration branch |
| `feature/*` | `develop`    | `develop`          | `feature/user-auth`     | New features       |
| `release/*` | `develop`    | `main` + `develop` | `release/1.2.0`         | Release prep       |
| `hotfix/*`  | `main`       | `main` + `develop` | `hotfix/security-patch` | Production fixes   |

#### Complete Git Flow Workflow

**Starting a New Feature:**

```bash
# 1. Update develop branch
git checkout develop
git pull origin develop

# 2. Create feature branch
git checkout -b feature/subscription-billing

# 3. Work on feature
# ... make changes ...
git add .
git commit -m "feat: add subscription billing logic"

# 4. Keep feature branch updated
git fetch origin
git rebase origin/develop

# 5. Push feature branch
git push -u origin feature/subscription-billing

# 6. Create pull request on GitHub/GitLab
# Review and approval process

# 7. Merge to develop (after approval)
git checkout develop
git pull origin develop
git merge --no-ff feature/subscription-billing
git push origin develop

# 8. Delete feature branch
git branch -d feature/subscription-billing
git push origin --delete feature/subscription-billing
```

**Creating a Release:**

```bash
# 1. Branch from develop
git checkout develop
git pull origin develop
git checkout -b release/1.2.0

# 2. Update version numbers
# package.json, pom.xml, etc.
git commit -am "chore: bump version to 1.2.0"

# 3. Bug fixes only (no new features)
git commit -am "fix: resolve issue in payment flow"

# 4. Merge to main
git checkout main
git pull origin main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin main --tags

# 5. Merge back to develop
git checkout develop
git merge --no-ff release/1.2.0
git push origin develop

# 6. Delete release branch
git branch -d release/1.2.0
git push origin --delete release/1.2.0
```

**Hotfix Process:**

```bash
# 1. Branch from main
git checkout main
git pull origin main
git checkout -b hotfix/critical-security-fix

# 2. Fix the issue
git commit -am "fix: patch security vulnerability CVE-2024-1234"

# 3. Bump patch version
# 1.2.0 → 1.2.1
git commit -am "chore: bump version to 1.2.1"

# 4. Merge to main
git checkout main
git merge --no-ff hotfix/critical-security-fix
git tag -a v1.2.1 -m "Hotfix version 1.2.1"
git push origin main --tags

# 5. Merge to develop
git checkout develop
git merge --no-ff hotfix/critical-security-fix
git push origin develop

# 6. Delete hotfix branch
git branch -d hotfix/critical-security-fix
```

**Git Flow Helper Tools:**

```bash
# Install git-flow extension
brew install git-flow-avh  # macOS
apt-get install git-flow   # Ubuntu

# Initialize git-flow
git flow init

# Feature workflow
git flow feature start user-authentication
# ... work on feature ...
git flow feature finish user-authentication

# Release workflow
git flow release start 1.2.0
# ... prepare release ...
git flow release finish 1.2.0

# Hotfix workflow
git flow hotfix start security-patch
# ... fix issue ...
git flow hotfix finish security-patch
```

**Pros:**

- ✅ Clear separation between development and production
- ✅ Multiple versions supported simultaneously
- ✅ Structured release process
- ✅ Easy to track what's in production

**Cons:**

- ❌ Complex for small teams
- ❌ Slower deployment cycle
- ❌ Merge conflicts in long-lived branches
- ❌ Overhead for continuous deployment

---

### 3.5.3 GitHub Flow Strategy

**Best For:**

- Small to medium teams (2-10 developers)
- Continuous deployment
- Fast iteration cycles
- Modern SaaS with frequent releases

**Branch Structure:**

```
main (always deployable)
  ↓
feature/user-auth ──→ PR → main → Deploy
feature/payments  ──→ PR → main → Deploy
bugfix/validation ──→ PR → main → Deploy
```

#### GitHub Flow Workflow

**Complete Feature Development:**

```bash
# 1. Always start from main
git checkout main
git pull origin main

# 2. Create descriptive branch
git checkout -b feature/add-oauth-login

# 3. Make changes frequently
git add src/auth/oauth.ts
git commit -m "feat: add Google OAuth provider"

git add src/auth/github.ts
git commit -m "feat: add GitHub OAuth provider"

git add tests/auth/oauth.test.ts
git commit -m "test: add OAuth integration tests"

# 4. Push early and often
git push -u origin feature/add-oauth-login

# 5. Open pull request on GitHub
# - Add description
# - Request reviewers
# - Link to issue

# 6. Make changes based on review
git add .
git commit -m "refactor: extract common OAuth logic"
git push

# 7. Deploy to staging for testing
# (Automated via CI/CD)

# 8. After approval, merge via GitHub UI
# - Squash and merge (recommended)
# - Merge commit
# - Rebase and merge

# 9. Pull latest main and delete branch
git checkout main
git pull origin main
git branch -d feature/add-oauth-login

# 10. Automatic deployment to production
# (Triggered by merge to main)
```

**Branch Naming Conventions:**

```bash
# Features
feature/add-user-authentication
feature/implement-payment-gateway
feature/csv-export

# Bug fixes
bugfix/fix-email-validation
bugfix/resolve-memory-leak
fix/correct-timezone-calculation

# Improvements
improve/optimize-database-queries
enhance/better-error-messages
refactor/simplify-auth-logic

# Documentation
docs/add-api-documentation
docs/update-readme

# Chores
chore/update-dependencies
chore/remove-deprecated-code
```

**Pull Request Best Practices:**

```markdown
## Description

Add OAuth 2.0 authentication support for Google and GitHub providers.

## Type of Change

- [x] New feature
- [ ] Bug fix
- [ ] Breaking change
- [ ] Documentation update

## Changes Made

- Implemented OAuth 2.0 flow for Google
- Implemented OAuth 2.0 flow for GitHub
- Added callback endpoints for OAuth providers
- Integrated with existing user system
- Added comprehensive tests

## Testing

- [x] Unit tests added/updated
- [x] Integration tests added/updated
- [x] Manual testing completed
- [x] Tested on staging environment

## Screenshots (if applicable)

[Add screenshots of UI changes]

## Checklist

- [x] Code follows project style guidelines
- [x] Self-review completed
- [x] Comments added for complex logic
- [x] Documentation updated
- [x] No console logs or debug code
- [x] Tests pass locally

## Related Issues

Closes #123
Related to #456

## Deployment Notes

- Requires new environment variables: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
- Database migration will run automatically
- No breaking changes to existing API

## Reviewers

@tech-lead @security-team
```

**Pros:**

- ✅ Simple and easy to understand
- ✅ Fast deployment cycle
- ✅ Main branch always deployable
- ✅ Great for continuous deployment
- ✅ Less overhead than Git Flow

**Cons:**

- ❌ Requires strong CI/CD pipeline
- ❌ Needs good automated testing
- ❌ Can be risky without feature flags
- ❌ Harder to maintain multiple versions

---

### 3.5.4 Trunk-Based Development

**Best For:**

- High-performing teams
- Very frequent deployments (multiple per day)
- Strong automated testing
- Feature flag infrastructure

**Branch Structure:**

```
main (trunk)
  ↓
short-lived-branch-1 (< 1 day) → main
short-lived-branch-2 (< 1 day) → main
```

#### Trunk-Based Development Rules

**Core Principles:**

1. Everyone commits to `main` (trunk) regularly
2. Branches live less than 24 hours
3. Use feature flags for incomplete features
4. Comprehensive automated testing required
5. Fast build and test feedback (<10 minutes)

**Small Change Workflow:**

```bash
# 1. Pull latest main
git checkout main
git pull origin main

# 2. Create short-lived branch
git checkout -b quick-fix/improve-validation

# 3. Make small, focused change
git add src/validators/email.ts
git commit -m "refactor: improve email validation regex"

# 4. Push immediately
git push -u origin quick-fix/improve-validation

# 5. Create PR (or commit directly if authorized)
# - Automated tests run
# - Quick review (< 1 hour)
# - Merge immediately

# 6. Delete branch
git checkout main
git pull origin main
git branch -d quick-fix/improve-validation

# Time from branch creation to merge: 2-4 hours
```

**Large Feature with Feature Flags:**

```bash
# 1. Add feature flag to code
git checkout main
git pull origin main

# 2. Commit code with feature disabled by default
# Code Example (Node.js):
const features = {
  newDashboard: process.env.ENABLE_NEW_DASHBOARD === 'true'
};

if (features.newDashboard) {
  return renderNewDashboard();
} else {
  return renderOldDashboard();
}

# 3. Commit to main multiple times per day
git add src/dashboard/new-dashboard.tsx
git commit -m "feat: add new dashboard layout (behind feature flag)"
git push origin main

# Next day
git add src/dashboard/charts.tsx
git commit -m "feat: add charts to new dashboard (behind feature flag)"
git push origin main

# 4. Enable for testing in staging
# ENABLE_NEW_DASHBOARD=true

# 5. Gradual rollout in production
# - Enable for internal users (1%)
# - Enable for beta users (10%)
# - Enable for all users (100%)

# 6. Remove feature flag after stable
git add src/dashboard/
git commit -m "chore: remove old dashboard, cleanup feature flag"
git push origin main
```

**Branch By Abstraction Pattern:**

```typescript
// Step 1: Create abstraction (commit to main)
interface PaymentGateway {
  processPayment(amount: number): Promise;
}

class OldPaymentGateway implements PaymentGateway {
  async processPayment(amount: number) {
    // Old implementation
  }
}

// Step 2: Add new implementation (commit to main)
class NewPaymentGateway implements PaymentGateway {
  async processPayment(amount: number) {
    // New implementation
  }
}

// Step 3: Use feature flag to switch (commit to main)
const paymentGateway: PaymentGateway = features.newPaymentGateway
  ? new NewPaymentGateway()
  : new OldPaymentGateway();

// Step 4: Test new implementation in production

// Step 5: Remove old implementation (commit to main)
// Delete OldPaymentGateway class
// Remove feature flag
```

**Commit Frequency:**

```bash
# Minimum: 1 commit per day to main
# Target: 3-5 commits per day to main
# High performers: 10+ commits per day to main

# Small commits example:
git commit -m "feat: add user model"                    # 10 AM
git commit -m "feat: add user repository"               # 11 AM
git commit -m "feat: add user service"                  # 2 PM
git commit -m "test: add user service tests"            # 3 PM
git commit -m "feat: add user REST endpoints"           # 4 PM
```

**Required Infrastructure:**

- **Fast CI/CD pipeline** (< 10 minutes)
- **Comprehensive automated tests** (80%+ coverage)
- **Feature flag system**
- **Automated deployment**
- **Good monitoring and rollback**
- **Team discipline and trust**

**Pros:**

- ✅ Fastest deployment cycle
- ✅ Simplest branching model
- ✅ Forces small, incremental changes
- ✅ Reduces merge conflicts
- ✅ Enables continuous delivery

**Cons:**

- ❌ Requires excellent testing
- ❌ Needs feature flag infrastructure
- ❌ Requires team maturity
- ❌ Can be scary initially
- ❌ Broken commits affect everyone

---

### 3.5.5 Choosing the Right Strategy

**Decision Matrix:**

| Factor               | Git Flow       | GitHub Flow        | Trunk-Based         |
| -------------------- | -------------- | ------------------ | ------------------- |
| **Team Size**        | 10+            | 2-10               | Any (with maturity) |
| **Deploy Frequency** | Weekly/Monthly | Daily/Weekly       | Multiple per day    |
| **Release Cycle**    | Scheduled      | Continuous         | Continuous          |
| **Testing Maturity** | Medium         | High               | Very High           |
| **Complexity**       | High           | Low                | Medium              |
| **Learning Curve**   | Steep          | Easy               | Medium              |
| **Best For**         | Enterprise     | Startups/Scale-ups | High performers     |

**Recommended Strategy by SaaS Type:**

```
Enterprise SaaS (compliance-heavy):
→ Git Flow
   - Scheduled releases
   - Audit requirements
   - Multiple environments

B2B SaaS (rapid iteration):
→ GitHub Flow
   - Weekly deployments
   - Feature branches
   - PR reviews

B2C SaaS (high-frequency changes):
→ Trunk-Based Development
   - Multiple daily deployments
   - Feature flags
   - Fast feedback

Early-stage Startup:
→ GitHub Flow
   - Simple to learn
   - Balance of safety and speed
   - Easy to evolve later
```

**Evolution Path:**

```
Startup Phase (1-5 developers)
└── GitHub Flow
    ↓ (Team grows, need more structure)

Growth Phase (5-15 developers)
├── GitHub Flow (if CD mature) → Trunk-Based
└── GitHub Flow → Git Flow (if scheduled releases)
    ↓ (Process mature, CI/CD excellent)

Scale Phase (15+ developers)
├── Trunk-Based (if high maturity)
└── Git Flow (if enterprise/regulated)
```

---

## 📖 Day 2: Code Review & Collaboration

### 3.5.6 Effective Code Reviews

**Why Code Reviews Matter:**

- Catch bugs before production (30-60% of bugs caught)
- Share knowledge across team
- Maintain code quality standards
- Onboard new team members
- Ensure consistent style

**Code Review Checklist:**

```markdown
## Functionality

- [ ] Code does what it's supposed to do
- [ ] Edge cases are handled
- [ ] Error handling is appropriate
- [ ] No obvious bugs

## Testing

- [ ] Tests are included
- [ ] Tests cover main scenarios
- [ ] Tests are readable and maintainable
- [ ] Tests pass locally

## Code Quality

- [ ] Code is readable and self-documenting
- [ ] Variable/function names are descriptive
- [ ] No unnecessary complexity
- [ ] DRY principle followed (no duplication)
- [ ] SOLID principles considered

## Security

- [ ] No sensitive data exposed
- [ ] Input validation present
- [ ] SQL injection prevented
- [ ] XSS vulnerabilities addressed
- [ ] Authentication/authorization correct

## Performance

- [ ] No obvious performance issues
- [ ] Database queries optimized
- [ ] No N+1 queries
- [ ] Appropriate indexing

## Architecture

- [ ] Follows project patterns
- [ ] Proper separation of concerns
- [ ] No circular dependencies
- [ ] Appropriate abstractions

## Documentation

- [ ] Complex logic is commented
- [ ] API documentation updated
- [ ] README updated if needed
- [ ] Breaking changes documented
```

**Review Comments: Good vs Bad:**

```typescript
// ❌ BAD: Vague and unhelpful
"This looks wrong"
"Why did you do it this way?"
"This won't work"

// ✅ GOOD: Specific and constructive
"This could cause a race condition when multiple users
update the same record. Consider using optimistic locking
or a transaction."

"The current approach works, but extracting this logic
into a separate function would make it more testable.
For example: extractUserPermissions(user)."

"I noticed this query could result in N+1 problem.
Have you considered using eager loading? Like:
User.findAll({ include: [{ model: Post }] })"

// ❌ BAD: Demanding
"Change this immediately"
"Use method X instead"
"This must be refactored"

// ✅ GOOD: Suggesting
"What do you think about using method X? It might simplify this."
"Have you considered refactoring this? Happy to discuss alternatives."
"Suggestion: We could extract this into a helper function."

// ❌ BAD: Personal attack
"You clearly don't understand async/await"
"This is terrible code"

// ✅ GOOD: Focusing on code, offering help
"The async/await pattern here could be improved.
Let me know if you'd like to pair on this."

"This approach might cause issues. Would you like to
discuss alternatives? I'm happy to help."
```

**Review Response Guidelines:**

```typescript
// ❌ BAD: Defensive
"That's how I always do it"
"The old code did it this way"
"It works fine"

// ✅ GOOD: Open and collaborative
"Good point! I'll update it."
"I hadn't considered that. Let me revise."
"Interesting suggestion. I chose this approach because X,
but I'm open to Y. What do you think?"

// When you disagree:
"I see your point about X. I chose this approach because
of Y and Z constraints. Do you think those constraints
justify the trade-off, or should we prioritize X?"
```

**Review Timing:**

```bash
# Small PRs (< 200 lines)
Target review time: 1-2 hours
Maximum wait: 4 hours

# Medium PRs (200-500 lines)
Target review time: 4 hours
Maximum wait: 1 day

# Large PRs (> 500 lines)
❌ Should be broken down
If unavoidable: 1-2 days

# Critical hotfixes
Target review time: 30 minutes
Can be merged with single approval
```

---

### 3.5.7 Pull Request Best Practices

**PR Size Guidelines:**

```bash
# Ideal PR sizes (lines of code changed)
Tiny:   1-50 lines    (5 minutes to review)
Small:  51-200 lines  (15-30 minutes to review)
Medium: 201-500 lines (1-2 hours to review)
Large:  501-1000 lines (half day to review)
Huge:   1000+ lines   (❌ Split into multiple PRs)

# How to check PR size
git diff --stat main...feature/my-branch
```

**Breaking Down Large PRs:**

```bash
# ❌ BAD: One giant PR
feature/complete-payment-system (2500 lines)
- Payment models
- Payment service
- Payment API
- Payment UI
- Tests
- Documentation

# ✅ GOOD: Multiple focused PRs
1. feature/payment-models (150 lines)
   - Database schema
   - Models
   - Basic tests

2. feature/payment-service (250 lines)
   - Business logic
   - Service layer
   - Unit tests

3. feature/payment-api (200 lines)
   - REST endpoints
   - Validation
   - Integration tests

4. feature/payment-ui (400 lines)
   - Frontend components
   - UI tests
```

**PR Title Conventions:**

```bash
# Use conventional commits format
feat: add user authentication
fix: resolve memory leak in payment processing
refactor: simplify database connection logic
docs: update API documentation
test: add integration tests for user service
chore: update dependencies
perf: optimize database queries
style: fix linting issues

# With scope
feat(auth): add OAuth 2.0 support
fix(payments): handle failed transactions correctly
refactor(database): extract connection logic

# Breaking changes
feat!: change API response format (BREAKING CHANGE)
```

**PR Description Template:**

```markdown
## What does this PR do?

Brief description of the changes and why they're needed.

## Type of Change

- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that causes existing functionality to not work as expected)
- [ ] Documentation update
- [ ] Refactoring (no functional changes)
- [ ] Performance improvement

## How has this been tested?

- [ ] Unit tests
- [ ] Integration tests
- [ ] Manual testing
- [ ] Tested on staging environment

## Screenshots/Recordings (if applicable)

[Add screenshots or screen recordings of UI changes]

## Checklist

- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes
- [ ] Any dependent changes have been merged and published

## Related Issues

Closes #123
Related to #456, #789

## Deployment Notes

- [ ] Requires database migration
- [ ] Requires environment variable changes
- [ ] Requires configuration updates
- [ ] Has breaking changes (document in release notes)

## Additional Context

Any additional information that reviewers should know.

## Questions for Reviewers

- Should we extract the validation logic into a separate service?
- Is the error handling approach appropriate?
```

---

### 3.5.8 Git Commit Message Best Practices

**Conventional Commits Format:**

```bash
():




```

**Types:**

```bash
feat:     New feature
fix:      Bug fix
docs:     Documentation only changes
style:    Code style changes (formatting, semicolons, etc)
refactor: Code change that neither fixes a bug nor adds a feature
perf:     Performance improvement
test:     Adding or updating tests
chore:    Changes to build process or auxiliary tools
ci:       Changes to CI configuration files
build:    Changes that affect the build system
revert:   Reverts a previous commit
```

**Examples:**

```bash
# Simple commit
feat: add user registration endpoint

# With scope
feat(auth): add JWT token generation

# With body
feat(payments): integrate Stripe payment gateway

Implement Stripe payment processing including:
- Payment intent creation
- Webhook handling
- Refund processing
- Error handling for failed payments

# Breaking change
feat!: change API response format

BREAKING CHANGE: API responses now use camelCase instead of snake_case.
Migration guide: https://docs.company.com/migration/v2

# Multiple changes (avoid if possible, break into separate commits)
feat(auth): add OAuth providers

- Add Google OAuth 2.0 integration
- Add GitHub OAuth integration
- Update user model to support OAuth
- Add OAuth callback endpoints

Closes #123, #124
```

**Bad vs Good Commits:**

```bash
# ❌ BAD: Too vague
git commit -m "fix bug"
git commit -m "update code"
git commit -m "changes"
git commit -m "wip"

# ✅ GOOD: Specific and descriptive
git commit -m "fix: resolve null pointer exception in user service"
git commit -m "refactor: extract email validation to separate utility"
git commit -m "feat: add pagination to user list endpoint"

# ❌ BAD: Too many changes in one commit
git commit -m "Add login, fix bug in payments, update documentation, refactor database queries"

# ✅ GOOD: One logical change per commit
git commit -m "feat(auth): add login endpoint"
git commit -m "fix(payments): handle failed transaction state correctly"
git commit -m "docs: update API documentation for auth endpoints"
git commit -m "perf(database): optimize user query with proper indexes"
```

**Atomic Commits:**

```bash
# Each commit should be a complete, working change
# That can be deployed independently

# ❌ BAD: Broken intermediate commits
git commit -m "add user model" # App is broken
git commit -m "add user service" # App is still broken
git commit -m "fix syntax error" # Now it works

# ✅ GOOD: Each commit works
git commit -m "feat: add user model with validation"
git commit -m "feat: add user service with CRUD operations"
git commit -m "test: add user service integration tests"
```

---

## 📖 Day 3: Git Hooks, Monorepo vs Polyrepo & Advanced Topics

### 3.5.9 Git Hooks for Automation

**What are Git Hooks?**
Scripts that run automatically at specific points in the Git workflow.

**Common Hooks:**

```bash
# Client-side hooks (in .git/hooks/)
pre-commit        # Before commit is created
prepare-commit-msg # Before commit message editor opens
commit-msg        # After commit message is entered
post-commit       # After commit is created
post-commit       # After commit is created
pre-push          # Before push to remote
post-merge        # After successful merge

# Server-side hooks
pre-receive       # Before accepting pushed commits
post-receive      # After accepting pushed commits
update            # Before updating each branch
```

#### Setting Up Git Hooks with Husky (Node.js/JavaScript)

**Installation:**

```bash
# Install husky
npm install --save-dev husky

# Install lint-staged for running linters on staged files
npm install --save-dev lint-staged

# Initialize husky
npx husky install

# Add to package.json
npm pkg set scripts.prepare="husky install"
```

**package.json Configuration:**

```json
{
  "scripts": {
    "prepare": "husky install",
    "lint": "eslint . --ext .ts,.tsx",
    "format": "prettier --write \"src/**/*.{ts,tsx,json}\"",
    "test": "jest"
  },
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

**Pre-commit Hook:**

```bash
# Create pre-commit hook
npx husky add .husky/pre-commit "npm run lint-staged"

# .husky/pre-commit file content:
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🔍 Running pre-commit checks..."

# Run lint-staged
npx lint-staged

# Run tests on staged files
npm run test -- --findRelatedTests --passWithNoTests

echo "✅ Pre-commit checks passed!"
```

**Commit-msg Hook (Validate commit messages):**

```bash
# Install commitlint
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Create commitlint.config.js
cat > commitlint.config.js << EOF
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',
        'fix',
        'docs',
        'style',
        'refactor',
        'perf',
        'test',
        'chore',
        'revert',
        'ci',
        'build'
      ]
    ],
    'subject-case': [0], // Allow any case
    'subject-max-length': [2, 'always', 100]
  }
};
EOF

# Add commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit ${1}'
```

**Pre-push Hook:**

```bash
# .husky/pre-push
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

echo "🧪 Running tests before push..."

# Run full test suite
npm run test

# Check for TypeScript errors
npm run type-check

# Build to ensure no build errors
npm run build

echo "✅ Pre-push checks passed!"
```

---

#### Git Hooks for Java Projects

**Maven Configuration (pom.xml):**

```xml





        com.github.phillipuniverse
        githook-maven-plugin
        1.0.5



              install




#!/bin/bash
echo "Running pre-commit checks..."

# Run checkstyle
mvn checkstyle:check

# Run tests
mvn test

exit $?


#!/bin/bash
echo "Running pre-push checks..."

# Run full build
mvn clean verify

exit $?









        org.apache.maven.plugins
        maven-checkstyle-plugin
        3.3.0

          checkstyle.xml
          true





```

**Pre-commit Hook (Manual setup):**

```bash
# .git/hooks/pre-commit
#!/bin/bash

echo "🔍 Running pre-commit checks for Java project..."

# Get list of staged Java files
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep ".java$")

if [ -z "$STAGED_FILES" ]; then
  echo "No Java files staged"
  exit 0
fi

# Run Maven checkstyle
echo "Running Checkstyle..."
mvn checkstyle:check
if [ $? -ne 0 ]; then
  echo "❌ Checkstyle failed. Please fix code style issues."
  exit 1
fi

# Run tests
echo "Running tests..."
mvn test -Dtest=*Test
if [ $? -ne 0 ]; then
  echo "❌ Tests failed. Please fix failing tests."
  exit 1
fi

echo "✅ Pre-commit checks passed!"
exit 0
```

---

#### Git Hooks for Go Projects

**Pre-commit Hook:**

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "🔍 Running pre-commit checks for Go project..."

# Format code
echo "Running go fmt..."
UNFORMATTED=$(gofmt -l .)
if [ -n "$UNFORMATTED" ]; then
  echo "❌ The following files are not formatted:"
  echo "$UNFORMATTED"
  echo "Run: go fmt ./..."
  exit 1
fi

# Run go vet
echo "Running go vet..."
go vet ./...
if [ $? -ne 0 ]; then
  echo "❌ go vet found issues"
  exit 1
fi

# Run golangci-lint
echo "Running golangci-lint..."
golangci-lint run
if [ $? -ne 0 ]; then
  echo "❌ golangci-lint found issues"
  exit 1
fi

# Run tests
echo "Running tests..."
go test ./... -race -coverprofile=coverage.out
if [ $? -ne 0 ]; then
  echo "❌ Tests failed"
  exit 1
fi

# Check test coverage
COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
THRESHOLD=80

if (( $(echo "$COVERAGE < $THRESHOLD" | bc -l) )); then
  echo "❌ Test coverage is below ${THRESHOLD}%: ${COVERAGE}%"
  exit 1
fi

echo "✅ Pre-commit checks passed! Coverage: ${COVERAGE}%"
exit 0
```

**Pre-push Hook:**

```bash
#!/bin/bash
# .git/hooks/pre-push

echo "🧪 Running pre-push checks for Go project..."

# Build binary
echo "Building binary..."
go build -o bin/app ./cmd/app
if [ $? -ne 0 ]; then
  echo "❌ Build failed"
  exit 1
fi

# Run integration tests
echo "Running integration tests..."
go test ./tests/integration/... -v
if [ $? -ne 0 ]; then
  echo "❌ Integration tests failed"
  exit 1
fi

# Check for security vulnerabilities
echo "Running security checks..."
gosec ./...
if [ $? -ne 0 ]; then
  echo "⚠️  Security issues found"
  # Don't fail on security warnings, just warn
fi

echo "✅ Pre-push checks passed!"
exit 0
```

**Makefile for Easy Hook Management:**

```makefile
# Makefile
.PHONY: install-hooks

install-hooks:
	@echo "Installing Git hooks..."
	chmod +x scripts/pre-commit.sh
	chmod +x scripts/pre-push.sh
	ln -sf ../../scripts/pre-commit.sh .git/hooks/pre-commit
	ln -sf ../../scripts/pre-push.sh .git/hooks/pre-push
	@echo "✅ Git hooks installed"

test:
	go test ./... -v -race -coverprofile=coverage.out

lint:
	golangci-lint run

fmt:
	go fmt ./...
	goimports -w .

check: fmt lint test
	@echo "✅ All checks passed"
```

---

### 3.5.10 Monorepo vs Polyrepo

**Monorepo (Single Repository):**

```
monorepo/
├── packages/
│   ├── api/              # Backend API
│   ├── web/              # Web frontend
│   ├── mobile/           # Mobile app
│   ├── shared/           # Shared utilities
│   └── admin/            # Admin dashboard
├── libs/
│   ├── auth/             # Authentication library
│   ├── database/         # Database utilities
│   └── logger/           # Logging utilities
├── tools/
│   └── scripts/          # Build scripts
└── package.json          # Root package.json
```

**Polyrepo (Multiple Repositories):**

```
organization/
├── saas-api              # Backend API repo
├── saas-web              # Web frontend repo
├── saas-mobile           # Mobile app repo
├── saas-shared-utils     # Shared utilities repo
├── saas-auth-lib         # Auth library repo
└── saas-admin            # Admin dashboard repo
```

#### Monorepo vs Polyrepo Comparison

| Aspect                 | Monorepo                         | Polyrepo                  |
| ---------------------- | -------------------------------- | ------------------------- |
| **Code Sharing**       | ✅ Easy (same repo)              | ⚠️ Harder (npm packages)  |
| **Atomic Changes**     | ✅ Single commit across projects | ❌ Multiple commits/PRs   |
| **Build Time**         | ⚠️ Can be slower                 | ✅ Faster (smaller scope) |
| **CI/CD Setup**        | ⚠️ More complex                  | ✅ Simpler                |
| **Version Management** | ✅ Single source of truth        | ⚠️ Multiple versions      |
| **Team Independence**  | ⚠️ Less independent              | ✅ More independent       |
| **Onboarding**         | ⚠️ Overwhelming initially        | ✅ Easier to understand   |
| **Refactoring**        | ✅ Easier across projects        | ❌ Harder across repos    |
| **Access Control**     | ⚠️ All or nothing                | ✅ Granular per repo      |

---

#### Implementing Monorepo with Modern Tools

**Option 1: npm/yarn Workspaces (Node.js)**

**Project Structure:**

```
saas-monorepo/
├── package.json
├── packages/
│   ├── api/
│   │   ├── package.json
│   │   ├── src/
│   │   └── tests/
│   ├── web/
│   │   ├── package.json
│   │   ├── src/
│   │   └── public/
│   ├── shared/
│   │   ├── package.json
│   │   └── src/
│   └── admin/
│       ├── package.json
│       └── src/
└── tsconfig.json
```

**Root package.json:**

```json
{
  "name": "saas-monorepo",
  "private": true,
  "workspaces": ["packages/*"],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces",
    "lint": "npm run lint --workspaces",
    "dev:api": "npm run dev --workspace=packages/api",
    "dev:web": "npm run dev --workspace=packages/web"
  },
  "devDependencies": {
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "typescript": "^5.0.0"
  }
}
```

**packages/api/package.json:**

```json
{
  "name": "@saas/api",
  "version": "1.0.0",
  "main": "dist/index.js",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "test": "jest"
  },
  "dependencies": {
    "@saas/shared": "*",
    "express": "^4.18.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.0",
    "tsx": "^4.0.0"
  }
}
```

**packages/shared/package.json:**

```json
{
  "name": "@saas/shared",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "jest"
  },
  "dependencies": {
    "zod": "^3.22.0"
  }
}
```

**Using Shared Code:**

```typescript
// packages/shared/src/validators.ts
import { z } from "zod";

export const emailSchema = z.string().email();
export const passwordSchema = z.string().min(8);

export const userSchema = z.object({
  email: emailSchema,
  password: passwordSchema,
  name: z.string().min(2),
});

// packages/api/src/routes/auth.ts
import { userSchema } from "@saas/shared";

app.post("/register", (req, res) => {
  const result = userSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json(result.error);
  }
  // Process registration...
});
```

**Installing Dependencies:**

```bash
# Install for all packages
npm install

# Install package-specific dependency
npm install express --workspace=packages/api

# Run scripts
npm run dev:api
npm run test --workspaces

# Build all packages
npm run build
```

---

**Option 2: Turborepo (Advanced Monorepo)**

```bash
# Install Turborepo
npm install turbo --save-dev
```

**turbo.json:**

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

**Benefits:**

- ⚡ Remote caching
- 📦 Incremental builds
- 🔄 Smart dependency management
- 🚀 Parallel execution

```bash
# Run build with caching
npx turbo run build

# Run dev for multiple packages
npx turbo run dev --parallel

# Run tests with dependencies
npx turbo run test
```

---

**Option 3: Nx (Enterprise Monorepo)**

```bash
# Create Nx workspace
npx create-nx-workspace@latest saas-monorepo

# Generate applications
nx generate @nx/node:application api
nx generate @nx/react:application web

# Generate libraries
nx generate @nx/js:library shared
```

**Benefits:**

- 🎯 Dependency graph visualization
- 📊 Affected commands (test only changed code)
- 🔧 Code generation
- 📈 Build optimization

```bash
# Run only affected tests
nx affected:test

# Visualize dependency graph
nx graph

# Build affected projects
nx affected:build
```

---

#### Polyrepo Dependency Management

**Using Private npm Registry:**

```bash
# Publish shared library
cd saas-shared-utils
npm version patch
npm publish --registry=https://npm.mycompany.com

# In other repos, install the package
npm install @mycompany/shared-utils@latest
```

**Git Submodules (Not Recommended):**

```bash
# Add submodule
git submodule add https://github.com/company/shared-utils.git libs/shared

# Clone repo with submodules
git clone --recursive https://github.com/company/saas-api.git

# Update submodules
git submodule update --remote
```

**Recommendation: Use private npm packages instead of git submodules**

---

### 3.5.11 Handling Merge Conflicts

**Common Conflict Scenarios:**

**Scenario 1: Same File, Different Changes**

```bash
# Developer A changes line 10
# Developer B changes line 15
# Both commit to different branches

git merge feature/b
# Auto-merge successful (no conflict)

# Developer A changes line 10 to "X"
# Developer B changes line 10 to "Y"
# Merge conflict occurs
```

**Resolving Conflicts:**

```bash
# Attempt merge
git merge feature/payment-integration

# Conflict occurs
Auto-merging src/services/payment.service.ts
CONFLICT (content): Merge conflict in src/services/payment.service.ts
Automatic merge failed; fix conflicts and then commit the result.

# Check conflicted files
git status

# Open conflicted file
```

**Conflict Markers:**

```typescript
// src/services/payment.service.ts
export class PaymentService {
  async processPayment(amount: number) {
<<<<<<< HEAD (Current changes - your branch)
    // Your implementation
    const result = await this.stripe.createPayment(amount);
    return { success: true, transactionId: result.id };
=======
    // Their implementation
    const payment = await this.stripeClient.process(amount);
    return { status: 'success', id: payment.paymentId };
>>>>>>> feature/payment-integration (Incoming changes)
  }
}
```

**Resolution Steps:**

```typescript
// Option 1: Keep your changes
export class PaymentService {
  async processPayment(amount: number) {
    const result = await this.stripe.createPayment(amount);
    return { success: true, transactionId: result.id };
  }
}

// Option 2: Keep their changes
export class PaymentService {
  async processPayment(amount: number) {
    const payment = await this.stripeClient.process(amount);
    return { status: 'success', id: payment.paymentId };
  }
}

// Option 3: Combine both (best in most cases)
export class PaymentService {
  async processPayment(amount: number) {
    // Use their naming but your return format
    const payment = await this.stripeClient.process(amount);
    return {
      success: true,
      transactionId: payment.paymentId,
      status: 'success'
    };
  }
}

// After resolving
git add src/services/payment.service.ts
git commit -m "merge: resolve payment service conflict"
```

**Conflict Resolution Tools:**

```bash
# Visual merge tool
git mergetool

# Use VS Code
code --wait --merge

# Aborting merge
git merge --abort

# Check which files have conflicts
git diff --name-only --diff-filter=U

# Take all changes from one side
git checkout --theirs src/services/payment.service.ts  # Take their version
git checkout --ours src/services/payment.service.ts    # Take your version
```

---

### 3.5.12 Advanced Git Techniques

#### Interactive Rebase

**Cleaning Up Commit History:**

```bash
# Last 5 commits
git rebase -i HEAD~5

# Interactive editor opens:
pick abc123 feat: add user model
pick def456 fix: typo in user model
pick ghi789 feat: add user service
pick jkl012 fix: import error
pick mno345 test: add user tests

# Change to:
pick abc123 feat: add user model
fixup def456 fix: typo in user model
pick ghi789 feat: add user service
fixup jkl012 fix: import error
pick mno345 test: add user tests

# Result: 3 clean commits instead of 5
```

**Rebase Commands:**

```bash
pick    # Use commit as-is
reword  # Use commit, but edit message
edit    # Use commit, but stop for amending
squash  # Combine with previous commit, edit message
fixup   # Combine with previous commit, discard message
drop    # Remove commit
```

**Example: Splitting a Commit:**

```bash
# Mark commit for editing
git rebase -i HEAD~3
# Change 'pick' to 'edit' for the commit

# Reset the commit
git reset HEAD^

# Stage and commit in parts
git add src/models/user.ts
git commit -m "feat: add user model"

git add src/services/user.ts
git commit -m "feat: add user service"

# Continue rebase
git rebase --continue
```

---

#### Git Bisect (Find Bug-Introducing Commit)

```bash
# Start bisect
git bisect start

# Mark current state as bad
git bisect bad

# Mark a known good commit
git bisect good v1.2.0

# Git checks out a commit in the middle
# Test if bug exists

# If bug exists
git bisect bad

# If bug doesn't exist
git bisect good

# Repeat until Git finds the commit
# Git will say:
# abc123 is the first bad commit

# End bisect
git bisect reset
```

**Automated Bisect:**

```bash
# Run bisect with test script
git bisect start HEAD v1.2.0
git bisect run npm test

# Git automatically finds the bad commit
```

---

#### Git Worktree (Multiple Working Directories)

```bash
# Create new worktree for hotfix
git worktree add ../saas-hotfix hotfix/critical-bug

# Work in hotfix directory
cd ../saas-hotfix
# Make changes
git commit -am "fix: resolve critical bug"
git push origin hotfix/critical-bug

# Back to main work
cd ../saas-main

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../saas-hotfix
```

---

#### Git Reflog (Recover Lost Commits)

```bash
# View reflog
git reflog

# Output:
abc123 (HEAD -> main) HEAD@{0}: commit: feat: add payment
def456 HEAD@{1}: reset: moving to HEAD~1
ghi789 HEAD@{2}: commit: feat: user authentication (lost commit!)

# Recover lost commit
git checkout ghi789
git branch recovered-feature ghi789

# Or reset to lost commit
git reset --hard ghi789
```

---

## 🎯 Practical Exercises

### Exercise 1: Implement Git Flow

**Task:** Set up a complete Git Flow workflow for a new feature.

```bash
# 1. Initialize repository
git clone https://github.com/company/saas-backend.git
cd saas-backend

# 2. Ensure you have main and develop
git checkout main
git pull origin main
git checkout -b develop

# 3. Start a new feature
git checkout develop
git pull origin develop
git checkout -b feature/add-notifications

# 4. Implement feature (make 3-4 commits)
touch src/services/notification.service.ts
git add .
git commit -m "feat(notifications): add notification service"

touch src/controllers/notification.controller.ts
git add .
git commit -m "feat(notifications): add notification endpoints"

touch tests/notification.test.ts
git add .
git commit -m "test(notifications): add notification tests"

# 5. Keep feature updated
git fetch origin
git rebase origin/develop

# 6. Push feature
git push -u origin feature/add-notifications

# 7. Create pull request (on GitHub/GitLab)
# Review and get approval

# 8. Merge to develop
git checkout develop
git pull origin develop
git merge --no-ff feature/add-notifications
git push origin develop

# 9. Clean up
git branch -d feature/add-notifications
git push origin --delete feature/add-notifications
```

---

### Exercise 2: Set Up Git Hooks

**Task:** Create pre-commit and pre-push hooks for code quality.

**For Node.js Project:**

```bash
# 1. Install dependencies
npm install --save-dev husky lint-staged @commitlint/cli @commitlint/config-conventional

# 2. Initialize husky
npx husky install
npm pkg set scripts.prepare="husky install"

# 3. Create pre-commit hook
npx husky add .husky/pre-commit "npx lint-staged"

# 4. Configure lint-staged in package.json
{
  "lint-staged": {
    "*.{ts,tsx,js,jsx}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}

# 5. Create commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit ${1}'

# 6. Create commitlint.config.js
echo "module.exports = {extends: ['@commitlint/config-conventional']}" > commitlint.config.js

# 7. Test
git add .
git commit -m "test: testing git hooks"  # Should run hooks
git commit -m "bad commit message"        # Should fail
```

---

### Exercise 3: Resolve Merge Conflict

**Task:** Practice resolving a realistic merge conflict.

```bash
# Setup: Create conflict scenario
git checkout -b feature-a
echo "const apiUrl = 'https://api-v1.example.com';" > config.ts
git add config.ts
git commit -m "feat: add API v1 URL"

git checkout main
git checkout -b feature-b
echo "const apiUrl = 'https://api-v2.example.com';" > config.ts
git add config.ts
git commit -m "feat: add API v2 URL"

# Merge feature-a first
git checkout main
git merge feature-a

# Now merge feature-b (conflict!)
git merge feature-b

# Resolve conflict in config.ts
# Choose the appropriate version or combine

# After resolving
git add config.ts
git commit -m "merge: resolve API URL conflict, use v2"
```

---

## 📚 Best Practices Summary

### Branching Strategy

✅ **DO:**

- Choose strategy based on team size and deploy frequency
- Keep branches short-lived (< 2 weeks)
- Update branches regularly from main/develop
- Delete branches after merging
- Use descriptive branch names
- Protect main branch (require PR reviews)

❌ **DON'T:**

- Create long-lived feature branches (>1 month)
- Commit directly to main/develop (except hotfixes)
- Leave abandoned branches
- Use vague branch names (fix, update, new-feature)

---

### Code Reviews

✅ **DO:**

- Review within 24 hours
- Keep PRs small (< 400 lines)
- Provide constructive feedback
- Ask questions, don't demand
- Approve when code is good enough
- Use automated checks (CI/CD)

❌ **DON'T:**

- Nitpick on style (use linters)
- Block PRs for perfect code
- Review line-by-line without understanding context
- Make it personal

---

### Commits

✅ **DO:**

- Write descriptive commit messages
- Use conventional commits format
- Make atomic commits (one logical change)
- Commit frequently
- Keep commits focused

❌ **DON'T:**

- Use vague messages ("fix", "update")
- Commit broken code
- Mix multiple unrelated changes
- Commit sensitive data
- Force push to shared branches

---

### Git Hygiene

✅ **DO:**

- Pull before pushing
- Rebase feature branches regularly
- Use `.gitignore` properly
- Tag releases
- Clean up old branches

❌ **DON'T:**

- Force push (`--force`) without `--force-with-lease`
- Commit generated files
- Leave merge conflict markers
- Rewrite published history

---

## 🔍 Debugging Git Issues

### Problem: Accidentally Committed to Wrong Branch

```bash
# Save commits to new branch
git branch feature/correct-branch

# Reset current branch
git reset --hard origin/main

# Switch to correct branch
git checkout feature/correct-branch
```

---

### Problem: Need to Undo Last Commit

```bash
# Keep changes, undo commit
git reset --soft HEAD~1

# Discard changes and commit
git reset --hard HEAD~1

# Safe undo (for shared branches)
git revert HEAD
```

---

### Problem: Merge Conflict is Too Complex

```bash
# Abort merge
git merge --abort

# Try rebase instead (cleaner history)
git rebase develop

# Or use theirs/ours strategy
git merge -X theirs feature/branch    # Prefer their changes
git merge -X ours feature/branch      # Prefer our changes
```

---

### Problem: Accidentally Pushed Sensitive Data

```bash
# Remove file from history (DANGEROUS)
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/secret.txt" \
  --prune-empty --tag-name-filter cat -- --all

# Force push (coordinate with team)
git push origin --force --all

# Better: Use BFG Repo-Cleaner
brew install bfg
bfg --delete-files secret.txt
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force
```

---

## 📖 Additional Resources

### Documentation

- **Git Official Docs**: https://git-scm.com/doc
- **Atlassian Git Tutorials**: https://www.atlassian.com/git/tutorials
- **GitHub Guides**: https://guides.github.com/
- **Git Flow**: https://nvie.com/posts/a-successful-git-branching-model/
- **Trunk-Based Development**: https://trunkbaseddevelopment.com/

### Tools

- **Husky**: https://typicode.github.io/husky/
- **Commitlint**: https://commitlint.js.org/
- **Turborepo**: https://turbo.build/
- **Nx**: https://nx.dev/
- **GitKraken**: Visual Git client
- **Sourcetree**: Free Git GUI

### Books

📚 **"Pro Git"** by Scott Chacon (Free online)
📚 **"Git for Teams"** by Emma Jane Hogbin Westby

---

## 🎯 Module Completion Checklist

- [ ] Understand Git Flow, GitHub Flow, and Trunk-Based Development
- [ ] Choose appropriate branching strategy for your team
- [ ] Write meaningful commit messages
- [ ] Create effective pull requests
- [ ] Review code constructively
- [ ] Set up Git hooks for automation
- [ ] Handle merge conflicts confidently
- [ ] Understand monorepo vs polyrepo trade-offs
- [ ] Use advanced Git commands (rebase, bisect, reflog)
- [ ] Implement branching strategy in your project

---
