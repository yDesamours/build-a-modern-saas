# Module 3.4: Environment Configuration

**Duration:** 4 days  
**Part of:** Phase 1 - Foundation & Architecture

---

## 📚 Learning Objectives

By the end of this module, you will:

- Understand environment-specific configuration strategies
- Implement secure secret management across Java, Node.js, and Go
- Use feature flags for controlled rollouts
- Configure applications for dev, staging, and production environments
- Handle sensitive data securely without hardcoding
- Implement configuration validation and type safety

---

## 🎯 Why Environment Configuration Matters

**The Problem:**

```java
// ❌ BAD: Hardcoded configuration
public class DatabaseConfig {
    private static final String DB_URL = "jdbc:postgresql://localhost:5432/mydb";
    private static final String DB_USER = "admin";
    private static final String DB_PASSWORD = "password123"; // Leaked on GitHub!
}
```

**Real-world Scenario:**
Your SaaS needs to run in:

- **Development:** Local database, debug logging, mock payment gateway
- **Staging:** Production-like setup, real payment sandbox, restricted access
- **Production:** High availability database, minimal logging, real payments, strict security

**Key Requirements:**

- Different database connections per environment
- API keys that vary (Stripe test vs live keys)
- Feature flags to enable/disable features
- Secrets that NEVER appear in code
- Easy configuration updates without redeployment

---

## 📖 Day 1: Configuration Management Fundamentals

### 3.4.1 The 12-Factor App Configuration Principle

**Principle #3:** Store config in the environment

> "An app's config is everything that is likely to vary between deploys (staging, production, developer environments, etc)."

**What belongs in configuration:**

- ✅ Database URLs and credentials
- ✅ API keys and secrets
- ✅ External service endpoints
- ✅ Feature flags
- ✅ Resource limits (connection pools, timeouts)
- ✅ Environment-specific behavior

**What does NOT belong:**

- ❌ Business logic
- ❌ Internal constants that never change
- ❌ Application code

**Source:** https://12factor.net/config

---

### 3.4.2 Configuration Hierarchy

**Principle:** Configurations should override in a predictable order

```
Default Values (in code)
    ↓
Configuration Files (application.yml, config.json)
    ↓
Environment Variables (DATABASE_URL)
    ↓
Command Line Arguments (--port=8080)
    ↓
External Config Service (Vault, AWS Secrets Manager)
```

**Most specific wins!**

---

### 3.4.3 Environment Variables vs Configuration Files

| Aspect             | Environment Variables          | Configuration Files           |
| ------------------ | ------------------------------ | ----------------------------- |
| **Secrets**        | ✅ Excellent (never committed) | ❌ Risk of committing         |
| **Complex Config** | ❌ Limited (strings only)      | ✅ Supports nested structures |
| **Portability**    | ✅ Platform-agnostic           | ⚠️ File path issues           |
| **Readability**    | ❌ Scattered across system     | ✅ Centralized                |
| **Best For**       | Secrets, environment-specific  | Defaults, structured config   |

**Best Practice:** Use both!

- Configuration files for structure and defaults
- Environment variables for secrets and environment-specific overrides

---

## 📖 Day 2: Implementation Across Tech Stacks

### 3.4.4 Java/Spring Boot Configuration

#### Configuration Files Structure

```
src/main/resources/
├── application.yml                 # Default configuration
├── application-dev.yml            # Development overrides
├── application-staging.yml        # Staging overrides
└── application-prod.yml           # Production overrides
```

#### application.yml (Default)

```yaml
# Application Configuration
spring:
  application:
    name: saas-backend

  # Database Configuration
  datasource:
    url: jdbc:postgresql://localhost:5432/saas_dev
    username: postgres
    password: ${DB_PASSWORD:defaultpass} # Environment variable with fallback
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000

  # JPA/Hibernate
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect

  # Redis Configuration
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    password: ${REDIS_PASSWORD:}

# Application Properties
app:
  name: My SaaS Platform
  version: @project.version@

  # JWT Configuration
  jwt:
    secret: ${JWT_SECRET:default-secret-change-in-production}
    expiration: 86400000 # 24 hours in milliseconds

  # External Services
  stripe:
    api-key: ${STRIPE_API_KEY}
    webhook-secret: ${STRIPE_WEBHOOK_SECRET}

  # File Upload
  upload:
    max-file-size: 10MB
    allowed-types: jpg,png,pdf
    storage-path: ${UPLOAD_PATH:/tmp/uploads}

# Logging
logging:
  level:
    root: INFO
    com.mycompany.saas: DEBUG
  file:
    name: logs/application.log
    max-size: 10MB
    max-history: 30

# Server Configuration
server:
  port: ${SERVER_PORT:8080}
  compression:
    enabled: true
  error:
    include-message: always
    include-stacktrace: on_param
```

#### application-prod.yml (Production Overrides)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10

  jpa:
    show-sql: false

logging:
  level:
    root: WARN
    com.mycompany.saas: INFO
  file:
    name: /var/log/saas/application.log

server:
  error:
    include-stacktrace: never
```

#### Type-Safe Configuration Properties

```java
package com.mycompany.saas.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.validation.annotation.Validated;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;

@Configuration
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {

    @NotBlank
    private String name;

    @NotBlank
    private String version;

    @NotNull
    private JwtProperties jwt;

    @NotNull
    private StripeProperties stripe;

    @NotNull
    private UploadProperties upload;

    // Getters and setters
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public JwtProperties getJwt() {
        return jwt;
    }

    public void setJwt(JwtProperties jwt) {
        this.jwt = jwt;
    }

    // Nested configuration classes
    public static class JwtProperties {
        @NotBlank
        private String secret;

        private long expiration = 86400000;

        public String getSecret() {
            return secret;
        }

        public void setSecret(String secret) {
            this.secret = secret;
        }

        public long getExpiration() {
            return expiration;
        }

        public void setExpiration(long expiration) {
            this.expiration = expiration;
        }
    }

    public static class StripeProperties {
        @NotBlank
        private String apiKey;

        @NotBlank
        private String webhookSecret;

        // Getters and setters
        public String getApiKey() {
            return apiKey;
        }

        public void setApiKey(String apiKey) {
            this.apiKey = apiKey;
        }

        public String getWebhookSecret() {
            return webhookSecret;
        }

        public void setWebhookSecret(String webhookSecret) {
            this.webhookSecret = webhookSecret;
        }
    }

    public static class UploadProperties {
        private String maxFileSize = "10MB";
        private String[] allowedTypes;
        private String storagePath = "/tmp/uploads";

        // Getters and setters
        public String getMaxFileSize() {
            return maxFileSize;
        }

        public void setMaxFileSize(String maxFileSize) {
            this.maxFileSize = maxFileSize;
        }

        public String[] getAllowedTypes() {
            return allowedTypes;
        }

        public void setAllowedTypes(String[] allowedTypes) {
            this.allowedTypes = allowedTypes;
        }

        public String getStoragePath() {
            return storagePath;
        }

        public void setStoragePath(String storagePath) {
            this.storagePath = storagePath;
        }
    }
}
```

#### Using Configuration in Services

```java
package com.mycompany.saas.service;

import com.mycompany.saas.config.AppProperties;
import org.springframework.stereotype.Service;

@Service
public class PaymentService {

    private final AppProperties appProperties;

    public PaymentService(AppProperties appProperties) {
        this.appProperties = appProperties;
    }

    public void processPayment(double amount) {
        String stripeKey = appProperties.getStripe().getApiKey();

        // Use Stripe API with the configured key
        // Stripe.apiKey = stripeKey;

        System.out.println("Processing payment with Stripe");
    }
}
```

#### Running with Different Profiles

```bash
# Development
java -jar app.jar --spring.profiles.active=dev

# Staging
java -jar app.jar --spring.profiles.active=staging

# Production
java -jar app.jar --spring.profiles.active=prod

# Multiple profiles
java -jar app.jar --spring.profiles.active=prod,monitoring
```

---

### 3.4.5 Node.js Configuration

#### Configuration Structure

```
project-root/
├── config/
│   ├── default.json           # Default configuration
│   ├── development.json       # Development overrides
│   ├── staging.json           # Staging overrides
│   ├── production.json        # Production overrides
│   └── custom-environment-variables.json  # Env var mapping
├── .env.example               # Template for .env file
└── .env                       # Local environment variables (gitignored)
```

#### Using `dotenv` and `config` packages

```bash
npm install dotenv config
npm install --save-dev @types/config  # For TypeScript
```

#### .env.example (Template)

```bash
# Database Configuration
DATABASE_URL=postgresql://user:password@localhost:5432/saas_dev
DATABASE_POOL_MIN=5
DATABASE_POOL_MAX=20

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# JWT
JWT_SECRET=your-secret-key-here
JWT_EXPIRATION=24h

# Stripe
STRIPE_API_KEY=sk_test_xxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxx

# AWS S3
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
S3_BUCKET_NAME=my-saas-uploads

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=
SMTP_PASSWORD=

# Application
NODE_ENV=development
PORT=3000
LOG_LEVEL=debug
```

#### config/default.json

```json
{
  "app": {
    "name": "My SaaS Platform",
    "version": "1.0.0",
    "port": 3000
  },
  "database": {
    "url": "postgresql://localhost:5432/saas_dev",
    "pool": {
      "min": 5,
      "max": 20
    },
    "ssl": false
  },
  "redis": {
    "host": "localhost",
    "port": 6379,
    "password": null,
    "db": 0
  },
  "jwt": {
    "secret": "default-secret-change-me",
    "expiration": "24h",
    "refreshExpiration": "7d"
  },
  "stripe": {
    "apiKey": "",
    "webhookSecret": ""
  },
  "upload": {
    "maxFileSize": 10485760,
    "allowedTypes": ["image/jpeg", "image/png", "application/pdf"],
    "storagePath": "/tmp/uploads"
  },
  "logging": {
    "level": "info",
    "format": "json"
  },
  "rateLimit": {
    "windowMs": 900000,
    "max": 100
  }
}
```

#### config/production.json

```json
{
  "database": {
    "pool": {
      "min": 10,
      "max": 50
    },
    "ssl": true
  },
  "logging": {
    "level": "warn",
    "format": "json"
  },
  "rateLimit": {
    "max": 50
  }
}
```

#### config/custom-environment-variables.json

```json
{
  "app": {
    "port": "PORT"
  },
  "database": {
    "url": "DATABASE_URL",
    "pool": {
      "min": "DATABASE_POOL_MIN",
      "max": "DATABASE_POOL_MAX"
    }
  },
  "redis": {
    "host": "REDIS_HOST",
    "port": "REDIS_PORT",
    "password": "REDIS_PASSWORD"
  },
  "jwt": {
    "secret": "JWT_SECRET",
    "expiration": "JWT_EXPIRATION"
  },
  "stripe": {
    "apiKey": "STRIPE_API_KEY",
    "webhookSecret": "STRIPE_WEBHOOK_SECRET"
  }
}
```

#### TypeScript Configuration Module

```typescript
// src/config/index.ts
import dotenv from "dotenv";
import config from "config";

// Load .env file
dotenv.config();

// Type-safe configuration interface
interface AppConfig {
  app: {
    name: string;
    version: string;
    port: number;
  };
  database: {
    url: string;
    pool: {
      min: number;
      max: number;
    };
    ssl: boolean;
  };
  redis: {
    host: string;
    port: number;
    password: string | null;
    db: number;
  };
  jwt: {
    secret: string;
    expiration: string;
    refreshExpiration: string;
  };
  stripe: {
    apiKey: string;
    webhookSecret: string;
  };
  upload: {
    maxFileSize: number;
    allowedTypes: string[];
    storagePath: string;
  };
  logging: {
    level: string;
    format: string;
  };
  rateLimit: {
    windowMs: number;
    max: number;
  };
}

// Validate required environment variables
function validateConfig(): void {
  const required = [
    "JWT_SECRET",
    "DATABASE_URL",
    "STRIPE_API_KEY",
    "STRIPE_WEBHOOK_SECRET",
  ];

  const missing = required.filter((key) => !process.env[key]);

  if (missing.length > 0 && process.env.NODE_ENV === "production") {
    throw new Error(
      `Missing required environment variables: ${missing.join(", ")}`
    );
  }
}

// Validate on startup
validateConfig();

// Export typed configuration
export const appConfig: AppConfig = {
  app: config.get("app"),
  database: config.get("database"),
  redis: config.get("redis"),
  jwt: config.get("jwt"),
  stripe: config.get("stripe"),
  upload: config.get("upload"),
  logging: config.get("logging"),
  rateLimit: config.get("rateLimit"),
};

// Helper to check environment
export const isDevelopment = process.env.NODE_ENV === "development";
export const isProduction = process.env.NODE_ENV === "production";
export const isTest = process.env.NODE_ENV === "test";
```

#### Using Configuration in Services

```typescript
// src/services/payment.service.ts
import { appConfig } from "../config";
import Stripe from "stripe";

export class PaymentService {
  private stripe: Stripe;

  constructor() {
    this.stripe = new Stripe(appConfig.stripe.apiKey, {
      apiVersion: "2023-10-16",
    });
  }

  async processPayment(amount: number, currency: string = "usd") {
    const paymentIntent = await this.stripe.paymentIntents.create({
      amount,
      currency,
    });

    return paymentIntent;
  }

  verifyWebhook(payload: string, signature: string): boolean {
    try {
      this.stripe.webhooks.constructEvent(
        payload,
        signature,
        appConfig.stripe.webhookSecret
      );
      return true;
    } catch (err) {
      return false;
    }
  }
}
```

#### Running with Different Environments

```bash
# Development (default)
npm run dev

# Production
NODE_ENV=production npm start

# Staging
NODE_ENV=staging npm start

# With custom port
PORT=4000 npm start
```

---

### 3.4.6 Go Configuration

#### Configuration Structure

```
project-root/
├── config/
│   ├── config.go              # Configuration struct and loader
│   ├── dev.yaml               # Development config
│   ├── staging.yaml           # Staging config
│   └── prod.yaml              # Production config
├── .env.example
└── .env
```

#### Using `viper` for Configuration

```bash
go get github.com/spf13/viper
go get github.com/joho/godotenv
```

#### config/config.go

```go
package config

import (
	"fmt"
	"log"
	"os"
	"time"

	"github.com/joho/godotenv"
	"github.com/spf13/viper"
)

// Config holds all configuration for the application
type Config struct {
	App      AppConfig
	Database DatabaseConfig
	Redis    RedisConfig
	JWT      JWTConfig
	Stripe   StripeConfig
	Upload   UploadConfig
	Logging  LoggingConfig
}

type AppConfig struct {
	Name        string
	Version     string
	Port        int
	Environment string
}

type DatabaseConfig struct {
	URL         string
	MaxOpenConn int
	MaxIdleConn int
	MaxLifetime time.Duration
	SSL         bool
}

type RedisConfig struct {
	Host     string
	Port     int
	Password string
	DB       int
}

type JWTConfig struct {
	Secret           string
	Expiration       time.Duration
	RefreshExpiration time.Duration
}

type StripeConfig struct {
	APIKey        string
	WebhookSecret string
}

type UploadConfig struct {
	MaxFileSize   int64
	AllowedTypes  []string
	StoragePath   string
}

type LoggingConfig struct {
	Level  string
	Format string
}

// Load reads configuration from file and environment variables
func Load() (*Config, error) {
	// Load .env file if exists
	if err := godotenv.Load(); err != nil {
		log.Println("No .env file found")
	}

	// Set default config file name
	env := os.Getenv("APP_ENV")
	if env == "" {
		env = "dev"
	}

	// Configure viper
	viper.SetConfigName(env)
	viper.SetConfigType("yaml")
	viper.AddConfigPath("./config")
	viper.AddConfigPath(".")

	// Enable environment variable override
	viper.AutomaticEnv()

	// Set defaults
	setDefaults()

	// Read config file
	if err := viper.ReadInConfig(); err != nil {
		return nil, fmt.Errorf("failed to read config file: %w", err)
	}

	// Unmarshal into struct
	var config Config
	if err := viper.Unmarshal(&config); err != nil {
		return nil, fmt.Errorf("failed to unmarshal config: %w", err)
	}

	// Validate required fields
	if err := validate(&config); err != nil {
		return nil, err
	}

	return &config, nil
}

func setDefaults() {
	viper.SetDefault("app.name", "My SaaS Platform")
	viper.SetDefault("app.port", 8080)
	viper.SetDefault("app.environment", "development")

	viper.SetDefault("database.maxopenconn", 25)
	viper.SetDefault("database.maxidleconn", 5)
	viper.SetDefault("database.maxlifetime", "5m")
	viper.SetDefault("database.ssl", false)

	viper.SetDefault("redis.port", 6379)
	viper.SetDefault("redis.db", 0)

	viper.SetDefault("jwt.expiration", "24h")
	viper.SetDefault("jwt.refreshexpiration", "168h")

	viper.SetDefault("upload.maxfilesize", 10485760) // 10MB
	viper.SetDefault("upload.storagepath", "/tmp/uploads")

	viper.SetDefault("logging.level", "info")
	viper.SetDefault("logging.format", "json")
}

func validate(config *Config) error {
	required := map[string]string{
		"JWT_SECRET":           config.JWT.Secret,
		"DATABASE_URL":         config.Database.URL,
		"STRIPE_API_KEY":       config.Stripe.APIKey,
		"STRIPE_WEBHOOK_SECRET": config.Stripe.WebhookSecret,
	}

	if config.App.Environment == "production" {
		for key, value := range required {
			if value == "" {
				return fmt.Errorf("required environment variable %s is not set", key)
			}
		}
	}

	return nil
}

// Helper functions
func (c *Config) IsDevelopment() bool {
	return c.App.Environment == "development"
}

func (c *Config) IsProduction() bool {
	return c.App.Environment == "production"
}

func (c *Config) IsStaging() bool {
	return c.App.Environment == "staging"
}
```

#### config/dev.yaml

```yaml
app:
  name: "My SaaS Platform"
  version: "1.0.0"
  port: 8080
  environment: "development"

database:
  url: "${DATABASE_URL:postgresql://localhost:5432/saas_dev?sslmode=disable}"
  maxopenconn: 25
  maxidleconn: 5
  maxlifetime: "5m"
  ssl: false

redis:
  host: "${REDIS_HOST:localhost}"
  port: 6379
  password: "${REDIS_PASSWORD:}"
  db: 0

jwt:
  secret: "${JWT_SECRET:default-secret-change-me}"
  expiration: "24h"
  refreshexpiration: "168h"

stripe:
  apikey: "${STRIPE_API_KEY}"
  webhooksecret: "${STRIPE_WEBHOOK_SECRET}"

upload:
  maxfilesize: 10485760
  allowedtypes:
    - "image/jpeg"
    - "image/png"
    - "application/pdf"
  storagepath: "/tmp/uploads"

logging:
  level: "debug"
  format: "text"
```

#### config/prod.yaml

```yaml
app:
  environment: "production"

database:
  maxopenconn: 100
  maxidleconn: 20
  ssl: true

logging:
  level: "warn"
  format: "json"
```

#### Using Configuration in Application

```go
// main.go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/mycompany/saas/config"
	"github.com/mycompany/saas/internal/service"
)

func main() {
	// Load configuration
	cfg, err := config.Load()
	if err != nil {
		log.Fatalf("Failed to load configuration: %v", err)
	}

	log.Printf("Starting %s v%s in %s mode",
		cfg.App.Name,
		cfg.App.Version,
		cfg.App.Environment,
	)

	// Initialize services with configuration
	paymentService := service.NewPaymentService(cfg)

	// Setup routes
	http.HandleFunc("/health", healthHandler(cfg))
	http.HandleFunc("/payment", paymentService.HandlePayment)

	// Start server
	addr := fmt.Sprintf(":%d", cfg.App.Port)
	log.Printf("Server listening on %s", addr)
	log.Fatal(http.ListenAndServe(addr, nil))
}

func healthHandler(cfg *config.Config) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprintf(w, `{"status":"ok","environment":"%s"}`, cfg.App.Environment)
	}
}
```

#### Payment Service Example

```go
// internal/service/payment.go
package service

import (
	"encoding/json"
	"net/http"

	"github.com/mycompany/saas/config"
	"github.com/stripe/stripe-go/v76"
	"github.com/stripe/stripe-go/v76/paymentintent"
)

type PaymentService struct {
	config *config.Config
}

func NewPaymentService(cfg *config.Config) *PaymentService {
	stripe.Key = cfg.Stripe.APIKey

	return &PaymentService{
		config: cfg,
	}
}

func (s *PaymentService) HandlePayment(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
		return
	}

	var req struct {
		Amount   int64  `json:"amount"`
		Currency string `json:"currency"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	// Create payment intent
	params := &stripe.PaymentIntentParams{
		Amount:   stripe.Int64(req.Amount),
		Currency: stripe.String(req.Currency),
	}

	pi, err := paymentintent.New(params)
	if err != nil {
		http.Error(w, "Payment failed", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"clientSecret": pi.ClientSecret,
		"status":       pi.Status,
	})
}

func (s *PaymentService) VerifyWebhook(payload []byte, signature string) bool {
	_, err := stripe.ParseWebhookEventWithOptions(
		payload,
		signature,
		s.config.Stripe.WebhookSecret,
		stripe.WebhookOptions{},
	)

	return err == nil
}
```

#### Running with Different Environments

```bash
# Development (default)
go run main.go

# Production
APP_ENV=prod go run main.go

# Staging
APP_ENV=staging go run main.go

# With environment variables
DATABASE_URL=postgresql://prod-server:5432/db APP_ENV=prod go run main.go

# Build and run
go build -o saas-app
APP_ENV=prod ./saas-app
```

---

## 📖 Day 3: Secret Management

### 3.4.7 Why Secret Management Matters

**The Horror Story:**

```
2021: Company commits AWS keys to GitHub
→ Automated bots find keys in < 5 minutes
→ Spin up expensive EC2 instances for crypto mining
→ $50,000 AWS bill in 24 hours
```

**Types of Secrets:**

- Database credentials
- API keys (Stripe, SendGrid, etc.)
- JWT signing keys
- OAuth client secrets
- Encryption keys
- SSH keys
- TLS certificates

---

### 3.4.8 Secret Management Best Practices

#### ✅ DO:

- Store secrets in environment variables
- Use secret management services (Vault, AWS Secrets Manager)
- Rotate secrets regularly
- Use different secrets per environment
- Encrypt secrets at rest
- Limit secret access (principle of least privilege)
- Audit secret access

#### ❌ DON'T:

- Commit secrets to version control
- Log secrets
- Display secrets in UI/responses
- Send secrets in URLs
- Store secrets in client-side code
- Share secrets via email/Slack
- Reuse secrets across environments

---

### 3.4.9 .gitignore Configuration

**Essential .gitignore entries:**

```bash
# Environment variables
.env
.env.local
.env.*.local

# Application config with secrets
application-local.yml
application-local.properties
config/local.json
config/secrets.json

# IDE files
.idea/
.vscode/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Build artifacts
target/
dist/
build/
node_modules/
*.log

# Secrets and keys
*.key
*.pem
*.p12
secrets/
```

---

### 3.4.10 Local Development Secret Management

#### Create .env.example (committed to git)

```bash
# .env.example - Template for developers

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/saas_dev
DATABASE_POOL_MIN=5
DATABASE_POOL_MAX=20

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# JWT - Generate with: openssl rand -base64 32
JWT_SECRET=CHANGE_ME_IN_PRODUCTION

# Stripe - Get from https://dashboard.stripe.com/test/apikeys
STRIPE_API_KEY=sk_test_xxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxx

# AWS S3
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=YOUR_ACCESS_KEY
AWS_SECRET_ACCESS_KEY=YOUR_SECRET_KEY
S3_BUCKET_NAME=my-saas-uploads-dev

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
```

#### Developer Onboarding Process

```bash
# 1. Clone repository
git clone https://github.com/mycompany/saas-backend.git
cd saas-backend

# 2. Copy environment template
cp .env.example .env

# 3. Edit .env with your local values
nano .env

# 4. Generate secure JWT secret
openssl rand -base64 32

# 5. Run application
# Java
./mvnw spring-boot:run

# Node.js
npm install
npm run dev

# Go
go run main.go
```

---

### 3.4.11 Production Secret Management Solutions

#### Option 1: HashiCorp Vault (Self-Hosted)

**When to use:**

- Enterprise with strict compliance requirements
- Multi-cloud deployments
- Need for secret rotation
- Advanced secret policies

**Setup:**

```bash
# Install Vault
brew install vault  # macOS
# or download from https://www.vaultproject.io/downloads

# Start Vault dev server (development only)
vault server -dev

# Set environment variable
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='dev-token'

# Store secrets
vault kv put secret/saas/prod \
  database_url="postgresql://prod-server:5432/saas" \
  jwt_secret="super-secure-secret" \
  stripe_api_key="sk_live_xxxxx"

# Read secrets
vault kv get secret/saas/prod
```

**Java Integration with Vault:**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml
spring:
  application:
    name: saas-backend
  cloud:
    vault:
      uri: http://vault.mycompany.com:8200
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        backend: secret
        profile-separator: "/"
```

**Node.js Integration with Vault:**

```bash
npm install node-vault
```

```typescript
// src/config/vault.ts
import Vault from "node-vault";

const vault = Vault({
  apiVersion: "v1",
  endpoint: process.env.VAULT_ADDR || "http://localhost:8200",
  token: process.env.VAULT_TOKEN,
});

export async function loadSecretsFromVault(path: string) {
  try {
    const result = await vault.read(`secret/data/${path}`);
    return result.data.data;
  } catch (error) {
    console.error("Failed to load secrets from Vault:", error);
    throw error;
  }
}

// Usage in config
export async function initializeConfig() {
  const secrets = await loadSecretsFromVault("saas/prod");

  return {
    database: {
      url: secrets.database_url,
    },
    jwt: {
      secret: secrets.jwt_secret,
    },
    stripe: {
      apiKey: secrets.stripe_api_key,
    },
  };
}
```

**Go Integration with Vault:**

```bash
go get github.com/hashicorp/vault/api
```

```go
// pkg/vault/client.go
package vault

import (
	"fmt"
	"log"

	vault "github.com/hashicorp/vault/api"
)

type Client struct {
	client *vault.Client
}

func NewClient(address, token string) (*Client, error) {
	config := vault.DefaultConfig()
	config.Address = address

	client, err := vault.NewClient(config)
	if err != nil {
		return nil, fmt.Errorf("failed to create vault client: %w", err)
	}

	client.SetToken(token)

	return &Client{client: client}, nil
}

func (c *Client) ReadSecrets(path string) (map[string]interface{}, error) {
	secret, err := c.client.Logical().Read(path)
	if err != nil {
		return nil, fmt.Errorf("failed to read secrets: %w", err)
	}

	if secret == nil {
		return nil, fmt.Errorf("no secrets found at path: %s", path)
	}

	// For KV v2
	data, ok := secret.Data["data"].(map[string]interface{})
	if !ok {
		return secret.Data, nil
	}

	return data, nil
}

// Usage
func LoadConfig() (*config.Config, error) {
	vaultClient, err := vault.NewClient(
		os.Getenv("VAULT_ADDR"),
		os.Getenv("VAULT_TOKEN"),
	)
	if err != nil {
		return nil, err
	}

	secrets, err := vaultClient.ReadSecrets("secret/data/saas/prod")
	if err != nil {
		return nil, err
	}

	cfg := &config.Config{
		Database: config.DatabaseConfig{
			URL: secrets["database_url"].(string),
		},
		JWT: config.JWTConfig{
			Secret: secrets["jwt_secret"].(string),
		},
	}

	return cfg, nil
}
```

---

#### Option 2: AWS Secrets Manager

**When to use:**

- Already using AWS infrastructure
- Need automatic secret rotation
- Integration with other AWS services
- Managed solution preferred

**AWS CLI Setup:**

```bash
# Install AWS CLI
brew install awscli  # macOS

# Configure credentials
aws configure

# Create secret
aws secretsmanager create-secret \
  --name saas/prod/database \
  --description "Production database credentials" \
  --secret-string '{"username":"admin","password":"secure-password","host":"db.example.com","port":"5432"}'

# Create secret for API keys
aws secretsmanager create-secret \
  --name saas/prod/stripe \
  --secret-string '{"api_key":"sk_live_xxxxx","webhook_secret":"whsec_xxxxx"}'

# Retrieve secret
aws secretsmanager get-secret-value --secret-id saas/prod/database
```

**Java Integration with AWS Secrets Manager:**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>secretsmanager</artifactId>
</dependency>
```

```java
package com.mycompany.saas.config;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueResponse;

@Configuration
public class SecretsConfig {

    @Value("${aws.region:us-east-1}")
    private String awsRegion;

    @Bean
    public SecretsManagerClient secretsManagerClient() {
        return SecretsManagerClient.builder()
                .region(Region.of(awsRegion))
                .build();
    }

    public String getSecret(String secretName) {
        SecretsManagerClient client = secretsManagerClient();

        GetSecretValueRequest request = GetSecretValueRequest.builder()
                .secretId(secretName)
                .build();

        GetSecretValueResponse response = client.getSecretValue(request);
        return response.secretString();
    }

    public DatabaseCredentials getDatabaseCredentials() throws Exception {
        String secret = getSecret("saas/prod/database");
        ObjectMapper mapper = new ObjectMapper();
        JsonNode node = mapper.readTree(secret);

        return DatabaseCredentials.builder()
                .username(node.get("username").asText())
                .password(node.get("password").asText())
                .host(node.get("host").asText())
                .port(node.get("port").asInt())
                .build();
    }
}
```

**Node.js Integration with AWS Secrets Manager:**

```bash
npm install @aws-sdk/client-secrets-manager
```

```typescript
// src/config/aws-secrets.ts
import {
  SecretsManagerClient,
  GetSecretValueCommand,
} from "@aws-sdk/client-secrets-manager";

const client = new SecretsManagerClient({
  region: process.env.AWS_REGION || "us-east-1",
});

export async function getSecret(secretName: string): Promise<any> {
  try {
    const command = new GetSecretValueCommand({
      SecretId: secretName,
    });

    const response = await client.send(command);

    if (response.SecretString) {
      return JSON.parse(response.SecretString);
    }

    throw new Error("Secret not found");
  } catch (error) {
    console.error(`Failed to retrieve secret ${secretName}:`, error);
    throw error;
  }
}

// Load all secrets at startup
export async function loadSecretsFromAWS() {
  const [database, stripe, jwt] = await Promise.all([
    getSecret("saas/prod/database"),
    getSecret("saas/prod/stripe"),
    getSecret("saas/prod/jwt"),
  ]);

  return {
    database: {
      url: `postgresql://${database.username}:${database.password}@${database.host}:${database.port}/saas`,
      username: database.username,
      password: database.password,
    },
    stripe: {
      apiKey: stripe.api_key,
      webhookSecret: stripe.webhook_secret,
    },
    jwt: {
      secret: jwt.secret,
    },
  };
}

// Usage in main application
import { loadSecretsFromAWS } from "./config/aws-secrets";
import { appConfig } from "./config";

async function bootstrap() {
  if (process.env.NODE_ENV === "production") {
    const secrets = await loadSecretsFromAWS();

    // Merge secrets with config
    Object.assign(appConfig.database, secrets.database);
    Object.assign(appConfig.stripe, secrets.stripe);
    Object.assign(appConfig.jwt, secrets.jwt);
  }

  // Start application
  startServer();
}

bootstrap().catch(console.error);
```

**Go Integration with AWS Secrets Manager:**

```bash
go get github.com/aws/aws-sdk-go-v2/service/secretsmanager
go get github.com/aws/aws-sdk-go-v2/config
```

```go
// pkg/secrets/aws.go
package secrets

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/secretsmanager"
)

type AWSSecretsManager struct {
	client *secretsmanager.Client
}

func NewAWSSecretsManager(region string) (*AWSSecretsManager, error) {
	cfg, err := config.LoadDefaultConfig(context.TODO(),
		config.WithRegion(region),
	)
	if err != nil {
		return nil, fmt.Errorf("failed to load AWS config: %w", err)
	}

	client := secretsmanager.NewFromConfig(cfg)

	return &AWSSecretsManager{client: client}, nil
}

func (s *AWSSecretsManager) GetSecret(secretName string) (map[string]interface{}, error) {
	input := &secretsmanager.GetSecretValueInput{
		SecretId: aws.String(secretName),
	}

	result, err := s.client.GetSecretValue(context.TODO(), input)
	if err != nil {
		return nil, fmt.Errorf("failed to get secret: %w", err)
	}

	var secretMap map[string]interface{}
	if err := json.Unmarshal([]byte(*result.SecretString), &secretMap); err != nil {
		return nil, fmt.Errorf("failed to unmarshal secret: %w", err)
	}

	return secretMap, nil
}

type DatabaseCredentials struct {
	Username string `json:"username"`
	Password string `json:"password"`
	Host     string `json:"host"`
	Port     int    `json:"port"`
}

func (s *AWSSecretsManager) GetDatabaseCredentials(secretName string) (*DatabaseCredentials, error) {
	secretMap, err := s.GetSecret(secretName)
	if err != nil {
		return nil, err
	}

	creds := &DatabaseCredentials{
		Username: secretMap["username"].(string),
		Password: secretMap["password"].(string),
		Host:     secretMap["host"].(string),
		Port:     int(secretMap["port"].(float64)),
	}

	return creds, nil
}

// Usage in main application
func LoadProductionConfig() (*appconfig.Config, error) {
	secretsManager, err := secrets.NewAWSSecretsManager("us-east-1")
	if err != nil {
		return nil, err
	}

	// Load database credentials
	dbCreds, err := secretsManager.GetDatabaseCredentials("saas/prod/database")
	if err != nil {
		return nil, err
	}

	// Load Stripe secrets
	stripeSecrets, err := secretsManager.GetSecret("saas/prod/stripe")
	if err != nil {
		return nil, err
	}

	// Load JWT secret
	jwtSecrets, err := secretsManager.GetSecret("saas/prod/jwt")
	if err != nil {
		return nil, err
	}

	cfg := &appconfig.Config{
		Database: appconfig.DatabaseConfig{
			URL: fmt.Sprintf("postgresql://%s:%s@%s:%d/saas",
				dbCreds.Username,
				dbCreds.Password,
				dbCreds.Host,
				dbCreds.Port,
			),
		},
		Stripe: appconfig.StripeConfig{
			APIKey:        stripeSecrets["api_key"].(string),
			WebhookSecret: stripeSecrets["webhook_secret"].(string),
		},
		JWT: appconfig.JWTConfig{
			Secret: jwtSecrets["secret"].(string),
		},
	}

	return cfg, nil
}
```

---

#### Option 3: Environment Variables (Simple Approach)

**When to use:**

- Small teams
- Simple deployments
- Cloud platforms with built-in secret management (Heroku, Render)

**Setting Environment Variables:**

```bash
# Linux/macOS - Session
export DATABASE_URL="postgresql://localhost:5432/saas"
export JWT_SECRET="my-secret-key"

# Linux/macOS - Permanent (.bashrc, .zshrc)
echo 'export JWT_SECRET="my-secret-key"' >> ~/.zshrc

# Docker
docker run -e DATABASE_URL="postgresql://..." myapp

# Docker Compose
# docker-compose.yml
services:
  app:
    environment:
      - DATABASE_URL=postgresql://db:5432/saas
      - JWT_SECRET=${JWT_SECRET}

# Kubernetes Secret
apiVersion: v1
kind: Secret
metadata:
  name: saas-secrets
type: Opaque
stringData:
  database-url: postgresql://localhost:5432/saas
  jwt-secret: my-secret-key
```

---

### 3.4.12 Secret Rotation Strategy

**Why Rotate Secrets?**

- Limit damage from compromised secrets
- Compliance requirements (PCI-DSS, HIPAA)
- Former employee access
- Regular security hygiene

**Rotation Schedule:**

- **Critical secrets** (root DB passwords): Every 30 days
- **API keys**: Every 90 days
- **JWT signing keys**: Every 6-12 months
- **After breach**: Immediately

**Rotation Process:**

```bash
# 1. Generate new secret
NEW_SECRET=$(openssl rand -base64 32)

# 2. Add new secret alongside old (dual-write period)
#    Application accepts both old and new secrets

# 3. Update all instances to use new secret

# 4. Monitor for errors

# 5. Remove old secret after grace period (24-48 hours)
```

**JWT Secret Rotation Example (Node.js):**

```typescript
// src/config/jwt.ts
export const jwtConfig = {
  // Support multiple secrets during rotation
  secrets: [
    process.env.JWT_SECRET_NEW, // New secret (for signing)
    process.env.JWT_SECRET_OLD, // Old secret (for verification only)
  ].filter(Boolean),

  currentSecret: process.env.JWT_SECRET_NEW,
  expiration: "24h",
};

// src/auth/jwt.service.ts
import jwt from "jsonwebtoken";
import { jwtConfig } from "../config/jwt";

export class JWTService {
  // Always sign with current secret
  sign(payload: any): string {
    return jwt.sign(payload, jwtConfig.currentSecret, {
      expiresIn: jwtConfig.expiration,
    });
  }

  // Verify against all valid secrets
  verify(token: string): any {
    let lastError: Error;

    for (const secret of jwtConfig.secrets) {
      try {
        return jwt.verify(token, secret);
      } catch (error) {
        lastError = error as Error;
        continue;
      }
    }

    throw lastError!;
  }
}
```

---

## 📖 Day 4: Feature Flags & Dynamic Configuration

### 3.4.13 Why Feature Flags?

**Benefits:**

- Deploy code without releasing features
- A/B testing and experimentation
- Gradual rollouts (canary releases)
- Quick rollback without deployment
- Testing in production safely
- Per-tenant feature control

**Use Cases:**

- New feature behind flag until ready
- Premium features for paid tiers
- Beta features for selected users
- Kill switch for problematic features
- Regional feature availability

---

### 3.4.14 Feature Flag Patterns

#### Simple Boolean Flags

```typescript
const features = {
  newDashboard: true,
  aiRecommendations: false,
  advancedAnalytics: true,
};

if (features.newDashboard) {
  // Show new dashboard
} else {
  // Show old dashboard
}
```

#### User-Based Flags

```typescript
function isFeatureEnabled(feature: string, userId: string): boolean {
  const config = {
    newDashboard: {
      enabled: true,
      users: ["user-123", "user-456"], // Whitelist
    },
    aiRecommendations: {
      enabled: true,
      percentage: 20, // 20% of users
    },
  };

  const featureConfig = config[feature];

  if (!featureConfig.enabled) return false;

  // Whitelist check
  if (featureConfig.users?.includes(userId)) {
    return true;
  }

  // Percentage rollout
  if (featureConfig.percentage) {
    const hash = hashUserId(userId);
    return hash % 100 < featureConfig.percentage;
  }

  return featureConfig.enabled;
}
```

#### Tenant-Based Flags (Multi-Tenancy)

```typescript
interface FeatureFlag {
  name: string;
  enabledForTenants: string[];
  enabledForPlans: string[];
  defaultEnabled: boolean;
}

function isFeatureEnabledForTenant(
  feature: string,
  tenantId: string,
  planType: string
): boolean {
  const featureFlags: Record<string, FeatureFlag> = {
    advancedReporting: {
      name: "Advanced Reporting",
      enabledForTenants: ["tenant-enterprise-1"],
      enabledForPlans: ["enterprise", "premium"],
      defaultEnabled: false,
    },
    apiAccess: {
      name: "API Access",
      enabledForTenants: [],
      enabledForPlans: ["premium", "enterprise"],
      defaultEnabled: false,
    },
  };

  const flag = featureFlags[feature];
  if (!flag) return false;

  // Check tenant whitelist
  if (flag.enabledForTenants.includes(tenantId)) {
    return true;
  }

  // Check plan type
  if (flag.enabledForPlans.includes(planType)) {
    return true;
  }

  return flag.defaultEnabled;
}
```

---

### 3.4.15 Feature Flag Implementation

#### Java/Spring Boot with Custom Implementation

```java
// FeatureFlag.java
package com.mycompany.saas.feature;

import java.util.Set;

public class FeatureFlag {
    private String name;
    private boolean enabled;
    private Set<String> enabledForTenants;
    private Set<String> enabledForPlans;
    private Integer rolloutPercentage;

    // Getters and setters
}

// FeatureFlagService.java
package com.mycompany.saas.feature;

import org.springframework.stereotype.Service;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class FeatureFlagService {

    private final Map<String, FeatureFlag> flags = new ConcurrentHashMap<>();

    public FeatureFlagService() {
        initializeFlags();
    }

    private void initializeFlags() {
        FeatureFlag newDashboard = new FeatureFlag();
        newDashboard.setName("new_dashboard");
        newDashboard.setEnabled(true);
        newDashboard.setRolloutPercentage(50);
        flags.put("new_dashboard", newDashboard);

        FeatureFlag apiAccess = new FeatureFlag();
        apiAccess.setName("api_access");
        apiAccess.setEnabled(true);
        apiAccess.setEnabledForPlans(Set.of("premium", "enterprise"));
        flags.put("api_access", apiAccess);
    }

    public boolean isEnabled(String featureName, String userId) {
        FeatureFlag flag = flags.get(featureName);
        if (flag == null || !flag.isEnabled()) {
            return false;
        }

        // Check rollout percentage
        if (flag.getRolloutPercentage() != null) {
            int hash = Math.abs(userId.hashCode() % 100);
            return hash < flag.getRolloutPercentage();
        }

        return true;
    }

    public boolean isEnabledForTenant(String featureName, String tenantId, String planType) {
        FeatureFlag flag = flags.get(featureName);
        if (flag == null || !flag.isEnabled()) {
            return false;
        }

        // Check tenant whitelist
        if (flag.getEnabledForTenants() != null &&
            flag.getEnabledForTenants().contains(tenantId)) {
            return true;
        }

        // Check plan type
        if (flag.getEnabledForPlans() != null &&
            flag.getEnabledForPlans().contains(planType)) {
            return true;
        }

        return false;
    }

    // Dynamic flag updates (from database or admin panel)
    public void updateFlag(String featureName, FeatureFlag updatedFlag) {
        flags.put(featureName, updatedFlag);
    }
}

// Usage in Controller
@RestController
@RequestMapping("/api/dashboard")
public class DashboardController {

    private final FeatureFlagService featureFlagService;

    public DashboardController(FeatureFlagService featureFlagService) {
        this.featureFlagService = featureFlagService;
    }

    @GetMapping
    public ResponseEntity<?> getDashboard(
            @RequestHeader("X-User-Id") String userId) {

        if (featureFlagService.isEnabled("new_dashboard", userId)) {
            return ResponseEntity.ok(getNewDashboard());
        } else {
            return ResponseEntity.ok(getOldDashboard());
        }
    }
}
```

#### Node.js with LaunchDarkly Integration

```bash
npm install launchdarkly-node-server-sdk
```

```typescript
// src/services/feature-flags.service.ts
import LaunchDarkly from "launchdarkly-node-server-sdk";

class FeatureFlagService {
  private client: LaunchDarkly.LDClient;

  async initialize() {
    this.client = LaunchDarkly.init(process.env.LAUNCHDARKLY_SDK_KEY!);
    await this.client.waitForInitialization();
    console.log("Feature flags initialized");
  }

  async isEnabled(
    flagKey: string,
    user: { key: string; email?: string; custom?: any }
  ): Promise<boolean> {
    try {
      return await this.client.variation(flagKey, user, false);
    } catch (error) {
      console.error(`Error checking flag ${flagKey}:`, error);
      return false;
    }
  }

  async getFlagValue<T>(
    flagKey: string,
    user: { key: string; email?: string; custom?: any },
    defaultValue: T
  ): Promise<T> {
    try {
      return await this.client.variation(flagKey, user, defaultValue);
    } catch (error) {
      console.error(`Error getting flag ${flagKey}:`, error);
      return defaultValue;
    }
  }

  async close() {
    await this.client.close();
  }
}

export const featureFlagService = new FeatureFlagService();

// Usage in Express middleware
import { featureFlagService } from "./services/feature-flags.service";

app.get("/api/dashboard", async (req, res) => {
  const user = {
    key: req.user.id,
    email: req.user.email,
    custom: {
      tenantId: req.user.tenantId,
      planType: req.user.planType,
    },
  };

  const useNewDashboard = await featureFlagService.isEnabled(
    "new-dashboard",
    user
  );

  if (useNewDashboard) {
    res.json(await getNewDashboard(req.user));
  } else {
    res.json(await getOldDashboard(req.user));
  }
});
```

#### Go with Unleash

```bash
go get github.com/Unleash/unleash-client-go/v3
```

```go
// pkg/featureflags/client.go
package featureflags

import (
	"github.com/Unleash/unleash-client-go/v3"
	"github.com/Unleash/unleash-client-go/v3/context"
)

type Client struct {
	unleash *unleash.Client
}

func NewClient(apiURL, apiToken string) (*Client, error) {
	unleashClient, err := unleash.NewClient(
		unleash.WithListener(&unleash.DebugListener{}),
		unleash.WithAppName("saas-backend"),
		unleash.WithUrl(apiURL),
		unleash.WithCustomHeaders(map[string]string{
			"Authorization": apiToken,
		}),
	)

	if err != nil {
		return nil, err
	}

	return &Client{unleash: unleashClient}, nil
}

func (c *Client) IsEnabled(featureName string, userID string, properties map[string]string) bool {
	ctx := context.Context{
		UserId:     userID,
		Properties: properties,
	}

	return c.unleash.IsEnabled(featureName, unleash.WithContext(ctx))
}

func (c *Client) Close() error {
	c.unleash.Close()
	return nil
}

// Usage in HTTP handler
func (h *Handler) GetDashboard(w http.ResponseWriter, r *http.Request) {
	userID := r.Header.Get("X-User-Id")
	tenantID := r.Header.Get("X-Tenant-Id")

	properties := map[string]string{
		"tenantId": tenantID,
		"planType": "enterprise",
	}

	if h.featureFlags.IsEnabled("new-dashboard", userID, properties) {
		json.NewEncoder(w).Encode(h.getNewDashboard(userID))
	} else {
		json.NewEncoder(w).Encode(h.getOldDashboard(userID))
	}
}
```

---

### 3.4.16 Feature Flag Services Comparison

| Service          | Best For          | Pricing    | Key Features                               |
| ---------------- | ----------------- | ---------- | ------------------------------------------ |
| **LaunchDarkly** | Enterprise        | $$         | Advanced targeting, A/B testing, Analytics |
| **Unleash**      | Self-hosted       | Free (OSS) | Open source, privacy-focused               |
| **Split.io**     | Data-driven teams | $$         | Built-in experimentation, metrics          |
| **Flagsmith**    | Startups          | $          | Simple, good free tier                     |
| **ConfigCat**    | Small teams       | $          | Easy setup, affordable                     |
| **Custom DB**    | Full control      | Free       | Complete control, more work                |

**Recommendation:**

- **Startups:** Custom implementation or Flagsmith
- **Scale-ups:** LaunchDarkly or Split.io
- **Enterprise:** LaunchDarkly
- **Self-hosted:** Unleash

---

### 3.4.17 Feature Flag Database Schema

```sql
CREATE TABLE feature_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    enabled BOOLEAN DEFAULT false,
    rollout_percentage INTEGER CHECK (rollout_percentage BETWEEN 0 AND 100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE feature_flag_tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    feature_flag_id UUID REFERENCES feature_flags(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL,
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(feature_flag_id, tenant_id)
);

CREATE TABLE feature_flag_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    feature_flag_id UUID REFERENCES feature_flags(id) ON DELETE CASCADE,
    plan_type VARCHAR(50) NOT NULL,
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(feature_flag_id, plan_type)
);

CREATE INDEX idx_feature_flags_name ON feature_flags(name);
CREATE INDEX idx_feature_flag_tenants_tenant ON feature_flag_tenants(tenant_id);
CREATE INDEX idx_feature_flag_plans_plan ON feature_flag_plans(plan_type);
```

---

## 🎯 Practical Exercises

### Exercise 1: Environment-Specific Configuration

**Task:** Set up a complete configuration system for a SaaS application with dev, staging, and production environments.

**Requirements:**

1. Create configuration files for all three environments
2. Use environment variables for secrets
3. Implement type-safe configuration loading
4. Add validation for required configuration

**Java Solution:**

```java
// Step 1: Create application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/saas_dev
    username: dev_user
    password: ${DB_PASSWORD:dev_password}
  jpa:
    show-sql: true

logging:
  level:
    root: DEBUG

// Step 2: Create application-prod.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false

logging:
  level:
    root: WARN

// Step 3: Run with profile
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

**Node.js Solution:**

```typescript
// Step 1: Create config/development.json
{
  "database": {
    "url": "postgresql://localhost:5432/saas_dev"
  },
  "logging": {
    "level": "debug"
  }
}

// Step 2: Create config/production.json
{
  "logging": {
    "level": "warn"
  }
}

// Step 3: Create .env for secrets
DATABASE_URL=postgresql://prod-server:5432/saas
JWT_SECRET=super-secret-key

// Step 4: Run
NODE_ENV=production npm start
```

**Go Solution:**

```go
// Step 1: Create config/dev.yaml
app:
  environment: "development"
logging:
  level: "debug"

// Step 2: Create config/prod.yaml
app:
  environment: "production"
database:
  maxopenconn: 100
logging:
  level: "warn"

// Step 3: Run
APP_ENV=prod go run main.go
```

---

### Exercise 2: Implement Secret Rotation

**Task:** Create a system that supports JWT secret rotation without downtime.

**Node.js Solution:**

```typescript
// src/auth/jwt-rotation.service.ts
interface JWTConfig {
  secrets: string[];
  currentSecretIndex: number;
}

class JWTRotationService {
  private config: JWTConfig;

  constructor() {
    this.config = {
      secrets: [
        process.env.JWT_SECRET_NEW!,
        process.env.JWT_SECRET_OLD!,
      ].filter(Boolean),
      currentSecretIndex: 0,
    };
  }

  // Always sign with newest secret
  sign(payload: any): string {
    const currentSecret = this.config.secrets[this.config.currentSecretIndex];
    return jwt.sign(payload, currentSecret, { expiresIn: "24h" });
  }

  // Verify with any valid secret
  verify(token: string): any {
    let lastError: Error | null = null;

    for (const secret of this.config.secrets) {
      try {
        return jwt.verify(token, secret);
      } catch (error) {
        lastError = error as Error;
      }
    }

    throw lastError || new Error("Token verification failed");
  }

  // Rotate to next secret
  rotateSecret(newSecret: string) {
    this.config.secrets.unshift(newSecret);
    this.config.currentSecretIndex = 0;

    // Keep only 2 most recent secrets
    if (this.config.secrets.length > 2) {
      this.config.secrets = this.config.secrets.slice(0, 2);
    }
  }
}

export const jwtRotationService = new JWTRotationService();
```

**Rotation Process:**

```bash
# Step 1: Generate new secret
openssl rand -base64 32

# Step 2: Add as JWT_SECRET_NEW (keep old as JWT_SECRET_OLD)
export JWT_SECRET_NEW="new-secret-here"
export JWT_SECRET_OLD="old-secret-here"

# Step 3: Restart application (now accepts both)

# Step 4: Wait 24 hours (token expiration)

# Step 5: Remove old secret
unset JWT_SECRET_OLD

# Step 6: Rotate variables
export JWT_SECRET_OLD="$JWT_SECRET_NEW"
export JWT_SECRET_NEW=""
```

---

### Exercise 3: Build a Feature Flag System

**Task:** Create a database-backed feature flag system with admin API.

**Database Schema:**

```sql
CREATE TABLE feature_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    enabled BOOLEAN DEFAULT false,
    rollout_percentage INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE feature_flag_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    feature_flag_id UUID REFERENCES feature_flags(id) ON DELETE CASCADE,
    rule_type VARCHAR(50) NOT NULL, -- 'tenant', 'plan', 'user'
    rule_value TEXT NOT NULL,
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Sample data
INSERT INTO feature_flags (name, description, enabled, rollout_percentage)
VALUES
    ('new_dashboard', 'New dashboard UI', true, 50),
    ('ai_features', 'AI-powered recommendations', true, 10),
    ('api_access', 'API access for integrations', true, 100);

INSERT INTO feature_flag_rules (feature_flag_id, rule_type, rule_value)
VALUES
    ((SELECT id FROM feature_flags WHERE name = 'api_access'), 'plan', 'premium'),
    ((SELECT id FROM feature_flags WHERE name = 'api_access'), 'plan', 'enterprise');
```

**Node.js Implementation:**

```typescript
// src/services/feature-flag.service.ts
import { Pool } from "pg";

interface FeatureFlag {
  id: string;
  name: string;
  enabled: boolean;
  rolloutPercentage: number;
}

interface FeatureFlagRule {
  ruleType: string;
  ruleValue: string;
  enabled: boolean;
}

class FeatureFlagService {
  private pool: Pool;
  private cache: Map<string, FeatureFlag> = new Map();

  constructor(pool: Pool) {
    this.pool = pool;
    this.refreshCache();

    // Refresh cache every 60 seconds
    setInterval(() => this.refreshCache(), 60000);
  }

  private async refreshCache() {
    const result = await this.pool.query(
      "SELECT * FROM feature_flags WHERE enabled = true"
    );

    this.cache.clear();
    result.rows.forEach((row) => {
      this.cache.set(row.name, {
        id: row.id,
        name: row.name,
        enabled: row.enabled,
        rolloutPercentage: row.rollout_percentage,
      });
    });
  }

  async isEnabled(
    featureName: string,
    context: {
      userId?: string;
      tenantId?: string;
      planType?: string;
    }
  ): Promise<boolean> {
    const flag = this.cache.get(featureName);
    if (!flag || !flag.enabled) {
      return false;
    }

    // Check rules
    const rules = await this.getRules(flag.id);

    // Check tenant rules
    if (context.tenantId) {
      const tenantRule = rules.find(
        (r) => r.ruleType === "tenant" && r.ruleValue === context.tenantId
      );
      if (tenantRule) return tenantRule.enabled;
    }

    // Check plan rules
    if (context.planType) {
      const planRule = rules.find(
        (r) => r.ruleType === "plan" && r.ruleValue === context.planType
      );
      if (planRule) return planRule.enabled;
    }

    // Check rollout percentage
    if (flag.rolloutPercentage < 100 && context.userId) {
      const hash = this.hashString(context.userId);
      return hash % 100 < flag.rolloutPercentage;
    }

    return flag.rolloutPercentage === 100;
  }

  private async getRules(flagId: string): Promise<FeatureFlagRule[]> {
    const result = await this.pool.query(
      "SELECT rule_type, rule_value, enabled FROM feature_flag_rules WHERE feature_flag_id = $1",
      [flagId]
    );

    return result.rows.map((row) => ({
      ruleType: row.rule_type,
      ruleValue: row.rule_value,
      enabled: row.enabled,
    }));
  }

  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = (hash << 5) - hash + char;
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  // Admin methods
  async createFlag(
    name: string,
    description: string,
    rolloutPercentage: number = 0
  ) {
    const result = await this.pool.query(
      `INSERT INTO feature_flags (name, description, enabled, rollout_percentage)
       VALUES ($1, $2, true, $3) RETURNING *`,
      [name, description, rolloutPercentage]
    );

    await this.refreshCache();
    return result.rows[0];
  }

  async updateFlag(name: string, updates: Partial<FeatureFlag>) {
    const setClauses = [];
    const values = [];
    let paramCount = 1;

    if (updates.enabled !== undefined) {
      setClauses.push(`enabled = ${paramCount++}`);
      values.push(updates.enabled);
    }

    if (updates.rolloutPercentage !== undefined) {
      setClauses.push(`rollout_percentage = ${paramCount++}`);
      values.push(updates.rolloutPercentage);
    }

    setClauses.push(`updated_at = CURRENT_TIMESTAMP`);
    values.push(name);

    await this.pool.query(
      `UPDATE feature_flags SET ${setClauses.join(
        ", "
      )} WHERE name = ${paramCount}`,
      values
    );

    await this.refreshCache();
  }

  async addRule(flagName: string, ruleType: string, ruleValue: string) {
    const flag = this.cache.get(flagName);
    if (!flag) throw new Error("Feature flag not found");

    await this.pool.query(
      `INSERT INTO feature_flag_rules (feature_flag_id, rule_type, rule_value)
       VALUES ($1, $2, $3)`,
      [flag.id, ruleType, ruleValue]
    );
  }
}

export { FeatureFlagService };
```

**Admin API Endpoints:**

```typescript
// src/routes/admin/feature-flags.routes.ts
import express from "express";
import { FeatureFlagService } from "../../services/feature-flag.service";

const router = express.Router();

// List all flags
router.get("/flags", async (req, res) => {
  const result = await pool.query("SELECT * FROM feature_flags ORDER BY name");
  res.json(result.rows);
});

// Create new flag
router.post("/flags", async (req, res) => {
  const { name, description, rolloutPercentage } = req.body;

  const flag = await featureFlagService.createFlag(
    name,
    description,
    rolloutPercentage
  );

  res.status(201).json(flag);
});

// Update flag
router.patch("/flags/:name", async (req, res) => {
  const { name } = req.params;
  const updates = req.body;

  await featureFlagService.updateFlag(name, updates);
  res.json({ success: true });
});

// Add rule to flag
router.post("/flags/:name/rules", async (req, res) => {
  const { name } = req.params;
  const { ruleType, ruleValue } = req.body;

  await featureFlagService.addRule(name, ruleType, ruleValue);
  res.status(201).json({ success: true });
});

// Test flag for user
router.post("/flags/:name/test", async (req, res) => {
  const { name } = req.params;
  const context = req.body;

  const enabled = await featureFlagService.isEnabled(name, context);
  res.json({ enabled });
});

export default router;
```

**Usage in Application:**

```typescript
// src/middleware/feature-flag.middleware.ts
import { featureFlagService } from "../services/feature-flag.service";

export function requireFeature(featureName: string) {
  return async (req: any, res: any, next: any) => {
    const context = {
      userId: req.user?.id,
      tenantId: req.user?.tenantId,
      planType: req.user?.planType,
    };

    const enabled = await featureFlagService.isEnabled(featureName, context);

    if (!enabled) {
      return res.status(403).json({
        error: "Feature not available",
        feature: featureName,
      });
    }

    next();
  };
}

// Usage in routes
app.get(
  "/api/ai/recommendations",
  authenticate,
  requireFeature("ai_features"),
  async (req, res) => {
    // AI recommendations logic
  }
);
```

---

## 📚 Best Practices Summary

### Configuration Management

✅ **DO:**

- Use environment variables for secrets
- Have separate configs for each environment
- Validate configuration on startup
- Use type-safe configuration objects
- Document all configuration options
- Provide sensible defaults
- Use configuration hierarchies

❌ **DON'T:**

- Commit secrets to version control
- Use production secrets in development
- Mix configuration with business logic
- Deploy with default/example secrets
- Store secrets in client-side code

---

### Secret Management

✅ **DO:**

- Use secret management services (Vault, AWS Secrets)
- Rotate secrets regularly
- Audit secret access
- Use different secrets per environment
- Encrypt secrets at rest
- Grant minimal permissions
- Monitor for leaked secrets

❌ **DON'T:**

- Log secrets
- Display secrets in UI
- Email/Slack secrets
- Store secrets in plain text
- Reuse secrets across services
- Hardcode secrets

---

### Feature Flags

✅ **DO:**

- Use feature flags for new features
- Clean up old flags regularly
- Document flag purposes
- Test both enabled/disabled states
- Use gradual rollouts
- Monitor flag usage
- Cache flag values appropriately

❌ **DON'T:**

- Leave abandoned flags in code
- Use flags for configuration
- Make flags too granular
- Skip testing disabled state
- Deploy flag changes without testing

---

## 🔍 Debugging Configuration Issues

### Common Problems and Solutions

#### Problem 1: Configuration Not Loading

**Symptoms:**

```
Error: Configuration file not found
Error: Required environment variable missing
```

**Debug Steps:**

```bash
# Java - Check active profile
java -jar app.jar --spring.profiles.active=prod --debug

# Node.js - Check NODE_ENV
echo $NODE_ENV
node -e "console.log(process.env)"

# Go - Check config path
APP_ENV=prod CONFIG_PATH=./config go run main.go

# Verify file exists
ls -la config/
cat config/prod.yaml
```

**Solution:**

```bash
# Ensure profile/environment is set
export NODE_ENV=production
export SPRING_PROFILES_ACTIVE=prod
export APP_ENV=prod

# Verify config file path is correct
export CONFIG_PATH=/etc/myapp/config
```

---

#### Problem 2: Environment Variables Not Overriding

**Symptoms:**

```
Application using default values instead of environment variables
```

**Debug Steps:**

```typescript
// Add debug logging
console.log("Environment Variables:", {
  DATABASE_URL: process.env.DATABASE_URL,
  JWT_SECRET: process.env.JWT_SECRET ? "***" : "NOT SET",
  NODE_ENV: process.env.NODE_ENV,
});

// Check if variables are being read
import config from "config";
console.log("Config:", config.util.toObject());
```

**Solution:**

```bash
# Ensure variables are exported, not just set
export DATABASE_URL="postgresql://..."  # ✅ Exported
DATABASE_URL="postgresql://..."        # ❌ Not exported

# Check variable scope
env | grep DATABASE_URL

# For Docker, ensure variables are passed
docker run -e DATABASE_URL="..." myapp
```

---

#### Problem 3: Secrets Not Found in Production

**Symptoms:**

```
Error: Secret 'saas/prod/database' not found
AWS Secrets Manager: AccessDeniedException
```

**Debug Steps:**

```bash
# Test AWS credentials
aws sts get-caller-identity

# Test secret access
aws secretsmanager get-secret-value --secret-id saas/prod/database

# Check IAM permissions
aws iam get-role-policy --role-name myapp-prod --policy-name SecretsAccess
```

**Solution:**

```json
// Add required IAM permissions
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789:secret:saas/prod/*"
    }
  ]
}
```

---

#### Problem 4: Feature Flags Not Working

**Symptoms:**

```
Feature flag always returns false
Feature flag not respecting rollout percentage
```

**Debug Steps:**

```typescript
// Add detailed logging
async isEnabled(featureName: string, context: any): Promise<boolean> {
  console.log('Checking feature:', featureName);
  console.log('Context:', context);

  const flag = this.cache.get(featureName);
  console.log('Flag from cache:', flag);

  if (!flag) {
    console.log('Flag not found in cache');
    return false;
  }

  const result = // ... evaluation logic
  console.log('Final result:', result);
  return result;
}

// Test manually
curl -X POST http://localhost:3000/admin/flags/new_dashboard/test \
  -H "Content-Type: application/json" \
  -d '{"userId":"user-123","tenantId":"tenant-456","planType":"premium"}'
```

**Solution:**

```typescript
// Ensure cache is refreshing
private async refreshCache() {
  console.log('Refreshing feature flag cache...');
  const result = await this.pool.query(
    'SELECT * FROM feature_flags WHERE enabled = true'
  );
  console.log(`Loaded ${result.rows.length} feature flags`);

  this.cache.clear();
  result.rows.forEach(row => {
    this.cache.set(row.name, { /* ... */ });
  });
}

// Force cache refresh
await featureFlagService.refreshCache();
```

---

## 📖 Additional Resources

### Documentation

**12-Factor App:**

- https://12factor.net/config
- Essential principles for modern SaaS configuration

**Spring Boot Configuration:**

- https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config
- Comprehensive guide to Spring Boot configuration

**Node.js Config Package:**

- https://github.com/node-config/node-config
- Popular configuration management for Node.js

**Viper (Go):**

- https://github.com/spf13/viper
- Complete configuration solution for Go

**HashiCorp Vault:**

- https://www.vaultproject.io/docs
- Secret management best practices

**AWS Secrets Manager:**

- https://docs.aws.amazon.com/secretsmanager/
- AWS secret management guide

**LaunchDarkly Documentation:**

- https://docs.launchdarkly.com/
- Feature flag implementation patterns

---

### Books

📚 **"The Twelve-Factor App"** by Adam Wiggins

- Free online: https://12factor.net/
- Essential reading for SaaS developers

📚 **"Release It!"** by Michael Nygard

- Chapter on configuration management
- Production-ready patterns

📚 **"Building Microservices"** by Sam Newman

- Configuration in distributed systems
- Service-specific configuration

---

### Tools

**Secret Scanning:**

- **GitGuardian**: Scans for secrets in Git repos
- **TruffleHog**: Finds secrets in Git history
- **git-secrets**: Prevents committing secrets

**Configuration Validation:**

- **JSON Schema**: Validate configuration structure
- **Joi** (Node.js): Schema validation
- **Hibernate Validator** (Java): Bean validation

**Feature Flag Services:**

- **LaunchDarkly**: Enterprise feature management
- **Unleash**: Open-source feature flags
- **Flagsmith**: Developer-friendly flags
- **Split.io**: Feature flags with experimentation

---

## 🎯 Module Completion Checklist

By the end of this module, you should be able to:

- [ ] Set up environment-specific configuration files
- [ ] Use environment variables for secrets
- [ ] Implement type-safe configuration in Java, Node.js, and Go
- [ ] Validate configuration on application startup
- [ ] Integrate with secret management services (Vault, AWS Secrets Manager)
- [ ] Implement secret rotation strategies
- [ ] Create and manage feature flags
- [ ] Build a database-backed feature flag system
- [ ] Use feature flags for gradual rollouts
- [ ] Debug configuration issues in production
- [ ] Follow configuration best practices

---

## 🚀 Next Steps

**In Module 3.5**, we'll cover:

- Git branching strategies (Git Flow, GitHub Flow, Trunk-Based)
- Code review best practices
- Pull request workflows
- Monorepo vs polyrepo strategies
- Git hooks and automation

**Coming Up:**

- **Module 4**: Authentication & Authorization (7 days)
- **Module 5**: Multi-Tenancy Architecture (6 days)
- **Module 6**: Database Design & Data Management (8 days)

---

## 💡 Key Takeaways

1. **Never commit secrets to version control** - Use environment variables and secret managers

2. **Configuration hierarchy matters** - Defaults → Files → Environment Variables → Command Line

3. **Validate early** - Check configuration on startup, fail fast with clear error messages

4. **Use type safety** - Leverage TypeScript interfaces, Java @ConfigurationProperties, Go structs

5. **Feature flags enable safe deployment** - Deploy code without releasing features

6. **Cache wisely** - Feature flags should be cached but refreshed regularly

7. **Document everything** - Configuration options, secret rotation procedures, feature flag purposes

8. **Plan for rotation** - Secrets should be rotatable without downtime

9. **Test all environments** - Don't assume production will work if dev does

10. **Monitor configuration** - Track which features are enabled, when secrets were rotated

---
