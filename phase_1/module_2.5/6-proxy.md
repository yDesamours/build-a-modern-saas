# Module 2.5: Core Design Patterns for SaaS Applications

---

## Part 3: Structural Patterns - Proxy Pattern

---

## 🎯 Learning Objectives

By the end of this section, you will:

- Understand what the Proxy pattern is and how it differs from Decorator
- Recognize scenarios where Proxy is the right choice
- Implement different types of proxies (Virtual, Protection, Remote)
- Apply Proxy for lazy loading, access control, and caching
- Use Proxy in Java (Spring AOP), Node.js, and Go

---

## 📖 Conceptual Overview

### What is the Proxy Pattern?

The **Proxy pattern** provides a **surrogate or placeholder** for another object to control access to it. Unlike Decorator which adds functionality, Proxy controls access and manages the lifecycle of the real object.

**Key Idea**: The proxy has the same interface as the real object but controls when and how the real object is accessed.

### Real-World Analogy

Think of a proxy as:

- **Lawyer**: Represents you in court (Protection Proxy)
- **Credit Card**: Proxy for your bank account (Protection + Virtual Proxy)
- **VPN**: Proxy for your internet connection (Remote Proxy)
- **Building Security**: Controls access to offices (Protection Proxy)

### Decorator vs Proxy

| Aspect              | Decorator                     | Proxy                                 |
| ------------------- | ----------------------------- | ------------------------------------- |
| **Intent**          | Add functionality             | Control access                        |
| **Focus**           | Enhancement                   | Management                            |
| **Object Creation** | Wraps existing object         | May create object on demand           |
| **Awareness**       | Client knows about decoration | Client thinks it's using real object  |
| **Use Case**        | Add features dynamically      | Lazy loading, access control, caching |

---

## 🤔 When to Use Proxy Pattern in SaaS

### Perfect Use Cases:

1. **Lazy Loading / Virtual Proxy** ⭐⭐⭐

   - Load expensive resources only when needed
   - Database connections
   - Large file loading
   - Heavy computations

2. **Access Control / Protection Proxy** ⭐⭐⭐

   - Permission checking
   - Authentication gates
   - Rate limiting
   - Subscription tier enforcement

3. **Remote Proxy** ⭐⭐

   - Microservices communication
   - API client wrappers
   - Network request handling

4. **Caching Proxy / Smart Reference** ⭐⭐⭐

   - Cache expensive operations
   - Reference counting
   - Copy-on-write

5. **Logging & Monitoring Proxy** ⭐⭐
   - Track method calls
   - Performance monitoring
   - Audit trails

---

## 🚫 When NOT to Use Proxy

- **Simple operations** - Don't add proxy overhead for trivial methods
- **When transparency isn't needed** - If clients need to know they're using a proxy, reconsider
- **Too many proxy layers** - Multiple proxies can hurt performance
- **When Decorator is better** - If you're adding behavior, not controlling access

---

## 🏗️ Structure

```
Subject (Interface)
    ↑
    |
RealSubject    Proxy → holds reference to RealSubject
```

**Key Elements:**

1. **Subject**: Interface defining operations
2. **RealSubject**: The actual object doing the work
3. **Proxy**: Controls access to RealSubject, same interface

---

## 💻 Implementation in Java/Spring Boot

### Scenario 1: Virtual Proxy - Lazy Loading Heavy Resources

```java
// Document.java - Subject interface
package com.saas.document;

public interface Document {
    void display();
    String getContent();
    byte[] renderPDF();
}
```

```java
// RealDocument.java - Real subject
package com.saas.document;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class RealDocument implements Document {
    private String filename;
    private byte[] content;

    public RealDocument(String filename) {
        this.filename = filename;
        loadFromDisk(); // Expensive operation!
    }

    private void loadFromDisk() {
        log.info("Loading document '{}' from disk - EXPENSIVE!", filename);
        try {
            // Simulate expensive file loading
            Thread.sleep(2000);
            this.content = ("Content of " + filename).getBytes();
            log.info("Document '{}' loaded successfully", filename);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    @Override
    public void display() {
        log.info("Displaying document: {}", filename);
    }

    @Override
    public String getContent() {
        return new String(content);
    }

    @Override
    public byte[] renderPDF() {
        log.info("Rendering PDF for: {}", filename);
        // Expensive PDF rendering
        return content;
    }
}
```

```java
// DocumentProxy.java - Virtual Proxy
package com.saas.document;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class DocumentProxy implements Document {
    private String filename;
    private RealDocument realDocument; // Lazy loaded

    public DocumentProxy(String filename) {
        this.filename = filename;
        log.info("Created proxy for document: {}", filename);
    }

    // Lazy initialization
    private RealDocument getRealDocument() {
        if (realDocument == null) {
            log.info("Lazy loading real document...");
            realDocument = new RealDocument(filename);
        }
        return realDocument;
    }

    @Override
    public void display() {
        // Don't need to load document just to display filename
        log.info("Displaying document metadata: {}", filename);
        // Only load if we actually need content
    }

    @Override
    public String getContent() {
        // Load only when content is actually needed
        return getRealDocument().getContent();
    }

    @Override
    public byte[] renderPDF() {
        // Load only when rendering is needed
        return getRealDocument().renderPDF();
    }
}
```

```java
// DocumentService.java - Usage
package com.saas.service;

import com.saas.document.Document;
import com.saas.document.DocumentProxy;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.stream.Collectors;

@Service
public class DocumentService {

    public List<Document> listDocuments(List<String> filenames) {
        // Create proxies - FAST, no loading
        return filenames.stream()
            .map(DocumentProxy::new)
            .collect(Collectors.toList());
    }

    public void displayDocumentList(List<Document> documents) {
        // Display metadata only - FAST
        documents.forEach(Document::display);
    }

    public String readDocument(Document document) {
        // Only now does the heavy loading happen
        return document.getContent();
    }
}
```

### Scenario 2: Protection Proxy - Access Control

```java
// UserRepository.java - Subject interface
package com.saas.repository;

import com.saas.model.User;
import java.util.List;

public interface UserRepository {
    User findById(Long id);
    List<User> findAll();
    void save(User user);
    void delete(Long id);
}
```

```java
// UserRepositoryImpl.java - Real subject
package com.saas.repository.impl;

import com.saas.model.User;
import com.saas.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Repository;
import javax.persistence.EntityManager;

@Repository
@RequiredArgsConstructor
@Slf4j
public class UserRepositoryImpl implements UserRepository {

    private final EntityManager entityManager;

    @Override
    public User findById(Long id) {
        log.info("Fetching user with id: {}", id);
        return entityManager.find(User.class, id);
    }

    @Override
    public List<User> findAll() {
        log.info("Fetching all users");
        return entityManager.createQuery("SELECT u FROM User u", User.class)
            .getResultList();
    }

    @Override
    public void save(User user) {
        log.info("Saving user: {}", user.getId());
        if (user.getId() == null) {
            entityManager.persist(user);
        } else {
            entityManager.merge(user);
        }
    }

    @Override
    public void delete(Long id) {
        log.info("Deleting user: {}", id);
        User user = findById(id);
        if (user != null) {
            entityManager.remove(user);
        }
    }
}
```

```java
// ProtectedUserRepository.java - Protection Proxy
package com.saas.repository.proxy;

import com.saas.model.User;
import com.saas.repository.UserRepository;
import com.saas.security.SecurityContext;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.access.AccessDeniedException;

@RequiredArgsConstructor
@Slf4j
public class ProtectedUserRepository implements UserRepository {

    private final UserRepository realRepository;
    private final SecurityContext securityContext;

    @Override
    public User findById(Long id) {
        // Check read permission
        if (!securityContext.hasPermission("USER_READ")) {
            log.warn("Access denied: USER_READ permission required");
            throw new AccessDeniedException("No permission to read users");
        }

        // Check if user can view this specific user
        if (!canAccessUser(id)) {
            log.warn("User {} cannot access user {}",
                securityContext.getCurrentUserId(), id);
            throw new AccessDeniedException("Cannot access this user");
        }

        return realRepository.findById(id);
    }

    @Override
    public List<User> findAll() {
        // Only admins can see all users
        if (!securityContext.hasRole("ADMIN")) {
            log.warn("Access denied: ADMIN role required for findAll");
            throw new AccessDeniedException("Admin role required");
        }

        return realRepository.findAll();
    }

    @Override
    public void save(User user) {
        // Check write permission
        if (!securityContext.hasPermission("USER_WRITE")) {
            log.warn("Access denied: USER_WRITE permission required");
            throw new AccessDeniedException("No permission to save users");
        }

        // Users can only update themselves unless they're admin
        if (user.getId() != null && !canModifyUser(user.getId())) {
            log.warn("User {} cannot modify user {}",
                securityContext.getCurrentUserId(), user.getId());
            throw new AccessDeniedException("Cannot modify this user");
        }

        realRepository.save(user);
    }

    @Override
    public void delete(Long id) {
        // Only admins can delete
        if (!securityContext.hasRole("ADMIN")) {
            log.warn("Access denied: ADMIN role required for delete");
            throw new AccessDeniedException("Admin role required to delete users");
        }

        realRepository.delete(id);
    }

    private boolean canAccessUser(Long userId) {
        // Users can access themselves, admins can access anyone
        return securityContext.hasRole("ADMIN") ||
               securityContext.getCurrentUserId().equals(userId);
    }

    private boolean canModifyUser(Long userId) {
        return securityContext.hasRole("ADMIN") ||
               securityContext.getCurrentUserId().equals(userId);
    }
}
```

### Scenario 3: Spring AOP as Proxy

Spring uses proxies extensively for AOP (Aspect-Oriented Programming).

```java
// CachingAspect.java - Using Spring AOP Proxy
package com.saas.aspect;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

@Aspect
@Component
@Slf4j
public class CachingAspect {

    private final Map<String, Object> cache = new ConcurrentHashMap<>();

    @Around("@annotation(cacheable)")
    public Object cacheResult(ProceedingJoinPoint joinPoint, Cacheable cacheable)
            throws Throwable {

        String key = generateKey(joinPoint);

        // Check cache
        if (cache.containsKey(key)) {
            log.info("Cache HIT for: {}", key);
            return cache.get(key);
        }

        log.info("Cache MISS for: {}", key);

        // Execute actual method
        Object result = joinPoint.proceed();

        // Store in cache
        cache.put(key, result);

        return result;
    }

    private String generateKey(ProceedingJoinPoint joinPoint) {
        StringBuilder key = new StringBuilder();
        key.append(joinPoint.getSignature().toShortString());

        for (Object arg : joinPoint.getArgs()) {
            key.append(":").append(arg);
        }

        return key.toString();
    }
}
```

```java
// Cacheable.java - Custom annotation
package com.saas.aspect;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Cacheable {
    int ttl() default 60; // seconds
}
```

```java
// ProductService.java - Using AOP Proxy
package com.saas.service;

import com.saas.aspect.Cacheable;
import com.saas.model.Product;
import org.springframework.stereotype.Service;

@Service
public class ProductService {

    @Cacheable(ttl = 300) // Spring creates a proxy for this
    public Product getProductById(Long id) {
        // Expensive operation
        return fetchProductFromDatabase(id);
    }

    private Product fetchProductFromDatabase(Long id) {
        // Database call
        return new Product(id, "Product " + id);
    }
}
```

---

## 💻 Implementation in Node.js

### Scenario 1: Virtual Proxy - Lazy Loading

```typescript
// document.ts - Subject interface
export interface IDocument {
  display(): void;
  getContent(): Promise<string>;
  renderPDF(): Promise<Buffer>;
}
```

```typescript
// realDocument.ts - Real subject
export class RealDocument implements IDocument {
  private filename: string;
  private content: string | null = null;

  constructor(filename: string) {
    this.filename = filename;
    this.loadFromDisk(); // Expensive!
  }

  private async loadFromDisk(): Promise<void> {
    console.log(`Loading document '${this.filename}' from disk - EXPENSIVE!`);

    // Simulate expensive file loading
    await new Promise((resolve) => setTimeout(resolve, 2000));

    this.content = `Content of ${this.filename}`;
    console.log(`Document '${this.filename}' loaded successfully`);
  }

  display(): void {
    console.log(`Displaying document: ${this.filename}`);
  }

  async getContent(): Promise<string> {
    if (!this.content) {
      await this.loadFromDisk();
    }
    return this.content!;
  }

  async renderPDF(): Promise<Buffer> {
    console.log(`Rendering PDF for: ${this.filename}`);
    if (!this.content) {
      await this.loadFromDisk();
    }
    return Buffer.from(this.content!);
  }
}
```

```typescript
// documentProxy.ts - Virtual Proxy
export class DocumentProxy implements IDocument {
  private filename: string;
  private realDocument: RealDocument | null = null;

  constructor(filename: string) {
    this.filename = filename;
    console.log(`Created proxy for document: ${filename}`);
  }

  private getRealDocument(): RealDocument {
    if (!this.realDocument) {
      console.log("Lazy loading real document...");
      this.realDocument = new RealDocument(this.filename);
    }
    return this.realDocument;
  }

  display(): void {
    // Don't need to load document just to display filename
    console.log(`Displaying document metadata: ${this.filename}`);
  }

  async getContent(): Promise<string> {
    // Load only when content is needed
    return this.getRealDocument().getContent();
  }

  async renderPDF(): Promise<Buffer> {
    // Load only when rendering is needed
    return this.getRealDocument().renderPDF();
  }
}
```

```typescript
// documentService.ts - Usage
export class DocumentService {
  listDocuments(filenames: string[]): IDocument[] {
    // Create proxies - FAST, no loading
    return filenames.map((filename) => new DocumentProxy(filename));
  }

  displayDocumentList(documents: IDocument[]): void {
    // Display metadata only - FAST
    documents.forEach((doc) => doc.display());
  }

  async readDocument(document: IDocument): Promise<string> {
    // Only now does the heavy loading happen
    return document.getContent();
  }
}
```

### Scenario 2: Protection Proxy with ES6 Proxy

JavaScript has built-in `Proxy` object - perfect for this pattern!

```typescript
// userRepository.ts - Real subject
import { User } from "./types";

export class UserRepository {
  private users: Map<number, User> = new Map();

  findById(id: number): User | undefined {
    console.log(`Fetching user with id: ${id}`);
    return this.users.get(id);
  }

  findAll(): User[] {
    console.log("Fetching all users");
    return Array.from(this.users.values());
  }

  save(user: User): void {
    console.log(`Saving user: ${user.id}`);
    this.users.set(user.id, user);
  }

  delete(id: number): void {
    console.log(`Deleting user: ${id}`);
    this.users.delete(id);
  }
}
```

```typescript
// protectedUserRepository.ts - Protection Proxy using ES6 Proxy
import { UserRepository } from "./userRepository";
import { SecurityContext } from "./securityContext";
import { User } from "./types";

export function createProtectedUserRepository(
  repository: UserRepository,
  securityContext: SecurityContext
): UserRepository {
  return new Proxy(repository, {
    get(target, prop, receiver) {
      const originalMethod = target[prop as keyof UserRepository];

      if (typeof originalMethod !== "function") {
        return originalMethod;
      }

      // Return wrapped method with security checks
      return function (...args: any[]) {
        // Apply security based on method name
        switch (prop) {
          case "findById":
            if (!securityContext.hasPermission("USER_READ")) {
              throw new Error("No permission to read users");
            }

            const userId = args[0] as number;
            if (!canAccessUser(userId, securityContext)) {
              throw new Error("Cannot access this user");
            }
            break;

          case "findAll":
            if (!securityContext.hasRole("ADMIN")) {
              throw new Error("Admin role required");
            }
            break;

          case "save":
            if (!securityContext.hasPermission("USER_WRITE")) {
              throw new Error("No permission to save users");
            }

            const user = args[0] as User;
            if (user.id && !canModifyUser(user.id, securityContext)) {
              throw new Error("Cannot modify this user");
            }
            break;

          case "delete":
            if (!securityContext.hasRole("ADMIN")) {
              throw new Error("Admin role required to delete users");
            }
            break;
        }

        // Call original method
        return originalMethod.apply(target, args);
      };
    },
  });
}

function canAccessUser(userId: number, context: SecurityContext): boolean {
  return context.hasRole("ADMIN") || context.getCurrentUserId() === userId;
}

function canModifyUser(userId: number, context: SecurityContext): boolean {
  return context.hasRole("ADMIN") || context.getCurrentUserId() === userId;
}
```

```typescript
// usage.ts
import { UserRepository } from "./userRepository";
import { createProtectedUserRepository } from "./protectedUserRepository";
import { SecurityContext } from "./securityContext";

const realRepository = new UserRepository();
const securityContext = new SecurityContext();

// Create protected proxy
const protectedRepository = createProtectedUserRepository(
  realRepository,
  securityContext
);

// Usage
try {
  const user = protectedRepository.findById(1); // Security check applied
  console.log(user);
} catch (error) {
  console.error("Access denied:", error.message);
}
```

### Scenario 3: Caching Proxy

```typescript
// cachingProxy.ts - Smart reference / Caching proxy
export function createCachingProxy<T extends object>(
  target: T,
  ttl: number = 60000
): T {
  const cache = new Map<string, { value: any; expiry: number }>();

  return new Proxy(target, {
    get(target, prop, receiver) {
      const originalMethod = target[prop as keyof T];

      if (typeof originalMethod !== "function") {
        return originalMethod;
      }

      return function (...args: any[]) {
        // Generate cache key
        const key = `${String(prop)}:${JSON.stringify(args)}`;

        // Check cache
        const cached = cache.get(key);
        if (cached && Date.now() < cached.expiry) {
          console.log(`Cache HIT for: ${key}`);
          return cached.value;
        }

        console.log(`Cache MISS for: ${key}`);

        // Call original method
        const result = originalMethod.apply(target, args);

        // Handle promises
        if (result instanceof Promise) {
          return result.then((value) => {
            cache.set(key, {
              value,
              expiry: Date.now() + ttl,
            });
            return value;
          });
        }

        // Cache synchronous result
        cache.set(key, {
          value: result,
          expiry: Date.now() + ttl,
        });

        return result;
      };
    },
  });
}
```

```typescript
// usage.ts
import { createCachingProxy } from "./cachingProxy";
import { UserRepository } from "./userRepository";

const repository = new UserRepository();

// Wrap with caching proxy
const cachedRepository = createCachingProxy(repository, 30000); // 30 sec TTL

// First call - cache miss
await cachedRepository.findById(1);

// Second call - cache hit!
await cachedRepository.findById(1);
```

---

## 💻 Implementation in Go

### Scenario 1: Virtual Proxy - Lazy Loading

```go
// document.go - Subject interface
package document

// Document interface
type Document interface {
    Display()
    GetContent() string
    RenderPDF() []byte
}
```

```go
// real_document.go - Real subject
package document

import (
    "fmt"
    "time"
)

// RealDocument - real subject
type RealDocument struct {
    filename string
    content  []byte
}

func NewRealDocument(filename string) *RealDocument {
    doc := &RealDocument{
        filename: filename,
    }
    doc.loadFromDisk()
    return doc
}

func (d *RealDocument) loadFromDisk() {
    fmt.Printf("Loading document '%s' from disk - EXPENSIVE!\n", d.filename)

    // Simulate expensive file loading
    time.Sleep(2 * time.Second)

    d.content = []byte("Content of " + d.filename)
    fmt.Printf("Document '%s' loaded successfully\n", d.filename)
}

func (d *RealDocument) Display() {
    fmt.Printf("Displaying document: %s\n", d.filename)
}

func (d *RealDocument) GetContent() string {
    return string(d.content)
}

func (d *RealDocument) RenderPDF() []byte {
    fmt.Printf("Rendering PDF for: %s\n", d.filename)
    return d.content
}
```

```go
// document_proxy.go - Virtual Proxy
package document

import "fmt"

// DocumentProxy - virtual proxy
type DocumentProxy struct {
    filename     string
    realDocument *RealDocument // Lazy loaded
}

func NewDocumentProxy(filename string) Document {
    fmt.Printf("Created proxy for document: %s\n", filename)
    return &DocumentProxy{
        filename: filename,
    }
}

func (p *DocumentProxy) getRealDocument() *RealDocument {
    if p.realDocument == nil {
        fmt.Println("Lazy loading real document...")
        p.realDocument = NewRealDocument(p.filename)
    }
    return p.realDocument
}

func (p *DocumentProxy) Display() {
    // Don't need to load document just to display filename
    fmt.Printf("Displaying document metadata: %s\n", p.filename)
}

func (p *DocumentProxy) GetContent() string {
    // Load only when content is needed
    return p.getRealDocument().GetContent()
}

func (p *DocumentProxy) RenderPDF() []byte {
    // Load only when rendering is needed
    return p.getRealDocument().RenderPDF()
}
```

```go
// document_service.go - Usage
package service

import "myapp/document"

type DocumentService struct{}

func (s *DocumentService) ListDocuments(filenames []string) []document.Document {
    // Create proxies - FAST, no loading
    docs := make([]document.Document, len(filenames))
    for i, filename := range filenames {
        docs[i] = document.NewDocumentProxy(filename)
    }
    return docs
}

func (s *DocumentService) DisplayDocumentList(docs []document.Document) {
    // Display metadata only - FAST
    for _, doc := range docs {
        doc.Display()
    }
}

func (s *DocumentService) ReadDocument(doc document.Document) string {
    // Only now does the heavy loading happen
    return doc.GetContent()
}
```

### Scenario 2: Protection Proxy

```go
// user_repository.go - Subject interface
package repository

import "myapp/model"

type UserRepository interface {
    FindByID(id int64) (*model.User, error)
    FindAll() ([]*model.User, error)
    Save(user *model.User) error
    Delete(id int64) error
}
```

```go
// user_repository_impl.go - Real subject
package repository

import (
    "fmt"
    "myapp/model"
)

type userRepositoryImpl struct {
    users map[int64]*model.User
}

func NewUserRepository() UserRepository {
    return &userRepositoryImpl{
        users: make(map[int64]*model.User),
    }
}

func (r *userRepositoryImpl) FindByID(id int64) (*model.User, error) {
    fmt.Printf("Fetching user with id: %d\n", id)
    user, ok := r.users[id]
    if !ok {
        return nil, fmt.Errorf("user not found: %d", id)
    }
    return user, nil
}

func (r *userRepositoryImpl) FindAll() ([]*model.User, error) {
    fmt.Println("Fetching all users")
    users := make([]*model.User, 0, len(r.users))
    for _, user := range r.users {
        users = append(users, user)
    }
    return users, nil
}

func (r *userRepositoryImpl) Save(user *model.User) error {
    fmt.Printf("Saving user: %d\n", user.ID)
    r.users[user.ID] = user
    return nil
}

func (r *userRepositoryImpl) Delete(id int64) error {
    fmt.Printf("Deleting user: %d\n", id)
    delete(r.users, id)
    return nil
}
```

```go
// protected_user_repository.go - Protection Proxy
package repository

import (
    "errors"
    "fmt"
    "myapp/model"
    "myapp/security"
)

type protectedUserRepository struct {
    real    UserRepository
    context *security.Context
}

func NewProtectedUserRepository(
    real UserRepository,
    context *security.Context,
) UserRepository {
    return &protectedUserRepository{
        real:    real,
        context: context,
    }
}

func (r *protectedUserRepository) FindByID(id int64) (*model.User, error) {
    // Check read permission
    if !r.context.HasPermission("USER_READ") {
        fmt.Println("Access denied: USER_READ permission required")
        return nil, errors.New("no permission to read users")
    }

    // Check if user can view this specific user
    if !r.canAccessUser(id) {
        fmt.Printf(
            "User %d cannot access user %d\n",
            r.context.GetCurrentUserID(),
            id,
        )
        return nil, errors.New("cannot access this user")
    }

    return r.real.FindByID(id)
}

func (r *protectedUserRepository) FindAll() ([]*model.User, error) {
    // Only admins can see all users
    if !r.context.HasRole("ADMIN") {
        fmt.Println("Access denied: ADMIN role required for FindAll")
        return nil, errors.New("admin role required")
    }

    return r.real.FindAll()
}

func (r *protectedUserRepository) Save(user *model.User) error {
    // Check write permission
    if !r.context.HasPermission("USER_WRITE") {
        fmt.Println("Access denied: USER_WRITE permission required")
        return errors.New("no permission to save users")
    }

    // Users can only update themselves unless they're admin
    if user.ID != 0 && !r.canModifyUser(user.ID) {
        fmt.Printf(
            "User %d cannot modify user %d\n",
            r.context.GetCurrentUserID(),
            user.ID,
        )
        return errors.New("cannot modify this user")
    }

    return r.real.Save(user)
}

func (r *protectedUserRepository) Delete(id int64) error {
    // Only admins can delete
    if !r.context.HasRole("ADMIN") {
        fmt.Println("Access denied: ADMIN role required for delete")
        return errors.New("admin role required to delete users")
    }

    return r.real.Delete(id)
}

func (r *protectedUserRepository) canAccessUser(userID int64) bool {
    // Users can access themsel
```

# Proxy Pattern - Best Practices & Common Pitfalls

---

## ✅ Best Practices

### 1. **Keep Proxies Transparent**

The client should not know or care if it's using a proxy or the real object.

```java
// ✅ GOOD - Client doesn't know about proxy
UserRepository repo = repositoryFactory.create(); // Could be proxy or real
User user = repo.findById(1);

// ❌ BAD - Client must know about proxy
UserRepositoryProxy proxy = new UserRepositoryProxy(realRepo);
if (proxy.isInitialized()) {
    User user = proxy.findById(1);
}
```

### 2. **Use the Same Interface**

Proxy must implement the exact same interface as the real subject.

```typescript
// ✅ GOOD - Same interface
interface IUserService {
  getUser(id: number): Promise<User>;
}

class UserService implements IUserService { ... }
class UserServiceProxy implements IUserService { ... }

// ❌ BAD - Different interfaces
class UserService {
  getUser(id: number): Promise<User> { ... }
}
class UserServiceProxy {
  getUserWithCache(id: number): Promise<User> { ... } // Different!
}
```

### 3. **Lazy Initialization Done Right**

Only initialize when absolutely necessary, but do it safely.

```go
// ✅ GOOD - Thread-safe lazy initialization
type DocumentProxy struct {
    filename string
    document *RealDocument
    mu       sync.Mutex
}

func (p *DocumentProxy) GetContent() string {
    p.mu.Lock()
    defer p.mu.Unlock()

    if p.document == nil {
        p.document = NewRealDocument(p.filename)
    }
    return p.document.GetContent()
}

// ❌ BAD - Not thread-safe
func (p *DocumentProxy) GetContent() string {
    if p.document == nil {  // Race condition!
        p.document = NewRealDocument(p.filename)
    }
    return p.document.GetContent()
}
```

### 4. **Clear Separation of Concerns**

Each proxy should have ONE responsibility.

```java
// ✅ GOOD - Single responsibility
CachingProxy cachingProxy = new CachingProxy(realService);
LoggingProxy loggingProxy = new LoggingProxy(cachingProxy);
SecurityProxy securityProxy = new SecurityProxy(loggingProxy);

// ❌ BAD - Multiple responsibilities
class CachingLoggingSecurityProxy extends AbstractProxy {
    // Too many concerns in one class!
}
```

### 5. **Handle Errors Gracefully**

Proxies should not hide errors from the real subject.

```typescript
// ✅ GOOD - Propagate errors
class CachingProxy implements IService {
  async getData(id: number): Promise<Data> {
    try {
      return this.getFromCache(id);
    } catch (cacheError) {
      console.warn("Cache error:", cacheError);
      // Fall back to real service
      try {
        return await this.realService.getData(id);
      } catch (serviceError) {
        // Propagate the real error
        throw serviceError;
      }
    }
  }
}

// ❌ BAD - Swallow errors
class CachingProxy implements IService {
  async getData(id: number): Promise<Data> {
    try {
      return await this.realService.getData(id);
    } catch (error) {
      return null; // Silently fails!
    }
  }
}
```

### 6. **Use Composition Over Inheritance**

Prefer holding a reference to the real object rather than extending it.

```java
// ✅ GOOD - Composition
public class CachingProxy implements UserService {
    private final UserService realService; // Composition

    public CachingProxy(UserService realService) {
        this.realService = realService;
    }
}

// ❌ BAD - Inheritance (tight coupling)
public class CachingProxy extends UserServiceImpl {
    // Tightly coupled to concrete implementation
}
```

### 7. **Document Proxy Behavior**

Make it clear what the proxy does and when it's used.

```typescript
/**
 * CachingProxy for UserService
 *
 * Caches user data for 60 seconds to reduce database load.
 * Cache is automatically invalidated on save/delete operations.
 * Thread-safe for concurrent access.
 *
 * Use when: High read-to-write ratio (>10:1)
 * Don't use when: Data must always be fresh
 *
 * @example
 * const proxy = new CachingProxy(realService, 60000);
 * const user = await proxy.getUser(1); // Cached for 60s
 */
class CachingProxy implements IUserService { ... }
```

### 8. **Provide Cache Invalidation**

For caching proxies, always provide a way to clear the cache.

```java
// ✅ GOOD - Invalidation methods
public class CachingProxy implements UserService {
    public void clearCache() { ... }
    public void invalidateUser(Long userId) { ... }
    public void clearCacheIf(Predicate<CacheEntry> condition) { ... }
}

// ❌ BAD - No way to invalidate
public class CachingProxy implements UserService {
    private Map<Long, User> cache = new ConcurrentHashMap<>();
    // No way to clear stale data!
}
```

### 9. **Set Appropriate Timeouts**

For remote proxies or network calls, always set timeouts.

```go
// ✅ GOOD - With timeout
type RemoteServiceProxy struct {
    client  *http.Client
    baseURL string
}

func NewRemoteServiceProxy(baseURL string) *RemoteServiceProxy {
    return &RemoteServiceProxy{
        client: &http.Client{
            Timeout: 10 * time.Second, // Always set timeout
        },
        baseURL: baseURL,
    }
}

// ❌ BAD - No timeout
func NewRemoteServiceProxy(baseURL string) *RemoteServiceProxy {
    return &RemoteServiceProxy{
        client:  &http.Client{}, // Can hang forever!
        baseURL: baseURL,
    }
}
```

### 10. **Use Metrics and Monitoring**

Track proxy behavior for debugging and optimization.

```typescript
// ✅ GOOD - With metrics
class CachingProxy implements IService {
  private hits = 0;
  private misses = 0;

  async getData(id: number): Promise<Data> {
    if (this.cache.has(id)) {
      this.hits++;
      console.log(`Cache hit rate: ${this.getHitRate()}%`);
      return this.cache.get(id);
    }

    this.misses++;
    const data = await this.realService.getData(id);
    this.cache.set(id, data);
    return data;
  }

  getHitRate(): number {
    const total = this.hits + this.misses;
    return total === 0 ? 0 : (this.hits / total) * 100;
  }
}
```

---

## 🚫 Common Pitfalls

### 1. **Thread Safety Issues**

**Problem**: Multiple threads accessing proxy simultaneously causing race conditions.

```java
// ❌ PITFALL - Not thread-safe
public class LazyProxy implements Service {
    private RealService realService; // Shared state

    public void doWork() {
        if (realService == null) { // Race condition!
            realService = new RealService();
        }
        realService.doWork();
    }
}

// ✅ SOLUTION - Thread-safe
public class LazyProxy implements Service {
    private volatile RealService realService;
    private final Object lock = new Object();

    public void doWork() {
        if (realService == null) {
            synchronized (lock) {
                if (realService == null) { // Double-check
                    realService = new RealService();
                }
            }
        }
        realService.doWork();
    }
}
```

### 2. **Memory Leaks in Caching Proxies**

**Problem**: Cache grows unbounded, consuming all memory.

```typescript
// ❌ PITFALL - Unbounded cache
class CachingProxy {
  private cache = new Map<number, Data>(); // Grows forever!

  async getData(id: number): Promise<Data> {
    if (this.cache.has(id)) {
      return this.cache.get(id);
    }
    const data = await this.realService.getData(id);
    this.cache.set(id, data); // Never removed!
    return data;
  }
}

// ✅ SOLUTION - LRU cache with size limit
class CachingProxy {
  private cache = new LRUCache<number, Data>({ max: 1000 });
  private ttl = 60000; // 60 seconds

  async getData(id: number): Promise<Data> {
    const cached = this.cache.get(id);
    if (cached && Date.now() - cached.timestamp < this.ttl) {
      return cached.data;
    }

    const data = await this.realService.getData(id);
    this.cache.set(id, { data, timestamp: Date.now() });
    return data;
  }
}
```

### 3. **Proxy Chain Explosion**

**Problem**: Too many nested proxies hurt performance and debugging.

```java
// ❌ PITFALL - Too many layers
Service service = new RealService();
service = new LoggingProxy(service);
service = new CachingProxy(service);
service = new SecurityProxy(service);
service = new RateLimitingProxy(service);
service = new MetricsProxy(service);
service = new RetryProxy(service);
service = new CircuitBreakerProxy(service);
// 7+ layers deep - hard to debug!

// ✅ SOLUTION - Combine related concerns
Service service = new RealService();
service = new ObservabilityProxy(service); // Logging + Metrics
service = new ResilienceProxy(service);    // Retry + Circuit Breaker
service = new SecurityProxy(service);       // Auth + Rate Limiting + Caching
// 3 layers - much cleaner
```

### 4. **Ignoring Proxy Order**

**Problem**: Wrong order of proxies causes incorrect behavior.

```go
// ❌ PITFALL - Wrong order
var service Service = realService
service = NewCachingProxy(service)      // Caches everything
service = NewSecurityProxy(service)     // Security check after cache!
// Unauthorized users can access cached data!

// ✅ SOLUTION - Correct order
var service Service = realService
service = NewSecurityProxy(service)     // Security check first
service = NewCachingProxy(service)      // Then cache
// Only authorized data is cached
```

### 5. **Forgetting to Implement All Methods**

**Problem**: Proxy doesn't implement all interface methods.

```typescript
// ❌ PITFALL - Incomplete implementation
class CachingProxy implements IUserService {
  getUser(id: number): Promise<User> { ... }
  // Missing: updateUser, deleteUser, etc.
}

// ✅ SOLUTION - Implement all methods
class CachingProxy implements IUserService {
  async getUser(id: number): Promise<User> {
    // Caching logic
  }

  async updateUser(user: User): Promise<void> {
    await this.realService.updateUser(user);
    this.cache.delete(user.id); // Invalidate cache
  }

  async deleteUser(id: number): Promise<void> {
    await this.realService.deleteUser(id);
    this.cache.delete(id); // Invalidate cache
  }
}
```

### 6. **Not Handling Proxy Initialization Failures**

**Problem**: Lazy proxy fails to create real object but continues.

```java
// ❌ PITFALL - Silent failure
public class LazyProxy implements Service {
    private RealService realService;

    private RealService getRealService() {
        if (realService == null) {
            try {
                realService = new RealService();
            } catch (Exception e) {
                // Swallowed! Returns null
            }
        }
        return realService; // Might be null!
    }
}

// ✅ SOLUTION - Proper error handling
public class LazyProxy implements Service {
    private RealService realService;
    private boolean initialized = false;

    private RealService getRealService() {
        if (!initialized) {
            try {
                realService = new RealService();
                initialized = true;
            } catch (Exception e) {
                throw new ServiceInitializationException(
                    "Failed to initialize real service", e
                );
            }
        }
        return realService;
    }
}
```

### 7. **Cache Stampede (Thundering Herd)**

**Problem**: When cache expires, multiple requests hit the database simultaneously.

```typescript
// ❌ PITFALL - Cache stampede
class CachingProxy {
  async getData(id: number): Promise<Data> {
    if (!this.cache.has(id)) {
      // Multiple concurrent requests all miss cache
      const data = await this.realService.getData(id); // All hit DB!
      this.cache.set(id, data);
      return data;
    }
    return this.cache.get(id);
  }
}

// ✅ SOLUTION - Request coalescing
class CachingProxy {
  private pending = new Map<number, Promise<Data>>();

  async getData(id: number): Promise<Data> {
    if (this.cache.has(id)) {
      return this.cache.get(id);
    }

    // Check if request is already in flight
    if (this.pending.has(id)) {
      return this.pending.get(id); // Reuse existing request
    }

    // Create new request
    const promise = this.realService
      .getData(id)
      .then((data) => {
        this.cache.set(id, data);
        this.pending.delete(id);
        return data;
      })
      .catch((error) => {
        this.pending.delete(id);
        throw error;
      });

    this.pending.set(id, promise);
    return promise;
  }
}
```

### 8. **Not Considering Serialization**

**Problem**: Proxies don't serialize/deserialize properly.

```java
// ❌ PITFALL - Proxy doesn't serialize
public class LazyProxy implements Service, Serializable {
    private transient RealService realService; // Lost on serialization!
}

// ✅ SOLUTION - Proper serialization support
public class LazyProxy implements Service, Serializable {
    private transient RealService realService;
    private final String serviceIdentifier;

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        // Save identifier to recreate service
    }

    private void readObject(ObjectInputStream in)
            throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        // Recreate service using identifier
        this.realService = ServiceFactory.create(serviceIdentifier);
    }
}
```

### 9. **Incorrect Equals/HashCode Implementation**

**Problem**: Proxy breaks equality checks.

```go
// ❌ PITFALL - Proxy changes identity
type UserProxy struct {
    real *User
}

func (p *UserProxy) GetID() int {
    return p.real.GetID()
}

// user1 := &User{ID: 1}
// proxy := &UserProxy{real: user1}
// user1 == proxy // FALSE! Different types

// ✅ SOLUTION - Implement proper equality
type UserProxy struct {
    real *User
}

func (p *UserProxy) Equals(other interface{}) bool {
    if otherProxy, ok := other.(*UserProxy); ok {
        return p.real.GetID() == otherProxy.real.GetID()
    }
    if otherUser, ok := other.(*User); ok {
        return p.real.GetID() == otherUser.GetID()
    }
    return false
}
```

### 10. **Not Documenting Proxy Behavior**

**Problem**: Developers don't understand what the proxy does.

```typescript
// ❌ PITFALL - No documentation
class MyProxy implements IService {
  // What does this do? When should I use it?
  // How does it affect performance?
}

// ✅ SOLUTION - Clear documentation
/**
 * Caching proxy for expensive service calls.
 *
 * Behavior:
 * - Caches results for 5 minutes
 * - Max 1000 entries (LRU eviction)
 * - Thread-safe
 * - Invalidates on write operations
 *
 * Performance Impact:
 * - Read: ~1ms (cached) vs ~100ms (uncached)
 * - Write: No overhead
 * - Memory: ~50KB per 1000 entries
 *
 * When to use:
 * - High read-to-write ratio (>100:1)
 * - Data doesn't change frequently
 * - Can tolerate stale data up to 5 minutes
 *
 * When NOT to use:
 * - Real-time data requirements
 * - Write-heavy workloads
 * - Low memory environment
 */
class CachingServiceProxy implements IService { ... }
```

---

## 📋 Checklist for Implementing Proxies

### Before Implementation

- [ ] Is proxy the right pattern? (Not decorator?)
- [ ] What type of proxy do I need? (Virtual, Protection, Remote, etc.)
- [ ] Will multiple proxies be chained?
- [ ] What's the performance impact?

### During Implementation

- [ ] Implements same interface as real subject
- [ ] Thread-safe if used concurrently
- [ ] Proper error handling and propagation
- [ ] Timeout handling for remote proxies
- [ ] Cache invalidation for caching proxies
- [ ] Resource cleanup (connections, files, etc.)

### After Implementation

- [ ] Unit tests for proxy behavior
- [ ] Integration tests with real subject
- [ ] Performance benchmarks
- [ ] Documentation of behavior and use cases
- [ ] Metrics/logging for monitoring
- [ ] Security review for protection proxies

---

## 🎯 Decision Matrix: When to Use Which Proxy

| Use Case                  | Proxy Type            | Key Features               | Example              |
| ------------------------- | --------------------- | -------------------------- | -------------------- |
| Expensive object creation | Virtual Proxy         | Lazy initialization        | Loading large images |
| Access control            | Protection Proxy      | Permission checking        | User authorization   |
| Network communication     | Remote Proxy          | Handle network calls       | Microservice client  |
| Performance optimization  | Caching Proxy         | Store results              | Database query cache |
| Resource management       | Smart Reference       | Track usage, cleanup       | Connection pooling   |
| Resilience                | Circuit Breaker Proxy | Prevent cascading failures | External API calls   |

---

## 📝 Quick Reference

### Virtual Proxy Pattern

```
Intent: Delay expensive object creation
When: Heavy initialization, large memory footprint
Example: Lazy loading images, documents
```

### Protection Proxy Pattern

```
Intent: Control access to object
When: Need authorization, rate limiting
Example: Checking user permissions before access
```

### Remote Proxy Pattern

```
Intent: Represent remote object locally
When: Distributed systems, microservices
Example: API client, RPC stub
```

### Smart Reference Pattern

```
Intent: Add extra behavior on access
When: Need logging, counting, locking
Example: Reference counting, audit logging
```

---

## 🚀 Performance Tips

1. **Cache Size Limits**: Always set maximum cache size
2. **TTL Configuration**: Tune TTL based on data change frequency
3. **Monitoring**: Track hit rates, latency, memory usage
4. **Lazy vs Eager**: Profile to decide initialization strategy
5. **Proxy Depth**: Keep to 3-4 maximum layers
6. **Thread Safety**: Only add synchronization where needed
7. **Memory Management**: Implement proper cleanup for long-lived proxies

---
