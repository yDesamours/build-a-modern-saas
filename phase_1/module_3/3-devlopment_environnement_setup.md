# Module 3.2: Development Environment Setup

---

## 🎯 Learning Objectives

By the end of this section, you will:

- Set up optimal development environments for Java, Node.js, and Go
- Configure IDEs for maximum productivity
- Use Docker for local development
- Set up essential development tools and utilities
- Configure linting, formatting, and code quality tools
- Implement Git hooks for automated checks
- Create reproducible development environments

---

## 💻 IDE Setup & Configuration

### Java Development - IntelliJ IDEA

#### Installation

**Download:** https://www.jetbrains.com/idea/download/

**Editions:**

- **Community Edition**: Free, sufficient for most development
- **Ultimate Edition**: Paid, includes advanced features (Spring Boot tools, database tools, HTTP client)

#### Essential Plugins

```
1. Lombok Plugin
   - Reduces boilerplate code
   - Settings → Plugins → Search "Lombok"

2. Rainbow Brackets
   - Colorizes matching brackets
   - Easier to read nested code

3. String Manipulation
   - Useful string utilities (camelCase, snake_case, etc.)

4. SonarLint
   - Real-time code quality feedback
   - Detects bugs and code smells

5. GitToolBox
   - Enhanced Git integration
   - Shows inline blame, ahead/behind counts

6. Key Promoter X
   - Helps learn keyboard shortcuts

7. RestfulTool
   - Quick navigation for REST APIs
```

#### Configuration

**1. Java SDK Setup**

```
File → Project Structure → Project Settings → Project
- Set Project SDK: Java 17 or 21
- Set Project language level: 17 or 21

File → Project Structure → Platform Settings → SDKs
- Add JDK if not present
- Recommended: Amazon Corretto, Eclipse Temurin, or Oracle JDK
```

**2. Maven/Gradle Configuration**

```
Settings → Build, Execution, Deployment → Build Tools → Maven
- ✓ Use settings from .mvn/maven.config
- ✓ Reload project after changes to the build script
- Set Maven home directory

For Gradle:
Settings → Build, Execution, Deployment → Build Tools → Gradle
- Gradle JVM: Use Project JDK
- ✓ Build and run using: Gradle
```

**3. Code Style**

```
Settings → Editor → Code Style → Java
- Tab size: 4
- Indent: 4
- ✓ Use tab character: OFF
- Import: Use "Optimize imports on the fly"

Download Google Java Style:
- Download from: https://github.com/google/styleguide
- Import: Settings → Editor → Code Style → Import Scheme
```

**4. Live Templates (Code Snippets)**

```
Settings → Editor → Live Templates

Create custom templates:
- "psvm" → public static void main(String[] args)
- "sout" → System.out.println()
- "test" → @Test public void testMethod() {}
```

**5. Keyboard Shortcuts (Essential)**

```
Ctrl+Space          - Basic code completion
Ctrl+Shift+Space    - Smart code completion
Ctrl+Alt+L          - Reformat code
Ctrl+Alt+O          - Optimize imports
Shift+F6            - Rename
Ctrl+W              - Extend selection
Ctrl+Shift+A        - Find action
Alt+Enter           - Show intention actions
Ctrl+B              - Go to declaration
Ctrl+Shift+F12      - Toggle maximize editor
```

---

### Node.js/TypeScript Development - VS Code

#### Installation

**Download:** https://code.visualstudio.com/

**Why VS Code?**

- Lightweight and fast
- Excellent TypeScript support
- Huge extension ecosystem
- Built-in Git integration

#### Essential Extensions

```json
{
  "recommendations": [
    // Core Development
    "dbaeumer.vscode-eslint", // ESLint
    "esbenp.prettier-vscode", // Prettier
    "ms-vscode.vscode-typescript-next", // TypeScript

    // Framework Specific
    "angular.ng-template", // Angular
    "octref.vetur", // Vue
    "dsznajder.es7-react-js-snippets", // React

    // Utilities
    "eamodio.gitlens", // Git supercharged
    "ms-azuretools.vscode-docker", // Docker
    "streetsidesoftware.code-spell-checker", // Spell checker
    "usernamehw.errorlens", // Better error display
    "christian-kohler.path-intellisense", // Path autocomplete
    "formulahendry.auto-rename-tag", // Auto rename HTML tags

    // Testing
    "orta.vscode-jest", // Jest
    "hbenl.vscode-test-explorer", // Test Explorer

    // Productivity
    "wayou.vscode-todo-highlight", // TODO/FIXME highlighter
    "oderwat.indent-rainbow", // Colorize indentation
    "coenraads.bracket-pair-colorizer-2", // Bracket colorization

    // API Development
    "humao.rest-client", // REST client
    "42crunch.vscode-openapi" // OpenAPI/Swagger
  ]
}
```

Save as `.vscode/extensions.json` in your project.

#### Configuration (settings.json)

```json
{
  // Editor Settings
  "editor.fontSize": 14,
  "editor.fontFamily": "'Fira Code', 'Cascadia Code', Consolas, monospace",
  "editor.fontLigatures": true,
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true,
    "source.organizeImports": true
  },
  "editor.bracketPairColorization.enabled": true,
  "editor.guides.bracketPairs": true,

  // File Settings
  "files.autoSave": "onFocusChange",
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.next": true,
    "**/coverage": true
  },

  // TypeScript Settings
  "typescript.updateImportsOnFileMove.enabled": "always",
  "typescript.preferences.importModuleSpecifier": "relative",
  "typescript.suggest.autoImports": true,

  // Prettier Settings
  "prettier.singleQuote": true,
  "prettier.trailingComma": "es5",
  "prettier.semi": true,
  "prettier.tabWidth": 2,

  // ESLint Settings
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact"
  ],

  // Git Settings
  "git.autofetch": true,
  "git.confirmSync": false,

  // Terminal
  "terminal.integrated.fontSize": 13,
  "terminal.integrated.defaultProfile.windows": "Git Bash",

  // Emmet
  "emmet.includeLanguages": {
    "typescript": "html",
    "javascript": "html"
  }
}
```

#### Keyboard Shortcuts (Essential)

```
Ctrl+P              - Quick file open
Ctrl+Shift+P        - Command palette
Ctrl+B              - Toggle sidebar
Ctrl+`              - Toggle terminal
Ctrl+Shift+F        - Search in files
F2                  - Rename symbol
Alt+↑/↓             - Move line up/down
Ctrl+D              - Multi-cursor (select next)
Ctrl+Shift+L        - Select all occurrences
Ctrl+/              - Toggle comment
Ctrl+Shift+[/]      - Fold/unfold code
```

---

### Go Development - VS Code or GoLand

#### VS Code Setup for Go

**Install Go Extension:**

```
Extension: golang.go
```

**Configuration:**

```json
{
  // Go Settings
  "go.useLanguageServer": true,
  "go.lintTool": "golangci-lint",
  "go.lintOnSave": "package",
  "go.formatTool": "goimports",
  "go.testFlags": ["-v", "-race"],
  "go.coverOnSave": false,
  "go.coverOnTestPackage": true,

  // Auto-install tools
  "go.toolsManagement.autoUpdate": true,

  "[go]": {
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": true
    },
    "editor.snippetSuggestions": "none"
  },

  // Go Test Explorer
  "go.testExplorer.enable": true
}
```

**Install Go Tools:**

```bash
# Install all Go tools at once
go install golang.org/x/tools/gopls@latest
go install github.com/go-delve/delve/cmd/dlv@latest
go install honnef.co/go/tools/cmd/staticcheck@latest
go install golang.org/x/tools/cmd/goimports@latest
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

#### GoLand Setup (Alternative)

**Download:** https://www.jetbrains.com/go/

**Essential Plugins:**

- Rainbow Brackets
- GitToolBox
- String Manipulation

**Configuration:**

```
Settings → Go → GOROOT: Set Go SDK path
Settings → Go → GOPATH: Set workspace path
Settings → Go → Go Modules: Enable integration
Settings → Tools → File Watchers: Enable gofmt/goimports
```

---

## 🐳 Docker for Local Development

### Why Docker for Development?

✅ **Consistent environments** across team
✅ **Easy service setup** (database, cache, queue)
✅ **No local installation pollution**
✅ **Production parity** (develop in same environment)
✅ **Quick onboarding** for new developers

### Installation

**Docker Desktop:**

- **Windows/Mac**: https://www.docker.com/products/docker-desktop/
- **Linux**: https://docs.docker.com/engine/install/

**Verify Installation:**

```bash
docker --version
docker-compose --version
```

### Docker Compose for Development

#### Example: Full SaaS Stack

```yaml
# docker-compose.yml
version: "3.8"

services:
  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: saas-postgres
    environment:
      POSTGRES_DB: saas_db
      POSTGRES_USER: dev_user
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev_user"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: saas-redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # RabbitMQ Message Queue
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: saas-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: dev_user
      RABBITMQ_DEFAULT_PASS: dev_password
    ports:
      - "5672:5672" # AMQP port
      - "15672:15672" # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Elasticsearch (for search functionality)
  elasticsearch:
    image: elasticsearch:8.11.0
    container_name: saas-elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    healthcheck:
      test:
        ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  # MongoDB (optional, for specific use cases)
  mongodb:
    image: mongo:7-jammy
    container_name: saas-mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: dev_user
      MONGO_INITDB_ROOT_PASSWORD: dev_password
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db

  # Mailhog (Email testing)
  mailhog:
    image: mailhog/mailhog:latest
    container_name: saas-mailhog
    ports:
      - "1025:1025" # SMTP
      - "8025:8025" # Web UI
    logging:
      driver: none # Disable logging for mailhog

  # MinIO (S3-compatible object storage)
  minio:
    image: minio/minio:latest
    container_name: saas-minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000" # API
      - "9001:9001" # Console
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # pgAdmin (PostgreSQL GUI)
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: saas-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - postgres
    volumes:
      - pgadmin_data:/var/lib/pgadmin

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
  elasticsearch_data:
  mongodb_data:
  minio_data:
  pgadmin_data:

networks:
  default:
    name: saas-network
```

#### Usage Commands

```bash
# Start all services
docker-compose up -d

# Start specific services
docker-compose up -d postgres redis

# View logs
docker-compose logs -f
docker-compose logs -f postgres  # Specific service

# Stop all services
docker-compose down

# Stop and remove volumes (DANGER: deletes data)
docker-compose down -v

# Restart services
docker-compose restart

# View running containers
docker-compose ps

# Execute command in container
docker-compose exec postgres psql -U dev_user -d saas_db

# Access Redis CLI
docker-compose exec redis redis-cli

# View service health
docker-compose ps
```

#### Access URLs

```
PostgreSQL:         localhost:5432
Redis:              localhost:6379
RabbitMQ Management: http://localhost:15672 (user: dev_user, pass: dev_password)
Elasticsearch:      http://localhost:9200
MongoDB:            localhost:27017
Mailhog UI:         http://localhost:8025
MinIO Console:      http://localhost:9001 (user: minioadmin, pass: minioadmin)
pgAdmin:            http://localhost:5050 (email: admin@example.com, pass: admin)
```

### Development Dockerfile Examples

#### Java/Spring Boot

```dockerfile
# Dockerfile.dev
FROM eclipse-temurin:17-jdk

WORKDIR /app

# Install Maven
RUN apt-get update && \
    apt-get install -y maven && \
    rm -rf /var/lib/apt/lists/*

# Copy project files
COPY pom.xml .
COPY src ./src

# Download dependencies (cached layer)
RUN mvn dependency:go-offline -B

# Development mode with hot reload
CMD ["mvn", "spring-boot:run", "-Dspring-boot.run.jvmArguments=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"]

EXPOSE 8080 5005
```

#### Node.js/TypeScript

```dockerfile
# Dockerfile.dev
FROM node:20-alpine

WORKDIR /app

# Install dependencies globally for tools
RUN npm install -g nodemon ts-node

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source
COPY . .

# Development mode with hot reload
CMD ["npm", "run", "dev"]

EXPOSE 3000
```

#### Go

```dockerfile
# Dockerfile.dev
FROM golang:1.21-alpine

WORKDIR /app

# Install air for hot reload
RUN go install github.com/cosmtrek/air@latest

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Development mode with hot reload
CMD ["air", "-c", ".air.toml"]

EXPOSE 8080
```

### Docker Compose with Application

```yaml
# docker-compose.dev.yml
version: "3.8"

services:
  # ... (all services from previous example)

  # Your Application
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: saas-api
    ports:
      - "8080:8080"
      - "5005:5005" # Debug port for Java
    volumes:
      - ./src:/app/src # Hot reload
      - ./node_modules:/app/node_modules # For Node.js
    environment:
      - DATABASE_URL=postgresql://dev_user:dev_password@postgres:5432/saas_db
      - REDIS_URL=redis://redis:6379
      - RABBITMQ_URL=amqp://dev_user:dev_password@rabbitmq:5672
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - SMTP_HOST=mailhog
      - SMTP_PORT=1025
      - S3_ENDPOINT=http://minio:9000
      - S3_ACCESS_KEY=minioadmin
      - S3_SECRET_KEY=minioadmin
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
```

---

## 🛠️ Essential Development Tools

### Terminal/Shell Setup

#### Windows: Git Bash or WSL2

**Git Bash**: Comes with Git for Windows
**WSL2** (Recommended):

```powershell
# Install WSL2
wsl --install

# Install Ubuntu
wsl --install -d Ubuntu

# Set as default
wsl --set-default Ubuntu
```

#### Mac/Linux: Oh My Zsh

```bash
# Install Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install useful plugins
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# Edit ~/.zshrc
plugins=(git docker docker-compose node npm golang zsh-autosuggestions zsh-syntax-highlighting)
```

### Version Managers

#### Node.js: nvm

```bash
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Install Node versions
nvm install 20
nvm install 18
nvm use 20

# Set default
nvm alias default 20
```

#### Java: SDKMAN

```bash
# Install SDKMAN
curl -s "https://get.sdkman.io" | bash

# Install Java versions
sdk install java 17.0.9-tem
sdk install java 21.0.1-tem

# Switch versions
sdk use java 17.0.9-tem

# Set default
sdk default java 17.0.9-tem
```

#### Go: gvm (Optional)

```bash
# Install gvm
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)

# Install Go versions
gvm install go1.21
gvm use go1.21 --default
```

### API Testing Tools

#### Postman

**Download**: https://www.postman.com/downloads/

**Alternative - Bruno** (Open Source):
**Download**: https://www.usebruno.com/

#### HTTPie (CLI)

```bash
# Install
pip install httpie

# or
brew install httpie  # Mac
apt install httpie   # Ubuntu

# Usage
http GET localhost:8080/api/users
http POST localhost:8080/api/users name="John" email="john@example.com"
```

### Database GUI Tools

#### DBeaver (Universal)

**Download**: https://dbeaver.io/download/

**Features**:

- Supports PostgreSQL, MySQL, MongoDB, etc.
- SQL editor with autocomplete
- ER diagrams
- Data export/import

#### TablePlus (Mac/Windows)

**Download**: https://tableplus.com/

**Features**:

- Clean, native interface
- Multiple database support
- Query history
- SSH tunneling

### Git GUI Tools

#### GitKraken

**Download**: https://www.gitkraken.com/

#### SourceTree (Free)

**Download**: https://www.sourcetreeapp.com/

---

## 🔧 Code Quality Tools

### Linting & Formatting

#### Java: Checkstyle + SpotBugs

**Maven Configuration** (pom.xml):

```xml
<build>
  <plugins>
    <!-- Checkstyle -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-checkstyle-plugin</artifactId>
      <version>3.3.1</version>
      <configuration>
        <configLocation>google_checks.xml</configLocation>
        <consoleOutput>true</consoleOutput>
        <failsOnError>true</failsOnError>
      </configuration>
      <executions>
        <execution>
          <phase>validate</phase>
          <goals>
            <goal>check</goal>
          </goals>
        </execution>
      </executions>
    </plugin>

    <!-- SpotBugs -->
    <plugin>
      <groupId>com.github.spotbugs</groupId>
      <artifactId>spotbugs-maven-plugin</artifactId>
      <version>4.8.2.0</version>
      <executions>
        <execution>
          <phase>verify</phase>
          <goals>
            <goal>check</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

#### Node.js: ESLint + Prettier

**Install**:

```bash
npm install --save-dev eslint prettier eslint-config-prettier eslint-plugin-prettier @typescript-eslint/eslint-plugin @typescript-eslint/parser
```

**.eslintrc.json**:

```json
{
  "parser": "@typescript-eslint/parser",
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ],
  "plugins": ["@typescript-eslint", "prettier"],
  "rules": {
    "prettier/prettier": "error",
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/explicit-function-return-type": "off"
  }
}
```

**.prettierrc**:

```json
{
  "singleQuote": true,
  "trailingComma": "es5",
  "semi": true,
  "tabWidth": 2,
  "printWidth": 100
}
```

**package.json** scripts:

```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,json}\""
  }
}
```

#### Go: golangci-lint

**Install**:

```bash
# Binary
curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin

# or via go install
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

**.golangci.yml**:

```yaml
linters:
  enable:
    - gofmt
    - goimports
    - govet
    - errcheck
    - staticcheck
    - unused
    - gosimple
    - structcheck
    - varcheck
    - ineffassign
    - deadcode
    - typecheck

linters-settings:
  gofmt:
    simplify: true
  goimports:
    local-prefixes: github.com/mycompany/myapp

issues:
  exclude-use-default: false
  max-issues-per-linter: 0
  max-same-issues: 0

run:
  timeout: 5m
```

**Usage**:

```bash
golangci-lint run
golangci-lint run --fix
```

---

## 🪝 Git Hooks Setup

### Using Husky (Node.js Projects)

**Install**:

```bash
npm install --save-dev husky lint-staged
npx husky install
```

**package.json**:

```json
{
  "scripts": {
    "prepare": "husky install"
  },
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

**Create pre-commit hook**:

```bash
npx husky add .husky/pre-commit "npx lint-staged"
```

**Create pre-push hook**:

```bash
npx husky add .husky/pre-push "npm test"
```

### Using pre-commit (Python-based, works for all languages)

**Install**:

```bash
pip install pre-commit
```

**.pre-commit-config.yaml** (for Node.js):

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files

  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.56.0
    hooks:
      - id: eslint
        files: \.[jt]sx?$
        types: [file]
        additional_dependencies:
          - eslint@8.56.0
          - "@typescript-eslint/eslint-plugin@6.19.0"
          - "@typescript-eslint/parser@6.19.0"

  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.1.0
    hooks:
      - id: prettier
```

**For Go**:

```yaml
repos:
  - repo: https://github.com/golangci/golangci-lint
    rev: v1.55.2
    hooks:
      - id: golangci-lint

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
```

**Install hooks**:

```bash
pre-commit install
pre-commit install --hook-type pre-push
```

---

## 📦 Package/Dependency Management Tools

### Renovate Bot (Automated Dependency Updates)

**Configuration** (.github/renovate.json):

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "schedule": ["before 3am on Monday"],
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true
    }
  ]
}
```

### Dependabot (GitHub)

**Configuration** (.github/dependabot.yml):

```yaml
version: 2
updates:
  # Node.js dependencies
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10

  # Docker dependencies
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## ✅ Development Environment Checklist

### Initial Setup

- [ ] Install IDE (IntelliJ IDEA / VS Code / GoLand)
- [ ] Install language runtime (JDK / Node.js / Go)
- [ ] Install Docker Desktop
- [ ] Install Git
- [ ] Configure IDE with essential plugins
- [ ] Set up code style and formatting rules
- [ ] Install version manager (SDKMAN / nvm / gvm)

### Project Setup

- [ ] Clone repository
- [ ] Install dependencies
- [ ] Copy .env.example to .env
- [ ] Start Docker services (docker-compose up)
- [ ] Run database migrations
- [ ] Seed test data
- [ ] Run tests to verify setup
- [ ] Start development server
- [ ] Verify application works

### Code Quality Setup

- [ ] Configure linter (ESLint / Checkstyle / golangci-lint)
- [ ] Configure formatter (Prettier / gofmt)
- [ ] Set up Git hooks (Husky / pre-commit)
- [ ] Configure CI/CD pipeline
- [ ] Enable automated dependency updates

---

## 🚀 Quick Start Scripts

### Makefile (Works for all platforms with make installed)

```makefile
.PHONY: help install dev test lint format docker-up docker-down clean

help:  ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $1, $2}'

install:  ## Install dependencies
	@echo "Installing dependencies..."
	@if [ -f "package.json" ]; then npm install; fi
	@if [ -f "go.mod" ]; then go mod download; fi
	@if [ -f "pom.xml" ]; then mvn install -DskipTests; fi

dev:  ## Start development server
	@echo "Starting development server..."
	docker-compose up -d
	@if [ -f "package.json" ]; then npm run dev; fi
	@if [ -f "go.mod" ]; then go run cmd/api/main.go; fi
	@if [ -f "pom.xml" ]; then mvn spring-boot:run; fi

test:  ## Run tests
	@echo "Running tests..."
	@if [ -f "package.json" ]; then npm test; fi
	@if [ -f "go.mod" ]; then go test -v ./...; fi
	@if [ -f "pom.xml" ]; then mvn test; fi

lint:  ## Run linter
	@echo "Running linter..."
	@if [ -f "package.json" ]; then npm run lint; fi
	@if [ -f "go.mod" ]; then golangci-lint run; fi
	@if [ -f "pom.xml" ]; then mvn checkstyle:check; fi

format:  ## Format code
	@echo "Formatting code..."
	@if [ -f "package.json" ]; then npm run format; fi
	@if [ -f "go.mod" ]; then gofmt -w .; fi

docker-up:  ## Start Docker services
	docker-compose up -d
	@echo "Services started. Waiting for health checks..."
	@sleep 5
	docker-compose ps

docker-down:  ## Stop Docker services
	docker-compose down

docker-logs:  ## View Docker logs
	docker-compose logs -f

clean:  ## Clean build artifacts
	@echo "Cleaning..."
	@if [ -d "node_modules" ]; then rm -rf node_modules; fi
	@if [ -d "dist" ]; then rm -rf dist; fi
	@if [ -d "build" ]; then rm -rf build; fi
	@if [ -d "target" ]; then rm -rf target; fi

setup:  ## Complete development setup
	@echo "Setting up development environment..."
	make install
	make docker-up
	@echo "✅ Setup complete! Run 'make dev' to start developing."
```

**Usage:**

```bash
make help       # Show all commands
make setup      # Complete setup
make dev        # Start development
make test       # Run tests
make lint       # Check code quality
```

---

## 🎨 Productivity Tips & Shortcuts

### General Productivity

**1. Multiple Cursors (VS Code)**

```
Ctrl+Alt+↑/↓     - Add cursor above/below
Ctrl+D           - Select next occurrence
Ctrl+Shift+L     - Select all occurrences
Alt+Click        - Add cursor
```

**2. Code Navigation (IntelliJ)**

```
Ctrl+N           - Go to class
Ctrl+Shift+N     - Go to file
Ctrl+Alt+Shift+N - Go to symbol
Ctrl+E           - Recent files
Ctrl+Shift+E     - Recent locations
```

**3. Quick Fixes**

```
Alt+Enter        - Show quick fixes (IntelliJ)
Ctrl+.           - Show quick fixes (VS Code)
```

### Code Snippets

**IntelliJ Live Templates:**

```java
// Type "logger" + Tab
private static final Logger logger = LoggerFactory.getLogger(ClassName.class);

// Type "testm" + Tab
@Test
public void testMethodName() {
    // Test implementation
}
```

**VS Code Snippets:**

Create `.vscode/snippets.code-snippets`:

```json
{
  "Console Log": {
    "prefix": "clg",
    "body": ["console.log('$1:', $1);"]
  },
  "Test Case": {
    "prefix": "test",
    "body": ["it('should $1', async () => {", "  $2", "});"]
  },
  "React Component": {
    "prefix": "rfc",
    "body": [
      "import React from 'react';",
      "",
      "interface ${1:Component}Props {",
      "  $2",
      "}",
      "",
      "export const ${1:Component}: React.FC<${1:Component}Props> = ({ $3 }) => {",
      "  return (",
      "    <div>",
      "      $4",
      "    </div>",
      "  );",
      "};"
    ]
  }
}
```

---

## 🔐 Environment Variables Management

### .env File Structure

```bash
# .env.example (commit this)
# Copy to .env and fill in values

# Application
NODE_ENV=development
PORT=3000
APP_NAME=my-saas-app
APP_URL=http://localhost:3000

# Database
DATABASE_URL=postgresql://dev_user:dev_password@localhost:5432/saas_db
DB_HOST=localhost
DB_PORT=5432
DB_NAME=saas_db
DB_USER=dev_user
DB_PASSWORD=dev_password

# Redis
REDIS_URL=redis://localhost:6379
REDIS_HOST=localhost
REDIS_PORT=6379

# JWT
JWT_SECRET=your-super-secret-jwt-key-change-this
JWT_EXPIRES_IN=7d

# Email
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USER=
SMTP_PASSWORD=
FROM_EMAIL=noreply@example.com

# AWS/S3
AWS_ACCESS_KEY_ID=minioadmin
AWS_SECRET_ACCESS_KEY=minioadmin
AWS_REGION=us-east-1
S3_BUCKET=my-bucket
S3_ENDPOINT=http://localhost:9000

# External APIs
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
```

### dotenv-vault (For team secret management)

```bash
# Install
npm install dotenv-vault-core

# Initialize
npx dotenv-vault new

# Push to vault
npx dotenv-vault push

# Pull from vault
npx dotenv-vault pull
```

### direnv (Automatic environment loading)

```bash
# Install
# Mac: brew install direnv
# Ubuntu: apt install direnv

# Add to shell (.bashrc or .zshrc)
eval "$(direnv hook bash)"  # or zsh

# Create .envrc
echo "dotenv" > .envrc

# Allow direnv to load it
direnv allow

# Now .env is automatically loaded when you cd into the directory
```

---

## 📊 Monitoring & Debugging Tools

### Browser DevTools Extensions

**React Developer Tools**

- Chrome: https://chrome.google.com/webstore (search: React Developer Tools)
- Firefox: https://addons.mozilla.org/firefox (search: React DevTools)

**Redux DevTools**

- Chrome/Firefox: Search for Redux DevTools in respective stores

**Vue.js DevTools**

- Chrome/Firefox: Search for Vue.js DevTools

### API Debugging

**Charles Proxy**

- Intercept HTTP/HTTPS traffic
- Modify requests/responses
- https://www.charlesproxy.com/

**Wireshark** (Advanced)

- Network protocol analyzer
- https://www.wireshark.org/

### Log Aggregation (Local)

**Docker Compose with Loki + Grafana:**

```yaml
# docker-compose.logging.yml
version: "3.8"

services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - loki_data:/loki

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /var/log:/var/log
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  grafana:
    image: grafana/grafana:10.2.0
    ports:
      - "3001:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  loki_data:
  grafana_data:
```

---

## 🧪 Testing Tools Setup

### Node.js Testing

```bash
# Install Jest
npm install --save-dev jest @types/jest ts-jest

# Install testing utilities
npm install --save-dev @testing-library/react @testing-library/jest-dom @testing-library/user-event

# Install E2E testing
npm install --save-dev @playwright/test
```

**jest.config.js:**

```javascript
module.exports = {
  preset: "ts-jest",
  testEnvironment: "node",
  roots: ["<rootDir>/src"],
  testMatch: ["**/__tests__/**/*.ts", "**/?(*.)+(spec|test).ts"],
  coverageDirectory: "coverage",
  coverageReporters: ["text", "lcov", "html"],
  collectCoverageFrom: [
    "src/**/*.ts",
    "!src/**/*.d.ts",
    "!src/**/*.interface.ts",
  ],
};
```

### Java Testing

**Maven dependencies:**

```xml
<dependencies>
  <!-- JUnit 5 -->
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
  </dependency>

  <!-- Mockito -->
  <dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <scope>test</scope>
  </dependency>

  <!-- TestContainers (for integration tests) -->
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

### Go Testing

```bash
# Install test tools
go install github.com/onsi/ginkgo/v2/ginkgo@latest
go install github.com/onsi/gomega@latest

# Install testify
go get github.com/stretchr/testify

# Install test coverage tools
go install github.com/axw/gocov/gocov@latest
go install github.com/matm/gocov-html@latest
```

---

## 🎯 IDE Performance Optimization

### IntelliJ IDEA Performance

**Increase Memory:**

Help → Edit Custom VM Options:

```
-Xms2048m
-Xmx4096m
-XX:ReservedCodeCacheSize=512m
```

**Exclude Directories:**

```
Settings → Project Structure → Modules → Select Module → Exclude
- node_modules
- target
- build
- dist
- .next
```

**Disable Unused Plugins:**

```
Settings → Plugins → Installed
Disable plugins you don't use
```

### VS Code Performance

**settings.json:**

```json
{
  // Exclude files from search
  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.git": true,
    "**/coverage": true
  },

  // Exclude files from file watcher
  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/dist/**": true
  },

  // Limit number of files to watch
  "files.watcherInclude": ["src/**", "package.json"],

  // Disable telemetry
  "telemetry.telemetryLevel": "off"
}
```

---

## 📝 Documentation Tools

### API Documentation

**Swagger/OpenAPI:**

```bash
# Node.js
npm install --save @nestjs/swagger swagger-ui-express

# Java (Spring Boot)
# Add to pom.xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>

# Go
go get github.com/swaggo/swag/cmd/swag
go get github.com/swaggo/gin-swagger
go get github.com/swaggo/files
```

### Code Documentation

**JSDoc (JavaScript/TypeScript):**

```bash
npm install --save-dev jsdoc

# Generate docs
npx jsdoc -c jsdoc.json
```

**JavaDoc (Java):**

```bash
# Generate with Maven
mvn javadoc:javadoc

# View at target/site/apidocs/index.html
```

**godoc (Go):**

```bash
# Install
go install golang.org/x/tools/cmd/godoc@latest

# Start server
godoc -http=:6060

# Visit http://localhost:6060/pkg/your-package/
```

---

## 🔄 Continuous Integration Setup

### GitHub Actions Example

**.github/workflows/ci.yml:**

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests
        run: npm test
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
```

---

## ✅ Final Development Environment Checklist

### Day 1: Core Setup

- [ ] Install IDE with essential plugins
- [ ] Install language runtime and version manager
- [ ] Install Docker Desktop
- [ ] Install Git and configure user
- [ ] Clone project repository
- [ ] Set up SSH keys for Git

### Day 2: Project Configuration

- [ ] Copy and configure .env file
- [ ] Install project dependencies
- [ ] Start Docker services
- [ ] Run database migrations
- [ ] Verify application starts successfully
- [ ] Run test suite to ensure everything works

### Day 3: Developer Tools

- [ ] Install API testing tool (Postman/Bruno)
- [ ] Install database GUI (DBeaver/TablePlus)
- [ ] Configure linter and formatter
- [ ] Set up Git hooks
- [ ] Install terminal enhancements (Oh My Zsh)

### Day 4: Productivity Setup

- [ ] Configure IDE shortcuts
- [ ] Create code snippets
- [ ] Set up workspace layouts
- [ ] Configure debugging
- [ ] Bookmark important URLs (docs, CI/CD, etc.)

### Week 1: Final Polish

- [ ] Document your setup process
- [ ] Create troubleshooting guide
- [ ] Share setup with team
- [ ] Optimize IDE performance
- [ ] Configure automated backups

---

## 🆘 Common Issues & Solutions

### Issue: Port Already in Use

```bash
# Find process using port
lsof -i :8080        # Mac/Linux
netstat -ano | findstr :8080  # Windows

# Kill process
kill -9 <PID>        # Mac/Linux
taskkill /PID <PID> /F  # Windows
```

### Issue: Docker Services Won't Start

```bash
# Check Docker is running
docker info

# View detailed logs
docker-compose logs -f <service-name>

# Remove and recreate containers
docker-compose down -v
docker-compose up -d --force-recreate
```

### Issue: Cannot Connect to Database

```bash
# Check if service is running
docker-compose ps

# Check connection from inside container
docker-compose exec postgres psql -U dev_user -d saas_db

# Verify connection string
echo $DATABASE_URL
```

### Issue: IDE is Slow

1. Increase memory allocation
2. Exclude build directories from indexing
3. Disable unused plugins
4. Clear cache and restart
5. Use SSD for project files

---

## 🎓 Learning Resources

### Documentation

- **IntelliJ IDEA**: https://www.jetbrains.com/idea/documentation/
- **VS Code**: https://code.visualstudio.com/docs
- **Docker**: https://docs.docker.com/
- **Docker Compose**: https://docs.docker.com/compose/

### Tutorials

- **IntelliJ Tips**: https://www.jetbrains.com/idea/guide/
- **VS Code Tips**: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
- **Docker Tutorial**: https://docker-curriculum.com/

### Communities

- **Stack Overflow**: https://stackoverflow.com/
- **Dev.to**: https://dev.to/
- **Reddit**: r/java, r/node, r/golang

---

## 🎉 Summary

You now have a complete development environment setup with:

✅ **Optimized IDEs** with essential plugins and configurations
✅ **Docker** for consistent local services
✅ **Code quality tools** (linting, formatting, testing)
✅ **Git hooks** for automated checks
✅ **Productivity tools** and shortcuts
✅ **Monitoring and debugging** tools
✅ **Reproducible setup** for the entire team
