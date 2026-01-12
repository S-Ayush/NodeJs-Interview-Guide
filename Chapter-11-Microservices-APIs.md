# Chapter 11: Microservices & APIs

## 11.1 Monolith vs Microservices

```
Monolith Architecture
┌─────────────────────────┐
│   Single Application    │
│  ┌──────────────────┐   │
│  │   Auth Module    │   │
│  ├──────────────────┤   │
│  │   User Module    │   │
│  ├──────────────────┤   │
│  │  Payment Module  │   │
│  ├──────────────────┤   │
│  │  Order Module    │   │
│  └──────────────────┘   │
│   Single Database        │
└─────────────────────────┘

Microservices Architecture
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│   Auth   │  │   User   │  │ Payment  │  │  Order   │
│  Service │  │ Service  │  │ Service  │  │ Service  │
│    +DB   │  │   +DB    │  │   +DB    │  │   +DB    │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │              │              │
     └─────────────┴──────────────┴──────────────┘
                    API Gateway
```

### Pros & Cons:

**Microservices Pros:**
- Independent deployment
- Technology diversity
- Better fault isolation
- Easier to scale specific services

**Microservices Cons:**
- Complex infrastructure
- Network latency
- Data consistency challenges
- More operational overhead

## 11.2 REST API Design

### RESTful Principles:
```
Resource-Based URLs
GET    /api/users         - List users
GET    /api/users/:id     - Get user
POST   /api/users         - Create user
PUT    /api/users/:id     - Update user (full)
PATCH  /api/users/:id     - Update user (partial)
DELETE /api/users/:id     - Delete user

Nested Resources
GET    /api/users/:id/posts        - Get user's posts
POST   /api/users/:id/posts        - Create post for user
GET    /api/posts/:id/comments     - Get post comments
```

### Best Practices:
```javascript
// 1. Versioning
app.use('/api/v1', routesV1);
app.use('/api/v2', routesV2);

// 2. Standard response format
{
  "success": true,
  "data": { /* resource data */ },
  "message": "Operation successful",
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}

// 3. Error format
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input",
    "details": [
      { "field": "email", "message": "Invalid email format" }
    ]
  }
}

// 4. Status codes
200 - OK
201 - Created
204 - No Content
400 - Bad Request
401 - Unauthorized
403 - Forbidden
404 - Not Found
500 - Internal Server Error
```

## 11.3 GraphQL

### Schema Definition:
```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Query {
  users: [User!]!
  user(id: ID!): User
  posts: [Post!]!
}

type Mutation {
  createUser(name: String!, email: String!): User!
  updateUser(id: ID!, name: String, email: String): User!
  deleteUser(id: ID!): Boolean!
}
```

### Implementation with Apollo Server:
```javascript
const { ApolloServer, gql } = require('apollo-server-express');

const typeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
  }

  type Query {
    users: [User!]!
    user(id: ID!): User
  }

  type Mutation {
    createUser(name: String!, email: String!): User!
  }
`;

const resolvers = {
  Query: {
    users: () => User.find(),
    user: (_, { id }) => User.findById(id)
  },
  Mutation: {
    createUser: (_, { name, email }) => {
      return User.create({ name, email });
    }
  }
};

const server = new ApolloServer({ typeDefs, resolvers });
await server.start();
server.applyMiddleware({ app });
```

### Query Example:
```graphql
# Query only what you need
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}

# Mutation
mutation {
  createUser(name: "John", email: "john@example.com") {
    id
    name
  }
}
```

## 11.4 API Gateway

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│   API Gateway       │
│  ├─ Authentication  │
│  ├─ Rate Limiting   │
│  ├─ Load Balancing  │
│  └─ Request Routing │
└──────┬──────────────┘
       │
   ┌───┼───┬───────┐
   ▼   ▼   ▼       ▼
┌────┐ ┌───┐ ┌────┐ ┌────┐
│Svc1│ │Sv2│ │Svc3│ │Svc4│
└────┘ └───┘ └────┘ └────┘
```

### Implementation:
```javascript
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');

const app = express();

// Authentication middleware
const authenticate = (req, res, next) => {
  // Verify JWT token
  next();
};

// Route to services
app.use('/auth', createProxyMiddleware({
  target: 'http://auth-service:3001',
  changeOrigin: true
}));

app.use('/users', authenticate, createProxyMiddleware({
  target: 'http://user-service:3002',
  changeOrigin: true
}));

app.use('/orders', authenticate, createProxyMiddleware({
  target: 'http://order-service:3003',
  changeOrigin: true
}));

app.listen(3000);
```

## 11.5 Message Queues (RabbitMQ)

```
Producer ──▶ │Queue│ ──▶ Consumer
             │     │
             │  █  │
             │  █  │
             │  █  │
             └─────┘
```

### Implementation:
```javascript
const amqp = require('amqplib');

// Producer
async function sendMessage(queueName, message) {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  await channel.assertQueue(queueName, { durable: true });
  channel.sendToQueue(queueName, Buffer.from(JSON.stringify(message)), {
    persistent: true
  });

  console.log('Message sent:', message);
  await channel.close();
  await connection.close();
}

// Consumer
async function consumeMessages(queueName) {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();

  await channel.assertQueue(queueName, { durable: true });
  channel.prefetch(1);  // Process one message at a time

  console.log('Waiting for messages...');

  channel.consume(queueName, async (msg) => {
    const message = JSON.parse(msg.content.toString());
    console.log('Received:', message);

    // Process message
    await processMessage(message);

    // Acknowledge
    channel.ack(msg);
  });
}

// Usage
await sendMessage('email-queue', { to: 'user@example.com', subject: 'Hello' });
await consumeMessages('email-queue');
```

## 11.6 Service Communication

### 1. Synchronous (HTTP/REST):
```javascript
const axios = require('axios');

// Service A calls Service B
async function getUserOrders(userId) {
  try {
    const userResponse = await axios.get(`http://user-service/users/${userId}`);
    const ordersResponse = await axios.get(`http://order-service/orders?userId=${userId}`);

    return {
      user: userResponse.data,
      orders: ordersResponse.data
    };
  } catch (err) {
    console.error('Service communication error:', err);
    throw err;
  }
}
```

### 2. Asynchronous (Message Queue):
```javascript
// Order Service publishes event
async function createOrder(orderData) {
  const order = await Order.create(orderData);

  // Publish event to queue
  await publishEvent('order.created', {
    orderId: order.id,
    userId: order.userId,
    amount: order.amount
  });

  return order;
}

// Payment Service consumes event
async function handleOrderCreated(event) {
  const { orderId, userId, amount } = event;

  // Process payment
  const payment = await processPayment(userId, amount);

  // Publish payment completed event
  await publishEvent('payment.completed', {
    orderId,
    paymentId: payment.id
  });
}
```

## 11.7 Service Discovery

```
┌─────────────────┐
│ Service Registry│  (Consul, Eureka)
└────────┬────────┘
         │
    ┌────┼────┬────────┐
    │    │    │        │
 Register│ Discover  Heartbeat
    │    │    │        │
    ▼    ▼    ▼        ▼
┌────┐ ┌────┐ ┌────┐ ┌────┐
│Svc1│ │Svc2│ │Svc3│ │Svc4│
└────┘ └────┘ └────┘ └────┘
```

## 11.8 Circuit Breaker Pattern

```javascript
class CircuitBreaker {
  constructor(request, options = {}) {
    this.request = request;
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
    this.failureThreshold = options.failureThreshold || 5;
    this.timeout = options.timeout || 60000;
  }

  async call(...args) {
    if (this.state === 'OPEN') {
      if (this.nextAttempt <= Date.now()) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const response = await this.request(...args);
      return this.onSuccess(response);
    } catch (err) {
      return this.onFailure(err);
    }
  }

  onSuccess(response) {
    this.failureCount = 0;

    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount > 2) {
        this.state = 'CLOSED';
        this.successCount = 0;
      }
    }

    return response;
  }

  onFailure(err) {
    this.failureCount++;

    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }

    throw err;
  }
}

// Usage
const breaker = new CircuitBreaker(
  (userId) => axios.get(`http://user-service/users/${userId}`),
  { failureThreshold: 3, timeout: 30000 }
);

try {
  const response = await breaker.call(123);
  console.log(response.data);
} catch (err) {
  console.error('Request failed:', err);
}
```

## 11.9 Interview Questions

### Basic:
1. **What are microservices?**
2. **What's the difference between REST and GraphQL?**
3. **What is an API Gateway?**

### Intermediate:
1. **Explain service communication patterns.**
2. **What is a message queue and when to use it?**
3. **How do you handle distributed transactions?**

### Advanced:
1. **Design a microservices architecture for an e-commerce platform.**
2. **Explain the circuit breaker pattern.**
3. **How do you maintain data consistency in microservices?**

## 11.10 Key Takeaways

- Microservices provide independent scalability and deployment
- Use API Gateway for centralized routing and auth
- REST for simple CRUD, GraphQL for complex data requirements
- Message queues enable asynchronous communication
- Circuit breakers prevent cascading failures
- Service discovery enables dynamic scaling
- Consider trade-offs: complexity vs benefits

**Next:** Chapter 12 - Interview Questions & Best Practices
