## Part 4: Behavioral Patterns - Observer Pattern

---

## 🎯 Learning Objectives

By the end of this section, you will:

- Understand what the Observer pattern is and when to use it
- Recognize scenarios where Observer shines in SaaS applications
- Implement Observer pattern in Java, Node.js, and Go
- Build event-driven architectures with loose coupling
- Handle asynchronous notifications and real-time updates
- Avoid common pitfalls like memory leaks and notification storms

---

## 📖 Conceptual Overview

### What is the Observer Pattern?

The **Observer pattern** defines a one-to-many dependency between objects so that when one object (the **Subject**) changes state, all its dependents (**Observers**) are notified and updated automatically.

**Key Idea**: Instead of objects checking for changes (polling), they subscribe to be notified when changes occur (push model).

### Real-World Analogy

Think of a YouTube channel:

- **Subject**: YouTube channel (PewDiePie, MrBeast, etc.)
- **Observers**: Subscribers
- **Event**: New video uploaded
- **Notification**: All subscribers get notified

You don't constantly check if there's a new video; YouTube notifies you when it happens.

---

## 🤔 When to Use Observer Pattern in SaaS

### Perfect Use Cases:

1. **Event Systems** ⭐⭐⭐

   - User registration → Send welcome email, create analytics profile, notify admin
   - Order placed → Update inventory, charge payment, send confirmation
   - Subscription cancelled → Stop billing, archive data, send feedback request

2. **Real-Time Notifications** ⭐⭐⭐

   - New message in chat
   - Task assigned to you
   - Comment on your post
   - System alerts

3. **Audit Logging** ⭐⭐⭐

   - Track all data changes
   - Record user actions
   - Compliance logging
   - Security monitoring

4. **Cache Invalidation** ⭐⭐

   - Data changed → Invalidate related caches
   - User updated → Clear user cache across services

5. **Webhooks** ⭐⭐⭐

   - External systems subscribe to your events
   - Payment processed → Notify Slack, update CRM, trigger fulfillment

6. **Analytics & Metrics** ⭐⭐

   - User action → Track in analytics
   - API call → Update usage metrics
   - Error occurred → Increment error counter

7. **UI Updates** ⭐⭐⭐

   - Model changed → Update all views
   - Data synced → Refresh dashboard
   - WebSocket message → Update UI

8. **Multi-Tenant Events** ⭐⭐
   - Tenant action → Notify only that tenant's subscribers
   - Cross-tenant events for integrations

---

## 🚫 When NOT to Use Observer

- **Simple direct calls** - If only one object needs to know, just call it directly
- **Synchronous workflows** - If order matters and you need guarantees, use callbacks
- **Too many observers** - Performance degrades with hundreds of observers
- **Complex dependencies** - If observers depend on each other, use mediator pattern

---

## 🏗️ Structure

Subject (Observable)
├── attach(observer)
├── detach(observer)
├── notify()
└── observers: List<Observer>
↓
Observer (Interface)
├── update(event)
↓
ConcreteObserver1, ConcreteObserver2, ConcreteObserver3...

**Key Elements:**

1. **Subject/Observable**: Object being watched, maintains list of observers
2. **Observer**: Interface for objects that should be notified
3. **ConcreteObserver**: Implements update logic
4. **Event**: Data passed to observers (optional)

---

## 💻 Implementation in Java/Spring Boot

### Scenario 1: User Registration Event System

When a user registers, multiple things need to happen: send welcome email, create analytics profile, notify admins, etc.

#### Step 1: Define Event

```java
// UserRegisteredEvent.java - The event
package com.saas.event;

import com.saas.model.User;
import lombok.Getter;
import lombok.RequiredArgsConstructor;
import java.time.LocalDateTime;

@Getter
@RequiredArgsConstructor
public class UserRegisteredEvent {
    private final User user;
    private final String registrationSource; // web, mobile, api
    private final LocalDateTime timestamp = LocalDateTime.now();
}
```

#### Step 2: Define Observer Interface (Spring ApplicationListener)

```java
// Using Spring's built-in event system
// No need to define observer interface, Spring provides it
```

#### Step 3: Create Concrete Observers (Event Listeners)

```java
// WelcomeEmailListener.java - Send welcome email
package com.saas.listener;

import com.saas.event.UserRegisteredEvent;
import com.saas.service.EmailService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class WelcomeEmailListener {

    private final EmailService emailService;

    @Async // Execute asynchronously
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        log.info("Sending welcome email to user: {}", event.getUser().getEmail());

        try {
            emailService.sendWelcomeEmail(
                event.getUser().getEmail(),
                event.getUser().getName()
            );
            log.info("Welcome email sent successfully");
        } catch (Exception e) {
            log.error("Failed to send welcome email", e);
            // Don't throw - don't want to break other listeners
        }
    }
}
```

```java
// AnalyticsListener.java - Track in analytics
package com.saas.listener;

import com.saas.event.UserRegisteredEvent;
import com.saas.service.AnalyticsService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class AnalyticsListener {

    private final AnalyticsService analyticsService;

    @Async
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        log.info("Creating analytics profile for user: {}", event.getUser().getId());

        try {
            analyticsService.trackUserRegistration(
                event.getUser().getId(),
                event.getRegistrationSource(),
                event.getTimestamp()
            );
            log.info("Analytics profile created");
        } catch (Exception e) {
            log.error("Failed to create analytics profile", e);
        }
    }
}
```

```java
// AdminNotificationListener.java - Notify admins
package com.saas.listener;

import com.saas.event.UserRegisteredEvent;
import com.saas.service.NotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class AdminNotificationListener {

    private final NotificationService notificationService;

    @Async
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        log.info("Notifying admins about new user registration");

        try {
            notificationService.notifyAdmins(
                "New User Registration",
                String.format("User %s registered via %s",
                    event.getUser().getEmail(),
                    event.getRegistrationSource())
            );
            log.info("Admin notification sent");
        } catch (Exception e) {
            log.error("Failed to notify admins", e);
        }
    }
}
```

```java
// AuditLogListener.java - Audit logging
package com.saas.listener;

import com.saas.event.UserRegisteredEvent;
import com.saas.repository.AuditLogRepository;
import com.saas.model.AuditLog;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class AuditLogListener {

    private final AuditLogRepository auditLogRepository;

    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        log.info("Recording audit log for user registration");

        AuditLog auditLog = AuditLog.builder()
            .eventType("USER_REGISTERED")
            .userId(event.getUser().getId())
            .details(String.format("User registered via %s",
                event.getRegistrationSource()))
            .timestamp(event.getTimestamp())
            .build();

        auditLogRepository.save(auditLog);
        log.info("Audit log recorded");
    }
}
```

#### Step 4: Publisher (Subject)

```java
// UserService.java - Publishes events
package com.saas.service;

import com.saas.event.UserRegisteredEvent;
import com.saas.model.User;
import com.saas.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {

    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public User registerUser(String email, String password, String source) {
        log.info("Registering new user: {}", email);

        // Create user
        User user = User.builder()
            .email(email)
            .password(hashPassword(password))
            .createdAt(LocalDateTime.now())
            .build();

        // Save to database
        user = userRepository.save(user);

        // Publish event - all listeners will be notified
        log.info("Publishing UserRegisteredEvent for user: {}", user.getId());
        eventPublisher.publishEvent(new UserRegisteredEvent(user, source));

        return user;
    }

    private String hashPassword(String password) {
        // Hash password with BCrypt
        return BCrypt.hashpw(password, BCrypt.gensalt());
    }
}
```

#### Step 5: Configuration for Async Execution

```java
// AsyncConfig.java
package com.saas.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.context.annotation.Bean;
import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "eventExecutor")
    public Executor eventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("event-");
        executor.initialize();
        return executor;
    }
}
```

### Scenario 2: Custom Observable for Real-Time Data

Sometimes you need more control than Spring's event system provides.

```java
// Subject.java - Observable interface
package com.saas.observer;

public interface Subject<T> {
    void attach(Observer<T> observer);
    void detach(Observer<T> observer);
    void notifyObservers(T data);
}
```

```java
// Observer.java - Observer interface
package com.saas.observer;

public interface Observer<T> {
    void update(T data);
}
```

```java
// StockPriceMonitor.java - Concrete Subject
package com.saas.observer;

import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

@Slf4j
public class StockPriceMonitor implements Subject<StockPrice> {

    // Thread-safe list for concurrent modifications
    private final List<Observer<StockPrice>> observers = new CopyOnWriteArrayList<>();
    private StockPrice currentPrice;

    @Override
    public void attach(Observer<StockPrice> observer) {
        log.info("Attaching observer: {}", observer.getClass().getSimpleName());
        observers.add(observer);
    }

    @Override
    public void detach(Observer<StockPrice> observer) {
        log.info("Detaching observer: {}", observer.getClass().getSimpleName());
        observers.remove(observer);
    }

    @Override
    public void notifyObservers(StockPrice data) {
        log.info("Notifying {} observers about price change", observers.size());
        observers.forEach(observer -> {
            try {
                observer.update(data);
            } catch (Exception e) {
                log.error("Observer update failed", e);
                // Continue with other observers
            }
        });
    }

    public void setPrice(String symbol, BigDecimal price) {
        this.currentPrice = new StockPrice(symbol, price);
        log.info("Price updated: {} = {}", symbol, price);
        notifyObservers(currentPrice);
    }
}
```

```java
// StockPrice.java - Event data
package com.saas.observer;

import lombok.AllArgsConstructor;
import lombok.Getter;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Getter
@AllArgsConstructor
public class StockPrice {
    private final String symbol;
    private final BigDecimal price;
    private final LocalDateTime timestamp = LocalDateTime.now();
}
```

```java
// PriceAlertObserver.java - Concrete Observer
package com.saas.observer;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;

@Slf4j
@RequiredArgsConstructor
public class PriceAlertObserver implements Observer<StockPrice> {

    private final String userEmail;
    private final BigDecimal threshold;

    @Override
    public void update(StockPrice stockPrice) {
        if (stockPrice.getPrice().compareTo(threshold) >= 0) {
            log.info("ALERT: {} reached threshold ${} for user {}",
                stockPrice.getSymbol(), threshold, userEmail);
            sendAlert(stockPrice);
        }
    }

    private void sendAlert(StockPrice stockPrice) {
        // Send email/SMS alert
        log.info("Sending alert to {}: {} is now ${}",
            userEmail, stockPrice.getSymbol(), stockPrice.getPrice());
    }
}
```

```java
// PriceLoggerObserver.java - Another concrete observer
package com.saas.observer;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class PriceLoggerObserver implements Observer<StockPrice> {

    @Override
    public void update(StockPrice stockPrice) {
        log.info("Price Log: {} = ${} at {}",
            stockPrice.getSymbol(),
            stockPrice.getPrice(),
            stockPrice.getTimestamp());
    }
}
```

```java
// PriceChartObserver.java - Updates UI chart
package com.saas.observer;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class PriceChartObserver implements Observer<StockPrice> {

    @Override
    public void update(StockPrice stockPrice) {
        log.info("Updating chart for: {}", stockPrice.getSymbol());
        // Update real-time chart via WebSocket
        updateChart(stockPrice);
    }

    private void updateChart(StockPrice stockPrice) {
        // Send WebSocket message to update chart
        log.info("Chart updated with new price: ${}", stockPrice.getPrice());
    }
}
```

```java
// Usage example
package com.saas;

import com.saas.observer.*;
import java.math.BigDecimal;

public class StockMonitoringExample {

    public static void main(String[] args) {
        // Create subject
        StockPriceMonitor monitor = new StockPriceMonitor();

        // Create and attach observers
        Observer<StockPrice> alertObserver =
            new PriceAlertObserver("user@example.com", new BigDecimal("150.00"));
        Observer<StockPrice> loggerObserver = new PriceLoggerObserver();
        Observer<StockPrice> chartObserver = new PriceChartObserver();

        monitor.attach(alertObserver);
        monitor.attach(loggerObserver);
        monitor.attach(chartObserver);

        // Update price - all observers notified
        monitor.setPrice("AAPL", new BigDecimal("145.50"));
        monitor.setPrice("AAPL", new BigDecimal("152.00")); // Triggers alert

        // Detach an observer
        monitor.detach(alertObserver);

        monitor.setPrice("AAPL", new BigDecimal("155.00")); // No alert
    }
}
```

### Scenario 3: Domain Events with Event Store

For audit trails and event sourcing.

```java
// DomainEvent.java - Base event
package com.saas.domain.event;

import lombok.Getter;
import java.time.LocalDateTime;
import java.util.UUID;

@Getter
public abstract class DomainEvent {
    private final String eventId = UUID.randomUUID().toString();
    private final LocalDateTime occurredAt = LocalDateTime.now();
    private final String eventType = this.getClass().getSimpleName();
}
```

```java
// SubscriptionCancelledEvent.java
package com.saas.domain.event;

import lombok.Getter;
import lombok.RequiredArgsConstructor;

@Getter
@RequiredArgsConstructor
public class SubscriptionCancelledEvent extends DomainEvent {
    private final Long subscriptionId;
    private final Long userId;
    private final String reason;
}
```

```java
// EventStore.java - Stores all events
package com.saas.domain.event;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

@Component
@Slf4j
public class EventStore {

    private final List<DomainEvent> events = new ArrayList<>();
    private final List<EventHandler> handlers = new CopyOnWriteArrayList<>();

    public void registerHandler(EventHandler handler) {
        handlers.add(handler);
        log.info("Registered event handler: {}", handler.getClass().getSimpleName());
    }

    public void publish(DomainEvent event) {
        // Store event
        events.add(event);
        log.info("Event stored: {} ({})", event.getEventType(), event.getEventId());

        // Notify all handlers
        handlers.forEach(handler -> {
            if (handler.canHandle(event)) {
                try {
                    handler.handle(event);
                } catch (Exception e) {
                    log.error("Handler failed for event: {}", event.getEventId(), e);
                }
            }
        });
    }

    public List<DomainEvent> getEvents() {
        return new ArrayList<>(events);
    }
}
```

```java
// EventHandler.java - Handler interface
package com.saas.domain.event;

public interface EventHandler {
    boolean canHandle(DomainEvent event);
    void handle(DomainEvent event);
}
```

```java
// SubscriptionCancellationHandler.java
package com.saas.domain.event.handler;

import com.saas.domain.event.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class SubscriptionCancellationHandler implements EventHandler {

    private final BillingService billingService;
    private final EmailService emailService;

    @Override
    public boolean canHandle(DomainEvent event) {
        return event instanceof SubscriptionCancelledEvent;
    }

    @Override
    public void handle(DomainEvent event) {
        SubscriptionCancelledEvent cancelEvent = (SubscriptionCancelledEvent) event;

        log.info("Handling subscription cancellation: {}",
            cancelEvent.getSubscriptionId());

        // Stop billing
        billingService.stopBilling(cancelEvent.getSubscriptionId());

        // Send cancellation email
        emailService.sendCancellationEmail(
            cancelEvent.getUserId(),
            cancelEvent.getReason()
        );

        // Request feedback
        emailService.sendFeedbackRequest(cancelEvent.getUserId());

        log.info("Subscription cancellation handled successfully");
    }
}
```

---

## 💻 Implementation in Node.js

### Scenario 1: EventEmitter for In-Process Events

Node.js has built-in EventEmitter - perfect for Observer pattern!

```typescript
// eventTypes.ts - Event type definitions
export const Events = {
  USER_REGISTERED: "user:registered",
  ORDER_PLACED: "order:placed",
  PAYMENT_PROCESSED: "payment:processed",
  SUBSCRIPTION_CANCELLED: "subscription:cancelled",
} as const;

export interface UserRegisteredEvent {
  userId: string;
  email: string;
  source: string;
  timestamp: Date;
}

export interface OrderPlacedEvent {
  orderId: string;
  userId: string;
  total: number;
  items: Array<{ id: string; quantity: number }>;
  timestamp: Date;
}
```

```typescript
// eventBus.ts - Global event bus (Subject)
import { EventEmitter } from "events";

class EventBus extends EventEmitter {
  constructor() {
    super();
    this.setMaxListeners(50); // Increase if needed
  }

  // Type-safe emit
  emitEvent<T>(eventName: string, data: T): void {
    console.log(`📢 Event emitted: ${eventName}`);
    this.emit(eventName, data);
  }

  // Type-safe subscribe
  onEvent<T>(
    eventName: string,
    handler: (data: T) => void | Promise<void>
  ): void {
    console.log(`👂 Listener registered for: ${eventName}`);
    this.on(eventName, handler);
  }

  // Unsubscribe
  offEvent<T>(
    eventName: string,
    handler: (data: T) => void | Promise<void>
  ): void {
    console.log(`👋 Listener removed for: ${eventName}`);
    this.off(eventName, handler);
  }
}

// Singleton instance
export const eventBus = new EventBus();
```

```typescript
// listeners/welcomeEmailListener.ts - Concrete Observer
import { eventBus } from "../eventBus";
import { Events, UserRegisteredEvent } from "../eventTypes";
import { EmailService } from "../services/emailService";

export class WelcomeEmailListener {
  constructor(private emailService: EmailService) {
    this.subscribe();
  }

  private subscribe(): void {
    eventBus.onEvent<UserRegisteredEvent>(
      Events.USER_REGISTERED,
      this.handleUserRegistered.bind(this)
    );
  }

  private async handleUserRegistered(
    event: UserRegisteredEvent
  ): Promise<void> {
    console.log(`📧 Sending welcome email to: ${event.email}`);

    try {
      await this.emailService.sendWelcomeEmail(event.email, event.userId);
      console.log("✅ Welcome email sent successfully");
    } catch (error) {
      console.error("❌ Failed to send welcome email:", error);
      // Don't throw - don't want to break other listeners
    }
  }
}
```

```typescript
// listeners/analyticsListener.ts
import { eventBus } from "../eventBus";
import { Events, UserRegisteredEvent } from "../eventTypes";
import { AnalyticsService } from "../services/analyticsService";

export class AnalyticsListener {
  constructor(private analyticsService: AnalyticsService) {
    this.subscribe();
  }

  private subscribe(): void {
    eventBus.onEvent<UserRegisteredEvent>(
      Events.USER_REGISTERED,
      this.handleUserRegistered.bind(this)
    );
  }

  private async handleUserRegistered(
    event: UserRegisteredEvent
  ): Promise<void> {
    console.log(`📊 Tracking user registration: ${event.userId}`);

    try {
      await this.analyticsService.trackEvent("user_registered", {
        userId: event.userId,
        source: event.source,
        timestamp: event.timestamp,
      });
      console.log("✅ Analytics tracked");
    } catch (error) {
      console.error("❌ Failed to track analytics:", error);
    }
  }
}
```

```typescript
// listeners/auditLogListener.ts
import { eventBus } from "../eventBus";
import { Events, UserRegisteredEvent } from "../eventTypes";
import { AuditLogRepository } from "../repositories/auditLogRepository";

export class AuditLogListener {
  constructor(private auditLogRepo: AuditLogRepository) {
    this.subscribe();
  }

  private subscribe(): void {
    eventBus.onEvent<UserRegisteredEvent>(
      Events.USER_REGISTERED,
      this.handleUserRegistered.bind(this)
    );
  }

  private async handleUserRegistered(
    event: UserRegisteredEvent
  ): Promise<void> {
    console.log(`📝 Recording audit log for user: ${event.userId}`);

    await this.auditLogRepo.create({
      eventType: "USER_REGISTERED",
      userId: event.userId,
      details: `User registered via ${event.source}`,
      timestamp: event.timestamp,
    });

    console.log("✅ Audit log recorded");
  }
}
```

```typescript
// services/userService.ts - Publisher (Subject)
import { eventBus } from "../eventBus";
import { Events, UserRegisteredEvent } from "../eventTypes";
import { UserRepository } from "../repositories/userRepository";

export class UserService {
  constructor(private userRepo: UserRepository) {}

  async registerUser(
    email: string,
    password: string,
    source: string
  ): Promise<User> {
    console.log(`👤 Registering new user: ${email}`);

    // Create user
    const user = await this.userRepo.create({
      email,
      password: await this.hashPassword(password),
      createdAt: new Date(),
    });

    // Publish event - all listeners will be notified
    console.log(`📢 Publishing USER_REGISTERED event for: ${user.id}`);
    eventBus.emitEvent<UserRegisteredEvent>(Events.USER_REGISTERED, {
      userId: user.id,
      email: user.email,
      source,
      timestamp: new Date(),
    });

    return user;
  }

  private async hashPassword(password: string): Promise<string> {
    const bcrypt = await import("bcrypt");
    return bcrypt.hash(password, 10);
  }
}
```

```typescript
// app.ts - Initialize listeners
import { eventBus } from "./eventBus";
import { WelcomeEmailListener } from "./listeners/welcomeEmailListener";
import { AnalyticsListener } from "./listeners/analyticsListener";
import { AuditLogListener } from "./listeners/auditLogListener";
import { EmailService } from "./services/emailService";
import { AnalyticsService } from "./services/analyticsService";
import { AuditLogRepository } from "./repositories/auditLogRepository";

// Initialize services
const emailService = new EmailService();
const analyticsService = new AnalyticsService();
const auditLogRepo = new AuditLogRepository();

// Register all listeners
new WelcomeEmailListener(emailService);
new AnalyticsListener(analyticsService);
new AuditLogListener(auditLogRepo);

console.log("✅ All event listeners registered");
```

### Scenario 2: Custom Observable for Real-Time Updates

```typescript
// observable.ts - Custom Observable implementation
export interface Observer<T> {
  update(data: T): void;
}

export class Observable<T> {
  private observers: Set<Observer<T>> = new Set();

  attach(observer: Observer<T>): void {
    console.log(`➕ Observer attached (total: ${this.observers.size + 1})`);
    this.observers.add(observer);
  }

  detach(observer: Observer<T>): void {
    console.log(`➖ Observer detached (total: ${this.observers.size - 1})`);
    this.observers.delete(observer);
  }

  notify(data: T): void {
    console.log(`📢 Notifying ${this.observers.size} observers`);
    this.observers.forEach((observer) => {
      try {
        observer.update(data);
      } catch (error) {
        console.error("Observer update failed:", error);
        // Continue with other observers
      }
    });
  }

  getObserverCount(): number {
    return this.observers.size;
  }
}
```

```typescript
// stockPriceMonitor.ts - Concrete Subject
import { Observable } from "./observable";

export interface StockPrice {
  symbol: string;
  price: number;
  timestamp: Date;
}

export class StockPriceMonitor extends Observable<StockPrice> {
  private currentPrices: Map<string, number> = new Map();

  updatePrice(symbol: string, price: number): void {
    this.currentPrices.set(symbol, price);

    console.log(`💰 Price updated: ${symbol} = $${price}`);

    this.notify({
      symbol,
      price,
      timestamp: new Date(),
    });
  }

  getPrice(symbol: string): number | undefined {
    return this.currentPrices.get(symbol);
  }
}
```

````typescript
// priceAlertObserver.ts - Concrete Observer
import { Observer } from './observable';
import { StockPrice } from './stockPriceMonitor';

export class PriceAlertObserver implements Observer<StockPrice> {
  constructor(
    private userEmail: string,
    private threshold: number
  ) {}

  update(stockPrice: StockPrice): void {
if (stockPrice.price >= this.threshold) {
console.log(
🚨 ALERT: ${stockPrice.symbol} reached threshold ${this.threshold}  +
for user ${this.userEmail}
);
this.sendAlert(stockPrice);
}
}
private sendAlert(stockPrice: StockPrice): void {
// Send email/SMS alert
console.log(
📧 Sending alert to ${this.userEmail}:  +
${stockPrice.symbol} is now ${stockPrice.price}
);
}
}
```typescript
// priceLoggerObserver.ts
import { Observer } from './observable';
import { StockPrice } from './stockPriceMonitor';

export class PriceLoggerObserver implements Observer<StockPrice> {
  update(stockPrice: StockPrice): void {
    console.log(
      `📝 Price Log: ${stockPrice.symbol} = ${stockPrice.price} ` +
      `at ${stockPrice.timestamp.toISOString()}`
    );
  }
}
````

```typescript
// priceChartObserver.ts
import { Observer } from "./observable";
import { StockPrice } from "./stockPriceMonitor";
import { WebSocket } from "ws";

export class PriceChartObserver implements Observer<StockPrice> {
  constructor(private wsServer: WebSocket.Server) {}

  update(stockPrice: StockPrice): void {
    console.log(`📊 Updating chart for: ${stockPrice.symbol}`);

    // Broadcast to all WebSocket clients
    this.wsServer.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(
          JSON.stringify({
            type: "price_update",
            data: stockPrice,
          })
        );
      }
    });
  }
}
```

```typescript
// usage.ts - Example usage
import { StockPriceMonitor } from "./stockPriceMonitor";
import { PriceAlertObserver } from "./priceAlertObserver";
import { PriceLoggerObserver } from "./priceLoggerObserver";
import { PriceChartObserver } from "./priceChartObserver";
import WebSocket from "ws";

// Create WebSocket server for real-time updates
const wss = new WebSocket.Server({ port: 8080 });

// Create subject
const monitor = new StockPriceMonitor();

// Create and attach observers
const alertObserver = new PriceAlertObserver("user@example.com", 150);
const loggerObserver = new PriceLoggerObserver();
const chartObserver = new PriceChartObserver(wss);

monitor.attach(alertObserver);
monitor.attach(loggerObserver);
monitor.attach(chartObserver);

// Simulate price updates
setInterval(() => {
  const price = 140 + Math.random() * 20; // Random price between 140-160
  monitor.updatePrice("AAPL", Number(price.toFixed(2)));
}, 2000);

console.log("✅ Stock price monitor started");
console.log(`👀 Observers: ${monitor.getObserverCount()}`);
```

### Scenario 3: Webhook System (External Observers)

```typescript
// webhook.ts - Webhook models
export interface Webhook {
  id: string;
  url: string;
  events: string[];
  secret: string;
  active: boolean;
}

export interface WebhookPayload {
  event: string;
  timestamp: Date;
  data: any;
  signature?: string;
}
```

```typescript
// webhookManager.ts - Manages webhook observers
import axios from "axios";
import crypto from "crypto";
import { Webhook, WebhookPayload } from "./webhook";
import { eventBus } from "./eventBus";

export class WebhookManager {
  private webhooks: Map<string, Webhook> = new Map();

  constructor() {
    this.subscribeToEvents();
  }

  private subscribeToEvents(): void {
    // Subscribe to all events and forward to webhooks
    eventBus.on("*", async (event: string, data: any) => {
      await this.notifyWebhooks(event, data);
    });
  }

  registerWebhook(webhook: Webhook): void {
    console.log(`🔗 Registering webhook: ${webhook.url}`);
    this.webhooks.set(webhook.id, webhook);
  }

  unregisterWebhook(webhookId: string): void {
    console.log(`🔓 Unregistering webhook: ${webhookId}`);
    this.webhooks.delete(webhookId);
  }

  private async notifyWebhooks(event: string, data: any): Promise<void> {
    const payload: WebhookPayload = {
      event,
      timestamp: new Date(),
      data,
    };

    // Find all webhooks interested in this event
    const interestedWebhooks = Array.from(this.webhooks.values()).filter(
      (wh) => wh.active && wh.events.includes(event)
    );

    console.log(
      `🔔 Notifying ${interestedWebhooks.length} webhooks about: ${event}`
    );

    // Send to all interested webhooks
    await Promise.allSettled(
      interestedWebhooks.map((webhook) => this.sendWebhook(webhook, payload))
    );
  }

  private async sendWebhook(
    webhook: Webhook,
    payload: WebhookPayload
  ): Promise<void> {
    try {
      // Sign payload
      const signature = this.generateSignature(payload, webhook.secret);
      payload.signature = signature;

      // Send webhook
      const response = await axios.post(webhook.url, payload, {
        headers: {
          "Content-Type": "application/json",
          "X-Webhook-Signature": signature,
        },
        timeout: 5000, // 5 second timeout
      });

      console.log(`✅ Webhook delivered to ${webhook.url}: ${response.status}`);
    } catch (error) {
      console.error(
        `❌ Webhook delivery failed to ${webhook.url}:`,
        error.message
      );

      // Could implement retry logic here
      this.handleWebhookFailure(webhook, payload, error);
    }
  }

  private generateSignature(payload: WebhookPayload, secret: string): string {
    const data = JSON.stringify(payload);
    return crypto.createHmac("sha256", secret).update(data).digest("hex");
  }

  private handleWebhookFailure(
    webhook: Webhook,
    payload: WebhookPayload,
    error: any
  ): void {
    // Store failed webhook for retry
    console.log(`📦 Queueing webhook for retry: ${webhook.id}`);

    // Could use Bull queue or similar for retry logic
  }
}
```

```typescript
// webhookController.ts - REST API for managing webhooks
import { Request, Response } from "express";
import { WebhookManager } from "./webhookManager";
import { v4 as uuidv4 } from "uuid";

export class WebhookController {
  constructor(private webhookManager: WebhookManager) {}

  async createWebhook(req: Request, res: Response): Promise<void> {
    const { url, events } = req.body;

    if (!url || !events || !Array.isArray(events)) {
      res.status(400).json({ error: "Invalid webhook configuration" });
      return;
    }

    const webhook = {
      id: uuidv4(),
      url,
      events,
      secret: this.generateSecret(),
      active: true,
    };

    this.webhookManager.registerWebhook(webhook);

    res.status(201).json({
      message: "Webhook registered successfully",
      webhook: {
        id: webhook.id,
        url: webhook.url,
        events: webhook.events,
        secret: webhook.secret, // Send once, store securely
      },
    });
  }

  async deleteWebhook(req: Request, res: Response): Promise<void> {
    const { id } = req.params;

    this.webhookManager.unregisterWebhook(id);

    res.json({ message: "Webhook unregistered successfully" });
  }

  private generateSecret(): string {
    return crypto.randomBytes(32).toString("hex");
  }
}
```

---

## 💻 Implementation in Go

### Scenario 1: Event System with Channels

```go
// event.go - Event types
package event

import "time"

// Event base interface
type Event interface {
    GetEventType() string
    GetTimestamp() time.Time
}

// BaseEvent common fields
type BaseEvent struct {
    EventType string
    Timestamp time.Time
}

func (e BaseEvent) GetEventType() string {
    return e.EventType
}

func (e BaseEvent) GetTimestamp() time.Time {
    return e.Timestamp
}

// UserRegisteredEvent
type UserRegisteredEvent struct {
    BaseEvent
    UserID string
    Email  string
    Source string
}

func NewUserRegisteredEvent(userID, email, source string) *UserRegisteredEvent {
    return &UserRegisteredEvent{
        BaseEvent: BaseEvent{
            EventType: "USER_REGISTERED",
            Timestamp: time.Now(),
        },
        UserID: userID,
        Email:  email,
        Source: source,
    }
}
```

```go
// observer.go - Observer interface
package event

// Observer interface
type Observer interface {
    OnEvent(event Event)
}

// ObserverFunc adapter for functions
type ObserverFunc func(Event)

func (f ObserverFunc) OnEvent(event Event) {
    f(event)
}
```

```go
// event_bus.go - Event bus (Subject)
package event

import (
    "fmt"
    "sync"
)

// EventBus manages observers and dispatches events
type EventBus struct {
    observers map[string][]Observer
    mu        sync.RWMutex
}

func NewEventBus() *EventBus {
    return &EventBus{
        observers: make(map[string][]Observer),
    }
}

// Subscribe adds an observer for specific event type
func (eb *EventBus) Subscribe(eventType string, observer Observer) {
    eb.mu.Lock()
    defer eb.mu.Unlock()

    eb.observers[eventType] = append(eb.observers[eventType], observer)
    fmt.Printf("👂 Observer subscribed to: %s (total: %d)\n",
        eventType, len(eb.observers[eventType]))
}

// Unsubscribe removes an observer
func (eb *EventBus) Unsubscribe(eventType string, observer Observer) {
    eb.mu.Lock()
    defer eb.mu.Unlock()

    observers := eb.observers[eventType]
    for i, obs := range observers {
        // Note: This comparison works for function observers but not for struct observers
        // For production, use a registration ID instead
        if fmt.Sprintf("%p", obs) == fmt.Sprintf("%p", observer) {
            eb.observers[eventType] = append(observers[:i], observers[i+1:]...)
            fmt.Printf("👋 Observer unsubscribed from: %s\n", eventType)
            return
        }
    }
}

// Publish sends event to all subscribed observers
func (eb *EventBus) Publish(event Event) {
    eb.mu.RLock()
    observers := eb.observers[event.GetEventType()]
    eb.mu.RUnlock()

    fmt.Printf("📢 Publishing event: %s to %d observers\n",
        event.GetEventType(), len(observers))

    // Notify all observers concurrently
    var wg sync.WaitGroup
    for _, observer := range observers {
        wg.Add(1)
        go func(obs Observer) {
            defer wg.Done()
            defer func() {
                if r := recover(); r != nil {
                    fmt.Printf("❌ Observer panic recovered: %v\n", r)
                }
            }()

            obs.OnEvent(event)
        }(observer)
    }

    wg.Wait()
}

// PublishAsync publishes event asynchronously (fire and forget)
func (eb *EventBus) PublishAsync(event Event) {
    go eb.Publish(event)
}
```

```go
// welcome_email_listener.go - Concrete Observer
package listener

import (
    "fmt"
    "myapp/event"
    "myapp/service"
)

type WelcomeEmailListener struct {
    emailService *service.EmailService
}

func NewWelcomeEmailListener(emailService *service.EmailService) *WelcomeEmailListener {
    return &WelcomeEmailListener{
        emailService: emailService,
    }
}

func (l *WelcomeEmailListener) OnEvent(e event.Event) {
    userEvent, ok := e.(*event.UserRegisteredEvent)
    if !ok {
        return
    }

    fmt.Printf("📧 Sending welcome email to: %s\n", userEvent.Email)

    err := l.emailService.SendWelcomeEmail(userEvent.Email, userEvent.UserID)
    if err != nil {
        fmt.Printf("❌ Failed to send welcome email: %v\n", err)
        return
    }

    fmt.Println("✅ Welcome email sent successfully")
}
```

```go
// analytics_listener.go
package listener

import (
    "fmt"
    "myapp/event"
    "myapp/service"
)

type AnalyticsListener struct {
    analyticsService *service.AnalyticsService
}

func NewAnalyticsListener(analyticsService *service.AnalyticsService) *AnalyticsListener {
    return &AnalyticsListener{
        analyticsService: analyticsService,
    }
}

func (l *AnalyticsListener) OnEvent(e event.Event) {
    userEvent, ok := e.(*event.UserRegisteredEvent)
    if !ok {
        return
    }

    fmt.Printf("📊 Tracking user registration: %s\n", userEvent.UserID)

    err := l.analyticsService.TrackEvent("user_registered", map[string]interface{}{
        "user_id":   userEvent.UserID,
        "source":    userEvent.Source,
        "timestamp": userEvent.Timestamp,
    })

    if err != nil {
        fmt.Printf("❌ Failed to track analytics: %v\n", err)
        return
    }

    fmt.Println("✅ Analytics tracked")
}
```

```go
// audit_log_listener.go
package listener

import (
    "fmt"
    "myapp/event"
    "myapp/repository"
)

type AuditLogListener struct {
    auditLogRepo *repository.AuditLogRepository
}

func NewAuditLogListener(auditLogRepo *repository.AuditLogRepository) *AuditLogListener {
    return &AuditLogListener{
        auditLogRepo: auditLogRepo,
    }
}

func (l *AuditLogListener) OnEvent(e event.Event) {
    userEvent, ok := e.(*event.UserRegisteredEvent)
    if !ok {
        return
    }

    fmt.Printf("📝 Recording audit log for user: %s\n", userEvent.UserID)

    err := l.auditLogRepo.Create(&repository.AuditLog{
        EventType: "USER_REGISTERED",
        UserID:    userEvent.UserID,
        Details:   fmt.Sprintf("User registered via %s", userEvent.Source),
        Timestamp: userEvent.Timestamp,
    })

    if err != nil {
        fmt.Printf("❌ Failed to record audit log: %v\n", err)
        return
    }

    fmt.Println("✅ Audit log recorded")
}
```

```go
// user_service.go - Publisher (Subject)
package service

import (
    "fmt"
    "myapp/event"
    "myapp/repository"
    "golang.org/x/crypto/bcrypt"
)

type UserService struct {
    userRepo *repository.UserRepository
    eventBus *event.EventBus
}

func NewUserService(
    userRepo *repository.UserRepository,
    eventBus *event.EventBus,
) *UserService {
    return &UserService{
        userRepo: userRepo,
        eventBus: eventBus,
    }
}

func (s *UserService) RegisterUser(email, password, source string) (*repository.User, error) {
    fmt.Printf("👤 Registering new user: %s\n", email)

    // Hash password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return nil, err
    }

    // Create user
    user := &repository.User{
        Email:    email,
        Password: string(hashedPassword),
    }

    err = s.userRepo.Create(user)
    if err != nil {
        return nil, err
    }

    // Publish event - all observers will be notified
    fmt.Printf("📢 Publishing USER_REGISTERED event for: %s\n", user.ID)
    userEvent := event.NewUserRegisteredEvent(user.ID, user.Email, source)
    s.eventBus.PublishAsync(userEvent) // Async to not block

    return user, nil
}
```

```go
// main.go - Wire everything together
package main

import (
    "myapp/event"
    "myapp/listener"
    "myapp/repository"
    "myapp/service"
)

func main() {
    // Create event bus
    eventBus := event.NewEventBus()

    // Create repositories
    userRepo := repository.NewUserRepository()
    auditLogRepo := repository.NewAuditLogRepository()

    // Create services
    emailService := service.NewEmailService()
    analyticsService := service.NewAnalyticsService()

    // Create listeners (observers)
    welcomeEmailListener := listener.NewWelcomeEmailListener(emailService)
    analyticsListener := listener.NewAnalyticsListener(analyticsService)
    auditLogListener := listener.NewAuditLogListener(auditLogRepo)

    // Subscribe listeners to events
    eventBus.Subscribe("USER_REGISTERED", welcomeEmailListener)
    eventBus.Subscribe("USER_REGISTERED", analyticsListener)
    eventBus.Subscribe("USER_REGISTERED", auditLogListener)

    // Create user service
    userService := service.NewUserService(userRepo, eventBus)

    // Register a user - all observers will be notified
    user, err := userService.RegisterUser(
        "user@example.com",
        "password123",
        "web",
    )

    if err != nil {
        panic(err)
    }

    fmt.Printf("✅ User registered: %s\n", user.ID)
}
```

### Scenario 2: Channel-Based Observable

Go's channels provide a natural way to implement observer pattern.

```go
// observable.go - Generic observable using channels
package observable

import (
    "fmt"
    "sync"
)

// Observer receives events on a channel
type Observer[T any] struct {
    ID      string
    Channel chan T
}

// Observable manages observers and broadcasts events
type Observable[T any] struct {
    observers map[string]*Observer[T]
    mu        sync.RWMutex
}

func NewObservable[T any]() *Observable[T] {
    return &Observable[T]{
        observers: make(map[string]*Observer[T]),
    }
}

// Subscribe creates a new observer and returns its channel
func (o *Observable[T]) Subscribe(id string, bufferSize int) <-chan T {
    o.mu.Lock()
    defer o.mu.Unlock()

    observer := &Observer[T]{
        ID:      id,
        Channel: make(chan T, bufferSize),
    }

    o.observers[id] = observer
    fmt.Printf("➕ Observer subscribed: %s (total: %d)\n", id, len(o.observers))

    return observer.Channel
}

// Unsubscribe removes an observer and closes its channel
func (o *Observable[T]) Unsubscribe(id string) {
    o.mu.Lock()
    defer o.mu.Unlock()

    if observer, exists := o.observers[id]; exists {
        close(observer.Channel)
        delete(o.observers, id)
        fmt.Printf("➖ Observer unsubscribed: %s (total: %d)\n", id, len(o.observers))
    }
}

// Notify sends event to all observers
func (o *Observable[T]) Notify(data T) {
    o.mu.RLock()
    defer o.mu.RUnlock()

    fmt.Printf("📢 Notifying %d observers\n", len(o.observers))

    for _, observer := range o.observers {
        // Non-blocking send
        select {
        case observer.Channel <- data:
            // Sent successfully
        default:
            fmt.Printf("⚠️  Channel full for observer: %s\n", observer.ID)
        }
    }
}

// NotifyAsync sends event to all observers asynchronously
func (o *Observable[T]) NotifyAsync(data T) {
    go o.Notify(data)
}

// Close closes all observer channels
func (o *Observable[T]) Close() {
    o.mu.Lock()
    defer o.mu.Unlock()

    for id, observer := range o.observers {
        close(observer.Channel)
        delete(o.observers, id)
    }

    fmt.Println("🔒 All observers closed")
}

// ObserverCount returns the number of active observers
func (o *Observable[T]) ObserverCount() int {
    o.mu.RLock()
    defer o.mu.RUnlock()
    return len(o.observers)
}
```

```go
// stock_price_monitor.go - Using channel-based observable
package main

import (
    "fmt"
    "myapp/observable"
    "time"
)

type StockPrice struct {
    Symbol    string
    Price     float64
    Timestamp time.Time
}

type StockPriceMonitor struct {
    observable *observable.Observable[StockPrice]
    prices     map[string]float64
}

func NewStockPriceMonitor() *StockPriceMonitor {
    return &StockPriceMonitor{
        observable: observable.NewObservable[StockPrice](),
        prices:     make(map[string]float64),
    }
}

func (m *StockPriceMonitor) Subscribe(id string) <-chan StockPrice {
    return m.observable.Subscribe(id, 10) // Buffer of 10
}

func (m *StockPriceMonitor) Unsubscribe(id string) {
    m.observable.Unsubscribe(id)
}

func (m *StockPriceMonitor) UpdatePrice(symbol string, price float64) {
    m.prices[symbol] = price

    fmt.Printf("💰 Price updated: %s = $%.2f\n", symbol, price)

    m.observable.NotifyAsync(StockPrice{
        Symbol:    symbol,
        Price:     price,
        Timestamp: time.Now(),
    })
}

func (m *StockPriceMonitor) Close() {
    m.observable.Close()
}
```

```go
// usage.go - Example usage
package main

import (
    "fmt"
    "time"
)

func main() {
    monitor := NewStockPriceMonitor()
    defer monitor.Close()

    // Subscribe observer 1: Price logger
    priceChannel1 := monitor.Subscribe("logger")
    go func() {
        for price := range priceChannel1 {
            fmt.Printf("📝 Logger: %s = $%.2f at %s\n",
                price.Symbol, price.Price, price.Timestamp.Format("15:04:05"))
        }
    }()

    // Subscribe observer 2: Price alert
    priceChannel2 := monitor.Subscribe("alert")
    go func() {
        threshold := 150.0
        for price := range priceChannel2 {
            if price.Price >= threshold {
                fmt.Printf("🚨 ALERT: %s reached $%.2f (threshold: $%.2f)\n",
                    price.Symbol, price.Price, threshold)
            }
        }
    }()

    // Subscribe observer 3: Average calculator
    priceChannel3 := monitor.Subscribe("average")
    go func() {
        var sum float64
        var count int

        for price := range priceChannel3 {
            sum += price.Price
            count++
            avg := sum / float64(count)
            fmt.Printf("📊 Average price: $%.2f (n=%d)\n", avg, count)
        }
    }()

    // Simulate price updates
    prices := []float64{145.50, 148.20, 152.00, 149.75, 155.00}
    for _, price := range prices {
        monitor.UpdatePrice("AAPL", price)
        time.Sleep(1 * time.Second)
    }

    // Unsubscribe one observer
    monitor.Unsubscribe("alert")

    // More price updates
    monitor.UpdatePrice("AAPL", 158.00)

    time.Sleep(1 * time.Second)
}
```

### Scenario 3: Event Store Pattern

```go
// domain_event.go - Domain events
package domain

import (
    "time"
    "github.com/google/uuid"
)

type DomainEvent interface {
    GetEventID() string
    GetEventType() string
    GetOccurredAt() time.Time
}

type BaseDomainEvent struct {
    EventID    string
    EventType  string
    OccurredAt time.Time
}

func NewBaseDomainEvent(eventType string) BaseDomainEvent {
    return BaseDomainEvent{
        EventID:    uuid.New().String(),
        EventType:  eventType,
        OccurredAt: time.Now(),
    }
}

func (e BaseDomainEvent) GetEventID() string {
    return e.EventID
}

func (e BaseDomainEvent) GetEventType() string {
    return e.EventType
}

func (e BaseDomainEvent) GetOccurredAt() time.Time {
    return e.OccurredAt
}

// SubscriptionCancelledEvent
type SubscriptionCancelledEvent struct {
    BaseDomainEvent
    SubscriptionID string
    UserID         string
    Reason         string
}

func NewSubscriptionCancelledEvent(subID, userID, reason string) *SubscriptionCancelledEvent {
    return &SubscriptionCancelledEvent{
        BaseDomainEvent: NewBaseDomainEvent("SUBSCRIPTION_CANCELLED"),
        SubscriptionID:  subID,
        UserID:          userID,
        Reason:          reason,
    }
}
```

```go
// event_store.go - Event store with handlers
package domain

import (
    "fmt"
    "sync"
)

type EventHandler interface {
    CanHandle(event DomainEvent) bool
    Handle(event DomainEvent) error
}

type EventStore struct {
    events   []DomainEvent
    handlers []EventHandler
    mu       sync.RWMutex
}

func NewEventStore() *EventStore {
    return &EventStore{
        events:   make([]DomainEvent, 0),
        handlers: make([]EventHandler, 0),
    }
}

func (es *EventStore) RegisterHandler(handler EventHandler) {
    es.mu.Lock()
    defer es.mu.Unlock()

    es.handlers = append(es.handlers, handler)
    fmt.Printf("✅ Handler registered: %T\n", handler)
}

func (es *EventStore) Publish(event DomainEvent) error {
    // Store event
    es.mu.Lock()
    es.events = append(es.events, event)
    handlers := make([]EventHandler, len(es.handlers))
    copy(handlers, es.handlers)
    es.mu.Unlock()

    fmt.Printf("📝 Event stored: %s (%s)\n",
        event.GetEventType(), event.GetEventID())

    // Notify all interested handlers
    var wg sync.WaitGroup
    for _, handler := range handlers {
        if handler.CanHandle(event) {
            wg.Add(1)
            go func(h EventHandler) {
                defer wg.Done()

                if err := h.Handle(event); err != nil {
                    fmt.Printf("❌ Handler failed: %v\n", err)
                }
            }(handler)
        }
    }

    wg.Wait()
    return nil
}

func (es *EventStore) GetEvents() []DomainEvent {
    es.mu.RLock()
    defer es.mu.RUnlock()

    events := make([]DomainEvent, len(es.events))
    copy(events, es.events)
    return events
}
```

```go
// subscription_cancellation_handler.go
package handler

import (
    "fmt"
    "myapp/domain"
    "myapp/service"
)

type SubscriptionCancellationHandler struct {
    billingService *service.BillingService
    emailService   *service.EmailService
}

func NewSubscriptionCancellationHandler(
    billingService *service.BillingService,
    emailService *service.EmailService,
) *SubscriptionCancellationHandler {
    return &SubscriptionCancellationHandler{
        billingService: billingService,
        emailService:   emailService,
    }
}

func (h *SubscriptionCancellationHandler) CanHandle(event domain.DomainEvent) bool {
    _, ok := event.(*domain.SubscriptionCancelledEvent)
    return ok
}

func (h *SubscriptionCancellationHandler) Handle(event domain.DomainEvent) error {
    cancelEvent := event.(*domain.SubscriptionCancelledEvent)

    fmt.Printf("⚙️  Handling subscription cancellation: %s\n",
        cancelEvent.SubscriptionID)

    // Stop billing
    if err := h.billingService.StopBilling(cancelEvent.SubscriptionID); err != nil {
        return fmt.Errorf("failed to stop billing: %w", err)
    }

    // Send cancellation email
    if err := h.emailService.SendCancellationEmail(
        cancelEvent.UserID,
        cancelEvent.Reason,
    ); err != nil {
        return fmt.Errorf("failed to send cancellation email: %w", err)
    }

    // Request feedback
    if err := h.emailService.SendFeedbackRequest(cancelEvent.UserID); err != nil {
        // Don't fail if feedback request fails
        fmt.Printf("⚠️  Failed to send feedback request: %v\n", err)
    }

    fmt.Println("✅ Subscription cancellation handled successfully")
    return nil
}
```

```go
// usage.go - Event store usage
package main

import (
    "myapp/domain"
    "myapp/handler"
    "myapp/service"
)

func main() {
    // Create event store
    eventStore := domain.NewEventStore()

    // Create services
    billingService := service.NewBillingService()
    emailService := service.NewEmailService()

    // Create and register handlers
    cancellationHandler := handler.NewSubscriptionCancellationHandler(
        billingService,
        emailService,
    )
    eventStore.RegisterHandler(cancellationHandler)

    // Publish event
    event := domain.NewSubscriptionCancelledEvent(
        "sub_123",
        "user_456",
        "Too expensive",
    )

    if err := eventStore.Publish(event); err != nil {
        panic(err)
    }

    // Get all events (for audit/replay)
    allEvents := eventStore.GetEvents()
    fmt.Printf("📚 Total events stored: %d\n", len(allEvents))
}
```

---

## ✅ Best Practices

### 1. **Use Weak References or Proper Cleanup**

Prevent memory leaks by ensuring observers are properly removed.

```java
// ✅ GOOD - Explicit cleanup
public class MyComponent implements AutoCloseable {
    private final EventBus eventBus;
    private final Observer observer;

    public MyComponent(EventBus eventBus) {
        this.eventBus = eventBus;
        this.observer = this::handleEvent;
        eventBus.subscribe("MY_EVENT", observer);
    }

    @Override
    public void close() {
        eventBus.unsubscribe("MY_EVENT", observer);
    }
}

// ❌ BAD - No cleanup (memory leak!)
public class MyComponent {
    public MyComponent(EventBus eventBus) {
        eventBus.subscribe("MY_EVENT", this::handleEvent);
        // Never unsubscribed!
    }
}
```

### 2. **Handle Observer Exceptions Gracefully**

One failing observer shouldn't break others.

```typescript
// ✅ GOOD - Isolated error handling
notify(data: T): void {
  this.observers.forEach(observer => {
    try {
      observer.update(data);
    } catch (error) {
      console.error('Observer failed:', error);
      // Continue with other observers
    }
  });
}

// ❌ BAD - One failure breaks all
notify(data: T): void {
  this.observers.forEach(observer => {
    observer.update(data); // If this throws, others don't run
  });
}
```

### 3. **Make Observers Idempotent**

Events might be delivered multiple times due to retries or network issues.

```go
// ✅ GOOD - Idempotent observer
func (l *EmailListener) OnEvent(event Event) {
    userEvent := event.(*UserRegisteredEvent)

    // Check if already processed
    if l.cache.Has(event.GetEventID()) {
        log.Info("Event already processed, skipping")
        return
    }

    l.emailService.SendWelcomeEmail(userEvent.Email)

    // Mark as processed
    l.cache.Set(event.GetEventID(), true)
}

// ❌ BAD - Not idempotent
func (l *EmailListener) OnEvent(event Event) {
    // Sends duplicate emails if event delivered twice
    l.emailService.SendWelcomeEmail(userEvent.Email)
}
```

### 4. **Use Async Notifications for Non-Critical Observers**

Don't block the main flow for non-essential notifications.

```java
// ✅ GOOD - Async notifications
@Async
@EventListener
public void handleUserRegistered(UserRegisteredEvent event) {
    // Runs in separate thread pool
    emailService.sendWelcomeEmail(event.getUser());
}

// ❌ BAD - Synchronous (blocks user registration)
@EventListener
public void handleUserRegistered(UserRegisteredEvent event) {
    emailService.sendWelcomeEmail(event.getUser()); // Blocks!
}
```

### 5. **Limit Observer Count**

Too many observers hurt performance.

```typescript
// ✅ GOOD - Limit observers
class Observable<T> {
  private maxObservers = 100;

  attach(observer: Observer<T>): void {
    if (this.observers.size >= this.maxObservers) {
      throw new Error("Maximum observers reached");
    }
    this.observers.add(observer);
  }
}
```

### 6. **Use Event Filtering**

Let observers filter events they care about.

```go
// ✅ GOOD - Observers self-filter
type FilteredObserver struct {
    eventTypes map[string]bool
}

func (o *FilteredObserver) OnEvent(event Event) {
    if !o.eventTypes[event.GetEventType()] {
        return // Not interested
    }

    // Process event
}
```

### 7. **Document Event Contracts**

Clearly define what events mean and what data they contain.

```typescript
/**
 * UserRegisteredEvent
 *
 * Fired when: New user completes registration
 *
 * Data:
 * - userId: Unique user identifier
 * - email: User's email address
 * - source: Registration source (web, mobile, api)
 * - timestamp: When registration occurred
 *
 * Observers:
 * - WelcomeEmailListener: Sends welcome email
 * - AnalyticsListener: Tracks in analytics
 * - AuditLogListener: Records audit trail
 *
 * Guarantees:
 * - Fired exactly once per registration
 * - Fired after database commit
 * - Fired asynchronously (non-blocking)
 */
export interface UserRegisteredEvent {
  userId: string;
  email: string;
  source: string;
  timestamp: Date;
}
```

### 8. **Use Dead Letter Queue for Failed Events**

Capture events that fail to process.

```java
// ✅ GOOD - DLQ for failed events
@EventListener
public void handleEvent(MyEvent event) {
    try {
        processEvent(event);
    } catch (Exception e) {
        log.error("Event processing failed", e);
        deadLetterQueue.add(event); // Retry later
    }
}
```

### 9. **Thread-Safe Observer Management**

Use thread-safe collections when observers can be added/removed concurrently.

```java
// ✅ GOOD - Thread-safe
private final List<Observer> observers = new CopyOnWriteArrayList<>();

// ❌ BAD - Not thread-safe
private final List<Observer> observers = new ArrayList<>();
```

### 10. **Provide Observer Priorities**

Sometimes order matters.

```typescript
// ✅ GOOD - Priority support
interface PrioritizedObserver<T> extends Observer<T> {
  priority: number; // Higher = earlier
}

class Observable<T> {
  private observers: PrioritizedObserver<T>[] = [];

  attach(observer: PrioritizedObserver<T>): void {
    this.observers.push(observer);
    this.observers.sort((a, b) => b.priority - a.priority);
  }

  notify(data: T): void {
    // Notifies in priority order
    this.observers.forEach((obs) => obs.update(data));
  }
}
```

---

## 🚫 Common Pitfalls

### 1. **Memory Leaks from Forgotten Observers**

**Problem**: Observers never unsubscribed keep references alive.

```typescript
// ❌ PITFALL
class Component {
  constructor(eventBus: EventBus) {
    eventBus.on("event", this.handler); // Never removed!
  }

  handler(data: any) {
    /* ... */
  }
}

// ✅ SOLUTION
class Component {
  constructor(private eventBus: EventBus) {
    this.eventBus.on("event", this.handler);
  }

  destroy() {
    this.eventBus.off("event", this.handler);
  }
}
```

### 2. **Notification Storms**

**Problem**: One change triggers cascade of updates.

```java
// ❌ PITFALL - Infinite loop
public void onUserUpdate(User user) {
    updateCache(user); // Triggers event
}

public void onCacheUpdate(User user) {
    updateDatabase(user); // Triggers event
}

public void onDatabaseUpdate(User user) {
    updateUser(user); // Triggers event - LOOP!
}

// ✅ SOLUTION - Break the cycle
public void onUserUpdate(User user) {
    updateCache(user, false); // Don't trigger event
}
```

### 3. **Order Dependency Between Observers**

**Problem**: Observers depend on execution order.

```go
// ❌ PITFALL
// Observer A must run before Observer B
// But observers are notified concurrently!

// ✅ SOLUTION - Use explicit ordering or separate events
func (s *Service) Process() {
    // Step 1: Critical synchronous operations
    s.saveToDatabase()

    // Step 2: Async notifications
    s.eventBus.Publish(event)
}
```

### 4. **Blocking the Publisher**

**Problem**: Slow observer blocks the subject.

```typescript
// ❌ PITFALL
notify(data: T): void {
  this.observers.forEach(observer => {
    observer.update(data); // If slow, blocks everything
  });
}

// ✅ SOLUTION - Async notification
async notify(data: T): Promise<void> {
  await Promise.all(
    this.observers.map(observer =>
      Promise.resolve(observer.update(data))
    )
  );
}
```

### 5. **Lost Events During High Load**

**Problem**: Events dropped when observers can't keep up.

```go
// ❌ PITFALL - Unbuffered channel
ch := make(chan Event) // Blocks if receiver slow

// ✅ SOLUTION - Buffered channel + overflow handling
ch := make(chan Event, 1000) // Buffer 1000 events

select {
case ch <- event:
    // Sent successfully
default:
    log.Error("Channel full, event dropped")
    metrics.IncrementDroppedEvents()
}
```

### 6. **Observer State Inconsistency**

**Problem**: Observers see inconsistent state.

```java
// ❌ PITFALL
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);
    eventPublisher.publish(new OrderCreatedEvent(order));
    // If transaction rolls back, event already published!
}

// ✅ SOLUTION - Publish after transaction
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleOrderCreated(OrderCreatedEvent event) {
    // Only fires if transaction commits
}
```

### 7. **No Event Versioning**

**Problem**: Event schema changes break observers.

```typescript
// ❌ PITFALL
interface UserEvent {
  userId: string;
  // Later: added email field - breaks old observers!
}

// ✅ SOLUTION - Version events
interface UserEventV1 {
  version: 1;
  userId: string;
}

interface UserEventV2 {
  version: 2;
  userId: string;
  email: string; // New field
}

// Observers handle both versions
onEvent(event: UserEventV1 | UserEventV2) {
  if (event.version === 2) {
    // Use new fields
  } else {
    // Handle old version
  }
}
```

### 8. **Synchronous Processing in Async Systems**

**Problem**: Mixing sync/async incorrectly.

```go
// ❌ PITFALL
func (s *Service) PublishAsync(event Event) {
    go s.Publish(event) // Fire and forget
}

// Caller assumes event processed immediately
s.PublishAsync(event)
return "Success" // Premature!

// ✅ SOLUTION - Be explicit about async
func (s *Service) PublishAsync(event Event) <-chan error {
    errChan := make(chan error, 1)
    go func() {
        errChan <- s.Publish(event)
    }()
    return errChan
}

// Caller can wait or proceed
errChan := s.PublishAsync(event)
// Do other work...
if err := <-errChan; err != nil {
    // Handle error
}
```

### 9. **No Monitoring or Visibility**

**Problem**: Can't debug observer issues.

```java
// ❌ PITFALL - No visibility
public void notify(Event event) {
    observers.forEach(o -> o.update(event));
}

// ✅ SOLUTION - Add metrics and logging
public void notify(Event event) {
    log.info("Notifying {} observers for event: {}",
        observers.size(), event.getType());

    long startTime = System.currentTimeMillis();

    observers.forEach(observer -> {
        try {
            long obsStart = System.currentTimeMillis();
            observer.update(event);
            long duration = System.currentTimeMillis() - obsStart;

            metrics.recordObserverDuration(
                observer.getClass().getSimpleName(),
                duration
            );
        } catch (Exception e) {
            log.error("Observer failed", e);
            metrics.incrementObserverErrors();
        }
    });

    long totalDuration = System.currentTimeMillis() - startTime;
    log.info("Total notification time: {}ms", totalDuration);
}
```

### 10. **Wrong Pattern Choice**

**Problem**: Using Observer when another pattern fits better.

```typescript
// ❌ PITFALL - Using Observer for request/response
eventBus.emit("getUser", userId);
eventBus.on("userResult", (user) => {
  // Process user - hard to correlate request/response!
});

// ✅ SOLUTION - Use callbacks or promises
async function getUser(userId: string): Promise<User> {
  return await userService.findById(userId);
}
```

---

## 🎯 Real-World SaaS Use Cases

### 1. **User Lifecycle Events** ⭐⭐⭐

USER_REGISTERED → WelcomeEmail, Analytics, AuditLog, AdminNotification
USER_UPGRADED → BillingUpdate, FeatureUnlock, ThankYouEmail
USER_CHURNED → StopBilling, ExitSurvey, WinbackCampaign

### 2. **Payment Processing** ⭐⭐⭐

PAYMENT_RECEIVED → UpdateSubscription, SendReceipt, UpdateAnalytics
PAYMENT_FAILED → SendAlert, RetryPayment, NotifyBilling
REFUND_ISSUED → UpdateBalance, NotifyUser, RecordAudit

### 3. **Real-Time Collaboration** ⭐⭐⭐

DOCUMENT_CHANGED → NotifyCollaborators, UpdateVersion, SaveBackup
COMMENT_ADDED → NotifyMentioned, UpdateThread, SendEmail
USER_JOINED → BroadcastPresence, UpdateParticipants

### 4. **Audit Logging & Compliance** ⭐⭐⭐

ANY_EVENT → AuditLog, ComplianceCheck, SecurityMonitor
DATA_ACCESSED → RecordAccess, CheckPermissions, AlertIfSuspicious
DATA_MODIFIED → VersionControl, ChangeLog, NotifyOwner

### 5. **Multi-Tenant Events** ⭐⭐

TENANT_CREATED → ProvisionResources, SetupBilling, SendWelcome
TENANT_UPGRADED → UnlockFeatures, MigrateData, UpdateQuotas
TENANT_DELETED → ArchiveData, StopBilling, CleanupResources

### 6. **Integration Webhooks** ⭐⭐⭐

ORDER_PLACED → Slack, Zapier, CustomWebhook, InternalSystems
LEAD_CAPTURED → CRM, EmailMarketing, Analytics, Slack
ERROR_OCCURRED → Sentry, PagerDuty, AdminAlert

### 7. **Cache Invalidation** ⭐⭐

USER_UPDATED → InvalidateUserCache, UpdateSession, RefreshUI
PRODUCT_CHANGED → InvalidateProductCache, UpdateSearchIndex
CONFIG_MODIFIED → ReloadConfig, RestartServices

---

## 📚 Resources for Deeper Understanding

### Books

1. **"Design Patterns: Elements of Reusable Object-Oriented Software"** - Gang of Four

   - Chapter: Behavioral Patterns → Observer

2. **"Head First Design Patterns"** - Freeman & Freeman

   - Chapter 2: The Observer Pattern (Weather Station example)

3. **"Reactive Programming with RxJS"** - Sergi Mansilla

   - Observer pattern in reactive systems

4. **"Enterprise Integration Patterns"** - Hohpe & Woolf
   - Event-Driven Architecture patterns

### Online Resources

1. **Refactoring Guru - Observer Pattern**

   - https://refactoring.guru/design-patterns/observer

2. **Spring Framework - Application Events**

   - https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events

3. **Node.js EventEmitter Documentation**

   - https://nodejs.org/api/events.html

4. **Go Concurrency Patterns**
   - https://go.dev/blog/pipelines

### Articles

1. **"Observer Pattern in Modern Applications"** - Baeldung

   - https://www.baeldung.com/java-observer-pattern

2. **"Event-Driven Architecture Fundamentals"** - Martin Fowler

   - https://martinfowler.com/articles/201701-event-driven.html

3. **"The Observer Pattern in JavaScript"** - Addy Osmani
   - Learning JavaScript Design Patterns book (free online)

---

## 🏋️ Practical Exercise

### Exercise: Build a Real-Time Task Management System

**Scenario**: You're building a collaborative task management SaaS. Multiple users work on tasks simultaneously, and everyone needs to see updates in real-time.

**Requirements**:

1. **Events to Handle**:

   - `TASK_CREATED`: New task added
   - `TASK_UPDATED`: Task modified (title, description, status)
   - `TASK_ASSIGNED`: Task assigned to user
   - `TASK_COMPLETED`: Task marked as done
   - `COMMENT_ADDED`: Comment added to task

2. **Observers**:

   - **WebSocket Observer**: Broadcast to connected clients
   - **Email Observer**: Send email notifications
   - **Analytics Observer**: Track task metrics
   - **Audit Log Observer**: Record all changes
   - **Search Index Observer**: Update search index
   - **Webhook Observer**: Call external webhooks

3. **Features**:
   - Real-time updates to all connected users
   - Email notifications based on user preferences
   - Audit trail of all task changes
   - Analytics dashboard showing task metrics
   - Integration with external systems via webhooks

**Your Task**:

Implement in your language of choice (Java/Node.js/Go):

- Event system with all 5 event types
- At least 4 observers
- Publisher that emits events when tasks change
- REST API to create/update tasks
- WebSocket endpoint for real-time updates
- Configuration for which notifications to enable

**Bonus Challenges**:

- Add event replay (replay last N events on reconnect)
- Implement event filtering (users only see relevant events)
- Add retry logic for failed webhook deliveries
- Implement event batching (group similar events)
- Add performance monitoring (observer execution time)
- Implement priority observers (some run before others)

**Acceptance Criteria**:

- ✅ All events properly published and handled
- ✅ WebSocket clients receive real-time updates
- ✅ No memory leaks (observers properly cleaned up)
- ✅ Observers don't block each other
- ✅ Failed observers don't break the system
- ✅ Observable performance: Handle 1000+ events/second
- ✅ Observer latency: <50ms for critical observers

---

## 📝 Summary

**Observer Pattern**:

- ✅ One-to-many dependency (one subject, many observers)
- ✅ Loose coupling between publisher and subscribers
- ✅ Dynamic subscription/unsubscription
- ✅ Perfect for event-driven architectures
- ✅ Essential for real-time systems
- ❌ Can cause memory leaks if not careful
- ❌ Difficult to debug with many observers
- ❌ No guarantee of notification order

**Key Concepts**:

1. **Subject/Observable**: Maintains list of observers, notifies them
2. **Observer**: Receives notifications, implements update logic
3. **Event**: Data passed to observers (optional)
4. **Subscription**: Observer registers with subject
5. **Notification**: Subject tells observers about changes

**When to Use**:

- Event-driven systems
- Real-time notifications
- Loose coupling between components
- Multiple objects need to react to changes
- Audit logging and monitoring
- Webhooks and integrations

**When NOT to Use**:

- Simple direct calls suffice
- Tight coupling is acceptable
- Synchronous workflows with guarantees
- Request/response patterns

**Remember**: Observer is the foundation of event-driven architecture in SaaS. It enables loose coupling, scalability, and extensibility. Use it for notifications, not for request/response!
