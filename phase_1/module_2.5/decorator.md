🎯 Learning Objectives
By the end of this section, you will:

Understand what the Decorator pattern is and when to use it
Recognize scenarios where Decorator shines in SaaS applications
Implement Decorator pattern in Java, Node.js, and Go
Apply Decorator for middleware, feature additions, and cross-cutting concerns
Avoid common pitfalls when using Decorator

📖 Conceptual Overview
What is the Decorator Pattern?
The Decorator pattern allows you to add new functionality to an object dynamically without altering its structure. It provides a flexible alternative to subclassing for extending functionality.
Key Idea: Wrap an object with another object that adds behavior, while maintaining the same interface.
Real-World Analogy
Think of decorating a coffee:

Start with plain coffee (base object)
Add milk (decorator 1)
Add sugar (decorator 2)
Add whipped cream (decorator 3)

Each addition wraps the previous coffee, adding functionality without changing the core coffee object.

🤔 When to Use Decorator Pattern in SaaS
Perfect Use Cases:

Middleware Pipelines ⭐⭐⭐

Request logging
Authentication checks
Rate limiting
Request validation
Response transformation

Feature Toggles / Premium Features ⭐⭐⭐

Adding features based on subscription tier
A/B testing variations
Feature flags

Cross-Cutting Concerns ⭐⭐

Caching
Performance monitoring
Error handling
Audit logging

Multi-Tenant Customization ⭐⭐

Tenant-specific behavior
Custom branding
Feature customization per tenant

API Response Enhancement ⭐⭐

Adding metadata
Response formatting
Field filtering based on permissions

🚫 When NOT to Use Decorator

Simple scenarios - Don't over-engineer with decorators for trivial additions
When inheritance is clearer - If you have a clear "is-a" relationship
Performance-critical paths - Each decorator adds a layer of indirection
Too many layers - More than 3-4 decorators can become hard to debug

🏗️ Structure
Component (Interface)
↑
|
ConcreteComponent ← Decorator (wraps Component)
↑
|
ConcreteDecoratorA
ConcreteDecoratorB
Key Elements:

Component: Interface defining operations
ConcreteComponent: Original object with base behavior
Decorator: Abstract class implementing Component and holding a Component reference
ConcreteDecorator: Adds specific behavior

💻 Implementation in Java/Spring Boot
Scenario: API Response Enhancement with Caching
We'll build a service that fetches user data, and we'll add caching and logging decorators.
Step 1: Define the Component Interface
java// UserService.java - The component interface
package com.saas.service;

public interface UserService {
UserDTO getUserById(Long userId);
List<UserDTO> getAllUsers();
}
Step 2: Create the Concrete Component
java// UserServiceImpl.java - Base implementation
package com.saas.service.impl;

import com.saas.service.UserService;
import com.saas.dto.UserDTO;
import com.saas.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    @Override
    public UserDTO getUserById(Long userId) {
        // Base implementation - fetch from database
        return userRepository.findById(userId)
            .map(this::convertToDTO)
            .orElseThrow(() -> new UserNotFoundException(userId));
    }

    @Override
    public List<UserDTO> getAllUsers() {
        return userRepository.findAll()
            .stream()
            .map(this::convertToDTO)
            .collect(Collectors.toList());
    }

    private UserDTO convertToDTO(User user) {
        return UserDTO.builder()
            .id(user.getId())
            .email(user.getEmail())
            .name(user.getName())
            .build();
    }

}
Step 3: Create Abstract Decorator
java// UserServiceDecorator.java - Abstract decorator
package com.saas.service.decorator;

import com.saas.service.UserService;
import com.saas.dto.UserDTO;
import lombok.RequiredArgsConstructor;

@RequiredArgsConstructor
public abstract class UserServiceDecorator implements UserService {

    protected final UserService wrappedService;

    @Override
    public UserDTO getUserById(Long userId) {
        return wrappedService.getUserById(userId);
    }

    @Override
    public List<UserDTO> getAllUsers() {
        return wrappedService.getAllUsers();
    }

}
Step 4: Create Concrete Decorators
java// CachingUserServiceDecorator.java - Adds caching
package com.saas.service.decorator;

import com.saas.service.UserService;
import com.saas.dto.UserDTO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.Cacheable;

import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

@Slf4j
public class CachingUserServiceDecorator extends UserServiceDecorator {

    private final Map<Long, UserDTO> cache = new ConcurrentHashMap<>();

    public CachingUserServiceDecorator(UserService wrappedService) {
        super(wrappedService);
    }

    @Override
    public UserDTO getUserById(Long userId) {
        // Check cache first
        if (cache.containsKey(userId)) {
            log.info("Cache HIT for user {}", userId);
            return cache.get(userId);
        }

        log.info("Cache MISS for user {}", userId);
        UserDTO user = wrappedService.getUserById(userId);

        // Store in cache
        cache.put(userId, user);
        return user;
    }

    @Override
    public List<UserDTO> getAllUsers() {
        // For simplicity, not caching list operations
        return wrappedService.getAllUsers();
    }

    public void clearCache() {
        cache.clear();
        log.info("Cache cleared");
    }

}
java// LoggingUserServiceDecorator.java - Adds logging
package com.saas.service.decorator;

import com.saas.service.UserService;
import com.saas.dto.UserDTO;
import lombok.extern.slf4j.Slf4j;

import java.util.List;

@Slf4j
public class LoggingUserServiceDecorator extends UserServiceDecorator {

    public LoggingUserServiceDecorator(UserService wrappedService) {
        super(wrappedService);
    }

    @Override
    public UserDTO getUserById(Long userId) {
        log.info("Fetching user with ID: {}", userId);
        long startTime = System.currentTimeMillis();

        try {
            UserDTO user = wrappedService.getUserById(userId);
            long duration = System.currentTimeMillis() - startTime;

            log.info("Successfully fetched user {} in {}ms", userId, duration);
            return user;

        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            log.error("Failed to fetch user {} after {}ms: {}",
                userId, duration, e.getMessage());
            throw e;
        }
    }

    @Override
    public List<UserDTO> getAllUsers() {
        log.info("Fetching all users");
        long startTime = System.currentTimeMillis();

        try {
            List<UserDTO> users = wrappedService.getAllUsers();
            long duration = System.currentTimeMillis() - startTime;

            log.info("Successfully fetched {} users in {}ms",
                users.size(), duration);
            return users;

        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            log.error("Failed to fetch users after {}ms: {}",
                duration, e.getMessage());
            throw e;
        }
    }

}
java// PermissionUserServiceDecorator.java - Adds permission checking
package com.saas.service.decorator;

import com.saas.service.UserService;
import com.saas.dto.UserDTO;
import com.saas.security.SecurityContext;
import lombok.extern.slf4j.Slf4j;

import java.util.List;
import java.util.stream.Collectors;

@Slf4j
public class PermissionUserServiceDecorator extends UserServiceDecorator {

    private final SecurityContext securityContext;

    public PermissionUserServiceDecorator(UserService wrappedService,
                                         SecurityContext securityContext) {
        super(wrappedService);
        this.securityContext = securityContext;
    }

    @Override
    public UserDTO getUserById(Long userId) {
        // Check if current user has permission to view this user
        if (!securityContext.canViewUser(userId)) {
            log.warn("User {} attempted to access user {} without permission",
                securityContext.getCurrentUserId(), userId);
            throw new AccessDeniedException("No permission to view user");
        }

        UserDTO user = wrappedService.getUserById(userId);

        // Filter sensitive fields based on permissions
        return filterSensitiveData(user);
    }

    @Override
    public List<UserDTO> getAllUsers() {
        List<UserDTO> users = wrappedService.getAllUsers();

        // Filter users based on permissions
        return users.stream()
            .filter(user -> securityContext.canViewUser(user.getId()))
            .map(this::filterSensitiveData)
            .collect(Collectors.toList());
    }

    private UserDTO filterSensitiveData(UserDTO user) {
        if (!securityContext.hasRole("ADMIN")) {
            // Remove sensitive fields for non-admin users
            user.setEmail(null);
            user.setPhoneNumber(null);
        }
        return user;
    }

}
Step 5: Configuration - Wire Decorators
java// UserServiceConfiguration.java
package com.saas.config;

import com.saas.service.UserService;
import com.saas.service.impl.UserServiceImpl;
import com.saas.service.decorator.\*;
import com.saas.security.SecurityContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class UserServiceConfiguration {

    @Bean
    public UserService baseUserService(UserRepository userRepository) {
        return new UserServiceImpl(userRepository);
    }

    @Bean
    @Primary  // This is the one Spring will inject
    public UserService decoratedUserService(
            UserService baseUserService,
            SecurityContext securityContext) {

        // Layer decorators: Base → Logging → Caching → Permission
        UserService service = baseUserService;

        // Add logging
        service = new LoggingUserServiceDecorator(service);

        // Add caching
        service = new CachingUserServiceDecorator(service);

        // Add permission checking
        service = new PermissionUserServiceDecorator(service, securityContext);

        return service;
    }

}
Step 6: Usage in Controller
java// UserController.java
package com.saas.controller;

import com.saas.service.UserService;
import com.saas.dto.UserDTO;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.\*;

@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;  // Decorated version injected

    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        // All decorators applied automatically:
        // 1. Permission check
        // 2. Cache lookup
        // 3. Logging
        // 4. Base implementation
        return ResponseEntity.ok(userService.getUserById(id));
    }

    @GetMapping
    public ResponseEntity<List<UserDTO>> getAllUsers() {
        return ResponseEntity.ok(userService.getAllUsers());
    }

}

💻 Implementation in Node.js
Scenario: Express Middleware as Decorators
Node.js/Express middleware is essentially the Decorator pattern in action!
Step 1: Base Service
typescript// userService.ts - Base service
import { UserRepository } from './userRepository';
import { UserDTO } from './types';

export interface IUserService {
getUserById(userId: number): Promise<UserDTO>;
getAllUsers(): Promise<UserDTO[]>;
}

export class UserService implements IUserService {
constructor(private userRepository: UserRepository) {}

async getUserById(userId: number): Promise<UserDTO> {
const user = await this.userRepository.findById(userId);
if (!user) {
throw new Error(`User ${userId} not found`);
}
return this.toDTO(user);
}

async getAllUsers(): Promise<UserDTO[]> {
const users = await this.userRepository.findAll();
return users.map(this.toDTO);
}

private toDTO(user: any): UserDTO {
return {
id: user.id,
email: user.email,
name: user.name,
};
}
}
Step 2: Abstract Decorator
typescript// userServiceDecorator.ts - Abstract decorator
import { IUserService } from './userService';
import { UserDTO } from './types';

export abstract class UserServiceDecorator implements IUserService {
constructor(protected wrappedService: IUserService) {}

async getUserById(userId: number): Promise<UserDTO> {
return this.wrappedService.getUserById(userId);
}

async getAllUsers(): Promise<UserDTO[]> {
return this.wrappedService.getAllUsers();
}
}
Step 3: Concrete Decorators
typescript// cachingUserServiceDecorator.ts - Caching decorator
import { UserServiceDecorator } from './userServiceDecorator';
import { IUserService } from './userService';
import { UserDTO } from './types';

export class CachingUserServiceDecorator extends UserServiceDecorator {
private cache: Map<number, UserDTO> = new Map();
private ttl: number = 60000; // 60 seconds

constructor(wrappedService: IUserService) {
super(wrappedService);
}

async getUserById(userId: number): Promise<UserDTO> {
// Check cache
if (this.cache.has(userId)) {
console.log(`Cache HIT for user ${userId}`);
return this.cache.get(userId)!;
}

    console.log(`Cache MISS for user ${userId}`);
    const user = await this.wrappedService.getUserById(userId);

    // Store in cache with TTL
    this.cache.set(userId, user);
    setTimeout(() => {
      this.cache.delete(userId);
    }, this.ttl);

    return user;

}

async getAllUsers(): Promise<UserDTO[]> {
// Not caching list operations for simplicity
return this.wrappedService.getAllUsers();
}

clearCache(): void {
this.cache.clear();
console.log('Cache cleared');
}
}
typescript// loggingUserServiceDecorator.ts - Logging decorator
import { UserServiceDecorator } from './userServiceDecorator';
import { IUserService } from './userService';
import { UserDTO } from './types';

export class LoggingUserServiceDecorator extends UserServiceDecorator {
constructor(wrappedService: IUserService) {
super(wrappedService);
}

async getUserById(userId: number): Promise<UserDTO> {
console.log(`Fetching user with ID: ${userId}`);
const startTime = Date.now();

    try {
      const user = await this.wrappedService.getUserById(userId);
      const duration = Date.now() - startTime;

      console.log(`Successfully fetched user ${userId} in ${duration}ms`);
      return user;
    } catch (error) {
      const duration = Date.now() - startTime;
      console.error(
        `Failed to fetch user ${userId} after ${duration}ms:`,
        error.message
      );
      throw error;
    }

}

async getAllUsers(): Promise<UserDTO[]> {
console.log('Fetching all users');
const startTime = Date.now();

    try {
      const users = await this.wrappedService.getAllUsers();
      const duration = Date.now() - startTime;

      console.log(`Successfully fetched ${users.length} users in ${duration}ms`);
      return users;
    } catch (error) {
      const duration = Date.now() - startTime;
      console.error(`Failed to fetch users after ${duration}ms:`, error.message);
      throw error;
    }

}
}
typescript// permissionUserServiceDecorator.ts - Permission checking decorator
import { UserServiceDecorator } from './userServiceDecorator';
import { IUserService } from './userService';
import { UserDTO } from './types';
import { SecurityContext } from './securityContext';

export class PermissionUserServiceDecorator extends UserServiceDecorator {
constructor(
wrappedService: IUserService,
private securityContext: SecurityContext
) {
super(wrappedService);
}

async getUserById(userId: number): Promise<UserDTO> {
// Check permissions
if (!this.securityContext.canViewUser(userId)) {
console.warn(
`User ${this.securityContext.getCurrentUserId()} attempted to access ` +
`user ${userId} without permission`
);
throw new Error('No permission to view user');
}

    const user = await this.wrappedService.getUserById(userId);
    return this.filterSensitiveData(user);

}

async getAllUsers(): Promise<UserDTO[]> {
const users = await this.wrappedService.getAllUsers();

    // Filter based on permissions
    return users
      .filter(user => this.securityContext.canViewUser(user.id))
      .map(user => this.filterSensitiveData(user));

}

private filterSensitiveData(user: UserDTO): UserDTO {
if (!this.securityContext.hasRole('ADMIN')) {
// Remove sensitive fields for non-admin users
return {
...user,
email: undefined,
phoneNumber: undefined,
};
}
return user;
}
}
Step 4: Service Factory/Composition
typescript// userServiceFactory.ts - Compose decorators
import { UserService, IUserService } from './userService';
import { LoggingUserServiceDecorator } from './loggingUserServiceDecorator';
import { CachingUserServiceDecorator } from './cachingUserServiceDecorator';
import { PermissionUserServiceDecorator } from './permissionUserServiceDecorator';
import { UserRepository } from './userRepository';
import { SecurityContext } from './securityContext';

export class UserServiceFactory {
static create(
userRepository: UserRepository,
securityContext: SecurityContext
): IUserService {
// Start with base service
let service: IUserService = new UserService(userRepository);

    // Layer decorators: Base → Logging → Caching → Permission
    service = new LoggingUserServiceDecorator(service);
    service = new CachingUserServiceDecorator(service);
    service = new PermissionUserServiceDecorator(service, securityContext);

    return service;

}
}
Step 5: Usage in Express Controller
typescript// userController.ts
import { Request, Response } from 'express';
import { UserServiceFactory } from './userServiceFactory';
import { UserRepository } from './userRepository';
import { SecurityContext } from './securityContext';

export class UserController {
private userService;

constructor() {
const userRepository = new UserRepository();
const securityContext = new SecurityContext();

    // Get fully decorated service
    this.userService = UserServiceFactory.create(
      userRepository,
      securityContext
    );

}

async getUser(req: Request, res: Response): Promise<void> {
try {
const userId = parseInt(req.params.id);

      // All decorators applied automatically
      const user = await this.userService.getUserById(userId);

      res.json(user);
    } catch (error) {
      res.status(error.message.includes('permission') ? 403 : 404)
        .json({ error: error.message });
    }

}

async getAllUsers(req: Request, res: Response): Promise<void> {
try {
const users = await this.userService.getAllUsers();
res.json(users);
} catch (error) {
res.status(500).json({ error: error.message });
}
}
}
Bonus: Express Middleware as Decorators
typescript// middleware/decorators.ts - Middleware as decorators
import { Request, Response, NextFunction } from 'express';

// Logging decorator (middleware)
export function loggingMiddleware(
req: Request,
res: Response,
next: NextFunction
): void {
const startTime = Date.now();

console.log(`${req.method} ${req.path}`);

// Capture response
res.on('finish', () => {
const duration = Date.now() - startTime;
console.log(
`${req.method} ${req.path} - ${res.statusCode} - ${duration}ms`
);
});

next();
}

// Caching decorator (middleware)
export function cachingMiddleware(ttl: number = 60000) {
const cache = new Map<string, { data: any; expiry: number }>();

return (req: Request, res: Response, next: NextFunction): void => {
const key = req.originalUrl;

    // Check cache
    const cached = cache.get(key);
    if (cached && cached.expiry > Date.now()) {
      console.log(`Cache HIT for ${key}`);
      return res.json(cached.data);
    }

    // Override res.json to cache response
    const originalJson = res.json.bind(res);
    res.json = function(data: any) {
      cache.set(key, {
        data,
        expiry: Date.now() + ttl,
      });
      return originalJson(data);
    };

    next();

};
}

// Rate limiting decorator (middleware)
export function rateLimitMiddleware(maxRequests: number, windowMs: number) {
const requests = new Map<string, number[]>();

return (req: Request, res: Response, next: NextFunction): void => {
const key = req.ip || 'unknown';
const now = Date.now();

    // Get request timestamps for this IP
    const timestamps = requests.get(key) || [];

    // Filter out old timestamps
    const recentTimestamps = timestamps.filter(
      time => now - time < windowMs
    );

    if (recentTimestamps.length >= maxRequests) {
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter: Math.ceil((recentTimestamps[0] + windowMs - now) / 1000),
      });
    }

    // Add current request
    recentTimestamps.push(now);
    requests.set(key, recentTimestamps);

    next();

};
}
typescript// app.ts - Using middleware decorators
import express from 'express';
import {
loggingMiddleware,
cachingMiddleware,
rateLimitMiddleware,
} from './middleware/decorators';
import { UserController } from './userController';

const app = express();
const userController = new UserController();

// Apply decorators (middleware) globally
app.use(loggingMiddleware);

// Apply decorators to specific routes
app.get(
'/api/users/:id',
rateLimitMiddleware(100, 60000), // 100 requests per minute
cachingMiddleware(30000), // 30 second cache
(req, res) => userController.getUser(req, res)
);

app.get(
'/api/users',
rateLimitMiddleware(50, 60000),
cachingMiddleware(60000),
(req, res) => userController.getAllUsers(req, res)
);

app.listen(3000, () => {
console.log('Server running on port 3000');
});

💻 Implementation in Go
Scenario: HTTP Handler Decorators
Go's http.Handler interface is perfect for the Decorator pattern!
Step 1: Base Service
go// user_service.go - Base service
package service

import (
"errors"
"myapp/model"
)

// UserService interface
type UserService interface {
GetUserByID(userID int64) (*model.UserDTO, error)
GetAllUsers() ([]*model.UserDTO, error)
}

// userServiceImpl - Concrete implementation
type userServiceImpl struct {
repo UserRepository
}

func NewUserService(repo UserRepository) UserService {
return &userServiceImpl{
repo: repo,
}
}

func (s *userServiceImpl) GetUserByID(userID int64) (*model.UserDTO, error) {
user, err := s.repo.FindByID(userID)
if err != nil {
return nil, err
}
if user == nil {
return nil, errors.New("user not found")
}
return s.toDTO(user), nil
}

func (s *userServiceImpl) GetAllUsers() ([]*model.UserDTO, error) {
users, err := s.repo.FindAll()
if err != nil {
return nil, err
}

    dtos := make([]*model.UserDTO, len(users))
    for i, user := range users {
        dtos[i] = s.toDTO(user)
    }
    return dtos, nil

}

func (s *userServiceImpl) toDTO(user *model.User) \*model.UserDTO {
return &model.UserDTO{
ID: user.ID,
Email: user.Email,
Name: user.Name,
}
}
Step 2: Base Decorator
go// user_service_decorator.go - Base decorator
package service

import "myapp/model"

// UserServiceDecorator - Abstract decorator
type UserServiceDecorator struct {
wrapped UserService
}

func (d *UserServiceDecorator) GetUserByID(userID int64) (*model.UserDTO, error) {
return d.wrapped.GetUserByID(userID)
}

func (d *UserServiceDecorator) GetAllUsers() ([]*model.UserDTO, error) {
return d.wrapped.GetAllUsers()
}
Step 3: Concrete Decorators
go// caching_decorator.go - Caching decorator
package service

import (
"fmt"
"myapp/model"
"sync"
"time"
)

type CachingUserServiceDecorator struct {
wrapped UserService
cache map[int64]\*cacheEntry
mu sync.RWMutex
ttl time.Duration
}

type cacheEntry struct {
user \*model.UserDTO
expiry time.Time
}

func NewCachingUserServiceDecorator(wrapped UserService, ttl time.Duration) UserService {
decorator := &CachingUserServiceDecorator{
wrapped: wrapped,
cache: make(map[int64]\*cacheEntry),
ttl: ttl,
}

    // Start cache cleanup goroutine
    go decorator.cleanupExpired()

    return decorator

}

func (d *CachingUserServiceDecorator) GetUserByID(userID int64) (*model.UserDTO, error) {
// Check cache
d.mu.RLock()
if entry, ok := d.cache[userID]; ok {
if time.Now().Before(entry.expiry) {
d.mu.RUnlock()
fmt.Printf("Cache HIT for user %d\n", userID)
return entry.user, nil
}
}
d.mu.RUnlock()

    fmt.Printf("Cache MISS for user %d\n", userID)

    // Fetch from wrapped service
    user, err := d.wrapped.GetUserByID(userID)
    if err != nil {
        return nil, err
    }

    // Store in cache
    d.mu.Lock()
    d.cache[userID] = &cacheEntry{
        user:   user,
        expiry: time.Now().Add(d.ttl),
    }
    d.mu.Unlock()

    return user, nil

}

func (d *CachingUserServiceDecorator) GetAllUsers() ([]*model.UserDTO, error) {
// Not caching list operations for simplicity
return d.wrapped.GetAllUsers()
}

func (d *CachingUserServiceDecorator) ClearCache() {
d.mu.Lock()
defer d.mu.Unlock()
d.cache = make(map[int64]*cacheEntry)
fmt.Println("Cache cleared")
}

func (d _CachingUserServiceDecorator) cleanupExpired() {
ticker := time.NewTicker(1 _ time.Minute)
defer ticker.Stop()

    for range ticker.C {
        d.mu.Lock()
        now := time.Now()
        for id, entry := range d.cache {
            if now.After(entry.expiry) {
                delete(d.cache, id)
            }
        }
        d.mu.Unlock()
    }

}
go// logging_decorator.go - Logging decorator
package service

import (
"fmt"
"myapp/model"
"time"
)

type LoggingUserServiceDecorator struct {
wrapped UserService
}

func NewLoggingUserServiceDecorator(wrapped UserService) UserService {
return &LoggingUserServiceDecorator{
wrapped: wrapped,
}
}

func (d *LoggingUserServiceDecorator) GetUserByID(userID int64) (*model.UserDTO, error) {
fmt.Printf("Fetching user with ID: %d\n", userID)
startTime := time.Now()

    user, err := d.wrapped.GetUserByID(userID)
    duration := time.Since(startTime)

    if err != nil {
        fmt.Printf("Failed to fetch user %d after %v: %v\n", userID, duration, err)
        return nil, err
    }

    fmt.Printf("Successfully fetched user %d in %v\n", userID, duration)
    return user, nil

}

func (d *LoggingUserServiceDecorator) GetAllUsers() ([]*model.UserDTO, error) {
fmt.Println("YDContinueFetching all users")
startTime := time.Now()
users, err := d.wrapped.GetAllUsers()
duration := time.Since(startTime)

if err != nil {
fmt.Printf("Failed to fetch users after %v: %v\n", duration, err)
return nil, err
}

fmt.Printf("Successfully fetched %d users in %v\n", len(users), duration)
return users, nil
}

```go
// permission_decorator.go - Permission checking decorator
package service

import (
    "errors"
    "fmt"
    "myapp/model"
    "myapp/security"
)

type PermissionUserServiceDecorator struct {
    wrapped         UserService
    securityContext *security.Context
}

func NewPermissionUserServiceDecorator(
    wrapped UserService,
    securityContext *security.Context,
) UserService {
    return &PermissionUserServiceDecorator{
        wrapped:         wrapped,
        securityContext: securityContext,
    }
}

func (d *PermissionUserServiceDecorator) GetUserByID(userID int64) (*model.UserDTO, error) {
    // Check permissions
    if !d.securityContext.CanViewUser(userID) {
        fmt.Printf(
            "User %d attempted to access user %d without permission\n",
            d.securityContext.GetCurrentUserID(),
            userID,
        )
        return nil, errors.New("no permission to view user")
    }

    user, err := d.wrapped.GetUserByID(userID)
    if err != nil {
        return nil, err
    }

    return d.filterSensitiveData(user), nil
}

func (d *PermissionUserServiceDecorator) GetAllUsers() ([]*model.UserDTO, error) {
    users, err := d.wrapped.GetAllUsers()
    if err != nil {
        return nil, err
    }

    // Filter based on permissions
    filtered := make([]*model.UserDTO, 0)
    for _, user := range users {
        if d.securityContext.CanViewUser(user.ID) {
            filtered = append(filtered, d.filterSensitiveData(user))
        }
    }

    return filtered, nil
}

func (d *PermissionUserServiceDecorator) filterSensitiveData(user *model.UserDTO) *model.UserDTO {
    if !d.securityContext.HasRole("ADMIN") {
        // Remove sensitive fields for non-admin users
        user.Email = ""
        user.PhoneNumber = ""
    }
    return user
}
```

#### Step 4: Service Composition

```go
// service_factory.go - Compose decorators
package service

import (
    "myapp/security"
    "time"
)

func NewDecoratedUserService(
    repo UserRepository,
    securityContext *security.Context,
) UserService {
    // Start with base service
    var service UserService = NewUserService(repo)

    // Layer decorators: Base → Logging → Caching → Permission
    service = NewLoggingUserServiceDecorator(service)
    service = NewCachingUserServiceDecorator(service, 60*time.Second)
    service = NewPermissionUserServiceDecorator(service, securityContext)

    return service
}
```

#### Step 5: HTTP Handler

```go
// user_handler.go
package handler

import (
    "encoding/json"
    "myapp/service"
    "net/http"
    "strconv"
)

type UserHandler struct {
    userService service.UserService
}

func NewUserHandler(userService service.UserService) *UserHandler {
    return &UserHandler{
        userService: userService,
    }
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    // Extract user ID from URL
    userIDStr := r.URL.Query().Get("id")
    userID, err := strconv.ParseInt(userIDStr, 10, 64)
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }

    // All decorators applied automatically
    user, err := h.userService.GetUserByID(userID)
    if err != nil {
        if err.Error() == "no permission to view user" {
            http.Error(w, err.Error(), http.StatusForbidden)
            return
        }
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func (h *UserHandler) GetAllUsers(w http.ResponseWriter, r *http.Request) {
    users, err := h.userService.GetAllUsers()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}
```

### Bonus: HTTP Middleware Decorators

```go
// middleware/decorators.go - HTTP middleware as decorators
package middleware

import (
    "fmt"
    "net/http"
    "sync"
    "time"
)

// Logging middleware decorator
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        startTime := time.Now()

        fmt.Printf("%s %s\n", r.Method, r.URL.Path)

        // Wrap ResponseWriter to capture status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        next.ServeHTTP(wrapped, r)

        duration := time.Since(startTime)
        fmt.Printf(
            "%s %s - %d - %v\n",
            r.Method,
            r.URL.Path,
            wrapped.statusCode,
            duration,
        )
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

// Caching middleware decorator
func CachingMiddleware(ttl time.Duration) func(http.Handler) http.Handler {
    cache := &responseCache{
        entries: make(map[string]*cacheEntry),
    }

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            key := r.URL.Path

            // Check cache
            if entry := cache.get(key); entry != nil {
                fmt.Printf("Cache HIT for %s\n", key)
                w.Header().Set("Content-Type", "application/json")
                w.Write(entry.body)
                return
            }

            // Capture response
            recorder := &responseRecorder{ResponseWriter: w}
            next.ServeHTTP(recorder, r)

            // Store in cache
            cache.set(key, recorder.body, ttl)
        })
    }
}

type responseCache struct {
    mu      sync.RWMutex
    entries map[string]*cacheEntry
}

type cacheEntry struct {
    body   []byte
    expiry time.Time
}

func (c *responseCache) get(key string) *cacheEntry {
    c.mu.RLock()
    defer c.mu.RUnlock()

    if entry, ok := c.entries[key]; ok {
        if time.Now().Before(entry.expiry) {
            return entry
        }
    }
    return nil
}

func (c *responseCache) set(key string, body []byte, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.entries[key] = &cacheEntry{
        body:   body,
        expiry: time.Now().Add(ttl),
    }
}

type responseRecorder struct {
    http.ResponseWriter
    body []byte
}

func (r *responseRecorder) Write(b []byte) (int, error) {
    r.body = append(r.body, b...)
    return r.ResponseWriter.Write(b)
}

// Rate limiting middleware decorator
func RateLimitMiddleware(maxRequests int, window time.Duration) func(http.Handler) http.Handler {
    limiter := &rateLimiter{
        requests: make(map[string][]time.Time),
    }

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            key := r.RemoteAddr

            if !limiter.allow(key, maxRequests, window) {
                http.Error(w, "Too many requests", http.StatusTooManyRequests)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

type rateLimiter struct {
    mu       sync.Mutex
    requests map[string][]time.Time
}

func (rl *rateLimiter) allow(key string, max int, window time.Duration) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    now := time.Now()

    // Get timestamps for this key
    timestamps := rl.requests[key]

    // Filter out old timestamps
    recent := make([]time.Time, 0)
    for _, ts := range timestamps {
        if now.Sub(ts) < window {
            recent = append(recent, ts)
        }
    }

    if len(recent) >= max {
        return false
    }

    // Add current request
    recent = append(recent, now)
    rl.requests[key] = recent

    return true
}
```

```go
// main.go - Using middleware decorators
package main

import (
    "myapp/handler"
    "myapp/middleware"
    "myapp/repository"
    "myapp/security"
    "myapp/service"
    "net/http"
    "time"
)

func main() {
    // Setup dependencies
    repo := repository.NewUserRepository()
    securityContext := security.NewContext()

    // Create decorated service
    userService := service.NewDecoratedUserService(repo, securityContext)
    userHandler := handler.NewUserHandler(userService)

    // Setup routes with middleware decorators
    mux := http.NewServeMux()

    // Chain decorators: Logging → RateLimit → Caching → Handler
    getUserHandler := middleware.LoggingMiddleware(
        middleware.RateLimitMiddleware(100, time.Minute)(
            middleware.CachingMiddleware(30 * time.Second)(
                http.HandlerFunc(userHandler.GetUser),
            ),
        ),
    )

    getAllUsersHandler := middleware.LoggingMiddleware(
        middleware.RateLimitMiddleware(50, time.Minute)(
            middleware.CachingMiddleware(60 * time.Second)(
                http.HandlerFunc(userHandler.GetAllUsers),
            ),
        ),
    )

    mux.Handle("/api/users", getAllUsersHandler)
    mux.Handle("/api/users/", getUserHandler)

    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", mux)
}
```

---

## ✅ Best Practices

### 1. **Keep Decorators Single-Purpose**

Each decorator should add ONE specific behavior:

- ✅ CachingDecorator
- ✅ LoggingDecorator
- ❌ CachingAndLoggingDecorator

### 2. **Order Matters**

The order of decorator wrapping affects behavior:
Base → Logging → Caching → Permission
vs
Base → Permission → Caching → Logging
Plan your decorator chain carefully!

### 3. **Interface Consistency**

All decorators must implement the same interface as the base component.

### 4. **Avoid Over-Decoration**

Don't stack too many decorators (3-5 max). Beyond that, consider alternative patterns.

### 5. **Thread Safety (Java/Go)**

Ensure decorators with state (like caching) are thread-safe:

- Java: Use `ConcurrentHashMap`, `synchronized`
- Go: Use `sync.RWMutex`, channels

### 6. **Transparent Decoration**

Decorators should be invisible to clients. The client shouldn't know or care if it's using a decorated or base object.

### 7. **Configuration Over Code**

Make decorator composition configurable:

```java
@Value("${feature.caching.enabled}")
private boolean cachingEnabled;

if (cachingEnabled) {
    service = new CachingDecorator(service);
}
```

---

## 🚫 Common Pitfalls

### 1. **Decorator Explosion**

**Problem**: Too many decorator classes
**Solution**: Use composition or higher-order functions

### 2. **Tight Coupling**

**Problem**: Decorators depend on concrete implementations
**Solution**: Always depend on interfaces

### 3. **State Leakage**

**Problem**: Decorators share mutable state
**Solution**: Each decorator should have isolated state

### 4. **Order Dependency Bugs**

**Problem**: Wrong decorator order breaks functionality
**Solution**: Document required order, write tests

### 5. **Performance Overhead**

**Problem**: Every decorator adds method call overhead
**Solution**: Profile and optimize critical paths

### 6. **Debugging Difficulty**

**Problem**: Stack traces become deeply nested
**Solution**: Add clear logging, use meaningful class names

---

## 🎯 Real-World SaaS Use Cases

### 1. **Subscription Tier Features** ⭐⭐⭐

```java
// Base service
UserService baseService = new UserServiceImpl();

// Add features based on subscription
if (tenant.getPlan() == Plan.PREMIUM) {
    service = new AdvancedAnalyticsDecorator(service);
    service = new ExportToPDFDecorator(service);
}

if (tenant.getPlan() == Plan.ENTERPRISE) {
    service = new SSODecorator(service);
    service = new CustomBrandingDecorator(service);
}
```

### 2. **API Throttling by Plan**

```typescript
// Free tier: 10 requests/minute
let handler = rateLimitMiddleware(10, 60000)(baseHandler);

// Pro tier: 100 requests/minute
if (user.plan === "pro") {
  handler = rateLimitMiddleware(100, 60000)(baseHandler);
}

// Enterprise: Unlimited
if (user.plan === "enterprise") {
  handler = baseHandler;
}
```

### 3. **Multi-Tenant Data Isolation**

```go
// Base service
var service UserService = NewUserService(repo)

// Add tenant isolation
service = NewTenantIsolationDecorator(service, tenantID)

// Add tenant-specific caching
service = NewTenantCachingDecorator(service, tenantID)
```

### 4. **A/B Testing**

```java
UserService service = baseService;

if (featureFlags.isEnabled("new-algorithm", userId)) {
    service = new NewAlgorithmDecorator(service);
} else {
    service = new OldAlgorithmDecorator(service);
}
```

### 5. **Audit Logging for Compliance**

```typescript
let service: IUserService = new UserService(repo);

// Add audit logging for GDPR/HIPAA compliance
if (tenant.requiresAuditLog) {
  service = new AuditLoggingDecorator(service, auditLogger);
}

// Add data encryption for sensitive data
if (tenant.encryptionRequired) {
  service = new EncryptionDecorator(service, encryptionKey);
}
```

---

## 📚 Resources for Deeper Understanding

### Books

1. **"Design Patterns: Elements of Reusable Object-Oriented Software"** - Gang of Four

   - Chapter: Structural Patterns → Decorator
   - The original and most authoritative source

2. **"Head First Design Patterns"** - Freeman & Freeman

   - Chapter 3: The Decorator Pattern
   - More accessible, with great examples (coffee shop analogy)

3. **"Clean Architecture"** - Robert C. Martin
   - Discusses decorators in context of cross-cutting concerns

### Online Resources

1. **Refactoring Guru - Decorator Pattern**

   - URL: https://refactoring.guru/design-patterns/decorator
   - Excellent visual explanations and code examples

2. **Spring Framework Documentation - AOP**

   - URL: https://docs.spring.io/spring-framework/reference/core/aop.html
   - Shows how Spring uses decorators (proxies) for AOP

3. **Express.js Middleware Guide**

   - URL: https://expressjs.com/en/guide/using-middleware.html
   - Node.js implementation of decorator pattern

4. **Go Patterns - Decorator**
   - URL: https://github.com/tmrts/go-patterns#decorator
   - Idiomatic Go implementations

### Videos

1. **"Design Patterns: Decorator Pattern"** - Christopher Okhravi (YouTube)

   - Clear explanations with whiteboard diagrams

2. **"Middleware in Express.js"** - Traversy Media
   - Practical Node.js decorator examples

### Articles

1. **"Decorators in Java"** - Baeldung
   - URL: https://www.baeldung.com/java-decorator-pattern
2. **"TypeScript Decorators"** - TypeScript Handbook
   - URL: https://www.typescriptlang.org/docs/handbook/decorators.html

---

## 🏋️ Practical Exercise

### Exercise: Build a Notification Service with Decorators

**Scenario**: You're building a notification system for a SaaS app. Different subscription tiers get different features.

**Requirements**:

1. **Base functionality**: Send plain text notifications
2. **Decorator 1**: Add rate limiting (max N notifications per hour)
3. **Decorator 2**: Add retry logic for failed deliveries
4. **Decorator 3**: Add rich formatting (HTML, Markdown)
5. **Decorator 4**: Add delivery tracking and read receipts

**Your Task**:
Implement this in your preferred language (Java/Node.js/Go) with:

- Base notification service
- At least 3 decorators
- Configuration to enable/disable decorators
- Unit tests for each decorator

**Bonus**:

- Add a decorator for multi-channel delivery (email + SMS + push)
- Add a decorator for notification scheduling
- Add a decorator for template rendering
