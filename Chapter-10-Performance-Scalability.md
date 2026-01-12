# Chapter 10: Performance & Scalability

## 10.1 Performance Optimization Techniques

```
┌────────────────────────────────────────┐
│    Performance Optimization Areas      │
│                                        │
│  Code Level    │  Application Level    │
│  ├─ Async ops  │  ├─ Caching          │
│  ├─ Algorithms │  ├─ Load balancing   │
│  └─ Memory     │  └─ Clustering       │
│                                        │
│  Database      │  Infrastructure       │
│  ├─ Indexing   │  ├─ CDN              │
│  ├─ Pooling    │  ├─ Scaling          │
│  └─ Queries    │  └─ Monitoring       │
└────────────────────────────────────────┘
```

## 10.2 Clustering

Node.js runs on a single thread. Clustering allows using all CPU cores.

### Cluster Module:
```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  console.log(`Master process ${process.pid} is running`);

  // Fork workers for each CPU
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    // Restart worker
    cluster.fork();
  });
} else {
  // Workers share the same port
  const server = http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Handled by worker ${process.pid}\n`);
  });

  server.listen(3000);
  console.log(`Worker ${process.pid} started`);
}
```

### Cluster Diagram:
```
         Master Process
               │
    ┌──────────┼──────────┐
    │          │          │
 Worker 1   Worker 2   Worker 3
    │          │          │
    └──────────┴──────────┘
         Port 3000
```

### PM2 (Production Process Manager):
```bash
npm install -g pm2

# Start with clustering
pm2 start app.js -i max  # max = number of CPUs

# Monitor
pm2 monit

# Logs
pm2 logs

# Restart
pm2 restart app

# Stop
pm2 stop app

# Startup script
pm2 startup
pm2 save
```

## 10.3 Caching

### 1. In-Memory Cache (Node-Cache):
```javascript
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 });  // 10 minutes

// Set cache
function getUser(id) {
  const cacheKey = `user_${id}`;
  const cached = cache.get(cacheKey);

  if (cached) {
    console.log('Cache hit');
    return cached;
  }

  console.log('Cache miss');
  const user = db.getUserById(id);
  cache.set(cacheKey, user);
  return user;
}

// Invalidate cache
function updateUser(id, data) {
  const user = db.updateUser(id, data);
  cache.del(`user_${id}`);
  return user;
}
```

### 2. Redis Caching:
```javascript
const redis = require('redis');
const client = redis.createClient();

await client.connect();

// Set with expiration
await client.setEx('key', 3600, JSON.stringify(data));

// Get
const cached = await client.get('key');
if (cached) {
  return JSON.parse(cached);
}

// Middleware example
const cacheMiddleware = async (req, res, next) => {
  const key = req.originalUrl;
  const cached = await client.get(key);

  if (cached) {
    return res.json(JSON.parse(cached));
  }

  res.originalJson = res.json;
  res.json = function(data) {
    client.setEx(key, 300, JSON.stringify(data));
    res.originalJson(data);
  };

  next();
};

app.get('/api/users', cacheMiddleware, async (req, res) => {
  const users = await User.find();
  res.json(users);
});
```

### 3. HTTP Caching Headers:
```javascript
app.get('/static-resource', (req, res) => {
  res.set({
    'Cache-Control': 'public, max-age=31536000',  // 1 year
    'ETag': '12345',
    'Last-Modified': new Date().toUTCString()
  });
  res.send('Static content');
});

// Conditional requests
app.get('/resource', (req, res) => {
  const etag = '12345';

  if (req.headers['if-none-match'] === etag) {
    return res.sendStatus(304);  // Not Modified
  }

  res.set('ETag', etag);
  res.json({ data: 'content' });
});
```

## 10.4 Database Optimization

### 1. Indexing:
```javascript
// MongoDB
userSchema.index({ email: 1 });  // Single field
userSchema.index({ name: 1, age: -1 });  // Compound index
userSchema.index({ location: '2dsphere' });  // Geospatial

// Check query performance
const explain = await User.find({ email: 'john@example.com' }).explain();
console.log(explain);
```

### 2. Query Optimization:
```javascript
// Bad: Loading entire documents
const users = await User.find();

// Good: Select only needed fields
const users = await User.find().select('name email');

// Bad: N+1 queries
for (const post of posts) {
  post.author = await User.findById(post.authorId);
}

// Good: Populate/join
const posts = await Post.find().populate('author');

// Pagination
const page = 1;
const limit = 20;
const users = await User.find()
  .skip((page - 1) * limit)
  .limit(limit);
```

### 3. Connection Pooling:
```javascript
// MongoDB
mongoose.connect('mongodb://localhost/myapp', {
  maxPoolSize: 10,
  minPoolSize: 5
});

// PostgreSQL
const pool = new Pool({
  max: 20,
  min: 5,
  idle: 10000
});
```

## 10.5 Load Balancing

```
         Load Balancer
               │
    ┌──────────┼──────────┐
    │          │          │
Server 1   Server 2   Server 3
    │          │          │
 Database  Database  Database
```

### Nginx Load Balancer:
```nginx
upstream backend {
  least_conn;  # Load balancing method
  server server1.example.com:3000;
  server server2.example.com:3000;
  server server3.example.com:3000;
}

server {
  listen 80;

  location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }
}
```

## 10.6 Horizontal vs Vertical Scaling

```
Vertical Scaling (Scale Up)
┌────────────┐      ┌────────────┐
│  2 CPU     │  →   │  8 CPU     │
│  4GB RAM   │      │  32GB RAM  │
└────────────┘      └────────────┘

Horizontal Scaling (Scale Out)
┌────────┐           ┌────────┐ ┌────────┐ ┌────────┐
│Server 1│    →      │Server 1│ │Server 2│ │Server 3│
└────────┘           └────────┘ └────────┘ └────────┘
```

## 10.7 Performance Monitoring

### 1. APM Tools:
```javascript
// New Relic
require('newrelic');

// DataDog
const tracer = require('dd-trace').init();

// Application Insights (Azure)
const appInsights = require('applicationinsights');
appInsights.setup('instrumentation-key').start();
```

### 2. Custom Metrics:
```javascript
const metrics = {
  requests: 0,
  errors: 0,
  responseTime: []
};

app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    metrics.requests++;
    if (res.statusCode >= 500) metrics.errors++;

    const duration = Date.now() - start;
    metrics.responseTime.push(duration);
  });

  next();
});

// Expose metrics endpoint
app.get('/metrics', (req, res) => {
  const avgResponseTime =
    metrics.responseTime.reduce((a, b) => a + b, 0) / metrics.responseTime.length;

  res.json({
    requests: metrics.requests,
    errors: metrics.errors,
    avgResponseTime: `${avgResponseTime.toFixed(2)}ms`,
    errorRate: `${((metrics.errors / metrics.requests) * 100).toFixed(2)}%`
  });
});
```

### 3. Health Checks:
```javascript
app.get('/health', async (req, res) => {
  const health = {
    uptime: process.uptime(),
    timestamp: Date.now(),
    status: 'OK'
  };

  try {
    // Check database
    await db.query('SELECT 1');
    health.database = 'OK';
  } catch (err) {
    health.database = 'ERROR';
    health.status = 'ERROR';
  }

  const statusCode = health.status === 'OK' ? 200 : 503;
  res.status(statusCode).json(health);
});
```

## 10.8 Code-Level Optimizations

### 1. Async Operations:
```javascript
// Bad: Sequential
async function getUsers() {
  const user1 = await fetchUser(1);
  const user2 = await fetchUser(2);
  const user3 = await fetchUser(3);
  return [user1, user2, user3];
}

// Good: Parallel
async function getUsers() {
  const [user1, user2, user3] = await Promise.all([
    fetchUser(1),
    fetchUser(2),
    fetchUser(3)
  ]);
  return [user1, user2, user3];
}
```

### 2. Avoid Blocking Operations:
```javascript
// Bad: Synchronous file read
const data = fs.readFileSync('large-file.txt');

// Good: Asynchronous
const data = await fs.promises.readFile('large-file.txt');

// Good: Stream for large files
const stream = fs.createReadStream('large-file.txt');
```

### 3. Memory Management:
```javascript
// Bad: Large arrays in memory
const allUsers = await User.find();  // 1 million records
processUsers(allUsers);

// Good: Streaming/cursor
const cursor = User.find().cursor();
for (let user = await cursor.next(); user != null; user = await cursor.next()) {
  await processUser(user);
}
```

## 10.9 CDN and Static Assets

```javascript
// Serve static files efficiently
app.use(express.static('public', {
  maxAge: '1y',  // Cache for 1 year
  etag: true,
  lastModified: true
}));

// Use CDN for assets
// Instead of: /static/image.jpg
// Use: https://cdn.example.com/static/image.jpg
```

## 10.10 Interview Questions

### Basic:
1. **What is clustering in Node.js?**
2. **What's the difference between vertical and horizontal scaling?**
3. **What is caching and why is it important?**

### Intermediate:
1. **How does the cluster module work?**
2. **Explain different caching strategies.**
3. **What are the benefits of database indexing?**

### Advanced:
1. **Design a scalable architecture for a high-traffic app.**
2. **How do you handle session management in a clustered environment?**
3. **Implement a rate limiter with Redis.**

## 10.11 Key Takeaways

- Use clustering to utilize all CPU cores
- Implement caching at multiple levels (memory, Redis, HTTP)
- Optimize database queries with indexes and proper query design
- Use connection pooling for databases
- Monitor performance with APM tools
- Choose horizontal scaling for better fault tolerance
- Avoid blocking the event loop
- Use PM2 for production process management
- Implement health checks and metrics endpoints
- Use CDN for static assets

**Next:** Chapter 11 - Microservices & APIs
