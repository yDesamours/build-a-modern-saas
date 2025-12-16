# Module 2.5: Core Design Patterns for SaaS Applications

---

## Part 4: Behavioral Patterns - Chain of Responsibility Pattern

---

## 🎯 Learning Objectives

By the end of this section, you will:

- Understand what the Chain of Responsibility pattern is and when to use it
- Recognize scenarios where Chain of Responsibility shines in SaaS applications
- Implement Chain of Responsibility in Java, Node.js, and Go
- Build middleware pipelines and request processing chains
- Create flexible validation and approval workflows
- Avoid common pitfalls like infinite loops and tight coupling

---

## 📖 Conceptual Overview

### What is the Chain of Responsibility Pattern?

The **Chain of Responsibility pattern** passes a request along a chain of handlers. Each handler decides either to process the request or pass it to the next handler in the chain.

**Key Idea**: Decouple senders and receivers by giving multiple objects a chance to handle the request. Chain handlers together and pass the request along until one handles it.

### Real-World Analogy

Think of a customer support escalation system:

- **Level 1**: Support agent tries to resolve
- **Level 2**: If can't resolve → escalate to senior agent
- **Level 3**: If still can't resolve → escalate to manager
- **Level 4**: If critical → escalate to director

Each level either handles the issue or passes it up the chain.

Another analogy is airport security:

- **Check-in counter**: Verify ticket
- **Security checkpoint**: Scan bags
- **Passport control**: Verify identity
- **Gate**: Final boarding pass check

Request (passenger) passes through each checkpoint.

---

## 🤔 When to Use Chain of Responsibility in SaaS

### Perfect Use Cases:

1. **Request Processing Pipelines** ⭐⭐⭐

   - HTTP middleware (authentication, logging, CORS)
   - API request validation
   - Request transformation
   - Response filtering

2. **Validation Chains** ⭐⭐⭐

   - Multi-level validation (syntax → business → security)
   - Form validation with multiple rules
   - Data quality checks
   - Compliance validation

3. **Approval Workflows** ⭐⭐⭐

   - Purchase approvals (manager → director → CFO)
   - Document review (editor → reviewer → publisher)
   - Access requests (team lead → admin → security)
   - Budget approvals

4. **Event Processing** ⭐⭐

   - Event filtering chains
   - Event transformation pipelines
   - Event routing based on conditions

5. **Logging Systems** ⭐⭐

   - Multi-level logging (console → file → remote)
   - Log filtering by severity
   - Log enrichment pipeline

6. **Error Handling** ⭐⭐

   - Exception handling hierarchy
   - Fallback strategies
   - Retry mechanisms with escalation

7. **Authentication/Authorization** ⭐⭐⭐

   - Multiple auth strategies (API key → JWT → OAuth)
   - Permission checking chain
   - Rate limiting → authentication → authorization

8. **Content Filtering** ⭐⭐
   - Spam detection
   - Content moderation
   - Profanity filtering
   - Security scanning

---

## 🚫 When NOT to Use Chain of Responsibility

- **Simple linear logic** - Direct if/else is clearer
- **Single handler** - No need for chain if only one handler
- **Order doesn't matter** - Use Observer pattern instead
- **All handlers must process** - Chain stops after first handler

---

## 🏗️ Structure

```
Client ──> Handler (Interface)
               ↑
               |
          AbstractHandler ──next──> AbstractHandler
               ↑                         ↑
               |                         |
        ConcreteHandlerA          ConcreteHandlerB
```

**Key Elements:**

1. **Handler Interface**: Defines handle method
2. **AbstractHandler**: Implements chain logic (optional)
3. **ConcreteHandler**: Implements specific handling logic
4. **Client**: Initiates request to chain
5. **Next Reference**: Link to next handler in chain

---

## 💻 Implementation in Java/Spring Boot

### Scenario 1: HTTP Request Processing Pipeline

Build a middleware chain for API requests with authentication, rate limiting, logging, and validation.

#### Step 1: Define Handler Interface

```java
// RequestHandler.java - Handler interface
package com.saas.handler;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public interface RequestHandler {
    /**
     * Handle the request
     * @return true if request should continue, false to stop chain
     */
    boolean handle(HttpServletRequest request, HttpServletResponse response);

    /**
     * Set next handler in chain
     */
    void setNext(RequestHandler next);
}
```

#### Step 2: Create Abstract Base Handler

```java
// AbstractRequestHandler.java - Base handler with chain logic
package com.saas.handler;

import lombok.extern.slf4j.Slf4j;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Slf4j
public abstract class AbstractRequestHandler implements RequestHandler {

    protected RequestHandler next;

    @Override
    public void setNext(RequestHandler next) {
        this.next = next;
    }

    @Override
    public boolean handle(HttpServletRequest request, HttpServletResponse response) {
        // Execute this handler's logic
        boolean shouldContinue = doHandle(request, response);

        // If should continue and there's a next handler, pass to next
        if (shouldContinue && next != null) {
            return next.handle(request, response);
        }

        return shouldContinue;
    }

    /**
     * Subclasses implement specific handling logic
     * @return true to continue chain, false to stop
     */
    protected abstract boolean doHandle(HttpServletRequest request,
                                        HttpServletResponse response);
}
```

#### Step 3: Implement Concrete Handlers

```java
// LoggingHandler.java - Logs incoming requests
package com.saas.handler.impl;

import com.saas.handler.AbstractRequestHandler;
import lombok.extern.slf4j.Slf4j;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Slf4j
public class LoggingHandler extends AbstractRequestHandler {

    @Override
    protected boolean doHandle(HttpServletRequest request,
                               HttpServletResponse response) {
        long startTime = System.currentTimeMillis();

        log.info("📝 Request: {} {} from {}",
            request.getMethod(),
            request.getRequestURI(),
            request.getRemoteAddr());

        // Store start time for response logging
        request.setAttribute("startTime", startTime);

        // Always continue to next handler
        return true;
    }
}
```

```java
// CorsHandler.java - Handles CORS headers
package com.saas.handler.impl;

import com.saas.handler.AbstractRequestHandler;
import lombok.extern.slf4j.Slf4j;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Slf4j
public class CorsHandler extends AbstractRequestHandler {

    private final String allowedOrigins;

    public CorsHandler(String allowedOrigins) {
        this.allowedOrigins = allowedOrigins;
    }

    @Override
    protected boolean doHandle(HttpServletRequest request,
                               HttpServletResponse response) {
        log.debug("🌐 Setting CORS headers");

        response.setHeader("Access-Control-Allow-Origin", allowedOrigins);
        response.setHeader("Access-Control-Allow-Methods",
            "GET, POST, PUT, DELETE, OPTIONS");
        response.setHeader("Access-Control-Allow-Headers",
            "Content-Type, Authorization");

        // Handle preflight requests
        if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
            response.setStatus(HttpServletResponse.SC_OK);
            return false; // Stop chain for OPTIONS
        }

        return true; // Continue for other methods
    }
}
```

```java
// RateLimitHandler.java - Rate limiting
package com.saas.handler.impl;

import com.saas.handler.AbstractRequestHandler;
import com.saas.service.RateLimiterService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Slf4j
@RequiredArgsConstructor
public class RateLimitHandler extends AbstractRequestHandler {

    private final RateLimiterService rateLimiter;

    @Override
    protected boolean doHandle(HttpServletRequest request,
                               HttpServletResponse response) {
        String clientId = getClientIdentifier(request);

        log.debug("🚦 Checking rate limit for: {}", clientId);

        boolean allowed = rateLimiter.allowRequest(clientId);

        if (!allowed) {
            log.warn("⛔ Rate limit exceeded for: {}", clientId);
            try {
                response.setStatus(429); // Too Many Requests
                response.setHeader("Retry-After", "60");
                response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
            } catch (IOException e) {
                log.error("Failed to write rate limit response", e);
            }
            return false; // Stop chain
        }

        return true; // Continue
    }

    private String getClientIdentifier(HttpServletRequest request) {
        // Try API key first
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey != null) {
            return "api:" + apiKey;
        }

        // Fall back to IP address
        return "ip:" + request.getRemoteAddr();
    }
}
```

```java
// AuthenticationHandler.java - JWT authentication
package com.saas.handler.impl;

import com.saas.handler.AbstractRequestHandler;
import com.saas.service.JwtService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Slf4j
@RequiredArgsConstructor
public class AuthenticationHandler extends AbstractRequestHandler {

    private final JwtService jwtService;

    @Override
    protected boolean doHandle(HttpServletRequest request,
                               HttpServletResponse response) {
        // Skip authentication for public endpoints
        if (isPublicEndpoint(request.getRequestURI())) {
            return true;
        }

        log.debug("🔐 Authenticating request");

        String token = extractToken(request);

        if (token == null) {
            log.warn("⛔ No authentication token provided");
            sendUnauthorized(response, "Authentication required");
            return false; // Stop chain
        }

        try {
            String userId = jwtService.validateToken(token);
            request.setAttribute("userId", userId);
            log.info("✅ Authenticated user: {}", userId);
            return true; // Continue
        } catch (Exception e) {
            log.warn("⛔ Invalid token: {}", e.getMessage());
            sendUnauthorized(response, "Invalid or expired token");
            return false; // Stop chain
        }
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }

    private boolean isPublicEndpoint(String uri) {
        return uri.startsWith("/api/public/") ||
               uri.equals("/api/health") ||
               uri.equals("/api/auth/login");
    }

    private void sendUnauthorized(HttpServletResponse response, String message) {
        try {
            response.setStatus(401);
            response.setContentType("application/json");
            response.getWriter().write(
                String.format("{\"error\": \"%s\"}", message)
            );
        } catch (IOException e) {
            log.error("Failed to write unauthorized response", e);
        }
    }
}
```

```java
// ValidationHandler.java - Request validation
package com.saas.handler.impl;

import com.saas.handler.AbstractRequestHandler;
import lombok.extern.slf4j.Slf4j;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Slf4j
public class ValidationHandler extends AbstractRequestHandler {

    @Override
    protected boolean doHandle(HttpServletRequest request,
                               HttpServletResponse response) {
        log.debug("✔️  Validating request");

        // Validate Content-Type for POST/PUT
        if (("POST".equals(request.getMethod()) ||
             "PUT".equals(request.getMethod()))) {
            String contentType = request.getContentType();

            if (contentType == null ||
                !contentType.contains("application/json")) {
                log.warn("⛔ Invalid Content-Type: {}", contentType);
                sendBadRequest(response, "Content-Type must be application/json");
                return false;
            }
        }

        // Validate required headers
        if (request.getHeader("X-Request-ID") == null) {
            log.warn("⛔ Missing X-Request-ID header");
            sendBadRequest(response, "X-Request-ID header required");
            return false;
        }

        return true; // Continue
    }

    private void sendBadRequest(HttpServletResponse response, String message) {
        try {
            response.setStatus(400);
            response.setContentType("application/json");
            response.getWriter().write(
                String.format("{\"error\": \"%s\"}", message)
            );
        } catch (IOException e) {
            log.error("Failed to write bad request response", e);
        }
    }
}
```

#### Step 4: Build and Configure Chain

```java
// HandlerChainFactory.java - Builds the handler chain
package com.saas.config;

import com.saas.handler.RequestHandler;
import com.saas.handler.impl.*;
import com.saas.service.JwtService;
import com.saas.service.RateLimiterService;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HandlerChainFactory {

    @Value("${cors.allowed-origins}")
    private String allowedOrigins;

    @Bean
    public RequestHandler requestHandlerChain(
            JwtService jwtService,
            RateLimiterService rateLimiter) {

        // Build chain: Logging → CORS → RateLimit → Auth → Validation
        RequestHandler loggingHandler = new LoggingHandler();
        RequestHandler corsHandler = new CorsHandler(allowedOrigins);
        RequestHandler rateLimitHandler = new RateLimitHandler(rateLimiter);
        RequestHandler authHandler = new AuthenticationHandler(jwtService);
        RequestHandler validationHandler = new ValidationHandler();

        // Link handlers
        loggingHandler.setNext(corsHandler);
        corsHandler.setNext(rateLimitHandler);
        rateLimitHandler.setNext(authHandler);
        authHandler.setNext(validationHandler);

        return loggingHandler; // Return head of chain
    }
}
```

#### Step 5: Use Chain in Filter

```java
// RequestHandlerFilter.java - Servlet filter
package com.saas.filter;

import com.saas.handler.RequestHandler;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Component
@RequiredArgsConstructor
@Slf4j
public class RequestHandlerFilter implements Filter {

    private final RequestHandler handlerChain;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        // Pass through handler chain
        boolean shouldContinue = handlerChain.handle(httpRequest, httpResponse);

        if (shouldContinue) {
            // Continue with servlet chain
            chain.doFilter(request, response);

            // Log response
            logResponse(httpRequest, httpResponse);
        }
        // If shouldContinue is false, handler already sent response
    }

    private void logResponse(HttpServletRequest request,
                            HttpServletResponse response) {
        Long startTime = (Long) request.getAttribute("startTime");
        if (startTime != null) {
            long duration = System.currentTimeMillis() - startTime;
            log.info("📤 Response: {} {} - {} ({}ms)",
                request.getMethod(),
                request.getRequestURI(),
                response.getStatus(),
                duration);
        }
    }
}
```

### Scenario 2: Validation Chain

```java
// Validator.java - Validator interface
package com.saas.validation;

public interface Validator<T> {
    ValidationResult validate(T data);
    void setNext(Validator<T> next);
}
```

```java
// ValidationResult.java
package com.saas.validation;

import lombok.Getter;
import java.util.ArrayList;
import java.util.List;

@Getter
public class ValidationResult {
    private final boolean valid;
    private final List<String> errors;

    private ValidationResult(boolean valid, List<String> errors) {
        this.valid = valid;
        this.errors = errors;
    }

    public static ValidationResult success() {
        return new ValidationResult(true, new ArrayList<>());
    }

    public static ValidationResult failure(String error) {
        List<String> errors = new ArrayList<>();
        errors.add(error);
        return new ValidationResult(false, errors);
    }

    public static ValidationResult failure(List<String> errors) {
        return new ValidationResult(false, errors);
    }
}
```

```java
// AbstractValidator.java - Base validator
package com.saas.validation;

public abstract class AbstractValidator<T> implements Validator<T> {

    protected Validator<T> next;

    @Override
    public void setNext(Validator<T> next) {
        this.next = next;
    }

    @Override
    public ValidationResult validate(T data) {
        // Validate with this validator
        ValidationResult result = doValidate(data);

        // If valid and there's a next validator, continue chain
        if (result.isValid() && next != null) {
            return next.validate(data);
        }

        return result;
    }

    protected abstract ValidationResult doValidate(T data);
}
```

```java
// UserRegistrationDTO.java - Data to validate
package com.saas.dto;

import lombok.Data;

@Data
public class UserRegistrationDTO {
    private String email;
    private String password;
    private String name;
    private String phoneNumber;
    private boolean agreedToTerms;
}
```

```java
// EmailFormatValidator.java
package com.saas.validation.impl;

import com.saas.dto.UserRegistrationDTO;
import com.saas.validation.AbstractValidator;
import com.saas.validation.ValidationResult;
import java.util.regex.Pattern;

public class EmailFormatValidator extends AbstractValidator<UserRegistrationDTO> {

    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$");

    @Override
    protected ValidationResult doValidate(UserRegistrationDTO data) {
        if (data.getEmail() == null || data.getEmail().trim().isEmpty()) {
            return ValidationResult.failure("Email is required");
        }

        if (!EMAIL_PATTERN.matcher(data.getEmail()).matches()) {
            return ValidationResult.failure("Invalid email format");
        }

        return ValidationResult.success();
    }
}
```

```java
// PasswordStrengthValidator.java
package com.saas.validation.impl;

import com.saas.dto.UserRegistrationDTO;
import com.saas.validation.AbstractValidator;
import com.saas.validation.ValidationResult;

public class PasswordStrengthValidator extends AbstractValidator<UserRegistrationDTO> {

    @Override
    protected ValidationResult doValidate(UserRegistrationDTO data) {
        String password = data.getPassword();

        if (password == null || password.length() < 8) {
            return ValidationResult.failure("Password must be at least 8 characters");
        }

        boolean hasUpper = password.chars().anyMatch(Character::isUpperCase);
        boolean hasLower = password.chars().anyMatch(Character::isLowerCase);
        boolean hasDigit = password.chars().anyMatch(Character::isDigit);

        if (!hasUpper || !hasLower || !hasDigit) {
            return ValidationResult.failure(
                "Password must contain uppercase, lowercase, and digit"
            );
        }

        return ValidationResult.success();
    }
}
```

```java
// TermsAgreementValidator.java
package com.saas.validation.impl;

import com.saas.dto.UserRegistrationDTO;
import com.saas.validation.AbstractValidator;
import com.saas.validation.ValidationResult;

public class TermsAgreementValidator extends AbstractValidator<UserRegistrationDTO> {

    @Override
    protected ValidationResult doValidate(UserRegistrationDTO data) {
        if (!data.isAgreedToTerms()) {
            return ValidationResult.failure("You must agree to terms and conditions");
        }

        return ValidationResult.success();
    }
}
```

```java
// UniqueEmailValidator.java
package com.saas.validation.impl;

import com.saas.dto.UserRegistrationDTO;
import com.saas.repository.UserRepository;
import com.saas.validation.AbstractValidator;
import com.saas.validation.ValidationResult;
import lombok.RequiredArgsConstructor;

@RequiredArgsConstructor
public class UniqueEmailValidator extends AbstractValidator<UserRegistrationDTO> {

    private final UserRepository userRepository;

    @Override
    protected ValidationResult doValidate(UserRegistrationDTO data) {
        boolean exists = userRepository.existsByEmail(data.getEmail());

        if (exists) {
            return ValidationResult.failure("Email already registered");
        }

        return ValidationResult.success();
    }
}
```

```java
// UserRegistrationService.java - Using validation chain
package com.saas.service;

import com.saas.dto.UserRegistrationDTO;
import com.saas.repository.UserRepository;
import com.saas.validation.Validator;
import com.saas.validation.ValidationResult;
import com.saas.validation.impl.*;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserRegistrationService {

    private final UserRepository userRepository;

    public ValidationResult validateRegistration(UserRegistrationDTO data) {
        // Build validation chain
        Validator<UserRegistrationDTO> chain = buildValidationChain();

        // Execute chain
        return chain.validate(data);
    }

    private Validator<UserRegistrationDTO> buildValidationChain() {
        // Chain: Email Format → Password → Terms → Unique Email
        Validator<UserRegistrationDTO> emailValidator = new EmailFormatValidator();
        Validator<UserRegistrationDTO> passwordValidator = new PasswordStrengthValidator();
        Validator<UserRegistrationDTO> termsValidator = new TermsAgreementValidator();
        Validator<UserRegistrationDTO> uniqueValidator = new UniqueEmailValidator(userRepository);

        emailValidator.setNext(passwordValidator);
        passwordValidator.setNext(termsValidator);
        termsValidator.setNext(uniqueValidator);

        return emailValidator; // Return head
    }
}
```

---

## 💻 Implementation in Node.js

### Scenario 1: Express Middleware Chain

Express middleware is a perfect example of Chain of Responsibility!

```typescript
// middleware/types.ts
import { Request, Response, NextFunction } from "express";

export type Middleware = (
  req: Request,
  res: Response,
  next: NextFunction
) => void | Promise<void>;
```

```typescript
// middleware/loggingMiddleware.ts
import { Request, Response, NextFunction } from "express";

export function loggingMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const startTime = Date.now();

  console.log(`📝 ${req.method} ${req.path} from ${req.ip}`);

  // Store start time for response logging
  res.on("finish", () => {
    const duration = Date.now() - startTime;
    console.log(
      `📤 ${req.method} ${req.path} - ${res.statusCode} (${duration}ms)`
    );
  });

  next(); // Pass to next handler
}
```

```typescript
// middleware/corsMiddleware.ts
import { Request, Response, NextFunction } from "express";

export function corsMiddleware(allowedOrigins: string) {
  return (req: Request, res: Response, next: NextFunction): void => {
    console.log("🌐 Setting CORS headers");

    res.setHeader("Access-Control-Allow-Origin", allowedOrigins);
    res.setHeader(
      "Access-Control-Allow-Methods",
      "GET, POST, PUT, DELETE, OPTIONS"
    );
    res.setHeader(
      "Access-Control-Allow-Headers",
      "Content-Type, Authorization"
    );

    // Handle preflight
    if (req.method === "OPTIONS") {
      res.sendStatus(200);
      return; // Stop chain
    }

    next(); // Continue
  };
}
```

```typescript
// middleware/rateLimitMiddleware.ts
import { Request, Response, NextFunction } from "express";
import { RateLimiterService } from "../services/rateLimiterService";

export function rateLimitMiddleware(rateLimiter: RateLimiterService) {
  return async (
    req: Request,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    const clientId = (req.headers["x-api-key"] as string) || req.ip;

    console.log(`🚦 Checking rate limit for: ${clientId}`);

    const allowed = await rateLimiter.allowRequest(clientId);

    if (!allowed) {
      console.warn(`⛔ Rate limit exceeded for: ${clientId}`);
      res.status(429).json({ error: "Rate limit exceeded" });
      return; // Stop chain
    }

    next(); // Continue
  };
}
```

```typescript
// middleware/authenticationMiddleware.ts
import { Request, Response, NextFunction } from "express";
import { JwtService } from "../services/jwtService";

const PUBLIC_ENDPOINTS = ["/api/public", "/api/health", "/api/auth/login"];

export function authenticationMiddleware(jwtService: JwtService) {
  return async (
    req: Request,
    res: Response,
    next: NextFunction
  ): Promise<void> => {
    // Skip for public endpoints
    if (PUBLIC_ENDPOINTS.some((endpoint) => req.path.startsWith(endpoint))) {
      return next();
    }

    console.log("🔐 Authenticating request");

    const token = extractToken(req);

    if (!token) {
      console.warn("⛔ No authentication token");
      res.status(401).json({ error: "Authentication required" });
      return; // Stop chain
    }

    try {
      const userId = await jwtService.validateToken(token);
      req.userId = userId; // Attach to request
      console.log(`✅ Authenticated user: ${userId}`);
      next(); // Continue
    } catch (error) {
      console.warn(`⛔ Invalid token: ${error.message}`);
      res.status(401).json({ error: "Invalid or expired token" });
      // Stop chain
    }
  };
}

function extractToken(req: Request): string | null {
  const header = req.headers.authorization;
  if (header && header.startsWith("Bearer ")) {
    return header.substring(7);
  }
  return null;
}
```

```typescript
// middleware/validationMiddleware.ts
import { Request, Response, NextFunction } from "express";

export function validationMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  console.log("✔️  Validating request");

  // Validate Content-Type for POST/PUT
  if (["POST", "PUT"].includes(req.method)) {
    const contentType = req.headers["content-type"];

    if (!contentType || !contentType.includes("application/json")) {
      console.warn(`⛔ Invalid Content-Type: ${contentType}`);
      res.status(400).json({ error: "Content-Type must be application/json" });
      return; // Stop chain
    }
  }

  // Validate required headers
  if (!req.headers["x-request-id"]) {
    console.warn("⛔ Missing X-Request-ID header");
    res.status(400).json({ error: "X-Request-ID header required" });
    return; // Stop chain
  }

  next(); // Continue
}
```

```typescript
// app.ts - Chain middleware together
import express from "express";
import { loggingMiddleware } from "./middleware/loggingMiddleware";
import { corsMiddleware } from "./middleware/corsMiddleware";
import { rateLimitMiddleware } from "./middleware/rateLimitMiddleware";
import { authenticationMiddleware } from "./middleware/authenticationMiddleware";
import { validationMiddleware } from "./middleware/validationMiddleware";
import { RateLimiterService } from "./services/rateLimiterService";
import { JwtService } from "./services/jwtService";

const app = express();
const rateLimiter = new RateLimiterService();
const jwtService = new JwtService();

// Build middleware chain
// Order matters! Logging → CORS → RateLimit → Auth → Validation
app.use(loggingMiddleware);
app.use(corsMiddleware("*"));
app.use(rateLimitMiddleware(rateLimiter));
app.use(authenticationMiddleware(jwtService));
app.use(validationMiddleware);

// Your routes here
app.get("/api/users", (req, res) => {
  res.json({ users: [] });
});

app.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

### Scenario 2: Custom Validation Chain

```typescript
// validator.ts - Validator interface

export interface Validator<T> {
  validate(data: T): ValidationResult;
  setNext(validator: Validator<T>): Validator<T>;
}
export interface ValidationResult {
  valid: boolean;
  errors: string[];
}
```

```typescript
// abstractValidator.ts - Base validator
export abstract class AbstractValidator<T> implements Validator<T> {
  protected next: Validator<T> | null = null;

  setNext(validator: Validator<T>): Validator<T> {
    this.next = validator;
    return validator; // Allow chaining
  }

  validate(data: T): ValidationResult {
    // Validate with this validator
    const result = this.doValidate(data);

    // If valid and there's a next validator, continue chain
    if (result.valid && this.next) {
      return this.next.validate(data);
    }

    return result;
  }

  protected abstract doValidate(data: T): ValidationResult;

  protected success(): ValidationResult {
    return { valid: true, errors: [] };
  }

  protected failure(error: string): ValidationResult {
    return { valid: false, errors: [error] };
  }
}
```

```typescript
// userRegistration.ts - Data type
export interface UserRegistration {
  email: string;
  password: string;
  name: string;
  phoneNumber?: string;
  agreedToTerms: boolean;
}
```

```typescript
// emailFormatValidator.ts
import { AbstractValidator } from "./abstractValidator";
import { UserRegistration } from "./userRegistration";
import { ValidationResult } from "./validator";

export class EmailFormatValidator extends AbstractValidator<UserRegistration> {
  private emailRegex = /^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$/;

  protected doValidate(data: UserRegistration): ValidationResult {
    if (!data.email || data.email.trim() === "") {
      return this.failure("Email is required");
    }

    if (!this.emailRegex.test(data.email)) {
      return this.failure("Invalid email format");
    }

    return this.success();
  }
}
```

```typescript
// passwordStrengthValidator.ts
import { AbstractValidator } from "./abstractValidator";
import { UserRegistration } from "./userRegistration";
import { ValidationResult } from "./validator";

export class PasswordStrengthValidator extends AbstractValidator<UserRegistration> {
  protected doValidate(data: UserRegistration): ValidationResult {
    const { password } = data;

    if (!password || password.length < 8) {
      return this.failure("Password must be at least 8 characters");
    }

    const hasUpper = /[A-Z]/.test(password);
    const hasLower = /[a-z]/.test(password);
    const hasDigit = /\d/.test(password);

    if (!hasUpper || !hasLower || !hasDigit) {
      return this.failure(
        "Password must contain uppercase, lowercase, and digit"
      );
    }

    return this.success();
  }
}
```

```typescript
// termsAgreementValidator.ts
import { AbstractValidator } from "./abstractValidator";
import { UserRegistration } from "./userRegistration";
import { ValidationResult } from "./validator";

export class TermsAgreementValidator extends AbstractValidator<UserRegistration> {
  protected doValidate(data: UserRegistration): ValidationResult {
    if (!data.agreedToTerms) {
      return this.failure("You must agree to terms and conditions");
    }

    return this.success();
  }
}
```

```typescript
// uniqueEmailValidator.ts
import { AbstractValidator } from "./abstractValidator";
import { UserRegistration } from "./userRegistration";
import { ValidationResult } from "./validator";
import { UserRepository } from "../repositories/userRepository";

export class UniqueEmailValidator extends AbstractValidator<UserRegistration> {
  constructor(private userRepository: UserRepository) {
    super();
  }

  protected async doValidate(
    data: UserRegistration
  ): Promise<ValidationResult> {
    const exists = await this.userRepository.existsByEmail(data.email);

    if (exists) {
      return this.failure("Email already registered");
    }

    return this.success();
  }
}
```

```typescript
// userRegistrationService.ts - Using validation chain
import { Validator } from "./validator";
import { UserRegistration } from "./userRegistration";
import { EmailFormatValidator } from "./emailFormatValidator";
import { PasswordStrengthValidator } from "./passwordStrengthValidator";
import { TermsAgreementValidator } from "./termsAgreementValidator";
import { UniqueEmailValidator } from "./uniqueEmailValidator";
import { UserRepository } from "../repositories/userRepository";

export class UserRegistrationService {
  constructor(private userRepository: UserRepository) {}

  async validateRegistration(data: UserRegistration) {
    const chain = this.buildValidationChain();
    return chain.validate(data);
  }

  private buildValidationChain(): Validator<UserRegistration> {
    // Build chain with fluent API
    const emailValidator = new EmailFormatValidator();

    emailValidator
      .setNext(new PasswordStrengthValidator())
      .setNext(new TermsAgreementValidator())
      .setNext(new UniqueEmailValidator(this.userRepository));

    return emailValidator; // Return head of chain
  }
}
```

```typescript
// Usage example
import { UserRegistrationService } from "./userRegistrationService";
import { UserRepository } from "../repositories/userRepository";

const userRepo = new UserRepository();
const registrationService = new UserRegistrationService(userRepo);

const userData = {
  email: "user@example.com",
  password: "SecurePass123",
  name: "John Doe",
  agreedToTerms: true,
};

const result = await registrationService.validateRegistration(userData);

if (result.valid) {
  console.log("✅ Validation passed");
} else {
  console.log("❌ Validation failed:", result.errors);
}
```

---

## 💻 Implementation in Go

### Scenario 1: HTTP Middleware Chain

```go
// handler.go - Handler interface
package middleware

import "net/http"

// Handler is the interface for request handlers
type Handler interface {
    Handle(w http.ResponseWriter, r *http.Request) bool
    SetNext(handler Handler)
}

// BaseHandler provides default chain behavior
type BaseHandler struct {
    next Handler
}

func (h *BaseHandler) SetNext(handler Handler) {
    h.next = handler
}

func (h *BaseHandler) CallNext(w http.ResponseWriter, r *http.Request) bool {
    if h.next != nil {
        return h.next.Handle(w, r)
    }
    return true
}
```

```go
// logging_handler.go
package middleware

import (
    "fmt"
    "net/http"
    "time"
)

type LoggingHandler struct {
    BaseHandler
}

func NewLoggingHandler() *LoggingHandler {
    return &LoggingHandler{}
}

func (h *LoggingHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    startTime := time.Now()

    fmt.Printf("📝 %s %s from %s\n", r.Method, r.URL.Path, r.RemoteAddr)

    // Wrap ResponseWriter to capture status code
    wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

    // Continue chain
    result := h.CallNext(wrapped, r)

    duration := time.Since(startTime)
    fmt.Printf("📤 %s %s - %d (%v)\n",
        r.Method, r.URL.Path, wrapped.statusCode, duration)

    return result
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

```go
// cors_handler.go
package middleware

import (
    "fmt"
    "net/http"
)

type CorsHandler struct {
    BaseHandler
    allowedOrigins string
}

func NewCorsHandler(allowedOrigins string) *CorsHandler {
    return &CorsHandler{
        allowedOrigins: allowedOrigins,
    }
}

func (h *CorsHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    fmt.Println("🌐 Setting CORS headers")

    w.Header().Set("Access-Control-Allow-Origin", h.allowedOrigins)
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

    // Handle preflight
    if r.Method == "OPTIONS" {
        w.WriteHeader(http.StatusOK)
        return false // Stop chain
    }

    return h.CallNext(w, r) // Continue
}
```

```go
// rate_limit_handler.go
package middleware

import (
    "encoding/json"
    "fmt"
    "myapp/service"
    "net/http"
)

type RateLimitHandler struct {
    BaseHandler
    rateLimiter *service.RateLimiterService
}

func NewRateLimitHandler(rateLimiter *service.RateLimiterService) *RateLimitHandler {
    return &RateLimitHandler{
        rateLimiter: rateLimiter,
    }
}

func (h *RateLimitHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    clientID := h.getClientIdentifier(r)

    fmt.Printf("🚦 Checking rate limit for: %s\n", clientID)

    allowed := h.rateLimiter.AllowRequest(clientID)

    if !allowed {
        fmt.Printf("⛔ Rate limit exceeded for: %s\n", clientID)
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusTooManyRequests)
        json.NewEncoder(w).Encode(map[string]string{
            "error": "Rate limit exceeded",
        })
        return false // Stop chain
    }

    return h.CallNext(w, r) // Continue
}

func (h *RateLimitHandler) getClientIdentifier(r *http.Request) string {
    // Try API key first
    apiKey := r.Header.Get("X-API-Key")
    if apiKey != "" {
        return "api:" + apiKey
    }

    // Fall back to IP
    return "ip:" + r.RemoteAddr
}
```

```go
// authentication_handler.go
package middleware

import (
    "encoding/json"
    "fmt"
    "myapp/service"
    "net/http"
    "strings"
)

type AuthenticationHandler struct {
    BaseHandler
    jwtService *service.JwtService
}

func NewAuthenticationHandler(jwtService *service.JwtService) *AuthenticationHandler {
    return &AuthenticationHandler{
        jwtService: jwtService,
    }
}

func (h *AuthenticationHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    // Skip for public endpoints
    if h.isPublicEndpoint(r.URL.Path) {
        return h.CallNext(w, r)
    }

    fmt.Println("🔐 Authenticating request")

    token := h.extractToken(r)

    if token == "" {
        fmt.Println("⛔ No authentication token")
        h.sendUnauthorized(w, "Authentication required")
        return false // Stop chain
    }

    userID, err := h.jwtService.ValidateToken(token)
    if err != nil {
        fmt.Printf("⛔ Invalid token: %v\n", err)
        h.sendUnauthorized(w, "Invalid or expired token")
        return false // Stop chain
    }

    // Store user ID in context (simplified - use context.Context in production)
    r.Header.Set("X-User-ID", userID)
    fmt.Printf("✅ Authenticated user: %s\n", userID)

    return h.CallNext(w, r) // Continue
}

func (h *AuthenticationHandler) extractToken(r *http.Request) string {
    header := r.Header.Get("Authorization")
    if header != "" && strings.HasPrefix(header, "Bearer ") {
        return header[7:]
    }
    return ""
}

func (h *AuthenticationHandler) isPublicEndpoint(path string) bool {
    publicPaths := []string{"/api/public/", "/api/health", "/api/auth/login"}
    for _, prefix := range publicPaths {
        if strings.HasPrefix(path, prefix) {
            return true
        }
    }
    return false
}

func (h *AuthenticationHandler) sendUnauthorized(w http.ResponseWriter, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusUnauthorized)
    json.NewEncoder(w).Encode(map[string]string{
        "error": message,
    })
}
```

```go
// validation_handler.go
package middleware

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
)

type ValidationHandler struct {
    BaseHandler
}

func NewValidationHandler() *ValidationHandler {
    return &ValidationHandler{}
}

func (h *ValidationHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    fmt.Println("✔️  Validating request")

    // Validate Content-Type for POST/PUT
    if r.Method == "POST" || r.Method == "PUT" {
        contentType := r.Header.Get("Content-Type")

        if !strings.Contains(contentType, "application/json") {
            fmt.Printf("⛔ Invalid Content-Type: %s\n", contentType)
            h.sendBadRequest(w, "Content-Type must be application/json")
            return false // Stop chain
        }
    }

    // Validate required headers
    if r.Header.Get("X-Request-ID") == "" {
        fmt.Println("⛔ Missing X-Request-ID header")
        h.sendBadRequest(w, "X-Request-ID header required")
        return false // Stop chain
    }

    return h.CallNext(w, r) // Continue
}

func (h *ValidationHandler) sendBadRequest(w http.ResponseWriter, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusBadRequest)
    json.NewEncoder(w).Encode(map[string]string{
        "error": message,
    })
}
```

```go
// chain.go - Build and use the chain
package middleware

import (
    "myapp/service"
)

// BuildHandlerChain creates the middleware chain
func BuildHandlerChain(
    jwtService *service.JwtService,
    rateLimiter *service.RateLimiterService,
) Handler {
    // Create handlers
    loggingHandler := NewLoggingHandler()
    corsHandler := NewCorsHandler("*")
    rateLimitHandler := NewRateLimitHandler(rateLimiter)
    authHandler := NewAuthenticationHandler(jwtService)
    validationHandler := NewValidationHandler()

    // Link handlers: Logging → CORS → RateLimit → Auth → Validation
    loggingHandler.SetNext(corsHandler)
    corsHandler.SetNext(rateLimitHandler)
    rateLimitHandler.SetNext(authHandler)
    authHandler.SetNext(validationHandler)

    return loggingHandler // Return head of chain
}
```

```go
// main.go - Using the chain
package main

import (
    "fmt"
    "myapp/middleware"
    "myapp/service"
    "net/http"
)

func main() {
    // Create services
    jwtService := service.NewJwtService()
    rateLimiter := service.NewRateLimiterService()

    // Build middleware chain
    handlerChain := middleware.BuildHandlerChain(jwtService, rateLimiter)

    // Wrap HTTP handler with chain
    http.HandleFunc("/api/users", func(w http.ResponseWriter, r *http.Request) {
        // Execute middleware chain
        shouldContinue := handlerChain.Handle(w, r)

        if !shouldContinue {
            return // Chain stopped, response already sent
        }

        // Handle actual request
        w.Header().Set("Content-Type", "application/json")
        w.Write([]byte(`{"users": []}`))
    })

    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

### Scenario 2: Validation Chain

```go
// validator.go - Validator interface
package validation

// Validator interface for validation chain
type Validator interface {
    Validate(data interface{}) ValidationResult
    SetNext(validator Validator) Validator
}

// ValidationResult holds validation outcome
type ValidationResult struct {
    Valid  bool
    Errors []string
}

// Success creates successful validation result
func Success() ValidationResult {
    return ValidationResult{Valid: true, Errors: []string{}}
}

// Failure creates failed validation result
func Failure(error string) ValidationResult {
    return ValidationResult{Valid: false, Errors: []string{error}}
}
```

```go
// base_validator.go - Base validator
package validation

// BaseValidator provides common chain behavior
type BaseValidator struct {
    next Validator
}

func (v *BaseValidator) SetNext(validator Validator) Validator {
    v.next = validator
    return validator // Allow fluent chaining
}

func (v *BaseValidator) CallNext(data interface{}) ValidationResult {
    if v.next != nil {
        return v.next.Validate(data)
    }
    return Success()
}
```

```go
// user_registration.go - Data type
package model

type UserRegistration struct {
    Email         string
    Password      string
    Name          string
    PhoneNumber   string
    AgreedToTerms bool
}
```

```go
// email_validator.go
package validation

import (
    "myapp/model"
    "regexp"
)

type EmailFormatValidator struct {
    BaseValidator
}

var emailRegex = regexp.MustCompile(`^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,})

func NewEmailFormatValidator() *EmailFormatValidator {
    return &EmailFormatValidator{}
}

func (v *EmailFormatValidator) Validate(data interface{}) ValidationResult {
    user := data.(*model.UserRegistration)

    if user.Email == "" {
        return Failure("Email is required")
    }

    if !emailRegex.MatchString(user.Email) {
        return Failure("Invalid email format")
    }

    // Continue chain
    return v.CallNext(data)
}
```

```go
// password_validator.go
package validation

import (
    "myapp/model"
    "unicode"
)

type PasswordStrengthValidator struct {
    BaseValidator
}

func NewPasswordStrengthValidator() *PasswordStrengthValidator {
    return &PasswordStrengthValidator{}
}

func (v *PasswordStrengthValidator) Validate(data interface{}) ValidationResult {
    user := data.(*model.UserRegistration)

    if len(user.Password) < 8 {
        return Failure("Password must be at least 8 characters")
    }

    var hasUpper, hasLower, hasDigit bool

    for _, char := range user.Password {
        if unicode.IsUpper(char) {
            hasUpper = true
        }
        if unicode.IsLower(char) {
            hasLower = true
        }
        if unicode.IsDigit(char) {
            hasDigit = true
        }
    }

    if !hasUpper || !hasLower || !hasDigit {
        return Failure("Password must contain uppercase, lowercase, and digit")
    }

    // Continue chain
    return v.CallNext(data)
}
```

```go
// terms_validator.go
package validation

import "myapp/model"

type TermsAgreementValidator struct {
    BaseValidator
}

func NewTermsAgreementValidator() *TermsAgreementValidator {
    return &TermsAgreementValidator{}
}

func (v *TermsAgreementValidator) Validate(data interface{}) ValidationResult {
    user := data.(*model.UserRegistration)

    if !user.AgreedToTerms {
        return Failure("You must agree to terms and conditions")
    }

    // Continue chain
    return v.CallNext(data)
}
```

```go
// unique_email_validator.go
package validation

import (
    "myapp/model"
    "myapp/repository"
)

type UniqueEmailValidator struct {
    BaseValidator
    userRepo *repository.UserRepository
}

func NewUniqueEmailValidator(userRepo *repository.UserRepository) *UniqueEmailValidator {
    return &UniqueEmailValidator{
        userRepo: userRepo,
    }
}

func (v *UniqueEmailValidator) Validate(data interface{}) ValidationResult {
    user := data.(*model.UserRegistration)

    exists, err := v.userRepo.ExistsByEmail(user.Email)
    if err != nil {
        return Failure("Failed to check email uniqueness")
    }

    if exists {
        return Failure("Email already registered")
    }

    // Continue chain (if any)
    return v.CallNext(data)
}
```

```go
// registration_service.go - Using validation chain
package service

import (
    "myapp/model"
    "myapp/repository"
    "myapp/validation"
)

type UserRegistrationService struct {
    userRepo *repository.UserRepository
}

func NewUserRegistrationService(userRepo *repository.UserRepository) *UserRegistrationService {
    return &UserRegistrationService{
        userRepo: userRepo,
    }
}

func (s *UserRegistrationService) ValidateRegistration(data *model.UserRegistration) validation.ValidationResult {
    chain := s.buildValidationChain()
    return chain.Validate(data)
}

func (s *UserRegistrationService) buildValidationChain() validation.Validator {
    // Build chain with fluent API
    emailValidator := validation.NewEmailFormatValidator()

    emailValidator.
        SetNext(validation.NewPasswordStrengthValidator()).
        SetNext(validation.NewTermsAgreementValidator()).
        SetNext(validation.NewUniqueEmailValidator(s.userRepo))

    return emailValidator // Return head of chain
}

// RegisterUser performs registration after validation
func (s *UserRegistrationService) RegisterUser(data *model.UserRegistration) error {
    // Validate first
    result := s.ValidateRegistration(data)

    if !result.Valid {
        return fmt.Errorf("validation failed: %v", result.Errors)
    }

    // Proceed with registration
    return s.userRepo.Create(data)
}
```

```go
// main.go - Usage example
package main

import (
    "fmt"
    "myapp/model"
    "myapp/repository"
    "myapp/service"
)

func main() {
    // Create repository
    userRepo := repository.NewUserRepository()

    // Create service
    registrationService := service.NewUserRegistrationService(userRepo)

    // Test data
    userData := &model.UserRegistration{
        Email:         "user@example.com",
        Password:      "SecurePass123",
        Name:          "John Doe",
        AgreedToTerms: true,
    }

    // Validate
    result := registrationService.ValidateRegistration(userData)

    if result.Valid {
        fmt.Println("✅ Validation passed")

        // Register user
        err := registrationService.RegisterUser(userData)
        if err != nil {
            fmt.Printf("❌ Registration failed: %v\n", err)
        } else {
            fmt.Println("✅ User registered successfully")
        }
    } else {
        fmt.Printf("❌ Validation failed: %v\n", result.Errors)
    }
}
```

---

## ✅ Best Practices

### 1. **Keep Handlers Single-Purpose**

Each handler should have ONE clear responsibility.

```java
// ✅ GOOD - Single responsibility
public class AuthenticationHandler extends AbstractRequestHandler {
    // Only handles authentication
}

public class RateLimitHandler extends AbstractRequestHandler {
    // Only handles rate limiting
}

// ❌ BAD - Multiple responsibilities
public class SecurityHandler extends AbstractRequestHandler {
    // Handles auth, rate limiting, CORS, validation - too much!
}
```

### 2. **Make Chain Order Explicit**

Document and enforce the correct order of handlers.

```typescript
// ✅ GOOD - Clear order documentation
/**
 * Middleware chain order (IMPORTANT):
 * 1. Logging - must be first to capture all requests
 * 2. CORS - before any auth checks
 * 3. RateLimit - prevent abuse before expensive operations
 * 4. Authentication - verify identity
 * 5. Validation - validate authenticated requests
 */
app.use(loggingMiddleware);
app.use(corsMiddleware);
app.use(rateLimitMiddleware);
app.use(authenticationMiddleware);
app.use(validationMiddleware);
```

### 3. **Provide Clear Stop Conditions**

Make it obvious when and why the chain stops.

```go
// ✅ GOOD - Clear stop with reason
func (h *AuthHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    if !isAuthenticated(r) {
        sendUnauthorized(w, "Authentication required")
        return false // STOP: Not authenticated
    }
    return h.CallNext(w, r) // CONTINUE
}

// ❌ BAD - Unclear stopping
func (h *AuthHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    if someCondition {
        doSomething()
        return false // Why stop? Not clear
    }
    return true
}
```

### 4. **Allow Optional Handlers**

Make handlers skippable based on configuration.

```java
// ✅ GOOD - Conditional handler
public class RateLimitHandler extends AbstractRequestHandler {
    private final boolean enabled;

    @Override
    protected boolean doHandle(Request req, Response res) {
        if (!enabled) {
            return true; // Skip if disabled
        }

        // Rate limit logic
    }
}
```

### 5. **Implement Proper Logging**

Log when handlers execute and their decisions.

```typescript
// ✅ GOOD - Comprehensive logging
protected doHandle(req: Request, res: Response): boolean {
  console.log(`[${this.constructor.name}] Processing request`);

  const result = this.processRequest(req, res);

  if (result) {
    console.log(`[${this.constructor.name}] ✅ Passed, continuing chain`);
  } else {
    console.log(`[${this.constructor.name}] ⛔ Stopped chain`);
  }

  return result;
}
```

### 6. **Use Fluent API for Chain Building**

Make chain construction readable.

```java
// ✅ GOOD - Fluent API
Handler chain = new LoggingHandler()
    .setNext(new CorsHandler())
    .setNext(new RateLimitHandler())
    .setNext(new AuthHandler())
    .setNext(new ValidationHandler());

// ❌ BAD - Verbose
Handler h1 = new LoggingHandler();
Handler h2 = new CorsHandler();
Handler h3 = new RateLimitHandler();
h1.setNext(h2);
h2.setNext(h3);
// Hard to read the chain flow
```

### 7. **Handle Errors Gracefully**

Don't let one handler crash the entire chain.

```typescript
// ✅ GOOD - Error handling
async handle(req: Request, res: Response, next: NextFunction) {
  try {
    await this.processRequest(req, res);
    next(); // Continue chain
  } catch (error) {
    console.error(`Handler error: ${error.message}`);
    res.status(500).json({ error: 'Internal server error' });
    // Don't call next() - stop chain on error
  }
}
```

### 8. **Make Handlers Testable**

Each handler should be independently testable.

```go
// ✅ GOOD - Testable handler
func TestAuthenticationHandler(t *testing.T) {
    handler := NewAuthenticationHandler(mockJwtService)

    req := httptest.NewRequest("GET", "/api/users", nil)
    req.Header.Set("Authorization", "Bearer valid-token")
    rec := httptest.NewRecorder()

    result := handler.Handle(rec, req)

    assert.True(t, result) // Should continue
    assert.Equal(t, "user123", req.Header.Get("X-User-ID"))
}
```

### 9. **Avoid Circular References**

Prevent handlers from forming loops.

```java
// ❌ BAD - Circular reference
handler1.setNext(handler2);
handler2.setNext(handler3);
handler3.setNext(handler1); // LOOP!

// ✅ GOOD - Linear chain
handler1.setNext(handler2);
handler2.setNext(handler3);
// handler3 has no next (end of chain)
```

### 10. **Document Handler Side Effects**

Clearly document what each handler modifies.

```typescript
/**
 * AuthenticationHandler
 *
 * Side effects:
 * - Adds req.userId if authenticated
 * - Sets response status 401 if not authenticated
 * - Stops chain if authentication fails
 *
 * Dependencies:
 * - Requires Authorization header
 * - Uses JwtService for validation
 */
class AuthenticationHandler implements Handler { ... }
```

---

## 🚫 Common Pitfalls

### 1. **Forgetting to Call Next**

**Problem**: Handler doesn't pass to next, breaking chain.

```typescript
// ❌ PITFALL
handle(req: Request, res: Response, next: NextFunction) {
  console.log('Logging request');
  // Forgot to call next()! Chain stops here
}

// ✅ SOLUTION
handle(req: Request, res: Response, next: NextFunction) {
  console.log('Logging request');
  next(); // Continue chain
}
```

### 2. **Wrong Handler Order**

**Problem**: Handlers in wrong sequence cause issues.

```java
// ❌ PITFALL - Auth before CORS
chain = cors → auth → ...
// CORS will fail for auth-protected endpoints!

// ✅ SOLUTION - CORS first
chain = cors → rateLimit → auth → ...
// CORS headers set before auth check
```

### 3. **Infinite Loops**

**Problem**: Handler calls itself or creates circular reference.

```go
// ❌ PITFALL
func (h *MyHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    // Process request
    return h.Handle(w, r) // INFINITE RECURSION!
}

// ✅ SOLUTION
func (h *MyHandler) Handle(w http.ResponseWriter, r *http.Request) bool {
    // Process request
    return h.CallNext(w, r) // Call next in chain
}
```

### 4. **Stateful Handlers**

**Problem**: Handlers maintain state that causes issues with concurrent requests.

```java
// ❌ PITFALL - Stateful
public class CounterHandler extends AbstractRequestHandler {
    private int requestCount = 0; // Shared state!

    @Override
    protected boolean doHandle(Request req, Response res) {
        requestCount++; // Race condition!
        return true;
    }
}

// ✅ SOLUTION - Stateless or thread-safe
public class CounterHandler extends AbstractRequestHandler {
    private final AtomicInteger requestCount = new AtomicInteger(0);

    @Override
    protected boolean doHandle(Request req, Response res) {
        requestCount.incrementAndGet(); // Thread-safe
        return true;
    }
}
```

### 5. **Not Handling Async Operations**

**Problem**: Async handlers don't wait for completion.

```typescript
// ❌ PITFALL - Doesn't wait
middleware(req, res, next) {
  this.doAsyncWork(req); // Fire and forget
  next(); // Continues immediately!
}

// ✅ SOLUTION - Properly await
async middleware(req, res, next) {
  await this.doAsyncWork(req); // Wait for completion
  next();
}
```

### 6. **Tight Coupling Between Handlers**

**Problem**: Handlers depend on specific other handlers.

```go
// ❌ PITFALL - Tight coupling
type ValidationHandler struct {
    authHandler *AuthenticationHandler // Depends on specific type!
}

// ✅ SOLUTION - Depend on interface
type ValidationHandler struct {
    next Handler // Depends on interface
}
```

### 7. **No Default Behavior**

**Problem**: Chain has no handler for unmatched cases.

```java
// ❌ PITFALL - No default
// Request not handled by any handler in chain - what happens?

// ✅ SOLUTION - Default/fallback handler
public class DefaultHandler extends AbstractRequestHandler {
    @Override
    protected boolean doHandle(Request req, Response res) {
        res.setStatus(404);
        res.getWriter().write("{\"error\": \"Not found\"}");
        return false; // End chain
    }
}

// Add at end of chain
chain.setLast(new DefaultHandler());
```

### 8. **Exception Swallowing**

**Problem**: Handlers catch exceptions but don't propagate or handle them.

```typescript
// ❌ PITFALL
handle(req, res, next) {
  try {
    this.process(req);
    next();
  } catch (error) {
    // Silently swallow error
  }
}

// ✅ SOLUTION - Proper error handling
handle(req, res, next) {
  try {
    this.process(req);
    next();
  } catch (error) {
    console.error('Handler error:', error);
    res.status(500).json({ error: 'Internal error' });
    // Don't call next() - stop on error
  }
}
```

### 9. **Too Many Handlers**

**Problem**: Chain becomes too long and hard to debug.

```java
// ❌ PITFALL - 20+ handlers
chain = h1 → h2 → h3 → ... → h20
// Hard to debug, performance issues

// ✅ SOLUTION - Group related handlers
chain = loggingHandler → securityHandlerGroup → validationHandlerGroup
// SecurityHandlerGroup internally chains auth, rate limit, cors
```

### 10. **Not Testing the Chain**

**Problem**: Individual handlers tested but not the chain as a whole.

```typescript
// ✅ SOLUTION - Test entire chain
describe("Middleware Chain", () => {
  it("should execute all handlers in order", async () => {
    const executionOrder: string[] = [];

    const h1 = createMockHandler("h1", executionOrder);
    const h2 = createMockHandler("h2", executionOrder);
    const h3 = createMockHandler("h3", executionOrder);

    h1.setNext(h2).setNext(h3);

    await h1.handle(mockReq, mockRes);

    expect(executionOrder).toEqual(["h1", "h2", "h3"]);
  });

  it("should stop chain when handler returns false", async () => {
    const executionOrder: string[] = [];

    const h1 = createMockHandler("h1", executionOrder, true);
    const h2 = createMockHandler("h2", executionOrder, false); // Stops here
    const h3 = createMockHandler("h3", executionOrder, true);

    h1.setNext(h2).setNext(h3);

    await h1.handle(mockReq, mockRes);

    expect(executionOrder).toEqual(["h1", "h2"]); // h3 not executed
  });
});
```

---

## 🎯 Real-World SaaS Use Cases

### 1. **API Request Pipeline** ⭐⭐⭐

Request → Logging → CORS → RateLimit → Auth → Validation → Controller

Every API request goes through this pipeline.

### 2. **Multi-Level Approval Workflow** ⭐⭐⭐

Purchase Request → Manager ($0-$1K) → Director ($1K-$10K) → CFO ($10K+)

Each level approves within their authority or escalates.

### 3. **Content Moderation Pipeline** ⭐⭐

User Post → SpamFilter → ProfanityFilter → AIModeration → ManualReview

Each filter can flag/remove content or pass to next.

### 4. **Payment Processing** ⭐⭐⭐

Payment → FraudCheck → BalanceCheck → ProcessPayment → SendReceipt

Any step can fail and stop the chain.

### 5. **Email Filtering** ⭐⭐

Incoming Email → SpamCheck → VirusCheck → PriorityClassifier → InboxRouter

### 6. **Form Validation** ⭐⭐⭐

Form Data → RequiredFields → FormatValidation → BusinessRules → DatabaseConstraints

### 7. **Access Control** ⭐⭐⭐

Resource Access → Authentication → Authorization → RateLimiting → AccessGranted

---

## 📚 Resources for Deeper Understanding

### Books

1. **"Design Patterns: Elements of Reusable Object-Oriented Software"** - Gang of Four

   - Chapter: Behavioral Patterns → Chain of Responsibility

2. **"Head First Design Patterns"** - Freeman & Freeman

   - Chapter: Chain of Responsibility Pattern

3. **"Clean Code"** - Robert C. Martin
   - Discusses pipeline and filter patterns

### Online Resources

1. **Refactoring Guru - Chain of Responsibility**

   - https://refactoring.guru/design-patterns/chain-of-responsibility

2. **Express.js Middleware Guide**

   - https://expressjs.com/en/guide/using-middleware.html
   - Real-world implementation

3. **Spring Interceptors**

   - https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/interceptors.html

4. **Go HTTP Middleware Patterns**
   - https://github.com/justinas/alice

### Articles

1. **"Chain of Responsibility Pattern in Java"** - Baeldung

   - https://www.baeldung.com/chain-of-responsibility-pattern

2. **"Middleware Pattern in Node.js"**
   - Building Express-like middleware systems

---

## 🏋️ Practical Exercise

### Exercise: Build an API Gateway with Request Processing Pipeline

**Scenario**: Create an API gateway that processes incoming requests through a chain of handlers before forwarding to backend services.

**Requirements**:

1. **Handlers to Implement**:

   - **LoggingHandler**: Log all requests/responses with timing
   - **CorsHandler**: Handle CORS headers and preflight
   - **RateLimitHandler**: Rate limit by IP/API key (100 req/min)
   - **AuthenticationHandler**: Validate JWT tokens
   - **AuthorizationHandler**: Check user permissions
   - **ValidationHandler**: Validate request format
   - **TransformHandler**: Transform request/response formats
   - **CacheHandler**: Cache responses for GET requests

2. **Features**:

   - Configurable handler order
   - Ability to enable/disable handlers
   - Per-endpoint handler configuration
   - Metrics collection (requests, timing, errors)
   - Circuit breaker for failing handlers
   - Handler priority/grouping

3. **Advanced**:
   - Dynamic handler addition at runtime
   - Handler hot-swapping without restart
   - Conditional handler execution (based on request attributes)
   - Handler performance monitoring
   - A/B testing different handler configurations
   - Distributed tracing through handler chain

**Your Task**:

Implement in your language of choice:

- Handler interface/base class
- At least 6 concrete handlers
- Chain builder/configurator
- HTTP server that uses the chain
- Metrics dashboard showing handler performance
- Configuration file for handler setup

**Bonus Challenges**:

- Implement async handlers with proper waiting
- Add handler timeout (kill slow handlers)
- Create handler groups (security group = auth + rate limit + validation)
- Implement request replay for debugging
- Add handler versioning (v1 vs v2 handlers)
- Build visual handler chain diagram

**Acceptance Criteria**:

- ✅ All handlers execute in correct order
- ✅ Chain stops properly when handler returns false
- ✅ Handlers are independently testable
- ✅ No memory leaks with long-running chains
- ✅ Performance: <5ms overhead per handler
- ✅ Handles 10,000+ requests/second
- ✅ Proper error handling in each handler

---

## 📝 Summary

**Chain of Responsibility Pattern**:

- ✅ Decouples sender from receivers
- ✅ Flexible request handling with multiple handlers
- ✅ Easy to add/remove handlers dynamically
- ✅ Perfect for pipelines and middleware
- ✅ Promotes single responsibility principle
- ❌ No guarantee request will be handled
- ❌ Hard to debug complex chains
- ❌ Performance overhead from multiple handlers

**Key Concepts**:

1. **Handler**: Processes request or passes to next
2. **Chain**: Linked sequence of handlers
3. **Request**: Object passed through chain
4. **Stopping**: Handler can stop chain propagation
5. **Order**: Handler sequence matters

**When to Use**:

- Request processing pipelines (middleware)
- Multi-level validation
- Approval workflows
- Event filtering/routing
- Authentication/authorization chains
- Logging systems
- Error handling hierarchies

**When NOT to Use**:

- Simple linear logic (if/else is clearer)
- Only one handler needed
- All handlers must execute (use Observer)
- Order doesn't matter

**Remember**: Chain of Responsibility is perfect for building flexible request processing pipelines where multiple handlers can process or filter requests. It's the foundation of middleware systems in modern web frameworks!

---

**Great job completing the Chain of Responsibility Pattern!** 🎉

You've now learned all the major behavioral patterns:

- ✅ **Strategy Pattern** - Interchangeable algorithms
- ✅ **Observer Pattern** - Event-driven notifications
- ✅ **Command Pattern** - Encapsulated requests
- ✅ **Chain of Responsibility** - Request pipelines

These patterns are essential for building robust, scalable SaaS applications. You now have the tools to:

- Build flexible pricing systems (Strategy)
- Implement event-driven architectures (Observer)
- Create background job queues (Command)
- Design middleware pipelines (Chain of Responsibility)

**Next steps**: Apply these patterns to real projects and see how they work together to create powerful, maintainable systems! 🚀# Module 2.5: Core Design Patterns for SaaS Applications

---
