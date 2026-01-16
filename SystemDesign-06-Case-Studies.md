# Chapter 6: Real-World Case Studies

## 6.1 Introduction

This chapter walks through complete system designs for popular applications. Each case study follows a structured approach:

1. **Requirements** - Functional and non-functional
2. **Capacity Estimation** - Scale calculations
3. **API Design** - Endpoints and contracts
4. **Database Design** - Schema and data models
5. **High-Level Architecture** - Component diagram
6. **Deep Dive** - Critical components
7. **Optimizations** - Performance and scalability
8. **Trade-offs** - Design decisions

---

## 6.2 Case Study 1: URL Shortener (like Bitly)

### Requirements

**Functional:**
- Generate short URL from long URL
- Redirect short URL to original URL
- Custom short URLs (optional)
- URL expiration (optional)
- Analytics (click count, referrers, locations)

**Non-Functional:**
- 100M URLs created per month
- 10B redirects per month (100:1 read-to-write ratio)
- Low latency (< 100ms for redirects)
- 99.9% availability
- URLs never lost (high durability)

### Capacity Estimation

```
Write Volume:
100M URLs/month = 100M / (30 * 24 * 60 * 60)
                ≈ 40 writes/second
                ≈ 80 writes/second (peak)

Read Volume:
10B redirects/month = 10B / (30 * 24 * 60 * 60)
                    ≈ 4000 reads/second
                    ≈ 8000 reads/second (peak)

Storage:
- URL record: ~500 bytes
  * short_code: 7 bytes
  * original_url: 200 bytes
  * user_id: 8 bytes
  * created_at: 8 bytes
  * expires_at: 8 bytes
  * metadata: ~270 bytes

- Per month: 100M * 500 bytes = 50 GB
- 5 years: 50 GB * 60 months = 3 TB
- With analytics: 3 TB * 2 = 6 TB

Bandwidth:
- Write: 40 req/s * 500 bytes = 20 KB/s (negligible)
- Read: 4000 req/s * 500 bytes = 2 MB/s

Cache Size (80/20 rule):
- 20% of URLs get 80% of traffic
- 100M URLs * 20% = 20M URLs
- 20M * 500 bytes = 10 GB (easily fits in Redis)
```

### API Design

```javascript
// POST /api/shorten - Create short URL
Request:
{
  "url": "https://www.example.com/very/long/url",
  "customAlias": "mylink",  // optional
  "expiresAt": "2026-12-31T23:59:59Z"  // optional
}

Response: 201 Created
{
  "shortUrl": "https://short.ly/abc123",
  "shortCode": "abc123",
  "originalUrl": "https://www.example.com/very/long/url",
  "createdAt": "2026-01-16T10:30:00Z",
  "expiresAt": "2026-12-31T23:59:59Z"
}

// GET /:shortCode - Redirect to original URL
Response: 302 Found
Location: https://www.example.com/very/long/url

// GET /api/stats/:shortCode - Get analytics
Response: 200 OK
{
  "shortCode": "abc123",
  "originalUrl": "https://www.example.com/very/long/url",
  "totalClicks": 1234,
  "clicksByDate": {
    "2026-01-15": 100,
    "2026-01-16": 134
  },
  "topReferrers": [
    { "referrer": "twitter.com", "count": 500 },
    { "referrer": "facebook.com", "count": 300 }
  ],
  "topCountries": [
    { "country": "US", "count": 600 },
    { "country": "UK", "count": 200 }
  ]
}

// DELETE /api/urls/:shortCode - Delete short URL
Response: 204 No Content
```

### Short Code Generation

**Option 1: Base62 Encoding**

```javascript
class ShortCodeGenerator {
  constructor() {
    this.charset = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    this.base = this.charset.length; // 62
  }

  // Generate short code from ID
  encode(id) {
    if (id === 0) return this.charset[0];

    let shortCode = '';
    while (id > 0) {
      shortCode = this.charset[id % this.base] + shortCode;
      id = Math.floor(id / this.base);
    }
    return shortCode;
  }

  // Decode short code to ID
  decode(shortCode) {
    let id = 0;
    for (let i = 0; i < shortCode.length; i++) {
      id = id * this.base + this.charset.indexOf(shortCode[i]);
    }
    return id;
  }
}

// Usage
const generator = new ShortCodeGenerator();

// Auto-increment ID: 1
const shortCode1 = generator.encode(1); // "1"

// Auto-increment ID: 1000000
const shortCode2 = generator.encode(1000000); // "4c92"

// Auto-increment ID: 1000000000
const shortCode3 = generator.encode(1000000000); // "15ftgG"

// How many URLs can we store with 7 characters?
// 62^7 = 3.5 trillion URLs ✓
```

**Option 2: Hash + Collision Resolution**

```javascript
const crypto = require('crypto');

class HashBasedGenerator {
  async generate(url) {
    // Generate hash
    const hash = crypto.createHash('md5').update(url).digest('hex');

    // Take first 7 characters
    let shortCode = hash.substring(0, 7);

    // Check for collision
    let attempt = 0;
    while (await this.exists(shortCode)) {
      // Collision! Try next 7 characters
      attempt++;
      shortCode = hash.substring(attempt, attempt + 7);

      // If we run out of hash, append counter
      if (attempt + 7 > hash.length) {
        shortCode = hash.substring(0, 6) + attempt.toString(36);
      }
    }

    return shortCode;
  }

  async exists(shortCode) {
    return await db.urls.findOne({ shortCode });
  }
}
```

**Comparison:**

```
┌──────────────────┬─────────────┬────────────────┐
│ Method           │ Pros        │ Cons           │
├──────────────────┼─────────────┼────────────────┤
│ Base62 Encoding  │ - Unique    │ - Sequential   │
│                  │ - Fast      │   (predictable)│
│                  │ - No lookup │ - Need DB ID   │
├──────────────────┼─────────────┼────────────────┤
│ Hash + Resolve   │ - Random    │ - Collisions   │
│                  │ - No DB ID  │ - DB lookup    │
└──────────────────┴─────────────┴────────────────┘
```

### Database Schema

**PostgreSQL:**

```sql
-- URLs table
CREATE TABLE urls (
  id BIGSERIAL PRIMARY KEY,
  short_code VARCHAR(10) UNIQUE NOT NULL,
  original_url TEXT NOT NULL,
  user_id BIGINT REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP,
  is_active BOOLEAN DEFAULT true
);

CREATE INDEX idx_urls_short_code ON urls(short_code);
CREATE INDEX idx_urls_user_id ON urls(user_id);
CREATE INDEX idx_urls_created_at ON urls(created_at DESC);

-- Analytics table (partitioned by date)
CREATE TABLE url_clicks (
  id BIGSERIAL,
  short_code VARCHAR(10) NOT NULL,
  clicked_at TIMESTAMP DEFAULT NOW(),
  referrer VARCHAR(500),
  user_agent TEXT,
  ip_address INET,
  country VARCHAR(2)
) PARTITION BY RANGE (clicked_at);

-- Create partitions (one per month)
CREATE TABLE url_clicks_2026_01 PARTITION OF url_clicks
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE INDEX idx_clicks_short_code ON url_clicks(short_code, clicked_at DESC);
```

### High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│                   Client                        │
└────────────────────┬────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────┐
│              Load Balancer (DNS)                │
│           Route to nearest region               │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ↓            ↓            ↓
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Region 1 │  │ Region 2 │  │ Region 3 │
│ US-East  │  │ EU-West  │  │ Asia-SE  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     ↓             ↓             ↓
┌─────────────────────────────────────┐
│         Cache Layer (Redis)         │
│   - Hot URLs (95%+ cache hit)       │
│   - TTL: 24 hours                   │
└────────────────┬────────────────────┘
                 │ Cache Miss
                 ↓
┌─────────────────────────────────────┐
│      Application Servers            │
│   - URL creation                    │
│   - Redirect logic                  │
│   - Analytics processing            │
└────────────────┬────────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
    ↓            ↓            ↓
┌────────┐  ┌─────────┐  ┌────────────┐
│Database│  │ Object  │  │  Message   │
│(Primary│  │ Storage │  │  Queue     │
│Replica)│  │ (S3)    │  │  (Kafka)   │
└────────┘  └─────────┘  └────────────┘
```

### Implementation

**URL Creation Service:**

```javascript
class URLShortenerService {
  constructor(db, cache, idGenerator) {
    this.db = db;
    this.cache = cache;
    this.idGenerator = idGenerator;
  }

  async createShortUrl(originalUrl, userId, customAlias = null, expiresAt = null) {
    // Validate URL
    if (!this.isValidUrl(originalUrl)) {
      throw new Error('Invalid URL');
    }

    // Check if custom alias is available
    if (customAlias) {
      const exists = await this.db.urls.findOne({ short_code: customAlias });
      if (exists) {
        throw new Error('Custom alias already taken');
      }
    }

    // Generate short code
    const shortCode = customAlias || await this.generateShortCode();

    // Create URL record
    const url = await this.db.urls.create({
      short_code: shortCode,
      original_url: originalUrl,
      user_id: userId,
      expires_at: expiresAt,
      created_at: new Date()
    });

    // Cache it
    await this.cache.setex(
      `url:${shortCode}`,
      86400, // 24 hours
      JSON.stringify({
        originalUrl: originalUrl,
        expiresAt: expiresAt
      })
    );

    return {
      shortUrl: `https://short.ly/${shortCode}`,
      shortCode,
      originalUrl,
      createdAt: url.created_at,
      expiresAt
    };
  }

  async generateShortCode() {
    // Get next ID from distributed ID generator
    const id = await this.idGenerator.getNextId();

    // Encode to base62
    return this.base62Encode(id);
  }

  async redirect(shortCode) {
    // Try cache first
    const cached = await this.cache.get(`url:${shortCode}`);

    if (cached) {
      const data = JSON.parse(cached);

      // Check expiration
      if (data.expiresAt && new Date(data.expiresAt) < new Date()) {
        throw new Error('URL expired');
      }

      // Track click asynchronously
      this.trackClick(shortCode);

      return data.originalUrl;
    }

    // Cache miss - query database
    const url = await this.db.urls.findOne({
      short_code: shortCode,
      is_active: true
    });

    if (!url) {
      throw new Error('URL not found');
    }

    // Check expiration
    if (url.expires_at && url.expires_at < new Date()) {
      throw new Error('URL expired');
    }

    // Cache for next time
    await this.cache.setex(
      `url:${shortCode}`,
      86400,
      JSON.stringify({
        originalUrl: url.original_url,
        expiresAt: url.expires_at
      })
    );

    // Track click asynchronously
    this.trackClick(shortCode);

    return url.original_url;
  }

  async trackClick(shortCode, metadata = {}) {
    // Send to message queue (async)
    await this.messageQueue.publish('url.clicked', {
      shortCode,
      timestamp: Date.now(),
      referrer: metadata.referrer,
      userAgent: metadata.userAgent,
      ipAddress: metadata.ipAddress
    });

    // Increment counter in cache (fast)
    await this.cache.incr(`clicks:${shortCode}`);
  }

  base62Encode(num) {
    const charset = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    let result = '';

    while (num > 0) {
      result = charset[num % 62] + result;
      num = Math.floor(num / 62);
    }

    return result || '0';
  }

  isValidUrl(url) {
    try {
      new URL(url);
      return true;
    } catch {
      return false;
    }
  }
}
```

**Analytics Service:**

```javascript
class AnalyticsService {
  constructor(db, cache, messageQueue) {
    this.db = db;
    this.cache = cache;
    this.messageQueue = messageQueue;

    // Start consuming click events
    this.startConsumer();
  }

  startConsumer() {
    this.messageQueue.subscribe('url.clicked', async (event) => {
      try {
        // Extract location from IP
        const country = await this.getCountryFromIP(event.ipAddress);

        // Write to database (batched for performance)
        await this.db.url_clicks.create({
          short_code: event.shortCode,
          clicked_at: new Date(event.timestamp),
          referrer: event.referrer,
          user_agent: event.userAgent,
          ip_address: event.ipAddress,
          country
        });

        // Update aggregated stats in cache
        const date = new Date(event.timestamp).toISOString().split('T')[0];
        await this.cache.hincrby(`stats:${event.shortCode}:${date}`, 'clicks', 1);

        if (event.referrer) {
          await this.cache.zincrby(
            `stats:${event.shortCode}:referrers`,
            1,
            event.referrer
          );
        }

        if (country) {
          await this.cache.zincrby(
            `stats:${event.shortCode}:countries`,
            1,
            country
          );
        }
      } catch (error) {
        console.error('Failed to track click:', error);
        // Could retry or send to dead letter queue
      }
    });
  }

  async getStats(shortCode) {
    // Get total clicks from cache
    const totalClicks = await this.cache.get(`clicks:${shortCode}`) || 0;

    // Get top referrers
    const topReferrers = await this.cache.zrevrange(
      `stats:${shortCode}:referrers`,
      0,
      9,
      'WITHSCORES'
    );

    // Get top countries
    const topCountries = await this.cache.zrevrange(
      `stats:${shortCode}:countries`,
      0,
      9,
      'WITHSCORES'
    );

    // Get clicks by date (last 30 days)
    const clicksByDate = await this.getClicksByDate(shortCode, 30);

    return {
      shortCode,
      totalClicks: parseInt(totalClicks),
      clicksByDate,
      topReferrers: this.formatZsetResults(topReferrers),
      topCountries: this.formatZsetResults(topCountries)
    };
  }

  async getClicksByDate(shortCode, days) {
    const result = {};
    const today = new Date();

    for (let i = 0; i < days; i++) {
      const date = new Date(today);
      date.setDate(date.getDate() - i);
      const dateStr = date.toISOString().split('T')[0];

      const clicks = await this.cache.hget(`stats:${shortCode}:${dateStr}`, 'clicks');
      result[dateStr] = parseInt(clicks || 0);
    }

    return result;
  }

  formatZsetResults(zsetResults) {
    const formatted = [];
    for (let i = 0; i < zsetResults.length; i += 2) {
      formatted.push({
        value: zsetResults[i],
        count: parseInt(zsetResults[i + 1])
      });
    }
    return formatted;
  }

  async getCountryFromIP(ipAddress) {
    // Use GeoIP service (MaxMind, IP2Location, etc.)
    try {
      const response = await fetch(`https://geoip.service/lookup/${ipAddress}`);
      const data = await response.json();
      return data.country_code;
    } catch (error) {
      return null;
    }
  }
}
```

### Optimizations

**1. Caching Strategy:**
```
- Cache hot URLs (95%+ hit rate)
- TTL: 24 hours for URL mappings
- Pre-warm cache for popular URLs
- Use Redis Cluster for horizontal scaling
```

**2. Database Optimization:**
```
- Partition analytics table by date
- Archive old data to cold storage
- Use read replicas for analytics queries
- Index on short_code for fast lookups
```

**3. CDN:**
```
- Static assets on CDN
- API responses cacheable (short TTL)
- Geographic distribution
```

### Trade-offs

**Sequential vs Random IDs:**
```
Sequential (Base62 encoding):
✓ Simple, predictable
✓ No collisions
✗ Guessable (security concern)
✗ Reveals scale

Random (Hash-based):
✓ Unpredictable
✓ No ID generation service needed
✗ Collision handling needed
✗ Database lookup on creation

Decision: Use Base62 for performance,
         add random salt for security
```

**Write-through vs Write-behind caching:**
```
Write-through:
✓ Always consistent
✗ Slower writes

Write-behind:
✓ Fast writes
✗ Risk of data loss

Decision: Write-through for URL creation
         (durability critical)
```

---

## 6.3 Case Study 2: Twitter-like Social Media

### Requirements

**Functional:**
- Post tweets (text, images, videos)
- Follow/unfollow users
- Like and retweet
- Timeline (home feed + user profile)
- Search tweets
- Trending topics
- Notifications

**Non-Functional:**
- 500M daily active users
- 200M tweets per day
- Average tweet size: 300 bytes
- Average user follows: 200 people
- Timeline generation: < 500ms
- 99.99% availability

### Capacity Estimation

```
Users:
- 500M DAU
- 1B total users

Tweets:
- 200M tweets/day
- 200M / (24 * 60 * 60) ≈ 2300 tweets/second
- Peak: 5000 tweets/second

Timeline Reads:
- Each user checks timeline: 5 times/day
- 500M * 5 = 2.5B timeline requests/day
- 2.5B / (24 * 60 * 60) ≈ 29,000 requests/second
- Peak: 60,000 requests/second

Storage:
- Tweet: 300 bytes
- Media (separate): varies
- 200M tweets/day * 365 days * 300 bytes = 22 TB/year
- 5 years: 110 TB (tweets only)
- With media: ~500 TB/year

Timeline Storage (fanout):
- Average user follows 200 people
- Each tweet fanned out to 200 timelines
- 200M tweets/day * 200 = 40B timeline inserts/day
- Each entry: 20 bytes (tweet_id + timestamp)
- 40B * 20 bytes = 800 GB/day = 292 TB/year

Bandwidth:
- Write: 2300 tweets/s * 300 bytes = 690 KB/s
- Read: 29,000 timeline/s * 10 tweets * 300 bytes = 87 MB/s
```

### API Design

```javascript
// POST /api/tweets - Create tweet
Request:
{
  "text": "Hello world!",
  "mediaIds": ["img123", "img456"],
  "replyToId": "tweet789"  // optional
}

Response: 201 Created
{
  "id": "tweet123",
  "userId": "user456",
  "text": "Hello world!",
  "mediaUrls": ["https://cdn.../img123.jpg"],
  "createdAt": "2026-01-16T10:30:00Z",
  "likes": 0,
  "retweets": 0,
  "replies": 0
}

// GET /api/timeline/home - Get home timeline
Response: 200 OK
{
  "tweets": [
    {
      "id": "tweet123",
      "user": {
        "id": "user456",
        "username": "john_doe",
        "displayName": "John Doe",
        "avatarUrl": "https://cdn.../avatar.jpg"
      },
      "text": "Hello world!",
      "createdAt": "2026-01-16T10:30:00Z",
      "likes": 42,
      "retweets": 10,
      "isLiked": false,
      "isRetweeted": false
    }
  ],
  "nextCursor": "cursor123"
}

// GET /api/users/:userId/tweets - Get user's tweets
// POST /api/tweets/:tweetId/like - Like tweet
// POST /api/tweets/:tweetId/retweet - Retweet
// POST /api/users/:userId/follow - Follow user
// GET /api/search?q=query - Search tweets
```

### Database Schema

```sql
-- Users
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  display_name VARCHAR(100),
  bio TEXT,
  avatar_url VARCHAR(500),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);

-- Tweets
CREATE TABLE tweets (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  text VARCHAR(280),
  reply_to_id BIGINT REFERENCES tweets(id),
  created_at TIMESTAMP DEFAULT NOW(),
  likes_count INT DEFAULT 0,
  retweets_count INT DEFAULT 0,
  replies_count INT DEFAULT 0
);

CREATE INDEX idx_tweets_user_id ON tweets(user_id, created_at DESC);
CREATE INDEX idx_tweets_created_at ON tweets(created_at DESC);

-- Followers (who follows whom)
CREATE TABLE followers (
  follower_id BIGINT NOT NULL REFERENCES users(id),
  following_id BIGINT NOT NULL REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (follower_id, following_id)
);

CREATE INDEX idx_followers_following ON followers(following_id);

-- Likes
CREATE TABLE likes (
  user_id BIGINT NOT NULL REFERENCES users(id),
  tweet_id BIGINT NOT NULL REFERENCES tweets(id),
  created_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (user_id, tweet_id)
);

CREATE INDEX idx_likes_tweet_id ON likes(tweet_id);

-- Timeline Cache (Redis Lists)
-- Key: timeline:{user_id}
-- Value: List of tweet IDs (most recent first)
```

### High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│                 Load Balancer                   │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ↓            ↓            ↓
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   API        │ │   API        │ │   API        │
│   Servers    │ │   Servers    │ │   Servers    │
└───┬──────┬───┘ └───┬──────┬───┘ └───┬──────┬───┘
    │      │         │      │         │      │
    │      └─────────┼──────┘         │      │
    │                │                │      │
    ↓                ↓                ↓      ↓
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   Tweet     │ │  Timeline   │ │   Social    │
│   Service   │ │  Service    │ │   Graph     │
│             │ │             │ │   Service   │
└──────┬──────┘ └──────┬──────┘ └──────┬──────┘
       │               │               │
       ↓               ↓               ↓
┌──────────────────────────────────────────┐
│         Timeline Cache (Redis)           │
│  - Pre-computed timelines                │
│  - 800 tweets per user                   │
└──────────────────────────────────────────┘
       │               │               │
       ↓               ↓               ↓
┌──────────────────────────────────────────┐
│      Database (Sharded PostgreSQL)       │
│  - Tweets (sharded by user_id)          │
│  - Social graph (sharded by user_id)    │
└──────────────────────────────────────────┘
       │
       ↓
┌──────────────────────────────────────────┐
│      Object Storage (S3)                 │
│  - Images, videos                        │
└──────────────────────────────────────────┘
       │
       ↓
┌──────────────────────────────────────────┐
│      Search (Elasticsearch)              │
│  - Full-text search                      │
│  - Trending topics                       │
└──────────────────────────────────────────┘
```

### Tweet Service (Fanout on Write)

```javascript
class TweetService {
  constructor(db, cache, queue) {
    this.db = db;
    this.cache = cache;
    this.queue = queue;
  }

  async createTweet(userId, text, mediaIds = []) {
    // 1. Save tweet to database
    const tweet = await this.db.tweets.create({
      user_id: userId,
      text,
      media_ids: mediaIds,
      created_at: Date.now()
    });

    // 2. Get user info
    const user = await this.db.users.findById(userId);

    // 3. Check if user is celebrity (many followers)
    const followerCount = await this.getFollowerCount(userId);

    if (followerCount < 1000000) {
      // Regular user - fanout on write
      await this.fanoutToFollowers(userId, tweet.id);
    } else {
      // Celebrity - skip fanout (will merge on read)
      console.log(`Skipping fanout for celebrity user ${userId}`);
    }

    // 4. Index for search
    await this.queue.publish('tweet.created', {
      tweetId: tweet.id,
      userId,
      text,
      timestamp: tweet.created_at
    });

    return tweet;
  }

  async fanoutToFollowers(userId, tweetId) {
    // Get followers in batches
    const batchSize = 1000;
    let offset = 0;

    while (true) {
      const followers = await this.db.followers.find({
        following_id: userId
      })
      .skip(offset)
      .limit(batchSize);

      if (followers.length === 0) break;

      // Add to each follower's timeline (Redis list)
      const promises = followers.map(async (follower) => {
        await this.cache.lpush(`timeline:${follower.follower_id}`, tweetId);
        await this.cache.ltrim(`timeline:${follower.follower_id}`, 0, 799);
      });

      await Promise.all(promises);

      offset += batchSize;

      // If small number of followers, process synchronously
      // If large, send to queue for async processing
      if (followers.length === batchSize) {
        // More followers remaining, use queue
        await this.queue.publish('fanout.continue', {
          userId,
          tweetId,
          offset
        });
        break;
      }
    }
  }

  async getFollowerCount(userId) {
    const cached = await this.cache.get(`followers:count:${userId}`);
    if (cached) return parseInt(cached);

    const count = await this.db.followers.count({
      following_id: userId
    });

    await this.cache.setex(`followers:count:${userId}`, 3600, count);
    return count;
  }
}
```

### Timeline Service

```javascript
class TimelineService {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
  }

  async getHomeTimeline(userId, limit = 20, cursor = null) {
    // Get timeline from cache (pre-computed)
    const cachedTimelineIds = await this.cache.lrange(
      `timeline:${userId}`,
      0,
      limit - 1
    );

    if (cachedTimelineIds.length > 0) {
      // Get tweets from cache/database
      const tweets = await this.hydrateTweets(cachedTimelineIds, userId);

      // Merge with celebrity tweets (fanout on read)
      const celebrityTweets = await this.getCelebrityTweets(userId);

      // Merge and sort
      const allTweets = [...tweets, ...celebrityTweets]
        .sort((a, b) => b.created_at - a.created_at)
        .slice(0, limit);

      return {
        tweets: allTweets,
        nextCursor: this.generateCursor(allTweets)
      };
    }

    // Cache miss - build timeline from scratch
    return await this.buildTimelineFromScratch(userId, limit);
  }

  async hydrateTweets(tweetIds, viewerUserId) {
    // Get tweets from database
    const tweets = await this.db.tweets.find({
      id: { $in: tweetIds }
    });

    // Create a map for quick lookup
    const tweetMap = new Map(tweets.map(t => [t.id, t]));

    // Maintain order from tweetIds
    const orderedTweets = tweetIds
      .map(id => tweetMap.get(id))
      .filter(t => t !== undefined);

    // Get user info for each tweet
    const userIds = [...new Set(orderedTweets.map(t => t.user_id))];
    const users = await this.db.users.find({
      id: { $in: userIds }
    });
    const userMap = new Map(users.map(u => [u.id, u]));

    // Get viewer's likes and retweets
    const viewerLikes = await this.db.likes.find({
      user_id: viewerUserId,
      tweet_id: { $in: tweetIds }
    });
    const likedTweetIds = new Set(viewerLikes.map(l => l.tweet_id));

    // Enrich tweets
    return orderedTweets.map(tweet => ({
      ...tweet,
      user: userMap.get(tweet.user_id),
      isLiked: likedTweetIds.has(tweet.id),
      isRetweeted: false // TODO: check retweets
    }));
  }

  async getCelebrityTweets(userId) {
    // Get celebrities that user follows
    const celebrities = await this.cache.smembers(`following:celebrities:${userId}`);

    if (celebrities.length === 0) return [];

    // Get recent tweets from celebrities (last 24 hours)
    const oneDayAgo = Date.now() - 86400000;
    const celebrityTweets = await this.db.tweets.find({
      user_id: { $in: celebrities },
      created_at: { $gte: oneDayAgo }
    })
    .sort({ created_at: -1 })
    .limit(50);

    return celebrityTweets;
  }

  async buildTimelineFromScratch(userId, limit) {
    // Get users that current user follows
    const following = await this.db.followers.find({
      follower_id: userId
    });

    const followingIds = following.map(f => f.following_id);

    // Get recent tweets from followed users
    const tweets = await this.db.tweets.find({
      user_id: { $in: followingIds }
    })
    .sort({ created_at: -1 })
    .limit(limit);

    // Cache the timeline
    const tweetIds = tweets.map(t => t.id);
    if (tweetIds.length > 0) {
      await this.cache.del(`timeline:${userId}`);
      await this.cache.rpush(`timeline:${userId}`, ...tweetIds);
      await this.cache.expire(`timeline:${userId}`, 3600);
    }

    return {
      tweets: await this.hydrateTweets(tweetIds, userId),
      nextCursor: this.generateCursor(tweets)
    };
  }

  generateCursor(tweets) {
    if (tweets.length === 0) return null;
    const lastTweet = tweets[tweets.length - 1];
    return Buffer.from(JSON.stringify({
      id: lastTweet.id,
      timestamp: lastTweet.created_at
    })).toString('base64');
  }
}
```

### Optimizations

**1. Hybrid Fanout Strategy:**
```javascript
// For regular users: Fanout on write (fast reads)
if (followerCount < 1_000_000) {
  await fanoutToFollowers(userId, tweetId);
}

// For celebrities: Fanout on read (avoid write amplification)
// Merge celebrity tweets during timeline generation
```

**2. Timeline Caching:**
```
- Pre-compute top 800 tweets per user
- Store in Redis list
- TTL: No expiry (manually invalidate)
- Lazy loading: Build on first access
```

**3. Database Sharding:**
```
Shard tweets by user_id:
- All user's tweets on same shard
- Efficient user timeline queries
- Trade-off: Global timeline harder

Shard social graph by follower_id:
- Efficient "who do I follow" queries
- Trade-off: "who follows me" spans shards
```

### Trade-offs

**Fanout on Write vs Fanout on Read:**

```
Fanout on Write:
✓ Fast reads (pre-computed)
✓ Simple read logic
✗ Write amplification (1 tweet → 200 timeline inserts)
✗ Celebrity problem (1 tweet → 100M inserts!)

Fanout on Read:
✓ Fast writes (no fanout)
✓ No celebrity problem
✗ Slow reads (compute on demand)
✗ Complex read logic

Decision: Hybrid approach
- Regular users: Fanout on write
- Celebrities: Fanout on read (merge during timeline generation)
```

**Consistency vs Availability:**

```
Strong Consistency:
✓ Always see latest tweets
✗ Slower, lower availability

Eventual Consistency:
✓ Fast, high availability
✗ May see slightly stale timeline

Decision: Eventual consistency for timeline
         (acceptable for social media)
         Strong consistency for likes/retweets
         (prevent double-counting)
```

---

## 6.4 Case Study 3: Netflix-like Video Streaming

### Requirements

**Functional:**
- Upload videos
- Transcode to multiple qualities
- Stream videos (adaptive bitrate)
- User profiles and watch history
- Recommendations
- Search and browse catalog

**Non-Functional:**
- 200M subscribers
- 100M concurrent streams (peak)
- 10,000 hours of content added daily
- Average video: 1 hour, 1 GB (compressed)
- 99.99% availability
- Start playback in < 1 second
- No buffering during playback

### Capacity Estimation

```
Users & Viewing:
- 200M subscribers
- 100M concurrent streams (peak)
- Average viewing: 2 hours/day per user
- 200M * 2 hours = 400M hours watched/day

Storage:
- 10,000 hours added daily
- Average: 1 GB/hour (source quality)
- 10,000 hours * 1 GB = 10 TB/day
- Transcoded (4 qualities): 10 TB * 4 = 40 TB/day
- 1 year: 40 TB * 365 = 14.6 PB/year

Bandwidth:
- 100M concurrent streams
- Average bitrate: 5 Mbps (1080p)
- 100M * 5 Mbps = 500 Tbps
- This is massive! Need global CDN network

Content Delivery:
- 90% of viewing is top 10% of content
- Cache this on edge servers (close to users)
- Pre-position popular content before demand
```

### Architecture

```
┌─────────────────────────────────────────────────┐
│              Client Applications                │
│     (Web, iOS, Android, Smart TV, etc.)        │
└────────────────────┬────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────┐
│            API Gateway / CDN                    │
│  - Route to nearest region                      │
│  - Authentication                               │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ↓            ↓            ↓
┌──────────────────────────────────────────────────┐
│         Edge Servers (Open Connect)              │
│  - 1000+ locations in ISP networks               │
│  - Cache popular content (90% cache hit)         │
│  - Adaptive bitrate streaming                    │
└────────────────────┬─────────────────────────────┘
                     │ Cache Miss (10%)
                     ↓
┌──────────────────────────────────────────────────┐
│          Regional Caches                         │
│  - Warm content                                  │
│  - 100+ locations                                │
└────────────────────┬─────────────────────────────┘
                     │ Cache Miss (1%)
                     ↓
┌──────────────────────────────────────────────────┐
│        Origin Storage (AWS S3)                   │
│  - All content (master copies)                   │
│  - Multiple quality levels                       │
└──────────────────────────────────────────────────┘

Backend Services:

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Metadata   │  │ Transcoding  │  │Recommendation│
│   Service    │  │   Service    │  │   Service    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                  │
       └─────────────────┼──────────────────┘
                         │
                         ↓
              ┌────────────────────┐
              │  Database Cluster  │
              │  - User profiles   │
              │  - Watch history   │
              │  - Catalog         │
              └────────────────────┘
```

### Video Upload & Transcoding Pipeline

```javascript
class VideoUploadService {
  async uploadVideo(videoFile, metadata) {
    // 1. Upload original to S3
    const videoId = generateUUID();
    const s3Key = `videos/originals/${videoId}.mp4`;

    await s3.upload({
      Bucket: 'netflix-videos',
      Key: s3Key,
      Body: videoFile,
      StorageClass: 'STANDARD_IA' // Infrequent access (originals rarely accessed)
    });

    // 2. Create metadata record
    const video = await db.videos.create({
      id: videoId,
      title: metadata.title,
      description: metadata.description,
      originalUrl: `s3://netflix-videos/${s3Key}`,
      status: 'processing',
      uploadedAt: Date.now()
    });

    // 3. Trigger transcoding pipeline
    await queue.publish('video.uploaded', {
      videoId,
      s3Key,
      metadata
    });

    return video;
  }
}

class TranscodingService {
  constructor() {
    this.profiles = [
      { name: '240p', width: 426, height: 240, bitrate: '400k' },
      { name: '480p', width: 854, height: 480, bitrate: '1000k' },
      { name: '720p', width: 1280, height: 720, bitrate: '2500k' },
      { name: '1080p', width: 1920, height: 1080, bitrate: '5000k' },
      { name: '4K', width: 3840, height: 2160, bitrate: '15000k' }
    ];

    this.setupConsumer();
  }

  setupConsumer() {
    queue.subscribe('video.uploaded', async (event) => {
      await this.transcodeVideo(event.videoId, event.s3Key);
    });
  }

  async transcodeVideo(videoId, originalS3Key) {
    try {
      // Update status
      await db.videos.update(videoId, { status: 'transcoding' });

      // Transcode to multiple qualities in parallel
      const transcodingJobs = this.profiles.map(profile =>
        this.transcodeToProfile(videoId, originalS3Key, profile)
      );

      const results = await Promise.all(transcodingJobs);

      // Generate manifest (HLS/DASH)
      const manifestUrl = await this.generateManifest(videoId, results);

      // Update video record
      await db.videos.update(videoId, {
        status: 'ready',
        manifestUrl,
        transcodedAt: Date.now(),
        profiles: results
      });

      // Trigger content pre-positioning
      await this.prePositionContent(videoId);

      console.log(`Video ${videoId} transcoding completed`);
    } catch (error) {
      await db.videos.update(videoId, {
        status: 'failed',
        error: error.message
      });
      console.error(`Transcoding failed for ${videoId}:`, error);
    }
  }

  async transcodeToProfile(videoId, originalS3Key, profile) {
    const outputKey = `videos/${videoId}/${profile.name}/index.m3u8`;

    // Use AWS MediaConvert or FFmpeg
    await mediaConvert.createJob({
      Input: {
        FileInput: `s3://netflix-videos/${originalS3Key}`
      },
      Output: {
        Destination: `s3://netflix-videos/videos/${videoId}/${profile.name}/`,
        VideoDescription: {
          Width: profile.width,
          Height: profile.height,
          CodecSettings: {
            Codec: 'H_264',
            H264Settings: {
              Bitrate: profile.bitrate,
              RateControlMode: 'CBR'
            }
          }
        },
        ContainerSettings: {
          Container: 'M3U8',
          M3u8Settings: {
            SegmentDuration: 10 // 10-second segments
          }
        }
      }
    });

    return {
      profile: profile.name,
      url: `https://cdn.netflix.com/${outputKey}`,
      width: profile.width,
      height: profile.height,
      bitrate: profile.bitrate
    };
  }

  async generateManifest(videoId, profiles) {
    // Generate HLS master playlist
    const manifest = `#EXTM3U
#EXT-X-VERSION:3
${profiles.map(p => `
#EXT-X-STREAM-INF:BANDWIDTH=${parseInt(p.bitrate) * 1000},RESOLUTION=${p.width}x${p.height}
${p.url}
`).join('\n')}`;

    const manifestKey = `videos/${videoId}/master.m3u8`;
    await s3.putObject({
      Bucket: 'netflix-videos',
      Key: manifestKey,
      Body: manifest,
      ContentType: 'application/vnd.apple.mpegurl'
    });

    return `https://cdn.netflix.com/${manifestKey}`;
  }

  async prePositionContent(videoId) {
    // Predict which regions will have high demand
    const targetRegions = await this.predictDemand(videoId);

    // Push to edge servers proactively
    for (const region of targetRegions) {
      await queue.publish('content.preposition', {
        videoId,
        region
      });
    }
  }

  async predictDemand(videoId) {
    // Machine learning model predicts demand
    // Based on: genre, actors, user preferences, time of day
    const video = await db.videos.findById(videoId);

    // Simplified: push to all major regions
    return ['us-east', 'us-west', 'eu-west', 'asia-southeast'];
  }
}
```

### Video Streaming Service

```javascript
class StreamingService {
  async startStream(userId, videoId) {
    // 1. Check subscription & authorization
    const user = await db.users.findById(userId);
    if (!user.isSubscribed) {
      throw new Error('Subscription required');
    }

    // 2. Get video manifest
    const video = await db.videos.findById(videoId);
    if (video.status !== 'ready') {
      throw new Error('Video not available');
    }

    // 3. Find nearest CDN edge server
    const edgeServer = await this.findNearestEdge(user.region);

    // 4. Generate playback token (time-limited, signed URL)
    const token = this.generatePlaybackToken(userId, videoId);

    // 5. Record view
    await this.recordView(userId, videoId);

    // 6. Return streaming info
    return {
      manifestUrl: `${edgeServer}/${video.manifestUrl}?token=${token}`,
      edgeServer,
      token,
      expiresAt: Date.now() + 3600000 // 1 hour
    };
  }

  async findNearestEdge(userRegion) {
    // Find closest edge server with content cached
    // If not cached, return regional cache

    const edgeServers = await db.edgeServers.find({
      region: userRegion,
      status: 'healthy',
      loadPercentage: { $lt: 80 }
    })
    .sort({ loadPercentage: 1 })
    .limit(1);

    return edgeServers[0]?.url || `https://cdn-${userRegion}.netflix.com`;
  }

  generatePlaybackToken(userId, videoId) {
    // JWT with short expiry
    return jwt.sign(
      { userId, videoId, type: 'playback' },
      JWT_SECRET,
      { expiresIn: '1h' }
    );
  }

  async recordView(userId, videoId) {
    // Record viewing session
    const session = await db.viewingSessions.create({
      userId,
      videoId,
      startedAt: Date.now(),
      device: 'web',
      quality: 'auto'
    });

    // Update watch history (async)
    await queue.publish('view.started', {
      userId,
      videoId,
      sessionId: session.id
    });

    return session.id;
  }

  async updateProgress(sessionId, position, quality) {
    // Update viewing progress (for resume functionality)
    await db.viewingSessions.update(sessionId, {
      currentPosition: position,
      quality,
      lastUpdatedAt: Date.now()
    });

    // Track quality switches (for adaptive bitrate analytics)
    await this.trackQualitySwitch(sessionId, quality);
  }

  async endStream(sessionId) {
    const session = await db.viewingSessions.findById(sessionId);

    const duration = Date.now() - session.startedAt;

    await db.viewingSessions.update(sessionId, {
      endedAt: Date.now(),
      totalDuration: duration
    });

    // Update user watch history
    await this.updateWatchHistory(session.userId, session.videoId, duration);

    // Analytics
    await queue.publish('view.completed', {
      userId: session.userId,
      videoId: session.videoId,
      duration,
      quality: session.quality
    });
  }

  async updateWatchHistory(userId, videoId, duration) {
    await db.watchHistory.upsert({
      userId,
      videoId
    }, {
      lastWatchedAt: Date.now(),
      totalWatchTime: { $inc: duration },
      watchCount: { $inc: 1 }
    });
  }
}
```

### Adaptive Bitrate Streaming (Client-Side)

```javascript
// Client-side video player
class AdaptiveBitratePlayer {
  constructor(videoElement, manifestUrl) {
    this.video = videoElement;
    this.manifestUrl = manifestUrl;
    this.currentQuality = null;
    this.availableQualities = [];
  }

  async initialize() {
    // Fetch master manifest
    const manifest = await fetch(this.manifestUrl);
    const manifestText = await manifest.text();

    // Parse available qualities
    this.availableQualities = this.parseManifest(manifestText);

    // Start with quality based on bandwidth
    const initialQuality = await this.selectInitialQuality();
    await this.switchQuality(initialQuality);

    // Monitor bandwidth and adjust
    this.startBandwidthMonitoring();
  }

  async selectInitialQuality() {
    // Measure bandwidth
    const bandwidth = await this.measureBandwidth();

    // Select appropriate quality
    for (const quality of this.availableQualities.reverse()) {
      if (bandwidth >= quality.bitrate * 1.5) { // 1.5x buffer
        return quality;
      }
    }

    // Default to lowest quality
    return this.availableQualities[0];
  }

  async measureBandwidth() {
    const testUrl = 'https://cdn.netflix.com/speedtest/10mb.bin';
    const startTime = Date.now();

    const response = await fetch(testUrl);
    await response.arrayBuffer();

    const endTime = Date.now();
    const duration = (endTime - startTime) / 1000; // seconds
    const sizeBytes = 10 * 1024 * 1024; // 10 MB
    const bandwidth = (sizeBytes * 8) / duration; // bits per second

    return bandwidth;
  }

  async switchQuality(quality) {
    console.log(`Switching to ${quality.name}`);

    this.currentQuality = quality;

    // Load quality-specific manifest
    const response = await fetch(quality.url);
    const playlistText = await response.text();

    // Parse segment URLs
    const segments = this.parsePlaylist(playlistText);

    // Start downloading segments
    this.downloadSegments(segments);
  }

  startBandwidthMonitoring() {
    setInterval(() => {
      const bufferHealth = this.video.buffered.length > 0
        ? this.video.buffered.end(0) - this.video.currentTime
        : 0;

      // If buffer is low, switch to lower quality
      if (bufferHealth < 5) { // Less than 5 seconds buffered
        this.switchToLowerQuality();
      }
      // If buffer is healthy and bandwidth improved, switch to higher quality
      else if (bufferHealth > 15) { // More than 15 seconds buffered
        this.tryHigherQuality();
      }
    }, 5000); // Check every 5 seconds
  }

  switchToLowerQuality() {
    const currentIndex = this.availableQualities.indexOf(this.currentQuality);
    if (currentIndex > 0) {
      this.switchQuality(this.availableQualities[currentIndex - 1]);
    }
  }

  async tryHigherQuality() {
    const currentIndex = this.availableQualities.indexOf(this.currentQuality);
    if (currentIndex < this.availableQualities.length - 1) {
      const bandwidth = await this.measureBandwidth();
      const nextQuality = this.availableQualities[currentIndex + 1];

      if (bandwidth >= nextQuality.bitrate * 1.5) {
        this.switchQuality(nextQuality);
      }
    }
  }
}
```

### Recommendation System (Simplified)

```javascript
class RecommendationService {
  async getRecommendations(userId, limit = 20) {
    // Get user's watch history
    const watchHistory = await db.watchHistory.find({ userId })
      .sort({ lastWatchedAt: -1 })
      .limit(50);

    if (watchHistory.length === 0) {
      // New user - show trending content
      return await this.getTrendingContent(limit);
    }

    // Extract genres user likes
    const videoIds = watchHistory.map(wh => wh.videoId);
    const videos = await db.videos.find({ id: { $in: videoIds } });
    const genres = this.extractPopularGenres(videos);

    // Find similar content
    const recommendations = await db.videos.find({
      genres: { $in: genres },
      id: { $nin: videoIds }, // Exclude already watched
      status: 'ready'
    })
    .sort({ rating: -1, releaseDate: -1 })
    .limit(limit);

    // Personalize order using collaborative filtering
    return await this.personalizeOrder(userId, recommendations);
  }

  extractPopularGenres(videos) {
    const genreCount = {};

    videos.forEach(video => {
      video.genres.forEach(genre => {
        genreCount[genre] = (genreCount[genre] || 0) + 1;
      });
    });

    return Object.entries(genreCount)
      .sort((a, b) => b[1] - a[1])
      .slice(0, 3)
      .map(([genre]) => genre);
  }

  async personalizeOrder(userId, videos) {
    // Simplified collaborative filtering
    // "Users who watched what you watched also watched..."

    // In production, use machine learning models
    // trained on viewing patterns

    return videos;
  }

  async getTrendingContent(limit) {
    // Content with most views in last 7 days
    const sevenDaysAgo = Date.now() - 7 * 86400000;

    return await db.videos.find({
      status: 'ready'
    })
    .sort({ weeklyViews: -1 })
    .limit(limit);
  }
}
```

### Key Optimizations

**1. Content Pre-positioning:**
```
- Predict demand using ML models
- Push popular content to edge servers before demand
- During off-peak hours (midnight-6am)
- Results: 95%+ cache hit rate
```

**2. Adaptive Bitrate:**
```
- Multiple quality levels (240p - 4K)
- Client automatically adjusts based on bandwidth
- Prevents buffering
- Optimizes bandwidth usage
```

**3. Edge Computing:**
```
- 1000+ edge servers in ISP networks
- Content closest to users
- Reduces origin load by 95%
- Lowers CDN costs significantly
```

---

## 6.5 Key Takeaways

### Design Patterns Observed

**URL Shortener:**
- Read-heavy workload → Aggressive caching
- Sequential IDs + Base62 encoding
- Multi-level caching (CDN + Redis + DB)
- Async analytics processing

**Twitter:**
- Hybrid fanout (write + read)
- Celebrity problem solution
- Timeline pre-computation
- Eventual consistency acceptable

**Netflix:**
- CDN-first architecture
- Content pre-positioning
- Adaptive bitrate streaming
- ML-powered recommendations

### Common Principles

```
1. Cache Aggressively
   - Every case study uses caching
   - Multiple cache levels
   - Cache hit rates > 90%

2. Asynchronous Processing
   - Don't block user-facing requests
   - Use message queues
   - Background workers

3. Horizontal Scalability
   - Stateless services
   - Sharding/partitioning
   - Load balancing

4. Trade-offs
   - Consistency vs availability
   - Latency vs accuracy
   - Cost vs performance
```

---

## 6.6 Interview Tips

**When designing a system:**

```
1. Ask clarifying questions (5 min)
   - Scale? Users? Geography?
   - Read vs write ratio?
   - Consistency requirements?

2. Define requirements (5 min)
   - Functional (features)
   - Non-functional (scale, latency, availability)

3. Capacity estimation (5 min)
   - QPS, storage, bandwidth
   - Shows you think about scale

4. API design (5 min)
   - Key endpoints
   - Request/response formats

5. High-level design (15 min)
   - Draw components
   - Explain data flow
   - Discuss trade-offs

6. Deep dive (15 min)
   - Pick 2-3 critical components
   - Database schema
   - Caching strategy
   - Scaling approach

7. Wrap up (5 min)
   - Bottlenecks
   - Monitoring
   - Future improvements
```

---

**Next Chapter:** [Chapter 7: Microservices Architecture](./SystemDesign-07-Microservices.md)

In the next chapter, we'll cover:
- Service decomposition strategies
- Inter-service communication
- API Gateway and service mesh
- Distributed transactions and saga pattern
- Service discovery and configuration
- Observability and monitoring
