# Module 2.5: Structural Patterns - Adapter Pattern

**Pattern 8: Adapter Pattern**\
**Duration:** 1 day\
**Prerequisites:** Repository patterns completed

***

## Pattern 8: Adapter Pattern

### What is the Adapter Pattern?

The **Adapter Pattern** allows incompatible interfaces to work together. It acts as a bridge between two incompatible interfaces.

**Real-World Analogy:**

* Power adapter for different countries
* USB-C to USB-A adapter
* HDMI to VGA adapter

**In Software:** Convert one interface into another that clients expect, without modifying the original code.

***

### The Problem

You need to integrate with external services that have different APIs:

```typescript
// ❌ BAD: Tightly coupled to specific service
class PaymentService {
  async processPayment(amount: number, customerId: string) {
    // Directly using Stripe API
    const stripe = require('stripe')(process.env.STRIPE_KEY);
    
    const charge = await stripe.charges.create({
      amount: amount * 100, // Stripe uses cents
      currency: 'usd',
      customer: customerId,
      description: 'Payment'
    });
    
    return charge;
  }
}

// Problems:
// 1. Can't switch to PayPal without rewriting code
// 2. Hard to test (requires real Stripe account)
// 3. Stripe-specific logic scattered everywhere
// 4. Can't support multiple payment providers
```

### The Solution: Adapter Pattern

Create a common interface and adapt each external service to it:

```
Your Application
       ↓
Payment Gateway Interface (your interface)
       ↓
   ┌────┴────┬────────┬────────┐
   ↓         ↓        ↓        ↓
Stripe   PayPal   Square   Coinbase
Adapter  Adapter  Adapter   Adapter
   ↓         ↓        ↓        ↓
Stripe   PayPal   Square   Coinbase
  API      API      API      API
```

***

## Implementation: Payment Gateway Adapters

### Step 1: Define Common Interface

```typescript
// domain/gateways/IPaymentGateway.ts

export interface PaymentMethod {
  type: 'card' | 'bank_account' | 'wallet';
  token: string;
  last4?: string;
  brand?: string;
}

export interface PaymentResult {
  success: boolean;
  transactionId: string;
  amount: number;
  currency: string;
  status: 'succeeded' | 'pending' | 'failed';
  errorMessage?: string;
  metadata?: Record<string, any>;
}

export interface RefundResult {
  success: boolean;
  refundId: string;
  amount: number;
  status: 'succeeded' | 'pending' | 'failed';
  errorMessage?: string;
}

export interface Customer {
  id: string;
  email: string;
  name: string;
}

export interface IPaymentGateway {
  // Customer management
  createCustomer(customer: Omit<Customer, 'id'>): Promise<Customer>;
  getCustomer(customerId: string): Promise<Customer | null>;
  deleteCustomer(customerId: string): Promise<boolean>;
  
  // Payment methods
  attachPaymentMethod(customerId: string, method: PaymentMethod): Promise<string>;
  detachPaymentMethod(paymentMethodId: string): Promise<boolean>;
  listPaymentMethods(customerId: string): Promise<PaymentMethod[]>;
  
  // Charges
  charge(params: {
    amount: number;
    currency: string;
    customerId: string;
    paymentMethodId: string;
    description?: string;
    metadata?: Record<string, any>;
  }): Promise<PaymentResult>;
  
  // Refunds
  refund(params: {
    transactionId: string;
    amount?: number;
    reason?: string;
  }): Promise<RefundResult>;
  
  // Webhooks
  verifyWebhook(payload: string, signature: string): boolean;
  parseWebhookEvent(payload: string): any;
}
```

### Step 2: Implement Stripe Adapter

```typescript
// infrastructure/gateways/StripePaymentAdapter.ts
import Stripe from 'stripe';
import {
  IPaymentGateway,
  PaymentMethod,
  PaymentResult,
  RefundResult,
  Customer
} from '../../domain/gateways/IPaymentGateway';

export class StripePaymentAdapter implements IPaymentGateway {
  private stripe: Stripe;
  private webhookSecret: string;
  
  constructor(apiKey: string, webhookSecret: string) {
    this.stripe = new Stripe(apiKey, {
      apiVersion: '2023-10-16'
    });
    this.webhookSecret = webhookSecret;
  }
  
  async createCustomer(customer: Omit<Customer, 'id'>): Promise<Customer> {
    const stripeCustomer = await this.stripe.customers.create({
      email: customer.email,
      name: customer.name,
      metadata: {
        source: 'saas-app'
      }
    });
    
    return {
      id: stripeCustomer.id,
      email: stripeCustomer.email!,
      name: stripeCustomer.name!
    };
  }
  
  async getCustomer(customerId: string): Promise<Customer | null> {
    try {
      const stripeCustomer = await this.stripe.customers.retrieve(customerId);
      
      if (stripeCustomer.deleted) {
        return null;
      }
      
      return {
        id: stripeCustomer.id,
        email: stripeCustomer.email!,
        name: stripeCustomer.name!
      };
    } catch (error) {
      if (error.code === 'resource_missing') {
        return null;
      }
      throw error;
    }
  }
  
  async deleteCustomer(customerId: string): Promise<boolean> {
    try {
      await this.stripe.customers.del(customerId);
      return true;
    } catch (error) {
      return false;
    }
  }
  
  async attachPaymentMethod(
    customerId: string,
    method: PaymentMethod
  ): Promise<string> {
    // Attach payment method to customer
    await this.stripe.paymentMethods.attach(method.token, {
      customer: customerId
    });
    
    return method.token;
  }
  
  async detachPaymentMethod(paymentMethodId: string): Promise<boolean> {
    try {
      await this.stripe.paymentMethods.detach(paymentMethodId);
      return true;
    } catch (error) {
      return false;
    }
  }
  
  async listPaymentMethods(customerId: string): Promise<PaymentMethod[]> {
    const paymentMethods = await this.stripe.paymentMethods.list({
      customer: customerId,
      type: 'card'
    });
    
    return paymentMethods.data.map(pm => ({
      type: 'card' as const,
      token: pm.id,
      last4: pm.card?.last4,
      brand: pm.card?.brand
    }));
  }
  
  async charge(params: {
    amount: number;
    currency: string;
    customerId: string;
    paymentMethodId: string;
    description?: string;
    metadata?: Record<string, any>;
  }): Promise<PaymentResult> {
    try {
      // Stripe uses cents, so convert
      const amountInCents = Math.round(params.amount * 100);
      
      const paymentIntent = await this.stripe.paymentIntents.create({
        amount: amountInCents,
        currency: params.currency.toLowerCase(),
        customer: params.customerId,
        payment_method: params.paymentMethodId,
        confirm: true,
        description: params.description,
        metadata: params.metadata,
        automatic_payment_methods: {
          enabled: true,
          allow_redirects: 'never'
        }
      });
      
      return {
        success: paymentIntent.status === 'succeeded',
        transactionId: paymentIntent.id,
        amount: params.amount,
        currency: params.currency,
        status: this.mapStripeStatus(paymentIntent.status),
        metadata: paymentIntent.metadata
      };
      
    } catch (error: any) {
      return {
        success: false,
        transactionId: '',
        amount: params.amount,
        currency: params.currency,
        status: 'failed',
        errorMessage: error.message
      };
    }
  }
  
  async refund(params: {
    transactionId: string;
    amount?: number;
    reason?: string;
  }): Promise<RefundResult> {
    try {
      const refundParams: Stripe.RefundCreateParams = {
        payment_intent: params.transactionId
      };
      
      if (params.amount) {
        refundParams.amount = Math.round(params.amount * 100);
      }
      
      if (params.reason) {
        refundParams.reason = params.reason as any;
      }
      
      const refund = await this.stripe.refunds.create(refundParams);
      
      return {
        success: refund.status === 'succeeded',
        refundId: refund.id,
        amount: refund.amount / 100,
        status: this.mapStripeStatus(refund.status)
      };
      
    } catch (error: any) {
      return {
        success: false,
        refundId: '',
        amount: params.amount || 0,
        status: 'failed',
        errorMessage: error.message
      };
    }
  }
  
  verifyWebhook(payload: string, signature: string): boolean {
    try {
      this.stripe.webhooks.constructEvent(
        payload,
        signature,
        this.webhookSecret
      );
      return true;
    } catch (error) {
      return false;
    }
  }
  
  parseWebhookEvent(payload: string): any {
    return JSON.parse(payload);
  }
  
  private mapStripeStatus(
    status: string
  ): 'succeeded' | 'pending' | 'failed' {
    switch (status) {
      case 'succeeded':
        return 'succeeded';
      case 'processing':
      case 'requires_action':
      case 'requires_confirmation':
      case 'requires_payment_method':
        return 'pending';
      default:
        return 'failed';
    }
  }
}
```

### Step 3: Implement PayPal Adapter

```typescript
// infrastructure/gateways/PayPalPaymentAdapter.ts
import paypal from '@paypal/checkout-server-sdk';
import {
  IPaymentGateway,
  PaymentMethod,
  PaymentResult,
  RefundResult,
  Customer
} from '../../domain/gateways/IPaymentGateway';

export class PayPalPaymentAdapter implements IPaymentGateway {
  private client: paypal.core.PayPalHttpClient;
  
  constructor(clientId: string, clientSecret: string, mode: 'sandbox' | 'live') {
    const environment = mode === 'live'
      ? new paypal.core.LiveEnvironment(clientId, clientSecret)
      : new paypal.core.SandboxEnvironment(clientId, clientSecret);
    
    this.client = new paypal.core.PayPalHttpClient(environment);
  }
  
  async createCustomer(customer: Omit<Customer, 'id'>): Promise<Customer> {
    // PayPal doesn't have a direct customer creation API
    // We'll generate an ID and store mapping in our database
    const customerId = `paypal_${Date.now()}_${Math.random().toString(36)}`;
    
    return {
      id: customerId,
      email: customer.email,
      name: customer.name
    };
  }
  
  async getCustomer(customerId: string): Promise<Customer | null> {
    // In real implementation, retrieve from your database
    return null;
  }
  
  async deleteCustomer(customerId: string): Promise<boolean> {
    // Mark as deleted in your database
    return true;
  }
  
  async attachPaymentMethod(
    customerId: string,
    method: PaymentMethod
  ): Promise<string> {
    // PayPal uses payment tokens differently
    // Store the association in your database
    return method.token;
  }
  
  async detachPaymentMethod(paymentMethodId: string): Promise<boolean> {
    // Remove from your database
    return true;
  }
  
  async listPaymentMethods(customerId: string): Promise<PaymentMethod[]> {
    // Retrieve from your database
    return [];
  }
  
  async charge(params: {
    amount: number;
    currency: string;
    customerId: string;
    paymentMethodId: string;
    description?: string;
    metadata?: Record<string, any>;
  }): Promise<PaymentResult> {
    try {
      const request = new paypal.orders.OrdersCreateRequest();
      request.prefer('return=representation');
      request.requestBody({
        intent: 'CAPTURE',
        purchase_units: [{
          amount: {
            currency_code: params.currency.toUpperCase(),
            value: params.amount.toFixed(2)
          },
          description: params.description
        }],
        payment_source: {
          token: {
            id: params.paymentMethodId,
            type: 'PAYMENT_METHOD_TOKEN'
          }
        }
      });
      
      const order = await this.client.execute(request);
      
      // Capture the order
      const captureRequest = new paypal.orders.OrdersCaptureRequest(order.result.id);
      const capture = await this.client.execute(captureRequest);
      
      return {
        success: capture.result.status === 'COMPLETED',
        transactionId: capture.result.id,
        amount: params.amount,
        currency: params.currency,
        status: this.mapPayPalStatus(capture.result.status),
        metadata: params.metadata
      };
      
    } catch (error: any) {
      return {
        success: false,
        transactionId: '',
        amount: params.amount,
        currency: params.currency,
        status: 'failed',
        errorMessage: error.message
      };
    }
  }
  
  async refund(params: {
    transactionId: string;
    amount?: number;
    reason?: string;
  }): Promise<RefundResult> {
    try {
      const request = new paypal.payments.CapturesRefundRequest(params.transactionId);
      
      if (params.amount) {
        request.requestBody({
          amount: {
            value: params.amount.toFixed(2),
            currency_code: 'USD' // Get from original transaction
          }
        });
      }
      
      const refund = await this.client.execute(request);
      
      return {
        success: refund.result.status === 'COMPLETED',
        refundId: refund.result.id,
        amount: parseFloat(refund.result.amount.value),
        status: this.mapPayPalStatus(refund.result.status)
      };
      
    } catch (error: any) {
      return {
        success: false,
        refundId: '',
        amount: params.amount || 0,
        status: 'failed',
        errorMessage: error.message
      };
    }
  }
  
  verifyWebhook(payload: string, signature: string): boolean {
    // PayPal webhook verification
    // Implementation depends on PayPal SDK
    return true;
  }
  
  parseWebhookEvent(payload: string): any {
    return JSON.parse(payload);
  }
  
  private mapPayPalStatus(status: string): 'succeeded' | 'pending' | 'failed' {
    switch (status) {
      case 'COMPLETED':
        return 'succeeded';
      case 'PENDING':
      case 'CREATED':
        return 'pending';
      default:
        return 'failed';
    }
  }
}
```

### Step 4: Factory to Create Adapters

```typescript
// infrastructure/gateways/PaymentGatewayFactory.ts
import { IPaymentGateway } from '../../domain/gateways/IPaymentGateway';
import { StripePaymentAdapter } from './StripePaymentAdapter';
import { PayPalPaymentAdapter } from './PayPalPaymentAdapter';

export type PaymentProvider = 'stripe' | 'paypal' | 'square';

export class PaymentGatewayFactory {
  static create(provider: PaymentProvider, config: any): IPaymentGateway {
    switch (provider) {
      case 'stripe':
        return new StripePaymentAdapter(
          config.apiKey,
          config.webhookSecret
        );
      
      case 'paypal':
        return new PayPalPaymentAdapter(
          config.clientId,
          config.clientSecret,
          config.mode
        );
      
      case 'square':
        // return new SquarePaymentAdapter(config);
        throw new Error('Square adapter not implemented yet');
      
      default:
        throw new Error(`Unknown payment provider: ${provider}`);
    }
  }
}
```

### Step 5: Using Adapters in Service Layer

```typescript
// application/services/PaymentService.ts
export class PaymentService {
  constructor(private paymentGateway: IPaymentGateway) {}
  
  async processSubscriptionPayment(
    tenantId: string,
    customerId: string,
    amount: number
  ): Promise<PaymentResult> {
    // Get customer's default payment method
    const paymentMethods = await this.paymentGateway.listPaymentMethods(customerId);
    
    if (paymentMethods.length === 0) {
      throw new Error('No payment method found');
    }
    
    // Charge using the adapter
    const result = await this.paymentGateway.charge({
      amount,
      currency: 'USD',
      customerId,
      paymentMethodId: paymentMethods[0].token,
      description: `Subscription payment for tenant ${tenantId}`,
      metadata: {
        tenantId,
        type: 'subscription'
      }
    });
    
    // Store transaction in database
    await this.recordTransaction(tenantId, result);
    
    return result;
  }
  
  async refundPayment(
    transactionId: string,
    amount?: number,
    reason?: string
  ): Promise<RefundResult> {
    // Refund using the adapter
    const result = await this.paymentGateway.refund({
      transactionId,
      amount,
      reason
    });
    
    // Store refund in database
    await this.recordRefund(result);
    
    return result;
  }
  
  async addPaymentMethod(
    customerId: string,
    paymentToken: string
  ): Promise<string> {
    return this.paymentGateway.attachPaymentMethod(customerId, {
      type: 'card',
      token: paymentToken
    });
  }
  
  private async recordTransaction(tenantId: string, result: PaymentResult): Promise<void> {
    // Store in database
  }
  
  private async recordRefund(result: RefundResult): Promise<void> {
    // Store in database
  }
}

// Configuration and DI
const config = {
  provider: process.env.PAYMENT_PROVIDER || 'stripe',
  stripe: {
    apiKey: process.env.STRIPE_API_KEY,
    webhookSecret: process.env.STRIPE_WEBHOOK_SECRET
  },
  paypal: {
    clientId: process.env.PAYPAL_CLIENT_ID,
    clientSecret: process.env.PAYPAL_CLIENT_SECRET,
    mode: process.env.PAYPAL_MODE
  }
};

const paymentGateway = PaymentGatewayFactory.create(
  config.provider as PaymentProvider,
  config[config.provider]
);

const paymentService = new PaymentService(paymentGateway);
```

***

## Benefits of Adapter Pattern

✅ **Easy to Switch Providers:**

```typescript
// Switch from Stripe to PayPal
// Just change configuration - no code changes!
const gateway = PaymentGatewayFactory.create('paypal', config);
```

✅ **Support Multiple Providers:**

```typescript
// Use different providers for different tenants
const getGatewayForTenant = (tenantId: string) => {
  const tenant = getTenant(tenantId);
  return PaymentGatewayFactory.create(tenant.paymentProvider, config);
};
```

✅ **Easy Testing:**

```typescript
// Mock adapter for tests
class MockPaymentAdapter implements IPaymentGateway {
  async charge(params): Promise<PaymentResult> {
    return {
      success: true,
      transactionId: 'mock-123',
      amount: params.amount,
      currency: params.currency,
      status: 'succeeded'
    };
  }
  // ... other methods
}

const service = new PaymentService(new MockPaymentAdapter());
```

✅ **Consistent Interface:**

```typescript
// Same code works with any provider
async function processPayment(gateway: IPaymentGateway, amount: number) {
  return gateway.charge({
    amount,
    currency: 'USD',
    customerId: 'cust-123',
    paymentMethodId: 'pm-123'
  });
}
```

***

## More Real-World SaaS Adapters

### Email Service Adapter

```typescript
// domain/services/IEmailService.ts
export interface EmailMessage {
  to: string[];
  cc?: string[];
  bcc?: string[];
  subject: string;
  textBody?: string;
  htmlBody?: string;
  attachments?: Array<{
    filename: string;
    content: Buffer | string;
    contentType?: string;
  }>;
}

export interface EmailResult {
  success: boolean;
  messageId: string;
  errorMessage?: string;
}

export interface IEmailService {
  send(message: EmailMessage): Promise<EmailResult>;
  sendBatch(messages: EmailMessage[]): Promise<EmailResult[]>;
  verifyWebhook(payload: string, signature: string): boolean;
}

// infrastructure/services/SendGridEmailAdapter.ts
export class SendGridEmailAdapter implements IEmailService {
  constructor(private apiKey: string) {}
  
  async send(message: EmailMessage): Promise<EmailResult> {
    const sgMail = require('@sendgrid/mail');
    sgMail.setApiKey(this.apiKey);
    
    try {
      const [response] = await sgMail.send({
        to: message.to,
        cc: message.cc,
        bcc: message.bcc,
        from: process.env.FROM_EMAIL,
        subject: message.subject,
        text: message.textBody,
        html: message.htmlBody,
        attachments: message.attachments
      });
      
      return {
        success: true,
        messageId: response.headers['x-message-id']
      };
    } catch (error: any) {
      return {
        success: false,
        messageId: '',
        errorMessage: error.message
      };
    }
  }
  
  async sendBatch(messages: EmailMessage[]): Promise<EmailResult[]> {
    return Promise.all(messages.map(msg => this.send(msg)));
  }
  
  verifyWebhook(payload: string, signature: string): boolean {
    // SendGrid webhook verification
    return true;
  }
}

// infrastructure/services/AWSEmailAdapter.ts
export class AWSEmailAdapter implements IEmailService {
  constructor(private ses: AWS.SES) {}
  
  async send(message: EmailMessage): Promise<EmailResult> {
    try {
      const result = await this.ses.sendEmail({
        Source: process.env.FROM_EMAIL,
        Destination: {
          ToAddresses: message.to,
          CcAddresses: message.cc,
          BccAddresses: message.bcc
        },
        Message: {
          Subject: { Data: message.subject },
          Body: {
            Text: message.textBody ? { Data: message.textBody } : undefined,
            Html: message.htmlBody ? { Data: message.htmlBody } : undefined
          }
        }
      }).promise();
      
      return {
        success: true,
        messageId: result.MessageId
      };
    } catch (error: any) {
      return {
        success: false,
        messageId: '',
        errorMessage: error.message
      };
    }
  }
  
  async sendBatch(messages: EmailMessage[]): Promise<EmailResult[]> {
    return Promise.all(messages.map(msg => this.send(msg)));
  }
  
  verifyWebhook(payload: string, signature: string): boolean {
    // AWS SNS verification
    return true;
  }
}
```

### Storage Service Adapter

```typescript
// domain/services/IStorageService.ts
export interface UploadOptions {
  contentType?: string;
  metadata?: Record<string, string>;
  public?: boolean;
}

export interface UploadResult {
  success: boolean;
  url: string;
  key: string;
  errorMessage?: string;
}

export interface IStorageService {
  upload(key: string, data: Buffer, options?: UploadOptions): Promise<UploadResult>;
  download(key: string): Promise<Buffer>;
  delete(key: string): Promise<boolean>;
  getSignedUrl(key: string, expiresIn: number): Promise<string>;
  list(prefix: string): Promise<string[]>;
}

// infrastructure/services/S3StorageAdapter.ts
export class S3StorageAdapter implements IStorageService {
  constructor(
    private s3: AWS.S3,
    private bucket: string
  ) {}
  
  async upload(
    key: string,
    data: Buffer,
    options?: UploadOptions
  ): Promise<UploadResult> {
    try {
      await this.s3.putObject({
        Bucket: this.bucket,
        Key: key,
        Body: data,
        ContentType: options?.contentType,
        Metadata: options?.metadata,
        ACL: options?.public ? 'public-read' : 'private'
      }).promise();
      
      const url = options?.public
        ? `https://${this.bucket}.s3.amazonaws.com/${key}`
        : await this.getSignedUrl(key, 3600);
      
      return {
        success: true,
        url,
        key
      };
    } catch (error: any) {
      return {
        success: false,
        url: '',
        key,
        errorMessage: error.message
      };
    }
  }
  
  async download(key: string): Promise<Buffer> {
    const result = await this.s3.getObject({
      Bucket: this.bucket,
      Key: key
    }).promise();
    
    return result.Body as Buffer;
  }
  
  async delete(key: string): Promise<boolean> {
    try {
      await this.s3.deleteObject({
        Bucket: this.bucket,
        Key: key
      }).promise();
      return true;
    } catch (error) {
      return false;
    }
  }
  
  async getSignedUrl(key: string, expiresIn: number): Promise<string> {
    return this.s3.getSignedUrlPromise('getObject', {
      Bucket: this.bucket,
      Key: key,
      Expires: expiresIn
    });
  }
  
  async list(prefix: string): Promise<string[]> {
    const result = await this.s3.listObjectsV2({
      Bucket: this.bucket,
      Prefix: prefix
    }).promise();
    
    return (result.Contents || []).map(obj => obj.Key!);
  }
}

// infrastructure/services/GoogleCloudStorageAdapter.ts
export class GoogleCloudStorageAdapter implements IStorageService {
  constructor(
    private storage: Storage,
    private bucketName: string
  ) {}
  
  async upload(
    key: string,
    data: Buffer,
    options?: UploadOptions
  ): Promise<UploadResult> {
    try {
      const bucket = this.storage.bucket(this.bucketName);
      const file = bucket.file(key);
      
      await file.save(data, {
        contentType: options?.contentType,
        metadata: {
          metadata: options?.metadata
        },
        public: options?.public
      });
      
      const url = options?.public
        ? file.publicUrl()
        : (await file.getSignedUrl({ action: 'read', expires: Date.now() + 3600000 }))[0];
      
      return {
        success: true,
        url,
        key
      };
    } catch (error: any) {
      return {
        success: false,
        url: '',
        key,
        errorMessage: error.message
      };
    }
  }
  
  async download(key: string): Promise<Buffer> {
    const [data] = await this.storage
      .bucket(this.bucketName)
      .file(key)
      .download();
    
    return data;
  }
  
  async delete(key: string): Promise<boolean> {
    try {
      await this.storage
        .bucket(this.bucketName)
        .file(key)
        .delete();
      return true;
    } catch (error) {
      return false;
    }
  }
  
  async getSignedUrl(key: string, expiresIn: number): Promise<string> {
    const [url] = await this.storage
      .bucket(this.bucketName)
      .file(key)
      .getSignedUrl({
        action: 'read',
        expires: Date.now() + expiresIn * 1000
      });
    
    return url;
  }
  
  async list(prefix: string): Promise<string[]> {
    const [files] = await this.storage
      .bucket(this.bucketName)
      .getFiles({ prefix });
    
    return files.map(file => file.name);
  }
}
```

***

## Summary

**Adapter Pattern:**

* Converts one interface to another
* Allows incompatible interfaces to work together
* Critical for third-party service integration

**When to Use:**

* Integrating external services (payments, email, storage)
* Need to support multiple providers
* Want to switch providers easily
* Testing with mocks

**SaaS Use Cases:**

* Payment gateways (Stripe, PayPal, Square)
* Email providers (SendGrid, AWS SES, Mailgun)
* Storage services (S3, Google Cloud, Azure)
* SMS providers (Twilio, Vonage)
* Analytics services (Segment, Mixpanel)

**Next:** Pattern 9: Decorator Pattern or Pattern 10: Proxy Pattern?
