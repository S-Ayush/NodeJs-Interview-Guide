# Chapter 4: Caching Strategies

## 4.1 Caching Fundamentals

Caching is storing frequently accessed data in a faster storage layer to reduce latency and load on slower systems.

### Why Cache?

```
Without Cache:
User Request → App Server → Database (100ms)
Total: 100ms per request

With Cache:
User Request → App Server → Cache (1ms) → Return
Total: 1ms per request (100x faster!)

If cache miss:
User Request → App Server → Cache Miss → Database (100ms) → Store in Cache → Return
```

### The Caching Hierarchy

```
Speed & Cost (Fastest/Most Expensive → Slowest/Cheapest)

┌─────────────────────────────────────────────────┐
│ L1 Cache (CPU)                    0.5 ns        │
│ L2 Cache (CPU)                    7 ns          │
│ RAM                               100 ns        │
│ SSD                               150 μs        │
│ HDD                               10 ms         │
│ Network (same datacenter)         0.5 ms        │
│ Network (cross-region)            50-100 ms     │
└─────────────────────────────────────────────────┘

Application-Level Caching:
┌─────────────────────────────────────────────────┐
│ In-Memory (Local)                 < 1 ms        │
│ Redis/Memcached (Network)         1-5 ms        │
│ CDN (Edge)                        10-50 ms      │
│ Database                          50-200 ms     │
└─────────────────────────────────────────────────┘
```

### When to Cache

```
✓ Frequently accessed data (80/20 rule)
✓ Expensive to compute
✓ Slow to fetch from database
✓ Static or semi-static content
✓ Session data
✓ API responses

✗ Frequently changing data
✗ User-specific sensitive data
✗ Data requiring strong consistency
✗ Large datasets (consider partial caching)
```

---

## 4.2 Caching Patterns

### Pattern 1: Cache-Aside (Lazy Loading)

**Most common pattern.** Application manages cache explicitly.

```
Read Flow:
┌─────────┐
│  App    │
└────┬────┘
     │
     │ 1. Check cache
     ↓
┌─────────┐
│  Cache  │
└────┬────┘
     │
     ├─→ Cache Hit: Return data ✓
     │
     └─→ Cache Miss:
         2. Query database
         ↓
    ┌─────────┐
    │Database │
    └────┬────┘
         │
         3. Store in cache
         4. Return data
```

**Implementation:**

```javascript
class CacheAside {
  constructor(cache, database) {
    this.cache = cache;
    this.db = database;
  }

  async get(key) {
    // 1. Try cache first
    let value = await this.cache.get(key);

    if (value !== null) {
      console.log('Cache hit');
      return JSON.parse(value);
    }

    // 2. Cache miss - query database
    console.log('Cache miss - querying database');
    value = await this.db.query('SELECT * FROM items WHERE id = ?', [key]);

    if (value) {
      // 3. Store in cache with TTL (1 hour)
      await this.cache.setex(key, 3600, JSON.stringify(value));
    }

    return value;
  }

  async set(key, value) {
    // Write to database
    await this.db.query('INSERT INTO items VALUES (?, ?)', [key, value]);

    // Invalidate cache (or update it)
    await this.cache.del(key);
  }
}

// Usage
const cache = new CacheAside(redisClient, dbConnection);

// Read (cache-aside)
const product = await cache.get('product:123');

// Write (invalidate cache)
await cache.set('product:123', { name: 'iPhone', price: 999 });
```

**Pros:**
- ✅ Simple to implement
- ✅ Cache failures don't break system (fallback to DB)
- ✅ Data model in cache = data model in DB
- ✅ Only cache what's accessed (efficient)

**Cons:**
- ❌ Cache miss penalty (2 network calls: cache + DB)
- ❌ Stale data possible (if DB updated directly)
- ❌ Cache stampede risk (many requests miss cache simultaneously)

**Best for:**
- Read-heavy workloads
- Infrequently changing data
- When cache failures are tolerable

### Pattern 2: Write-Through

**Application writes to cache, cache writes to database synchronously.**

```
Write Flow:
┌─────────┐
│  App    │
└────┬────┘
     │
     │ 1. Write to cache
     ↓
┌─────────┐
│  Cache  │
└────┬────┘
     │
     │ 2. Cache writes to DB (sync)
     ↓
┌─────────┐
│Database │
└─────────┘
     │
     │ 3. Acknowledge
     └────────────────→ App
```

**Implementation:**

```javascript
class WriteThrough {
  constructor(cache, database) {
    this.cache = cache;
    this.db = database;
  }

  async get(key) {
    // Always read from cache
    const value = await this.cache.get(key);
    return value ? JSON.parse(value) : null;
  }

  async set(key, value) {
    // 1. Write to database first (ensure durability)
    await this.db.query('INSERT INTO items VALUES (?, ?)', [key, value]);

    // 2. Write to cache
    await this.cache.set(key, JSON.stringify(value));

    return { success: true };
  }
}

// Usage
const cache = new WriteThrough(redisClient, dbConnection);

// Write (write-through)
await cache.set('product:123', { name: 'iPhone', price: 999 });

// Read (always from cache)
const product = await cache.get('product:123');
```

**Pros:**
- ✅ Cache always consistent with database
- ✅ No stale data
- ✅ Simple read logic (always from cache)

**Cons:**
- ❌ Write latency (must write to both cache and DB)
- ❌ Unused data in cache (writes everything)
- ❌ Cache failure breaks writes

**Best for:**
- Read-heavy with occasional writes
- When consistency is critical
- Small dataset that fits in cache

### Pattern 3: Write-Behind (Write-Back)

**Application writes to cache, cache writes to database asynchronously.**

```
Write Flow:
┌─────────┐
│  App    │
└────┬────┘
     │
     │ 1. Write to cache
     ↓
┌─────────┐
│  Cache  │
└────┬────┘
     │
     │ 2. Return immediately ✓
     │
     │ 3. Async write to DB (later)
     ↓
┌─────────┐
│Database │
└─────────┘
```

**Implementation:**

```javascript
class WriteBehind {
  constructor(cache, database) {
    this.cache = cache;
    this.db = database;
    this.writeQueue = [];
    this.flushInterval = 5000; // Flush every 5 seconds

    // Start background flusher
    this.startFlusher();
  }

  async get(key) {
    const value = await this.cache.get(key);
    return value ? JSON.parse(value) : null;
  }

  async set(key, value) {
    // 1. Write to cache immediately
    await this.cache.set(key, JSON.stringify(value));

    // 2. Queue for database write
    this.writeQueue.push({ key, value });

    // 3. Return immediately (don't wait for DB)
    return { success: true };
  }

  startFlusher() {
    setInterval(async () => {
      if (this.writeQueue.length === 0) return;

      console.log(`Flushing ${this.writeQueue.length} writes to database`);

      // Batch write to database
      const batch = this.writeQueue.splice(0);

      try {
        await this.db.transaction(async (tx) => {
          for (const { key, value } of batch) {
            await tx.query(
              'INSERT INTO items VALUES (?, ?) ON DUPLICATE KEY UPDATE value = ?',
              [key, value, value]
            );
          }
        });
        console.log('Batch write completed');
      } catch (error) {
        console.error('Batch write failed:', error);
        // Could implement retry logic here
      }
    }, this.flushInterval);
  }
}

// Usage
const cache = new WriteBehind(redisClient, dbConnection);

// Write (immediate)
await cache.set('product:123', { name: 'iPhone', price: 999 });
// Returns immediately, DB write happens later
```

**Pros:**
- ✅ Very fast writes (no DB wait)
- ✅ Can batch writes (better throughput)
- ✅ Reduces database load

**Cons:**
- ❌ Risk of data loss (if cache fails before flush)
- ❌ Complex error handling
- ❌ Cache and DB can be out of sync

**Best for:**
- Write-heavy workloads
- When write latency is critical
- Acceptable to lose recent writes in failure

**Real-World Example: Social Media Likes**

```javascript
class LikeCounter {
  constructor(cache, database) {
    this.cache = cache;
    this.db = database;
  }

  async incrementLike(postId) {
    // Increment in cache (very fast)
    const newCount = await this.cache.incr(`likes:${postId}`);

    // Async write to database (every 100 likes or 1 minute)
    if (newCount % 100 === 0) {
      this.flushToDatabase(postId, newCount);
    }

    return newCount;
  }

  async flushToDatabase(postId, count) {
    // Background job
    setTimeout(async () => {
      await this.db.query(
        'UPDATE posts SET likes = ? WHERE id = ?',
        [count, postId]
      );
    }, 0);
  }

  async getLikes(postId) {
    // Always read from cache (fast)
    const count = await this.cache.get(`likes:${postId}`);
    return parseInt(count || 0);
  }
}
```

### Pattern 4: Refresh-Ahead

**Proactively refresh cache before expiration.**

```
Timeline:
T0: Cache entry created (TTL = 60s)
T50: Entry accessed → TTL remaining = 10s
     → Trigger background refresh
T55: New data loaded from DB
T60: Old cache entry expires
     → New entry already available ✓
```

**Implementation:**

```javascript
class RefreshAhead {
  constructor(cache, database, refreshThreshold = 0.2) {
    this.cache = cache;
    this.db = database;
    this.refreshThreshold = refreshThreshold; // Refresh when 20% TTL remaining
    this.refreshing = new Set(); // Prevent duplicate refreshes
  }

  async get(key, fetchFunction, ttl = 3600) {
    const value = await this.cache.get(key);

    if (value) {
      // Check if should refresh
      const ttlRemaining = await this.cache.ttl(key);
      const refreshPoint = ttl * this.refreshThreshold;

      if (ttlRemaining > 0 && ttlRemaining < refreshPoint) {
        // Refresh in background
        this.refreshAsync(key, fetchFunction, ttl);
      }

      return JSON.parse(value);
    }

    // Cache miss - fetch synchronously
    const data = await fetchFunction();
    await this.cache.setex(key, ttl, JSON.stringify(data));
    return data;
  }

  async refreshAsync(key, fetchFunction, ttl) {
    // Prevent duplicate refreshes
    if (this.refreshing.has(key)) return;

    this.refreshing.add(key);

    try {
      const data = await fetchFunction();
      await this.cache.setex(key, ttl, JSON.stringify(data));
      console.log(`Refreshed cache for ${key}`);
    } catch (error) {
      console.error(`Failed to refresh ${key}:`, error);
    } finally {
      this.refreshing.delete(key);
    }
  }
}

// Usage
const cache = new RefreshAhead(redisClient, dbConnection);

// Fetch with automatic refresh-ahead
const product = await cache.get(
  'product:123',
  async () => {
    // This function is called when cache miss or refresh needed
    return await db.query('SELECT * FROM products WHERE id = 123');
  },
  3600 // TTL: 1 hour
);
```

**Best for:**
- High-traffic keys
- Expensive computations
- Preventing cache miss latency

---

## 4.3 Cache Invalidation

**Phil Karlton:** "There are only two hard things in Computer Science: cache invalidation and naming things."

### Invalidation Strategies

#### 1. Time-To-Live (TTL)

**Simplest approach:** Cache expires after fixed time.

```javascript
// Set with TTL
await redis.setex('product:123', 3600, JSON.stringify(product)); // 1 hour

// TTL based on data freshness requirements
const TTL = {
  userProfile: 3600,        // 1 hour (changes occasionally)
  productCatalog: 600,      // 10 minutes (updates frequently)
  staticAssets: 86400 * 30, // 30 days (rarely changes)
  trendingPosts: 300        // 5 minutes (real-time feel)
};
```

**Pros:**
- ✅ Simple to implement
- ✅ Automatic cleanup
- ✅ Predictable behavior

**Cons:**
- ❌ Stale data until expiry
- ❌ Unnecessary refreshes if data unchanged
- ❌ Cache misses after expiry (stampede risk)

#### 2. Explicit Invalidation

**Invalidate cache when data changes.**

```javascript
class ExplicitInvalidation {
  async updateProduct(productId, updates) {
    // 1. Update database
    await db.query('UPDATE products SET ? WHERE id = ?', [updates, productId]);

    // 2. Invalidate cache
    await redis.del(`product:${productId}`);

    // 3. Invalidate related caches
    await redis.del(`product:${productId}:reviews`);
    await redis.del(`category:${updates.categoryId}:products`);
  }
}
```

**Challenge: Cascade Invalidation**

```javascript
// Product update affects multiple caches
class CascadeInvalidation {
  async updateProduct(productId, updates) {
    await db.query('UPDATE products SET ? WHERE id = ?', [updates, productId]);

    // Invalidate product cache
    await redis.del(`product:${productId}`);

    // Invalidate category cache
    const categories = await this.getProductCategories(productId);
    for (const categoryId of categories) {
      await redis.del(`category:${categoryId}:products`);
    }

    // Invalidate search results that might contain this product
    await redis.del('search:*'); // Dangerous! Clears all search cache

    // Invalidate user carts containing this product
    // ... complex to track
  }
}
```

**Better: Tag-Based Invalidation**

```javascript
class TagBasedInvalidation {
  async set(key, value, tags = []) {
    // Store value
    await redis.set(key, JSON.stringify(value));

    // Store tags mapping
    for (const tag of tags) {
      await redis.sadd(`tag:${tag}`, key);
    }
  }

  async invalidateTag(tag) {
    // Get all keys with this tag
    const keys = await redis.smembers(`tag:${tag}`);

    // Delete all keys
    if (keys.length > 0) {
      await redis.del(...keys);
    }

    // Delete tag set
    await redis.del(`tag:${tag}`);
  }

  async updateProduct(productId, updates) {
    await db.query('UPDATE products SET ? WHERE id = ?', [updates, productId]);

    // Invalidate all caches tagged with this product
    await this.invalidateTag(`product:${productId}`);

    // Invalidate category tag
    await this.invalidateTag(`category:${updates.categoryId}`);
  }
}

// Usage
const cache = new TagBasedInvalidation();

// Cache with tags
await cache.set(
  'product:123',
  productData,
  ['product:123', 'category:electronics', 'brand:apple']
);

// Invalidate by tag
await cache.invalidateTag('category:electronics'); // Clears all electronics
```

#### 3. Write-Through Invalidation

**Update cache when database updates.**

```javascript
async function updateProduct(productId, updates) {
  // 1. Update database
  await db.query('UPDATE products SET ? WHERE id = ?', [updates, productId]);

  // 2. Update cache (not delete)
  const product = await db.query('SELECT * FROM products WHERE id = ?', [productId]);
  await redis.setex(`product:${productId}`, 3600, JSON.stringify(product));
}
```

**Pros:**
- ✅ Cache always fresh
- ✅ No cache miss after update

**Cons:**
- ❌ Extra database read
- ❌ What if cache write fails?

#### 4. Event-Based Invalidation

**Use message queue for cache invalidation.**

```javascript
// Publisher (when data changes)
async function updateProduct(productId, updates) {
  await db.query('UPDATE products SET ? WHERE id = ?', [updates, productId]);

  // Publish invalidation event
  await messageQueue.publish('cache.invalidate', {
    type: 'product',
    id: productId,
    relatedTags: ['category:123', 'brand:apple']
  });
}

// Subscriber (cache invalidation service)
messageQueue.subscribe('cache.invalidate', async (event) => {
  console.log(`Invalidating cache for ${event.type}:${event.id}`);

  // Invalidate primary key
  await redis.del(`${event.type}:${event.id}`);

  // Invalidate related keys
  for (const tag of event.relatedTags) {
    await redis.del(`tag:${tag}`);
  }
});
```

**Pros:**
- ✅ Decoupled (updates don't directly touch cache)
- ✅ Can have multiple cache instances
- ✅ Reliable (queue ensures delivery)

**Cons:**
- ❌ Added complexity (message queue)
- ❌ Eventual consistency (small delay)

---

## 4.4 Cache Eviction Policies

When cache is full, which items to remove?

### 1. LRU (Least Recently Used)

**Remove items not accessed recently.**

```
Cache (max 3 items):

Access pattern: A, B, C, A, D

Step 1: Access A → [A]
Step 2: Access B → [A, B]
Step 3: Access C → [A, B, C]
Step 4: Access A → [B, C, A] (A moved to front)
Step 5: Access D → [C, A, D] (B evicted - least recently used)
```

**Redis Configuration:**
```
maxmemory 2gb
maxmemory-policy allkeys-lru
```

**Use Cases:**
- ✅ General-purpose caching
- ✅ User sessions
- ✅ API responses

### 2. LFU (Least Frequently Used)

**Remove items accessed least often.**

```
Cache with access counts:

A: 5 accesses
B: 2 accesses
C: 8 accesses
D: 1 access

Need space → Evict D (least frequent)
```

**Redis Configuration:**
```
maxmemory-policy allkeys-lfu
```

**Use Cases:**
- ✅ Popular content (trending posts)
- ✅ Product catalogs (bestsellers)

### 3. FIFO (First In First Out)

**Remove oldest items.**

```
Cache (max 3 items):

Step 1: Add A → [A]
Step 2: Add B → [A, B]
Step 3: Add C → [A, B, C]
Step 4: Add D → [B, C, D] (A evicted - oldest)
```

**Use Cases:**
- ✅ Time-series data
- ✅ Logs

### 4. TTL-Based

**Remove expired items first.**

```javascript
// Items with TTL
await redis.setex('session:123', 3600, sessionData);  // 1 hour
await redis.setex('temp:456', 300, tempData);         // 5 minutes

// Expired items removed automatically
```

### Comparison

```
┌──────────┬─────────────┬────────────┬─────────────┐
│ Policy   │ Complexity  │ Hit Rate   │ Use Case    │
├──────────┼─────────────┼────────────┼─────────────┤
│ LRU      │ Medium      │ Good       │ General     │
│ LFU      │ High        │ Better     │ Popular     │
│ FIFO     │ Low         │ Poor       │ Simple      │
│ TTL      │ Low         │ Good       │ Expiring    │
└──────────┴─────────────┴────────────┴─────────────┘
```

---

## 4.5 Distributed Caching

### Single Redis vs Redis Cluster

**Single Redis:**
```
┌─────────────────┐
│  Redis Instance │
│  - Max 256 GB   │
│  - Single point │
└─────────────────┘
```

**Redis Cluster:**
```
┌────────────┐  ┌────────────┐  ┌────────────┐
│  Redis 1   │  │  Redis 2   │  │  Redis 3   │
│  Shard 0   │  │  Shard 1   │  │  Shard 2   │
│  Keys:     │  │  Keys:     │  │  Keys:     │
│  0-5460    │  │  5461-10922│  │  10923-16383│
└────────────┘  └────────────┘  └────────────┘
        ↓              ↓              ↓
   Replicas      Replicas      Replicas
```

**Sharding Keys:**

```javascript
// Redis Cluster automatically shards by key
await redis.set('user:123', userData);  // Goes to specific shard
await redis.get('user:123');            // Routed to same shard

// Hash tags for co-location
await redis.set('user:{123}:profile', profile);    // Same shard
await redis.set('user:{123}:settings', settings);  // Same shard
// Both keys go to same shard (hash {123})
```

### Consistent Hashing in Distributed Cache

```javascript
class DistributedCache {
  constructor(cacheNodes) {
    this.nodes = cacheNodes;
    this.ring = new ConsistentHash(cacheNodes);
  }

  async get(key) {
    const node = this.ring.getNode(key);
    return await node.get(key);
  }

  async set(key, value, ttl) {
    const node = this.ring.getNode(key);
    return await node.setex(key, ttl, value);
  }

  // Add new cache node (minimal redistribution)
  addNode(newNode) {
    this.nodes.push(newNode);
    this.ring.addNode(newNode);
    // Only ~1/N keys need to move
  }
}
```

---

## 4.6 Multi-Level Caching

**Use multiple cache layers for optimal performance.**

```
┌──────────────────────────────────────────────┐
│ Level 1: In-Memory (Application)             │
│ - Fastest (< 1ms)                            │
│ - Smallest (100s MB)                         │
│ - Per-server cache                           │
└────────────────┬─────────────────────────────┘
                 │ Miss
                 ↓
┌──────────────────────────────────────────────┐
│ Level 2: Redis (Distributed)                 │
│ - Fast (1-5ms)                               │
│ - Medium (10s GB)                            │
│ - Shared across servers                      │
└────────────────┬─────────────────────────────┘
                 │ Miss
                 ↓
┌──────────────────────────────────────────────┐
│ Level 3: Database (Persistent)               │
│ - Slower (50-200ms)                          │
│ - Large (TBs)                                │
│ - Source of truth                            │
└──────────────────────────────────────────────┘
```

**Implementation:**

```javascript
class MultiLevelCache {
  constructor() {
    this.l1Cache = new Map();           // In-memory
    this.l1MaxSize = 1000;
    this.l2Cache = redisClient;         // Redis
    this.database = dbConnection;       // Database
  }

  async get(key) {
    // Level 1: In-memory
    if (this.l1Cache.has(key)) {
      console.log('L1 cache hit');
      return this.l1Cache.get(key);
    }

    // Level 2: Redis
    const l2Value = await this.l2Cache.get(key);
    if (l2Value) {
      console.log('L2 cache hit');
      const value = JSON.parse(l2Value);

      // Promote to L1
      this.setL1(key, value);

      return value;
    }

    // Level 3: Database
    console.log('Cache miss - querying database');
    const value = await this.database.query(
      'SELECT * FROM items WHERE id = ?',
      [key]
    );

    if (value) {
      // Store in all levels
      this.setL1(key, value);
      await this.l2Cache.setex(key, 3600, JSON.stringify(value));
    }

    return value;
  }

  setL1(key, value) {
    // LRU eviction for L1
    if (this.l1Cache.size >= this.l1MaxSize) {
      const firstKey = this.l1Cache.keys().next().value;
      this.l1Cache.delete(firstKey);
    }
    this.l1Cache.set(key, value);
  }

  async set(key, value) {
    // Write to database
    await this.database.query('INSERT INTO items VALUES (?, ?)', [key, value]);

    // Invalidate caches
    this.l1Cache.delete(key);
    await this.l2Cache.del(key);
  }
}
```

**Performance Impact:**

```
Cache Hit Rates:
L1: 40% → avg latency: 0.5ms
L2: 50% → avg latency: 2ms
DB: 10% → avg latency: 100ms

Average Latency:
(0.40 × 0.5) + (0.50 × 2) + (0.10 × 100)
= 0.2 + 1.0 + 10.0
= 11.2ms

Without caching: 100ms
With multi-level: 11.2ms (9x faster!)
```

---

## 4.7 Real-World Caching Examples

### Example 1: Facebook's TAO (The Associations and Objects)

**Problem:** Billions of social graph queries per second.

**Solution:** Multi-level caching architecture.

```
┌─────────────────────────────────────────────┐
│ Client Request                              │
└────────────┬────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────┐
│ TAO Cache Layer (Memcached)                 │
│ - Caches objects and associations           │
│ - 99%+ hit rate                             │
└────────────┬────────────────────────────────┘
             │ Miss
             ↓
┌─────────────────────────────────────────────┐
│ MySQL (Persistent Storage)                  │
└─────────────────────────────────────────────┘
```

**Key Decisions:**
- Write-through caching (consistency)
- Lease-based invalidation (prevent thundering herd)
- Separate cache for associations (edges) and objects (nodes)

**Results:**
- 99.8% cache hit rate
- Sub-millisecond latency
- Handles billions of queries/second

### Example 2: Netflix Content Delivery

**Problem:** Serve millions of concurrent video streams globally.

**Caching Strategy:**

```
┌─────────────────────────────────────────────┐
│ User Device                                 │
└────────────┬────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────┐
│ CDN Edge (Closest to User)                  │
│ - Hot content (90%+ hit rate)               │
│ - 200ms from user                           │
└────────────┬────────────────────────────────┘
             │ Miss
             ↓
┌─────────────────────────────────────────────┐
│ Regional Cache                              │
│ - Warm content                              │
│ - 1000ms from user                          │
└────────────┬────────────────────────────────┘
             │ Miss
             ↓
┌─────────────────────────────────────────────┐
│ Origin Servers (AWS S3)                     │
│ - All content                               │
│ - 5000ms from user                          │
└─────────────────────────────────────────────┘
```

**Pre-positioning:**
```javascript
// Netflix pre-positions popular content
async function prePositionContent(contentId, regions) {
  const videoFile = await s3.getObject(contentId);

  // Push to edge servers before demand
  for (const region of regions) {
    await cdnEdge[region].put(contentId, videoFile);
  }

  console.log(`Pre-positioned ${contentId} to ${regions.length} regions`);
}

// Predict popular content
const trendingShows = await analytics.getTrending();
for (const show of trendingShows) {
  await prePositionContent(show.id, ['us-east', 'eu-west', 'asia-pacific']);
}
```

**Results:**
- 95%+ content served from edge
- < 1 second to start playback
- Handles 200M+ subscribers

### Example 3: Twitter Timeline Cache

**Problem:** Generate timeline for 400M+ users instantly.

**Solution: Fanout on Write + Cache**

```javascript
class TwitterTimelineCache {
  async postTweet(userId, content) {
    // 1. Save tweet
    const tweet = await db.tweets.create({ userId, content });

    // 2. Get followers
    const followers = await db.followers.find({ following: userId });

    // 3. Fanout to follower timelines (Redis lists)
    const fanoutPromises = followers.map(async (follower) => {
      // Add tweet to follower's cached timeline
      await redis.lpush(`timeline:${follower.id}`, tweet.id);

      // Trim to latest 800 tweets
      await redis.ltrim(`timeline:${follower.id}`, 0, 799);
    });

    await Promise.all(fanoutPromises);

    return tweet;
  }

  async getTimeline(userId, limit = 20) {
    // Get from cached timeline (Redis list)
    const tweetIds = await redis.lrange(`timeline:${userId}`, 0, limit - 1);

    if (tweetIds.length > 0) {
      // Hydrate tweets (can be cached too)
      const tweets = await Promise.all(
        tweetIds.map(id => this.getTweet(id))
      );
      return tweets;
    }

    // Cache miss - build timeline from scratch
    return await this.buildTimeline(userId, limit);
  }

  async getTweet(tweetId) {
    // Multi-level cache for tweets
    const cached = await redis.get(`tweet:${tweetId}`);
    if (cached) return JSON.parse(cached);

    const tweet = await db.tweets.findById(tweetId);
    await redis.setex(`tweet:${tweetId}`, 3600, JSON.stringify(tweet));
    return tweet;
  }
}
```

---

## 4.8 Cache Monitoring & Metrics

**Key Metrics to Track:**

```javascript
class CacheMetrics {
  constructor() {
    this.hits = 0;
    this.misses = 0;
    this.errors = 0;
  }

  recordHit() {
    this.hits++;
  }

  recordMiss() {
    this.misses++;
  }

  recordError() {
    this.errors++;
  }

  getHitRate() {
    const total = this.hits + this.misses;
    return total > 0 ? (this.hits / total) * 100 : 0;
  }

  getStats() {
    return {
      hits: this.hits,
      misses: this.misses,
      errors: this.errors,
      hitRate: this.getHitRate().toFixed(2) + '%',
      total: this.hits + this.misses
    };
  }
}

// Instrumented cache
class MonitoredCache {
  constructor(cache) {
    this.cache = cache;
    this.metrics = new CacheMetrics();
  }

  async get(key) {
    try {
      const value = await this.cache.get(key);

      if (value) {
        this.metrics.recordHit();
      } else {
        this.metrics.recordMiss();
      }

      return value;
    } catch (error) {
      this.metrics.recordError();
      throw error;
    }
  }

  getMetrics() {
    return this.metrics.getStats();
  }
}

// Usage
const cache = new MonitoredCache(redisClient);

// Check metrics
setInterval(() => {
  console.log('Cache stats:', cache.getMetrics());
  // Output: { hits: 8500, misses: 1500, errors: 0, hitRate: '85.00%' }
}, 60000); // Every minute
```

**Target Metrics:**

```
┌─────────────────────┬──────────┬───────────┐
│ Metric              │ Good     │ Excellent │
├─────────────────────┼──────────┼───────────┤
│ Hit Rate            │ > 80%    │ > 95%     │
│ Avg Latency         │ < 10ms   │ < 5ms     │
│ P99 Latency         │ < 50ms   │ < 20ms    │
│ Memory Usage        │ < 80%    │ < 70%     │
│ Eviction Rate       │ < 5%     │ < 1%      │
└─────────────────────┴──────────┴───────────┘
```

---

## 4.9 Common Caching Pitfalls

### Pitfall 1: Cache Stampede (Thundering Herd)

**Problem:** Popular key expires, many requests hit database simultaneously.

```
Cache expires at T0

T0: 1000 requests arrive
    All see cache miss
    All query database
    Database overloaded! 💥
```

**Solution 1: Locking**
```javascript
async function getWithLock(key, fetchFn) {
  const cached = await redis.get(key);
  if (cached) return cached;

  // Try to acquire lock
  const locked = await redis.set(`lock:${key}`, '1', 'NX', 'EX', 10);

  if (locked) {
    // This request fetches data
    try {
      const data = await fetchFn();
      await redis.setex(key, 3600, data);
      return data;
    } finally {
      await redis.del(`lock:${key}`);
    }
  } else {
    // Wait for the other request to finish
    await sleep(100);
    return getWithLock(key, fetchFn);
  }
}
```

**Solution 2: Probabilistic Early Expiration**
```javascript
async function getWithProbabilisticRefresh(key, fetchFn, ttl = 3600) {
  const cached = await redis.get(key);

  if (cached) {
    const ttlRemaining = await redis.ttl(key);
    const xfetch = ttlRemaining / ttl;
    const beta = 1; // Tuning parameter

    // Probabilistically refresh early
    if (Math.random() < beta * (1 / xfetch)) {
      // Refresh in background
      fetchFn().then(data => {
        redis.setex(key, ttl, data);
      });
    }

    return cached;
  }

  // Cache miss
  const data = await fetchFn();
  await redis.setex(key, ttl, data);
  return data;
}
```

### Pitfall 2: Stale Data

**Problem:** Cache not invalidated when data changes.

**Solution: Cache Versioning**
```javascript
async function setWithVersion(key, value, ttl) {
  const version = Date.now();
  await redis.setex(
    key,
    ttl,
    JSON.stringify({ data: value, version })
  );
  return version;
}

async function getWithVersion(key, minVersion) {
  const cached = await redis.get(key);
  if (!cached) return null;

  const { data, version } = JSON.parse(cached);

  // Check if version is acceptable
  if (version < minVersion) {
    await redis.del(key); // Stale data
    return null;
  }

  return data;
}
```

### Pitfall 3: Large Cache Values

**Problem:** Storing large objects (> 1MB) slows down cache.

**Solution: Compress or Split**
```javascript
const zlib = require('zlib');
const { promisify } = require('util');
const gzip = promisify(zlib.gzip);
const gunzip = promisify(zlib.gunzip);

async function setCompressed(key, value, ttl) {
  const json = JSON.stringify(value);
  const compressed = await gzip(json);
  await redis.setex(key, ttl, compressed);
}

async function getCompressed(key) {
  const compressed = await redis.getBuffer(key);
  if (!compressed) return null;

  const json = await gunzip(compressed);
  return JSON.parse(json.toString());
}
```

---

## 4.10 Key Takeaways

### When to Cache

```
✓ Read >> Write (read-heavy workload)
✓ Expensive to compute or fetch
✓ Tolerate some staleness
✓ Frequently accessed (80/20 rule)

✗ Frequently changing data
✗ Strong consistency required
✗ User-specific sensitive data
✗ Rarely accessed data
```

### Pattern Selection

```
┌──────────────────┬─────────────────┬─────────────┐
│ Use Case         │ Pattern         │ Consistency │
├──────────────────┼─────────────────┼─────────────┤
│ General purpose  │ Cache-Aside     │ Eventual    │
│ Read-heavy       │ Write-Through   │ Strong      │
│ Write-heavy      │ Write-Behind    │ Eventual    │
│ Hot keys         │ Refresh-Ahead   │ Eventual    │
└──────────────────┴─────────────────┴─────────────┘
```

### Cache Hierarchy

```
1. In-Memory (L1): Fastest, smallest, per-server
2. Redis (L2): Fast, shared, distributed
3. Database: Slower, source of truth
```

---

## 4.11 Interview Questions

### Basic:
1. What is caching and why is it important?
2. Explain cache-aside pattern.
3. What is cache invalidation?

### Intermediate:
1. Compare cache-aside, write-through, and write-behind patterns.
2. What is a cache stampede and how do you prevent it?
3. Explain LRU vs LFU eviction policies.

### Advanced:
1. Design a multi-level caching system for an e-commerce platform.
2. How would you handle cache invalidation for a social network?
3. Design Facebook's news feed caching system.

---

**Next Chapter:** [Chapter 5: API Design & Communication](./SystemDesign-05-API-Design.md)

In the next chapter, we'll cover:
- REST vs GraphQL vs gRPC
- API Gateway patterns
- Rate limiting and throttling
- WebSockets and real-time communication
- Message queues and async patterns
