# Chapter 2: Scalability & Performance

## 2.1 Understanding Scalability

Scalability is the capability of a system to handle increased load by adding resources. For senior engineers, it's not just about "making things faster"—it's about designing systems that gracefully handle 10x, 100x, or even 1000x growth.

### The Scalability Mindset

```
Current: 1,000 users
Question: "How will this work with 1,000,000 users?"

Current: 100 requests/second
Question: "What breaks at 10,000 requests/second?"

Current: 10 GB database
Question: "How do we handle 10 TB?"
```

### Scalability vs Performance

```
┌─────────────────────────────────────────────────────┐
│ Performance: How fast for single user               │
│ Example: API responds in 100ms                      │
│                                                     │
│ Scalability: How system handles growing load       │
│ Example: API still responds in 100ms with          │
│          1000x more users                          │
└─────────────────────────────────────────────────────┘
```

**Important**: A fast system isn't necessarily scalable!

```
Example: Single high-performance server
- Handles 1000 req/s with 50ms latency ✓ Fast!
- Cannot handle 100,000 req/s ✗ Not scalable!

Better: Distributed system with load balancer
- Each server handles 1000 req/s with 100ms latency
- Can scale to 100 servers → 100,000 req/s ✓ Scalable!
```

---

## 2.2 Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)

**Definition**: Adding more resources (CPU, RAM, SSD) to existing server.

```
┌─────────────────────────────────────┐
│           Before                    │
│   ┌────────────────────┐           │
│   │  Server             │           │
│   │  4 CPU cores        │           │
│   │  8 GB RAM           │           │
│   │  100 GB SSD         │           │
│   │                     │           │
│   │  Handles:           │           │
│   │  1000 req/s         │           │
│   └────────────────────┘           │
└─────────────────────────────────────┘

                 ↓ Scale Up

┌─────────────────────────────────────┐
│           After                     │
│   ┌────────────────────┐           │
│   │  Server             │           │
│   │  16 CPU cores   ↑   │           │
│   │  64 GB RAM      ↑   │           │
│   │  1 TB SSD       ↑   │           │
│   │                     │           │
│   │  Handles:           │           │
│   │  4000 req/s         │           │
│   └────────────────────┘           │
└─────────────────────────────────────┘
```

**Pros:**
- ✅ Simple to implement (no code changes)
- ✅ No distributed system complexity
- ✅ Maintains data consistency
- ✅ Lower latency (no network calls)

**Cons:**
- ❌ Hard limit (can't scale infinitely)
- ❌ Expensive (costs grow exponentially)
- ❌ Downtime during upgrades
- ❌ Single point of failure
- ❌ Geographic limitations

**When to Use:**
- Early stage systems (< 10,000 users)
- Databases (before sharding)
- Monolithic applications
- When complexity isn't worth it yet

**Real-World Example: Stack Overflow**

Stack Overflow famously runs on just a few powerful servers:
```
2013 Architecture:
- 2 web servers (11 million page views/month)
- 2 SQL servers
- 1 Redis server

Why vertical scaling works for them:
✓ Mature, optimized codebase
✓ Heavy caching (hit rate > 95%)
✓ Not rapidly growing
✓ Team expertise in optimization
```

### Horizontal Scaling (Scale Out)

**Definition**: Adding more servers to distribute load.

```
┌──────────────────────────────────────────┐
│           Before                         │
│        ┌────────────┐                    │
│        │  Server 1  │                    │
│        │  1000 req/s│                    │
│        └────────────┘                    │
└──────────────────────────────────────────┘

                 ↓ Scale Out

┌──────────────────────────────────────────┐
│           After                          │
│      ┌─────────────────┐                │
│      │ Load Balancer   │                │
│      └────────┬────────┘                │
│               │                          │
│     ┌─────────┼─────────┐               │
│     │         │         │               │
│  ┌──▼──┐   ┌──▼──┐   ┌──▼──┐           │
│  │Srv 1│   │Srv 2│   │Srv 3│           │
│  │333  │   │333  │   │333  │           │
│  │req/s│   │req/s│   │req/s│           │
│  └─────┘   └─────┘   └─────┘           │
│                                          │
│  Total: 1000 req/s (can add more!)      │
└──────────────────────────────────────────┘
```

**Pros:**
- ✅ Nearly unlimited scaling
- ✅ Better fault tolerance (redundancy)
- ✅ Cost-effective (use commodity hardware)
- ✅ No downtime for scaling
- ✅ Geographic distribution possible

**Cons:**
- ❌ Complex architecture
- ❌ Data consistency challenges
- ❌ Requires load balancing
- ❌ More moving parts to manage
- ❌ Higher operational complexity

**When to Use:**
- High-traffic systems (> 100,000 users)
- Need high availability
- Global user base
- Unpredictable growth
- Stateless services

**Real-World Example: Netflix**

```
Netflix Architecture (Simplified):
- 1000+ microservices
- Deployed across 3 AWS regions
- Auto-scaling based on demand
- Can handle 200M+ subscribers

Why horizontal scaling is essential:
✓ Massive global scale
✓ Variable load (evening peak)
✓ Need high availability (99.99%+)
✓ Geographic distribution
```

### Comparison Table

```
┌──────────────────┬───────────────────┬────────────────────┐
│ Aspect           │ Vertical          │ Horizontal         │
├──────────────────┼───────────────────┼────────────────────┤
│ Complexity       │ Low               │ High               │
│ Cost (small)     │ Low               │ Higher             │
│ Cost (large)     │ Very High         │ Moderate           │
│ Scalability Limit│ Hardware limits   │ Nearly unlimited   │
│ Availability     │ Single point fail │ High (redundancy)  │
│ Consistency      │ Easy              │ Challenging        │
│ Downtime         │ Required          │ Zero downtime      │
│ Implementation   │ Hours             │ Weeks/Months       │
└──────────────────┴───────────────────┴────────────────────┘
```

---

## 2.3 Load Balancing

Load balancers distribute traffic across multiple servers to ensure no single server is overwhelmed.

### Load Balancer Types

```
┌────────────────────────────────────────────────────┐
│  Layer 4 (Transport Layer) Load Balancing          │
│  - Works with TCP/UDP                              │
│  - Routes based on IP address and port             │
│  - Very fast (no packet inspection)                │
│  - Example: AWS Network Load Balancer (NLB)        │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  Layer 7 (Application Layer) Load Balancing        │
│  - Works with HTTP/HTTPS                           │
│  - Routes based on URL, headers, cookies           │
│  - Slower (inspects content)                       │
│  - More intelligent routing                        │
│  - Example: AWS Application Load Balancer (ALB)    │
└────────────────────────────────────────────────────┘
```

### Load Balancing Algorithms

#### 1. Round Robin

**How it works**: Distribute requests sequentially to each server.

```
Request Flow:
┌─────────┐
│ Client  │
└────┬────┘
     │ Req 1 → Server 1
     │ Req 2 → Server 2
     │ Req 3 → Server 3
     │ Req 4 → Server 1
     │ Req 5 → Server 2
     └─ Req 6 → Server 3
```

**Implementation:**
```javascript
class RoundRobinLoadBalancer {
  constructor(servers) {
    this.servers = servers;
    this.currentIndex = 0;
  }

  getNextServer() {
    const server = this.servers[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.servers.length;
    return server;
  }
}

// Usage
const lb = new RoundRobinLoadBalancer([
  'server1.example.com',
  'server2.example.com',
  'server3.example.com'
]);

console.log(lb.getNextServer()); // server1
console.log(lb.getNextServer()); // server2
console.log(lb.getNextServer()); // server3
console.log(lb.getNextServer()); // server1
```

**When to use:**
- ✅ Servers have similar capacity
- ✅ Requests have similar processing time
- ✅ Simple, predictable distribution

**When NOT to use:**
- ❌ Servers have different capacities
- ❌ Requests vary significantly in processing time

#### 2. Weighted Round Robin

**How it works**: Servers with higher capacity receive more requests.

```
Server Capacities:
- Server 1: Weight 3 (powerful)
- Server 2: Weight 1 (standard)

Distribution:
Req 1 → Server 1
Req 2 → Server 1
Req 3 → Server 1
Req 4 → Server 2
Req 5 → Server 1 (cycle repeats)
```

**Implementation:**
```javascript
class WeightedRoundRobinLoadBalancer {
  constructor(servers) {
    // servers = [{ url: 'server1', weight: 3 }, ...]
    this.servers = [];

    // Expand servers based on weight
    servers.forEach(server => {
      for (let i = 0; i < server.weight; i++) {
        this.servers.push(server.url);
      }
    });

    this.currentIndex = 0;
  }

  getNextServer() {
    const server = this.servers[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.servers.length;
    return server;
  }
}

// Usage
const lb = new WeightedRoundRobinLoadBalancer([
  { url: 'server1.example.com', weight: 5 },  // Powerful
  { url: 'server2.example.com', weight: 3 },  // Medium
  { url: 'server3.example.com', weight: 1 }   // Basic
]);
```

#### 3. Least Connections

**How it works**: Route to server with fewest active connections.

```
State at time T:
┌─────────────────────────────────────┐
│ Server 1: 10 active connections     │
│ Server 2: 5 active connections  ←   │ Choose this!
│ Server 3: 15 active connections     │
└─────────────────────────────────────┘
```

**Implementation:**
```javascript
class LeastConnectionsLoadBalancer {
  constructor(servers) {
    this.servers = servers.map(url => ({
      url,
      activeConnections: 0
    }));
  }

  getNextServer() {
    // Find server with least connections
    const server = this.servers.reduce((min, current) =>
      current.activeConnections < min.activeConnections ? current : min
    );

    return server.url;
  }

  incrementConnections(serverUrl) {
    const server = this.servers.find(s => s.url === serverUrl);
    if (server) server.activeConnections++;
  }

  decrementConnections(serverUrl) {
    const server = this.servers.find(s => s.url === serverUrl);
    if (server && server.activeConnections > 0) {
      server.activeConnections--;
    }
  }
}

// Usage
const lb = new LeastConnectionsLoadBalancer([
  'server1.example.com',
  'server2.example.com',
  'server3.example.com'
]);

async function handleRequest(req) {
  const server = lb.getNextServer();
  lb.incrementConnections(server);

  try {
    const response = await forwardToServer(server, req);
    return response;
  } finally {
    lb.decrementConnections(server);
  }
}
```

**When to use:**
- ✅ Requests have varying processing times
- ✅ Long-lived connections (WebSockets)
- ✅ Servers have similar capacity

#### 4. IP Hash

**How it works**: Route based on client IP address (same client → same server).

```
hash(ClientIP) % NumberOfServers = ServerIndex

Client 192.168.1.100 → hash % 3 = 1 → Server 1
Client 192.168.1.101 → hash % 3 = 2 → Server 2
Client 192.168.1.102 → hash % 3 = 0 → Server 0

Same client always goes to same server!
```

**Implementation:**
```javascript
class IPHashLoadBalancer {
  constructor(servers) {
    this.servers = servers;
  }

  getNextServer(clientIP) {
    const hash = this.hashIP(clientIP);
    const serverIndex = hash % this.servers.length;
    return this.servers[serverIndex];
  }

  hashIP(ip) {
    // Simple hash function (production: use better hash)
    let hash = 0;
    for (let i = 0; i < ip.length; i++) {
      hash = ((hash << 5) - hash) + ip.charCodeAt(i);
      hash = hash & hash; // Convert to 32-bit integer
    }
    return Math.abs(hash);
  }
}

// Usage
const lb = new IPHashLoadBalancer([
  'server1.example.com',
  'server2.example.com',
  'server3.example.com'
]);

app.use((req, res, next) => {
  const clientIP = req.ip;
  const server = lb.getNextServer(clientIP);

  // Forward to selected server
  proxy.web(req, res, { target: server });
});
```

**When to use:**
- ✅ Need session stickiness
- ✅ Server-side caching per user
- ✅ Consistent routing required

**Problem: Server Addition/Removal**
```
3 Servers: hash % 3
Client A → Server 1

Add 1 server (now 4):
Client A → hash % 4 → Server 2 (changed!)

Solution: Consistent Hashing (see below)
```

#### 5. Consistent Hashing

**Problem with simple hash**: Adding/removing servers redistributes ALL keys.

**Solution**: Hash both servers and keys onto a ring.

```
Consistent Hash Ring:

         Server A (hash=0)
              ↓
        0────────→ 90
        ↑          ↓
      360 ←─────── 180
        ↑          ↓
      270 ←──────→ 180
              ↑
         Server B (hash=90)
         Server C (hash=180)

Key "user123" → hash=45
Clockwise to next server → Server B

Add Server D (hash=135):
Only keys between 90-135 move to D
Keys at other positions unchanged!
```

**Implementation:**
```javascript
class ConsistentHashLoadBalancer {
  constructor(servers, virtualNodes = 150) {
    this.ring = new Map();
    this.servers = servers;
    this.virtualNodes = virtualNodes;

    // Add servers to ring with virtual nodes
    servers.forEach(server => {
      for (let i = 0; i < virtualNodes; i++) {
        const hash = this.hash(`${server}:${i}`);
        this.ring.set(hash, server);
      }
    });

    // Sort ring by hash value
    this.sortedHashes = Array.from(this.ring.keys()).sort((a, b) => a - b);
  }

  getNextServer(key) {
    const hash = this.hash(key);

    // Find first server clockwise from hash
    for (const serverHash of this.sortedHashes) {
      if (serverHash >= hash) {
        return this.ring.get(serverHash);
      }
    }

    // Wrap around to first server
    return this.ring.get(this.sortedHashes[0]);
  }

  hash(key) {
    // Simple hash (production: use MD5 or SHA1)
    let hash = 0;
    for (let i = 0; i < key.length; i++) {
      hash = ((hash << 5) - hash) + key.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  addServer(server) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const hash = this.hash(`${server}:${i}`);
      this.ring.set(hash, server);
    }
    this.sortedHashes = Array.from(this.ring.keys()).sort((a, b) => a - b);
  }

  removeServer(server) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const hash = this.hash(`${server}:${i}`);
      this.ring.delete(hash);
    }
    this.sortedHashes = Array.from(this.ring.keys()).sort((a, b) => a - b);
  }
}

// Usage
const lb = new ConsistentHashLoadBalancer([
  'server1.example.com',
  'server2.example.com',
  'server3.example.com'
]);

// Route request
const server = lb.getNextServer('user:12345');

// Add new server (minimal redistribution)
lb.addServer('server4.example.com');
```

**Benefits:**
- ✅ Adding server: Only ~1/N keys redistributed
- ✅ Removing server: Only that server's keys move
- ✅ Predictable, consistent routing

**Real-World Usage:**
- Amazon DynamoDB (data partitioning)
- Memcached (cache distribution)
- Cassandra (data distribution)

### Load Balancer Architecture

**Single Load Balancer (Not Recommended):**
```
           ┌──────────────┐
           │Load Balancer │ ← Single Point of Failure!
           └──────┬───────┘
                  │
        ┌─────────┼─────────┐
        │         │         │
     Server1   Server2   Server3
```

**Highly Available Load Balancer (Recommended):**
```
         ┌─────────────────┐
         │   DNS (Route53) │
         └────────┬─────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
   ┌────▼──────┐      ┌─────▼─────┐
   │ LB Active │      │ LB Standby│
   │ (Primary) │◄────►│ (Backup)  │
   └────┬──────┘      └───────────┘
        │              Heartbeat
        │
   ┌────┼─────┐
   │    │     │
Server1 Server2 Server3
```

---

## 2.4 Content Delivery Network (CDN)

CDN is a geographically distributed network of servers that cache content close to users.

### How CDN Works

```
Without CDN:
┌──────────┐                           ┌──────────┐
│ User in  │  ──── 5000 km ───────→   │ Server   │
│ Asia     │  ←─── 500ms latency ───  │ in USA   │
└──────────┘                           └──────────┘

With CDN:
┌──────────┐        ┌──────────────┐
│ User in  │  ──→   │ CDN Edge in  │  (Cache Hit)
│ Asia     │  ←──   │ Tokyo (50ms) │
└──────────┘        └──────┬───────┘
                           │
                           │ (Cache Miss)
                           │
                    ┌──────▼───────┐
                    │ Origin Server│
                    │ in USA       │
                    └──────────────┘
```

### CDN Architecture

```
             ┌─────────────────┐
             │ Origin Server    │
             │ (Your backend)   │
             └────────┬─────────┘
                      │
          ┌───────────┼───────────┐
          │           │           │
    ┌─────▼────┐ ┌───▼────┐ ┌───▼────┐
    │ CDN Edge │ │CDN Edge│ │CDN Edge│
    │ US-East  │ │EU-West │ │Asia-SE │
    └────┬─────┘ └───┬────┘ └───┬────┘
         │           │           │
    Users in US  Users in EU  Users in Asia
```

### What to Cache on CDN

```
✅ Static Assets:
- Images (JPEG, PNG, WebP)
- Videos
- CSS, JavaScript files
- Fonts
- PDFs, documents

✅ Semi-Static Content:
- HTML pages (with short TTL)
- API responses (for public data)
- RSS feeds

❌ Don't Cache:
- User-specific data
- Real-time data
- Authentication tokens
- Checkout/payment pages
```

### CDN Cache Headers

```javascript
// Express.js example: Setting cache headers
app.get('/api/products', (req, res) => {
  // Cache for 5 minutes
  res.set('Cache-Control', 'public, max-age=300');
  res.json(products);
});

app.get('/images/:filename', (req, res) => {
  // Cache for 1 year (immutable)
  res.set('Cache-Control', 'public, max-age=31536000, immutable');
  res.sendFile(filename);
});

app.get('/api/user/profile', (req, res) => {
  // Don't cache user-specific data
  res.set('Cache-Control', 'private, no-cache, no-store, must-revalidate');
  res.json(userProfile);
});
```

### Cache Invalidation

**Problem**: How to update cached content when origin changes?

**Solutions:**

1. **Time-Based (TTL)**
```
Set Cache-Control: max-age=3600

Content expires after 1 hour, CDN fetches fresh copy
```

2. **Versioned URLs**
```
Old: /static/app.js
New: /static/app.v2.js  or  /static/app.js?v=2

Each version is a new URL, cached separately
```

3. **Cache Purge (Invalidation API)**
```javascript
// CloudFront invalidation
const cloudfront = new AWS.CloudFront();

await cloudfront.createInvalidation({
  DistributionId: 'DISTRIBUTION_ID',
  InvalidationBatch: {
    Paths: {
      Quantity: 1,
      Items: ['/images/logo.png']
    },
    CallerReference: Date.now().toString()
  }
}).promise();
```

### Real-World Example: Netflix

```
Netflix CDN Strategy:

1. Own CDN (Open Connect)
   - 1000+ servers in ISP data centers
   - Content pre-positioned during off-peak
   - Serves 95%+ of traffic

2. Multi-tiered Caching:
   ┌──────────────┐
   │ Origin       │ (Master copy)
   └──────┬───────┘
          │
   ┌──────▼────────┐
   │ Regional Cache│ (Popular content)
   └──────┬────────┘
          │
   ┌──────▼────────┐
   │ Edge Cache    │ (All content)
   └───────────────┘

3. Adaptive Bitrate Streaming:
   - Multiple quality levels cached
   - Client selects based on bandwidth
```

**Results:**
- 200M+ subscribers served globally
- < 100ms start time for most users
- 99.99% availability
- Massive cost savings (vs third-party CDN)

---

## 2.5 Caching Strategies

Caching is the most effective way to improve performance and scalability.

### Where to Cache

```
Browser → CDN → Load Balancer → App Server → Database

Each layer can cache:

┌──────────────────────────────────────────────────┐
│ Browser Cache (localStorage, cookies)            │
│ - User preferences, auth tokens                  │
│ - Reduces server requests                        │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ CDN Cache (edge servers)                         │
│ - Static assets, images, videos                  │
│ - Closest to user (lowest latency)               │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ Application Cache (Redis, Memcached)             │
│ - API responses, computed results                │
│ - Shared across app servers                      │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ Database Cache (query cache, buffer pool)        │
│ - Frequently accessed rows                       │
│ - Reduces disk I/O                               │
└──────────────────────────────────────────────────┘
```

### Application-Level Caching

**Redis Example:**

```javascript
const redis = require('redis');
const client = redis.createClient();

class ProductService {
  async getProduct(productId) {
    // Try cache first
    const cached = await client.get(`product:${productId}`);
    if (cached) {
      console.log('Cache hit!');
      return JSON.parse(cached);
    }

    // Cache miss - fetch from database
    console.log('Cache miss - querying database');
    const product = await db.products.findById(productId);

    // Store in cache for 1 hour
    await client.setex(
      `product:${productId}`,
      3600,
      JSON.stringify(product)
    );

    return product;
  }

  async updateProduct(productId, updates) {
    // Update database
    await db.products.update(productId, updates);

    // Invalidate cache
    await client.del(`product:${productId}`);
  }
}
```

### Cache Eviction Policies

When cache is full, which items to remove?

```
┌─────────────────────────────────────────────────┐
│ LRU (Least Recently Used)                       │
│ - Remove items not accessed recently            │
│ - Best for most use cases                       │
│ - Redis default                                 │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ LFU (Least Frequently Used)                     │
│ - Remove items accessed least often             │
│ - Good for popular items                        │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ FIFO (First In First Out)                       │
│ - Remove oldest items first                     │
│ - Simple but less effective                     │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ TTL (Time To Live)                              │
│ - Items expire after set time                   │
│ - Combines with other policies                  │
└─────────────────────────────────────────────────┘
```

### Cache Stampede Problem

**Problem**: Cache expires, many requests hit database simultaneously.

```
Time: T0 - Cache expires
Time: T1 - 1000 requests arrive simultaneously
         - All see cache miss
         - All query database
         - Database overloaded! 💥
```

**Solution 1: Locking**
```javascript
class CacheWithLock {
  async get(key, fetchFunction) {
    // Try cache
    const cached = await redis.get(key);
    if (cached) return JSON.parse(cached);

    // Acquire lock
    const lockKey = `lock:${key}`;
    const locked = await redis.set(lockKey, '1', 'NX', 'EX', 10);

    if (locked) {
      // This request won the lock - fetch data
      try {
        const data = await fetchFunction();
        await redis.setex(key, 3600, JSON.stringify(data));
        return data;
      } finally {
        await redis.del(lockKey);
      }
    } else {
      // Another request is fetching - wait and retry
      await new Promise(resolve => setTimeout(resolve, 100));
      return this.get(key, fetchFunction);
    }
  }
}
```

**Solution 2: Early Recomputation**
```javascript
// Refresh cache before expiry
class CacheWithRefresh {
  async get(key, fetchFunction, ttl = 3600) {
    const cached = await redis.get(key);

    if (cached) {
      const data = JSON.parse(cached);

      // Check if cache is expiring soon (within 10% of TTL)
      const ttlRemaining = await redis.ttl(key);
      if (ttlRemaining < ttl * 0.1) {
        // Refresh in background
        this.refreshAsync(key, fetchFunction, ttl);
      }

      return data;
    }

    // Cache miss - fetch synchronously
    const data = await fetchFunction();
    await redis.setex(key, ttl, JSON.stringify(data));
    return data;
  }

  async refreshAsync(key, fetchFunction, ttl) {
    // Don't await - runs in background
    fetchFunction().then(data => {
      redis.setex(key, ttl, JSON.stringify(data));
    });
  }
}
```

---

## 2.6 Real-World Scalability Case Studies

### Case Study 1: Twitter's Fanout Architecture

**Problem**: When @elonmusk tweets (100M+ followers), how to deliver to all timelines?

**Naive Approach (Pull Model):**
```
User opens app:
1. Query: "Get tweets from everyone I follow"
2. SELECT * FROM tweets
   WHERE user_id IN (following_list)
   ORDER BY timestamp DESC
   LIMIT 20

Problem: Query is slow (joins many users)
Can't scale to millions of users
```

**Twitter's Solution (Push Model - Fanout):**
```
When user tweets:
1. Write tweet to database
2. Fanout: Push to followers' timelines

┌─────────────────┐
│ User tweets     │
│ "Hello world!"  │
└────────┬────────┘
         │
    ┌────▼──────────────────┐
    │ Fanout Service        │
    │ (Async queue workers) │
    └────┬──────────────────┘
         │
         ├─ Push to Follower 1 timeline (Redis)
         ├─ Push to Follower 2 timeline (Redis)
         ├─ Push to Follower 3 timeline (Redis)
         └─ ... (100M followers)

When follower opens app:
- Read from pre-computed timeline (Redis)
- Super fast! (< 10ms)
```

**Hybrid Approach** (for celebrities):
```
Regular users (< 1M followers):
- Use push model (fanout on write)

Celebrities (> 1M followers):
- DON'T fanout on write (too slow)
- Merge tweets in real-time on read

Timeline = Pre-computed timeline + Celebrity tweets
```

**Implementation Sketch:**
```javascript
class TwitterTimeline {
  async postTweet(userId, content) {
    // Save tweet
    const tweet = await db.tweets.create({ userId, content });

    // Check if user is celebrity
    const followerCount = await db.users.getFollowerCount(userId);

    if (followerCount < 1000000) {
      // Regular user - fanout
      await this.fanoutToFollowers(userId, tweet);
    }
    // Celebrities: skip fanout (too expensive)

    return tweet;
  }

  async fanoutToFollowers(userId, tweet) {
    // Get followers (paginated)
    const followers = await db.users.getFollowers(userId);

    // Queue fanout jobs
    for (const followerId of followers) {
      await queue.add('fanout', {
        followerId,
        tweet
      });
    }
  }

  async getTimeline(userId) {
    // Get pre-computed timeline
    const timeline = await redis.lrange(`timeline:${userId}`, 0, 19);

    // Get celebrities user follows
    const celebrities = await db.users.getFollowedCelebrities(userId);

    // Fetch recent celebrity tweets
    const celebrityTweets = await db.tweets.find({
      userId: { $in: celebrities },
      timestamp: { $gt: Date.now() - 86400000 } // Last 24h
    });

    // Merge and sort
    const merged = [...timeline, ...celebrityTweets]
      .sort((a, b) => b.timestamp - a.timestamp)
      .slice(0, 20);

    return merged;
  }
}
```

### Case Study 2: Instagram's Photo Storage

**Challenge**: Store billions of photos efficiently.

**Requirements:**
- 100M photos uploaded per day
- Average size: 300 KB
- Storage: 100M × 300 KB = 30 TB per day!
- 5 years: 30 TB × 365 × 5 = 55 PB

**Solution: Tiered Storage**

```
┌────────────────────────────────────────────┐
│ Hot Storage (Recent photos - last 30 days) │
│ - Fast SSD storage                         │
│ - 100M × 30 × 300KB = 900 TB              │
│ - Cost: High, but accessed frequently     │
└────────────────────────────────────────────┘
              ↓ (aging)
┌────────────────────────────────────────────┐
│ Warm Storage (30 days - 1 year)           │
│ - Standard HDD storage                     │
│ - Accessed occasionally                    │
│ - Cost: Medium                             │
└────────────────────────────────────────────┘
              ↓ (aging)
┌────────────────────────────────────────────┐
│ Cold Storage (> 1 year)                    │
│ - Glacier / archival storage               │
│ - Rarely accessed                          │
│ - Cost: Very low                           │
└────────────────────────────────────────────┘
```

**Image Processing Pipeline:**

```
1. User uploads photo (5 MB)
         ↓
2. Resize & optimize
   - Original: 5 MB
   - Large: 1920px → 500 KB
   - Medium: 1080px → 200 KB
   - Thumbnail: 320px → 50 KB
   Total: 5.75 MB
         ↓
3. Store in CDN
   - Serve optimized version based on device
   - Mobile → Thumbnail
   - Desktop → Large
         ↓
4. Lazy loading
   - Load thumbnails first
   - Load full resolution on click
```

**Cost Savings:**
```
Before optimization:
- 100M photos/day × 5 MB = 500 TB/day
- AWS S3: $0.023/GB = $11,500/day

After optimization:
- Serve 80% from thumbnails (50 KB)
- CDN cache hit rate: 95%
- Storage: 100M × 575 KB = 57.5 TB/day
- Cost reduced by ~85%
```

---

## 2.7 Key Takeaways

### Scalability Principles

1. **Start Simple, Scale When Needed**
   - Don't over-engineer early
   - Vertical scaling first, horizontal when necessary
   - Measure before optimizing

2. **Cache Aggressively**
   - Cache at every layer
   - Cache hit rate > 80% for best results
   - Invalidate carefully

3. **Design for Failure**
   - Load balancers should be redundant
   - No single points of failure
   - Health checks and automatic failover

4. **Optimize for Common Case**
   - Twitter: Optimize reads (fanout on write)
   - Instagram: Optimize delivery (CDN, lazy load)
   - Identify your bottleneck first

### Performance Metrics to Track

```
┌─────────────────────┬──────────────┬─────────────┐
│ Metric              │ Good         │ Excellent   │
├─────────────────────┼──────────────┼─────────────┤
│ API Latency (p50)   │ < 200ms      │ < 100ms     │
│ API Latency (p99)   │ < 1000ms     │ < 500ms     │
│ Cache Hit Rate      │ > 80%        │ > 95%       │
│ Availability        │ 99.9%        │ 99.99%      │
│ Error Rate          │ < 1%         │ < 0.1%      │
└─────────────────────┴──────────────┴─────────────┘
```

### Common Scaling Mistakes

❌ **Premature Optimization**: Building for 1M users when you have 100
✅ **Do**: Start simple, measure, then scale

❌ **Caching Everything**: Including user-specific or real-time data
✅ **Do**: Cache static and semi-static content only

❌ **No Monitoring**: Can't improve what you don't measure
✅ **Do**: Monitor latency, throughput, error rates

❌ **Ignoring Database**: Scaling app servers but database is bottleneck
✅ **Do**: Profile queries, add indexes, consider read replicas

---

## 2.8 Interview Questions

### Basic:
1. What's the difference between vertical and horizontal scaling?
2. What is a load balancer and why do we need it?
3. Explain how CDN works.

### Intermediate:
1. Compare round-robin and least connections load balancing.
2. What is consistent hashing and when would you use it?
3. How do you handle cache invalidation?

### Advanced:
1. Design Twitter's timeline delivery system for 500M users.
2. How would you scale a write-heavy application?
3. Explain the trade-offs between fanout-on-write vs fanout-on-read.

---

**Next Chapter:** [Chapter 3: Database Design & Patterns](./SystemDesign-03-Database.md)

In the next chapter, we'll dive deep into:
- SQL vs NoSQL trade-offs
- Database sharding strategies
- Replication patterns
- Indexing optimization
- Real-world data modeling
