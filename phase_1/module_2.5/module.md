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

***

## Part 1: Pattern Fundamentals (Day 1)

### What Are Design Patterns?

**Design Patterns** are reusable solutions to common problems in software design. They're like recipes - proven approaches that experienced developers have refined over time.

**Key Points:**

* Patterns are NOT code you copy-paste
* Patterns are templates for solving problems
* Patterns provide a common vocabulary for teams
* Patterns capture best practices

**The Gang of Four (GoF):** In 1994, four authors published "Design Patterns: Elements of Reusable Object-Oriented Software" - the foundation of pattern thinking.

***

### SOLID Principles (Quick Review)

Before diving into patterns, let's review SOLID - principles that guide good design:

#### S - Single Responsibility Principle

**"A class should have one, and only one, reason to change"**

```javascript
// ❌ BAD: UserService does too much
class UserService {
  createUser(data) { /* creates user */ }
  sendWelcomeEmail(user) { /* sends email */ }
  validateUser(user) { /* validates data */ }
  generateReport(userId) { /* generates report */ }
}

// ✅ GOOD: Separate responsibilities
class UserService {
  createUser(data) { /* creates user */ }
}

class EmailService {
  sendWelcomeEmail(user) { /* sends email */ }
}

class UserValidator {
  validate(user) { /* validates data */ }
}

class ReportService {
  generateUserReport(userId) { /* generates report */ }
}
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

***

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

***

## Part 2: Creational Patterns (Days 1-2)

Creational patterns deal with object creation mechanisms.

### Pattern 1: Singleton

**Purpose:** Ensure a class has only one instance and provide global access to it.

**When to Use:**

* Database connection pools
* Configuration managers
* Logger instances
* Cache managers
* Application-wide state

**When NOT to Use:**

* When you need multiple instances
* When testing (singletons are hard to mock)
* When it hides dependencies

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
      console.log('Database connected');
    }
  }
  
  public getConnection(): any {
    if (!this.connection) {
      throw new Error('Database not connected. Call connect() first.');
    }
    return this.connection;
  }
  
  public async disconnect(): Promise<void> {
    if (this.connection) {
      await this.connection.close();
      this.connection = null;
      console.log('Database disconnected');
    }
  }
}

// Usage
const db1 = DatabaseConnection.getInstance();
const db2 = DatabaseConnection.getInstance();

console.log(db1 === db2); // true - same instance

await db1.connect({ host: 'localhost', port: 5432 });
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

**Pros:**

* Controlled access to single instance
* Reduced memory footprint
* Global access point

**Cons:**

* Hard to test (mocking difficult)
* Hidden dependencies
* Potential threading issues (if not implemented correctly)
* Violates Single Responsibility Principle

**SaaS Use Cases:**

* Database connection pool
* Application configuration
* Logger instance
* Cache manager

***

### Pattern 2: Factory Method

**Purpose:** Define an interface for creating objects, but let subclasses decide which class to instantiate.

**When to Use:**

* When you don't know beforehand the exact types of objects needed
* When object creation logic is complex
* When you want to centralize object creation

#### Example: Notification Factory

**Problem:** We need to send notifications via multiple channels (email, SMS, push, Slack).

#### Implementation: Node.js (TypeScript)

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
      subject: 'Notification',
      body: message
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
      body: message
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
      notification: { body: message }
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
      text: message
    });
  }
}

// notifications/NotificationFactory.ts
class NotificationFactory {
  static create(channel: string): INotificationChannel {
    switch (channel) {
      case 'email':
        return new EmailChannel();
      case 'sms':
        return new SMSChannel();
      case 'push':
        return new PushChannel();
      case 'slack':
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
const channel = NotificationFactory.create('email');
await channel.send('user@example.com', 'Hello, World!');

// Or send to multiple channels based on user preference
const user = await userService.getUser(userId);
const channels = NotificationFactory.createForUser(user);

for (const channel of channels) {
  await channel.send(user.contactInfo, 'Your report is ready!');
}
```

#### Implementation: Java

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

#### Implementation: Go

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

* Decouples client code from concrete classes
* Easy to add new types without modifying existing code (Open/Closed Principle)
* Centralized object creation logic

**Cons:**

* Can lead to many small classes
* Factory must be updated when adding new types

**SaaS Use Cases:**

* Creating different notification channels
* Creating different payment gateway instances
* Creating different storage providers (S3, Azure Blob, Google Cloud Storage)
* Creating different authentication providers (JWT, OAuth, SAML)

***

### Pattern 3: Builder

**Purpose:** Separate the construction of a complex object from its representation, allowing step-by-step construction.

**When to Use:**

* Object construction requires many optional parameters
* Want to create different representations using same construction process
* Construction process is complex

#### Example: Query Builder

**Problem:** Building complex database queries with many optional filters.

#### Implementation: Node.js (TypeScript)

```typescript
// database/QueryBuilder.ts
class QueryBuilder {
  private table: string = '';
  private selectColumns: string[] = ['*'];
  private whereConditions: string[] = [];
  private orderByClause: string = '';
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
  orderBy(column: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
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
    let sql = `SELECT ${this.selectColumns.join(', ')} FROM ${this.table}`;
    
    if (this.whereConditions.length > 0) {
      sql += ` WHERE ${this.whereConditions.join(' AND ')}`;
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
const query1 = QueryBuilder
  .from('users')
  .select('id', 'name', 'email')
  .where('tenant_id = $1', tenantId)
  .orderBy('created_at', 'DESC')
  .limit(10)
  .build();

console.log(query1.sql);
// SELECT id, name, email FROM users 
// WHERE tenant_id = $1 
// ORDER BY created_at DESC 
// LIMIT 10

// Usage - Complex query with multiple conditions
const activeUsers = await QueryBuilder
  .from('users')
  .where('tenant_id = $1', tenantId)
  .and('status = $2', 'active')
  .and('created_at > $3', new Date('2024-01-01'))
  .orderBy('last_login', 'DESC')
  .limit(50)
  .execute(db);

// Usage - Search with OR conditions
const searchResults = await QueryBuilder
  .from('projects')
  .select('id', 'name', 'description')
  .where('tenant_id = $1', tenantId)
  .and('(name ILIKE $2', `%${searchTerm}%`)
  .or('description ILIKE $3)', `%${searchTerm}%`)
  .orderBy('updated_at', 'DESC')
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
    public headers: Record<string, string>
  ) {}
}

class EmailBuilder {
  private from: string = '';
  private to: string[] = [];
  private cc: string[] = [];
  private bcc: string[] = [];
  private subject: string = '';
  private textBody: string = '';
  private htmlBody: string = '';
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
      throw new Error('From email is required');
    }
    if (this.to.length === 0) {
      throw new Error('At least one recipient is required');
    }
    if (!this.subject) {
      throw new Error('Subject is required');
    }
    if (!this.textBody && !this.htmlBody) {
      throw new Error('Email body is required');
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
      this.headers
    );
  }
}

// Usage
const email = new EmailBuilder()
  .setFrom('noreply@app.com')
  .addTo('user@example.com')
  .addCc('admin@example.com')
  .setSubject('Welcome to Our SaaS!')
  .setTextBody('Thank you for signing up!')
  .setHtmlBody('<h1>Thank you for signing up!</h1>')
  .addAttachment({
    filename: 'welcome.pdf',
    content: pdfBuffer,
    contentType: 'application/pdf'
  })
  .addHeader('X-Priority', '1')
  .build();

await emailService.send(email);
```

**Pros:**

* Clear, readable object construction
* Immutable objects (if built correctly)
* Can create different representations
* Step-by-step construction

**Cons:**

* More verbose than simple constructors
* Requires separate builder class

**SaaS Use Cases:**

* Complex query construction
* Email/notification message building
* API request building
* Report generation with many options
* Configuration objects

***

## Next: Continue to Part 3 (Structural Patterns) in next response...

Should I continue with Part 3: Structural Patterns (Repository, Adapter, Decorator, Proxy)?
