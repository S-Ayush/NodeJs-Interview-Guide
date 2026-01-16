# Chapter 8: Reliability & Fault Tolerance

## 8.1 Introduction

Building reliable systems means designing for failure. In distributed systems, failures are inevitable, not exceptional.

### Murphy's Law in Distributed Systems

```
"Anything that can go wrong, will go wrong"

In distributed systems:
- Networks fail
- Servers crash
- Disks fill up
- Dependencies become unavailable
- Code has bugs
- Traffic spikes unexpectedly
```

### Reliability Principles

```
┌────────────────────────────────────────────────┐
│ 1. Design for Failure                          │
│    - Assume everything can fail                │
│    - Plan failure recovery                     │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 2. Fail Fast                                   │
│    - Detect failures quickly                   │
│    - Don't wait for timeouts                   │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 3. Graceful Degradation                        │
│    - Provide reduced functionality             │
│    - Better than complete failure              │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 4. Isolate Failures                            │
│    - Prevent cascading failures                │
│    - Use bulkheads                             │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 5. Monitor Everything                          │
│    - Know when things fail                     │
│    - Alert proactively                         │
└────────────────────────────────────────────────┘
```

---

## 8.2 Circuit Breaker Pattern

The Circuit Breaker pattern prevents an application from repeatedly trying to execute an operation that's likely to fail.

### How It Works

```
States:

┌────────────┐
│   CLOSED   │  ← Normal operation
│            │    All requests pass through
└──────┬─────┘
       │ Failures exceed threshold
       ↓
┌────────────┐
│    OPEN    │  ← Circuit is open
│            │    All requests fail immediately
└──────┬─────┘
       │ After timeout period
       ↓
┌────────────┐
│ HALF_OPEN  │  ← Testing if service recovered
│            │    Limited requests allowed
└──────┬─────┘
       │
       ├─ Success → CLOSED
       └─ Failure → OPEN
```

### Implementation

```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 60000; // 60 seconds
    this.resetTimeout = null;

    // State
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();

    // Metrics
    this.metrics = {
      totalRequests: 0,
      successfulRequests: 0,
      failedRequests: 0,
      rejectedRequests: 0
    };
  }

  async execute(fn) {
    this.metrics.totalRequests++;

    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        // Circuit is open, reject immediately
        this.metrics.rejectedRequests++;
        throw new Error('Circuit breaker is OPEN');
      } else {
        // Timeout expired, try half-open
        this.state = 'HALF_OPEN';
        console.log('Circuit breaker entering HALF_OPEN state');
      }
    }

    try {
      // Execute the function
      const result = await fn();

      // Success
      this.onSuccess();
      return result;

    } catch (error) {
      // Failure
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.metrics.successfulRequests++;

    if (this.state === 'HALF_OPEN') {
      this.successCount++;

      if (this.successCount >= this.successThreshold) {
        // Recovered, close circuit
        this.state = 'CLOSED';
        this.successCount = 0;
        console.log('Circuit breaker CLOSED (recovered)');
      }
    }
  }

  onFailure() {
    this.failureCount++;
    this.metrics.failedRequests++;

    if (this.state === 'HALF_OPEN') {
      // Failed during testing, open circuit again
      this.trip();
    } else if (this.failureCount >= this.failureThreshold) {
      // Failures exceeded threshold, trip circuit
      this.trip();
    }
  }

  trip() {
    this.state = 'OPEN';
    this.nextAttempt = Date.now() + this.timeout;
    this.successCount = 0;

    console.log(`Circuit breaker OPEN (will retry after ${this.timeout}ms)`);
  }

  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
      nextAttempt: this.nextAttempt,
      metrics: this.metrics
    };
  }
}

// Usage example
const paymentServiceBreaker = new CircuitBreaker({
  failureThreshold: 5,
  successThreshold: 2,
  timeout: 30000 // 30 seconds
});

async function callPaymentService(orderId, amount) {
  try {
    const result = await paymentServiceBreaker.execute(async () => {
      // Actual service call
      const response = await fetch('http://payment-service/charge', {
        method: 'POST',
        body: JSON.stringify({ orderId, amount }),
        headers: { 'Content-Type': 'application/json' }
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      return await response.json();
    });

    return result;

  } catch (error) {
    if (error.message === 'Circuit breaker is OPEN') {
      // Handle circuit breaker open
      console.log('Payment service unavailable, using fallback');
      return { status: 'pending', message: 'Payment will be processed later' };
    }

    throw error;
  }
}
```

### Real-World Example: Netflix Hystrix Pattern

```javascript
class HystrixCommand {
  constructor(options) {
    this.serviceName = options.serviceName;
    this.timeout = options.timeout || 1000;
    this.fallback = options.fallback;
    this.circuitBreaker = new CircuitBreaker(options);
  }

  async execute(fn) {
    try {
      // Wrap with timeout
      const result = await this.withTimeout(
        this.circuitBreaker.execute(fn),
        this.timeout
      );

      return result;

    } catch (error) {
      console.error(`${this.serviceName} failed:`, error.message);

      // Execute fallback if available
      if (this.fallback) {
        console.log(`Using fallback for ${this.serviceName}`);
        return await this.fallback(error);
      }

      throw error;
    }
  }

  async withTimeout(promise, ms) {
    return Promise.race([
      promise,
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), ms)
      )
    ]);
  }
}

// Usage
const getUserCommand = new HystrixCommand({
  serviceName: 'UserService',
  timeout: 2000,
  failureThreshold: 5,
  timeout: 30000,
  fallback: async (error) => {
    // Return cached data or default
    return await cache.get('user:default') || {
      id: 'unknown',
      name: 'Guest User'
    };
  }
});

const user = await getUserCommand.execute(async () => {
  return await userService.getUser(userId);
});
```

---

## 8.3 Retry Mechanisms

Retry failed operations with intelligent backoff strategies.

### Simple Retry

```javascript
async function retry(fn, maxAttempts = 3) {
  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      console.log(`Attempt ${attempt} failed:`, error.message);

      if (attempt < maxAttempts) {
        // Wait before retrying
        await sleep(1000);
      }
    }
  }

  throw lastError;
}

// Usage
const result = await retry(async () => {
  return await fetch('http://api.example.com/data');
}, 3);
```

### Exponential Backoff

```javascript
class ExponentialBackoff {
  constructor(options = {}) {
    this.maxAttempts = options.maxAttempts || 5;
    this.initialDelay = options.initialDelay || 1000; // 1 second
    this.maxDelay = options.maxDelay || 30000; // 30 seconds
    this.factor = options.factor || 2;
    this.jitter = options.jitter !== false; // Add randomness by default
  }

  async execute(fn) {
    let lastError;

    for (let attempt = 1; attempt <= this.maxAttempts; attempt++) {
      try {
        return await fn();

      } catch (error) {
        lastError = error;

        if (attempt === this.maxAttempts) {
          // Last attempt, don't retry
          break;
        }

        // Check if error is retryable
        if (!this.isRetryable(error)) {
          throw error;
        }

        // Calculate delay
        const delay = this.calculateDelay(attempt);

        console.log(
          `Attempt ${attempt} failed. Retrying in ${delay}ms...`,
          error.message
        );

        await sleep(delay);
      }
    }

    throw lastError;
  }

  calculateDelay(attempt) {
    // Exponential: 1s, 2s, 4s, 8s, 16s, ...
    let delay = this.initialDelay * Math.pow(this.factor, attempt - 1);

    // Cap at max delay
    delay = Math.min(delay, this.maxDelay);

    // Add jitter (randomness) to prevent thundering herd
    if (this.jitter) {
      delay = delay * (0.5 + Math.random() * 0.5);
    }

    return Math.floor(delay);
  }

  isRetryable(error) {
    // Don't retry client errors (4xx)
    if (error.status >= 400 && error.status < 500) {
      return false;
    }

    // Retry server errors (5xx) and network errors
    return true;
  }
}

// Usage
const backoff = new ExponentialBackoff({
  maxAttempts: 5,
  initialDelay: 1000,
  maxDelay: 30000
});

const result = await backoff.execute(async () => {
  const response = await fetch('http://api.example.com/data');

  if (!response.ok) {
    const error = new Error(`HTTP ${response.status}`);
    error.status = response.status;
    throw error;
  }

  return await response.json();
});
```

### Retry Budget

Limit total retries to prevent resource exhaustion.

```javascript
class RetryBudget {
  constructor(options = {}) {
    this.budget = options.budget || 0.1; // 10% of requests can retry
    this.windowMs = options.windowMs || 60000; // 1 minute window

    this.requests = [];
    this.retries = [];
  }

  canRetry() {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    // Remove old entries
    this.requests = this.requests.filter(t => t > windowStart);
    this.retries = this.retries.filter(t => t > windowStart);

    // Calculate retry ratio
    const retryRatio = this.retries.length / (this.requests.length || 1);

    return retryRatio < this.budget;
  }

  recordRequest() {
    this.requests.push(Date.now());
  }

  recordRetry() {
    this.retries.push(Date.now());
  }

  async execute(fn, maxAttempts = 3) {
    this.recordRequest();

    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        return await fn();

      } catch (error) {
        if (attempt === maxAttempts) {
          throw error;
        }

        // Check budget before retrying
        if (!this.canRetry()) {
          console.log('Retry budget exceeded, failing fast');
          throw error;
        }

        this.recordRetry();
        console.log(`Retrying (attempt ${attempt + 1}/${maxAttempts})`);
        await sleep(1000 * attempt);
      }
    }
  }
}

// Usage
const retryBudget = new RetryBudget({
  budget: 0.1, // Allow 10% retries
  windowMs: 60000 // 1 minute window
});

const result = await retryBudget.execute(async () => {
  return await apiCall();
}, 3);
```

---

## 8.4 Bulkhead Pattern

Isolate resources to prevent cascading failures.

### Concept

```
Without Bulkhead:
┌────────────────────────────────────┐
│  Thread Pool (100 threads)         │
│                                    │
│  Service A: 90 threads (slow)      │
│  Service B: 10 threads (fast)      │
│  Service C: 0 threads (starved!)   │
└────────────────────────────────────┘

With Bulkhead:
┌────────────────────────────────────┐
│  Service A                         │
│  Thread Pool: 30 threads           │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│  Service B                         │
│  Thread Pool: 30 threads           │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│  Service C                         │
│  Thread Pool: 30 threads           │
└────────────────────────────────────┘
```

### Implementation

```javascript
class Bulkhead {
  constructor(name, options = {}) {
    this.name = name;
    this.maxConcurrent = options.maxConcurrent || 10;
    this.maxQueue = options.maxQueue || 100;

    this.running = 0;
    this.queue = [];

    // Metrics
    this.metrics = {
      executed: 0,
      rejected: 0,
      queued: 0
    };
  }

  async execute(fn) {
    // Check if we can execute immediately
    if (this.running < this.maxConcurrent) {
      return await this.run(fn);
    }

    // Check if we can queue
    if (this.queue.length >= this.maxQueue) {
      this.metrics.rejected++;
      throw new Error(`Bulkhead ${this.name}: Queue full (rejected)`);
    }

    // Add to queue
    return new Promise((resolve, reject) => {
      this.queue.push({ fn, resolve, reject });
      this.metrics.queued++;
    });
  }

  async run(fn) {
    this.running++;
    this.metrics.executed++;

    try {
      const result = await fn();
      return result;

    } finally {
      this.running--;

      // Process next item in queue
      if (this.queue.length > 0) {
        const { fn, resolve, reject } = this.queue.shift();

        this.run(fn)
          .then(resolve)
          .catch(reject);
      }
    }
  }

  getMetrics() {
    return {
      name: this.name,
      running: this.running,
      queued: this.queue.length,
      ...this.metrics
    };
  }
}

// Usage: Separate bulkheads for different services
const userServiceBulkhead = new Bulkhead('UserService', {
  maxConcurrent: 20,
  maxQueue: 100
});

const paymentServiceBulkhead = new Bulkhead('PaymentService', {
  maxConcurrent: 10,
  maxQueue: 50
});

// User service calls
async function getUser(userId) {
  return await userServiceBulkhead.execute(async () => {
    return await fetch(`http://user-service/users/${userId}`);
  });
}

// Payment service calls
async function processPayment(orderId, amount) {
  return await paymentServiceBulkhead.execute(async () => {
    return await fetch('http://payment-service/charge', {
      method: 'POST',
      body: JSON.stringify({ orderId, amount })
    });
  });
}

// Even if payment service is slow/failing,
// user service calls remain unaffected!
```

### Combined Pattern: Circuit Breaker + Bulkhead

```javascript
class ResilientService {
  constructor(name, options = {}) {
    this.name = name;
    this.circuitBreaker = new CircuitBreaker(options);
    this.bulkhead = new Bulkhead(name, options);
  }

  async execute(fn) {
    // First check bulkhead (resource isolation)
    return await this.bulkhead.execute(async () => {
      // Then check circuit breaker (fail fast)
      return await this.circuitBreaker.execute(fn);
    });
  }

  getHealth() {
    return {
      name: this.name,
      circuitBreaker: this.circuitBreaker.getState(),
      bulkhead: this.bulkhead.getMetrics()
    };
  }
}

// Usage
const paymentService = new ResilientService('PaymentService', {
  maxConcurrent: 10,
  maxQueue: 50,
  failureThreshold: 5,
  timeout: 30000
});

const result = await paymentService.execute(async () => {
  return await callPaymentAPI();
});

// Monitor health
console.log(paymentService.getHealth());
```

---

## 8.5 Timeout Strategies

Always set timeouts to prevent indefinite waiting.

### Types of Timeouts

```
1. Connection Timeout
   - Time to establish connection
   - Typically: 1-5 seconds

2. Request Timeout
   - Time for entire request-response
   - Typically: 5-30 seconds

3. Idle Timeout
   - Time connection can be idle
   - Typically: 30-60 seconds
```

### Implementation

```javascript
class TimeoutWrapper {
  constructor(timeoutMs) {
    this.timeoutMs = timeoutMs;
  }

  async execute(fn, timeoutMs = this.timeoutMs) {
    return Promise.race([
      fn(),
      new Promise((_, reject) =>
        setTimeout(
          () => reject(new Error(`Timeout after ${timeoutMs}ms`)),
          timeoutMs
        )
      )
    ]);
  }
}

// Usage
const timeout = new TimeoutWrapper(5000); // 5 seconds

try {
  const result = await timeout.execute(async () => {
    return await fetch('http://slow-api.example.com/data');
  });

  console.log('Success:', result);

} catch (error) {
  if (error.message.includes('Timeout')) {
    console.log('Request timed out, using cached data');
    return await cache.get('fallback-data');
  }

  throw error;
}
```

### Adaptive Timeouts

```javascript
class AdaptiveTimeout {
  constructor(options = {}) {
    this.initialTimeout = options.initialTimeout || 1000;
    this.minTimeout = options.minTimeout || 500;
    this.maxTimeout = options.maxTimeout || 10000;

    this.latencies = [];
    this.maxSamples = 100;
  }

  async execute(fn) {
    const timeout = this.calculateTimeout();
    const start = Date.now();

    try {
      const result = await Promise.race([
        fn(),
        new Promise((_, reject) =>
          setTimeout(() => reject(new Error('Timeout')), timeout)
        )
      ]);

      // Record successful latency
      const latency = Date.now() - start;
      this.recordLatency(latency);

      return result;

    } catch (error) {
      if (error.message === 'Timeout') {
        // Record timeout as max latency
        this.recordLatency(timeout);
      }

      throw error;
    }
  }

  calculateTimeout() {
    if (this.latencies.length === 0) {
      return this.initialTimeout;
    }

    // Use P99 latency + buffer
    const p99 = this.percentile(99);
    const timeout = p99 * 1.5; // 50% buffer

    // Clamp to min/max
    return Math.max(
      this.minTimeout,
      Math.min(this.maxTimeout, timeout)
    );
  }

  recordLatency(latency) {
    this.latencies.push(latency);

    // Keep only recent samples
    if (this.latencies.length > this.maxSamples) {
      this.latencies.shift();
    }
  }

  percentile(p) {
    const sorted = [...this.latencies].sort((a, b) => a - b);
    const index = Math.ceil((sorted.length * p) / 100) - 1;
    return sorted[index];
  }
}

// Usage
const adaptiveTimeout = new AdaptiveTimeout({
  initialTimeout: 1000,
  minTimeout: 500,
  maxTimeout: 10000
});

const result = await adaptiveTimeout.execute(async () => {
  return await apiCall();
});
```

---

## 8.6 Health Checks

Monitor service health and remove unhealthy instances from load balancer.

### Health Check Types

```
1. Liveness Check
   - Is the service alive?
   - Returns: 200 OK or 503 Unavailable

2. Readiness Check
   - Is the service ready to handle traffic?
   - Checks: DB connection, dependencies

3. Startup Check
   - Has the service started successfully?
   - Used during initialization
```

### Implementation

```javascript
class HealthChecker {
  constructor() {
    this.checks = new Map();
  }

  registerCheck(name, checkFn) {
    this.checks.set(name, checkFn);
  }

  async checkHealth() {
    const results = {};
    let overallHealthy = true;

    for (const [name, checkFn] of this.checks) {
      try {
        const result = await checkFn();

        results[name] = {
          status: 'healthy',
          ...result
        };

      } catch (error) {
        results[name] = {
          status: 'unhealthy',
          error: error.message
        };

        overallHealthy = false;
      }
    }

    return {
      status: overallHealthy ? 'healthy' : 'unhealthy',
      timestamp: new Date().toISOString(),
      checks: results
    };
  }
}

// Setup health checks
const healthChecker = new HealthChecker();

// Database check
healthChecker.registerCheck('database', async () => {
  const start = Date.now();

  await db.query('SELECT 1');

  const latency = Date.now() - start;

  return {
    latency: `${latency}ms`,
    healthy: latency < 1000
  };
});

// Cache check
healthChecker.registerCheck('cache', async () => {
  await redis.ping();

  return {
    healthy: true
  };
});

// External API check
healthChecker.registerCheck('paymentService', async () => {
  const response = await fetch('http://payment-service/health', {
    timeout: 5000
  });

  return {
    healthy: response.ok,
    status: response.status
  };
});

// Disk space check
healthChecker.registerCheck('diskSpace', async () => {
  const usage = await checkDiskUsage();

  return {
    usage: `${usage}%`,
    healthy: usage < 90
  };
});

// Health endpoint
app.get('/health', async (req, res) => {
  const health = await healthChecker.checkHealth();

  const statusCode = health.status === 'healthy' ? 200 : 503;

  res.status(statusCode).json(health);
});

// Liveness endpoint (simple)
app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

// Readiness endpoint (checks dependencies)
app.get('/health/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');
    await redis.ping();

    res.status(200).json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});
```

### Graceful Shutdown

```javascript
class GracefulShutdown {
  constructor(server, options = {}) {
    this.server = server;
    this.shutdownTimeout = options.shutdownTimeout || 30000; // 30 seconds
    this.isShuttingDown = false;
  }

  setup() {
    // Handle shutdown signals
    process.on('SIGTERM', () => this.shutdown('SIGTERM'));
    process.on('SIGINT', () => this.shutdown('SIGINT'));
  }

  async shutdown(signal) {
    if (this.isShuttingDown) {
      return;
    }

    this.isShuttingDown = true;

    console.log(`Received ${signal}, starting graceful shutdown...`);

    // 1. Stop accepting new requests
    this.server.close(() => {
      console.log('Server stopped accepting new connections');
    });

    // 2. Wait for existing requests to complete
    const timeout = setTimeout(() => {
      console.log('Forcefully shutting down...');
      process.exit(1);
    }, this.shutdownTimeout);

    try {
      // 3. Close database connections
      await db.close();
      console.log('Database connections closed');

      // 4. Close cache connections
      await redis.quit();
      console.log('Cache connections closed');

      // 5. Finish background jobs
      await jobQueue.close();
      console.log('Background jobs completed');

      clearTimeout(timeout);
      console.log('Graceful shutdown completed');
      process.exit(0);

    } catch (error) {
      console.error('Error during shutdown:', error);
      process.exit(1);
    }
  }
}

// Usage
const gracefulShutdown = new GracefulShutdown(server, {
  shutdownTimeout: 30000
});

gracefulShutdown.setup();
```

---

## 8.7 Graceful Degradation

Provide reduced functionality when dependencies fail.

### Fallback Strategies

```javascript
class FallbackService {
  constructor(primaryService, options = {}) {
    this.primaryService = primaryService;
    this.cache = options.cache;
    this.defaultValue = options.defaultValue;
    this.circuitBreaker = new CircuitBreaker(options);
  }

  async execute(operation) {
    try {
      // Try primary service
      const result = await this.circuitBreaker.execute(async () => {
        return await this.primaryService[operation.method](...operation.args);
      });

      // Cache successful result
      if (this.cache && operation.cacheKey) {
        await this.cache.set(operation.cacheKey, result, 3600);
      }

      return result;

    } catch (error) {
      console.log('Primary service failed, trying fallback');

      // Fallback 1: Return cached data
      if (this.cache && operation.cacheKey) {
        const cached = await this.cache.get(operation.cacheKey);

        if (cached) {
          console.log('Using cached data');
          return { ...cached, fromCache: true };
        }
      }

      // Fallback 2: Return default value
      if (this.defaultValue) {
        console.log('Using default value');
        return this.defaultValue;
      }

      // Fallback 3: Return degraded response
      if (operation.degradedResponse) {
        console.log('Using degraded response');
        return operation.degradedResponse;
      }

      // No fallback available
      throw error;
    }
  }
}

// Usage: Product recommendation service
const recommendationService = new FallbackService(
  primaryRecommendationService,
  {
    cache: redisCache,
    defaultValue: {
      recommendations: [],
      message: 'Recommendations unavailable'
    }
  }
);

const recommendations = await recommendationService.execute({
  method: 'getRecommendations',
  args: [userId],
  cacheKey: `recommendations:${userId}`,
  degradedResponse: {
    recommendations: await getPopularProducts(), // Fallback to popular items
    message: 'Showing popular products'
  }
});
```

### Feature Flags for Degradation

```javascript
class FeatureFlags {
  constructor() {
    this.flags = new Map();
  }

  setFlag(name, enabled) {
    this.flags.set(name, enabled);
  }

  isEnabled(name) {
    return this.flags.get(name) !== false; // Default: enabled
  }

  async checkHealth() {
    // Automatically disable features based on health
    const dbHealth = await checkDatabaseHealth();
    if (!dbHealth.healthy) {
      this.setFlag('advanced-search', false);
      this.setFlag('real-time-updates', false);
    }

    const cacheHealth = await checkCacheHealth();
    if (!cacheHealth.healthy) {
      this.setFlag('personalization', false);
    }
  }
}

const featureFlags = new FeatureFlags();

// Check and update feature flags periodically
setInterval(() => {
  featureFlags.checkHealth();
}, 30000); // Every 30 seconds

// Use in application
app.get('/api/search', async (req, res) => {
  if (featureFlags.isEnabled('advanced-search')) {
    // Full search with filters, sorting, facets
    return await advancedSearch(req.query);
  } else {
    // Simple keyword search only
    return await basicSearch(req.query.q);
  }
});

app.get('/api/feed', async (req, res) => {
  if (featureFlags.isEnabled('personalization')) {
    // Personalized feed
    return await getPersonalizedFeed(req.user.id);
  } else {
    // Generic trending feed
    return await getTrendingFeed();
  }
});
```

---

## 8.8 Chaos Engineering

Proactively inject failures to test resilience.

### Chaos Monkey

```javascript
class ChaosMonkey {
  constructor(options = {}) {
    this.enabled = options.enabled !== false;
    this.failureRate = options.failureRate || 0.01; // 1% failure rate
    this.latencyRate = options.latencyRate || 0.05; // 5% slow requests
    this.latencyMs = options.latencyMs || 5000; // 5 second delay
  }

  async maybeInjectFailure(operation) {
    if (!this.enabled) {
      return await operation();
    }

    const random = Math.random();

    // Inject failure
    if (random < this.failureRate) {
      console.log('Chaos Monkey: Injecting failure');
      throw new Error('Chaos Monkey: Simulated failure');
    }

    // Inject latency
    if (random < this.failureRate + this.latencyRate) {
      const delay = Math.random() * this.latencyMs;
      console.log(`Chaos Monkey: Injecting ${delay}ms latency`);
      await sleep(delay);
    }

    return await operation();
  }

  // Specific chaos scenarios
  async killRandomInstance(instances) {
    if (!this.enabled) return;

    const victim = instances[Math.floor(Math.random() * instances.length)];
    console.log(`Chaos Monkey: Killing instance ${victim.id}`);

    // Simulate instance failure
    victim.kill();
  }

  async injectNetworkPartition(service1, service2) {
    if (!this.enabled) return;

    console.log(`Chaos Monkey: Creating network partition between ${service1} and ${service2}`);

    // Simulate network partition (block communication)
    // In real scenario, use iptables or network policies
  }

  async fillDisk(percentage) {
    if (!this.enabled) return;

    console.log(`Chaos Monkey: Filling disk to ${percentage}%`);

    // Create large file to fill disk
    // Monitor how application handles low disk space
  }
}

// Usage in middleware
const chaosMonkey = new ChaosMonkey({
  enabled: process.env.NODE_ENV !== 'production',
  failureRate: 0.01,
  latencyRate: 0.05,
  latencyMs: 5000
});

app.use(async (req, res, next) => {
  try {
    await chaosMonkey.maybeInjectFailure(async () => {
      next();
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Scheduled chaos experiments
setInterval(() => {
  if (Math.random() < 0.1) { // 10% chance per hour
    chaosMonkey.killRandomInstance(serviceInstances);
  }
}, 3600000); // Every hour
```

### Testing Resilience

```javascript
class ResilienceTest {
  async testCircuitBreaker() {
    console.log('Testing circuit breaker...');

    const breaker = new CircuitBreaker({
      failureThreshold: 3,
      timeout: 5000
    });

    // Simulate failures
    for (let i = 0; i < 5; i++) {
      try {
        await breaker.execute(async () => {
          throw new Error('Simulated failure');
        });
      } catch (error) {
        console.log(`Attempt ${i + 1}: ${error.message}`);
      }
    }

    console.log('Circuit breaker state:', breaker.getState());

    // Verify circuit is open
    assert(breaker.state === 'OPEN', 'Circuit should be OPEN');
  }

  async testBulkhead() {
    console.log('Testing bulkhead...');

    const bulkhead = new Bulkhead('TestService', {
      maxConcurrent: 2,
      maxQueue: 2
    });

    // Saturate bulkhead
    const promises = [];

    for (let i = 0; i < 10; i++) {
      promises.push(
        bulkhead.execute(async () => {
          await sleep(1000);
          return i;
        }).catch(err => ({ error: err.message }))
      );
    }

    const results = await Promise.all(promises);

    // Verify some requests were rejected
    const rejected = results.filter(r => r.error).length;
    console.log(`Rejected ${rejected} requests`);

    assert(rejected > 0, 'Some requests should be rejected');
  }

  async testGracefulDegradation() {
    console.log('Testing graceful degradation...');

    const service = new FallbackService(failingService, {
      cache: mockCache,
      defaultValue: { status: 'degraded' }
    });

    // Primary service will fail
    const result = await service.execute({
      method: 'getData',
      args: ['test']
    });

    // Verify fallback was used
    assert(result.status === 'degraded', 'Should return fallback');
  }

  async runAll() {
    await this.testCircuitBreaker();
    await this.testBulkhead();
    await this.testGracefulDegradation();

    console.log('All resilience tests passed!');
  }
}

// Run tests
const tests = new ResilienceTest();
await tests.runAll();
```

---

## 8.9 High Availability Strategies

### Multi-Region Deployment

```
┌────────────────────────────────────────────────┐
│              Global Load Balancer              │
│         (Route53 / Cloudflare)                 │
└────────────┬───────────────┬───────────────────┘
             │               │
    ┌────────▼─────┐   ┌────▼──────────┐
    │  US Region   │   │  EU Region    │
    │              │   │               │
    │  ┌─────┐    │   │  ┌─────┐     │
    │  │ LB  │    │   │  │ LB  │     │
    │  └──┬──┘    │   │  └──┬──┘     │
    │     │       │   │     │         │
    │  ┌──▼──┐    │   │  ┌──▼──┐     │
    │  │App  │    │   │  │App  │     │
    │  └──┬──┘    │   │  └──┬──┘     │
    │     │       │   │     │         │
    │  ┌──▼──┐    │   │  ┌──▼──┐     │
    │  │ DB  │◄───┼───┼──┤ DB  │     │
    │  └─────┘    │   │  └─────┘     │
    │ (Primary)   │   │  (Replica)   │
    └─────────────┘   └──────────────┘
```

### Active-Active vs Active-Passive

```javascript
// Active-Active: Both regions serve traffic
class ActiveActiveDeployment {
  constructor() {
    this.regions = ['us-east', 'eu-west', 'asia-southeast'];
  }

  routeRequest(request) {
    // Route based on user location
    const userRegion = this.getUserRegion(request);

    // Route to nearest region
    return this.regions.find(r => r === userRegion) || this.regions[0];
  }

  getUserRegion(request) {
    // Determine from IP geolocation
    const ip = request.headers['x-forwarded-for'];
    return geoip.lookup(ip).region;
  }
}

// Active-Passive: One region active, others standby
class ActivePassiveDeployment {
  constructor() {
    this.primary = 'us-east';
    this.secondary = 'eu-west';
  }

  async routeRequest(request) {
    // Check primary health
    const primaryHealthy = await this.checkHealth(this.primary);

    if (primaryHealthy) {
      return this.primary;
    }

    // Failover to secondary
    console.log('Primary unhealthy, failing over to secondary');
    return this.secondary;
  }

  async checkHealth(region) {
    try {
      const response = await fetch(`https://${region}.api.example.com/health`);
      return response.ok;
    } catch (error) {
      return false;
    }
  }
}
```

### Database Replication

```javascript
class DatabaseFailover {
  constructor() {
    this.primary = 'db-primary';
    this.replicas = ['db-replica-1', 'db-replica-2'];
    this.currentPrimary = this.primary;
  }

  async query(sql) {
    try {
      return await this.executeQuery(this.currentPrimary, sql);
    } catch (error) {
      console.error('Primary failed:', error);

      // Attempt failover
      await this.failover();

      // Retry on new primary
      return await this.executeQuery(this.currentPrimary, sql);
    }
  }

  async failover() {
    console.log('Initiating database failover...');

    // Promote first healthy replica to primary
    for (const replica of this.replicas) {
      const healthy = await this.checkHealth(replica);

      if (healthy) {
        await this.promoteReplica(replica);
        this.currentPrimary = replica;

        console.log(`Promoted ${replica} to primary`);
        return;
      }
    }

    throw new Error('No healthy replicas available for failover');
  }

  async promoteReplica(replica) {
    // Execute promotion command
    await this.executeQuery(replica, 'PROMOTE TO PRIMARY');

    // Update DNS/load balancer
    await this.updateDNS(replica);
  }
}
```

---

## 8.10 Key Takeaways

### Reliability Patterns Summary

```
┌─────────────────────┬─────────────────────────────┐
│ Pattern             │ Purpose                     │
├─────────────────────┼─────────────────────────────┤
│ Circuit Breaker     │ Fail fast, prevent cascade  │
│ Retry + Backoff     │ Handle transient failures   │
│ Bulkhead            │ Isolate resources           │
│ Timeout             │ Prevent indefinite waiting  │
│ Health Check        │ Monitor service health      │
│ Graceful Degradation│ Reduced functionality       │
│ Chaos Engineering   │ Test resilience             │
└─────────────────────┴─────────────────────────────┘
```

### Design Principles

```
1. Design for Failure
   - Assume everything fails
   - Plan recovery strategies

2. Fail Fast
   - Don't wait for timeouts
   - Use circuit breakers

3. Isolate Failures
   - Use bulkheads
   - Prevent cascading

4. Degrade Gracefully
   - Provide fallbacks
   - Reduce functionality vs total failure

5. Monitor Everything
   - Know when failures occur
   - Alert proactively

6. Test Failure Scenarios
   - Use chaos engineering
   - Regular resilience testing
```

### Reliability Metrics

```
Target Availability:
- 99%    → 3.65 days/year downtime
- 99.9%  → 8.76 hours/year downtime
- 99.99% → 52.56 minutes/year downtime
- 99.999% → 5.26 minutes/year downtime

How to achieve 99.99%:
✓ No single point of failure
✓ Automatic failover
✓ Health checks
✓ Circuit breakers
✓ Multi-region deployment
✓ Comprehensive monitoring
```

---

## 8.11 Interview Questions

### Basic:
1. What is the Circuit Breaker pattern?
2. Why do we need retry mechanisms?
3. What is graceful degradation?

### Intermediate:
1. Explain exponential backoff and why it's better than fixed delay.
2. What is the Bulkhead pattern and when should you use it?
3. How do health checks help with high availability?

### Advanced:
1. Design a resilient payment processing system.
2. How would you implement chaos engineering in production?
3. Explain strategies for achieving 99.99% availability.

---

**Next Chapter:** [Chapter 9: System Design Interviews](./SystemDesign-09-Interviews.md)

In the final chapter, we'll cover:
- Interview framework (REDSAD)
- Common system design questions
- How to approach design problems
- Evaluation criteria
- Mock interview examples
- Tips and best practices
