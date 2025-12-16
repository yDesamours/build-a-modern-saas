# Module 4: Authentication & Authorization

**Duration:** 7 days  
**Part of:** Phase 2 - Core SaaS Functionalities

---

## 📚 Learning Objectives

By the end of this module, you will:

- Understand the difference between authentication and authorization
- Implement JWT-based authentication in Java, Node.js, and Go
- Integrate OAuth 2.0 and social login providers
- Build Role-Based Access Control (RBAC) systems
- Implement Multi-Factor Authentication (MFA)
- Handle password security correctly
- Implement Single Sign-On (SSO) for enterprise customers
- Secure APIs with API keys and tokens
- Understand session management strategies

---

## 🎯 Authentication vs Authorization

### Key Differences

**Authentication (AuthN):** _"Who are you?"_

- Verifying user identity
- Username/password, biometrics, tokens
- Answers: "Are you really John Doe?"

**Authorization (AuthZ):** _"What can you do?"_

- Verifying permissions
- Roles, permissions, access control
- Answers: "Can John Doe access this resource?"

### Real-World Example

```typescript
// Authentication: Verifying identity
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Response: JWT token (proves identity)

// Authorization: Checking permissions
GET /api/admin/users
Headers: { Authorization: "Bearer " }

// System checks:
// 1. Is token valid? (Authentication)
// 2. Does user have 'admin' role? (Authorization)
// 3. Allow or deny request
```

---

## 📖 Day 1: Authentication Fundamentals & Session-Based Auth

### 4.1 Authentication Strategies Overview

#### Strategy Comparison

| Strategy              | Best For                | Storage     | Scalability | Mobile Support |
| --------------------- | ----------------------- | ----------- | ----------- | -------------- |
| **Session-Based**     | Traditional web apps    | Server-side | Challenging | Limited        |
| **Token-Based (JWT)** | Modern SPAs, APIs       | Client-side | Excellent   | Excellent      |
| **OAuth 2.0**         | Third-party integration | Varies      | Excellent   | Excellent      |
| **API Keys**          | Server-to-server        | Server-side | Excellent   | N/A            |
| **Magic Links**       | Passwordless auth       | Server-side | Good        | Good           |

---

### 4.2 Session-Based Authentication

**How It Works:**

```
1. User logs in → Server validates credentials
2. Server creates session → Stores in database/Redis
3. Server sends session ID → Cookie to client
4. Client sends cookie → With every request
5. Server validates session ID → Checks session store
6. Server returns data → If session valid
```

**When to Use:**

- Traditional server-rendered applications
- Applications with sticky sessions
- When you need instant session invalidation
- Internal enterprise applications

**Pros:**

- ✅ Server controls sessions completely
- ✅ Easy to invalidate immediately
- ✅ No token expiration issues
- ✅ Session data on server (more secure)

**Cons:**

- ❌ Requires session storage (Redis, database)
- ❌ Harder to scale horizontally
- ❌ CORS complications
- ❌ Not suitable for mobile apps

---

#### Implementation: Node.js/Express with Sessions

**Installation:**

```bash
npm install express express-session connect-redis redis
npm install --save-dev @types/express @types/express-session
```

**Session Configuration:**

```typescript
// src/config/session.ts
import session from "express-session";
import RedisStore from "connect-redis";
import { createClient } from "redis";

// Create Redis client
const redisClient = createClient({
  host: process.env.REDIS_HOST || "localhost",
  port: parseInt(process.env.REDIS_PORT || "6379"),
  password: process.env.REDIS_PASSWORD,
});

redisClient.connect().catch(console.error);

// Session configuration
export const sessionConfig = session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  name: "sessionId", // Cookie name
  cookie: {
    secure: process.env.NODE_ENV === "production", // HTTPS only in production
    httpOnly: true, // Prevent JavaScript access
    maxAge: 1000 * 60 * 60 * 24 * 7, // 7 days
    sameSite: "lax", // CSRF protection
  },
});

// Extend Express Session type
declare module "express-session" {
  interface SessionData {
    userId: string;
    email: string;
    role: string;
  }
}
```

**Authentication Routes:**

```typescript
// src/routes/auth.routes.ts
import express, { Request, Response } from "express";
import bcrypt from "bcrypt";
import { User } from "../models/user.model";

const router = express.Router();

// Register
router.post("/register", async (req: Request, res: Response) => {
  try {
    const { email, password, name } = req.body;

    // Check if user exists
    const existingUser = await User.findByEmail(email);
    if (existingUser) {
      return res.status(400).json({ error: "Email already registered" });
    }

    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);

    // Create user
    const user = await User.create({
      email,
      password: hashedPassword,
      name,
      role: "user",
    });

    // Create session
    req.session.userId = user.id;
    req.session.email = user.email;
    req.session.role = user.role;

    res.status(201).json({
      message: "User registered successfully",
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
      },
    });
  } catch (error) {
    console.error("Registration error:", error);
    res.status(500).json({ error: "Registration failed" });
  }
});

// Login
router.post("/login", async (req: Request, res: Response) => {
  try {
    const { email, password } = req.body;

    // Find user
    const user = await User.findByEmail(email);
    if (!user) {
      return res.status(401).json({ error: "Invalid credentials" });
    }

    // Verify password
    const isValidPassword = await bcrypt.compare(password, user.password);
    if (!isValidPassword) {
      return res.status(401).json({ error: "Invalid credentials" });
    }

    // Create session
    req.session.userId = user.id;
    req.session.email = user.email;
    req.session.role = user.role;

    res.json({
      message: "Login successful",
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
      },
    });
  } catch (error) {
    console.error("Login error:", error);
    res.status(500).json({ error: "Login failed" });
  }
});

// Logout
router.post("/logout", (req: Request, res: Response) => {
  req.session.destroy((err) => {
    if (err) {
      return res.status(500).json({ error: "Logout failed" });
    }
    res.clearCookie("sessionId");
    res.json({ message: "Logout successful" });
  });
});

// Get current user
router.get("/me", async (req: Request, res: Response) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: "Not authenticated" });
  }

  try {
    const user = await User.findById(req.session.userId);
    if (!user) {
      return res.status(404).json({ error: "User not found" });
    }

    res.json({
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
    });
  } catch (error) {
    res.status(500).json({ error: "Failed to get user" });
  }
});

export default router;
```

**Authentication Middleware:**

```typescript
// src/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from "express";

export function requireAuth(req: Request, res: Response, next: NextFunction) {
  if (!req.session.userId) {
    return res.status(401).json({ error: "Authentication required" });
  }
  next();
}

export function requireRole(role: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.session.userId) {
      return res.status(401).json({ error: "Authentication required" });
    }

    if (req.session.role !== role) {
      return res.status(403).json({ error: "Insufficient permissions" });
    }

    next();
  };
}

// Usage in routes
import { requireAuth, requireRole } from "./middleware/auth.middleware";

app.get("/api/dashboard", requireAuth, (req, res) => {
  res.json({ message: "Dashboard data" });
});

app.get("/api/admin/users", requireRole("admin"), (req, res) => {
  res.json({ message: "Admin user list" });
});
```

**App Configuration:**

```typescript
// src/app.ts
import express from "express";
import { sessionConfig } from "./config/session";
import authRoutes from "./routes/auth.routes";

const app = express();

// Middleware
app.use(express.json());
app.use(sessionConfig);

// Routes
app.use("/api/auth", authRoutes);

// Protected routes
app.get("/api/profile", requireAuth, async (req, res) => {
  // Access session data
  const userId = req.session.userId;
  // Fetch and return user profile
});

export default app;
```

---

#### Implementation: Java/Spring Boot with Sessions

**Dependencies (pom.xml):**

```xml



        org.springframework.boot
        spring-boot-starter-web




        org.springframework.boot
        spring-boot-starter-data-redis


        org.springframework.session
        spring-session-data-redis




        org.springframework.boot
        spring-boot-starter-security




        org.springframework.security
        spring-security-crypto


```

**Application Configuration:**

```yaml
# application.yml
spring:
  session:
    store-type: redis
    timeout: 7d
    redis:
      namespace: saas:sessions

  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    password: ${REDIS_PASSWORD:}

  security:
    filter:
      order: 5

server:
  servlet:
    session:
      cookie:
        name: SESSIONID
        http-only: true
        secure: true
        same-site: lax
        max-age: 604800 # 7 days
```

**User Entity:**

```java
// User.java
package com.mycompany.saas.model;

import javax.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String password;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String role = "USER";

    @Column(name = "created_at")
    private LocalDateTime createdAt = LocalDateTime.now();

    @Column(name = "last_login")
    private LocalDateTime lastLogin;

    // Getters and setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getRole() { return role; }
    public void setRole(String role) { this.role = role; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }

    public LocalDateTime getLastLogin() { return lastLogin; }
    public void setLastLogin(LocalDateTime lastLogin) { this.lastLogin = lastLogin; }
}
```

**Security Configuration:**

```java
// SecurityConfig.java
package com.mycompany.saas.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable() // Disable for API, enable for web apps
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .maximumSessions(1) // One session per user
                .maxSessionsPreventsLogin(false) // Allow new login, invalidate old
            );

        return http.build();
    }
}
```

**Auth Controller:**

```java
// AuthController.java
package com.mycompany.saas.controller;

import com.mycompany.saas.model.User;
import com.mycompany.saas.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.*;

import javax.servlet.http.HttpSession;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @PostMapping("/register")
    public ResponseEntity register(@RequestBody RegisterRequest request, HttpSession session) {
        // Check if user exists
        if (userRepository.findByEmail(request.getEmail()).isPresent()) {
            return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(Map.of("error", "Email already registered"));
        }

        // Create user
        User user = new User();
        user.setEmail(request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        user.setName(request.getName());
        user.setRole("USER");

        user = userRepository.save(user);

        // Create session
        session.setAttribute("userId", user.getId());
        session.setAttribute("email", user.getEmail());
        session.setAttribute("role", user.getRole());

        Map response = new HashMap<>();
        response.put("message", "User registered successfully");
        response.put("user", Map.of(
            "id", user.getId(),
            "email", user.getEmail(),
            "name", user.getName()
        ));

        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @PostMapping("/login")
    public ResponseEntity login(@RequestBody LoginRequest request, HttpSession session) {
        // Find user
        Optional userOpt = userRepository.findByEmail(request.getEmail());
        if (userOpt.isEmpty()) {
            return ResponseEntity
                .status(HttpStatus.UNAUTHORIZED)
                .body(Map.of("error", "Invalid credentials"));
        }

        User user = userOpt.get();

        // Verify password
        if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            return ResponseEntity
                .status(HttpStatus.UNAUTHORIZED)
                .body(Map.of("error", "Invalid credentials"));
        }

        // Update last login
        user.setLastLogin(LocalDateTime.now());
        userRepository.save(user);

        // Create session
        session.setAttribute("userId", user.getId());
        session.setAttribute("email", user.getEmail());
        session.setAttribute("role", user.getRole());

        Map response = new HashMap<>();
        response.put("message", "Login successful");
        response.put("user", Map.of(
            "id", user.getId(),
            "email", user.getEmail(),
            "name", user.getName(),
            "role", user.getRole()
        ));

        return ResponseEntity.ok(response);
    }

    @PostMapping("/logout")
    public ResponseEntity logout(HttpSession session) {
        session.invalidate();
        return ResponseEntity.ok(Map.of("message", "Logout successful"));
    }

    @GetMapping("/me")
    public ResponseEntity getCurrentUser(HttpSession session) {
        String userId = (String) session.getAttribute("userId");

        if (userId == null) {
            return ResponseEntity
                .status(HttpStatus.UNAUTHORIZED)
                .body(Map.of("error", "Not authenticated"));
        }

        Optional userOpt = userRepository.findById(userId);
        if (userOpt.isEmpty()) {
            return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(Map.of("error", "User not found"));
        }

        User user = userOpt.get();
        return ResponseEntity.ok(Map.of(
            "id", user.getId(),
            "email", user.getEmail(),
            "name", user.getName(),
            "role", user.getRole()
        ));
    }
}

// DTOs
class RegisterRequest {
    private String email;
    private String password;
    private String name;

    // Getters and setters
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}

class LoginRequest {
    private String email;
    private String password;

    // Getters and setters
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}
```

**Protected Endpoint Example:**

```java
// UserController.java
package com.mycompany.saas.controller;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.servlet.http.HttpSession;
import java.util.Map;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/profile")
    public ResponseEntity getProfile(HttpSession session) {
        String userId = (String) session.getAttribute("userId");

        if (userId == null) {
            return ResponseEntity
                .status(401)
                .body(Map.of("error", "Authentication required"));
        }

        // Fetch and return user profile
        return ResponseEntity.ok(Map.of(
            "userId", userId,
            "message", "Profile data"
        ));
    }
}
```

---

#### Implementation: Go with Sessions

**Dependencies:**

```bash
go get github.com/gorilla/sessions
go get github.com/gomodule/redigo/redis
go get github.com/rbcervilla/redisstore/v9
go get golang.org/x/crypto/bcrypt
```

**Session Configuration:**

```go
// config/session.go
package config

import (
	"os"
	"time"

	"github.com/gomodule/redigo/redis"
	"github.com/gorilla/sessions"
	"github.com/rbcervilla/redisstore/v9"
)

var Store *redisstore.RedisStore

func InitSessionStore() error {
	// Create Redis pool
	pool := &redis.Pool{
		MaxIdle:     10,
		MaxActive:   100,
		IdleTimeout: 240 * time.Second,
		Dial: func() (redis.Conn, error) {
			return redis.Dial("tcp", os.Getenv("REDIS_HOST")+":"+os.Getenv("REDIS_PORT"))
		},
	}

	// Create store
	store, err := redisstore.NewRedisStore(pool)
	if err != nil {
		return err
	}

	// Configure session options
	store.Options = &sessions.Options{
		Path:     "/",
		MaxAge:   60 * 60 * 24 * 7, // 7 days
		HttpOnly: true,
		Secure:   os.Getenv("ENV") == "production",
		SameSite: http.SameSiteLaxMode,
	}

	Store = store
	return nil
}
```

**User Model:**

```go
// models/user.go
package models

import (
	"time"

	"golang.org/x/crypto/bcrypt"
)

type User struct {
	ID        string    `json:"id" db:"id"`
	Email     string    `json:"email" db:"email"`
	Password  string    `json:"-" db:"password"` // Never serialize password
	Name      string    `json:"name" db:"name"`
	Role      string    `json:"role" db:"role"`
	CreatedAt time.Time `json:"created_at" db:"created_at"`
	LastLogin *time.Time `json:"last_login" db:"last_login"`
}

func (u *User) SetPassword(password string) error {
	hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return err
	}
	u.Password = string(hashedPassword)
	return nil
}

func (u *User) CheckPassword(password string) bool {
	err := bcrypt.CompareHashAndPassword([]byte(u.Password), []byte(password))
	return err == nil
}
```

**Auth Handler:**

```go
// handlers/auth.go
package handlers

import (
	"database/sql"
	"encoding/json"
	"net/http"
	"time"

	"github.com/google/uuid"
	"myapp/config"
	"myapp/models"
	"myapp/repository"
)

type AuthHandler struct {
	userRepo *repository.UserRepository
}

func NewAuthHandler(userRepo *repository.UserRepository) *AuthHandler {
	return &AuthHandler{userRepo: userRepo}
}

type RegisterRequest struct {
	Email    string `json:"email"`
	Password string `json:"password"`
	Name     string `json:"name"`
}

type LoginRequest struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
	var req RegisterRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	// Check if user exists
	_, err := h.userRepo.FindByEmail(req.Email)
	if err == nil {
		http.Error(w, "Email already registered", http.StatusBadRequest)
		return
	}

	// Create user
	user := &models.User{
		ID:        uuid.New().String(),
		Email:     req.Email,
		Name:      req.Name,
		Role:      "user",
		CreatedAt: time.Now(),
	}

	if err := user.SetPassword(req.Password); err != nil {
		http.Error(w, "Failed to hash password", http.StatusInternalServerError)
		return
	}

	if err := h.userRepo.Create(user); err != nil {
		http.Error(w, "Failed to create user", http.StatusInternalServerError)
		return
	}

	// Create session
	session, _ := config.Store.Get(r, "session")
	session.Values["userId"] = user.ID
	session.Values["email"] = user.Email
	session.Values["role"] = user.Role
	session.Save(r, w)

	w.Header().Set("Content-Type", "application/json")
	w.WriteStatus(http.StatusCreated)
	json.NewEncoder(w).Encode(map[string]interface{}{
		"message": "User registered successfully",
		"user": map[string]interface{}{
			"id":    user.ID,
			"email": user.Email,
			"name":  user.Name,
		},
	})
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
	var req LoginRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	// Find user
	user, err := h.userRepo.FindByEmail(req.Email)
	if err != nil {
		http.Error(w, "Invalid credentials", http.StatusUnauthorized)
		return
	}

	// Check password
	if !user.CheckPassword(req.Password) {
		http.Error(w, "Invalid credentials", http.StatusUnauthorized)
		return
	}

	// Update last login
	now := time.Now()
	user.LastLogin = &now
	h.userRepo.Update(user)

	// Create session
	session, _ := config.Store.Get(r, "session")
	session.Values["userId"] = user.ID
	session.Values["email"] = user.Email
	session.Values["role"] = user.Role
	session.Save(r, w)

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"message": "Login successful",
		"user": map[string]interface{}{
			"id":    user.ID,
			"email": user.Email,
			"name":  user.Name,
			"role":  user.Role,
		},
	})
}

func (h *AuthHandler) Logout(w http.ResponseWriter, r *http.Request) {
	session, _ := config.Store.Get(r, "session")
	session.Options.MaxAge = -1
	session.Save(r, w)

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{
		"message": "Logout successful",
	})
}

func (h *AuthHandler) GetCurrentUser(w http.ResponseWriter, r *http.Request) {
	session, _ := config.Store.Get(r, "session")
	userID, ok := session.Values["userId"].(string)

	if !ok {
		http.Error(w, "Not authenticated", http.StatusUnauthorized)
		return
	}

	user, err := h.userRepo.FindByID(userID)
	if err != nil {
		http.Error(w, "User not found", http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(user)
}
```

**Auth Middleware:**

```go
// middleware/auth.go
package middleware

import (
	"context"
	"net/http"

	"myapp/config"
)

type contextKey string

const UserIDKey contextKey = "userId"

func RequireAuth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		session, _ := config.Store.Get(r, "session")
		userID, ok := session.Values["userId"].(string)

		if !ok {
			http.Error(w, "Authentication required", http.StatusUnauthorized)
			return
		}

		// Add user ID to context
		ctx := context.WithValue(r.Context(), UserIDKey, userID)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func RequireRole(role string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			session, _ := config.Store.Get(r, "session")
			userRole, ok := session.Values["role"].(string)

			if !ok {
				http.Error(w, "Authentication required", http.StatusUnauthorized)
				return
			}

			if userRole != role {
				http.Error(w, "Insufficient permissions", http.StatusForbidden)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}
```

**Router Setup:**

```go
// main.go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/mux"
	"myapp/config"
	"myapp/handlers"
	"myapp/middleware"
	"myapp/repository"
)

	// Initialize session store
	if err := config.InitSessionStore(); err != nil {
		log.Fatal("Failed to initialize session store:", err)
	}

	// Initialize database
	db := config.InitDB()
	defer db.Close()

	// Initialize repositories
	userRepo := repository.NewUserRepository(db)

	// Initialize handlers
	authHandler := handlers.NewAuthHandler(userRepo)

	// Setup router
	r := mux.NewRouter()

	// Auth routes (public)
	r.HandleFunc("/api/auth/register", authHandler.Register).Methods("POST")
	r.HandleFunc("/api/auth/login", authHandler.Login).Methods("POST")
	r.HandleFunc("/api/auth/logout", authHandler.Logout).Methods("POST")
	r.HandleFunc("/api/auth/me", authHandler.GetCurrentUser).Methods("GET")

	// Protected routes
	api := r.PathPrefix("/api").Subrouter()
	api.Use(middleware.RequireAuth)

	api.HandleFunc("/profile", func(w http.ResponseWriter, r *http.Request) {
		// Access user ID from context
		userID := r.Context().Value(middleware.UserIDKey).(string)
		w.Write([]byte("Profile for user: " + userID))
	}).Methods("GET")

	// Admin routes
	admin := r.PathPrefix("/api/admin").Subrouter()
	admin.Use(middleware.RequireAuth)
	admin.Use(middleware.RequireRole("admin"))

	admin.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Admin user list"))
	}).Methods("GET")

	log.Println("Server starting on :8080")
	log.Fatal(http.ListenAndServe(":8080", r))
}
```

---

### 4.3 Password Security Best Practices

#### Password Hashing Algorithms

**Comparison:**

| Algorithm   | Status       | Strength    | Speed        | Recommended       |
| ----------- | ------------ | ----------- | ------------ | ----------------- |
| **MD5**     | ❌ Broken    | Very Weak   | Very Fast    | Never use         |
| **SHA-1**   | ❌ Broken    | Weak        | Very Fast    | Never use         |
| **SHA-256** | ⚠️ Not ideal | Moderate    | Fast         | Not for passwords |
| **bcrypt**  | ✅ Good      | Strong      | Slow (good!) | Recommended       |
| **Argon2**  | ✅ Best      | Very Strong | Adjustable   | Best choice       |
| **scrypt**  | ✅ Good      | Strong      | Slow         | Good alternative  |

**Why slow is good:** Slow hashing makes brute-force attacks impractical.

---

#### bcrypt Implementation

**Node.js:**

```typescript
import bcrypt from "bcrypt";

// Hash password
const SALT_ROUNDS = 10; // Higher = more secure but slower
const hashedPassword = await bcrypt.hash(plainPassword, SALT_ROUNDS);

// Verify password
const isValid = await bcrypt.compare(plainPassword, hashedPassword);

// Adjust work factor over time
// Every year or two, increase SALT_ROUNDS by 1
// Current recommendation: 10-12 rounds
```

**Java:**

```java
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();

// Hash password
String hashedPassword = encoder.encode(plainPassword);

// Verify password
boolean isValid = encoder.matches(plainPassword, hashedPassword);
```

**Go:**

```go
import "golang.org/x/crypto/bcrypt"

// Hash password
hashedPassword, err := bcrypt.GenerateFromPassword([]byte(plainPassword), bcrypt.DefaultCost)

// Verify password
err := bcrypt.CompareHashAndPassword(hashedPassword, []byte(plainPassword))
isValid := err == nil
```

---

#### Argon2 Implementation (More Secure)

**Node.js:**

```bash
npm install argon2
```

```typescript
import argon2 from "argon2";

// Hash password
const hashedPassword = await argon2.hash(plainPassword, {
  type: argon2.argon2id, // Best type
  memoryCost: 65536, // 64 MB
  timeCost: 3, // Iterations
  parallelism: 4, // Threads
});

// Verify password
const isValid = await argon2.verify(hashedPassword, plainPassword);

// Argon2 automatically handles salt
```

**Java:**

```xml

    de.mkammerer
    argon2-jvm
    2.11

```

```java
import de.mkammerer.argon2.Argon2;
import de.mkammerer.argon2.Argon2Factory;

Argon2 argon2 = Argon2Factory.create(
    Argon2Factory.Argon2Types.ARGON2id,
    32, // Salt length
    64  // Hash length
);

// Hash password
String hashedPassword = argon2.hash(
    3,      // Iterations
    65536,  // Memory (KB)
    4,      // Parallelism
    plainPassword.toCharArray()
);

// Verify password
boolean isValid = argon2.verify(hashedPassword, plainPassword.toCharArray());

// Always clean up
argon2.wipeArray(plainPassword.toCharArray());
```

**Go:**

```bash
go get golang.org/x/crypto/argon2
```

```go
import (
	"crypto/rand"
	"crypto/subtle"
	"encoding/base64"
	"fmt"
	"strings"

	"golang.org/x/crypto/argon2"
)

type Argon2Params struct {
	Memory      uint32
	Iterations  uint32
	Parallelism uint8
	SaltLength  uint32
	KeyLength   uint32
}

var DefaultParams = &Argon2Params{
	Memory:      64 * 1024, // 64 MB
	Iterations:  3,
	Parallelism: 4,
	SaltLength:  16,
	KeyLength:   32,
}

func HashPassword(password string) (string, error) {
	// Generate salt
	salt := make([]byte, DefaultParams.SaltLength)
	if _, err := rand.Read(salt); err != nil {
		return "", err
	}

	// Hash password
	hash := argon2.IDKey(
		[]byte(password),
		salt,
		DefaultParams.Iterations,
		DefaultParams.Memory,
		DefaultParams.Parallelism,
		DefaultParams.KeyLength,
	)

	// Encode to string
	b64Salt := base64.RawStdEncoding.EncodeToString(salt)
	b64Hash := base64.RawStdEncoding.EncodeToString(hash)

	encodedHash := fmt.Sprintf(
		"$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
		argon2.Version,
		DefaultParams.Memory,
		DefaultParams.Iterations,
		DefaultParams.Parallelism,
		b64Salt,
		b64Hash,
	)

	return encodedHash, nil
}

func VerifyPassword(password, encodedHash string) (bool, error) {
	// Parse encoded hash
	parts := strings.Split(encodedHash, "$")
	if len(parts) != 6 {
		return false, fmt.Errorf("invalid hash format")
	}

	var memory, iterations uint32
	var parallelism uint8
	_, err := fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d", &memory, &iterations, &parallelism)
	if err != nil {
		return false, err
	}

	salt, err := base64.RawStdEncoding.DecodeString(parts[4])
	if err != nil {
		return false, err
	}

	decodedHash, err := base64.RawStdEncoding.DecodeString(parts[5])
	if err != nil {
		return false, err
	}

	// Hash provided password with same parameters
	comparisonHash := argon2.IDKey(
		[]byte(password),
		salt,
		iterations,
		memory,
		parallelism,
		uint32(len(decodedHash)),
	)

	// Use constant-time comparison
	return subtle.ConstantTimeCompare(decodedHash, comparisonHash) == 1, nil
}
```

---

#### Password Validation

**Strong Password Requirements:**

```typescript
// src/utils/password-validator.ts
export interface PasswordRequirements {
  minLength: number;
  requireUppercase: boolean;
  requireLowercase: boolean;
  requireNumbers: boolean;
  requireSpecialChars: boolean;
  preventCommon: boolean;
}

const DEFAULT_REQUIREMENTS: PasswordRequirements = {
  minLength: 8,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: true,
  preventCommon: true,
};

// Common passwords to reject (from "Have I Been Pwned" list)
const COMMON_PASSWORDS = [
  "password",
  "123456",
  "12345678",
  "qwerty",
  "abc123",
  "monkey",
  "1234567",
  "letmein",
  "trustno1",
  "dragon",
  "baseball",
  "iloveyou",
  "master",
  "sunshine",
  "ashley",
  // ... add more
];

export interface ValidationResult {
  valid: boolean;
  errors: string[];
  strength: "weak" | "fair" | "good" | "strong";
}

export function validatePassword(
  password: string,
  requirements: PasswordRequirements = DEFAULT_REQUIREMENTS
): ValidationResult {
  const errors: string[] = [];

  // Check length
  if (password.length < requirements.minLength) {
    errors.push(
      `Password must be at least ${requirements.minLength} characters`
    );
  }

  // Check uppercase
  if (requirements.requireUppercase && !/[A-Z]/.test(password)) {
    errors.push("Password must contain at least one uppercase letter");
  }

  // Check lowercase
  if (requirements.requireLowercase && !/[a-z]/.test(password)) {
    errors.push("Password must contain at least one lowercase letter");
  }

  // Check numbers
  if (requirements.requireNumbers && !/[0-9]/.test(password)) {
    errors.push("Password must contain at least one number");
  }

  // Check special characters
  if (
    requirements.requireSpecialChars &&
    !/[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/.test(password)
  ) {
    errors.push("Password must contain at least one special character");
  }

  // Check against common passwords
  if (
    requirements.preventCommon &&
    COMMON_PASSWORDS.includes(password.toLowerCase())
  ) {
    errors.push("Password is too common");
  }

  // Calculate strength
  const strength = calculatePasswordStrength(password);

  return {
    valid: errors.length === 0,
    errors,
    strength,
  };
}

function calculatePasswordStrength(
  password: string
): "weak" | "fair" | "good" | "strong" {
  let score = 0;

  // Length bonus
  if (password.length >= 8) score++;
  if (password.length >= 12) score++;
  if (password.length >= 16) score++;

  // Character variety
  if (/[a-z]/.test(password)) score++;
  if (/[A-Z]/.test(password)) score++;
  if (/[0-9]/.test(password)) score++;
  if (/[^a-zA-Z0-9]/.test(password)) score++;

  // Patterns (penalize)
  if (/(.)\1{2,}/.test(password)) score--; // Repeated characters
  if (/012|123|234|345|456|567|678|789|890/.test(password)) score--; // Sequential numbers
  if (
    /abc|bcd|cde|def|efg|fgh|ghi|hij|ijk|jkl|klm|lmn|mno|nop|opq|pqr|qrs|rst|stu|tuv|uvw|vwx|wxy|xyz/i.test(
      password
    )
  )
    score--; // Sequential letters

  if (score <= 3) return "weak";
  if (score <= 5) return "fair";
  if (score <= 7) return "good";
  return "strong";
}

// Usage
const result = validatePassword("MyP@ssw0rd123");
if (!result.valid) {
  console.error("Password validation failed:", result.errors);
} else {
  console.log("Password strength:", result.strength);
}
```

---

#### Rate Limiting for Login Attempts

**Prevent Brute Force Attacks:**

```typescript
// src/middleware/rate-limit.middleware.ts
import rateLimit from "express-rate-limit";
import RedisStore from "rate-limit-redis";
import Redis from "ioredis";

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: parseInt(process.env.REDIS_PORT || "6379"),
});

// Login rate limiter: 5 attempts per 15 minutes per IP
export const loginLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: "rl:login:",
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 requests per window
  message: {
    error: "Too many login attempts, please try again later",
  },
  standardHeaders: true,
  legacyHeaders: false,
  // Skip successful requests (don't count against limit)
  skipSuccessfulRequests: true,
  // Use email + IP for tracking
  keyGenerator: (req) => {
    const email = req.body.email || "unknown";
    const ip = req.ip;
    return `${email}:${ip}`;
  },
});

// Register rate limiter: 3 registrations per hour per IP
export const registerLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: "rl:register:",
  }),
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 3,
  message: {
    error: "Too many registration attempts, please try again later",
  },
});

// Password reset rate limiter: 3 attempts per hour per email
export const passwordResetLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: "rl:password-reset:",
  }),
  windowMs: 60 * 60 * 1000,
  max: 3,
  keyGenerator: (req) => req.body.email || req.ip,
  message: {
    error: "Too many password reset attempts, please try again later",
  },
});

// Usage in routes
app.post("/api/auth/login", loginLimiter, authController.login);
app.post("/api/auth/register", registerLimiter, authController.register);
app.post(
  "/api/auth/password-reset",
  passwordResetLimiter,
  authController.resetPassword
);
```

**Account Lockout Strategy:**

```typescript
// src/services/account-lockout.service.ts
import Redis from "ioredis";

const redis = new Redis();

export class AccountLockoutService {
  private static MAX_ATTEMPTS = 5;
  private static LOCKOUT_DURATION = 30 * 60; // 30 minutes in seconds

  static async recordFailedAttempt(email: string): Promise {
    const key = `lockout:${email}`;
    const attempts = await redis.incr(key);

    if (attempts === 1) {
      // Set expiry on first attempt
      await redis.expire(key, this.LOCKOUT_DURATION);
    }

    if (attempts >= this.MAX_ATTEMPTS) {
      await this.lockAccount(email);
    }
  }

  static async resetFailedAttempts(email: string): Promise {
    await redis.del(`lockout:${email}`);
  }

  static async isAccountLocked(email: string): Promise {
    const locked = await redis.get(`locked:${email}`);
    return locked === "1";
  }

  private static async lockAccount(email: string): Promise {
    await redis.setex(`locked:${email}`, this.LOCKOUT_DURATION, "1");

    // Send email notification
    // await emailService.sendAccountLockedNotification(email);
  }

  static async getRemainingAttempts(email: string): Promise {
    const attempts = await redis.get(`lockout:${email}`);
    const currentAttempts = attempts ? parseInt(attempts) : 0;
    return Math.max(0, this.MAX_ATTEMPTS - currentAttempts);
  }
}

// Usage in login handler
async function login(req: Request, res: Response) {
  const { email, password } = req.body;

  // Check if account is locked
  if (await AccountLockoutService.isAccountLocked(email)) {
    return res.status(423).json({
      error: "Account temporarily locked due to too many failed attempts",
      lockedUntil: new Date(Date.now() + 30 * 60 * 1000),
    });
  }

  const user = await User.findByEmail(email);

  if (!user || !(await bcrypt.compare(password, user.password))) {
    await AccountLockoutService.recordFailedAttempt(email);

    const remainingAttempts = await AccountLockoutService.getRemainingAttempts(
      email
    );

    return res.status(401).json({
      error: "Invalid credentials",
      remainingAttempts,
    });
  }

  // Successful login - reset attempts
  await AccountLockoutService.resetFailedAttempts(email);

  // Continue with login...
}
```

---

## 📖 Day 2: JWT (JSON Web Tokens) Authentication

### 4.4 Understanding JWT

**What is JWT?**

A JSON Web Token is a compact, URL-safe token format used to securely transmit information between parties.

**JWT Structure:**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

[Header].[Payload].[Signature]
```

**Header:**

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload (Claims):**

```json
{
  "sub": "1234567890", // Subject (user ID)
  "name": "John Doe", // User name
  "email": "john@example.com",
  "role": "admin",
  "iat": 1516239022, // Issued at
  "exp": 1516242622 // Expiration time
}
```

**Signature:**

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

---

### 4.5 JWT vs Sessions

| Aspect          | JWT                               | Sessions                   |
| --------------- | --------------------------------- | -------------------------- |
| **Storage**     | Client-side (localStorage/cookie) | Server-side (Redis/DB)     |
| **Scalability** | Excellent (stateless)             | Requires shared storage    |
| **Revocation**  | Difficult (need blacklist)        | Easy (delete from storage) |
| **Size**        | Larger (sent with every request)  | Small (just session ID)    |
| **Security**    | Can't modify token server-side    | Full server control        |
| **Mobile/API**  | Excellent                         | Limited                    |
| **Expiration**  | Built-in                          | Managed by server          |

**When to use JWT:**

- ✅ Microservices architecture
- ✅ Mobile applications
- ✅ Single Page Applications (SPAs)
- ✅ API-first architecture
- ✅ Third-party API access

**When to use Sessions:**

- ✅ Traditional server-rendered apps
- ✅ Need instant token revocation
- ✅ Highly sensitive applications
- ✅ Simple authentication requirements

---

### 4.6 JWT Implementation

#### Node.js/Express with JWT

**Installation:**

```bash
npm install jsonwebtoken
npm install --save-dev @types/jsonwebtoken
```

**JWT Configuration:**

```typescript
// src/config/jwt.ts
export const jwtConfig = {
  accessTokenSecret: process.env.JWT_ACCESS_SECRET!,
  refreshTokenSecret: process.env.JWT_REFRESH_SECRET!,
  accessTokenExpiry: "15m", // 15 minutes
  refreshTokenExpiry: "7d", // 7 days
  issuer: "myapp.com",
  audience: "myapp-users",
};

// Validate secrets exist
if (!jwtConfig.accessTokenSecret || !jwtConfig.refreshTokenSecret) {
  throw new Error("JWT secrets must be configured");
}
```

**JWT Service:**

```typescript
// src/services/jwt.service.ts
import jwt from "jsonwebtoken";
import { jwtConfig } from "../config/jwt";

export interface TokenPayload {
  userId: string;
  email: string;
  role: string;
}

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
}

export class JWTService {
  static generateAccessToken(payload: TokenPayload): string {
    return jwt.sign(payload, jwtConfig.accessTokenSecret, {
      expiresIn: jwtConfig.accessTokenExpiry,
      issuer: jwtConfig.issuer,
      audience: jwtConfig.audience,
    });
  }

  static generateRefreshToken(payload: TokenPayload): string {
    return jwt.sign(
      { userId: payload.userId }, // Minimal payload for refresh token
      jwtConfig.refreshTokenSecret,
      {
        expiresIn: jwtConfig.refreshTokenExpiry,
        issuer: jwtConfig.issuer,
        audience: jwtConfig.audience,
      }
    );
  }

  static generateTokenPair(payload: TokenPayload): TokenPair {
    return {
      accessToken: this.generateAccessToken(payload),
      refreshToken: this.generateRefreshToken(payload),
    };
  }

  static verifyAccessToken(token: string): TokenPayload {
    try {
      const decoded = jwt.verify(token, jwtConfig.accessTokenSecret, {
        issuer: jwtConfig.issuer,
        audience: jwtConfig.audience,
      }) as TokenPayload;

      return decoded;
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new Error("Access token expired");
      }
      if (error instanceof jwt.JsonWebTokenError) {
        throw new Error("Invalid access token");
      }
      throw error;
    }
  }

  static verifyRefreshToken(token: string): { userId: string } {
    try {
      const decoded = jwt.verify(token, jwtConfig.refreshTokenSecret, {
        issuer: jwtConfig.issuer,
        audience: jwtConfig.audience,
      }) as { userId: string };

      return decoded;
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new Error("Refresh token expired");
      }
      if (error instanceof jwt.JsonWebTokenError) {
        throw new Error("Invalid refresh token");
      }
      throw error;
    }
  }

  static decodeToken(token: string): any {
    return jwt.decode(token);
  }
}
```

**Refresh Token Storage:**

```typescript
// src/models/refresh-token.model.ts
import { v4 as uuidv4 } from 'uuid';

export interface RefreshToken {
  id: string;
  userId: string;
  token: string;
  expiresAt: Date;
  createdAt: Date;
  ipAddress?: string;
  userAgent?: string;
}

// Store in database
// Schema:
CREATE TABLE refresh_tokens (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token TEXT NOT NULL UNIQUE,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  ip_address VARCHAR(45),
  user_agent TEXT,
  INDEX idx_user_id (user_id),
  INDEX idx_token (token),
  INDEX idx_expires_at (expires_at)
);
```

**Auth Controller with JWT:**

```typescript
// src/controllers/auth.controller.ts
import { Request, Response } from "express";
import bcrypt from "bcrypt";
import { User } from "../models/user.model";
import { JWTService, TokenPayload } from "../services/jwt.service";
import { RefreshTokenRepository } from "../repositories/refresh-token.repository";

export class AuthController {
  static async register(req: Request, res: Response) {
    try {
      const { email, password, name } = req.body;

      // Validate input
      if (!email || !password || !name) {
        return res.status(400).json({ error: "Missing required fields" });
      }

      // Check if user exists
      const existingUser = await User.findByEmail(email);
      if (existingUser) {
        return res.status(400).json({ error: "Email already registered" });
      }

      // Hash password
      const hashedPassword = await bcrypt.hash(password, 10);

      // Create user
      const user = await User.create({
        email,
        password: hashedPassword,
        name,
        role: "user",
      });

      // Generate tokens
      const tokenPayload: TokenPayload = {
        userId: user.id,
        email: user.email,
        role: user.role,
      };

      const { accessToken, refreshToken } =
        JWTService.generateTokenPair(tokenPayload);

      // Store refresh token
      await RefreshTokenRepository.create({
        userId: user.id,
        token: refreshToken,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
        ipAddress: req.ip,
        userAgent: req.headers["user-agent"],
      });

      res.status(201).json({
        message: "User registered successfully",
        user: {
          id: user.id,
          email: user.email,
          name: user.name,
        },
        accessToken,
        refreshToken,
      });
    } catch (error) {
      console.error("Registration error:", error);
      res.status(500).json({ error: "Registration failed" });
    }
  }

  static async login(req: Request, res: Response) {
    try {
      const { email, password } = req.body;

      // Find user
      const user = await User.findByEmail(email);
      if (!user) {
        return res.status(401).json({ error: "Invalid credentials" });
      }

      // Verify password
      const isValidPassword = await bcrypt.compare(password, user.password);
      if (!isValidPassword) {
        return res.status(401).json({ error: "Invalid credentials" });
      }

      // Generate tokens
      const tokenPayload: TokenPayload = {
        userId: user.id,
        email: user.email,
        role: user.role,
      };

      const { accessToken, refreshToken } =
        JWTService.generateTokenPair(tokenPayload);

      // Store refresh token
      await RefreshTokenRepository.create({
        userId: user.id,
        token: refreshToken,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
        ipAddress: req.ip,
        userAgent: req.headers["user-agent"],
      });

      // Update last login
      await User.updateLastLogin(user.id);

      res.json({
        message: "Login successful",
        user: {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role,
        },
        accessToken,
        refreshToken,
      });
    } catch (error) {
      console.error("Login error:", error);
      res.status(500).json({ error: "Login failed" });
    }
  }

  static async refreshToken(req: Request, res: Response) {
    try {
      const { refreshToken } = req.body;

      if (!refreshToken) {
        return res.status(400).json({ error: "Refresh token required" });
      }

      // Verify refresh token
      const decoded = JWTService.verifyRefreshToken(refreshToken);

      // Check if refresh token exists in database
      const storedToken = await RefreshTokenRepository.findByToken(
        refreshToken
      );
      if (!storedToken) {
        return res.status(401).json({ error: "Invalid refresh token" });
      }

      // Check if token is expired
      if (storedToken.expiresAt < new Date()) {
        await RefreshTokenRepository.delete(storedToken.id);
        return res.status(401).json({ error: "Refresh token expired" });
      }

      // Get user
      const user = await User.findById(decoded.userId);
      if (!user) {
        return res.status(404).json({ error: "User not found" });
      }

      // Generate new tokens
      const tokenPayload: TokenPayload = {
        userId: user.id,
        email: user.email,
        role: user.role,
      };

      const { accessToken, refreshToken: newRefreshToken } =
        JWTService.generateTokenPair(tokenPayload);

      // Delete old refresh token
      await RefreshTokenRepository.delete(storedToken.id);

      // Store new refresh token
      await RefreshTokenRepository.create({
        userId: user.id,
        token: newRefreshToken,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
        ipAddress: req.ip,
        userAgent: req.headers["user-agent"],
      });

      res.json({
        accessToken,
        refreshToken: newRefreshToken,
      });
    } catch (error) {
      console.error("Token refresh error:", error);
      res.status(401).json({ error: "Token refresh failed" });
    }
  }

  static async logout(req: Request, res: Response) {
    try {
      const { refreshToken } = req.body;

      if (refreshToken) {
        // Delete refresh token from database
        await RefreshTokenRepository.deleteByToken(refreshToken);
      }

      res.json({ message: "Logout successful" });
    } catch (error) {
      console.error("Logout error:", error);
      res.status(500).json({ error: "Logout failed" });
    }
  }

  static async logoutAll(req: Request, res: Response) {
    try {
      const userId = req.user!.userId; // From JWT middleware

      // Delete all refresh tokens for user
      await RefreshTokenRepository.deleteAllForUser(userId);

      res.json({ message: "Logged out from all devices" });
    } catch (error) {
      console.error("Logout all error:", error);
      res.status(500).json({ error: "Logout failed" });
    }
  }
}
```

**JWT Authentication Middleware:**

```typescript
// src/middleware/jwt-auth.middleware.ts
import { Request, Response, NextFunction } from "express";
import { JWTService, TokenPayload } from "../services/jwt.service";

// Extend Express Request type
declare global {
  namespace Express {
    interface Request {
      user?: TokenPayload;
    }
  }
}

export function authenticateJWT(
  req: Request,
  res: Response,
  next: NextFunction
) {
  // Get token from header
  const authHeader = req.headers.authorization;

  if (!authHeader) {
    return res.status(401).json({ error: "No token provided" });
  }

  // Extract token (format: "Bearer <token>")
  const token = authHeader.split(" ")[1];

  if (!token) {
    return res.status(401).json({ error: "Invalid token format" });
  }

  try {
    // Verify token
    const decoded = JWTService.verifyAccessToken(token);

    // Attach user to request
    req.user = decoded;

    next();
  } catch (error: any) {
    return res.status(401).json({
      error: error.message || "Invalid token",
    });
  }
}

export function requireRole(role: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: "Authentication required" });
    }

    if (req.user.role !== role) {
      return res.status(403).json({ error: "Insufficient permissions" });
    }

    next();
  };
}

export function requireRoles(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: "Authentication required" });
    }

    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        error: "Insufficient permissions",
        required: roles,
        current: req.user.role,
      });
    }

    next();
  };
}

// Optional authentication (doesn't fail if no token)
export function optionalAuth(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;

  if (!authHeader) {
    return next();
  }

  const token = authHeader.split(" ")[1];

  if (!token) {
    return next();
  }

  try {
    const decoded = JWTService.verifyAccessToken(token);
    req.user = decoded;
  } catch (error) {
    // Ignore errors, just don't set user
  }

  next();
}
```

**Routes Setup:**

```typescript
// src/routes/index.ts
import express from "express";
import { AuthController } from "../controllers/auth.controller";
import {
  authenticateJWT,
  requireRole,
  requireRoles,
} from "../middleware/jwt-auth.middleware";

const router = express.Router();

// Public routes
router.post("/auth/register", AuthController.register);
router.post("/auth/login", AuthController.login);
router.post("/auth/refresh", AuthController.refreshToken);

// Protected routes (require authentication)
router.post("/auth/logout", authenticateJWT, AuthController.logout);
router.post("/auth/logout-all", authenticateJWT, AuthController.logoutAll);

router.get("/profile", authenticateJWT, (req, res) => {
  res.json({ user: req.user });
});

// Admin only routes
router.get(
  "/admin/users",
  authenticateJWT,
  requireRole("admin"),
  (req, res) => {
    res.json({ message: "Admin user list" });
  }
);

// Multiple roles allowed
router.get(
  "/dashboard",
  authenticateJWT,
  requireRoles("admin", "manager"),
  (req, res) => {
    res.json({ message: "Dashboard data" });
  }
);

export default router;
```

---

#### Java/Spring Boot with JWT

**Dependencies (pom.xml):**

```xml



        io.jsonwebtoken
        jjwt-api
        0.12.3


        io.jsonwebtoken
        jjwt-impl
        0.12.3
        runtime


        io.jsonwebtoken
        jjwt-jackson
        0.12.3
        runtime


```

**JWT Configuration:**

```java
// JwtConfig.java
package com.mycompany.saas.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConfigurationProperties(prefix = "jwt")
public class JwtConfig {

    private String accessSecret;
    private String refreshSecret;
    private long accessExpiration = 900000; // 15 minutes
    private long refreshExpiration = 604800000; // 7 days
    private String issuer = "myapp.com";
    private String audience = "myapp-users";

    // Getters and setters
    public String getAccessSecret() { return accessSecret; }
    public void setAccessSecret(String accessSecret) { this.accessSecret = accessSecret; }

    public String getRefreshSecret() { return refreshSecret; }
    public void setRefreshSecret(String refreshSecret) { this.refreshSecret = refreshSecret; }

    public long getAccessExpiration() { return accessExpiration; }
    public void setAccessExpiration(long accessExpiration) { this.accessExpiration = accessExpiration; }

    public long getRefreshExpiration() { return refreshExpiration; }
    public void setRefreshExpiration(long refreshExpiration) { this.refreshExpiration = refreshExpiration; }

    public String getIssuer() { return issuer; }
    public void setIssuer(String issuer) { this.issuer = issuer; }

    public String getAudience() { return audience; }
    public void setAudience(String audience) { this.audience = audience; }
}
```

**application.yml:**

```yaml
jwt:
  access-secret: ${JWT_ACCESS_SECRET}
  refresh-secret: ${JWT_REFRESH_SECRET}
  access-expiration: 900000 # 15 minutes
  refresh-expiration: 604800000 # 7 days
  issuer: myapp.com
  audience: myapp-users
```

**JWT Service:**

```java
// JwtService.java
package com.mycompany.saas.service;

import com.mycompany.saas.config.JwtConfig;
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.Map;

@Service
public class JwtService {

    @Autowired
    private JwtConfig jwtConfig;

    private SecretKey getAccessSecretKey() {
        return Keys.hmacShaKeyFor(jwtConfig.getAccessSecret().getBytes(StandardCharsets.UTF_8));
    }

    private SecretKey getRefreshSecretKey() {
        return Keys.hmacShaKeyFor(jwtConfig.getRefreshSecret().getBytes(StandardCharsets.UTF_8));
    }

    public String generateAccessToken(String userId, String email, String role) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtConfig.getAccessExpiration());

        return Jwts.builder()
                .setSubject(userId)
                .claim("email", email)
                .claim("role", role)
                .setIssuer(jwtConfig.getIssuer())
                .setAudience(jwtConfig.getAudience())
                .setIssuedAt(now)
                .setExpiration(expiryDate)
                .signWith(getAccessSecretKey(), SignatureAlgorithm.HS512)
                .compact();
    }

    public String generateRefreshToken(String userId) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtConfig.getRefreshExpiration());

        return Jwts.builder()
                .setSubject(userId)
                .setIssuer(jwtConfig.getIssuer())
                .setAudience(jwtConfig.getAudience())
                .setIssuedAt(now)
                .setExpiration(expiryDate)
                .signWith(getRefreshSecretKey(), SignatureAlgorithm.HS512)
                .compact();
    }

    public Claims validateAccessToken(String token) {
        try {
            return Jwts.parserBuilder()
                    .setSigningKey(getAccessSecretKey())
                    .requireIssuer(jwtConfig.getIssuer())
                    .requireAudience(jwtConfig.getAudience())
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (ExpiredJwtException e) {
            throw new RuntimeException("Access token expired");
        } catch (JwtException e) {
            throw new RuntimeException("Invalid access token");
        }
    }

    public Claims validateRefreshToken(String token) {
        try {
            return Jwts.parserBuilder()
                    .setSigningKey(getRefreshSecretKey())
                    .requireIssuer(jwtConfig.getIssuer())
                    .requireAudience(jwtConfig.getAudience())
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (ExpiredJwtException e) {
            throw new RuntimeException("Refresh token expired");
        } catch (JwtException e) {
            throw new RuntimeException("Invalid refresh token");
        }
    }

    public String getUserIdFromToken(String token) {
        Claims claims = validateAccessToken(token);
        return claims.getSubject();
    }

    public String getRoleFromToken(String token) {
        Claims claims = validateAccessToken(token);
        return claims.get("role", String.class);
    }
}
```

**JWT Authentication Filter:**

```java
// JwtAuthenticationFilter.java
package com.mycompany.saas.security;

import com.mycompany.saas.service.JwtService;
import io.jsonwebtoken.Claims;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Collections;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtService jwtService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        try {
            String jwt = extractJwtFromRequest(request);

            if (jwt != null) {
                Claims claims = jwtService.validateAccessToken(jwt);
                String userId = claims.getSubject();
                String role = claims.get("role", String.class);

                // Create authentication
                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userId,
                        null,
                        Collections.singletonList(new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                    );

                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                // Set authentication in context
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception e) {
            logger.error("Cannot set user authentication", e);
        }

        filterChain.doFilter(request, response);
    }

    private String extractJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");

        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }

        return null;
    }
}
```

**Security Configuration:**

```java
// SecurityConfig.java
package com.mycompany.saas.config;

import com.mycompany.saas.security.JwtAuthenticationFilter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Autowired
    private JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

**Auth Controller:**

```java
// AuthController.java
package com.mycompany.saas.controller;

import com.mycompany.saas.dto.LoginRequest;
import com.mycompany.saas.dto.RegisterRequest;
import com.mycompany.saas.dto.TokenResponse;
import com.mycompany.saas.model.User;
import com.mycompany.saas.service.AuthService;
import com.mycompany.saas.service.JwtService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    private AuthService authService;

    @Autowired
    private JwtService jwtService;

    @PostMapping("/register")
    public ResponseEntity register(@RequestBody RegisterRequest request) {
        try {
            User user = authService.register(request);

            String accessToken = jwtService.generateAccessToken(
                user.getId(),
                user.getEmail(),
                user.getRole()
            );
            String refreshToken = jwtService.generateRefreshToken(user.getId());

            return ResponseEntity.ok(new TokenResponse(
                accessToken,
                refreshToken,
                "Bearer",
                Map.of(
                    "id", user.getId(),
                    "email", user.getEmail(),
                    "name", user.getName(),
                    "role", user.getRole()
                )
            ));
        } catch (Exception e) {
            return ResponseEntity.badRequest().body(Map.of("error", e.getMessage()));
        }
    }

    @PostMapping("/login")
    public ResponseEntity login(@RequestBody LoginRequest request) {
        try {
            User user = authService.login(request);

            String accessToken = jwtService.generateAccessToken(
                user.getId(),
                user.getEmail(),
                user.getRole()
            );
            String refreshToken = jwtService.generateRefreshToken(user.getId());

            return ResponseEntity.ok(new TokenResponse(
                accessToken,
                refreshToken,
                "Bearer",
                Map.of(
                    "id", user.getId(),
                    "email", user.getEmail(),
                    "name", user.getName(),
                    "role", user.getRole()
                )
            ));
        } catch (Exception e) {
            return ResponseEntity.status(401).body(Map.of("error", e.getMessage()));
        }
    }

    @PostMapping("/refresh")
    public ResponseEntity refreshToken(@RequestBody Map request) {
        try {
            String refreshToken = request.get("refreshToken");

            // Validate refresh token and get user
            User user = authService.refreshToken(refreshToken);

            String newAccessToken = jwtService.generateAccessToken(
                user.getId(),
                user.getEmail(),
                user.getRole()
            );
            String newRefreshToken = jwtService.generateRefreshToken(user.getId());

            return ResponseEntity.ok(new TokenResponse(
                newAccessToken,
                newRefreshToken,
                "Bearer",
                null
            ));
        } catch (Exception e) {
            return ResponseEntity.status(401).body(Map.of("error", e.getMessage()));
        }
    }

    @PostMapping("/logout")
    public ResponseEntity logout(@RequestBody Map request, Authentication authentication) {
        try {
            String refreshToken = request.get("refreshToken");
            authService.logout(refreshToken);

            return ResponseEntity.ok(Map.of("message", "Logout successful"));
        } catch (Exception e) {
            return ResponseEntity.status(500).body(Map.of("error", e.getMessage()));
        }
    }

    @GetMapping("/me")
    public ResponseEntity getCurrentUser(Authentication authentication) {
        String userId = (String) authentication.getPrincipal();
        User user = authService.getUserById(userId);

        return ResponseEntity.ok(Map.of(
            "id", user.getId(),
            "email", user.getEmail(),
            "name", user.getName(),
            "role", user.getRole()
        ));
    }
}
```

**DTOs:**

```java
// TokenResponse.java
package com.mycompany.saas.dto;

import java.util.Map;

public class TokenResponse {
    private String accessToken;
    private String refreshToken;
    private String tokenType;
    private Map user;

    public TokenResponse(String accessToken, String refreshToken, String tokenType, Map user) {
        this.accessToken = accessToken;
        this.refreshToken = refreshToken;
        this.tokenType = tokenType;
        this.user = user;
    }

    // Getters and setters
    public String getAccessToken() { return accessToken; }
    public void setAccessToken(String accessToken) { this.accessToken = accessToken; }

    public String getRefreshToken() { return refreshToken; }
    public void setRefreshToken(String refreshToken) { this.refreshToken = refreshToken; }

    public String getTokenType() { return tokenType; }
    public void setTokenType(String tokenType) { this.tokenType = tokenType; }

    public Map getUser() { return user; }
    public void setUser(Map user) { this.user = user; }
}
```

---

#### Go with JWT

**Dependencies:**

```bash
go get github.com/golang-jwt/jwt/v5
go get github.com/google/uuid
```

**JWT Configuration:**

```go
// config/jwt.go
package config

import (
	"os"
	"time"
)

type JWTConfig struct {
	AccessSecret      string
	RefreshSecret     string
	AccessExpiration  time.Duration
	RefreshExpiration time.Duration
	Issuer            string
	Audience          string
}

var JWT = &JWTConfig{
	AccessSecret:      os.Getenv("JWT_ACCESS_SECRET"),
	RefreshSecret:     os.Getenv("JWT_REFRESH_SECRET"),
	AccessExpiration:  15 * time.Minute,
	RefreshExpiration: 7 * 24 * time.Hour,
	Issuer:            "myapp.com",
	Audience:          "myapp-users",
}

func InitJWT() error {
	if JWT.AccessSecret == "" || JWT.RefreshSecret == "" {
		return fmt.Errorf("JWT secrets must be configured")
	}
	return nil
}
```

**JWT Service:**

```go
// services/jwt.go
package services

import (
	"errors"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"myapp/config"
)

type TokenClaims struct {
	UserID string `json:"userId"`
	Email  string `json:"email"`
	Role   string `json:"role"`
	jwt.RegisteredClaims
}

type RefreshTokenClaims struct {
	UserID string `json:"userId"`
	jwt.RegisteredClaims
}

type TokenPair struct {
	AccessToken  string `json:"accessToken"`
	RefreshToken string `json:"refreshToken"`
}

func GenerateAccessToken(userID, email, role string) (string, error) {
	claims := TokenClaims{
		UserID: userID,
		Email:  email,
		Role:   role,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(config.JWT.AccessExpiration)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
			Issuer:    config.JWT.Issuer,
			Audience:  jwt.ClaimStrings{config.JWT.Audience},
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS512, claims)
	return token.SignedString([]byte(config.JWT.AccessSecret))
}

func GenerateRefreshToken(userID string) (string, error) {
	claims := RefreshTokenClaims{
		UserID: userID,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(config.JWT.RefreshExpiration)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
			Issuer:    config.JWT.Issuer,
			Audience:  jwt.ClaimStrings{config.JWT.Audience},
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS512, claims)
	return token.SignedString([]byte(config.JWT.RefreshSecret))
}

func GenerateTokenPair(userID, email, role string) (*TokenPair, error) {
	accessToken, err := GenerateAccessToken(userID, email, role)
	if err != nil {
		return nil, err
	}

	refreshToken, err := GenerateRefreshToken(userID)
	if err != nil {
		return nil, err
	}

	return &TokenPair{
		AccessToken:  accessToken,
		RefreshToken: refreshToken,
	}, nil
}

func ValidateAccessToken(tokenString string) (*TokenClaims, error) {
	token, err := jwt.ParseWithClaims(
		tokenString,
		&TokenClaims{},
		func(token *jwt.Token) (interface{}, error) {
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, errors.New("unexpected signing method")
			}
			return []byte(config.JWT.AccessSecret), nil
		},
	)

	if err != nil {
		return nil, err
	}

	if claims, ok := token.Claims.(*TokenClaims); ok && token.Valid {
		return claims, nil
	}

	return nil, errors.New("invalid token")
}

func ValidateRefreshToken(tokenString string) (*RefreshTokenClaims, error) {
	token, err := jwt.ParseWithClaims(
		tokenString,
		&RefreshTokenClaims{},
		func(token *jwt.Token) (interface{}, error) {
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, errors.New("unexpected signing method")
			}
			return []byte(config.JWT.RefreshSecret), nil
		},
	)

	if err != nil {
		return nil, err
	}

	if claims, ok := token.Claims.(*RefreshTokenClaims); ok && token.Valid {
		return claims, nil
	}

	return nil, errors.New("invalid token")
}
```

**JWT Middleware:**

```go
// middleware/jwt.go
package middleware

import (
	"context"
	"net/http"
	"strings"

	"myapp/services"
)

type contextKey string

const (
	UserIDKey  contextKey = "userId"
	EmailKey   contextKey = "email"
	RoleKey    contextKey = "role"
	ClaimsKey  contextKey = "claims"
)

func JWTAuth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Get token from header
		authHeader := r.Header.Get("Authorization")

		if authHeader == "" {
			http.Error(w, "No token provided", http.StatusUnauthorized)
			return
		}

		// Extract token (format: "Bearer ")
		parts := strings.Split(authHeader, " ")
		if len(parts) != 2 || parts[0] != "Bearer" {
			http.Error(w, "Invalid token format", http.StatusUnauthorized)
			return
		}

		tokenString := parts[1]

		// Validate token
		claims, err := services.ValidateAccessToken(tokenString)
		if err != nil {
			http.Error(w, "Invalid token: "+err.Error(), http.StatusUnauthorized)
			return
		}

		// Add claims to context
		ctx := r.Context()
		ctx = context.WithValue(ctx, UserIDKey, claims.UserID)
		ctx = context.WithValue(ctx, EmailKey, claims.Email)
		ctx = context.WithValue(ctx, RoleKey, claims.Role)
		ctx = context.WithValue(ctx, ClaimsKey, claims)

		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func RequireRole(role string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			userRole, ok := r.Context().Value(RoleKey).(string)

			if !ok {
				http.Error(w, "Authentication required", http.StatusUnauthorized)
				return
			}

			if userRole != role {
				http.Error(w, "Insufficient permissions", http.StatusForbidden)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

func RequireRoles(roles ...string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			userRole, ok := r.Context().Value(RoleKey).(string)

			if !ok {
				http.Error(w, "Authentication required", http.StatusUnauthorized)
				return
			}

			allowed := false
			for _, role := range roles {
				if userRole == role {
					allowed = true
					break
				}
			}

			if !allowed {
				http.Error(w, "Insufficient permissions", http.StatusForbidden)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}
```

**Auth Handler:**

```go
// handlers/auth.go
package handlers

import (
	"encoding/json"
	"net/http"
	"time"

	"github.com/google/uuid"
	"golang.org/x/crypto/bcrypt"
	"myapp/models"
	"myapp/repositories"
	"myapp/services"
)

type AuthHandler struct {
	userRepo         *repositories.UserRepository
	refreshTokenRepo *repositories.RefreshTokenRepository
}

func NewAuthHandler(
	userRepo *repositories.UserRepository,
	refreshTokenRepo *repositories.RefreshTokenRepository,
) *AuthHandler {
	return &AuthHandler{
		userRepo:         userRepo,
		refreshTokenRepo: refreshTokenRepo,
	}
}

type RegisterRequest struct {
	Email    string `json:"email"`
	Password string `json:"password"`
	Name     string `json:"name"`
}

type LoginRequest struct {
	Email    string `json:"email"`
	Password string `json:"password"`
}

type TokenResponse struct {
	AccessToken  string                 `json:"accessToken"`
	RefreshToken string                 `json:"refreshToken"`
	TokenType    string                 `json:"tokenType"`
	User         map[string]interface{} `json:"user,omitempty"`
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
	var req RegisterRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	// Check if user exists
	_, err := h.userRepo.FindByEmail(req.Email)
	if err == nil {
		http.Error(w, "Email already registered", http.StatusBadRequest)
		return
	}

	// Hash password
	hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
	if err != nil {
		http.Error(w, "Failed to hash password", http.StatusInternalServerError)
		return
	}

	// Create user
	user := &models.User{
		ID:        uuid.New().String(),
		Email:     req.Email,
		Password:  string(hashedPassword),
		Name:      req.Name,
		Role:      "user",
		CreatedAt: time.Now(),
	}

	if err := h.userRepo.Create(user); err != nil {
		http.Error(w, "Failed to create user", http.StatusInternalServerError)
		return
	}

	// Generate tokens
	tokenPair, err := services.GenerateTokenPair(user.ID, user.Email, user.Role)
	if err != nil {
		http.Error(w, "Failed to generate tokens", http.StatusInternalServerError)
		return
	}

	// Store refresh token
	refreshToken := &models.RefreshToken{
		ID:        uuid.New().String(),
		UserID:    user.ID,
		Token:     tokenPair.RefreshToken,
		ExpiresAt: time.Now().Add(7 * 24 * time.Hour),
		CreatedAt: time.Now(),
		IPAddress: r.RemoteAddr,
		UserAgent: r.UserAgent(),
	}

	if err := h.refreshTokenRepo.Create(refreshToken); err != nil {
		http.Error(w, "Failed to store refresh token", http.StatusInternalServerError)
		return
	}

	// Send response
	response := TokenResponse{
		AccessToken:  tokenPair.AccessToken,
		RefreshToken: tokenPair.RefreshToken,
		TokenType:    "Bearer",
		User: map[string]interface{}{
			"id":    user.ID,
			"email": user.Email,
			"name":  user.Name,
			"role":  user.Role,
		},
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(response)
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
	var req LoginRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	// Find user
	user, err := h.userRepo.FindByEmail(req.Email)
	if err != nil {
		http.Error(w, "Invalid credentials", http.StatusUnauthorized)
		return
	}

	// Verify password
	if err := bcrypt.CompareHashAndPassword([]byte(user.Password), []byte(req.Password)); err != nil {
		http.Error(w, "Invalid credentials", http.StatusUnauthorized)
		return
	}

	// Update last login
	now := time.Now()
	user.LastLogin = &now
	h.userRepo.Update(user)

	// Generate tokens
	tokenPair, err := services.GenerateTokenPair(user.ID, user.Email, user.Role)
	if err != nil {
		http.Error(w, "Failed to generate tokens", http.StatusInternalServerError)
		return
	}

	// Store refresh token
	refreshToken := &models.RefreshToken{
		ID:        uuid.New().String(),
		UserID:    user.ID,
		Token:     tokenPair.RefreshToken,
		ExpiresAt: time.Now().Add(7 * 24 * time.Hour),
		CreatedAt: time.Now(),
		IPAddress: r.RemoteAddr,
		UserAgent: r.UserAgent(),
	}

	if err := h.refreshTokenRepo.Create(refreshToken); err != nil {
		http.Error(w, "Failed to store refresh token", http.StatusInternalServerError)
		return
	}

	// Send response
	response := TokenResponse{
		AccessToken:  tokenPair.AccessToken,
		RefreshToken: tokenPair.RefreshToken,
		TokenType:    "Bearer",
		User: map[string]interface{}{
			"id":    user.ID,
			"email": user.Email,
			"name":  user.Name,
			"role":  user.Role,
		},
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response)
}

func (h *AuthHandler) RefreshToken(w http.ResponseWriter, r *http.Request) {
	var req struct {
		RefreshToken string `json:"refreshToken"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	// Validate refresh token
	claims, err := services.ValidateRefreshToken(req.RefreshToken)
	if err != nil {
		http.Error(w, "Invalid refresh token", http.StatusUnauthorized)
		return
	}

	// Check if refresh token exists in database
	storedToken, err := h.refreshTokenRepo.FindByToken(req.RefreshToken)
	if err != nil {
		http.Error(w, "Invalid refresh token", http.StatusUnauthorized)
		return
	}

	// Check if expired
	if storedToken.ExpiresAt.Before(time.Now()) {
		h.refreshTokenRepo.Delete(storedToken.ID)
		http.Error(w, "Refresh token expired", http.StatusUnauthorized)
		return
	}

	// Get user
	user, err := h.userRepo.FindByID(claims.UserID)
	if err != nil {
		http.Error(w, "User not found", http.StatusNotFound)
		return
	}

	// Generate new tokens
	tokenPair, err := services.GenerateTokenPair(user.ID, user.Email, user.Role)
	if err != nil {
		http.Error(w, "Failed to generate tokens", http.StatusInternalServerError)
		return
	}

	// Delete old refresh token
	h.refreshTokenRepo.Delete(storedToken.ID)

	// Store new refresh token
	newRefreshToken := &models.RefreshToken{
		ID:        uuid.New().String(),
		UserID:    user.ID,
		Token:     tokenPair.RefreshToken,
		ExpiresAt: time.Now().Add(7 * 24 * time.Hour),
		CreatedAt: time.Now(),
		IPAddress: r.RemoteAddr,
		UserAgent: r.UserAgent(),
	}

	if err := h.refreshTokenRepo.Create(newRefreshToken); err != nil {
		http.Error(w, "Failed to store refresh token", http.StatusInternalServerError)
		return
	}

	// Send response
	response := TokenResponse{
		AccessToken:  tokenPair.AccessToken,
		RefreshToken: tokenPair.RefreshToken,
		TokenType:    "Bearer",
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response)
}

func (h *AuthHandler) Logout(w http.ResponseWriter, r *http.Request) {
	var req struct {
		RefreshToken string `json:"refreshToken"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Invalid request", http.StatusBadRequest)
		return
	}

	// Delete refresh token
	if err := h.refreshTokenRepo.DeleteByToken(req.RefreshToken); err != nil {
		// Don't fail if token doesn't exist
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{
		"message": "Logout successful",
	})
}

func (h *AuthHandler) LogoutAll(w http.ResponseWriter, r *http.Request) {
	userID := r.Context().Value(middleware.UserIDKey).(string)

	// Delete all refresh tokens for user
	if err := h.refreshTokenRepo.DeleteAllForUser(userID); err != nil {
		http.Error(w, "Logout failed", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{
		"message": "Logged out from all devices",
	})
}

func (h *AuthHandler) GetCurrentUser(w http.ResponseWriter, r *http.Request) {
	userID := r.Context().Value(middleware.UserIDKey).(string)

	user, err := h.userRepo.FindByID(userID)
	if err != nil {
		http.Error(w, "User not found", http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"id":    user.ID,
		"email": user.Email,
		"name":  user.Name,
		"role":  user.Role,
	})
}
```

**Router Setup:**

```go
// main.go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/mux"
	"myapp/config"
	"myapp/handlers"
	"myapp/middleware"
	"myapp/repositories"
)

func main() {
	// Initialize configuration
	if err := config.InitJWT(); err != nil {
		log.Fatal("Failed to initialize JWT config:", err)
	}

	// Initialize database
	db := config.InitDB()
	defer db.Close()

	// Initialize repositories
	userRepo := repositories.NewUserRepository(db)
	refreshTokenRepo := repositories.NewRefreshTokenRepository(db)

	// Initialize handlers
	authHandler := handlers.NewAuthHandler(userRepo, refreshTokenRepo)

	// Setup router
	r := mux.NewRouter()

	// Public routes
	r.HandleFunc("/api/auth/register", authHandler.Register).Methods("POST")
	r.HandleFunc("/api/auth/login", authHandler.Login).Methods("POST")
	r.HandleFunc("/api/auth/refresh", authHandler.RefreshToken).Methods("POST")

	// Protected routes
	api := r.PathPrefix("/api").Subrouter()
	api.Use(middleware.JWTAuth)

	api.HandleFunc("/auth/logout", authHandler.Logout).Methods("POST")
	api.HandleFunc("/auth/logout-all", authHandler.LogoutAll).Methods("POST")
	api.HandleFunc("/auth/me", authHandler.GetCurrentUser).Methods("GET")

	api.HandleFunc("/profile", func(w http.ResponseWriter, r *http.Request) {
		userID := r.Context().Value(middleware.UserIDKey).(string)
		w.Write([]byte("Profile for user: " + userID))
	}).Methods("GET")

	// Admin routes
	admin := r.PathPrefix("/api/admin").Subrouter()
	admin.Use(middleware.JWTAuth)
	admin.Use(middleware.RequireRole("admin"))

	admin.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Admin user list"))
	}).Methods("GET")

	log.Println("Server starting on :8080")
	log.Fatal(http.ListenAndServe(":8080", r))
}
```

---

### 4.7 JWT Best Practices

#### 1. Token Storage

**❌ Bad - localStorage (vulnerable to XSS):**

```javascript
// Don't do this!
localStorage.setItem("token", accessToken);
```

**✅ Good - httpOnly Cookie (for web apps):**

```typescript
// Backend sets cookie
res.cookie("accessToken", token, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  maxAge: 15 * 60 * 1000,
});

// Frontend automatically sends cookie
// No JavaScript access = safer
```

**✅ Good - Memory + Refresh Token Pattern (for SPAs):**

```typescript
// Store access token in memory
let accessToken: string | null = null;

// Store refresh token in httpOnly cookie
// When access token expires, use refresh token to get new one

class TokenManager {
  private static accessToken: string | null = null;

  static setAccessToken(token: string) {
    this.accessToken = token;
  }

  static getAccessToken(): string | null {
    return this.accessToken;
  }

  static clearAccessToken() {
    this.accessToken = null;
  }
}
```

---

#### 2. Token Expiration Strategy

**Short-lived Access Tokens + Long-lived Refresh Tokens:**

```typescript
const TOKEN_STRATEGY = {
  // Access token: Short lifespan for security
  accessToken: {
    expiry: "15m", // 15 minutes
    use: "API requests",
    storage: "Memory or httpOnly cookie",
  },

  // Refresh token: Long lifespan for UX
  refreshToken: {
    expiry: "7d", // 7 days
    use: "Get new access token",
    storage: "httpOnly cookie or secure storage",
  },
};

// Automatic token refresh
async function makeAuthenticatedRequest(
  url: string,
  options: RequestInit = {}
) {
  let token = TokenManager.getAccessToken();

  // Try request with current token
  let response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      Authorization: `Bearer ${token}`,
    },
  });

  // If token expired, refresh and retry
  if (response.status === 401) {
    // Refresh token
    const newToken = await refreshAccessToken();
    TokenManager.setAccessToken(newToken);

    // Retry request
    response = await fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        Authorization: `Bearer ${newToken}`,
      },
    });
  }

  return response;
}

async function refreshAccessToken(): Promise {
  const response = await fetch("/api/auth/refresh", {
    method: "POST",
    credentials: "include", // Send refresh token cookie
  });

  if (!response.ok) {
    // Refresh token also expired - redirect to login
    window.location.href = "/login";
    throw new Error("Session expired");
  }

  const data = await response.json();
  return data.accessToken;
}
```

---

#### 3. Token Revocation (Blacklisting)

**Challenge:** JWTs are stateless - can't revoke them easily

**Solution 1: Refresh Token Rotation**

```typescript
// On every refresh, issue new refresh token and invalidate old one
// Limits damage if refresh token is stolen

async function refreshToken(oldRefreshToken: string) {
  // Validate old token
  const claims = validateRefreshToken(oldRefreshToken);

  // Delete old token from database
  await RefreshToken.delete(oldRefreshToken);

  // Generate new tokens
  const newAccessToken = generateAccessToken(claims.userId);
  const newRefreshToken = generateRefreshToken(claims.userId);

  // Store new refresh token
  await RefreshToken.create(newRefreshToken);

  return { newAccessToken, newRefreshToken };
}
```

**Solution 2: Token Blacklist (Redis)**

```typescript
import Redis from "ioredis";

const redis = new Redis();

// Blacklist a token
async function blacklistToken(token: string) {
  const decoded = jwt.decode(token) as any;
  const expiresIn = decoded.exp - Math.floor(Date.now() / 1000);

  // Store in Redis with TTL matching token expiration
  await redis.setex(`blacklist:${token}`, expiresIn, "1");
}

// Check if token is blacklisted
async function isTokenBlacklisted(token: string): Promise {
  const result = await redis.get(`blacklist:${token}`);
  return result === "1";
}

// Middleware to check blacklist
export function checkBlacklist(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const token = extractToken(req);

  if (await isTokenBlacklisted(token)) {
    return res.status(401).json({ error: "Token has been revoked" });
  }

  next();
}
```

**Solution 3: Short TTL + Version Number**

```typescript
// Add version to token
interface TokenPayload {
  userId: string;
  email: string;
  role: string;
  tokenVersion: number; // Track version
}

// Store current version in database
interface User {
  id: string;
  email: string;
  tokenVersion: number; // Increment to invalidate all tokens
}

// Validate token version
async function validateToken(token: string): Promise {
  const payload = jwt.verify(token, secret) as TokenPayload;

  // Check version
  const user = await User.findById(payload.userId);
  if (user.tokenVersion !== payload.tokenVersion) {
    throw new Error("Token version mismatch");
  }

  return payload;
}

// Revoke all tokens for user
async function revokeAllUserTokens(userId: string) {
  await User.update(userId, {
    tokenVersion: (currentVersion) => currentVersion + 1,
  });
}
```

---

#### 4. JWT Security Checklist

```typescript
// ✅ DO:
const SECURE_JWT_CONFIG = {
  // Use strong secret (256+ bits)
  secret: crypto.randomBytes(32).toString("hex"),

  // Short expiration for access tokens
  accessExpiry: "15m",

  // Include necessary claims
  claims: {
    iss: "myapp.com", // Issuer
    aud: "myapp-users", // Audience
    sub: userId, // Subject (user ID)
    iat: Date.now(), // Issued at
    exp: expiryTime, // Expiration
    jti: uniqueId, // JWT ID (for revocation)
  },

  // Use RS256 for public APIs (asymmetric)
  algorithm: "RS256",

  // Validate all claims
  validateIssuer: true,
  validateAudience: true,
  validateExpiration: true,
};

// ❌ DON'T:
const INSECURE_JWT_CONFIG = {
  secret: "mysecret", // Weak secret
  accessExpiry: "30d", // Too long
  algorithm: "none", // No signature!
  validateExpiration: false, // Security risk
};
```

**Sensitive Data in JWT:**

```typescript
// ❌ BAD: Sensitive data in JWT
const payload = {
  userId: "123",
  email: "user@example.com",
  password: "hashedPassword", // NO!
  ssn: "123-45-6789", // NO!
  creditCard: "4111-1111-1111", // NO!
  apiKeys: ["secret-key-123"], // NO!
};

// ✅ GOOD: Only non-sensitive data
const payload = {
  userId: "123",
  email: "user@example.com",
  role: "user",
  permissions: ["read", "write"], // OK if not sensitive
};
```

---

## 📖 Day 3: OAuth 2.0 & Social Login

### 4.8 Understanding OAuth 2.0

**What is OAuth 2.0?**
An authorization framework that allows third-party applications to access user data without exposing passwords.

**OAuth 2.0 Flow:**

```
1. User clicks "Login with Google"
   ↓
2. Redirect to Google's authorization page
   ↓
3. User grants permission
   ↓
4. Google redirects back with authorization code
   ↓
5. Exchange code for access token
   ↓
6. Use access token to get user info
   ↓
7. Create/login user in your system
```

**Key Terms:**

- **Resource Owner:** User
- **Client:** Your application
- **Authorization Server:** Google, GitHub, etc.
- **Resource Server:** API that has user data
- **Authorization Code:** Temporary code exchanged for token
- **Access Token:** Token to access protected resources
- **Refresh Token:** Token to get new access token

---

### 4.9 OAuth 2.0 Providers

#### Google OAuth Setup

**1. Create Google OAuth App:**

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create new project
3. Enable Google+ API
4. Create OAuth 2.0 credentials
5. Add authorized redirect URIs

**2. Configuration:**

```bash
# .env
GOOGLE_CLIENT_ID=your_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_client_secret
GOOGLE_REDIRECT_URI=http://localhost:3000/api/auth/google/callback
```

**3. Node.js Implementation:**

```bash
npm install passport passport-google-oauth20
npm install --save-dev @types/passport @types/passport-google-oauth20
```

```typescript
// src/config/passport.ts
import passport from "passport";
import { Strategy as GoogleStrategy } from "passport-google-oauth20";
import { User } from "../models/user.model";

passport.use(
  new GoogleStrategy(
    {
      clientID: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      callbackURL: process.env.GOOGLE_REDIRECT_URI!,
    },
    async (accessToken, refreshToken, profile, done) => {
      try {
        // Check if user exists
        let user = await User.findByEmail(profile.emails![0].value);

        if (!user) {
          // Create new user
          user = await User.create({
            email: profile.emails![0].value,
            name: profile.displayName,
            googleId: profile.id,
            avatar: profile.photos![0].value,
            role: "user",
            emailVerified: true, // Google emails are pre-verified
          });
        } else if (!user.googleId) {
          // Link Google account to existing user
          user = await User.update(user.id, {
            googleId: profile.id,
            avatar: profile.photos![0].value,
          });
        }

        done(null, user);
      } catch (error) {
        done(error as Error);
      }
    }
  )
);

export default passport;
```

```typescript
// src/routes/auth.routes.ts
import express from "express";
import passport from "../config/passport";
import { JWTService } from "../services/jwt.service";

const router = express.Router();

// Initiate Google OAuth
router.get(
  "/auth/google",
  passport.authenticate("google", {
    scope: ["profile", "email"],
    session: false,
  })
);

// Google OAuth callback
router.get(
  "/auth/google/callback",
  passport.authenticate("google", {
    session: false,
    failureRedirect: "/login?error=oauth_failed",
  }),
  (req, res) => {
    const user = req.user as any;

    // Generate JWT tokens
    const { accessToken, refreshToken } = JWTService.generateTokenPair({
      userId: user.id,
      email: user.email,
      role: user.role,
    });

    // Redirect to frontend with tokens
    res.redirect(
      `${process.env.FRONTEND_URL}/auth/callback?` +
        `accessToken=${accessToken}&` +
        `refreshToken=${refreshToken}`
    );
  }
);

export default router;
```

---

#### GitHub OAuth Setup

**1. Create GitHub OAuth App:**

1. Go to Settings → Developer settings → OAuth Apps
2. New OAuth App
3. Add authorization callback URL

**2. Configuration:**

```bash
GITHUB_CLIENT_ID=your_client_id
GITHUB_CLIENT_SECRET=your_client_secret
GITHUB_REDIRECT_URI=http://localhost:3000/api/auth/github/callback
```

**3. Implementation:**

```bash
npm install passport-github2
```

```typescript
// src/config/passport.ts
import { Strategy as GitHubStrategy } from "passport-github2";

passport.use(
  new GitHubStrategy(
    {
      clientID: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
      callbackURL: process.env.GITHUB_REDIRECT_URI!,
      scope: ["user:email"],
    },
    async (
      accessToken: string,
      refreshToken: string,
      profile: any,
      done: any
    ) => {
      try {
        const email = profile.emails![0].value;

        let user = await User.findByEmail(email);

        if (!user) {
          user = await User.create({
            email,
            name: profile.displayName || profile.username,
            githubId: profile.id,
            avatar: profile.photos![0].value,
            role: "user",
            emailVerified: true,
          });
        } else if (!user.githubId) {
          user = await User.update(user.id, {
            githubId: profile.id,
          });
        }

        done(null, user);
      } catch (error) {
        done(error);
      }
    }
  )
);
```

```typescript
// Routes
router.get(
  "/auth/github",
  passport.authenticate("github", { scope: ["user:email"], session: false })
);

router.get(
  "/auth/github/callback",
  passport.authenticate("github", {
    session: false,
    failureRedirect: "/login",
  }),
  (req, res) => {
    // Same as Google callback
  }
);
```

---

### 4.10 Database Schema for Social Auth

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255),  -- NULL for OAuth-only users
  name VARCHAR(255) NOT NULL,
  avatar VARCHAR(500),
  role VARCHAR(50) DEFAULT 'user',

  -- OAuth IDs
  google_id VARCHAR(255) UNIQUE,
  github_id VARCHAR(255) UNIQUE,
  facebook_id VARCHAR(255) UNIQUE,

  -- Email verification
  email_verified BOOLEAN DEFAULT FALSE,
  email_verification_token VARCHAR(255),
  email_verification_expires TIMESTAMP,

  -- Timestamps
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_login TIMESTAMP,

  -- Indexes
  INDEX idx_email (email),
  INDEX idx_google_id (google_id),
  INDEX idx_github_id (github_id)
);
```
