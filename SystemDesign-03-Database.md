# Chapter 3: Database Design & Patterns

## 3.1 SQL vs NoSQL: The Real Trade-offs

Choosing between SQL and NoSQL isn't about which is "better"—it's about matching database characteristics to your system requirements.

### The Decision Matrix

```
Choose SQL when:
✓ Need ACID transactions (banking, e-commerce)
✓ Complex queries with JOINs
✓ Data has clear relationships
✓ Schema is well-defined and stable
✓ Data integrity is critical

Choose NoSQL when:
✓ Need horizontal scalability
✓ Schema flexibility required
✓ High write throughput
✓ Data is document/key-value/graph oriented
✓ Eventually consistent is acceptable
```

### SQL Databases: Deep Dive

**Strengths:**

```
┌──────────────────────────────────────────────────┐
│ ACID Guarantees                                  │
│ ├─ Atomicity: All or nothing                    │
│ ├─ Consistency: Valid state always              │
│ ├─ Isolation: Concurrent transactions safe      │
│ └─ Durability: Committed data persists          │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ Powerful Query Language (SQL)                    │
│ ├─ Complex JOINs                                │
│ ├─ Aggregations (SUM, AVG, GROUP BY)           │
│ ├─ Subqueries                                    │
│ └─ Window functions                              │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ Data Integrity                                   │
│ ├─ Foreign key constraints                      │
│ ├─ Unique constraints                            │
│ ├─ Check constraints                             │
│ └─ Triggers                                      │
└──────────────────────────────────────────────────┘
```

**Weaknesses:**
```
❌ Vertical scaling limits (single server capacity)
❌ Difficult to shard (maintain ACID across shards)
❌ Schema migrations can be expensive
❌ Not optimized for high write throughput
```

**Real-World Example: E-Commerce Orders**

```sql
-- Order with multiple items, inventory check, payment
-- All must succeed or all must fail (ACID)

BEGIN TRANSACTION;

-- Create order
INSERT INTO orders (user_id, total, status)
VALUES (12345, 99.99, 'pending')
RETURNING id INTO @order_id;

-- Add order items
INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES
  (@order_id, 101, 2, 29.99),
  (@order_id, 102, 1, 39.99);

-- Decrease inventory (must not go negative)
UPDATE products
SET inventory = inventory - 2
WHERE id = 101 AND inventory >= 2;

UPDATE products
SET inventory = inventory - 1
WHERE id = 102 AND inventory >= 1;

-- Check if inventory updates succeeded
IF @@ROWCOUNT != 2 THEN
  ROLLBACK;
  RETURN 'Insufficient inventory';
END IF;

-- Process payment
INSERT INTO payments (order_id, amount, status)
VALUES (@order_id, 99.99, 'completed');

COMMIT;
```

**Why SQL is Perfect Here:**
- Order must be atomic (either complete or nothing)
- Inventory must be accurate (no overselling)
- Payment must be linked to order
- Need referential integrity

### NoSQL Databases: Deep Dive

**Types of NoSQL:**

```
┌────────────────────────────────────────────────┐
│ Document Store (MongoDB, Couchbase)            │
│ - Store JSON-like documents                    │
│ - Flexible schema                              │
│ - Good for: Content management, user profiles │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ Key-Value Store (Redis, DynamoDB)              │
│ - Simple key → value mapping                   │
│ - Extremely fast                               │
│ - Good for: Caching, session storage           │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ Column-Family (Cassandra, HBase)               │
│ - Optimized for column-based queries           │
│ - High write throughput                        │
│ - Good for: Time-series data, logs             │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ Graph Database (Neo4j, Neptune)                │
│ - Store nodes and relationships                │
│ - Efficient graph traversal                    │
│ - Good for: Social networks, recommendations   │
└────────────────────────────────────────────────┘
```

**Strengths:**
```
✅ Horizontal scalability (add more nodes)
✅ Schema flexibility (evolve without downtime)
✅ High write throughput
✅ Optimized for specific data models
✅ Better geographic distribution
```

**Weaknesses:**
```
❌ Eventual consistency (by default)
❌ Limited query capabilities (no complex JOINs)
❌ Application handles data integrity
❌ Learning curve for specific models
```

**Real-World Example: Social Media Posts**

```javascript
// MongoDB document store - User posts
{
  "_id": "post:12345",
  "userId": "user:67890",
  "content": "Just deployed to production!",
  "timestamp": "2026-01-15T10:30:00Z",
  "likes": 42,
  "comments": [
    {
      "userId": "user:11111",
      "text": "Congrats!",
      "timestamp": "2026-01-15T10:31:00Z"
    },
    {
      "userId": "user:22222",
      "text": "Amazing work!",
      "timestamp": "2026-01-15T10:32:00Z"
    }
  ],
  "hashtags": ["deployment", "production"],
  "media": {
    "type": "image",
    "url": "https://cdn.example.com/image.jpg"
  }
}

// Easy to query
db.posts.find({
  userId: "user:67890",
  timestamp: { $gte: "2026-01-01T00:00:00Z" }
}).sort({ timestamp: -1 }).limit(20);

// Easy to update (no schema migration needed)
db.posts.updateOne(
  { _id: "post:12345" },
  { $inc: { likes: 1 } }
);
```

**Why NoSQL is Perfect Here:**
- Posts are independent (no complex relationships)
- Schema varies (text, images, videos, polls)
- High write volume (millions of posts/hour)
- Eventual consistency acceptable (likes can be slightly off)
- Easy to shard by userId

### Polyglot Persistence

**Best Practice:** Use multiple databases for different needs.

```
E-Commerce System Architecture:

┌─────────────────────────────────────────────┐
│ Orders & Payments                           │
│ Database: PostgreSQL (ACID required)        │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Product Catalog                             │
│ Database: MongoDB (flexible schema)         │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Session Storage                             │
│ Database: Redis (fast key-value)            │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Search                                      │
│ Database: Elasticsearch (full-text)         │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Recommendations                             │
│ Database: Neo4j (graph relationships)       │
└─────────────────────────────────────────────┘
```

**Trade-off:** More complexity vs optimal performance for each use case.

---

## 3.2 Database Sharding (Horizontal Partitioning)

Sharding splits data across multiple databases to scale beyond single server limits.

### Why Shard?

```
Single Database Limits:

Storage: Limited by disk size (10 TB typical)
Memory: Limited by RAM (1 TB max typical)
CPU: Limited by cores (64-96 cores typical)
Connections: Limited by max connections (10k typical)

Sharding breaks these limits!
```

### Sharding Strategies

#### 1. Hash-Based Sharding

**How it works:** `shard = hash(shardKey) % numShards`

```
Example: User database sharded by user_id

hash(user_id=12345) = 8472 % 4 = 0 → Shard 0
hash(user_id=67890) = 2839 % 4 = 3 → Shard 3

┌─────────────────────────────────────────┐
│           Load Balancer                 │
└────────────┬────────────────────────────┘
             │
    ┌────────┼────────┬────────┐
    │        │        │        │
┌───▼──┐ ┌───▼──┐ ┌───▼──┐ ┌───▼──┐
│Shard0│ │Shard1│ │Shard2│ │Shard3│
│Users:│ │Users:│ │Users:│ │Users:│
│0-24% │ │25-49%│ │50-74%│ │75-99%│
└──────┘ └──────┘ └──────┘ └──────┘
```

**Pros:**
- ✅ Even distribution of data
- ✅ Simple to implement
- ✅ Works well for high cardinality keys

**Cons:**
- ❌ Can't easily add/remove shards (requires rehashing all data)
- ❌ Range queries span multiple shards
- ❌ Related data may be on different shards

**Implementation:**
```javascript
class HashSharding {
  constructor(shards) {
    this.shards = shards; // Array of database connections
  }

  getShardForUser(userId) {
    const hash = this.hashCode(userId);
    const shardIndex = hash % this.shards.length;
    return this.shards[shardIndex];
  }

  hashCode(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // Convert to 32-bit integer
    }
    return Math.abs(hash);
  }

  async getUser(userId) {
    const shard = this.getShardForUser(userId);
    return await shard.query('SELECT * FROM users WHERE id = ?', [userId]);
  }

  async createUser(userId, userData) {
    const shard = this.getShardForUser(userId);
    return await shard.query('INSERT INTO users VALUES (?, ?)', [userId, userData]);
  }
}

// Usage
const sharding = new HashSharding([
  dbConnection1, // Shard 0
  dbConnection2, // Shard 1
  dbConnection3, // Shard 2
  dbConnection4  // Shard 3
]);

const user = await sharding.getUser('user:12345');
```

#### 2. Range-Based Sharding

**How it works:** Partition data by key ranges.

```
Example: User database sharded by user_id ranges

user_id 1-1M      → Shard 0
user_id 1M-2M     → Shard 1
user_id 2M-3M     → Shard 2
user_id 3M-4M     → Shard 3

┌──────────────────────────────────┐
│     Routing Table                │
│  Range      →   Shard            │
│  1-1M       →   Shard 0          │
│  1M-2M      →   Shard 1          │
│  2M-3M      →   Shard 2          │
│  3M-4M      →   Shard 3          │
└──────────────────────────────────┘
```

**Pros:**
- ✅ Easy to add new shards (just add new range)
- ✅ Range queries are efficient (single shard)
- ✅ Related data can be co-located

**Cons:**
- ❌ Uneven distribution (hotspots)
- ❌ New users all go to same shard initially
- ❌ Requires rebalancing over time

**Real-World Problem: Hotspot**

```
Chronological user IDs:

2020: user_id 1-1M      → Shard 0 (old, inactive users)
2021: user_id 1M-2M     → Shard 1 (older users)
2022: user_id 2M-3M     → Shard 2 (active users)
2023: user_id 3M-4M     → Shard 3 (very active! 🔥)

Problem: Shard 3 gets all new users → overloaded!
```

**Solution: Dynamic Range Splitting**

```javascript
class RangeSharding {
  constructor() {
    this.ranges = [
      { min: 1, max: 1000000, shard: 'shard0' },
      { min: 1000001, max: 2000000, shard: 'shard1' },
      { min: 2000001, max: 3000000, shard: 'shard2' },
      { min: 3000001, max: 4000000, shard: 'shard3' }
    ];
  }

  getShardForUser(userId) {
    const userIdNum = parseInt(userId.split(':')[1]);
    const range = this.ranges.find(r =>
      userIdNum >= r.min && userIdNum <= r.max
    );
    return range ? range.shard : null;
  }

  // Split a range when it gets too large
  splitRange(shardName, splitPoint) {
    const rangeIndex = this.ranges.findIndex(r => r.shard === shardName);
    const oldRange = this.ranges[rangeIndex];

    // Split into two ranges
    this.ranges[rangeIndex] = {
      min: oldRange.min,
      max: splitPoint,
      shard: shardName
    };

    this.ranges.push({
      min: splitPoint + 1,
      max: oldRange.max,
      shard: `${shardName}_new`
    });
  }
}
```

#### 3. Geographic Sharding

**How it works:** Shard by user location for low latency.

```
Users by Region:

US Users       → US-East Shard
EU Users       → EU-West Shard
Asia Users     → Asia-Southeast Shard

┌──────────────────────────────────────────┐
│            GeoDNS / Router               │
└────────┬─────────────┬──────────┬────────┘
         │             │          │
    ┌────▼──┐     ┌────▼──┐  ┌───▼────┐
    │US-East│     │EU-West│  │Asia-SE │
    │Shard  │     │Shard  │  │Shard   │
    │       │     │       │  │        │
    │Users: │     │Users: │  │Users:  │
    │50M    │     │30M    │  │20M     │
    └───────┘     └───────┘  └────────┘
```

**Pros:**
- ✅ Low latency (data close to users)
- ✅ Compliance (data residency requirements)
- ✅ Fault isolation (region failure doesn't affect others)

**Cons:**
- ❌ Cross-region queries are slow
- ❌ Users traveling have higher latency
- ❌ Uneven growth across regions

**Real-World: Facebook's Geography-Based Sharding**

```javascript
class GeoSharding {
  constructor() {
    this.regionShards = {
      'US': ['us-east-1', 'us-west-2'],
      'EU': ['eu-west-1', 'eu-central-1'],
      'ASIA': ['ap-southeast-1', 'ap-northeast-1']
    };
  }

  getShardForUser(userId, userRegion) {
    const shards = this.regionShards[userRegion];
    if (!shards) return this.regionShards['US'][0]; // fallback

    // Use hash within region for distribution
    const hash = this.hashCode(userId);
    const shardIndex = hash % shards.length;
    return shards[shardIndex];
  }

  async getUser(userId, userRegion) {
    const shard = this.getShardForUser(userId, userRegion);
    const connection = await this.getConnection(shard);
    return await connection.query('SELECT * FROM users WHERE id = ?', [userId]);
  }
}
```

#### 4. Directory-Based Sharding

**How it works:** Maintain a lookup table (directory) mapping keys to shards.

```
┌─────────────────────────────────┐
│      Shard Directory            │
│  (Lightweight DB or Cache)      │
│                                 │
│  user:123 → Shard 2             │
│  user:456 → Shard 1             │
│  user:789 → Shard 3             │
└─────────────────────────────────┘
         │
    Query directory first
         │
         ↓
    Route to correct shard
```

**Pros:**
- ✅ Very flexible (can move data between shards)
- ✅ Can optimize placement (hot users on powerful shards)
- ✅ Easy to rebalance

**Cons:**
- ❌ Extra lookup overhead
- ❌ Directory becomes bottleneck
- ❌ Directory must be highly available

**Implementation:**
```javascript
class DirectorySharding {
  constructor() {
    this.directory = new Map(); // In production: Redis or similar
    this.shards = ['shard0', 'shard1', 'shard2', 'shard3'];
  }

  async getShardForUser(userId) {
    // Check directory
    let shard = await this.directory.get(userId);

    if (!shard) {
      // First time seeing this user - assign shard
      shard = this.assignShard(userId);
      await this.directory.set(userId, shard);
    }

    return shard;
  }

  assignShard(userId) {
    // Could use various strategies:
    // 1. Least loaded shard
    // 2. Round-robin
    // 3. Based on user tier
    return this.getLeastLoadedShard();
  }

  async moveUser(userId, fromShard, toShard) {
    // 1. Copy user data to new shard
    const userData = await this.getConnection(fromShard)
      .query('SELECT * FROM users WHERE id = ?', [userId]);

    await this.getConnection(toShard)
      .query('INSERT INTO users VALUES (?)', [userData]);

    // 2. Update directory (atomic)
    await this.directory.set(userId, toShard);

    // 3. Delete from old shard
    await this.getConnection(fromShard)
      .query('DELETE FROM users WHERE id = ?', [userId]);
  }
}
```

### Challenges with Sharding

#### Challenge 1: Cross-Shard Queries

**Problem:** Query needs data from multiple shards.

```sql
-- Find all orders for a specific product
-- Product ID determines shard, but orders are on user shards!

SELECT * FROM orders
WHERE product_id = 12345;

-- This query must run on ALL shards! 😱
```

**Solutions:**

1. **Scatter-Gather Pattern**
```javascript
async function findOrdersByProduct(productId) {
  // Query all shards in parallel
  const promises = shards.map(shard =>
    shard.query('SELECT * FROM orders WHERE product_id = ?', [productId])
  );

  const results = await Promise.all(promises);

  // Merge and sort results
  return results
    .flat()
    .sort((a, b) => b.timestamp - a.timestamp);
}
```

2. **Denormalization (Duplicate Data)**
```javascript
// Store order in both user shard AND product shard
async function createOrder(userId, productId, orderData) {
  const userShard = getShardForUser(userId);
  const productShard = getShardForProduct(productId);

  // Write to both shards
  await userShard.query('INSERT INTO orders VALUES (?)', [orderData]);
  await productShard.query('INSERT INTO product_orders VALUES (?)', [orderData]);
}
```

3. **Secondary Index Service**
```
┌──────────────────────────────────┐
│  Elasticsearch / Secondary Index │
│  product_id → [order_ids]        │
└──────────────────────────────────┘
         │
    Query index first
         │
    Then fetch from shards
```

#### Challenge 2: Distributed Transactions

**Problem:** Transaction spans multiple shards.

```javascript
// Transfer money between users on different shards
async function transferMoney(fromUserId, toUserId, amount) {
  const fromShard = getShardForUser(fromUserId);
  const toShard = getShardForUser(toUserId);

  if (fromShard === toShard) {
    // Same shard - easy!
    await fromShard.transaction(async (tx) => {
      await tx.query('UPDATE accounts SET balance = balance - ? WHERE user_id = ?',
        [amount, fromUserId]);
      await tx.query('UPDATE accounts SET balance = balance + ? WHERE user_id = ?',
        [amount, toUserId]);
    });
  } else {
    // Different shards - need distributed transaction! 😱
    // Options: 2PC, Saga pattern, or avoid cross-shard transactions
  }
}
```

**Solution: Two-Phase Commit (2PC)**

```
Phase 1: Prepare
┌──────────┐           ┌──────────┐
│ Shard A  │           │ Shard B  │
└────┬─────┘           └────┬─────┘
     │ Can you commit?      │
     │ (deduct $100)        │
     ├─────────────────────►│
     │                      │
     │     YES (locked)     │
     │◄─────────────────────┤
     │                      │

Phase 2: Commit
     │ COMMIT!              │
     ├─────────────────────►│
     │                      │
     │ Committed ✓          │
     │◄─────────────────────┤

If any shard says NO → ROLLBACK on all shards
```

**Problem with 2PC:**
- Slow (multiple round trips)
- Coordinator failure can block
- Not suitable for high throughput

**Better Solution: Saga Pattern (for eventual consistency)**

```javascript
// Saga: Series of local transactions with compensating actions
async function transferMoneySaga(fromUserId, toUserId, amount) {
  try {
    // Step 1: Deduct from source
    const fromShard = getShardForUser(fromUserId);
    await fromShard.query(
      'UPDATE accounts SET balance = balance - ? WHERE user_id = ?',
      [amount, fromUserId]
    );

    // Step 2: Add to destination
    const toShard = getShardForUser(toUserId);
    await toShard.query(
      'UPDATE accounts SET balance = balance + ? WHERE user_id = ?',
      [amount, toUserId]
    );

    // Success!
    return { success: true };

  } catch (error) {
    // Compensating action: Refund to source
    await fromShard.query(
      'UPDATE accounts SET balance = balance + ? WHERE user_id = ?',
      [amount, fromUserId]
    );

    throw error;
  }
}
```

---

## 3.3 Database Replication

Replication creates copies of data for availability and read scalability.

### Master-Slave (Primary-Replica) Replication

```
┌────────────────────────────────────────┐
│              Master (Primary)          │
│         All writes go here             │
└───────────────┬────────────────────────┘
                │
                │ Replication
                │
      ┌─────────┼─────────┐
      │         │         │
 ┌────▼───┐ ┌──▼────┐ ┌──▼────┐
 │Replica1│ │Replica2│ │Replica3│
 │Read-only│ │Read-only│ │Read-only│
 └────────┘ └───────┘ └───────┘
      ↑         ↑         ↑
      │         │         │
   Read      Read      Read
  Traffic   Traffic   Traffic
```

**How it works:**

```
1. Write to Master:
   INSERT INTO users (name) VALUES ('Alice')

2. Master logs change to binary log

3. Replicas pull changes from binary log

4. Replicas apply changes to their data

5. Replication lag: typically 0-100ms
```

**Benefits:**
- ✅ Read scalability (add more replicas)
- ✅ High availability (failover to replica)
- ✅ Backup without affecting production
- ✅ Analytics on replica (no impact on master)

**Challenges:**
```
❌ Replication lag (eventual consistency)
❌ Single write point (master bottleneck)
❌ Failover complexity
```

**Implementation:**

```javascript
class DatabaseWithReplicas {
  constructor(master, replicas) {
    this.master = master;
    this.replicas = replicas;
    this.currentReplicaIndex = 0;
  }

  // All writes go to master
  async write(query, params) {
    return await this.master.query(query, params);
  }

  // Reads from replicas (round-robin)
  async read(query, params) {
    const replica = this.getNextReplica();
    return await replica.query(query, params);
  }

  // For critical reads that need latest data
  async readFromMaster(query, params) {
    return await this.master.query(query, params);
  }

  getNextReplica() {
    const replica = this.replicas[this.currentReplicaIndex];
    this.currentReplicaIndex = (this.currentReplicaIndex + 1) % this.replicas.length;
    return replica;
  }
}

// Usage
const db = new DatabaseWithReplicas(
  masterConnection,
  [replica1, replica2, replica3]
);

// Write
await db.write('INSERT INTO users VALUES (?)', [userData]);

// Read (from replica)
const users = await db.read('SELECT * FROM users WHERE id = ?', [userId]);

// Critical read (from master, to avoid replication lag)
const balance = await db.readFromMaster(
  'SELECT balance FROM accounts WHERE user_id = ?',
  [userId]
);
```

**Replication Lag Problem:**

```javascript
// User updates profile
await db.write('UPDATE users SET name = ? WHERE id = ?', ['Bob', 123]);

// Immediately read (from replica)
const user = await db.read('SELECT * FROM users WHERE id = ?', [123]);

console.log(user.name); // Might still be old name! (replication lag)
```

**Solutions:**

1. **Read from Master for Critical Data**
```javascript
// After write, read from master for some time
async function updateProfile(userId, newName) {
  await db.write('UPDATE users SET name = ? WHERE id = ?', [newName, userId]);

  // Read from master for next 5 seconds
  markUserForMasterReads(userId, 5000);
}
```

2. **Read Your Writes Consistency**
```javascript
class ConsistentDatabase {
  constructor(master, replicas) {
    this.master = master;
    this.replicas = replicas;
    this.recentWrites = new Map(); // userId → timestamp
  }

  async write(userId, query, params) {
    await this.master.query(query, params);
    this.recentWrites.set(userId, Date.now());
  }

  async read(userId, query, params) {
    const lastWrite = this.recentWrites.get(userId);

    // If user wrote recently, read from master
    if (lastWrite && Date.now() - lastWrite < 5000) {
      return await this.master.query(query, params);
    }

    // Otherwise, read from replica
    return await this.getNextReplica().query(query, params);
  }
}
```

### Multi-Master Replication

```
┌─────────────────────────────────────────┐
│  US-East Master                         │
│  ◄─────────────────────────────┐       │
│  ├───────────────────────────┐ │       │
└──┼───────────────────────────┼─┼───────┘
   │                           │ │
   │ Bi-directional           │ │
   │ Replication              │ │
   │                           │ │
┌──▼───────────────────────────▼─┼───────┐
│  EU-West Master                │       │
│  ├───────────────────────────┐ │       │
│  ◄─────────────────────────────┘       │
└─────────────────────────────────────────┘
```

**Benefits:**
- ✅ Write scalability (multiple write points)
- ✅ Low latency (write to local master)
- ✅ High availability (no single point of failure)

**Challenges:**
- ❌ Conflict resolution (same data updated in two places)
- ❌ Complex to implement
- ❌ Data consistency is difficult

**Conflict Example:**

```
Time: T0

┌──────────────────┐         ┌──────────────────┐
│  US Master       │         │  EU Master       │
│  User balance:   │         │  User balance:   │
│  $100            │         │  $100            │
└──────────────────┘         └──────────────────┘

Time: T1

User deducts $30 in US      User deducts $40 in EU
Balance = $70               Balance = $60

Time: T2 (after replication)

Conflict! Which balance is correct?
- Last Write Wins (LWW): Use timestamp → $60
- Custom Logic: Keep both, flag for review
- Vector Clocks: Detect concurrent writes
```

**Conflict Resolution Strategies:**

```javascript
// 1. Last Write Wins (LWW)
class LWWResolver {
  resolve(value1, timestamp1, value2, timestamp2) {
    return timestamp1 > timestamp2 ? value1 : value2;
  }
}

// 2. Application-Specific Logic
class AccountBalanceResolver {
  resolve(balance1, balance2) {
    // For account balance, keep minimum (safer)
    return Math.min(balance1, balance2);
  }
}

// 3. Keep Both (for shopping cart)
class ShoppingCartResolver {
  resolve(cart1, cart2) {
    // Merge both carts
    return [...new Set([...cart1, ...cart2])];
  }
}
```

---

## 3.4 Database Indexing

Indexes dramatically improve read performance but have trade-offs.

### How Indexes Work

**Without Index:**
```
Query: SELECT * FROM users WHERE email = 'alice@example.com'

┌────┬──────────┬─────────────────────┐
│ id │   name   │       email         │
├────┼──────────┼─────────────────────┤
│ 1  │  Alice   │ alice@example.com   │ ← Check
│ 2  │  Bob     │ bob@example.com     │ ← Check
│ 3  │  Charlie │ charlie@example.com │ ← Check
│... │  ...     │ ...                 │ ← Check all!
│ 1M │  Zoe     │ zoe@example.com     │ ← Check
└────┴──────────┴─────────────────────┘

Full table scan: O(n) - Slow! 🐌
```

**With Index:**
```
Query: SELECT * FROM users WHERE email = 'alice@example.com'

Index (B-Tree):
                [m]
               /   \
        [a-l]         [n-z]
       /     \       /     \
   [a-e]   [f-l]  [n-s]   [t-z]
     |
[alice@example.com] → Row pointer → id: 1

Index lookup: O(log n) - Fast! 🚀
```

### Index Types

#### 1. B-Tree Index (Default)

**Best for:**
- Equality searches: `WHERE id = 5`
- Range searches: `WHERE age BETWEEN 20 AND 30`
- Sorting: `ORDER BY created_at`

```sql
-- Create B-Tree index
CREATE INDEX idx_users_email ON users(email);

-- Queries that use this index
SELECT * FROM users WHERE email = 'alice@example.com'; ✓
SELECT * FROM users WHERE email LIKE 'alice%'; ✓
SELECT * FROM users WHERE email > 'alice@example.com'; ✓
```

#### 2. Hash Index

**Best for:**
- Exact matches only
- Very fast (O(1))

```sql
-- Create hash index (PostgreSQL)
CREATE INDEX idx_users_email_hash ON users USING HASH (email);

-- Works
SELECT * FROM users WHERE email = 'alice@example.com'; ✓

-- Doesn't work
SELECT * FROM users WHERE email LIKE 'alice%'; ✗
SELECT * FROM users WHERE email > 'alice@example.com'; ✗
```

#### 3. Composite Index

**Multiple columns in one index.**

```sql
-- Index on (user_id, created_at)
CREATE INDEX idx_posts_user_created
ON posts(user_id, created_at);

-- Uses index (left-most column rule)
SELECT * FROM posts
WHERE user_id = 123
ORDER BY created_at DESC; ✓

SELECT * FROM posts
WHERE user_id = 123
AND created_at > '2026-01-01'; ✓

-- Doesn't use index (skips left-most column)
SELECT * FROM posts
WHERE created_at > '2026-01-01'; ✗
```

**Left-Most Prefix Rule:**
```
Index: (A, B, C)

Uses index:
- WHERE A = ?
- WHERE A = ? AND B = ?
- WHERE A = ? AND B = ? AND C = ?

Doesn't use index:
- WHERE B = ?
- WHERE C = ?
- WHERE B = ? AND C = ?
```

#### 4. Covering Index

**Index includes all columns needed by query.**

```sql
-- Query
SELECT user_id, created_at
FROM posts
WHERE user_id = 123
ORDER BY created_at DESC;

-- Normal index
CREATE INDEX idx_posts_user ON posts(user_id);
-- Query still needs to access table rows

-- Covering index
CREATE INDEX idx_posts_user_created_covering
ON posts(user_id, created_at);
-- Query satisfied entirely from index! (faster)
```

### Index Trade-offs

```
┌──────────────────────────────────────────────┐
│ Benefits                                     │
├──────────────────────────────────────────────┤
│ ✅ Much faster reads (log n vs n)           │
│ ✅ Faster sorting                            │
│ ✅ Enforce uniqueness (unique indexes)       │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│ Costs                                        │
├──────────────────────────────────────────────┤
│ ❌ Slower writes (must update index)         │
│ ❌ More storage space (duplicate data)       │
│ ❌ Memory usage (indexes cached in RAM)      │
└──────────────────────────────────────────────┘
```

**Performance Impact:**

```
Without index:
- Read: 1000ms (full table scan)
- Write: 10ms

With index:
- Read: 5ms (index lookup) ← 200x faster!
- Write: 15ms (update table + index) ← 1.5x slower

Write-heavy workload → Fewer indexes
Read-heavy workload → More indexes
```

### Index Best Practices

**1. Index Foreign Keys**
```sql
-- Always index foreign keys for JOINs
CREATE INDEX idx_orders_user_id ON orders(user_id);

SELECT orders.*, users.name
FROM orders
JOIN users ON orders.user_id = users.id
WHERE orders.status = 'pending';
-- Fast! (uses index for JOIN)
```

**2. Index WHERE Clauses**
```sql
-- If you frequently query by status
CREATE INDEX idx_orders_status ON orders(status);

SELECT * FROM orders WHERE status = 'pending';
-- Fast!
```

**3. Don't Index Low-Cardinality Columns**
```sql
-- Bad: gender has only 2-3 values
CREATE INDEX idx_users_gender ON users(gender); ❌

-- Database will often skip this index (not selective enough)
```

**4. Analyze Query Patterns**
```sql
-- PostgreSQL: See query plan
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'alice@example.com';

-- MySQL: See query plan
EXPLAIN
SELECT * FROM users WHERE email = 'alice@example.com';
```

---

## 3.5 Real-World Data Modeling

### Example 1: Instagram-like Photo Sharing

**Requirements:**
- Users can post photos
- Photos can have likes and comments
- Need to show user's photo feed
- Need to show photo details with comments

**Schema Design (SQL):**

```sql
-- Users table
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  username VARCHAR(30) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);

-- Photos table
CREATE TABLE photos (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  url VARCHAR(500) NOT NULL,
  caption TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_photos_user_id ON photos(user_id);
CREATE INDEX idx_photos_created_at ON photos(created_at DESC);

-- Likes table
CREATE TABLE likes (
  user_id BIGINT NOT NULL REFERENCES users(id),
  photo_id BIGINT NOT NULL REFERENCES photos(id),
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (user_id, photo_id)
);
CREATE INDEX idx_likes_photo_id ON likes(photo_id);

-- Comments table
CREATE TABLE comments (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  photo_id BIGINT NOT NULL REFERENCES photos(id),
  text TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_comments_photo_id ON comments(photo_id);

-- Followers table (for feed)
CREATE TABLE followers (
  follower_id BIGINT NOT NULL REFERENCES users(id),
  following_id BIGINT NOT NULL REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (follower_id, following_id)
);
CREATE INDEX idx_followers_following_id ON followers(following_id);
```

**Query: User Feed (photos from people I follow)**

```sql
-- Get latest 20 photos from people I follow
SELECT
  p.*,
  u.username,
  u.profile_pic,
  COUNT(DISTINCT l.user_id) as like_count,
  COUNT(DISTINCT c.id) as comment_count
FROM photos p
JOIN users u ON p.user_id = u.id
JOIN followers f ON p.user_id = f.following_id
LEFT JOIN likes l ON p.id = l.photo_id
LEFT JOIN comments c ON p.id = c.photo_id
WHERE f.follower_id = 123  -- Current user
GROUP BY p.id, u.id
ORDER BY p.created_at DESC
LIMIT 20;
```

**Problem: This query is slow at scale!**

**Solution: Fanout on Write (like Twitter)**

```sql
-- Pre-computed feed table
CREATE TABLE user_feeds (
  user_id BIGINT NOT NULL,
  photo_id BIGINT NOT NULL,
  created_at TIMESTAMP NOT NULL,
  PRIMARY KEY (user_id, created_at, photo_id)
);
CREATE INDEX idx_user_feeds ON user_feeds(user_id, created_at DESC);

-- When user posts photo, insert into all followers' feeds
```

```javascript
async function postPhoto(userId, photoUrl, caption) {
  // 1. Insert photo
  const photo = await db.query(
    'INSERT INTO photos (user_id, url, caption) VALUES ($1, $2, $3) RETURNING id',
    [userId, photoUrl, caption]
  );

  // 2. Get followers
  const followers = await db.query(
    'SELECT follower_id FROM followers WHERE following_id = $1',
    [userId]
  );

  // 3. Fanout: Insert into each follower's feed
  const feedInserts = followers.map(f =>
    db.query(
      'INSERT INTO user_feeds (user_id, photo_id, created_at) VALUES ($1, $2, NOW())',
      [f.follower_id, photo.id]
    )
  );

  await Promise.all(feedInserts);

  return photo;
}

// Getting feed is now super fast!
async function getFeed(userId, limit = 20) {
  return await db.query(`
    SELECT p.*, u.username
    FROM user_feeds uf
    JOIN photos p ON uf.photo_id = p.id
    JOIN users u ON p.user_id = u.id
    WHERE uf.user_id = $1
    ORDER BY uf.created_at DESC
    LIMIT $2
  `, [userId, limit]);
}
```

### Example 2: Uber-like Ride Matching

**Requirements:**
- Drivers report location updates
- Riders request rides
- Match rider with nearby drivers
- Track ride status

**Schema Design:**

```sql
-- Drivers table
CREATE TABLE drivers (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  phone VARCHAR(20) UNIQUE NOT NULL,
  status VARCHAR(20) DEFAULT 'offline', -- offline, available, busy
  current_lat DECIMAL(10, 8),
  current_lng DECIMAL(11, 8),
  last_location_update TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Spatial index for location queries
CREATE INDEX idx_drivers_location
ON drivers USING GIST (
  ll_to_earth(current_lat, current_lng)
);

-- Riders table
CREATE TABLE riders (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  phone VARCHAR(20) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Rides table
CREATE TABLE rides (
  id BIGSERIAL PRIMARY KEY,
  rider_id BIGINT NOT NULL REFERENCES riders(id),
  driver_id BIGINT REFERENCES drivers(id),
  status VARCHAR(20) DEFAULT 'requested', -- requested, accepted, in_progress, completed, cancelled
  pickup_lat DECIMAL(10, 8) NOT NULL,
  pickup_lng DECIMAL(11, 8) NOT NULL,
  dropoff_lat DECIMAL(10, 8),
  dropoff_lng DECIMAL(11, 8),
  fare DECIMAL(10, 2),
  requested_at TIMESTAMP DEFAULT NOW(),
  accepted_at TIMESTAMP,
  started_at TIMESTAMP,
  completed_at TIMESTAMP
);
CREATE INDEX idx_rides_rider_id ON rides(rider_id);
CREATE INDEX idx_rides_driver_id ON rides(driver_id);
CREATE INDEX idx_rides_status ON rides(status);
```

**Query: Find Nearby Drivers**

```sql
-- Find available drivers within 5km
SELECT
  id,
  name,
  earth_distance(
    ll_to_earth(current_lat, current_lng),
    ll_to_earth(37.7749, -122.4194)  -- Rider location
  ) / 1000 AS distance_km
FROM drivers
WHERE status = 'available'
  AND earth_distance(
    ll_to_earth(current_lat, current_lng),
    ll_to_earth(37.7749, -122.4194)
  ) < 5000  -- 5km radius
ORDER BY distance_km
LIMIT 10;
```

**Alternative: Use Redis for Real-Time Location**

```javascript
const redis = require('redis');
const client = redis.createClient();

class LocationService {
  // Driver updates location
  async updateDriverLocation(driverId, lat, lng) {
    await client.geoadd(
      'drivers:locations',
      lng, lat, driverId
    );

    await client.hset(`driver:${driverId}`, {
      status: 'available',
      lastUpdate: Date.now()
    });
  }

  // Find nearby drivers
  async findNearbyDrivers(lat, lng, radiusKm = 5) {
    const drivers = await client.georadius(
      'drivers:locations',
      lng, lat,
      radiusKm, 'km',
      'WITHDIST', 'ASC', 'COUNT', 10
    );

    // Filter only available drivers
    const availableDrivers = [];
    for (const [driverId, distance] of drivers) {
      const status = await client.hget(`driver:${driverId}`, 'status');
      if (status === 'available') {
        availableDrivers.push({ driverId, distance });
      }
    }

    return availableDrivers;
  }
}
```

---

## 3.6 Key Takeaways

### Database Selection Checklist

```
Choose SQL if:
✓ ACID transactions required
✓ Complex relationships and JOINs
✓ Data integrity is critical
✓ Schema is stable

Choose NoSQL if:
✓ Need horizontal scalability
✓ Schema flexibility needed
✓ High write throughput
✓ Eventually consistent is OK
```

### Sharding Decision Tree

```
Can single database handle load?
├─ Yes → Don't shard (complexity not worth it)
└─ No → Continue

Can vertical scaling help?
├─ Yes → Scale up first (simpler)
└─ No → Need sharding

Choose sharding strategy:
├─ Even distribution needed → Hash sharding
├─ Range queries common → Range sharding
├─ Geographic distribution → Geo sharding
└─ Maximum flexibility → Directory sharding
```

### Index Guidelines

```
Index these:
✓ Primary keys (automatic)
✓ Foreign keys
✓ Columns in WHERE clauses
✓ Columns in JOIN conditions
✓ Columns in ORDER BY

Don't index:
✗ Low-cardinality columns (< 100 unique values)
✗ Columns rarely queried
✗ Small tables (< 1000 rows)
✗ Columns with frequent updates
```

---

## 3.7 Interview Questions

### Basic:
1. What's the difference between SQL and NoSQL?
2. Explain database replication and its benefits.
3. What is a database index and why is it important?

### Intermediate:
1. Compare hash-based and range-based sharding.
2. How do you handle replication lag?
3. Explain the trade-offs of composite indexes.

### Advanced:
1. Design a sharding strategy for a social network with 1B users.
2. How would you handle distributed transactions across shards?
3. Design the database schema for Uber's ride matching system.

---

**Next Chapter:** [Chapter 4: Caching Strategies](./SystemDesign-04-Caching.md)

In the next chapter, we'll explore:
- Cache-aside vs Write-through vs Write-behind
- Cache invalidation strategies
- Redis patterns and use cases
- CDN caching best practices
- Multi-level caching architectures
