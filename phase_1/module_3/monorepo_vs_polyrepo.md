# Module 3: Project Setup & Development Environment

---

## Module 3.1: Project Structure & Organization

---

## 🎯 Learning Objectives

By the end of this section, you will:

- Understand the differences between monorepo and polyrepo strategies
- Know when to use each approach for SaaS applications
- Implement effective monorepo management with modern tools
- Design scalable polyrepo architectures
- Make informed decisions based on team size and project complexity
- Set up proper code organization for all three tech stacks

---

## 📖 Monorepo vs Polyrepo: The Great Debate

### What is a Monorepo?

A **monorepo** (monolithic repository) is a single repository that contains multiple projects, services, or packages. All code lives in one place with shared tooling and dependencies.

**Examples**: Google (entire company in one repo), Meta, Microsoft, Uber

### What is a Polyrepo?

A **polyrepo** (multiple repositories) strategy uses separate repositories for different services, libraries, or components. Each repo is independent with its own versioning and tooling.

**Examples**: Netflix (microservices), Amazon, most traditional organizations

---

## 🏗️ Monorepo Strategy

### Characteristics

```
my-saas-app/
├── packages/
│   ├── api/              # Backend API
│   ├── web/              # Web frontend
│   ├── mobile/           # Mobile app
│   ├── shared/           # Shared utilities
│   └── design-system/    # UI components
├── services/
│   ├── auth-service/
│   ├── payment-service/
│   └── notification-service/
├── tools/                # Build tools
├── docs/                 # Documentation
└── package.json          # Root config
```

### ✅ Advantages

**1. Code Sharing is Effortless**

- Share code between projects instantly
- No need to publish packages
- Import directly: `import { utils } from '@myapp/shared'`

**2. Atomic Changes Across Projects**

- Update API and frontend in single commit
- No version mismatch issues
- Easier to maintain consistency

**3. Unified Tooling & Dependencies**

- One set of build tools
- Consistent linting, formatting, testing
- Single dependency update affects all projects

**4. Simplified Dependency Management**

- See all dependencies in one place
- Easier to detect and remove duplicates
- Better version control

**5. Better Refactoring**

- Refactor across multiple projects safely
- IDE finds all usages across entire codebase
- Rename/move with confidence

**6. Easier Developer Onboarding**

- Clone once, see everything
- Consistent structure everywhere
- Single setup command

**7. Coordinated Releases**

- Deploy related changes together
- Easier to track what changed
- Better release notes

### ❌ Disadvantages

**1. Scalability Challenges**

- Large repos slow down Git operations
- IDE can become sluggish
- Long CI/CD times

**2. Tight Coupling Risk**

- Teams might create dependencies too easily
- Harder to enforce boundaries
- Service isolation can suffer

**3. Complex Tooling Required**

- Need specialized tools (Nx, Turborepo, Lerna)
- Learning curve for build systems
- More complex CI/CD setup

**4. Access Control Limitations**

- Hard to restrict access to specific projects
- Everyone sees everything
- Security concerns for sensitive code

**5. Single Point of Failure**

- One bad commit can break everything
- Repo corruption affects all projects
- Backup/recovery more critical

**6. Longer Build Times**

- Must rebuild affected projects
- Complex dependency graphs
- Need incremental builds

### 🎯 When to Use Monorepo

**Perfect For:**

- **Startups** (5-50 developers): Fast iteration, high cohesion
- **Tight Integration**: Frontend + Backend + Mobile working together
- **Platform Products**: Shared design system, common utilities
- **Small to Medium Teams**: Can manage complexity
- **Rapid Prototyping**: Quick changes across stack

**Examples:**

- SaaS app with web + mobile + API
- E-commerce platform (frontend, backend, admin panel)
- Internal tools suite (all tools share auth, UI)

### 🛠️ Monorepo Tools by Language

#### JavaScript/TypeScript

- **Nx** (Recommended): https://nx.dev/
- **Turborepo**: https://turbo.build/
- **Lerna**: https://lerna.js.org/
- **Yarn Workspaces**: Built-in
- **pnpm Workspaces**: Built-in

#### Java

- **Gradle Multi-Project**: Built-in
- **Maven Modules**: Built-in
- **Bazel**: Google's build system

#### Go

- **Go Workspaces**: Built-in (Go 1.18+)
- **Bazel**: For large projects
- **Native multi-module**: Simple approach

---

## 🏢 Polyrepo Strategy

### Characteristics

```
Organization: mycompany
├── repo: api-service
│   └── (Complete API codebase)
├── repo: web-frontend
│   └── (Complete frontend codebase)
├── repo: mobile-app
│   └── (Complete mobile codebase)
├── repo: shared-utils (Published as package)
│   └── (Shared utilities library)
├── repo: auth-service
│   └── (Authentication microservice)
└── repo: payment-service
    └── (Payment microservice)
```

### ✅ Advantages

**1. Clear Separation of Concerns**

- Each repo has single responsibility
- Strong service boundaries
- Easier to reason about

**2. Independent Scaling**

- Repos can grow independently
- No single giant repository
- Better Git performance per repo

**3. Team Autonomy**

- Teams own their repositories
- Independent release cycles
- Flexible tech stack choices

**4. Granular Access Control**

- Easy to restrict access per repo
- Better security model
- Clearer ownership

**5. Simpler CI/CD Per Repo**

- Faster builds (only one service)
- Easier to set up
- Less complex pipelines

**6. Technology Flexibility**

- Different languages per service
- Different tooling per repo
- Experiment easily

**7. Easier Open Source**

- Can open source individual repos
- Clearer licensing per project
- Community contributions easier

### ❌ Disadvantages

**1. Code Duplication**

- Same utility code in multiple repos
- Harder to share code
- Inconsistencies develop

**2. Dependency Hell**

- Version mismatches between repos
- Breaking changes harder to coordinate
- Dependency graph sprawl

**3. Harder to Refactor**

- Can't refactor across repos atomically
- Must update multiple repos sequentially
- Risk of breaking changes

**4. Coordination Overhead**

- Must coordinate releases
- More communication needed
- Cross-repo changes difficult

**5. Duplicate Tooling Setup**

- Configure CI/CD for each repo
- Maintain multiple build configs
- Keep tooling versions in sync

**6. Onboarding Complexity**

- Must clone multiple repos
- Setup multiple environments
- Harder to see full picture

**7. Shared Library Overhead**

- Must publish packages
- Version management complexity
- Update propagation delays

### 🎯 When to Use Polyrepo

**Perfect For:**

- **Large Organizations** (50+ developers): Clear ownership needed
- **Microservices Architecture**: Truly independent services
- **Different Tech Stacks**: Each service uses different language
- **Multiple Teams**: Teams want autonomy
- **Open Source Projects**: Need separate repos for community

**Examples:**

- Large microservices architecture (10+ services)
- Platform with plugins (core + many plugins)
- Multi-tenant SaaS with customer-specific code
- Open source ecosystem (core + extensions)

### 🛠️ Polyrepo Management Tools

- **Git Submodules**: Link repos together
- **Meta**: Facebook's tool for managing multiple repos
- **mkrepo**: Manage multiple repos
- **GitHub Organizations**: Group related repos
- **Private NPM/Maven/Go Registries**: Share packages

---

## 📊 Comparison Matrix

| Aspect                 | Monorepo               | Polyrepo                  |
| ---------------------- | ---------------------- | ------------------------- |
| **Team Size**          | Best for 5-50 devs     | Better for 50+ devs       |
| **Code Sharing**       | ⭐⭐⭐⭐⭐ Instant     | ⭐⭐ Via packages         |
| **Atomic Changes**     | ⭐⭐⭐⭐⭐ Easy        | ⭐ Complex                |
| **Service Boundaries** | ⭐⭐ Can blur          | ⭐⭐⭐⭐⭐ Clear          |
| **CI/CD Complexity**   | ⭐⭐⭐ Moderate        | ⭐⭐⭐⭐ Simpler per repo |
| **Build Speed**        | ⭐⭐ Can be slow       | ⭐⭐⭐⭐ Fast per repo    |
| **Tooling Setup**      | ⭐⭐⭐ Complex upfront | ⭐⭐⭐⭐ Simple per repo  |
| **Access Control**     | ⭐⭐ Limited           | ⭐⭐⭐⭐⭐ Granular       |
| **Refactoring**        | ⭐⭐⭐⭐⭐ Easy        | ⭐⭐ Hard                 |
| **Onboarding**         | ⭐⭐⭐⭐ One clone     | ⭐⭐ Multiple clones      |
| **Version Management** | ⭐⭐⭐⭐ Unified       | ⭐⭐ Complex              |
| **Tech Diversity**     | ⭐⭐ Limited           | ⭐⭐⭐⭐⭐ Flexible       |

---

## 🎭 Hybrid Approaches

You don't have to choose just one! Many organizations use hybrid strategies.

### Strategy 1: Monorepo per Team

```
Team A Monorepo:
├── api
├── web
└── shared

Team B Monorepo:
├── mobile
├── backend
└── common

Shared Libraries Polyrepo:
├── design-system (published package)
├── auth-sdk (published package)
└── analytics (published package)
```

**Use When**: Multiple teams, but each team works on tightly coupled projects.

### Strategy 2: Monorepo for Frontend, Polyrepo for Services

```
Frontend Monorepo:
├── web-app
├── mobile-app
├── admin-panel
└── shared-ui

Backend Polyrepos:
├── repo: auth-service
├── repo: payment-service
├── repo: user-service
└── repo: notification-service
```

**Use When**: Frontend needs tight integration, backend services are independent.

### Strategy 3: Monorepo with Published Packages

```
Main Monorepo:
├── packages/
│   ├── app/ (private)
│   ├── api/ (private)
│   └── ui-kit/ (published to registry)
└── services/
    └── core/ (private)

External Repos can import:
npm install @mycompany/ui-kit
```

**Use When**: Want monorepo benefits but need to share with external projects.

---

## 💻 Monorepo Setup Examples

### JavaScript/TypeScript with Nx

#### Setup

```bash
# Install Nx globally
npm install -g nx

# Create new Nx workspace
npx create-nx-workspace@latest my-saas-app

# Choose: integrated monorepo
# Choose: apps (for multiple apps)
```

#### Directory Structure

```
my-saas-app/
├── apps/
│   ├── api/                    # NestJS API
│   │   ├── src/
│   │   ├── project.json
│   │   └── tsconfig.json
│   ├── web/                    # React Web App
│   │   ├── src/
│   │   ├── project.json
│   │   └── tsconfig.json
│   └── mobile/                 # React Native
│       ├── src/
│       └── project.json
├── libs/
│   ├── shared/
│   │   ├── utils/             # Shared utilities
│   │   ├── types/             # Shared TypeScript types
│   │   └── config/            # Shared configuration
│   ├── ui/                    # Shared UI components
│   │   ├── button/
│   │   ├── input/
│   │   └── modal/
│   └── data-access/           # Shared data layer
│       ├── api-client/
│       └── models/
├── tools/                      # Build/dev tools
├── nx.json                     # Nx configuration
├── package.json               # Root package.json
├── tsconfig.base.json         # Base TypeScript config
└── README.md
```

#### Configuration

```json
// nx.json
{
  "npmScope": "myapp",
  "affected": {
    "defaultBase": "main"
  },
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint"]
      }
    }
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"]
    }
  }
}
```

```json
// package.json
{
  "name": "my-saas-app",
  "version": "1.0.0",
  "private": true,
  "workspaces": ["apps/*", "libs/*"],
  "scripts": {
    "start:api": "nx serve api",
    "start:web": "nx serve web",
    "build:all": "nx run-many --target=build --all",
    "test:all": "nx run-many --target=test --all",
    "lint:all": "nx run-many --target=lint --all"
  },
  "devDependencies": {
    "nx": "^17.0.0",
    "@nx/node": "^17.0.0",
    "@nx/react": "^17.0.0",
    "typescript": "^5.0.0"
  }
}
```

#### Usage

```typescript
// In apps/web/src/app.tsx
// Import from shared library
import { formatCurrency } from "@myapp/shared/utils";
import { Button } from "@myapp/ui/button";
import { UserService } from "@myapp/data-access/api-client";

export function App() {
  const user = UserService.getCurrentUser();

  return (
    <div>
      <h1>Welcome {user.name}</h1>
      <p>Balance: {formatCurrency(user.balance)}</p>
      <Button onClick={() => console.log("Clicked")}>Click Me</Button>
    </div>
  );
}
```

#### Commands

```bash
# Serve specific app
nx serve api
nx serve web

# Build specific app
nx build api --prod

# Run tests
nx test shared-utils
nx test api

# Build all affected projects (after changes)
nx affected:build

# Visualize dependency graph
nx graph
```

### JavaScript/TypeScript with Turborepo

#### Setup

```bash
# Create new Turborepo
npx create-turbo@latest my-saas-app
```

#### Directory Structure

```
my-saas-app/
├── apps/
│   ├── web/                   # Next.js app
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── api/                   # Express API
│       ├── src/
│       ├── package.json
│       └── tsconfig.json
├── packages/
│   ├── ui/                    # Shared UI components
│   │   ├── src/
│   │   └── package.json
│   ├── config/                # Shared configs
│   │   └── package.json
│   └── tsconfig/              # Shared TS configs
│       └── package.json
├── turbo.json                 # Turborepo config
├── package.json               # Root package.json
└── pnpm-workspace.yaml        # Workspace config
```

#### Configuration

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["^build"]
    },
    "lint": {},
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

```json
// package.json
{
  "name": "my-saas-app",
  "private": true,
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint"
  },
  "devDependencies": {
    "turbo": "^1.10.0"
  },
  "packageManager": "pnpm@8.0.0"
}
```

#### Commands

```bash
# Run dev for all apps
pnpm dev

# Build all packages/apps
pnpm build

# Build only changed packages
turbo run build --filter=[main]

# Run specific app
pnpm --filter web dev
pnpm --filter api build
```

### Java with Gradle Multi-Project

#### Setup

```bash
# Create root project
mkdir my-saas-app
cd my-saas-app
gradle init
```

#### Directory Structure

```
my-saas-app/
├── api/                       # API module
│   ├── src/
│   │   ├── main/java/
│   │   └── test/java/
│   └── build.gradle
├── web/                       # Web module
│   ├── src/
│   │   ├── main/java/
│   │   └── test/java/
│   └── build.gradle
├── shared/                    # Shared module
│   ├── utils/
│   │   ├── src/
│   │   └── build.gradle
│   ├── models/
│   │   ├── src/
│   │   └── build.gradle
│   └── config/
│       ├── src/
│       └── build.gradle
├── services/                  # Microservices
│   ├── auth-service/
│   │   ├── src/
│   │   └── build.gradle
│   └── payment-service/
│       ├── src/
│       └── build.gradle
├── settings.gradle            # Project structure
├── build.gradle              # Root build config
└── gradle.properties         # Global properties
```

#### Configuration

```groovy
// settings.gradle
rootProject.name = 'my-saas-app'

include 'api'
include 'web'
include 'shared:utils'
include 'shared:models'
include 'shared:config'
include 'services:auth-service'
include 'services:payment-service'
```

```groovy
// build.gradle (root)
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0' apply false
    id 'io.spring.dependency-management' version '1.1.0' apply false
}

allprojects {
    group = 'com.myapp'
    version = '1.0.0'
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'

    repositories {
        mavenCentral()
    }

    java {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    dependencies {
        testImplementation 'org.junit.jupiter:junit-jupiter:5.9.0'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
    }

    test {
        useJUnitPlatform()
    }
}
```

```groovy
// api/build.gradle
plugins {
    id 'org.springframework.boot'
}

dependencies {
    // Depend on shared modules
    implementation project(':shared:utils')
    implementation project(':shared:models')
    implementation project(':shared:config')

    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
}
```

#### Usage

```java
// In api module - import from shared modules
package com.myapp.api;

import com.myapp.shared.utils.StringUtils;  // From shared:utils
import com.myapp.shared.models.User;         // From shared:models
import com.myapp.shared.config.AppConfig;    // From shared:config

@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable String id) {
        String formattedId = StringUtils.format(id);
        return userService.findById(formattedId);
    }
}
```

#### Commands

```bash
# Build all modules
./gradlew build

# Build specific module
./gradlew :api:build

# Run specific module
./gradlew :api:bootRun

# Run tests for all modules
./gradlew test

# Run tests for specific module
./gradlew :shared:utils:test

# Clean build
./gradlew clean build

# Show dependency tree
./gradlew :api:dependencies
```

### Go with Workspaces

#### Setup

```bash
# Create workspace
mkdir my-saas-app
cd my-saas-app
go work init
```

#### Directory Structure

```
my-saas-app/
├── cmd/
│   ├── api/                   # API service
│   │   └── main.go
│   ├── web/                   # Web service
│   │   └── main.go
│   └── worker/                # Background worker
│       └── main.go
├── internal/
│   ├── api/                   # API package
│   │   ├── handler/
│   │   ├── middleware/
│   │   └── router/
│   ├── domain/                # Domain models
│   │   ├── user/
│   │   └── order/
│   └── infrastructure/        # Infrastructure
│       ├── database/
│       ├── cache/
│       └── queue/
├── pkg/                       # Public packages
│   ├── utils/                 # Shared utilities
│   │   ├── go.mod
│   │   └── string.go
│   ├── config/                # Shared config
│   │   ├── go.mod
│   │   └── config.go
│   └── logger/                # Shared logger
│       ├── go.mod
│       └── logger.go
├── go.work                    # Workspace config
├── go.mod                     # Root module
└── README.md
```

#### Configuration

```go
// go.work
go 1.21

use (
    .
    ./pkg/utils
    ./pkg/config
    ./pkg/logger
)
```

```go
// go.mod (root)
module github.com/mycompany/my-saas-app

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/lib/pq v1.10.9
)
```

```go
// pkg/utils/go.mod
module github.com/mycompany/my-saas-app/pkg/utils

go 1.21
```

#### Usage

```go
// cmd/api/main.go
package main

import (
    "github.com/mycompany/my-saas-app/pkg/utils"
    "github.com/mycompany/my-saas-app/pkg/config"
    "github.com/mycompany/my-saas-app/pkg/logger"
    "github.com/mycompany/my-saas-app/internal/api/router"
)

func main() {
    // Use shared packages
    cfg := config.Load()
    log := logger.New(cfg.LogLevel)

    // Format using shared utils
    appName := utils.FormatString(cfg.AppName)
    log.Info("Starting application: " + appName)

    // Setup router
    r := router.Setup()
    r.Run(":8080")
}
```

#### Commands

```bash
# Build all services
go build ./cmd/...

# Build specific service
go build ./cmd/api

# Run specific service
go run ./cmd/api

# Test all packages
go test ./...

# Test specific package
go test ./pkg/utils

# Update dependencies
go get -u ./...
go mod tidy

# Show workspace info
go work use
```

---

## 🎯 Decision Framework

### Choose Monorepo If:

✅ Team size: 5-50 developers
✅ Projects are tightly coupled
✅ Need frequent cross-project changes
✅ Want unified tooling and dependencies
✅ Rapid iteration is priority
✅ Code sharing is common
✅ Single product/platform

### Choose Polyrepo If:

✅ Team size: 50+ developers
✅ Services are independent (true microservices)
✅ Different teams with autonomy
✅ Need granular access control
✅ Different tech stacks per service
✅ Open source components
✅ Clear service boundaries matter

### Choose Hybrid If:

✅ Have multiple teams
✅ Some projects coupled, others independent
✅ Want team autonomy but code sharing
✅ Evolving architecture
✅ Want flexibility

---

## 📊 Real-World Examples

### Monorepo Success Stories

**Google**: Entire company in one repo (~2 billion lines of code)

- Custom build system (Bazel)
- Massive code sharing
- Unified tooling

**Meta (Facebook)**: Single repo for main products

- Custom VCS (Mercurial)
- Extensive code sharing
- Unified development

**Microsoft**: Windows and Office teams use monorepo

- Git + Virtual File System
- Better coordination
- Shared components

### Polyrepo Success Stories

**Netflix**: Hundreds of microservices repos

- Team autonomy
- Independent scaling
- Clear boundaries

**Amazon**: Service-oriented architecture

- Each team owns repos
- Maximum flexibility
- Independent releases

**Spotify**: Squad model with polyrepo

- Team ownership
- Autonomous squads
- Different tech stacks

---

## 🚀 Migration Strategies

### From Polyrepo to Monorepo

```bash
# 1. Create monorepo structure
mkdir my-saas-app
cd my-saas-app
npm init -y

# 2. Move repos as subdirectories
git clone https://github.com/mycompany/api ./apps/api
git clone https://github.com/mycompany/web ./apps/web

# 3. Remove .git directories
rm -rf apps/api/.git
rm -rf apps/web/.git

# 4. Setup monorepo tooling
npx create-nx-workspace@latest --preset=apps
# or
npx create-turbo@latest

# 5. Migrate dependencies to workspace
# Update package.json, build configs

# 6. Test everything
npm run build
npm run test
```

### From Monorepo to Polyrepo

```bash
# 1. Identify boundaries
# Analyze dependencies
nx graph

# 2. Extract subdirectory to new repo
git subtree split -P apps/api -b api-branch
mkdir ../api-service
cd ../api-service
git init
git pull ../my-saas-app api-branch

# 3. Publish shared packages
cd packages/shared-utils
npm publish

# 4. Update dependencies in extracted repo
npm install @mycompany/shared-utils

# 5. Repeat for each service
```

---

## ✅ Best Practices

### For Monorepo:

1. **Use workspace manager**: Nx, Turborepo, or native workspaces
2. **Implement caching**: Cache build outputs to speed up CI
3. **Set up affected builds**: Only build/test changed projects
4. **Define clear boundaries**: Even in monorepo, maintain module boundaries
5. **Use path aliases**: `@myapp/shared` not `../../shared`
6. **Enforce dependencies**: Prevent circular or invalid imports
7. **Document architecture**: Make structure clear to newcomers
8. **Incremental adoption**: Start small, migrate gradually

### For Polyrepo:

1. **Use package registry**: Publish shared code as packages
2. **Version carefully**: Semantic versioning for all packages
3. **Automate updates**: Use Dependabot or Renovate
4. **Template repos**: Standardize structure across repos
5. **Shared tooling**: Keep CI/CD configs consistent
6. **Central documentation**: One place for all repos' docs
7. **Clear ownership**: Each repo has designated owners
8. **API contracts**: Define clear interfaces between services

---
