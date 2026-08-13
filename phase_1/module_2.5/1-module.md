# Module 2.5: Core Design Patterns for SaaS Applications

## Learning Objectives

By the end of this module, you will:

1. Understand when and why to use design patterns
2. Master creational patterns essential for SaaS (Factory, Builder, Singleton)
3. Implement structural patterns for clean architecture (Repository, Adapter, Decorator, Proxy)
4. Apply behavioral patterns for complex workflows (Strategy, Observer, Command, Chain of Responsibility)
5. Understand enterprise patterns (Unit of Work, Service Layer, DTO)
6. Implement SaaS-specific patterns (Tenant Context, Circuit Breaker, Retry)
7. Recognize and avoid anti-patterns
8. Implement patterns in Java, Node.js, and Go

---

## Part 1: Pattern Fundamentals (Day 1)

### What Are Design Patterns?

**Design Patterns** are reusable solutions to common problems in software design. They're like recipes - proven approaches that experienced developers have refined over time.

**Key Points:**

- Patterns are NOT code you copy-paste
- Patterns are templates for solving problems
- Patterns provide a common vocabulary for teams
- Patterns capture best practices

**The Gang of Four (GoF):** In 1994, four authors published "Design Patterns: Elements of Reusable Object-Oriented Software" - the foundation of pattern thinking.

### Why Three Categories?

The GoF organized their 23 patterns into three families, and this module follows the same structure across its parts — it's worth naming what each family actually answers, since that's what makes the grouping more than an arbitrary way to split a table of contents:

- **Creational** patterns (Part 2, this module) answer: _how_ do objects get created? They deal with instantiation logic — deciding which class to instantiate, how, and when.
- **Structural** patterns (Part 3, upcoming) answer: _how_ are objects composed into larger structures? Repository, Adapter, Decorator, Proxy — all about assembling pieces cleanly.
- **Behavioral** patterns (Part 4, upcoming) answer: _how_ do objects communicate and share responsibility? Strategy, Observer, Command — all about the flow of control and messages between objects.

Keep this in mind as the module moves forward: every pattern you meet fits one of these three questions, and knowing which question you're answering helps you recognize when a pattern is the right shape for the problem in front of you.

### SOLID Principles (Quick Review)

Before diving into patterns, let's review SOLID - principles that guide good design:

#### S - Single Responsibility Principle

**"A class should have one, and only one, reason to change"**

```javascript
// ❌ BAD: UserService does too much
class UserService {
  createUser(data) {
    /* creates user */
  }
  sendWelcomeEmail(user) {
    /* sends email */
  }
  validateUser(user) {
    /* validates data */
  }
  generateReport(userId) {
    /* generates report */
  }
}

// ✅ GOOD: Separate responsibilities
class UserService {
  createUser(data) {
    /* creates user */
  }
}

class EmailService {
  sendWelcomeEmail(user) {
    /* sends email */
  }
}

class UserValidator {
  validate(user) {
    /* validates data */
  }
}

class ReportService {
  generateUserReport(userId) {
    /* generates report */
  }
}
```

Same idea in Java and Go:

```java
// ❌ BAD: UserService does too much
public class UserService {
    public void createUser(UserData data) { /* creates user */ }
    public void sendWelcomeEmail(User user) { /* sends email */ }
    public void validateUser(User user) { /* validates data */ }
    public void generateReport(String userId) { /* generates report */ }
}

// ✅ GOOD: Separate responsibilities
public class UserService {
    public void createUser(UserData data) { /* creates user */ }
}

public class EmailService {
    public void sendWelcomeEmail(User user) { /* sends email */ }
}

public class UserValidator {
    public void validate(User user) { /* validates data */ }
}

public class ReportService {
    public void generateUserReport(String userId) { /* generates report */ }
}
```

```go
// ❌ BAD: UserService does too much
type UserService struct{}

func (s *UserService) CreateUser(data UserData) { /* creates user */ }
func (s *UserService) SendWelcomeEmail(user User) { /* sends email */ }
func (s *UserService) ValidateUser(user User) { /* validates data */ }
func (s *UserService) GenerateReport(userID string) { /* generates report */ }

// ✅ GOOD: Separate responsibilities
type UserService struct{}
func (s *UserService) CreateUser(data UserData) { /* creates user */ }

type EmailService struct{}
func (e *EmailService) SendWelcomeEmail(user User) { /* sends email */ }

type UserValidator struct{}
func (v *UserValidator) Validate(user User) { /* validates data */ }

type ReportService struct{}
func (r *ReportService) GenerateUserReport(userID string) { /* generates report */ }
```

#### O - Open/Closed Principle

**"Software entities should be open for extension, but closed for modification"**

```javascript
// ❌ BAD: Must modify class to add new payment methods
class PaymentProcessor {
  processPayment(amount, method) {
    if (method === 'stripe') {
      // Stripe logic
    } else if (method === 'paypal') {
      // PayPal logic
    }
    // Adding new method requires modifying this class
  }
}

// ✅ GOOD: Can extend without modifying
interface PaymentGateway {
  process(amount: number): Promise<void>;
}

class StripeGateway implements PaymentGateway {
  async process(amount: number) {
    // Stripe logic
  }
}

class PayPalGateway implements PaymentGateway {
  async process(amount: number) {
    // PayPal logic
  }
}

class PaymentProcessor {
  constructor(private gateway: PaymentGateway) {}

  async processPayment(amount: number) {
    await this.gateway.process(amount);
  }
}

// Add new gateway without modifying PaymentProcessor
class BitcoinGateway implements PaymentGateway {
  async process(amount: number) {
    // Bitcoin logic
  }
}
```

#### L - Liskov Substitution Principle

**"Derived classes must be substitutable for their base classes"**

```javascript
// ❌ BAD: Rectangle and Square violate LSP
class Rectangle {
  setWidth(width) { this.width = width; }
  setHeight(height) { this.height = height; }
  getArea() { return this.width * this.height; }
}

class Square extends Rectangle {
  setWidth(width) {
    this.width = width;
    this.height = width; // Forces width = height
  }
  setHeight(height) {
    this.width = height;
    this.height = height; // Forces width = height
  }
}

// Problem: Breaks expectations
const rect = new Square();
rect.setWidth(5);
rect.setHeight(4);
console.log(rect.getArea()); // Expected: 20, Got: 16

// ✅ GOOD: Separate abstractions
interface Shape {
  getArea(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  getArea() { return this.width * this.height; }
}

class Square implements Shape {
  constructor(private side: number) {}
  getArea() { return this.side * this.side; }
}
```

#### I - Interface Segregation Principle

**"Many client-specific interfaces are better than one general-purpose interface"**

```javascript
// ❌ BAD: Fat interface forces classes to implement methods they don't need
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
}

class Human implements Worker {
  work() { /* works */ }
  eat() { /* eats */ }
  sleep() { /* sleeps */ }
}

class Robot implements Worker {
  work() { /* works */ }
  eat() { /* doesn't need this */ }
  sleep() { /* doesn't need this */ }
}

// ✅ GOOD: Segregated interfaces
interface Workable {
  work(): void;
}

interface Eatable {
  eat(): void;
}

interface Sleepable {
  sleep(): void;
}

class Human implements Workable, Eatable, Sleepable {
  work() { /* works */ }
  eat() { /* eats */ }
  sleep() { /* sleeps */ }
}

class Robot implements Workable {
  work() { /* works */ }
}
```

#### D - Dependency Inversion Principle

**"Depend on abstractions, not concretions"**

```javascript
// ❌ BAD: High-level module depends on low-level module
class EmailSender {
  send(message) {
    // Gmail-specific code
  }
}

class UserService {
  constructor() {
    this.emailSender = new EmailSender(); // Tightly coupled
  }

  createUser(data) {
    // ...
    this.emailSender.send('Welcome!');
  }
}

// ✅ GOOD: Both depend on abstraction
interface IEmailSender {
  send(message: string): Promise<void>;
}

class GmailSender implements IEmailSender {
  async send(message: string) {
    // Gmail-specific code
  }
}

class SendGridSender implements IEmailSender {
  async send(message: string) {
    // SendGrid-specific code
  }
}

class UserService {
  constructor(private emailSender: IEmailSender) {}

  async createUser(data) {
    // ...
    await this.emailSender.send('Welcome!');
  }
}

// Can swap implementations easily
const userService = new UserService(new SendGridSender());
```

Same idea in Java and Go — this principle is the one that will come back explicitly when we look at Singleton vs dependency injection later in this module:

```java
// ❌ BAD: High-level module depends on low-level module
public class EmailSender {
    public void send(String message) {
        // Gmail-specific code
    }
}

public class UserService {
    private EmailSender emailSender = new EmailSender(); // Tightly coupled

    public void createUser(UserData data) {
        // ...
        emailSender.send("Welcome!");
    }
}

// ✅ GOOD: Both depend on abstraction
public interface IEmailSender {
    void send(String message);
}

public class GmailSender implements IEmailSender {
    public void send(String message) {
        // Gmail-specific code
    }
}

public class SendGridSender implements IEmailSender {
    public void send(String message) {
        // SendGrid-specific code
    }
}

public class UserService {
    private final IEmailSender emailSender;

    public UserService(IEmailSender emailSender) {
        this.emailSender = emailSender;
    }

    public void createUser(UserData data) {
        // ...
        emailSender.send("Welcome!");
    }
}

// Can swap implementations easily
UserService userService = new UserService(new SendGridSender());
```

```go
// ❌ BAD: High-level module depends on low-level module
type EmailSender struct{}

func (e *EmailSender) Send(message string) {
    // Gmail-specific code
}

type UserService struct {
    emailSender *EmailSender // Tightly coupled
}

func (s *UserService) CreateUser(data UserData) {
    // ...
    s.emailSender.Send("Welcome!")
}

// ✅ GOOD: Both depend on abstraction
type EmailSender interface {
    Send(message string) error
}

type GmailSender struct{}
func (g *GmailSender) Send(message string) error {
    // Gmail-specific code
    return nil
}

type SendGridSender struct{}
func (s *SendGridSender) Send(message string) error {
    // SendGrid-specific code
    return nil
}

type UserService struct {
    emailSender EmailSender
}

func NewUserService(emailSender EmailSender) *UserService {
    return &UserService{emailSender: emailSender}
}

func (s *UserService) CreateUser(data UserData) error {
    // ...
    return s.emailSender.Send("Welcome!")
}

// Can swap implementations easily
userService := NewUserService(&SendGridSender{})
```

### When to Use Patterns (and When Not To)

#### ✅ Use Patterns When:

1. **Problem is recurring** - You've seen this problem multiple times
2. **Solution is proven** - Pattern has been validated in production
3. **Team understands it** - Pattern improves communication
4. **Flexibility is needed** - Need to change implementation later
5. **Complexity is justified** - Benefits outweigh added abstraction

#### ❌ Don't Use Patterns When:

1. **Over-engineering** - Pattern adds complexity without benefit
2. **Problem is unique** - No recurring need
3. **YAGNI** (You Aren't Gonna Need It) - Premature abstraction
4. **Team confusion** - Pattern obscures rather than clarifies
5. **Performance critical** - Pattern adds unnecessary overhead

**Example: Over-engineering**

```javascript
// ❌ BAD: Over-engineered for simple use case
interface GreeterStrategy {
  greet(): string;
}

class EnglishGreeter implements GreeterStrategy {
  greet() { return 'Hello'; }
}

class FrenchGreeter implements GreeterStrategy {
  greet() { return 'Bonjour'; }
}

class GreeterFactory {
  static create(language: string): GreeterStrategy {
    if (language === 'en') return new EnglishGreeter();
    if (language === 'fr') return new FrenchGreeter();
    throw new Error('Unknown language');
  }
}

class GreeterService {
  constructor(private strategy: GreeterStrategy) {}
  executeGreeting() { return this.strategy.greet(); }
}

// Just to say hello!
const greeter = new GreeterService(GreeterFactory.create('en'));
console.log(greeter.executeGreeting());

// ✅ GOOD: Simple solution for simple problem
function greet(language: string): string {
  return language === 'en' ? 'Hello' : 'Bonjour';
}

console.log(greet('en'));
```

**Rule of Three:** Don't abstract until you've written the same code three times. Then consider a pattern.

---

## Part 2: Creational Patterns (Days 1-2)

Creational patterns deal with object creation mechanisms.

### Pattern 1: Singleton

**Purpose:** Ensure a class has only one instance and provide global access to it.

**When to Use:**

- Database connection pools
- Configuration managers
- Logger instances
- Cache managers
- Application-wide state

**When NOT to Use:**

- When you need multiple instances
- When testing (singletons are hard to mock)
- When it hides dependencies

#### Implementation: Node.js (TypeScript)

```typescript
// config/DatabaseConnection.ts
class DatabaseConnection {
  private static instance: DatabaseConnection;
  private connection: any;

  // Private constructor prevents direct instantiation
  private constructor() {
    this.connection = null;
  }

  // Get the singleton instance
  public static getInstance(): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }

  public async connect(config: any): Promise<void> {
    if (!this.connection) {
      this.connection = await createConnection(config);
      console.log("Database connected");
    }
  }

  public getConnection(): any {
    if (!this.connection) {
      throw new Error("Database not connected. Call connect() first.");
    }
    return this.connection;
  }

  public async disconnect(): Promise<void> {
    if (this.connection) {
      await this.connection.close();
      this.connection = null;
      console.log("Database disconnected");
    }
  }
}

// Usage
const db1 = DatabaseConnection.getInstance();
const db2 = DatabaseConnection.getInstance();

console.log(db1 === db2); // true - same instance

await db1.connect({ host: "localhost", port: 5432 });
const connection = db1.getConnection();
```

#### Implementation: Java (Spring Boot)

```java
// config/DatabaseConnection.java
public class DatabaseConnection {
    private static DatabaseConnection instance;
    private Connection connection;

    // Private constructor
    private DatabaseConnection() {
        this.connection = null;
    }

    // Thread-safe singleton with double-checked locking
    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {
                if (instance == null) {
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }

    public void connect(String url, String user, String password)
            throws SQLException {
        if (connection == null || connection.isClosed()) {
            connection = DriverManager.getConnection(url, user, password);
            System.out.println("Database connected");
        }
    }

    public Connection getConnection() throws SQLException {
        if (connection == null || connection.isClosed()) {
            throw new SQLException("Database not connected");
        }
        return connection;
    }

    public void disconnect() throws SQLException {
        if (connection != null && !connection.isClosed()) {
            connection.close();
            System.out.println("Database disconnected");
        }
    }
}

// Usage
DatabaseConnection db1 = DatabaseConnection.getInstance();
DatabaseConnection db2 = DatabaseConnection.getInstance();

System.out.println(db1 == db2); // true - same instance
```

#### Implementation: Go

```go
// database/connection.go
package database

import (
    "database/sql"
    "sync"
)

type DatabaseConnection struct {
    connection *sql.DB
}

var (
    instance *DatabaseConnection
    once     sync.Once
)

// GetInstance returns the singleton instance
func GetInstance() *DatabaseConnection {
    once.Do(func() {
        instance = &DatabaseConnection{}
    })
    return instance
}

func (db *DatabaseConnection) Connect(dsn string) error {
    if db.connection == nil {
        conn, err := sql.Open("postgres", dsn)
        if err != nil {
            return err
        }
        db.connection = conn
        println("Database connected")
    }
    return nil
}

func (db *DatabaseConnection) GetConnection() *sql.DB {
    if db.connection == nil {
        panic("Database not connected. Call Connect() first.")
    }
    return db.connection
}

func (db *DatabaseConnection) Disconnect() error {
    if db.connection != nil {
        err := db.connection.Close()
        db.connection = nil
        println("Database disconnected")
        return err
    }
    return nil
}

// Usage
db1 := database.GetInstance()
db2 := database.GetInstance()

fmt.Println(db1 == db2) // true - same instance
```

#### A Race Condition Hiding in the Node.js Version

Look again at the Node.js `connect()` method:

```typescript
public async connect(config: any): Promise<void> {
  if (!this.connection) {
    this.connection = await createConnection(config); // <-- await here
    console.log('Database connected');
  }
}
```

The Java version above guards its equivalent check with `synchronized` and double-checked locking, explicitly called out as "thread-safe." The Node.js version has no equivalent guard — and it needs one. JavaScript's event loop is single-threaded, but that does **not** mean this code is free of race conditions, because `await` yields control back to the event loop. If two requests both call `db1.connect(config)` around the same time:

1. Request A checks `if (!this.connection)` → true, starts `await createConnection(config)`
2. Before A's `await` resolves, Request B runs, checks `if (!this.connection)` → **still true**, because A hasn't assigned `this.connection` yet
3. Both A and B now call `createConnection(config)` — two connections get created, and whichever assignment happens last silently wins

This is exactly the bug the Java double-checked lock exists to prevent — "single-threaded" only means one line of synchronous code runs at a time, not that concurrent `async` operations can't interleave around an `await`. A correct Node.js version needs its own guard, for example caching the in-flight promise itself rather than just the eventual connection:

```typescript
private connectingPromise: Promise<any> | null = null;

public async connect(config: any): Promise<void> {
  if (this.connection) return;
  if (!this.connectingPromise) {
    this.connectingPromise = createConnection(config);
  }
  this.connection = await this.connectingPromise;
}
```

Now every concurrent caller awaits the _same_ in-flight promise instead of each starting its own connection attempt.

**Pros:**

- Controlled access to single instance
- Reduced memory footprint
- Global access point

**Cons:**

- Hard to test (mocking difficult)
- Hidden dependencies
- Potential threading issues (if not implemented correctly)
- Violates Single Responsibility Principle

**SaaS Use Cases:**

- Database connection pool
- Application configuration
- Logger instance
- Cache manager

#### Singleton vs a Dependency Injection Container

The hand-rolled `getInstance()` pattern above is worth knowing, but in a real Spring Boot application (the stack used for the Java examples throughout this course) you will rarely write it yourself. A Spring bean annotated `@Component` or `@Service` is **already singleton-scoped by default** — the container creates it once and hands out the same instance to every class that asks for it via constructor injection:

```java
// No getInstance() needed — Spring's container is already a singleton registry
@Service
public class DatabaseConnectionService {
    // Spring creates exactly one instance of this bean by default
    public Connection getConnection() { /* ... */ }
}

@Service
public class ProjectService {
    private final DatabaseConnectionService dbService;

    // Spring injects the same singleton instance here
    public ProjectService(DatabaseConnectionService dbService) {
        this.dbService = dbService;
    }
}
```

This gets you the "only one instance" guarantee of the Singleton pattern without its worst downside: because `ProjectService` receives its dependency through the constructor rather than reaching for a static `getInstance()`, a test can simply pass in a mock `DatabaseConnectionService` — no global state to reset between tests, no hidden coupling to track down. The equivalent NestJS (`@Injectable()`) and Go (manual constructor wiring, or a lightweight DI library) setups follow the same idea: let a container or explicit wiring own the "one instance" guarantee, rather than baking `getInstance()` into the class itself.

#### The "Singleton as Anti-Pattern" Debate

The Cons above list "hard to test" and "hidden dependencies" — it's worth naming why these are considered serious enough that many experienced developers treat Singleton as an anti-pattern rather than a pattern to reach for by default:

- **Global mutable state.** A singleton's internal state is shared by every piece of code that touches it, anywhere in the program. A bug caused by one caller's usage can surface in a completely unrelated part of the codebase.
- **Hidden dependencies.** A method that calls `DatabaseConnection.getInstance()` internally doesn't reveal that dependency in its signature — you have to read the method body to discover it, unlike a constructor parameter which documents the dependency up front.
- **Untestable in isolation.** Because the dependency isn't passed in, a unit test can't easily substitute a fake or mock for it — you're stuck with the real singleton (or global monkey-patching tricks) even when you just want to test unrelated logic.

The standard alternative is exactly what Dependency Inversion argued for earlier in this module: **inject the dependency instead of reaching for it globally.**

```typescript
// ❌ Hidden dependency via singleton
class ProjectService {
  async createProject(data: any) {
    const db = DatabaseConnection.getInstance().getConnection(); // hidden!
    return db.query("INSERT INTO projects ...", [data]);
  }
}

// ✅ Explicit dependency via constructor injection
class ProjectService {
  constructor(private db: DatabaseConnection) {} // visible in the signature

  async createProject(data: any) {
    return this.db.getConnection().query("INSERT INTO projects ...", [data]);
  }
}

// Testing becomes straightforward:
const fakeDb = { getConnection: () => mockConnection };
const service = new ProjectService(fakeDb as any);
// no global state, no real database needed
```

A single instance can still exist at runtime — the DI container (or your composition root) can construct exactly one `DatabaseConnection` and hand it to every service that needs it. The difference is that no class _reaches for_ it globally; every class _receives_ it. That's what makes the second version trivial to test and the first version genuinely painful.

---

### Pattern 2: Factory

"Factory" is often used loosely to mean "a function or class whose job is to create other objects." In practice this covers at least three related but distinct ideas, worth telling apart.

#### 2a. Simple Factory

**Purpose:** Centralize object creation behind a single method, so calling code doesn't need to know which concrete class to instantiate.

A quick naming note before the code: what follows is usually what people mean when they casually say "the Factory pattern" in day-to-day work — but it's technically called the **Simple Factory**, and it isn't even one of the 23 original GoF patterns; it's just a very common idiom, easily confused with the GoF's actual **Factory Method** pattern (covered next). Knowing the difference matters mainly so that when you encounter a "real" Factory Method elsewhere — built around subclassing rather than a switch statement — you're not thrown off by how different it looks from this.

**When to Use:**

- When you don't know beforehand the exact types of objects needed
- When object creation logic is complex
- When you want to centralize object creation

##### Example: Notification Factory

**Problem:** We need to send notifications via multiple channels (email, SMS, push, Slack).

##### Implementation: Node.js (TypeScript)

```typescript
// notifications/INotificationChannel.ts
interface INotificationChannel {
  send(recipient: string, message: string): Promise<void>;
}

// notifications/EmailChannel.ts
class EmailChannel implements INotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`Sending email to ${recipient}: ${message}`);
    // Actual email sending logic
    await emailService.send({
      to: recipient,
      subject: "Notification",
      body: message,
    });
  }
}

// notifications/SMSChannel.ts
class SMSChannel implements INotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`Sending SMS to ${recipient}: ${message}`);
    // Actual SMS sending logic
    await twilioClient.messages.create({
      to: recipient,
      body: message,
    });
  }
}

// notifications/PushChannel.ts
class PushChannel implements INotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`Sending push notification to ${recipient}: ${message}`);
    // Actual push notification logic
    await fcm.send({
      token: recipient,
      notification: { body: message },
    });
  }
}

// notifications/SlackChannel.ts
class SlackChannel implements INotificationChannel {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`Sending Slack message to ${recipient}: ${message}`);
    // Actual Slack sending logic
    await slackClient.chat.postMessage({
      channel: recipient,
      text: message,
    });
  }
}

// notifications/NotificationFactory.ts
class NotificationFactory {
  static create(channel: string): INotificationChannel {
    switch (channel) {
      case "email":
        return new EmailChannel();
      case "sms":
        return new SMSChannel();
      case "push":
        return new PushChannel();
      case "slack":
        return new SlackChannel();
      default:
        throw new Error(`Unknown notification channel: ${channel}`);
    }
  }

  // Alternative: Create based on user preferences
  static createForUser(user: User): INotificationChannel[] {
    const channels: INotificationChannel[] = [];

    if (user.preferences.emailNotifications) {
      channels.push(new EmailChannel());
    }
    if (user.preferences.smsNotifications) {
      channels.push(new SMSChannel());
    }
    if (user.preferences.pushNotifications) {
      channels.push(new PushChannel());
    }

    return channels;
  }
}

// Usage
const channel = NotificationFactory.create("email");
await channel.send("user@example.com", "Hello, World!");

// Or send to multiple channels based on user preference
const user = await userService.getUser(userId);
const channels = NotificationFactory.createForUser(user);

for (const channel of channels) {
  await channel.send(user.contactInfo, "Your report is ready!");
}
```

##### Implementation: Java

```java
// notifications/INotificationChannel.java
public interface INotificationChannel {
    void send(String recipient, String message) throws Exception;
}

// notifications/EmailChannel.java
public class EmailChannel implements INotificationChannel {
    @Override
    public void send(String recipient, String message) throws Exception {
        System.out.println("Sending email to " + recipient + ": " + message);
        // Actual email sending logic
        emailService.send(recipient, "Notification", message);
    }
}

// notifications/SMSChannel.java
public class SMSChannel implements INotificationChannel {
    @Override
    public void send(String recipient, String message) throws Exception {
        System.out.println("Sending SMS to " + recipient + ": " + message);
        // Actual SMS sending logic
        twilioClient.sendSMS(recipient, message);
    }
}

// notifications/NotificationFactory.java
public class NotificationFactory {
    public static INotificationChannel create(String channel) {
        switch (channel) {
            case "email":
                return new EmailChannel();
            case "sms":
                return new SMSChannel();
            case "push":
                return new PushChannel();
            case "slack":
                return new SlackChannel();
            default:
                throw new IllegalArgumentException("Unknown channel: " + channel);
        }
    }

    public static List<INotificationChannel> createForUser(User user) {
        List<INotificationChannel> channels = new ArrayList<>();

        if (user.getPreferences().isEmailNotifications()) {
            channels.add(new EmailChannel());
        }
        if (user.getPreferences().isSmsNotifications()) {
            channels.add(new SMSChannel());
        }
        if (user.getPreferences().isPushNotifications()) {
            channels.add(new PushChannel());
        }

        return channels;
    }
}

// Usage
INotificationChannel channel = NotificationFactory.create("email");
channel.send("user@example.com", "Hello, World!");
```

##### Implementation: Go

```go
// notifications/channel.go
package notifications

type NotificationChannel interface {
    Send(recipient string, message string) error
}

// notifications/email.go
type EmailChannel struct{}

func (e *EmailChannel) Send(recipient string, message string) error {
    fmt.Printf("Sending email to %s: %s\n", recipient, message)
    // Actual email sending logic
    return emailService.Send(recipient, "Notification", message)
}

// notifications/sms.go
type SMSChannel struct{}

func (s *SMSChannel) Send(recipient string, message string) error {
    fmt.Printf("Sending SMS to %s: %s\n", recipient, message)
    // Actual SMS sending logic
    return twilioClient.SendSMS(recipient, message)
}

// notifications/factory.go
func CreateChannel(channelType string) (NotificationChannel, error) {
    switch channelType {
    case "email":
        return &EmailChannel{}, nil
    case "sms":
        return &SMSChannel{}, nil
    case "push":
        return &PushChannel{}, nil
    case "slack":
        return &SlackChannel{}, nil
    default:
        return nil, fmt.Errorf("unknown channel type: %s", channelType)
    }
}

func CreateChannelsForUser(user *User) []NotificationChannel {
    var channels []NotificationChannel

    if user.Preferences.EmailNotifications {
        channels = append(channels, &EmailChannel{})
    }
    if user.Preferences.SMSNotifications {
        channels = append(channels, &SMSChannel{})
    }
    if user.Preferences.PushNotifications {
        channels = append(channels, &PushChannel{})
    }

    return channels
}

// Usage
channel, err := notifications.CreateChannel("email")
if err != nil {
    log.Fatal(err)
}
err = channel.Send("user@example.com", "Hello, World!")
```

**Pros:**

- Decouples client code from concrete classes
- Easy to add new types without modifying existing code (Open/Closed Principle)
- Centralized object creation logic

**Cons:**

- Can lead to many small classes
- Factory must be updated when adding new types

**SaaS Use Cases:**

- Creating different notification channels
- Creating different payment gateway instances
- Creating different storage providers (S3, Azure Blob, Google Cloud Storage)
- Creating different authentication providers (JWT, OAuth, SAML)

#### 2b. Factory Method (the GoF Pattern)

The Simple Factory above centralizes creation behind a single method with a switch statement. The GoF's actual **Factory Method** pattern solves a related but different problem: it lets **subclasses** decide which concrete class to instantiate, by overriding a creation method — the branching is done through polymorphism, not a conditional.

The shape looks like this: an abstract `Creator` class declares a `createProduct()` method (often with a default or abstract implementation), and concrete subclasses override it to return their own product type. Client code calls a method on the `Creator` without ever knowing which concrete product comes back.

```typescript
// The abstract Creator
abstract class NotificationChannelCreator {
  // Factory Method - subclasses override this
  abstract createChannel(): INotificationChannel;

  // Client-facing logic that uses the product,
  // without knowing its concrete type
  async notify(recipient: string, message: string): Promise<void> {
    const channel = this.createChannel();
    await channel.send(recipient, message);
  }
}

// Concrete Creators - each decides which product to build
class EmailChannelCreator extends NotificationChannelCreator {
  createChannel(): INotificationChannel {
    return new EmailChannel();
  }
}

class SMSChannelCreator extends NotificationChannelCreator {
  createChannel(): INotificationChannel {
    return new SMSChannel();
  }
}

// Usage
const creator: NotificationChannelCreator = new EmailChannelCreator();
await creator.notify("user@example.com", "Hello, World!");
```

Notice what changed compared to the Simple Factory: there's no `switch` anywhere. Choosing `EmailChannelCreator` versus `SMSChannelCreator` _is_ the decision — each subclass already knows what it creates. Adding a new channel means adding a new `Creator` subclass, not adding a new `case` to an existing method.

In most day-to-day SaaS code, developers reach for the Simple Factory shown above and just call it "the Factory pattern" — it's simpler and usually sufficient. The real Factory Method is more common in frameworks and libraries (where subclassing is already the extension mechanism) than in typical application code. Knowing it exists mainly protects you from confusion the first time you encounter it elsewhere and it doesn't look anything like the switch-based factories you're used to.

#### 2c. Abstract Factory

**Purpose:** Create **families** of related objects that must stay consistent with each other, without the client code specifying their concrete classes.

Where a (Simple or Method) Factory produces one kind of product, an Abstract Factory produces a _matched set_ of several related products, guaranteeing they're compatible with each other.

**SaaS-relevant example:** a multi-cloud SaaS product needs `Storage`, `Queue`, and `Database` clients — but they must always come from the _same_ cloud provider. Mixing an AWS storage client with a GCP queue client would be a configuration bug; an Abstract Factory prevents that by construction.

```typescript
// The family of related products
interface Storage {
  upload(file: Buffer): Promise<string>;
}
interface Queue {
  publish(message: any): Promise<void>;
}
interface Database {
  query(sql: string): Promise<any>;
}

// The Abstract Factory interface
interface CloudProviderFactory {
  createStorage(): Storage;
  createQueue(): Queue;
  createDatabase(): Database;
}

// Concrete factory: AWS family
class AWSProviderFactory implements CloudProviderFactory {
  createStorage(): Storage {
    return new S3Storage();
  }
  createQueue(): Queue {
    return new SQSQueue();
  }
  createDatabase(): Database {
    return new RDSDatabase();
  }
}

// Concrete factory: GCP family
class GCPProviderFactory implements CloudProviderFactory {
  createStorage(): Storage {
    return new GCSStorage();
  }
  createQueue(): Queue {
    return new PubSubQueue();
  }
  createDatabase(): Database {
    return new CloudSQLDatabase();
  }
}

// Usage: swapping the whole family is a one-line change
function buildInfrastructure(factory: CloudProviderFactory) {
  const storage = factory.createStorage();
  const queue = factory.createQueue();
  const database = factory.createDatabase();
  // storage, queue, and database are guaranteed to belong to the same provider
  return { storage, queue, database };
}

const infra = buildInfrastructure(new AWSProviderFactory());
```

The distinction from what came before: Simple Factory produces one product from a flat list of options via a conditional; Factory Method produces one product via subclass polymorphism; Abstract Factory produces a _whole family_ of related products via a factory _object_ passed around as a dependency — and swapping that one object swaps every product in the family consistently.

---

### Pattern 3: Builder

**Purpose:** Separate the construction of a complex object from its representation, allowing step-by-step construction.

**When to Use:**

- Object construction requires many optional parameters
- Want to create different representations using same construction process
- Construction process is complex

#### Example: Query Builder

**Problem:** Building complex database queries with many optional filters.

#### Implementation: Node.js (TypeScript)

```typescript
// database/QueryBuilder.ts
class QueryBuilder {
  private table: string = "";
  private selectColumns: string[] = ["*"];
  private whereConditions: string[] = [];
  private orderByClause: string = "";
  private limitValue: number | null = null;
  private offsetValue: number | null = null;
  private params: any[] = [];

  // Start building a query
  static from(table: string): QueryBuilder {
    const builder = new QueryBuilder();
    builder.table = table;
    return builder;
  }

  // Select specific columns
  select(...columns: string[]): this {
    this.selectColumns = columns;
    return this;
  }

  // Add WHERE condition
  where(condition: string, ...params: any[]): this {
    this.whereConditions.push(condition);
    this.params.push(...params);
    return this;
  }

  // Add AND condition
  and(condition: string, ...params: any[]): this {
    return this.where(condition, ...params);
  }

  // Add OR condition
  or(condition: string, ...params: any[]): this {
    if (this.whereConditions.length > 0) {
      const lastCondition = this.whereConditions.pop();
      this.whereConditions.push(`(${lastCondition} OR ${condition})`);
    } else {
      this.whereConditions.push(condition);
    }
    this.params.push(...params);
    return this;
  }

  // Order by
  orderBy(column: string, direction: "ASC" | "DESC" = "ASC"): this {
    this.orderByClause = `ORDER BY ${column} ${direction}`;
    return this;
  }

  // Limit
  limit(value: number): this {
    this.limitValue = value;
    return this;
  }

  // Offset
  offset(value: number): this {
    this.offsetValue = value;
    return this;
  }

  // Build the final query
  build(): { sql: string; params: any[] } {
    let sql = `SELECT ${this.selectColumns.join(", ")} FROM ${this.table}`;

    if (this.whereConditions.length > 0) {
      sql += ` WHERE ${this.whereConditions.join(" AND ")}`;
    }

    if (this.orderByClause) {
      sql += ` ${this.orderByClause}`;
    }

    if (this.limitValue !== null) {
      sql += ` LIMIT ${this.limitValue}`;
    }

    if (this.offsetValue !== null) {
      sql += ` OFFSET ${this.offsetValue}`;
    }

    return { sql, params: this.params };
  }

  // Execute the query
  async execute(db: any): Promise<any[]> {
    const { sql, params } = this.build();
    const result = await db.query(sql, params);
    return result.rows;
  }
}

// Usage - Simple query
const query1 = QueryBuilder.from("users")
  .select("id", "name", "email")
  .where("tenant_id = $1", tenantId)
  .orderBy("created_at", "DESC")
  .limit(10)
  .build();

console.log(query1.sql);
// SELECT id, name, email FROM users
// WHERE tenant_id = $1
// ORDER BY created_at DESC
// LIMIT 10

// Usage - Complex query with multiple conditions
const activeUsers = await QueryBuilder.from("users")
  .where("tenant_id = $1", tenantId)
  .and("status = $2", "active")
  .and("created_at > $3", new Date("2024-01-01"))
  .orderBy("last_login", "DESC")
  .limit(50)
  .execute(db);

// Usage - Search with OR conditions
const searchResults = await QueryBuilder.from("projects")
  .select("id", "name", "description")
  .where("tenant_id = $1", tenantId)
  .and("(name ILIKE $2", `%${searchTerm}%`)
  .or("description ILIKE $3)", `%${searchTerm}%`)
  .orderBy("updated_at", "DESC")
  .execute(db);
```

#### More Complex Builder Example: Email Builder

```typescript
// email/EmailBuilder.ts
interface EmailAttachment {
  filename: string;
  content: Buffer | string;
  contentType?: string;
}

class Email {
  constructor(
    public from: string,
    public to: string[],
    public cc: string[],
    public bcc: string[],
    public subject: string,
    public textBody: string,
    public htmlBody: string,
    public attachments: EmailAttachment[],
    public headers: Record<string, string>,
  ) {}
}

class EmailBuilder {
  private from: string = "";
  private to: string[] = [];
  private cc: string[] = [];
  private bcc: string[] = [];
  private subject: string = "";
  private textBody: string = "";
  private htmlBody: string = "";
  private attachments: EmailAttachment[] = [];
  private headers: Record<string, string> = {};

  setFrom(email: string): this {
    this.from = email;
    return this;
  }

  addTo(...emails: string[]): this {
    this.to.push(...emails);
    return this;
  }

  addCc(...emails: string[]): this {
    this.cc.push(...emails);
    return this;
  }

  addBcc(...emails: string[]): this {
    this.bcc.push(...emails);
    return this;
  }

  setSubject(subject: string): this {
    this.subject = subject;
    return this;
  }

  setTextBody(body: string): this {
    this.textBody = body;
    return this;
  }

  setHtmlBody(body: string): this {
    this.htmlBody = body;
    return this;
  }

  addAttachment(attachment: EmailAttachment): this {
    this.attachments.push(attachment);
    return this;
  }

  addHeader(key: string, value: string): this {
    this.headers[key] = value;
    return this;
  }

  build(): Email {
    // Validation
    if (!this.from) {
      throw new Error("From email is required");
    }
    if (this.to.length === 0) {
      throw new Error("At least one recipient is required");
    }
    if (!this.subject) {
      throw new Error("Subject is required");
    }
    if (!this.textBody && !this.htmlBody) {
      throw new Error("Email body is required");
    }

    return new Email(
      this.from,
      this.to,
      this.cc,
      this.bcc,
      this.subject,
      this.textBody,
      this.htmlBody,
      this.attachments,
      this.headers,
    );
  }
}

// Usage
const email = new EmailBuilder()
  .setFrom("noreply@app.com")
  .addTo("user@example.com")
  .addCc("admin@example.com")
  .setSubject("Welcome to Our SaaS!")
  .setTextBody("Thank you for signing up!")
  .setHtmlBody("<h1>Thank you for signing up!</h1>")
  .addAttachment({
    filename: "welcome.pdf",
    content: pdfBuffer,
    contentType: "application/pdf",
  })
  .addHeader("X-Priority", "1")
  .build();

await emailService.send(email);
```

#### Fluent Builder vs the Original GoF Builder

Both examples above use the style most developers mean today when they say "Builder pattern": a **fluent interface**, where each setter method returns `this`, allowing calls to be chained one after another. It's simple, reads well, and is by far the most common variant in real-world code — but it's a simplified take on the pattern as originally described by the GoF.

The original GoF Builder separates two responsibilities into two different objects:

- A **Builder** — knows _how_ to construct something step by step (add a wheel, add an engine, add seats), exposing simple methods that build one part at a time.
- A **Director** — knows _what sequence of steps_ to run against a Builder to produce a specific, named configuration (a "SportsCar" director calls a different sequence than a "Truck" director), without knowing anything about how each step is actually implemented.

This separation earns its keep when the same set of construction steps needs to reliably produce several distinct, named configurations — the Director encodes each configuration as a fixed recipe, so callers don't have to remember the right chain of method calls themselves:

```typescript
// The Builder only knows how to do individual steps
interface CarBuilder {
  reset(): void;
  setSeats(count: number): void;
  setEngine(type: string): void;
  setGPS(hasGPS: boolean): void;
  getResult(): Car;
}

// The Director knows the recipes
class CarDirector {
  makeSportsCar(builder: CarBuilder): Car {
    builder.reset();
    builder.setSeats(2);
    builder.setEngine("V8");
    builder.setGPS(true);
    return builder.getResult();
  }

  makeFamilyCar(builder: CarBuilder): Car {
    builder.reset();
    builder.setSeats(7);
    builder.setEngine("V6");
    builder.setGPS(true);
    return builder.getResult();
  }
}

// Usage: the caller just names the configuration, never the steps
const director = new CarDirector();
const sportsCar = director.makeSportsCar(new StandardCarBuilder());
```

For the QueryBuilder and EmailBuilder above, a Director would be overkill — each call site naturally expresses its own unique combination of steps, and there's no fixed set of named configurations to encode. That's exactly why the fluent style, without a Director, is the right (and far more common) choice for those cases. Reach for the full GoF split only when you find yourself repeating the _same_ sequence of builder calls in several places — at that point, naming that sequence inside a Director removes the duplication.

**Pros:**

- Clear, readable object construction
- Immutable objects (if built correctly)
- Can create different representations
- Step-by-step construction

**Cons:**

- More verbose than simple constructors
- Requires separate builder class

**SaaS Use Cases:**

- Complex query construction
- Email/notification message building
- API request building
- Report generation with many options
- Configuration objects

---
