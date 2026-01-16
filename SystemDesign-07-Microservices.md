# Chapter 7: Microservices Architecture

## 7.1 Introduction to Microservices

Microservices is an architectural style where an application is composed of small, independent services that communicate over well-defined APIs.

### Monolith vs Microservices

```
Monolithic Architecture:
┌────────────────────────────────────────┐
│         Single Application             │
│                                        │
│  ┌──────────┐  ┌──────────┐          │
│  │   UI     │  │ Business │          │
│  │  Layer   │→ │  Logic   │          │
│  └──────────┘  └──────────┘          │
│                      ↓                 │
│              ┌──────────┐             │
│              │ Database │             │
│              └──────────┘             │
│                                        │
│  - Single codebase                    │
│  - Single deployment                  │
│  - Shared database                    │
└────────────────────────────────────────┘

Microservices Architecture:
┌──────────┐  ┌──────────┐  ┌──────────┐
│ User     │  │ Product  │  │ Order    │
│ Service  │  │ Service  │  │ Service  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     ↓             ↓             ↓
┌─────────┐  ┌─────────┐  ┌─────────┐
│User DB  │  │Product  │  │Order DB │
└─────────┘  │DB       │  └─────────┘
             └─────────┘

- Multiple services
- Independent deployment
- Database per service
```

### When to Use Microservices

```
✓ Large, complex applications
✓ Multiple teams working independently
✓ Need for independent scaling
✓ Different technology stacks
✓ Frequent deployments
✓ Long-term project (> 2 years)

✗ Small applications
✗ Small team (< 5 developers)
✗ Simple CRUD operations
✗ Tight coupling requirements
✗ Limited operational capability
```

### Microservices Benefits

```
┌────────────────────────────────────────────────┐
│ 1. Independent Deployment                      │
│    - Deploy services separately                │
│    - Faster releases                           │
│    - Reduced risk                              │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 2. Technology Diversity                        │
│    - Different languages/frameworks per service│
│    - Use best tool for each job               │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 3. Scalability                                 │
│    - Scale services independently              │
│    - Optimize resources                        │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 4. Fault Isolation                             │
│    - Failure contained to single service       │
│    - System remains partially functional       │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 5. Team Autonomy                               │
│    - Teams own services end-to-end             │
│    - Faster decision making                    │
└────────────────────────────────────────────────┘
```

### Microservices Challenges

```
┌────────────────────────────────────────────────┐
│ 1. Distributed System Complexity               │
│    - Network latency                           │
│    - Partial failures                          │
│    - Data consistency                          │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 2. Operational Overhead                        │
│    - More services to deploy                   │
│    - Monitoring complexity                     │
│    - Infrastructure management                 │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 3. Testing Complexity                          │
│    - End-to-end testing harder                 │
│    - Integration testing needed                │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 4. Data Management                             │
│    - No distributed transactions               │
│    - Data duplication                          │
│    - Eventual consistency                      │
└────────────────────────────────────────────────┘
```

---

## 7.2 Service Decomposition Strategies

### Strategy 1: Decompose by Business Capability

Organize services around business functions.

```
E-Commerce System:

┌─────────────────────────────────────────┐
│         Business Capabilities           │
└─────────────────────────────────────────┘
           │
     ┌─────┼─────┬─────┬─────┬─────┐
     │     │     │     │     │     │
┌────▼──┐ │  ┌──▼──┐ │  ┌──▼───┐ │
│User   │ │  │Cart │ │  │Order │ │
│Mgmt   │ │  │Mgmt │ │  │Mgmt  │ │
└───────┘ │  └─────┘ │  └──────┘ │
          │          │            │
     ┌────▼───┐ ┌───▼────┐  ┌───▼──────┐
     │Product │ │Payment │  │Shipping  │
     │Catalog │ │Process │  │          │
     └────────┘ └────────┘  └──────────┘
```

**Example: E-Commerce**

```javascript
// User Service - Handles user management
class UserService {
  async createUser(userData) {
    const user = await db.users.create(userData);

    // Publish event for other services
    await eventBus.publish('user.created', {
      userId: user.id,
      email: user.email
    });

    return user;
  }

  async getUser(userId) {
    return await db.users.findById(userId);
  }

  async updateProfile(userId, updates) {
    const user = await db.users.update(userId, updates);

    await eventBus.publish('user.updated', {
      userId: user.id,
      changes: updates
    });

    return user;
  }
}

// Product Service - Manages product catalog
class ProductService {
  async createProduct(productData) {
    const product = await db.products.create(productData);

    await eventBus.publish('product.created', {
      productId: product.id
    });

    return product;
  }

  async getProduct(productId) {
    // Check cache first
    const cached = await cache.get(`product:${productId}`);
    if (cached) return JSON.parse(cached);

    const product = await db.products.findById(productId);

    // Cache for 1 hour
    await cache.setex(`product:${productId}`, 3600, JSON.stringify(product));

    return product;
  }

  async updateInventory(productId, quantity) {
    const product = await db.products.findById(productId);

    if (product.inventory < quantity) {
      throw new Error('Insufficient inventory');
    }

    await db.products.update(productId, {
      inventory: product.inventory - quantity
    });

    await eventBus.publish('inventory.updated', {
      productId,
      newQuantity: product.inventory - quantity
    });
  }
}

// Order Service - Handles order processing
class OrderService {
  async createOrder(userId, items) {
    // 1. Validate items with Product Service
    for (const item of items) {
      const product = await this.productService.getProduct(item.productId);
      if (!product) {
        throw new Error(`Product ${item.productId} not found`);
      }
    }

    // 2. Calculate total
    const total = items.reduce((sum, item) => sum + (item.price * item.quantity), 0);

    // 3. Create order
    const order = await db.orders.create({
      userId,
      items,
      total,
      status: 'pending'
    });

    // 4. Publish event (triggers downstream processing)
    await eventBus.publish('order.created', {
      orderId: order.id,
      userId,
      items,
      total
    });

    return order;
  }
}
```

### Strategy 2: Decompose by Subdomain (Domain-Driven Design)

Use DDD bounded contexts to define service boundaries.

```
Online Banking:

┌─────────────────────────────────────────┐
│     Account Management Context          │
│  - Account creation                     │
│  - Account details                      │
│  - Balance inquiry                      │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│     Transaction Context                 │
│  - Transfer money                       │
│  - Payment processing                   │
│  - Transaction history                  │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│     Loan Context                        │
│  - Loan application                     │
│  - Approval workflow                    │
│  - Repayment tracking                   │
└─────────────────────────────────────────┘
```

### Strategy 3: Decompose by Use Case/Transaction

One service per user journey or transaction type.

```
Travel Booking:

┌─────────────────────────────────────────┐
│  Flight Search Service                  │
│  - Search flights                       │
│  - Filter results                       │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Booking Service                        │
│  - Reserve seats                        │
│  - Hold booking                         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  Payment Service                        │
│  - Process payment                      │
│  - Confirm booking                      │
└─────────────────────────────────────────┘
```

### Service Size Guidelines

**Goldilocks Principle: Not too big, not too small**

```
Too Large (Mini-Monolith):
- Multiple teams need to coordinate
- Deployment takes > 15 minutes
- > 10,000 lines of code
- Multiple unrelated responsibilities

Just Right:
- 2-5 developers can own it
- Deploy in < 5 minutes
- 1,000-5,000 lines of code
- Single responsibility

Too Small (Nano-service):
- Only a few functions
- Always deployed together with others
- Adds network overhead without benefit
```

---

## 7.3 Inter-Service Communication

### Synchronous Communication (REST/gRPC)

**When to use:**
- Need immediate response
- Client needs to know if operation succeeded
- Simple request-reply pattern

**Example: REST**

```javascript
// Order Service calls Product Service
class OrderService {
  async createOrder(userId, items) {
    // Synchronous call to Product Service
    for (const item of items) {
      try {
        const response = await fetch(
          `http://product-service/api/products/${item.productId}`
        );

        if (!response.ok) {
          throw new Error(`Product ${item.productId} not found`);
        }

        const product = await response.json();

        // Check inventory
        if (product.inventory < item.quantity) {
          throw new Error('Insufficient inventory');
        }
      } catch (error) {
        console.error('Failed to validate product:', error);
        throw error;
      }
    }

    // Create order
    const order = await db.orders.create({ userId, items });

    return order;
  }
}
```

**Problems with Synchronous:**
```
1. Tight Coupling
   - Services must be available
   - Changes affect callers

2. Cascading Failures
   - If Product Service down → Order Service fails

3. Performance
   - Blocked waiting for response
   - Network latency accumulates
```

### Asynchronous Communication (Message Queue/Events)

**When to use:**
- Fire and forget operations
- Event notifications
- Long-running processes
- Decoupling services

**Example: Event-Driven**

```javascript
// Order Service publishes events
class OrderService {
  async createOrder(userId, items) {
    // 1. Create order immediately
    const order = await db.orders.create({
      userId,
      items,
      status: 'pending'
    });

    // 2. Publish event (async - no waiting)
    await eventBus.publish('order.created', {
      orderId: order.id,
      userId,
      items
    });

    // 3. Return immediately
    return order;
  }
}

// Payment Service subscribes to events
class PaymentService {
  constructor() {
    this.setupSubscriptions();
  }

  setupSubscriptions() {
    eventBus.subscribe('order.created', async (event) => {
      await this.processPayment(event.orderId, event.items);
    });
  }

  async processPayment(orderId, items) {
    try {
      // Process payment
      const payment = await this.chargeCustomer(orderId, items);

      // Publish success event
      await eventBus.publish('payment.succeeded', {
        orderId,
        paymentId: payment.id
      });
    } catch (error) {
      // Publish failure event
      await eventBus.publish('payment.failed', {
        orderId,
        reason: error.message
      });
    }
  }
}

// Inventory Service subscribes to payment events
class InventoryService {
  constructor() {
    this.setupSubscriptions();
  }

  setupSubscriptions() {
    eventBus.subscribe('payment.succeeded', async (event) => {
      await this.reserveInventory(event.orderId);
    });

    eventBus.subscribe('payment.failed', async (event) => {
      // Release any holds
      await this.releaseInventory(event.orderId);
    });
  }

  async reserveInventory(orderId) {
    const order = await this.getOrder(orderId);

    for (const item of order.items) {
      await db.inventory.update(item.productId, {
        available: { $decrement: item.quantity }
      });
    }

    await eventBus.publish('inventory.reserved', { orderId });
  }
}
```

**Benefits of Asynchronous:**
```
✓ Loose coupling
✓ Better fault tolerance
✓ Natural load leveling
✓ Parallel processing
✓ Easy to add new services
```

### Service Communication Patterns

#### 1. Request-Response (Sync)

```
Client → Service A → Service B → Service A → Client
         (request)   (request)   (response)  (response)

Use for: Immediate data needs
```

#### 2. Publish-Subscribe (Async)

```
Service A → Event Bus → Service B
                     → Service C
                     → Service D

Use for: Event notifications, fanout
```

#### 3. Request-Async Response

```
Client → Service A → Queue
         ↓ (accepted)
         Client

Queue → Service B → Process → Queue
                              ↓
Client ← Notification ← Service A

Use for: Long-running operations
```

---

## 7.4 API Gateway Pattern

API Gateway is a single entry point for all client requests.

### Architecture

```
                    ┌──────────────┐
                    │   Clients    │
                    │ (Web/Mobile) │
                    └──────┬───────┘
                           │
                           ↓
                    ┌──────────────┐
                    │ API Gateway  │
                    │ - Routing    │
                    │ - Auth       │
                    │ - Rate Limit │
                    │ - Aggregation│
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ↓                ↓                ↓
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │  User    │    │ Product  │    │  Order   │
    │ Service  │    │ Service  │    │ Service  │
    └──────────┘    └──────────┘    └──────────┘
```

### Implementation

```javascript
const express = require('express');
const httpProxy = require('http-proxy-middleware');

class APIGateway {
  constructor() {
    this.app = express();
    this.proxy = httpProxy.createProxyMiddleware;

    this.services = {
      user: 'http://user-service:3001',
      product: 'http://product-service:3002',
      order: 'http://order-service:3003'
    };

    this.setupMiddleware();
    this.setupRoutes();
  }

  setupMiddleware() {
    // Authentication
    this.app.use(async (req, res, next) => {
      if (req.path.startsWith('/public')) {
        return next();
      }

      const token = req.headers.authorization?.split(' ')[1];
      if (!token) {
        return res.status(401).json({ error: 'Unauthorized' });
      }

      try {
        const user = await this.verifyToken(token);
        req.user = user;
        next();
      } catch (error) {
        res.status(401).json({ error: 'Invalid token' });
      }
    });

    // Rate Limiting
    this.app.use(this.rateLimiter());

    // Logging
    this.app.use((req, res, next) => {
      console.log(`${req.method} ${req.path} - User: ${req.user?.id}`);
      next();
    });
  }

  setupRoutes() {
    // Simple routing
    this.app.use('/api/users', this.proxy({
      target: this.services.user,
      changeOrigin: true
    }));

    this.app.use('/api/products', this.proxy({
      target: this.services.product,
      changeOrigin: true
    }));

    this.app.use('/api/orders', this.proxy({
      target: this.services.order,
      changeOrigin: true
    }));

    // Aggregation endpoint
    this.app.get('/api/dashboard', async (req, res) => {
      try {
        // Call multiple services in parallel
        const [user, orders, recommendations] = await Promise.all([
          this.callService('user', `/users/${req.user.id}`),
          this.callService('order', `/orders/user/${req.user.id}`),
          this.callService('product', `/recommendations/${req.user.id}`)
        ]);

        res.json({
          user,
          recentOrders: orders.slice(0, 5),
          recommendations
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
  }

  async callService(serviceName, path) {
    const url = `${this.services[serviceName]}${path}`;
    const response = await fetch(url);

    if (!response.ok) {
      throw new Error(`${serviceName} error: ${response.statusText}`);
    }

    return await response.json();
  }

  rateLimiter() {
    const limits = new Map();

    return (req, res, next) => {
      const key = req.user?.id || req.ip;
      const now = Date.now();
      const windowMs = 60000; // 1 minute
      const maxRequests = 100;

      if (!limits.has(key)) {
        limits.set(key, { count: 1, resetAt: now + windowMs });
        return next();
      }

      const limit = limits.get(key);

      if (now > limit.resetAt) {
        limit.count = 1;
        limit.resetAt = now + windowMs;
        return next();
      }

      if (limit.count >= maxRequests) {
        return res.status(429).json({
          error: 'Too many requests',
          retryAfter: Math.ceil((limit.resetAt - now) / 1000)
        });
      }

      limit.count++;
      next();
    };
  }

  async verifyToken(token) {
    // JWT verification
    const decoded = jwt.verify(token, JWT_SECRET);
    return { id: decoded.userId };
  }
}

const gateway = new APIGateway();
gateway.app.listen(3000, () => {
  console.log('API Gateway running on port 3000');
});
```

### API Gateway Responsibilities

```
1. Routing
   - Direct requests to appropriate service
   - Load balancing

2. Authentication & Authorization
   - Verify user identity
   - Check permissions

3. Rate Limiting
   - Prevent abuse
   - Fair usage

4. Request/Response Transformation
   - Protocol translation
   - Data aggregation

5. Caching
   - Cache responses
   - Reduce backend load

6. Monitoring & Logging
   - Centralized logging
   - Metrics collection

7. Circuit Breaking
   - Fail fast when service down
   - Prevent cascading failures
```

---

## 7.5 Service Discovery

In microservices, services need to find each other dynamically.

### Service Registry Pattern

```
                    ┌──────────────┐
                    │   Service    │
                    │   Registry   │
                    │ (Consul/     │
                    │  Eureka)     │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │ Register   │ Discover   │
              │            │            │
    ┌─────────▼──┐    ┌───▼────────┐  │
    │ Service A  │    │ Service B  │  │
    │ (Instance) │    │ (Consumer) │  │
    └────────────┘    └────────────┘  │
                                       │
                           ┌───────────▼──┐
                           │ Service C    │
                           │ (Consumer)   │
                           └──────────────┘
```

### Implementation with Consul

```javascript
const Consul = require('consul');

class ServiceRegistry {
  constructor() {
    this.consul = new Consul({
      host: 'consul-server',
      port: 8500
    });
  }

  // Service registration
  async registerService(serviceName, serviceId, host, port) {
    await this.consul.agent.service.register({
      id: serviceId,
      name: serviceName,
      address: host,
      port: port,
      check: {
        http: `http://${host}:${port}/health`,
        interval: '10s',
        timeout: '5s'
      }
    });

    console.log(`Registered ${serviceName} (${serviceId})`);

    // Deregister on shutdown
    process.on('SIGINT', async () => {
      await this.deregisterService(serviceId);
      process.exit();
    });
  }

  async deregisterService(serviceId) {
    await this.consul.agent.service.deregister(serviceId);
    console.log(`Deregistered ${serviceId}`);
  }

  // Service discovery
  async discoverService(serviceName) {
    const result = await this.consul.health.service({
      service: serviceName,
      passing: true // Only healthy instances
    });

    if (result.length === 0) {
      throw new Error(`No healthy instances of ${serviceName} found`);
    }

    // Load balancing: Random selection
    const instance = result[Math.floor(Math.random() * result.length)];

    return {
      host: instance.Service.Address,
      port: instance.Service.Port,
      id: instance.Service.ID
    };
  }

  // Watch for service changes
  watchService(serviceName, callback) {
    const watch = this.consul.watch({
      method: this.consul.health.service,
      options: {
        service: serviceName,
        passing: true
      }
    });

    watch.on('change', (data) => {
      const instances = data.map(item => ({
        host: item.Service.Address,
        port: item.Service.Port,
        id: item.Service.ID
      }));

      callback(instances);
    });

    watch.on('error', (err) => {
      console.error('Watch error:', err);
    });
  }
}

// Usage in a service
class UserService {
  async start() {
    const registry = new ServiceRegistry();

    // Register this service
    await registry.registerService(
      'user-service',
      `user-service-${process.pid}`,
      'localhost',
      3001
    );

    // Start HTTP server
    this.app.listen(3001, () => {
      console.log('User Service running on port 3001');
    });
  }

  async callOrderService(orderId) {
    const registry = new ServiceRegistry();

    // Discover Order Service
    const instance = await registry.discoverService('order-service');

    // Make request
    const response = await fetch(
      `http://${instance.host}:${instance.port}/orders/${orderId}`
    );

    return await response.json();
  }
}
```

### Client-Side Load Balancing

```javascript
class LoadBalancedServiceClient {
  constructor(serviceName) {
    this.serviceName = serviceName;
    this.instances = [];
    this.currentIndex = 0;
    this.registry = new ServiceRegistry();

    // Watch for instance changes
    this.registry.watchService(serviceName, (instances) => {
      this.instances = instances;
      console.log(`Updated ${serviceName} instances:`, instances.length);
    });

    // Initial discovery
    this.refreshInstances();
  }

  async refreshInstances() {
    const instance = await this.registry.discoverService(this.serviceName);
    this.instances = [instance];
  }

  getNextInstance() {
    if (this.instances.length === 0) {
      throw new Error('No instances available');
    }

    // Round-robin
    const instance = this.instances[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.instances.length;

    return instance;
  }

  async call(path, options = {}) {
    let lastError;

    // Retry with different instances
    for (let attempt = 0; attempt < 3; attempt++) {
      try {
        const instance = this.getNextInstance();
        const url = `http://${instance.host}:${instance.port}${path}`;

        const response = await fetch(url, options);

        if (response.ok) {
          return await response.json();
        }

        lastError = new Error(`HTTP ${response.status}`);
      } catch (error) {
        lastError = error;
        console.error(`Attempt ${attempt + 1} failed:`, error.message);
      }
    }

    throw lastError;
  }
}

// Usage
const orderClient = new LoadBalancedServiceClient('order-service');
const order = await orderClient.call('/orders/123');
```

---

## 7.6 Distributed Transactions

In microservices, each service has its own database. How to maintain consistency across services?

### The Problem

```
Order Creation Flow:

1. Order Service: Create order
2. Payment Service: Charge customer
3. Inventory Service: Reduce inventory
4. Shipping Service: Create shipment

What if step 3 fails? Need to rollback steps 1 & 2!
```

### Solution 1: Two-Phase Commit (2PC)

**Not recommended for microservices** - blocking, single point of failure

```
Phase 1: Prepare
Coordinator → All Services: "Can you commit?"
All Services → Coordinator: "Yes" or "No"

Phase 2: Commit
If all "Yes":
  Coordinator → All Services: "Commit!"
Else:
  Coordinator → All Services: "Abort!"
```

**Problems:**
```
- Blocking (services locked during coordination)
- Coordinator is single point of failure
- Doesn't scale well
- Network partitions problematic
```

### Solution 2: Saga Pattern (Recommended)

**Saga**: Sequence of local transactions with compensating transactions.

#### Choreography-Based Saga

Each service publishes events; others react.

```javascript
// Order Service
class OrderService {
  async createOrder(userId, items) {
    // 1. Create order
    const order = await db.orders.create({
      userId,
      items,
      status: 'pending',
      sagaState: 'started'
    });

    // 2. Publish event
    await eventBus.publish('order.created', {
      orderId: order.id,
      userId,
      items,
      total: order.total
    });

    return order;
  }

  // Listen for success/failure events
  setupSagaListeners() {
    // Success path
    eventBus.subscribe('shipping.created', async (event) => {
      await db.orders.update(event.orderId, {
        status: 'confirmed',
        sagaState: 'completed'
      });
    });

    // Compensation path (rollback)
    eventBus.subscribe('payment.failed', async (event) => {
      await this.cancelOrder(event.orderId);
    });

    eventBus.subscribe('inventory.failed', async (event) => {
      await this.cancelOrder(event.orderId);
    });
  }

  async cancelOrder(orderId) {
    await db.orders.update(orderId, {
      status: 'cancelled',
      sagaState: 'compensated'
    });

    // Publish compensation event
    await eventBus.publish('order.cancelled', { orderId });
  }
}

// Payment Service
class PaymentService {
  setupSagaListeners() {
    eventBus.subscribe('order.created', async (event) => {
      try {
        await this.processPayment(event.orderId, event.total);

        await eventBus.publish('payment.succeeded', {
          orderId: event.orderId,
          amount: event.total
        });
      } catch (error) {
        await eventBus.publish('payment.failed', {
          orderId: event.orderId,
          reason: error.message
        });
      }
    });

    // Compensating transaction
    eventBus.subscribe('order.cancelled', async (event) => {
      await this.refundPayment(event.orderId);
    });
  }

  async refundPayment(orderId) {
    const payment = await db.payments.findOne({ orderId });

    if (payment && payment.status === 'completed') {
      await this.issueRefund(payment.id);

      await eventBus.publish('payment.refunded', {
        orderId,
        amount: payment.amount
      });
    }
  }
}

// Inventory Service
class InventoryService {
  setupSagaListeners() {
    eventBus.subscribe('payment.succeeded', async (event) => {
      try {
        await this.reserveInventory(event.orderId);

        await eventBus.publish('inventory.reserved', {
          orderId: event.orderId
        });
      } catch (error) {
        await eventBus.publish('inventory.failed', {
          orderId: event.orderId,
          reason: error.message
        });
      }
    });

    // Compensating transaction
    eventBus.subscribe('order.cancelled', async (event) => {
      await this.releaseInventory(event.orderId);
    });
  }

  async releaseInventory(orderId) {
    const reservation = await db.reservations.findOne({ orderId });

    if (reservation) {
      for (const item of reservation.items) {
        await db.inventory.update(item.productId, {
          available: { $increment: item.quantity }
        });
      }

      await db.reservations.delete({ orderId });

      await eventBus.publish('inventory.released', { orderId });
    }
  }
}

// Shipping Service
class ShippingService {
  setupSagaListeners() {
    eventBus.subscribe('inventory.reserved', async (event) => {
      try {
        await this.createShipment(event.orderId);

        await eventBus.publish('shipping.created', {
          orderId: event.orderId
        });
      } catch (error) {
        await eventBus.publish('shipping.failed', {
          orderId: event.orderId,
          reason: error.message
        });
      }
    });

    // Compensating transaction
    eventBus.subscribe('order.cancelled', async (event) => {
      await this.cancelShipment(event.orderId);
    });
  }
}
```

#### Orchestration-Based Saga

Central orchestrator coordinates the saga.

```javascript
class OrderSagaOrchestrator {
  async executeOrderSaga(orderId) {
    const saga = await db.sagas.create({
      orderId,
      status: 'started',
      steps: []
    });

    try {
      // Step 1: Process Payment
      await this.executeStep(saga, 'payment', async () => {
        const result = await this.paymentService.processPayment(orderId);
        return result;
      });

      // Step 2: Reserve Inventory
      await this.executeStep(saga, 'inventory', async () => {
        const result = await this.inventoryService.reserve(orderId);
        return result;
      });

      // Step 3: Create Shipment
      await this.executeStep(saga, 'shipping', async () => {
        const result = await this.shippingService.createShipment(orderId);
        return result;
      });

      // All steps succeeded
      await db.sagas.update(saga.id, { status: 'completed' });
      await db.orders.update(orderId, { status: 'confirmed' });

    } catch (error) {
      // Rollback (execute compensating transactions in reverse order)
      await this.compensate(saga);
      await db.orders.update(orderId, { status: 'cancelled' });
    }
  }

  async executeStep(saga, stepName, action) {
    console.log(`Executing step: ${stepName}`);

    try {
      const result = await action();

      // Record successful step
      await db.sagas.update(saga.id, {
        steps: {
          $push: {
            name: stepName,
            status: 'completed',
            result,
            timestamp: Date.now()
          }
        }
      });

      return result;
    } catch (error) {
      // Record failed step
      await db.sagas.update(saga.id, {
        steps: {
          $push: {
            name: stepName,
            status: 'failed',
            error: error.message,
            timestamp: Date.now()
          }
        }
      });

      throw error;
    }
  }

  async compensate(saga) {
    console.log('Starting compensation...');

    // Execute compensating transactions in reverse order
    const completedSteps = saga.steps
      .filter(s => s.status === 'completed')
      .reverse();

    for (const step of completedSteps) {
      try {
        await this.compensateStep(step);
      } catch (error) {
        console.error(`Failed to compensate ${step.name}:`, error);
        // Log but continue with other compensations
      }
    }

    await db.sagas.update(saga.id, { status: 'compensated' });
  }

  async compensateStep(step) {
    switch (step.name) {
      case 'payment':
        await this.paymentService.refund(step.result.paymentId);
        break;
      case 'inventory':
        await this.inventoryService.release(step.result.reservationId);
        break;
      case 'shipping':
        await this.shippingService.cancel(step.result.shipmentId);
        break;
    }
  }
}

// Usage
const orchestrator = new OrderSagaOrchestrator();
await orchestrator.executeOrderSaga(orderId);
```

### Choreography vs Orchestration

```
┌──────────────────┬─────────────────┬─────────────────┐
│ Aspect           │ Choreography    │ Orchestration   │
├──────────────────┼─────────────────┼─────────────────┤
│ Coordination     │ Distributed     │ Centralized     │
│ Coupling         │ Loose           │ Tighter         │
│ Complexity       │ High (global)   │ High (central)  │
│ Visibility       │ Harder          │ Easier          │
│ Single Point     │ No              │ Yes (orchest.)  │
│ Best for         │ Simple sagas    │ Complex sagas   │
└──────────────────┴─────────────────┴─────────────────┘
```

---

## 7.7 Observability & Monitoring

In microservices, observability is critical for debugging and operations.

### Three Pillars of Observability

```
1. Logging
   - What happened?
   - Structured logs
   - Centralized aggregation

2. Metrics
   - How is the system performing?
   - Time-series data
   - Alerting

3. Tracing
   - Where is the request going?
   - Distributed tracing
   - Performance analysis
```

### Distributed Tracing

```
Request Flow:

API Gateway → User Service → Order Service → Payment Service
    ↓            ↓              ↓                ↓
  Trace ID    Span ID        Span ID          Span ID
  abc123      span-1         span-2           span-3
```

**Implementation with OpenTelemetry:**

```javascript
const { trace, context } = require('@opentelemetry/api');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

// Initialize tracer
const provider = new NodeTracerProvider();
provider.register();

registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation()
  ]
});

const tracer = trace.getTracer('order-service');

// Use in service
class OrderService {
  async createOrder(userId, items) {
    // Create a span
    const span = tracer.startSpan('createOrder');
    span.setAttribute('user.id', userId);
    span.setAttribute('items.count', items.length);

    try {
      // Your business logic
      const order = await db.orders.create({ userId, items });

      // Call other service (span automatically propagated)
      await this.callPaymentService(order.id, order.total);

      span.setStatus({ code: 0 }); // Success
      return order;
    } catch (error) {
      span.setStatus({ code: 2, message: error.message }); // Error
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  }

  async callPaymentService(orderId, amount) {
    const span = tracer.startSpan('callPaymentService');

    try {
      const response = await fetch('http://payment-service/charge', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          // Trace context automatically injected
        },
        body: JSON.stringify({ orderId, amount })
      });

      return await response.json();
    } finally {
      span.end();
    }
  }
}
```

### Centralized Logging

```javascript
const winston = require('winston');

// Structured logging
const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'app.log' })
  ]
});

// Add trace ID to all logs
app.use((req, res, next) => {
  req.traceId = req.headers['x-trace-id'] || generateTraceId();
  next();
});

// Usage in service
class OrderService {
  async createOrder(req, userId, items) {
    logger.info('Creating order', {
      traceId: req.traceId,
      userId,
      itemCount: items.length,
      service: 'order-service'
    });

    try {
      const order = await db.orders.create({ userId, items });

      logger.info('Order created successfully', {
        traceId: req.traceId,
        orderId: order.id,
        userId,
        service: 'order-service'
      });

      return order;
    } catch (error) {
      logger.error('Failed to create order', {
        traceId: req.traceId,
        userId,
        error: error.message,
        stack: error.stack,
        service: 'order-service'
      });

      throw error;
    }
  }
}
```

### Metrics Collection

```javascript
const prometheus = require('prom-client');

// Create metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code']
});

const orderCounter = new prometheus.Counter({
  name: 'orders_created_total',
  help: 'Total number of orders created',
  labelNames: ['status']
});

// Middleware to track request duration
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .observe(duration);
  });

  next();
});

// Track business metrics
class OrderService {
  async createOrder(userId, items) {
    try {
      const order = await db.orders.create({ userId, items });

      // Increment success counter
      orderCounter.labels('success').inc();

      return order;
    } catch (error) {
      // Increment failure counter
      orderCounter.labels('failure').inc();
      throw error;
    }
  }
}

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await prometheus.register.metrics());
});
```

---

## 7.8 Key Takeaways

### When to Use Microservices

```
✓ Large, complex systems
✓ Multiple independent teams
✓ Different scalability requirements
✓ Technology diversity needed
✓ Rapid, independent deployments

✗ Small applications
✗ Small teams
✗ Tight coupling required
✗ Limited operational capability
```

### Critical Patterns

```
1. API Gateway
   - Single entry point
   - Cross-cutting concerns

2. Service Discovery
   - Dynamic service location
   - Health checking

3. Saga Pattern
   - Distributed transactions
   - Compensating transactions

4. Circuit Breaker
   - Fail fast
   - Prevent cascading failures

5. Event-Driven
   - Loose coupling
   - Async communication
```

### Best Practices

```
1. Each service owns its data
2. Communicate via well-defined APIs
3. Design for failure
4. Automate everything
5. Monitor and observe
6. Version APIs
7. Use async communication when possible
8. Keep services small and focused
```

---

## 7.9 Interview Questions

### Basic:
1. What are microservices and how do they differ from monoliths?
2. What is service discovery?
3. Explain the Saga pattern.

### Intermediate:
1. How do you handle distributed transactions in microservices?
2. Compare synchronous vs asynchronous communication.
3. What is an API Gateway and why is it needed?

### Advanced:
1. Design a microservices architecture for an e-commerce platform.
2. How do you ensure data consistency across microservices?
3. Explain how you would implement distributed tracing.

---

**Next Chapter:** [Chapter 8: Reliability & Fault Tolerance](./SystemDesign-08-Reliability.md)

In the next chapter, we'll cover:
- Circuit breaker pattern
- Retry mechanisms and exponential backoff
- Bulkhead pattern
- Rate limiting and throttling
- Chaos engineering
- High availability strategies
