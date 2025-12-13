# Module 2.5: Core Design Patterns for SaaS Applications

---

## Part 4: Behavioral Patterns - Strategy Pattern

---

## 🎯 Learning Objectives

By the end of this section, you will:

- Understand what the Strategy pattern is and when to use it
- Recognize scenarios where Strategy shines in SaaS applications
- Implement Strategy pattern in Java, Node.js, and Go
- Apply Strategy for pricing models, payment gateways, and algorithms
- Switch behaviors at runtime without modifying client code
- Avoid common pitfalls when using Strategy

---

## 📖 Conceptual Overview

### What is the Strategy Pattern?

The **Strategy pattern** defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

**Key Idea**: Instead of implementing a single algorithm directly, code receives run-time instructions as to which in a family of algorithms to use.

### Real-World Analogy

Think of different payment methods:

- **Credit Card**: Instant payment, 2.9% fee
- **Bank Transfer**: 2-3 day delay, $0 fee
- **PayPal**: Instant payment, 3.5% fee
- **Crypto**: Variable time, network fee

You (the client) want to pay, but the **strategy** (payment method) can change based on circumstances without changing how you initiate payment.

---

## 🤔 When to Use Strategy Pattern in SaaS

### Perfect Use Cases:

1. **Pricing Strategies** ⭐⭐⭐

   - Tiered pricing (Free, Pro, Enterprise)
   - Usage-based billing
   - Per-seat pricing
   - Dynamic pricing based on demand

2. **Payment Processing** ⭐⭐⭐

   - Multiple payment gateways (Stripe, PayPal, Square)
   - Different payment methods per region
   - Fallback payment processors

3. **Notification Delivery** ⭐⭐⭐

   - Email, SMS, Push, Slack, Webhook
   - User preference-based delivery
   - Multi-channel notifications

4. **Authentication Methods** ⭐⭐

   - Password, OAuth, SAML, Magic Link
   - SSO providers (Google, GitHub, Microsoft)
   - Multi-factor authentication strategies

5. **Data Export Formats** ⭐⭐

   - CSV, Excel, JSON, PDF, XML
   - User-selected format
   - API response formats

6. **Search Algorithms** ⭐⭐

   - Exact match, fuzzy search, full-text
   - Different engines (Elasticsearch, Postgres FTS)
   - Ranking algorithms

7. **Caching Strategies** ⭐⭐

   - LRU, LFU, FIFO
   - Cache-aside, write-through, write-behind
   - Different cache backends

8. **Validation Rules** ⭐⭐
   - Different validation per country
   - Custom validation per tenant
   - Industry-specific rules

---

## 🚫 When NOT to Use Strategy

- **Single algorithm** - If you only have one way of doing something
- **Algorithms rarely change** - If the behavior is stable and unlikely to vary
- **Too many strategies** - More than 10-15 strategies becomes hard to manage
- **Simple conditionals** - Don't over-engineer simple if/else logic

---

## 🏗️ Structure

```
Context ──uses──> Strategy (Interface)
                      ↑
                      |
        ┌─────────────┼─────────────┐
        |             |             |
ConcreteStrategyA  ConcreteStrategyB  ConcreteStrategyC
```

**Key Elements:**

1. **Strategy Interface**: Defines common interface for all algorithms
2. **Concrete Strategies**: Implement different algorithms
3. **Context**: Holds reference to a strategy and delegates work to it
4. **Client**: Chooses which strategy to use

---

## 💻 Implementation in Java/Spring Boot

### Scenario 1: Pricing Strategies

We'll build a subscription pricing system with different pricing models.

#### Step 1: Define Strategy Interface

```java
// PricingStrategy.java - Strategy interface
package com.saas.pricing;

import com.saas.model.Subscription;
import java.math.BigDecimal;

public interface PricingStrategy {
    /**
     * Calculate price for a subscription
     * @param subscription The subscription to price
     * @return The calculated price
     */
    BigDecimal calculatePrice(Subscription subscription);

    /**
     * Get strategy name for logging/display
     */
    String getStrategyName();
}
```

#### Step 2: Implement Concrete Strategies

```java
// TieredPricingStrategy.java - Fixed tier pricing
package com.saas.pricing.strategy;

import com.saas.model.Subscription;
import com.saas.pricing.PricingStrategy;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;

@Slf4j
public class TieredPricingStrategy implements PricingStrategy {

    private static final BigDecimal FREE_PRICE = BigDecimal.ZERO;
    private static final BigDecimal STARTER_PRICE = new BigDecimal("29.99");
    private static final BigDecimal PRO_PRICE = new BigDecimal("99.99");
    private static final BigDecimal ENTERPRISE_PRICE = new BigDecimal("499.99");

    @Override
    public BigDecimal calculatePrice(Subscription subscription) {
        log.info("Calculating tiered price for plan: {}", subscription.getPlan());

        return switch (subscription.getPlan()) {
            case "FREE" -> FREE_PRICE;
            case "STARTER" -> STARTER_PRICE;
            case "PRO" -> PRO_PRICE;
            case "ENTERPRISE" -> ENTERPRISE_PRICE;
            default -> throw new IllegalArgumentException("Unknown plan: " + subscription.getPlan());
        };
    }

    @Override
    public String getStrategyName() {
        return "Tiered Pricing";
    }
}
```

```java
// PerSeatPricingStrategy.java - Price per user/seat
package com.saas.pricing.strategy;

import com.saas.model.Subscription;
import com.saas.pricing.PricingStrategy;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;

@Slf4j
public class PerSeatPricingStrategy implements PricingStrategy {

    private static final BigDecimal PRICE_PER_SEAT = new BigDecimal("15.00");
    private static final int MINIMUM_SEATS = 1;

    @Override
    public BigDecimal calculatePrice(Subscription subscription) {
        int seats = Math.max(subscription.getSeats(), MINIMUM_SEATS);
        BigDecimal price = PRICE_PER_SEAT.multiply(BigDecimal.valueOf(seats));

        log.info("Calculating per-seat price: {} seats × ${} = ${}",
            seats, PRICE_PER_SEAT, price);

        return price;
    }

    @Override
    public String getStrategyName() {
        return "Per-Seat Pricing";
    }
}
```

```java
// UsageBasedPricingStrategy.java - Pay for what you use
package com.saas.pricing.strategy;

import com.saas.model.Subscription;
import com.saas.pricing.PricingStrategy;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.math.RoundingMode;

@Slf4j
public class UsageBasedPricingStrategy implements PricingStrategy {

    private static final BigDecimal BASE_PRICE = new BigDecimal("10.00");
    private static final BigDecimal PRICE_PER_API_CALL = new BigDecimal("0.001"); // $0.001 per call
    private static final BigDecimal PRICE_PER_GB_STORAGE = new BigDecimal("0.50"); // $0.50 per GB
    private static final int FREE_API_CALLS = 1000;
    private static final int FREE_GB_STORAGE = 5;

    @Override
    public BigDecimal calculatePrice(Subscription subscription) {
        BigDecimal totalPrice = BASE_PRICE;

        // Calculate API usage cost
        int billableApiCalls = Math.max(0, subscription.getApiCalls() - FREE_API_CALLS);
        BigDecimal apiCost = PRICE_PER_API_CALL.multiply(BigDecimal.valueOf(billableApiCalls));

        // Calculate storage cost
        int billableStorage = Math.max(0, subscription.getStorageGB() - FREE_GB_STORAGE);
        BigDecimal storageCost = PRICE_PER_GB_STORAGE.multiply(BigDecimal.valueOf(billableStorage));

        totalPrice = totalPrice.add(apiCost).add(storageCost);
        totalPrice = totalPrice.setScale(2, RoundingMode.HALF_UP);

        log.info("Usage-based pricing: Base=${}, API=${}, Storage=${}, Total=${}",
            BASE_PRICE, apiCost, storageCost, totalPrice);

        return totalPrice;
    }

    @Override
    public String getStrategyName() {
        return "Usage-Based Pricing";
    }
}
```

```java
// VolumeDiscountPricingStrategy.java - Bulk discounts
package com.saas.pricing.strategy;

import com.saas.model.Subscription;
import com.saas.pricing.PricingStrategy;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.math.RoundingMode;

@Slf4j
public class VolumeDiscountPricingStrategy implements PricingStrategy {

    private static final BigDecimal BASE_PRICE_PER_UNIT = new BigDecimal("10.00");

    @Override
    public BigDecimal calculatePrice(Subscription subscription) {
        int quantity = subscription.getQuantity();
        BigDecimal discount = calculateDiscount(quantity);

        BigDecimal basePrice = BASE_PRICE_PER_UNIT.multiply(BigDecimal.valueOf(quantity));
        BigDecimal discountAmount = basePrice.multiply(discount);
        BigDecimal finalPrice = basePrice.subtract(discountAmount);

        log.info("Volume discount pricing: Qty={}, Base=${}, Discount={}%, Final=${}",
            quantity, basePrice, discount.multiply(BigDecimal.valueOf(100)), finalPrice);

        return finalPrice.setScale(2, RoundingMode.HALF_UP);
    }

    private BigDecimal calculateDiscount(int quantity) {
        if (quantity >= 100) {
            return new BigDecimal("0.20"); // 20% discount
        } else if (quantity >= 50) {
            return new BigDecimal("0.15"); // 15% discount
        } else if (quantity >= 20) {
            return new BigDecimal("0.10"); // 10% discount
        } else if (quantity >= 10) {
            return new BigDecimal("0.05"); // 5% discount
        }
        return BigDecimal.ZERO;
    }

    @Override
    public String getStrategyName() {
        return "Volume Discount Pricing";
    }
}
```

#### Step 3: Create Context Class

```java
// PricingContext.java - Context that uses strategies
package com.saas.pricing;

import com.saas.model.Subscription;
import lombok.Setter;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;

@Slf4j
public class PricingContext {

    @Setter
    private PricingStrategy strategy;

    public PricingContext(PricingStrategy strategy) {
        this.strategy = strategy;
    }

    /**
     * Calculate price using current strategy
     */
    public BigDecimal calculatePrice(Subscription subscription) {
        if (strategy == null) {
            throw new IllegalStateException("Pricing strategy not set");
        }

        log.info("Using pricing strategy: {}", strategy.getStrategyName());
        return strategy.calculatePrice(subscription);
    }

    /**
     * Switch to a different pricing strategy at runtime
     */
    public void changeStrategy(PricingStrategy newStrategy) {
        log.info("Switching pricing strategy from {} to {}",
            strategy.getStrategyName(), newStrategy.getStrategyName());
        this.strategy = newStrategy;
    }
}
```

#### Step 4: Strategy Factory (Optional but Recommended)

```java
// PricingStrategyFactory.java - Factory for creating strategies
package com.saas.pricing.factory;

import com.saas.pricing.PricingStrategy;
import com.saas.pricing.strategy.*;
import org.springframework.stereotype.Component;
import java.util.HashMap;
import java.util.Map;

@Component
public class PricingStrategyFactory {

    private final Map<String, PricingStrategy> strategies = new HashMap<>();

    public PricingStrategyFactory() {
        // Register all available strategies
        strategies.put("TIERED", new TieredPricingStrategy());
        strategies.put("PER_SEAT", new PerSeatPricingStrategy());
        strategies.put("USAGE_BASED", new UsageBasedPricingStrategy());
        strategies.put("VOLUME_DISCOUNT", new VolumeDiscountPricingStrategy());
    }

    public PricingStrategy getStrategy(String strategyName) {
        PricingStrategy strategy = strategies.get(strategyName.toUpperCase());
        if (strategy == null) {
            throw new IllegalArgumentException("Unknown pricing strategy: " + strategyName);
        }
        return strategy;
    }

    public Map<String, PricingStrategy> getAllStrategies() {
        return new HashMap<>(strategies);
    }
}
```

#### Step 5: Service Layer

```java
// SubscriptionService.java - Using pricing strategies
package com.saas.service;

import com.saas.model.Subscription;
import com.saas.pricing.PricingContext;
import com.saas.pricing.PricingStrategy;
import com.saas.pricing.factory.PricingStrategyFactory;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.math.BigDecimal;

@Service
@RequiredArgsConstructor
@Slf4j
public class SubscriptionService {

    private final PricingStrategyFactory strategyFactory;

    public BigDecimal calculateSubscriptionPrice(Subscription subscription) {
        // Get appropriate strategy based on subscription type
        PricingStrategy strategy = determineStrategy(subscription);

        // Create context with strategy
        PricingContext context = new PricingContext(strategy);

        // Calculate price
        return context.calculatePrice(subscription);
    }

    private PricingStrategy determineStrategy(Subscription subscription) {
        // Business logic to determine which strategy to use
        String pricingModel = subscription.getPricingModel();

        log.info("Determining pricing strategy for model: {}", pricingModel);
        return strategyFactory.getStrategy(pricingModel);
    }

    /**
     * Calculate price comparison across different strategies
     */
    public Map<String, BigDecimal> compareAllPricingStrategies(Subscription subscription) {
        Map<String, BigDecimal> comparison = new HashMap<>();

        for (Map.Entry<String, PricingStrategy> entry :
                strategyFactory.getAllStrategies().entrySet()) {

            PricingContext context = new PricingContext(entry.getValue());
            try {
                BigDecimal price = context.calculatePrice(subscription);
                comparison.put(entry.getKey(), price);
            } catch (Exception e) {
                log.warn("Could not calculate price for strategy {}: {}",
                    entry.getKey(), e.getMessage());
            }
        }

        return comparison;
    }
}
```

#### Step 6: REST Controller

```java
// PricingController.java
package com.saas.controller;

import com.saas.model.Subscription;
import com.saas.service.SubscriptionService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.math.BigDecimal;
import java.util.Map;

@RestController
@RequestMapping("/api/pricing")
@RequiredArgsConstructor
public class PricingController {

    private final SubscriptionService subscriptionService;

    @PostMapping("/calculate")
    public ResponseEntity<BigDecimal> calculatePrice(
            @RequestBody Subscription subscription) {
        BigDecimal price = subscriptionService.calculateSubscriptionPrice(subscription);
        return ResponseEntity.ok(price);
    }

    @PostMapping("/compare")
    public ResponseEntity<Map<String, BigDecimal>> compareStrategies(
            @RequestBody Subscription subscription) {
        Map<String, BigDecimal> comparison =
            subscriptionService.compareAllPricingStrategies(subscription);
        return ResponseEntity.ok(comparison);
    }
}
```

### Scenario 2: Payment Gateway Strategies

```java
// PaymentStrategy.java - Strategy interface
package com.saas.payment;

import com.saas.model.PaymentRequest;
import com.saas.model.PaymentResult;

public interface PaymentStrategy {
    PaymentResult processPayment(PaymentRequest request);
    boolean isAvailable();
    String getProviderName();
    BigDecimal getFeePercentage();
}
```

```java
// StripePaymentStrategy.java
package com.saas.payment.strategy;

import com.saas.model.PaymentRequest;
import com.saas.model.PaymentResult;
import com.saas.payment.PaymentStrategy;
import com.stripe.Stripe;
import com.stripe.model.Charge;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import java.math.BigDecimal;

@Component
@Slf4j
public class StripePaymentStrategy implements PaymentStrategy {

    @Value("${stripe.api.key}")
    private String apiKey;

    private static final BigDecimal FEE_PERCENTAGE = new BigDecimal("0.029"); // 2.9%

    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        log.info("Processing payment via Stripe: ${}", request.getAmount());

        try {
            Stripe.apiKey = apiKey;

            Map<String, Object> params = new HashMap<>();
            params.put("amount", request.getAmount().multiply(BigDecimal.valueOf(100)).intValue());
            params.put("currency", request.getCurrency());
            params.put("source", request.getToken());
            params.put("description", request.getDescription());

            Charge charge = Charge.create(params);

            return PaymentResult.builder()
                .success(true)
                .transactionId(charge.getId())
                .provider("Stripe")
                .message("Payment processed successfully")
                .build();

        } catch (Exception e) {
            log.error("Stripe payment failed", e);
            return PaymentResult.builder()
                .success(false)
                .provider("Stripe")
                .message("Payment failed: " + e.getMessage())
                .build();
        }
    }

    @Override
    public boolean isAvailable() {
        return apiKey != null && !apiKey.isEmpty();
    }

    @Override
    public String getProviderName() {
        return "Stripe";
    }

    @Override
    public BigDecimal getFeePercentage() {
        return FEE_PERCENTAGE;
    }
}
```

```java
// PayPalPaymentStrategy.java
package com.saas.payment.strategy;

import com.saas.model.PaymentRequest;
import com.saas.model.PaymentResult;
import com.saas.payment.PaymentStrategy;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import java.math.BigDecimal;

@Component
@Slf4j
public class PayPalPaymentStrategy implements PaymentStrategy {

    private static final BigDecimal FEE_PERCENTAGE = new BigDecimal("0.035"); // 3.5%

    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        log.info("Processing payment via PayPal: ${}", request.getAmount());

        // PayPal SDK integration would go here
        // Simplified for demonstration

        return PaymentResult.builder()
            .success(true)
            .transactionId("PAYPAL_" + System.currentTimeMillis())
            .provider("PayPal")
            .message("Payment processed successfully")
            .build();
    }

    @Override
    public boolean isAvailable() {
        return true; // Check PayPal configuration
    }

    @Override
    public String getProviderName() {
        return "PayPal";
    }

    @Override
    public BigDecimal getFeePercentage() {
        return FEE_PERCENTAGE;
    }
}
```

```java
// PaymentService.java - With fallback strategy
package com.saas.service;

import com.saas.model.PaymentRequest;
import com.saas.model.PaymentResult;
import com.saas.payment.PaymentStrategy;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class PaymentService {

    private final List<PaymentStrategy> paymentStrategies;

    /**
     * Process payment with automatic fallback
     */
    public PaymentResult processPayment(PaymentRequest request, String preferredProvider) {
        // Try preferred provider first
        PaymentStrategy preferredStrategy = findStrategy(preferredProvider);
        if (preferredStrategy != null && preferredStrategy.isAvailable()) {
            PaymentResult result = preferredStrategy.processPayment(request);
            if (result.isSuccess()) {
                return result;
            }
            log.warn("Preferred provider {} failed, trying fallback", preferredProvider);
        }

        // Fallback to other available providers
        for (PaymentStrategy strategy : paymentStrategies) {
            if (!strategy.getProviderName().equals(preferredProvider) &&
                strategy.isAvailable()) {

                log.info("Attempting payment with fallback provider: {}",
                    strategy.getProviderName());

                PaymentResult result = strategy.processPayment(request);
                if (result.isSuccess()) {
                    return result;
                }
            }
        }

        // All strategies failed
        return PaymentResult.builder()
            .success(false)
            .message("All payment providers failed")
            .build();
    }

    private PaymentStrategy findStrategy(String providerName) {
        return paymentStrategies.stream()
            .filter(s -> s.getProviderName().equalsIgnoreCase(providerName))
            .findFirst()
            .orElse(null);
    }
}
```

---

## 💻 Implementation in Node.js

### Scenario 1: Notification Delivery Strategies

```typescript
// notificationStrategy.ts - Strategy interface
export interface NotificationStrategy {
  send(recipient: string, subject: string, message: string): Promise<boolean>;
  getChannelName(): string;
  isAvailable(): boolean;
}
```

```typescript
// emailNotificationStrategy.ts
import { NotificationStrategy } from "./notificationStrategy";
import nodemailer from "nodemailer";

export class EmailNotificationStrategy implements NotificationStrategy {
  private transporter;

  constructor(config: any) {
    this.transporter = nodemailer.createTransport(config);
  }

  async send(
    recipient: string,
    subject: string,
    message: string
  ): Promise<boolean> {
    console.log(`Sending email to ${recipient}`);

    try {
      await this.transporter.sendMail({
        from: "noreply@saas.com",
        to: recipient,
        subject: subject,
        html: message,
      });

      console.log("Email sent successfully");
      return true;
    } catch (error) {
      console.error("Email send failed:", error);
      return false;
    }
  }

  getChannelName(): string {
    return "Email";
  }

  isAvailable(): boolean {
    return this.transporter != null;
  }
}
```

```typescript
// smsNotificationStrategy.ts
import { NotificationStrategy } from "./notificationStrategy";
import twilio from "twilio";

export class SmsNotificationStrategy implements NotificationStrategy {
  private client;
  private fromNumber: string;

  constructor(accountSid: string, authToken: string, fromNumber: string) {
    this.client = twilio(accountSid, authToken);
    this.fromNumber = fromNumber;
  }

  async send(
    recipient: string,
    subject: string,
    message: string
  ): Promise<boolean> {
    console.log(`Sending SMS to ${recipient}`);

    try {
      await this.client.messages.create({
        body: `${subject}: ${message}`,
        from: this.fromNumber,
        to: recipient,
      });

      console.log("SMS sent successfully");
      return true;
    } catch (error) {
      console.error("SMS send failed:", error);
      return false;
    }
  }

  getChannelName(): string {
    return "SMS";
  }

  isAvailable(): boolean {
    return this.client != null && this.fromNumber != null;
  }
}
```

```typescript
// pushNotificationStrategy.ts
import { NotificationStrategy } from "./notificationStrategy";

export class PushNotificationStrategy implements NotificationStrategy {
  async send(
    recipient: string,
    subject: string,
    message: string
  ): Promise<boolean> {
    console.log(`Sending push notification to ${recipient}`);

    // Firebase Cloud Messaging or similar integration
    try {
      // Simplified for demonstration
      console.log("Push notification sent successfully");
      return true;
    } catch (error) {
      console.error("Push notification failed:", error);
      return false;
    }
  }

  getChannelName(): string {
    return "Push";
  }

  isAvailable(): boolean {
    return true;
  }
}
```

```typescript
// slackNotificationStrategy.ts
import { NotificationStrategy } from "./notificationStrategy";
import { WebClient } from "@slack/web-api";

export class SlackNotificationStrategy implements NotificationStrategy {
  private client: WebClient;

  constructor(token: string) {
    this.client = new WebClient(token);
  }

  async send(
    recipient: string,
    subject: string,
    message: string
  ): Promise<boolean> {
    console.log(`Sending Slack message to ${recipient}`);

    try {
      await this.client.chat.postMessage({
        channel: recipient,
        text: `*${subject}*\n${message}`,
      });

      console.log("Slack message sent successfully");
      return true;
    } catch (error) {
      console.error("Slack send failed:", error);
      return false;
    }
  }

  getChannelName(): string {
    return "Slack";
  }

  isAvailable(): boolean {
    return this.client != null;
  }
}
```

```typescript
// notificationContext.ts - Context
import { NotificationStrategy } from "./notificationStrategy";

export class NotificationContext {
  private strategy: NotificationStrategy;

  constructor(strategy: NotificationStrategy) {
    this.strategy = strategy;
  }

  setStrategy(strategy: NotificationStrategy): void {
    console.log(
      `Switching notification channel to: ${strategy.getChannelName()}`
    );
    this.strategy = strategy;
  }

  async notify(
    recipient: string,
    subject: string,
    message: string
  ): Promise<boolean> {
    if (!this.strategy.isAvailable()) {
      throw new Error(`${this.strategy.getChannelName()} is not available`);
    }

    return this.strategy.send(recipient, subject, message);
  }
}
```

```typescript
// notificationService.ts - Multi-channel with preferences
import { NotificationStrategy } from "./notificationStrategy";
import { NotificationContext } from "./notificationContext";
import { EmailNotificationStrategy } from "./emailNotificationStrategy";
import { SmsNotificationStrategy } from "./smsNotificationStrategy";
import { PushNotificationStrategy } from "./pushNotificationStrategy";
import { SlackNotificationStrategy } from "./slackNotificationStrategy";

export class NotificationService {
  private strategies: Map<string, NotificationStrategy> = new Map();

  constructor() {
    // Register all available strategies
    this.strategies.set("email", new EmailNotificationStrategy(emailConfig));
    this.strategies.set("sms", new SmsNotificationStrategy(sid, token, from));
    this.strategies.set("push", new PushNotificationStrategy());
    this.strategies.set("slack", new SlackNotificationStrategy(slackToken));
  }

  async sendNotification(
    userId: string,
    subject: string,
    message: string,
    channels: string[]
  ): Promise<void> {
    // Get user preferences (from database)
    const userPreferences = await this.getUserPreferences(userId);

    // Send to all requested channels
    for (const channel of channels) {
      if (!userPreferences.enabledChannels.includes(channel)) {
        console.log(`Channel ${channel} disabled for user ${userId}`);
        continue;
      }

      const strategy = this.strategies.get(channel);
      if (!strategy) {
        console.warn(`Unknown channel: ${channel}`);
        continue;
      }

      if (!strategy.isAvailable()) {
        console.warn(`Channel ${channel} not available`);
        continue;
      }

      const context = new NotificationContext(strategy);
      const recipient = this.getRecipient(userId, channel, userPreferences);

      try {
        await context.notify(recipient, subject, message);
      } catch (error) {
        console.error(`Failed to send via ${channel}:`, error);
      }
    }
  }

  private async getUserPreferences(userId: string): Promise<any> {
    // Fetch from database
    return {
      enabledChannels: ["email", "push"],
      email: "user@example.com",
      phone: "+1234567890",
      slackId: "U12345678",
    };
  }

  private getRecipient(
    userId: string,
    channel: string,
    preferences: any
  ): string {
    switch (channel) {
      case "email":
        return preferences.email;
      case "sms":
        return preferences.phone;
      case "slack":
        return preferences.slackId;
      case "push":
        return userId;
      default:
        throw new Error(`Unknown channel: ${channel}`);
    }
  }
}
```

### Scenario 2: Export Strategies

```typescript
// exportStrategy.ts - Strategy interface
export interface ExportStrategy {
  export(data: any[]): Promise<Buffer>;
  getFileExtension(): string;
  getMimeType(): string;
}
```

```typescript
// csvExportStrategy.ts
import { ExportStrategy } from "./exportStrategy";
import { stringify } from "csv-stringify/sync";

export class CsvExportStrategy implements ExportStrategy {
  async export(data: any[]): Promise<Buffer> {
    console.log("Exporting data to CSV format");

    if (!data || data.length === 0) {
      return Buffer.from("");
    }

    // Convert to CSV
    const csv = stringify(data, {
      header: true,
      columns: Object.keys(data[0]),
    });

    return Buffer.from(csv);
  }

  getFileExtension(): string {
    return "csv";
  }

  getMimeType(): string {
    return "text/csv";
  }
}
```

```typescript
// excelExportStrategy.ts
import { ExportStrategy } from "./exportStrategy";
import ExcelJS from "exceljs";

export class ExcelExportStrategy implements ExportStrategy {
  async export(data: any[]): Promise<Buffer> {
    console.log("Exporting data to Excel format");

    const workbook = new ExcelJS.Workbook();
    const worksheet = workbook.addWorksheet("Data");

    if (!data || data.length === 0) {
      return Buffer.from("");
    }

    // Add headers
    const headers = Object.keys(data[0]);
    worksheet.addRow(headers);

    // Style headers
    worksheet.getRow(1).font = { bold: true };
    worksheet.getRow(1).fill = {
      type: "pattern",
      pattern: "solid",
      fgColor: { argb: "FFD3D3D3" },
    };

    // Add data rows
    data.forEach((row) => {
      const values = headers.map((header) => row[header]);
      worksheet.addRow(values);
    });

    // Auto-fit columns
    worksheet.columns.forEach((column) => {
      column.width = 15;
    });

    // Generate buffer
    return (await workbook.xlsx.writeBuffer()) as Buffer;
  }

  getFileExtension(): string {
    return "xlsx";
  }

  getMimeType(): string {
    return "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";
  }
}
```

```typescript
// jsonExportStrategy.ts
import { ExportStrategy } from "./exportStrategy";

export class JsonExportStrategy implements ExportStrategy {
  async export(data: any[]): Promise<Buffer> {
    console.log("Exporting data to JSON format");

    const json = JSON.stringify(data, null, 2);
    return Buffer.from(json);
  }

  getFileExtension(): string {
    return "json";
  }

  getMimeType(): string {
    return "application/json";
  }
}
```

```typescript
// pdfExportStrategy.ts
import { ExportStrategy } from "./exportStrategy";
import PDFDocument from "pdfkit";

export class PdfExportStrategy implements ExportStrategy {
  async export(data: any[]): Promise<Buffer> {
    console.log("Exporting data to PDF format");

    return new Promise((resolve, reject) => {
      const doc = new PDFDocument();
      const chunks: Buffer[] = [];

      doc.on("data", (chunk) => chunks.push(chunk));
      doc.on("end", () => resolve(Buffer.concat(chunks)));
      doc.on("error", reject);

      // Title
      doc.fontSize(20).text("Data Export", { align: "center" });
      doc.moveDown();

      if (!data || data.length === 0) {
        doc.fontSize(12).text("No data to export");
        doc.end();
        return;
      }

      // Table header
      const headers = Object.keys(data[0]);
      doc.fontSize(10).fillColor("blue");
      headers.forEach((header, index) => {
        doc.text(header, 50 + index * 100, doc.y, { width: 90 });
      });
      doc.moveDown();

      // Table rows
      doc.fillColor("black");
      data.forEach((row) => {
        const y = doc.y;
        headers.forEach((header, index) => {
          doc.text(String(row[header] || ""), 50 + index * 100, y, {
            width: 90,
            height: 20,
          });
        });
        doc.moveDown(0.5);
      });

      doc.end();
    });
  }

  getFileExtension(): string {
    return "pdf";
  }

  getMimeType(): string {
    return "application/pdf";
  }
}
```

```typescript
// xmlExportStrategy.ts
import { ExportStrategy } from "./exportStrategy";
import { create } from "xmlbuilder2";

export class XmlExportStrategy implements ExportStrategy {
  async export(data: any[]): Promise<Buffer> {
    console.log("Exporting data to XML format");

    const root = create({ version: "1.0", encoding: "UTF-8" }).ele("data");

    data.forEach((item) => {
      const itemElement = root.ele("item");
      Object.entries(item).forEach(([key, value]) => {
        itemElement.ele(key).txt(String(value));
      });
    });

    const xml = root.end({ prettyPrint: true });
    return Buffer.from(xml);
  }

  getFileExtension(): string {
    return "xml";
  }

  getMimeType(): string {
    return "application/xml";
  }
}
```

```typescript
// exportContext.ts - Context
import { ExportStrategy } from "./exportStrategy";

export class ExportContext {
  private strategy: ExportStrategy;

  constructor(strategy: ExportStrategy) {
    this.strategy = strategy;
  }

  setStrategy(strategy: ExportStrategy): void {
    console.log(`Switching export format to: ${strategy.getFileExtension()}`);
    this.strategy = strategy;
  }

  async exportData(data: any[]): Promise<{
    buffer: Buffer;
    filename: string;
    mimeType: string;
  }> {
    const buffer = await this.strategy.export(data);
    const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
    const filename = `export_${timestamp}.${this.strategy.getFileExtension()}`;
    const mimeType = this.strategy.getMimeType();

    return { buffer, filename, mimeType };
  }
}
```

```typescript
// exportService.ts - Service with factory
import { ExportStrategy } from "./exportStrategy";
import { ExportContext } from "./exportContext";
import { CsvExportStrategy } from "./csvExportStrategy";
import { ExcelExportStrategy } from "./excelExportStrategy";
import { JsonExportStrategy } from "./jsonExportStrategy";
import { PdfExportStrategy } from "./pdfExportStrategy";
import { XmlExportStrategy } from "./xmlExportStrategy";

export class ExportService {
  private strategies: Map<string, ExportStrategy> = new Map();

  constructor() {
    // Register all export strategies
    this.strategies.set("csv", new CsvExportStrategy());
    this.strategies.set("excel", new ExcelExportStrategy());
    this.strategies.set("json", new JsonExportStrategy());
    this.strategies.set("pdf", new PdfExportStrategy());
    this.strategies.set("xml", new XmlExportStrategy());
  }

  async exportData(
    data: any[],
    format: string
  ): Promise<{ buffer: Buffer; filename: string; mimeType: string }> {
    const strategy = this.strategies.get(format.toLowerCase());

    if (!strategy) {
      throw new Error(`Unsupported export format: ${format}`);
    }

    const context = new ExportContext(strategy);
    return context.exportData(data);
  }

  getSupportedFormats(): string[] {
    return Array.from(this.strategies.keys());
  }
}
```

```typescript
// exportController.ts - Express controller
import { Request, Response } from "express";
import { ExportService } from "./exportService";

export class ExportController {
  private exportService: ExportService;

  constructor() {
    this.exportService = new ExportService();
  }

  async export(req: Request, res: Response): Promise<void> {
    try {
      const { format = "csv" } = req.query;
      const data = req.body.data; // Array of objects to export

      if (!data || !Array.isArray(data)) {
        res.status(400).json({ error: "Data must be an array" });
        return;
      }

      const result = await this.exportService.exportData(
        data,
        format as string
      );

      // Set response headers
      res.setHeader("Content-Type", result.mimeType);
      res.setHeader(
        "Content-Disposition",
        `attachment; filename="${result.filename}"`
      );

      // Send file
      res.send(result.buffer);
    } catch (error) {
      console.error("Export failed:", error);
      res.status(500).json({
        error: "Export failed",
        message: error.message,
      });
    }
  }

  async getSupportedFormats(req: Request, res: Response): Promise<void> {
    const formats = this.exportService.getSupportedFormats();
    res.json({ formats });
  }
}
```

```typescript
// Usage example
import express from "express";
import { ExportController } from "./exportController";

const app = express();
app.use(express.json());

const exportController = new ExportController();

// Export endpoint
app.post("/api/export", (req, res) => exportController.export(req, res));

// Get supported formats
app.get("/api/export/formats", (req, res) =>
  exportController.getSupportedFormats(req, res)
);

// Example usage:
// POST /api/export?format=csv
// Body: { "data": [{"name": "John", "age": 30}, {"name": "Jane", "age": 25}] }
// Downloads: export_2024-01-15T10-30-00.csv

app.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

---

## 💻 Implementation in Go

### Scenario 1: Pricing Strategies

```go
// pricing_strategy.go - Strategy interface
package pricing

import "math/big"

// PricingStrategy defines the interface for pricing algorithms
type PricingStrategy interface {
    CalculatePrice(subscription *Subscription) *big.Float
    GetStrategyName() string
}

// Subscription model
type Subscription struct {
    Plan        string
    Seats       int
    ApiCalls    int
    StorageGB   int
    Quantity    int
    PricingModel string
}
```

```go
// tiered_pricing_strategy.go
package pricing

import (
    "fmt"
    "math/big"
)

type TieredPricingStrategy struct{}

func NewTieredPricingStrategy() *TieredPricingStrategy {
    return &TieredPricingStrategy{}
}

func (s *TieredPricingStrategy) CalculatePrice(subscription *Subscription) *big.Float {
    fmt.Printf("Calculating tiered price for plan: %s\n", subscription.Plan)

    var price *big.Float

    switch subscription.Plan {
    case "FREE":
        price = big.NewFloat(0.00)
    case "STARTER":
        price = big.NewFloat(29.99)
    case "PRO":
        price = big.NewFloat(99.99)
    case "ENTERPRISE":
        price = big.NewFloat(499.99)
    default:
        panic(fmt.Sprintf("Unknown plan: %s", subscription.Plan))
    }

    return price
}

func (s *TieredPricingStrategy) GetStrategyName() string {
    return "Tiered Pricing"
}
```

```go
// per_seat_pricing_strategy.go
package pricing

import (
    "fmt"
    "math/big"
)

type PerSeatPricingStrategy struct {
    pricePerSeat *big.Float
    minimumSeats int
}

func NewPerSeatPricingStrategy() *PerSeatPricingStrategy {
    return &PerSeatPricingStrategy{
        pricePerSeat: big.NewFloat(15.00),
        minimumSeats: 1,
    }
}

func (s *PerSeatPricingStrategy) CalculatePrice(subscription *Subscription) *big.Float {
    seats := subscription.Seats
    if seats < s.minimumSeats {
        seats = s.minimumSeats
    }

    price := new(big.Float).Mul(
        s.pricePerSeat,
        big.NewFloat(float64(seats)),
    )

    fmt.Printf("Calculating per-seat price: %d seats × $%.2f = $%.2f\n",
        seats, s.pricePerSeat, price)

    return price
}

func (s *PerSeatPricingStrategy) GetStrategyName() string {
    return "Per-Seat Pricing"
}
```

```go
// usage_based_pricing_strategy.go
package pricing

import (
    "fmt"
    "math/big"
)

type UsageBasedPricingStrategy struct {
    basePrice          *big.Float
    pricePerApiCall    *big.Float
    pricePerGBStorage  *big.Float
    freeApiCalls       int
    freeGBStorage      int
}

func NewUsageBasedPricingStrategy() *UsageBasedPricingStrategy {
    return &UsageBasedPricingStrategy{
        basePrice:         big.NewFloat(10.00),
        pricePerApiCall:   big.NewFloat(0.001),
        pricePerGBStorage: big.NewFloat(0.50),
        freeApiCalls:      1000,
        freeGBStorage:     5,
    }
}

func (s *UsageBasedPricingStrategy) CalculatePrice(subscription *Subscription) *big.Float {
    totalPrice := new(big.Float).Set(s.basePrice)

    // Calculate API usage cost
    billableApiCalls := subscription.ApiCalls - s.freeApiCalls
    if billableApiCalls > 0 {
        apiCost := new(big.Float).Mul(
            s.pricePerApiCall,
            big.NewFloat(float64(billableApiCalls)),
        )
        totalPrice.Add(totalPrice, apiCost)
    }

    // Calculate storage cost
    billableStorage := subscription.StorageGB - s.freeGBStorage
    if billableStorage > 0 {
        storageCost := new(big.Float).Mul(
            s.pricePerGBStorage,
            big.NewFloat(float64(billableStorage)),
        )
        totalPrice.Add(totalPrice, storageCost)
    }

    fmt.Printf("Usage-based pricing: Total=$%.2f\n", totalPrice)

    return totalPrice
}

func (s *UsageBasedPricingStrategy) GetStrategyName() string {
    return "Usage-Based Pricing"
}
```

```go
// volume_discount_pricing_strategy.go
package pricing

import (
    "fmt"
    "math/big"
)

type VolumeDiscountPricingStrategy struct {
    basePricePerUnit *big.Float
}

func NewVolumeDiscountPricingStrategy() *VolumeDiscountPricingStrategy {
    return &VolumeDiscountPricingStrategy{
        basePricePerUnit: big.NewFloat(10.00),
    }
}

func (s *VolumeDiscountPricingStrategy) CalculatePrice(subscription *Subscription) *big.Float {
    quantity := subscription.Quantity
    discount := s.calculateDiscount(quantity)

    basePrice := new(big.Float).Mul(
        s.basePricePerUnit,
        big.NewFloat(float64(quantity)),
    )

    discountAmount := new(big.Float).Mul(basePrice, discount)
    finalPrice := new(big.Float).Sub(basePrice, discountAmount)

    discountPercent := new(big.Float).Mul(discount, big.NewFloat(100))

    fmt.Printf("Volume discount pricing: Qty=%d, Base=$%.2f, Discount=%.0f%%, Final=$%.2f\n",
        quantity, basePrice, discountPercent, finalPrice)

    return finalPrice
}

func (s *VolumeDiscountPricingStrategy) calculateDiscount(quantity int) *big.Float {
    if quantity >= 100 {
        return big.NewFloat(0.20) // 20%
    } else if quantity >= 50 {
        return big.NewFloat(0.15) // 15%
    } else if quantity >= 20 {
        return big.NewFloat(0.10) // 10%
    } else if quantity >= 10 {
        return big.NewFloat(0.05) // 5%
    }
    return big.NewFloat(0.00)
}

func (s *VolumeDiscountPricingStrategy) GetStrategyName() string {
    return "Volume Discount Pricing"
}
```

```go
// pricing_context.go - Context
package pricing

import (
    "fmt"
    "math/big"
)

type PricingContext struct {
    strategy PricingStrategy
}

func NewPricingContext(strategy PricingStrategy) *PricingContext {
    return &PricingContext{
        strategy: strategy,
    }
}

func (c *PricingContext) SetStrategy(strategy PricingStrategy) {
    fmt.Printf("Switching pricing strategy from %s to %s\n",
        c.strategy.GetStrategyName(), strategy.GetStrategyName())
    c.strategy = strategy
}

func (c *PricingContext) CalculatePrice(subscription *Subscription) *big.Float {
    if c.strategy == nil {
        panic("Pricing strategy not set")
    }

    fmt.Printf("Using pricing strategy: %s\n", c.strategy.GetStrategyName())
    return c.strategy.CalculatePrice(subscription)
}
```

```go
// pricing_strategy_factory.go - Factory
package pricing

import (
    "fmt"
    "strings"
)

type PricingStrategyFactory struct {
    strategies map[string]PricingStrategy
}

func NewPricingStrategyFactory() *PricingStrategyFactory {
    factory := &PricingStrategyFactory{
        strategies: make(map[string]PricingStrategy),
    }

    // Register all available strategies
    factory.strategies["TIERED"] = NewTieredPricingStrategy()
    factory.strategies["PER_SEAT"] = NewPerSeatPricingStrategy()
    factory.strategies["USAGE_BASED"] = NewUsageBasedPricingStrategy()
    factory.strategies["VOLUME_DISCOUNT"] = NewVolumeDiscountPricingStrategy()

    return factory
}

func (f *PricingStrategyFactory) GetStrategy(strategyName string) (PricingStrategy, error) {
    strategy, ok := f.strategies[strings.ToUpper(strategyName)]
    if !ok {
        return nil, fmt.Errorf("unknown pricing strategy: %s", strategyName)
    }
    return strategy, nil
}

func (f *PricingStrategyFactory) GetAllStrategies() map[string]PricingStrategy {
    result := make(map[string]PricingStrategy)
    for k, v := range f.strategies {
        result[k] = v
    }
    return result
}
```

```go
// subscription_service.go - Service layer
package service

import (
    "fmt"
    "math/big"
    "myapp/pricing"
)

type SubscriptionService struct {
    strategyFactory *pricing.PricingStrategyFactory
}

func NewSubscriptionService() *SubscriptionService {
    return &SubscriptionService{
        strategyFactory: pricing.NewPricingStrategyFactory(),
    }
}

func (s *SubscriptionService) CalculateSubscriptionPrice(
    subscription *pricing.Subscription,
) (*big.Float, error) {
    // Get appropriate strategy based on subscription type
    strategy, err := s.determineStrategy(subscription)
    if err != nil {
        return nil, err
    }

    // Create context with strategy
    context := pricing.NewPricingContext(strategy)

    // Calculate price
    return context.CalculatePrice(subscription), nil
}

func (s *SubscriptionService) determineStrategy(
    subscription *pricing.Subscription,
) (pricing.PricingStrategy, error) {
    fmt.Printf("Determining pricing strategy for model: %s\n",
        subscription.PricingModel)
    return s.strategyFactory.GetStrategy(subscription.PricingModel)
}

func (s *SubscriptionService) CompareAllPricingStrategies(
    subscription *pricing.Subscription,
) map[string]*big.Float {
    comparison := make(map[string]*big.Float)

    for name, strategy := range s.strategyFactory.GetAllStrategies() {
        context := pricing.NewPricingContext(strategy)

        // Some strategies might not apply to all subscription types
        func() {
            defer func() {
                if r := recover(); r != nil {
                    fmt.Printf("Could not calculate price for strategy %s: %v\n",
                        name, r)
                }
            }()

            price := context.CalculatePrice(subscription)
            comparison[name] = price
        }()
    }

    return comparison
}
```

```go
// pricing_handler.go - HTTP handler
package handler

import (
    "encoding/json"
    "myapp/pricing"
    "myapp/service"
    "net/http"
)

type PricingHandler struct {
    subscriptionService *service.SubscriptionService
}

func NewPricingHandler() *PricingHandler {
    return &PricingHandler{
        subscriptionService: service.NewSubscriptionService(),
    }
}

func (h *PricingHandler) CalculatePrice(w http.ResponseWriter, r *http.Request) {
    var subscription pricing.Subscription

    if err := json.NewDecoder(r.Body).Decode(&subscription); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    price, err := h.subscriptionService.CalculateSubscriptionPrice(&subscription)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    priceFloat, _ := price.Float64()

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "price": priceFloat,
    })
}

func (h *PricingHandler) CompareStrategies(w http.ResponseWriter, r *http.Request) {
    var subscription pricing.Subscription

    if err := json.NewDecoder(r.Body).Decode(&subscription); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    comparison := h.subscriptionService.CompareAllPricingStrategies(&subscription)

    // Convert big.Float to float64 for JSON
    result := make(map[string]float64)
    for name, price := range comparison {
        priceFloat, _ := price.Float64()
        result[name] = priceFloat
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(result)
}
```

```go
// main.go - Setup and usage
package main

import (
    "fmt"
    "myapp/handler"
    "myapp/pricing"
    "myapp/service"
    "net/http"
)

func main() {
    // Example 1: Direct usage
    subscription := &pricing.Subscription{
        Plan:         "PRO",
        PricingModel: "TIERED",
    }

    service := service.NewSubscriptionService()
    price, err := service.CalculateSubscriptionPrice(subscription)
    if err != nil {
        panic(err)
    }

    fmt.Printf("Price: $%.2f\n", price)

    // Example 2: HTTP server
    pricingHandler := handler.NewPricingHandler()

    http.HandleFunc("/api/pricing/calculate", pricingHandler.CalculatePrice)
    http.HandleFunc("/api/pricing/compare", pricingHandler.CompareStrategies)

    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

### Scenario 2: Authentication Strategies

```go
// auth_strategy.go - Strategy interface
package auth

type Credentials struct {
    Username string
    Password string
    Token    string
    Provider string
}

type AuthResult struct {
    Success bool
    UserID  string
    Message string
}

type AuthStrategy interface {
    Authenticate(credentials Credentials) (*AuthResult, error)
    GetProviderName() string
}
```

```go
// password_auth_strategy.go
package auth

import (
    "errors"
    "fmt"
    "golang.org/x/crypto/bcrypt"
)

type PasswordAuthStrategy struct {
    userRepository UserRepository
}

func NewPasswordAuthStrategy(repo UserRepository) *PasswordAuthStrategy {
    return &PasswordAuthStrategy{
        userRepository: repo,
    }
}

func (s *PasswordAuthStrategy) Authenticate(credentials Credentials) (*AuthResult, error) {
    fmt.Println("Authenticating with password")

    if credentials.Username == "" || credentials.Password == "" {
        return &AuthResult{
            Success: false,
            Message: "Username and password required",
        }, nil
    }

    user, err := s.userRepository.FindByUsername(credentials.Username)
    if err != nil {
        return &AuthResult{
            Success: false,
            Message: "Invalid credentials",
        }, nil
    }

    // Compare password
    err = bcrypt.CompareHashAndPassword(
        []byte(user.PasswordHash),
        []byte(credentials.Password),
    )

    if err != nil {
        return &AuthResult{
            Success: false,
            Message: "Invalid credentials",
        }, nil
    }

    return &AuthResult{
        Success: true,
        UserID:  user.ID,
        Message: "Authentication successful",
    }, nil
}

func (s *PasswordAuthStrategy) GetProviderName() string {
    return "Password"
}
```

```go
// oauth_auth_strategy.go
package auth

import (
    "fmt"
)

type OAuthAuthStrategy struct {
    provider      string
    clientID      string
    clientSecret  string
    oauthClient   OAuthClient
}

func NewOAuthAuthStrategy(
    provider string,
    clientID string,
    clientSecret string,
) *OAuthAuthStrategy {
    return &OAuthAuthStrategy{
        provider:     provider,
        clientID:     clientID,
        clientSecret: clientSecret,
        oauthClient:  NewOAuthClient(provider, clientID, clientSecret),
    }
}

func (s *OAuthAuthStrategy) Authenticate(credentials Credentials) (*AuthResult, error) {
    fmt.Printf("Authenticating with OAuth provider: %s\n", s.provider)

    if credentials.Token == "" {
        return &AuthResult{
            Success: false,
            Message: "OAuth token required",
        }, nil
    }

    // Verify token with OAuth provider
    userInfo, err := s.oauthClient.VerifyToken(credentials.Token)
    if err != nil {
        return &AuthResult{
            Success: false,
            Message: "Invalid OAuth token",
        }, err
    }

    return &AuthResult{
        Success: true,
        UserID:  userInfo.ID,
        Message: fmt.Sprintf("Authenticated via %s", s.provider),
    }, nil
}

func (s *OAuthAuthStrategy) GetProviderName() string {
    return fmt.Sprintf("OAuth-%s", s.provider)
}
```

```go
// magic_link_auth_strategy.go
package auth

import (
    "errors"
    "fmt"
    "time"
)

type MagicLinkAuthStrategy struct {
    tokenStore TokenStore
}

func NewMagicLinkAuthStrategy(store TokenStore) *MagicLinkAuthStrategy {
    return &MagicLinkAuthStrategy{
        tokenStore: store,
    }
}

func (s *MagicLinkAuthStrategy) Authenticate(credentials Credentials) (*AuthResult, error) {
    fmt.Println("Authenticating with magic link")

    if credentials.Token == "" {
        return &AuthResult{
            Success: false,
            Message: "Magic link token required",
        }, nil
    }

    // Verify token
    tokenData, err := s.tokenStore.GetToken(credentials.Token)
    if err != nil {
        return &AuthResult{
            Success: false,
            Message: "Invalid or expired magic link",
        }, nil
    }

    // Check expiration (typically 15 minutes)
    if time.Now().After(tokenData.ExpiresAt) {
        return &AuthResult{
            Success: false,
            Message: "Magic link expired",
        }, nil
    }

    // Invalidate token (one-time use)
    s.tokenStore.DeleteToken(credentials.Token)

    return &AuthResult{
        Success: true,
        UserID:  tokenData.UserID,
        Message: "Authenticated via magic link",
    }, nil
}

func (s *MagicLinkAuthStrategy) GetProviderName() string {
    return "MagicLink"
}
```

```go
// auth_context.go - Context
package auth

import "fmt"

type AuthContext struct {
    strategy AuthStrategy
}

func NewAuthContext(strategy AuthStrategy) *AuthContext {
    return &AuthContext{
        strategy: strategy,
    }
}

func (c *AuthContext) SetStrategy(strategy AuthStrategy) {
    fmt.Printf("Switching auth strategy to: %s\n", strategy.GetProviderName())
    c.strategy = strategy
}

func (c *AuthContext) Authenticate(credentials Credentials) (*AuthResult, error) {
    if c.strategy == nil {
        return nil, fmt.Errorf("authentication strategy not set")
    }

    fmt.Printf("Authenticating with: %s\n", c.strategy.GetProviderName())
    return c.strategy.Authenticate(credentials)
}
```

```go
// auth_service.go - Service with strategy selection
package service

import (
    "fmt"
    "myapp/auth"
)

type AuthService struct {
    strategies map[string]auth.AuthStrategy
}

func NewAuthService(
    userRepo auth.UserRepository,
    tokenStore auth.TokenStore,
    oauthClient auth.OAuthClient,
) *AuthService {
    service := &AuthService{
        strategies: make(map[string]auth.AuthStrategy),
    }

    // Register all authentication strategies
    service.strategies["password"] = auth.NewPasswordAuthStrategy(userRepo)
    service.strategies["google"] = auth.NewOAuthAuthStrategy("google", "client-id", "secret")
    service.strategies["github"] = auth.NewOAuthAuthStrategy("github", "client-id", "secret")
    service.strategies["magic-link"] = auth.NewMagicLinkAuthStrategy(tokenStore)

    return service
}

func (s *AuthService) Authenticate(
    credentials auth.Credentials,
) (*auth.AuthResult, error) {
    strategy, ok := s.strategies[credentials.Provider]
    if !ok {
        return nil, fmt.Errorf("unsupported authentication provider: %s",
            credentials.Provider)
    }

    context := auth.NewAuthContext(strategy)
    return context.Authenticate(credentials)
}

func (s *AuthService) GetSupportedProviders() []string {
    providers := make([]string, 0, len(s.strategies))
    for provider := range s.strategies {
        providers = append(providers, provider)
    }
    return providers
}
```

```go
// auth_handler.go - HTTP handler
package handler

import (
    "encoding/json"
    "myapp/auth"
    "myapp/service"
    "net/http"
)

type AuthHandler struct {
    authService *service.AuthService
}

func NewAuthHandler(authService *service.AuthService) *AuthHandler {
    return &AuthHandler{
        authService: authService,
    }
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var credentials auth.Credentials

    if err := json.NewDecoder(r.Body).Decode(&credentials); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    result, err := h.authService.Authenticate(credentials)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    if !result.Success {
        w.WriteHeader(http.StatusUnauthorized)
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(result)
}

func (h *AuthHandler) GetProviders(w http.ResponseWriter, r *http.Request) {
    providers := h.authService.GetSupportedProviders()

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "providers": providers,
    })
}
```

---

## ✅ Best Practices

### 1. **Keep Strategies Focused and Single-Purpose**

Each strategy should implement ONE algorithm or behavior.

```java
// ✅ GOOD - Single purpose
public class StripePaymentStrategy implements PaymentStrategy { ... }
public class PayPalPaymentStrategy implements PaymentStrategy { ... }

// ❌ BAD - Multiple responsibilities
public class PaymentStrategy {
    public void processStripe() { ... }
    public void processPayPal() { ... }
    public void processCrypto() { ... }
}
```

### 2. **Use Strategy Factory for Complex Selection Logic**

Don't let clients choose strategies directly if selection is complex.

```typescript
// ✅ GOOD - Factory handles selection
class PricingStrategyFactory {
  getStrategy(subscription: Subscription): PricingStrategy {
    if (subscription.isEnterprise) {
      return new CustomPricingStrategy();
    } else if (subscription.hasVolume) {
      return new VolumeDiscountStrategy();
    }
    return new StandardPricingStrategy();
  }
}

// ❌ BAD - Client must know selection logic
const strategy = subscription.isEnterprise
  ? new CustomPricingStrategy()
  : subscription.hasVolume
  ? new VolumeDiscountStrategy()
  : new StandardPricingStrategy();
```

### 3. **Make Strategies Stateless When Possible**

Prefer stateless strategies that can be reused.

```go
// ✅ GOOD - Stateless, reusable
type TieredPricingStrategy struct {
    // Configuration only, no mutable state
}

// ❌ BAD - Stateful
type TieredPricingStrategy struct {
    lastCalculatedPrice float64  // Mutable state!
    calculationCount    int
}
```

### 4. **Provide Default/Fallback Strategy**

Always have a safe default when strategy selection fails.

```java
// ✅ GOOD - Default fallback
public PaymentStrategy getPaymentStrategy(String provider) {
    PaymentStrategy strategy = strategies.get(provider);
    if (strategy == null || !strategy.isAvailable()) {
        log.warn("Provider {} unavailable, using default", provider);
        return defaultStrategy;
    }
    return strategy;
}
```

### 5. **Document When to Use Each Strategy**

Make it clear which strategy is appropriate for which scenario.

```typescript
/**
 * Pricing Strategies:
 *
 * - TieredPricing: Use for simple, fixed-price plans (SaaS standard)
 * - PerSeatPricing: Use when pricing per user/seat (team collaboration tools)
 * - UsageBasedPricing: Use for consumption-based pricing (APIs, storage)
 * - VolumePricing: Use for bulk discounts (wholesale, enterprise)
 *
 * Choose based on:
 * 1. Business model
 * 2. Customer expectations
 * 3. Revenue predictability needs
 */
interface PricingStrategy { ... }
```

### 6. **Use Dependency Injection for Strategies**

Let DI container manage strategy lifecycle.

```java
// ✅ GOOD - Spring manages strategies
@Configuration
public class StrategyConfig {

    @Bean
    public Map<String, PaymentStrategy> paymentStrategies(
            StripePaymentStrategy stripe,
            PayPalPaymentStrategy paypal) {
        Map<String, PaymentStrategy> strategies = new HashMap<>();
        strategies.put("stripe", stripe);
        strategies.put("paypal", paypal);
        return strategies;
    }
}

@Service
public class PaymentService {
    private final Map<String, PaymentStrategy> strategies;

    @Autowired
    public PaymentService(Map<String, PaymentStrategy> strategies) {
        this.strategies = strategies;
    }
}
```

### 7. **Handle Strategy Failures Gracefully**

Don't let one failing strategy break the entire system.

```typescript
// ✅ GOOD - Graceful degradation
async sendNotification(channels: string[]): Promise<void> {
  const results = await Promise.allSettled(
    channels.map(channel => this.sendViaChannel(channel))
  );

  results.forEach((result, index) => {
    if (result.status === 'rejected') {
      console.error(`Channel ${channels[index]} failed:`, result.reason);
      // Log to monitoring, but don't fail entire operation
    }
  });
}
```

### 8. **Consider Strategy Composition**

Sometimes combining strategies is more powerful than choosing one.

```java
// ✅ GOOD - Composite strategy
public class CompositeNotificationStrategy implements NotificationStrategy {
    private final List<NotificationStrategy> strategies;

    @Override
    public void send(Notification notification) {
        // Send via all channels
        strategies.forEach(strategy -> {
            try {
                strategy.send(notification);
            } catch (Exception e) {
                log.error("Strategy failed", e);
            }
        });
    }
}
```

### 9. **Validate Strategy Input**

Each strategy should validate its input requirements.

```go
// ✅ GOOD - Input validation
func (s *PerSeatPricingStrategy) CalculatePrice(sub *Subscription) (*big.Float, error) {
    if sub.Seats <= 0 {
        return nil, errors.New("seats must be positive")
    }

    // Calculate price...
}
```

### 10. **Cache Strategy Results When Appropriate**

If calculation is expensive and data doesn't change often.

```typescript
// ✅ GOOD - Caching decorator for strategy
class CachedPricingStrategy implements PricingStrategy {
  private cache = new Map<string, number>();

  constructor(private strategy: PricingStrategy) {}

  calculatePrice(subscription: Subscription): number {
    const key = this.getCacheKey(subscription);

    if (this.cache.has(key)) {
      return this.cache.get(key)!;
    }

    const price = this.strategy.calculatePrice(subscription);
    this.cache.set(key, price);

    return price;
  }

  private getCacheKey(subscription: Subscription): string {
    return `${subscription.plan}-${subscription.seats}`;
  }
}
```

---

## 🚫 Common Pitfalls

### 1. **Over-Engineering Simple Logic**

**Problem**: Using Strategy for simple if/else.

```typescript
// ❌ PITFALL - Unnecessary complexity
interface DiscountStrategy {
  calculate(price: number): number;
}

class SummerDiscountStrategy implements DiscountStrategy { ... }
class WinterDiscountStrategy implements DiscountStrategy { ... }

// ✅ SOLUTION - Simple enough without Strategy
function calculateDiscount(price: number, season: string): number {
  return season === 'summer' ? price * 0.9 : price * 0.95;
}
```

### 2. **Strategies with Too Much Shared Code**

**Problem**: Duplicating logic across strategies.

```java
// ❌ PITFALL - Code duplication
public class StripeStrategy implements PaymentStrategy {
    public PaymentResult process(Payment payment) {
        validatePayment(payment); // Duplicated
        logPayment(payment);       // Duplicated
        // Stripe-specific logic
    }
}

public class PayPalStrategy implements PaymentStrategy {
    public PaymentResult process(Payment payment) {
        validatePayment(payment); // Duplicated
        logPayment(payment);       // Duplicated
        // PayPal-specific logic
    }
}

// ✅ SOLUTION - Template Method or abstract base class
public abstract class AbstractPaymentStrategy implements PaymentStrategy {
    public final PaymentResult process(Payment payment) {
        validatePayment(payment);  // Shared
        logPayment(payment);        // Shared
        return doProcess(payment);  // Strategy-specific
    }

    protected abstract PaymentResult doProcess(Payment payment);
}
```

### 3. **Not Handling Strategy Unavailability**

**Problem**: Assuming strategy is always available.

```go
// ❌ PITFALL - No availability check
func (s *Service) Process() error {
    strategy := s.getStrategy()
    return strategy.Execute() // What if strategy is nil or unavailable?
}

// ✅ SOLUTION - Check availability
func (s *Service) Process() error {
    strategy := s.getStrategy()
    if strategy == nil {
        return errors.New("no strategy available")
    }

    if !strategy.IsAvailable() {
        return errors.New("strategy not available")
    }

    return strategy.Execute()
}
```

### 4. **Tight Coupling Between Context and Concrete Strategies**

**Problem**: Context knows about concrete strategy implementations.

```typescript
// ❌ PITFALL - Tight coupling
class PaymentContext {
  processPayment(amount: number, provider: string) {
    if (provider === "stripe") {
      const stripe = new StripePaymentStrategy(); // Direct coupling!
      return stripe.process(amount);
    }
    // More if/else...
  }
}

// ✅ SOLUTION - Inject strategy
class PaymentContext {
  constructor(private strategy: PaymentStrategy) {}

  processPayment(amount: number) {
    return this.strategy.process(amount);
  }
}
```

### 5. **Forgetting to Update Strategy Factory**

**Problem**: Adding new strategy but not registering it.

```java
// ❌ PITFALL - New strategy exists but not registered
public class NewAwesomeStrategy implements PricingStrategy { ... }

// Factory still doesn't know about it!
public class StrategyFactory {
    public StrategyFactory() {
        // NewAwesomeStrategy not added here
    }
}

// ✅ SOLUTION - Auto-registration or clear process
@Component // Spring auto-detects
public class NewAwesomeStrategy implements PricingStrategy { ... }

// Or use clear registration process
public class StrategyFactory {
    public StrategyFactory(List<PricingStrategy> strategies) {
        // Auto-inject all strategies
        strategies.forEach(s -> register(s));
    }
}
```

### 6. **Not Considering Performance Implications**

**Problem**: Creating new strategy instances repeatedly.

```typescript
// ❌ PITFALL - Creating strategy on every call
async function processPayment(amount: number, provider: string) {
  const strategy = createPaymentStrategy(provider); // New instance every time!
  return strategy.process(amount);
}

// ✅ SOLUTION - Reuse stateless strategies
class PaymentService {
  private strategies = new Map<string, PaymentStrategy>([
    ["stripe", new StripeStrategy()], // Created once
    ["paypal", new PayPalStrategy()],
  ]);

  async processPayment(amount: number, provider: string) {
    const strategy = this.strategies.get(provider);
    return strategy.process(amount);
  }
}
```

### 7. **Missing Strategy Validation**

**Problem**: No validation that strategy can handle the input.

```go
// ❌ PITFALL - No validation
func (s *UsageBasedPricing) Calculate(sub *Subscription) float64 {
    // Assumes ApiCalls and StorageGB exist
    return basePrice + (sub.ApiCalls * apiRate) + (sub.StorageGB * storageRate)
}

// ✅ SOLUTION - Validate compatibility
func (s *UsageBasedPricing) Calculate(sub *Subscription) (float64, error) {
    if sub.PricingModel != "usage_based" {
        return 0, errors.New("subscription not compatible with usage-based pricing")
    }

    if sub.ApiCalls < 0 || sub.StorageGB < 0 {
        return 0, errors.New("invalid usage metrics")
    }

    return s.calculate(sub), nil
}
```

### 8. **Strategy Explosion**

**Problem**: Too many similar strategies.

```java
// ❌ PITFALL - 50+ nearly identical strategies
public class Tier1PricingStrategy implements PricingStrategy { ... }
public class Tier2PricingStrategy implements PricingStrategy { ... }
// ... 48 more tiers

// ✅ SOLUTION - Parameterized strategy
public class TieredPricingStrategy implements PricingStrategy {
    private final Map<String, BigDecimal> tierPrices;

    public TieredPricingStrategy(Map<String, BigDecimal> tierPrices) {
        this.tierPrices = tierPrices;
    }

    @Override
    public BigDecimal calculate(Subscription sub) {
        return tierPrices.get(sub.getTier());
    }
}
```

### 9. **Not Considering Strategy Ordering**

**Problem**: Order matters but not documented or enforced.

```typescript
// ❌ PITFALL - Order-dependent without documentation
strategies.forEach((strategy) => strategy.apply(data));

// ✅ SOLUTION - Document and enforce order
/**
 * Strategies applied in order:
 * 1. Validation - ensures data is valid
 * 2. Transformation - modifies data
 * 3. Enrichment - adds additional data
 * 4. Persistence - saves data
 */
class OrderedStrategyPipeline {
  private strategies: Strategy[] = [];

  addStrategy(strategy: Strategy, order: number) {
    this.strategies[order] = strategy;
  }

  execute(data: any) {
    return this.strategies
      .filter((s) => s != null)
      .reduce((result, strategy) => strategy.apply(result), data);
  }
}
```

### 10. **Ignoring Strategy Context Requirements**

**Problem**: Strategy needs more context than provided.

```java
// ❌ PITFALL - Insufficient context
public interface PricingStrategy {
    BigDecimal calculate(int quantity); // Not enough info!
}

// Strategy needs user's country for tax, but can't access it

// ✅ SOLUTION - Rich context object
public class PricingContext {
    private final Subscription subscription;
    private final User user;
    private final Currency currency;
    private final TaxInfo taxInfo;
    // All necessary context
}

public interface PricingStrategy {
    BigDecimal calculate(PricingContext context);
}
```

---

## 🎯 Real-World SaaS Use Cases

### 1. **Subscription Pricing Models** ⭐⭐⭐

Different pricing for different market segments:

- **Tiered**: Fixed prices (Free/$10/$50/month)
- **Per-Seat**: Team collaboration tools ($15/user/month)
- **Usage-Based**: API platforms ($0.001/request)
- **Hybrid**: Base + usage (Twilio model)

### 2. **Multi-Gateway Payment Processing** ⭐⭐⭐

```typescript
// Primary: Stripe (2.9% + $0.30)
// Fallback: PayPal (3.5%)
// International: Adyen (varies by region)
// Crypto: Coinbase Commerce

const result = await paymentService.processWithFallback(payment, [
  "stripe",
  "paypal",
  "adyen",
]);
```

### 3. **Multi-Channel Notifications** ⭐⭐⭐

Send based on urgency and user preferences:

- **Critical**: SMS + Push + Email
- **Important**: Push + Email
- **Info**: Email only
- **Marketing**: Email (if opted in)

### 4. **Regional Compliance Strategies** ⭐⭐

Different validation/processing for different regions:

- **GDPR** (EU): Strict data handling
- **CCPA** (California): Privacy rights
- **LGPD** (Brazil): Consent management
- **PIPEDA** (Canada): Privacy protection

### 5. **A/B Testing Variations** ⭐⭐

```java
PricingStrategy strategy = abTestService.getVariant(userId)
    ? new ExperimentalPricingStrategy()
    : new StandardPricingStrategy();
```

### 6. **Feature Rollout** ⭐⭐

Gradually roll out new algorithms:

- **Canary**: 5% of users get new strategy
- **Beta**: Opted-in users
- **Full**: Everyone

### 7. **Tenant-Specific Customization** ⭐⭐⭐

Enterprise customers get custom strategies:

```typescript
const strategy = tenant.isEnterprise
  ? new CustomPricingStrategy(tenant.contract)
  : new StandardPricingStrategy();
```

---

## 📚 Resources for Deeper Understanding

### Books

1. **"Design Patterns: Elements of Reusable Object-Oriented Software"** - Gang of Four

   - Chapter: Behavioral Patterns → Strategy

2. **"Head First Design Patterns"** - Freeman & Freeman

   - Chapter 1: Strategy Pattern (with duck simulation example)

3. **"Refactoring: Improving the Design of Existing Code"** - Martin Fowler
   - Replace Conditional with Polymorphism → Strategy Pattern

### Online Resources

1. **Refactoring Guru - Strategy Pattern**

   - https://refactoring.guru/design-patterns/strategy

2. **Spring Framework - Strategy Pattern**

   - Used in RestTemplate, TransactionManager

3. **Effective Java (3rd Edition)** - Joshua Bloch
   - Item 4: Enforce noninstantiability with a private constructor
   - Item 21: Design interfaces for posterity

### Articles

1. **"Strategy Pattern in Real-World Applications"** - Baeldung

   - https://www.baeldung.com/java-strategy-pattern

2. **"When to Use Strategy Pattern"** - Martin Fowler
   - https://martinfowler.com/bliki/

---

## 🏋️ Practical Exercise

### Exercise: Build a Multi-Tenant Pricing System

**Scenario**: You're building a SaaS platform where different tenants can have completely different pricing models.

**Requirements**:

1. **Tenant A** (Startup): Simple tiered pricing ($0/$29/$99/month)
2. **Tenant B** (Enterprise): Per-seat with volume discount
3. **Tenant C** (API Platform): Usage-based with overage charges
4. **Tenant D** (Marketplace): Percentage of transaction value

**Your Task**:
Implement in your language of choice:

- Strategy interface for pricing
- 4 concrete pricing strategies
- Context that selects strategy based on tenant
- Service layer that handles price calculation
- API endpoint to calculate and compare prices

**Bonus Challenges**:

- Add caching for calculated prices
- Implement A/B testing (10% of users get experimental pricing)
- Add currency conversion support
- Create pricing comparison dashboard
- Implement proration for mid-cycle upgrades

**Acceptance Criteria**:

- Each strategy is independent and testable
- Adding a new strategy doesn't require changing existing code
- Strategies are configured via database/config, not hardcoded
- System handles strategy failures gracefully
- Performance: Calculate price for 10,000 subscriptions in <1 second

---

## 📝 Summary

**Strategy Pattern**:

- ✅ Defines family of interchangeable algorithms
- ✅ Eliminates conditional logic
- ✅ Makes algorithms easy to test independently
- ✅ Open/Closed Principle - add strategies without modifying context
- ✅ Perfect for SaaS pricing, payments, notifications
- ❌ Can increase number of classes
- ❌ Clients must understand differences between strategies

**Key Concepts**:

1. **Strategy Interface**: Common contract for all algorithms
2. **Concrete Strategies**: Specific implementations
3. **Context**: Uses a strategy to do work
4. **Client**: Chooses which strategy to use

**When to Use**:

- Multiple ways of doing the same thing
- Need to switch algorithms at runtime
- Want to eliminate complex conditionals
- Different behaviors for different scenarios

**Remember**: Use Strategy when you have multiple algorithms for the same task and want to make them interchangeable. It's one of the most useful patterns for SaaS applications!

---

## 🔜 What's Next?

Next pattern: **Observer Pattern** - Essential for event-driven systems, notifications, and real-time updates

Ready to continue? 🚀
