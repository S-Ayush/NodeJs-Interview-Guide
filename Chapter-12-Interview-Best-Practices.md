# Chapter 12: Interview Questions & Best Practices

## 12.1 Most Asked Interview Questions by Topic

### Architecture & Fundamentals (20%)

**Q1: Explain how Node.js works internally.**
```
Answer Structure:
1. V8 Engine (JavaScript execution)
2. libuv (Event loop, async I/O)
3. Single-threaded event loop
4. Thread pool for heavy operations
5. Non-blocking I/O model

Key Points:
- JavaScript → V8 → Machine code
- Event loop processes callbacks
- Thread pool (4 threads) for fs, crypto, DNS
- OS handles network I/O asynchronously
```

**Q2: What is the event loop? Explain its phases.**
```
Phases in Order:
1. Timers - setTimeout, setInterval
2. Pending callbacks - I/O callbacks
3. Idle, prepare - Internal
4. Poll - Retrieve I/O events
5. Check - setImmediate
6. Close callbacks - socket.on('close')

Critical Understanding:
- Microtasks (nextTick, Promises) run between phases
- Poll phase can block waiting for I/O
- setImmediate after poll, setTimeout in timers
```

**Q3: process.nextTick() vs setImmediate() - Which executes first?**
```javascript
// process.nextTick() ALWAYS executes first
process.nextTick(() => console.log('nextTick'));
setImmediate(() => console.log('immediate'));
// Output: nextTick, immediate

// EXCEPT inside I/O cycle
fs.readFile('file.txt', () => {
  setImmediate(() => console.log('immediate'));  // First
  setTimeout(() => console.log('timeout'), 0);   // Second
});
```

### Asynchronous Programming (25%)

**Q4: Explain callback hell and how to avoid it.**
```javascript
// Problem
getData(function(a) {
  getMoreData(a, function(b) {
    getMoreData(b, function(c) {
      // Pyramid of doom
    });
  });
});

// Solutions:
// 1. Named functions
// 2. Promises
// 3. Async/Await (Best)
async function fetchData() {
  const a = await getData();
  const b = await getMoreData(a);
  const c = await getMoreData(b);
  return c;
}
```

**Q5: Promise.all() vs Promise.allSettled() vs Promise.race()**
```javascript
const p1 = Promise.resolve(1);
const p2 = Promise.reject('error');
const p3 = Promise.resolve(3);

// Promise.all - Fails if any rejects
Promise.all([p1, p2, p3]);  // Rejects with 'error'

// Promise.allSettled - Never fails, returns all results
Promise.allSettled([p1, p2, p3]);
// [{status: 'fulfilled', value: 1},
//  {status: 'rejected', reason: 'error'},
//  {status: 'fulfilled', value: 3}]

// Promise.race - Returns first settled
Promise.race([p1, p2, p3]);  // Resolves with 1
```

**Q6: How do you handle errors in async/await?**
```javascript
// Method 1: try-catch
async function fetchUser() {
  try {
    const user = await User.findById(id);
    return user;
  } catch (err) {
    console.error('Error:', err);
    throw err;  // Re-throw or handle
  }
}

// Method 2: .catch()
async function fetchUser() {
  const user = await User.findById(id).catch(err => {
    console.error(err);
    return null;  // Fallback
  });
  return user;
}

// Method 3: Wrapper function
const asyncHandler = fn => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};
```

### Express.js & Web Development (20%)

**Q7: Explain middleware in Express.**
```javascript
// Middleware signature: (req, res, next)
app.use((req, res, next) => {
  console.log('Request:', req.method, req.url);
  next();  // MUST call next() or send response
});

// Order matters
app.use(middleware1);  // Executes first
app.use(middleware2);  // Executes second
app.get('/route', handler);
app.use(errorHandler);  // Executes last (4 params)

// Error handler MUST have 4 parameters
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```

**Q8: How do you handle CORS in Node.js?**
```javascript
// Method 1: Using cors package
const cors = require('cors');
app.use(cors({
  origin: 'https://yourdomain.com',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE']
}));

// Method 2: Manual headers
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', '*');
  res.header('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE');
  res.header('Access-Control-Allow-Headers', 'Content-Type,Authorization');
  if (req.method === 'OPTIONS') {
    return res.sendStatus(200);
  }
  next();
});
```

### Database (15%)

**Q9: What's the difference between SQL and NoSQL? When to use each?**
```
SQL (PostgreSQL, MySQL):
Use When:
- Complex relationships
- ACID compliance needed
- Structured data
- Complex queries/joins
Example: Banking, ERP systems

NoSQL (MongoDB, Redis):
Use When:
- Flexible schema
- Horizontal scaling needed
- Simple queries
- High write throughput
Example: Social media, real-time analytics
```

**Q10: Explain connection pooling.**
```javascript
// Without pooling (BAD)
async function query() {
  const connection = await createConnection();  // Slow
  const result = await connection.query('SELECT * FROM users');
  await connection.close();
  return result;
}

// With pooling (GOOD)
const pool = new Pool({ max: 10, min: 2 });

async function query() {
  const connection = await pool.connect();  // Fast (reuses)
  const result = await connection.query('SELECT * FROM users');
  connection.release();  // Returns to pool
  return result;
}

Benefits:
- Faster (no connection overhead)
- Limits concurrent connections
- Automatic reconnection
```

### Performance & Scalability (10%)

**Q11: How does clustering work in Node.js?**
```javascript
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  const numCPUs = os.cpus().length;

  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  // Restart on crash
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.id} died`);
    cluster.fork();
  });
} else {
  // Workers share the same port
  app.listen(3000);
}

Key Points:
- Master process manages workers
- Workers share port (load balanced by OS)
- Each worker has separate V8 instance
- No shared memory between workers
```

**Q12: How do you implement caching?**
```javascript
// 1. In-Memory (fast, limited)
const cache = new Map();
cache.set('key', value);
const cached = cache.get('key');

// 2. Redis (distributed, scalable)
const redis = require('redis');
const client = redis.createClient();
await client.setEx('key', 3600, JSON.stringify(value));
const cached = JSON.parse(await client.get('key'));

// 3. HTTP Caching
res.set('Cache-Control', 'public, max-age=3600');
res.set('ETag', '12345');

Strategies:
- Cache-Aside: App checks cache first
- Write-Through: App writes to cache + DB
- Write-Behind: App writes to cache, syncs later
```

### Security (10%)

**Q13: How do you secure a Node.js application?**
```javascript
// 1. Authentication
const bcrypt = require('bcrypt');
const hashedPassword = await bcrypt.hash(password, 10);

const jwt = require('jsonwebtoken');
const token = jwt.sign({ userId }, SECRET, { expiresIn: '24h' });

// 2. Input Validation
const { body } = require('express-validator');
app.post('/user',
  body('email').isEmail(),
  body('password').isLength({ min: 8 }),
  handler
);

// 3. Security Headers
const helmet = require('helmet');
app.use(helmet());

// 4. Rate Limiting
const rateLimit = require('express-rate-limit');
app.use(rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }));

// 5. CSRF Protection
const csrf = require('csurf');
app.use(csrf({ cookie: true }));

// 6. Parameterized Queries (prevent SQL injection)
await pool.query('SELECT * FROM users WHERE id = $1', [userId]);
```

## 12.2 Coding Challenges

### Challenge 1: Implement Promise.all()
```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('Argument must be an array'));
    }

    const results = [];
    let completed = 0;

    if (promises.length === 0) {
      return resolve(results);
    }

    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(value => {
          results[index] = value;
          completed++;

          if (completed === promises.length) {
            resolve(results);
          }
        })
        .catch(reject);
    });
  });
}
```

### Challenge 2: Implement Retry Logic
```javascript
async function retry(fn, maxAttempts = 3, delay = 1000, backoff = 2) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxAttempts) {
        throw err;
      }

      const waitTime = delay * Math.pow(backoff, attempt - 1);
      console.log(`Attempt ${attempt} failed, retrying in ${waitTime}ms...`);
      await new Promise(resolve => setTimeout(resolve, waitTime));
    }
  }
}

// Usage
await retry(() => fetchUser(id), 5, 1000, 2);
```

### Challenge 3: Rate Limiter
```javascript
class RateLimiter {
  constructor(maxRequests, windowMs) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
    this.requests = new Map();
  }

  isAllowed(key) {
    const now = Date.now();
    const userRequests = this.requests.get(key) || [];

    // Remove old requests outside window
    const validRequests = userRequests.filter(
      time => now - time < this.windowMs
    );

    if (validRequests.length < this.maxRequests) {
      validRequests.push(now);
      this.requests.set(key, validRequests);
      return true;
    }

    return false;
  }
}

// Usage
const limiter = new RateLimiter(100, 60000);  // 100 req/min

app.use((req, res, next) => {
  const ip = req.ip;
  if (!limiter.isAllowed(ip)) {
    return res.status(429).json({ error: 'Too many requests' });
  }
  next();
});
```

### Challenge 4: Debounce Function
```javascript
function debounce(func, delay) {
  let timeoutId;

  return function(...args) {
    clearTimeout(timeoutId);

    timeoutId = setTimeout(() => {
      func.apply(this, args);
    }, delay);
  };
}

// Usage
const debouncedSearch = debounce((query) => {
  console.log('Searching for:', query);
}, 300);

debouncedSearch('hello');  // Waits 300ms
debouncedSearch('hello');  // Cancels previous, waits 300ms
```

## 12.3 System Design Questions

### Q: Design a URL Shortener

```
Requirements:
- Shorten long URLs
- Redirect to original URL
- Track analytics (clicks)
- High availability

Architecture:
┌─────────┐
│ Client  │
└────┬────┘
     │
┌────▼─────────┐
│ Load Balancer│
└────┬─────────┘
     │
 ┌───┼───┐
 │   │   │
▼   ▼   ▼
┌──────────┐
│ App      │  ┌────────────┐
│ Servers  │─▶│ Redis Cache│
└────┬─────┘  └────────────┘
     │
┌────▼──────┐
│ MongoDB   │
│ {         │
│  short_url│
│  long_url │
│  clicks   │
│ }         │
└───────────┘

Implementation:
```javascript
// Generate short code
function generateShortCode(length = 6) {
  const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let code = '';
  for (let i = 0; i < length; i++) {
    code += chars[Math.floor(Math.random() * chars.length)];
  }
  return code;
}

// Shorten URL
app.post('/shorten', async (req, res) => {
  const { longUrl } = req.body;

  // Check cache
  let shortCode = await redis.get(`url:${longUrl}`);

  if (!shortCode) {
    // Generate unique code
    shortCode = generateShortCode();

    // Save to database
    await URL.create({ shortCode, longUrl, clicks: 0 });

    // Cache it
    await redis.setEx(`url:${longUrl}`, 3600, shortCode);
    await redis.setEx(`code:${shortCode}`, 3600, longUrl);
  }

  res.json({ shortUrl: `https://short.url/${shortCode}` });
});

// Redirect
app.get('/:code', async (req, res) => {
  const { code } = req.params;

  // Check cache
  let longUrl = await redis.get(`code:${code}`);

  if (!longUrl) {
    const url = await URL.findOne({ shortCode: code });
    if (!url) return res.status(404).send('URL not found');

    longUrl = url.longUrl;
    await redis.setEx(`code:${code}`, 3600, longUrl);
  }

  // Increment clicks (async)
  URL.updateOne({ shortCode: code }, { $inc: { clicks: 1 } });

  res.redirect(longUrl);
});
```

## 12.4 Best Practices Checklist

### Code Organization
- [ ] Use MVC or layered architecture
- [ ] Separate routes, controllers, services, models
- [ ] Use environment variables for config
- [ ] Follow consistent naming conventions

### Error Handling
- [ ] Use try-catch for async operations
- [ ] Create custom error classes
- [ ] Implement global error handler
- [ ] Log errors properly

### Security
- [ ] Hash passwords with bcrypt (10+ rounds)
- [ ] Use JWT for authentication
- [ ] Validate all user input
- [ ] Use parameterized queries
- [ ] Implement rate limiting
- [ ] Use HTTPS in production
- [ ] Set security headers (helmet)

### Performance
- [ ] Use connection pooling
- [ ] Implement caching (Redis)
- [ ] Use clustering for production
- [ ] Optimize database queries
- [ ] Use streams for large files
- [ ] Enable gzip compression

### Testing
- [ ] Write unit tests (Jest)
- [ ] Write integration tests
- [ ] Aim for 80%+ code coverage
- [ ] Test error scenarios
- [ ] Use mocking for external dependencies

### Deployment
- [ ] Use PM2 for process management
- [ ] Set up CI/CD pipeline
- [ ] Use Docker for containerization
- [ ] Implement health checks
- [ ] Set up monitoring (logs, metrics)
- [ ] Use load balancer

## 12.5 Interview Tips

### Before the Interview:
1. Review Node.js fundamentals (event loop, async)
2. Practice coding on a whiteboard/shared editor
3. Prepare questions about the role/company
4. Review recent projects you've worked on

### During the Interview:
1. **Think aloud** - Explain your thought process
2. **Ask clarifying questions** before coding
3. **Start with brute force**, then optimize
4. **Test your code** with examples
5. **Discuss trade-offs** of your solution

### Common Mistakes to Avoid:
- Not handling errors
- Blocking the event loop
- Not using async/await properly
- Poor variable naming
- Overcomplicating solutions

### Key Topics to Master:
1. Event loop (phases, microtasks, macrotasks)
2. Promises, async/await
3. Streams and buffers
4. Express middleware
5. Database optimization
6. Security best practices
7. Performance optimization
8. System design patterns

## 12.6 Resources for Further Learning

### Official Documentation:
- Node.js Docs: https://nodejs.org/docs
- Express Docs: https://expressjs.com
- MongoDB Docs: https://docs.mongodb.com

### Online Courses:
- NodeSchool.io (interactive tutorials)
- Udemy Node.js courses
- freeCodeCamp Node.js certification

### Books:
- "Node.js Design Patterns" by Mario Casciaro
- "You Don't Know JS" series
- "Designing Data-Intensive Applications"

### Practice Platforms:
- LeetCode
- HackerRank
- CodeWars
- Exercism

## 12.7 Final Takeaways

### Must-Know Concepts:
1. **Event Loop** - Understand all phases
2. **Async Patterns** - Callbacks, Promises, async/await
3. **Streams** - Memory-efficient data processing
4. **Express Middleware** - Request/response pipeline
5. **Database Optimization** - Indexing, pooling, queries
6. **Security** - Authentication, validation, OWASP Top 10
7. **Performance** - Clustering, caching, profiling
8. **Testing** - Unit, integration, E2E

### Interview Success Formula:
```
Knowledge + Practice + Communication = Success

Knowledge: Study these 12 chapters thoroughly
Practice: Solve coding challenges daily
Communication: Explain your thinking clearly
```

### Remember:
- It's okay to not know everything
- Focus on fundamentals first
- Practice explaining concepts simply
- Real-world experience matters most
- Keep learning and stay curious

---

## Congratulations!

You've completed the Node.js Interview Preparation Guide. You now have comprehensive knowledge covering:

- Node.js Architecture & Event Loop
- Asynchronous Programming Patterns
- Core Modules & File System
- Streams & Buffers
- Express.js & Web Development
- Database Integration
- Authentication & Security
- Testing & Debugging
- Performance & Scalability
- Microservices & APIs
- Interview Questions & Best Practices

**Next Steps:**
1. Practice coding challenges daily
2. Build a complete project using concepts learned
3. Review weak areas identified during practice
4. Mock interviews with peers
5. Stay updated with Node.js releases

**Good luck with your interview!**

---

*"The only way to learn a new programming language is by writing programs in it." - Dennis Ritchie*
