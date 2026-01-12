# Chapter 6: Express.js & Web Frameworks

## 6.1 What is Express.js?

Express is a minimal and flexible Node.js web application framework that provides a robust set of features for web and mobile applications.

```
┌────────────────────────────────────────┐
│      Raw Node.js HTTP Server          │
│  (Complex, verbose, low-level)         │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│          Express.js                    │
│  (Simple, elegant, feature-rich)       │
│  ├─ Routing                            │
│  ├─ Middleware                         │
│  ├─ Template engines                   │
│  └─ Error handling                     │
└────────────────────────────────────────┘
```

### Installation:

```bash
npm install express
```

### Basic Server:

```javascript
// app.js
const express = require('express');
const app = express();
const PORT = 3000;

app.get('/', (req, res) => {
  res.send('Hello World!');
});

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
```

## 6.2 Routing

Routing determines how an application responds to client requests.

### Route Structure:

```
app.METHOD(PATH, HANDLER)
     │      │      │
     │      │      └─ Function to execute
     │      └──────── URL path
     └─────────────── HTTP method
```

### HTTP Methods:

```javascript
const express = require('express');
const app = express();

// GET - Retrieve data
app.get('/users', (req, res) => {
  res.json({ users: [] });
});

// POST - Create data
app.post('/users', (req, res) => {
  res.status(201).json({ message: 'User created' });
});

// PUT - Update entire resource
app.put('/users/:id', (req, res) => {
  res.json({ message: 'User updated' });
});

// PATCH - Partial update
app.patch('/users/:id', (req, res) => {
  res.json({ message: 'User partially updated' });
});

// DELETE - Remove data
app.delete('/users/:id', (req, res) => {
  res.status(204).send();
});

// Handle ALL methods
app.all('/secret', (req, res) => {
  res.send('Accessed with any method');
});
```

### Route Parameters:

```javascript
// 1. Route parameters
app.get('/users/:id', (req, res) => {
  const userId = req.params.id;
  res.json({ userId });
});

// 2. Multiple parameters
app.get('/users/:userId/posts/:postId', (req, res) => {
  const { userId, postId } = req.params;
  res.json({ userId, postId });
});

// 3. Optional parameters with regex
app.get('/users/:id(\\d+)', (req, res) => {
  // Only matches numeric IDs
  res.json({ id: req.params.id });
});

// 4. Query parameters
app.get('/search', (req, res) => {
  const { q, page, limit } = req.query;
  // URL: /search?q=node&page=1&limit=10
  res.json({ query: q, page, limit });
});
```

### Route Organization:

```javascript
// routes/users.js
const express = require('express');
const router = express.Router();

router.get('/', (req, res) => {
  res.json({ users: [] });
});

router.get('/:id', (req, res) => {
  res.json({ user: { id: req.params.id } });
});

router.post('/', (req, res) => {
  res.status(201).json({ message: 'User created' });
});

module.exports = router;

// app.js
const userRoutes = require('./routes/users');
app.use('/api/users', userRoutes);
```

### Route Patterns:

```javascript
// Wildcard
app.get('/files/*', (req, res) => {
  res.send(`File path: ${req.params[0]}`);
});

// Regular expressions
app.get(/.*fly$/, (req, res) => {
  // Matches butterfly, dragonfly, etc.
  res.send('Pattern matched!');
});

// Route chaining
app.route('/book')
  .get((req, res) => res.send('Get book'))
  .post((req, res) => res.send('Add book'))
  .put((req, res) => res.send('Update book'));
```

## 6.3 Middleware

Middleware functions have access to request, response objects, and the next middleware function.

### Middleware Flow:

```
Request
   │
   ▼
┌─────────────────┐
│  Middleware 1   │  next()
└────────┬────────┘
         ▼
┌─────────────────┐
│  Middleware 2   │  next()
└────────┬────────┘
         ▼
┌─────────────────┐
│  Middleware 3   │  next()
└────────┬────────┘
         ▼
┌─────────────────┐
│  Route Handler  │
└────────┬────────┘
         ▼
      Response
```

### Types of Middleware:

```javascript
// 1. Application-level middleware
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// 2. Router-level middleware
const router = express.Router();
router.use((req, res, next) => {
  console.log('Router middleware');
  next();
});

// 3. Built-in middleware
app.use(express.json());                    // Parse JSON
app.use(express.urlencoded({ extended: true })); // Parse URL-encoded
app.use(express.static('public'));          // Serve static files

// 4. Third-party middleware
const morgan = require('morgan');
const cors = require('cors');

app.use(morgan('dev'));   // Logging
app.use(cors());          // CORS

// 5. Error-handling middleware (4 parameters!)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: err.message });
});
```

### Common Middleware Patterns:

```javascript
// Authentication middleware
const authMiddleware = (req, res, next) => {
  const token = req.headers.authorization;

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    // Verify token
    const decoded = verifyToken(token);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Usage
app.get('/protected', authMiddleware, (req, res) => {
  res.json({ user: req.user });
});

// Validation middleware
const validateUser = (req, res, next) => {
  const { email, password } = req.body;

  if (!email || !password) {
    return res.status(400).json({ error: 'Email and password required' });
  }

  if (password.length < 8) {
    return res.status(400).json({ error: 'Password too short' });
  }

  next();
};

app.post('/register', validateUser, (req, res) => {
  // Register user
});

// Logging middleware
const logger = (req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.url} - ${res.statusCode} - ${duration}ms`);
  });

  next();
};

// Rate limiting middleware
const rateLimit = {
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,  // Limit each IP to 100 requests per windowMs
  message: 'Too many requests'
};

// Async middleware wrapper
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Usage
app.get('/users', asyncHandler(async (req, res) => {
  const users = await User.find();
  res.json(users);
}));
```

## 6.4 Request and Response Objects

### Request Object (req):

```javascript
app.get('/demo', (req, res) => {
  // 1. Request parameters
  console.log(req.params);    // Route parameters
  console.log(req.query);     // Query string
  console.log(req.body);      // Request body (needs body parser)

  // 2. Headers
  console.log(req.headers);
  console.log(req.get('User-Agent'));

  // 3. URL information
  console.log(req.url);         // '/demo?page=1'
  console.log(req.originalUrl); // Full URL
  console.log(req.path);        // '/demo'
  console.log(req.protocol);    // 'http' or 'https'
  console.log(req.hostname);    // 'localhost'
  console.log(req.ip);          // Client IP

  // 4. Request method
  console.log(req.method);      // 'GET'

  // 5. Cookies (needs cookie-parser)
  console.log(req.cookies);

  // 6. Check content type
  console.log(req.is('json'));  // true if JSON
  console.log(req.is('html'));  // false

  // 7. Custom properties (set by middleware)
  console.log(req.user);        // Added by auth middleware
});
```

### Response Object (res):

```javascript
app.get('/demo', (req, res) => {
  // 1. Send response
  res.send('Hello');                    // Auto-detects type
  res.send({ message: 'Hello' });      // Sends JSON
  res.send(Buffer.from('data'));       // Sends buffer

  // 2. JSON response
  res.json({ status: 'success' });

  // 3. Status codes
  res.status(404).send('Not Found');
  res.sendStatus(200);  // Sends status message too

  // 4. Redirect
  res.redirect('/home');
  res.redirect(301, '/new-url');  // Permanent redirect

  // 5. Set headers
  res.set('Content-Type', 'text/html');
  res.set({
    'Content-Type': 'text/html',
    'X-Custom-Header': 'value'
  });

  // 6. Cookies
  res.cookie('token', 'abc123', {
    httpOnly: true,
    maxAge: 3600000
  });
  res.clearCookie('token');

  // 7. Download file
  res.download('/path/to/file.pdf');

  // 8. Send file
  res.sendFile('/path/to/file.html');

  // 9. Render template
  res.render('index', { title: 'Home' });

  // 10. End response
  res.end();

  // 11. Chain methods
  res.status(200).json({ message: 'Success' });
});
```

## 6.5 Error Handling

### Error Handling Flow:

```
Request
   │
   ▼
Middleware/Route
   │
   ├──▶ Success ──▶ Response
   │
   └──▶ Error ──▶ next(error) ──▶ Error Handler ──▶ Error Response
```

### Error Handling Patterns:

```javascript
// 1. Synchronous errors (automatically caught)
app.get('/error', (req, res) => {
  throw new Error('Something went wrong!');
});

// 2. Asynchronous errors (must call next)
app.get('/async-error', (req, res, next) => {
  setTimeout(() => {
    next(new Error('Async error'));
  }, 100);
});

// 3. Promise rejection (needs handling)
app.get('/promise-error', (req, res, next) => {
  someAsyncOperation()
    .then(result => res.json(result))
    .catch(next);  // Pass error to error handler
});

// 4. Async/await errors (use try-catch)
app.get('/await-error', async (req, res, next) => {
  try {
    const result = await someAsyncOperation();
    res.json(result);
  } catch (err) {
    next(err);
  }
});

// 5. Custom error handler (MUST have 4 parameters)
app.use((err, req, res, next) => {
  console.error(err.stack);

  res.status(err.statusCode || 500).json({
    error: {
      message: err.message,
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
    }
  });
});

// 6. 404 handler (place after all routes)
app.use((req, res) => {
  res.status(404).json({ error: 'Route not found' });
});
```

### Custom Error Class:

```javascript
// errors/AppError.js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

module.exports = AppError;

// Usage
const AppError = require('./errors/AppError');

app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await User.findById(req.params.id);

    if (!user) {
      throw new AppError('User not found', 404);
    }

    res.json(user);
  } catch (err) {
    next(err);
  }
});

// Error handler
app.use((err, req, res, next) => {
  if (err.isOperational) {
    res.status(err.statusCode).json({ error: err.message });
  } else {
    console.error('UNEXPECTED ERROR:', err);
    res.status(500).json({ error: 'Something went wrong' });
  }
});
```

## 6.6 Building a RESTful API

### REST Principles:

```
Resource-based URLs:
GET    /api/users        - Get all users
GET    /api/users/:id    - Get specific user
POST   /api/users        - Create user
PUT    /api/users/:id    - Update user (full)
PATCH  /api/users/:id    - Update user (partial)
DELETE /api/users/:id    - Delete user
```

### Complete API Example:

```javascript
// models/User.js
const users = [
  { id: 1, name: 'John', email: 'john@example.com' },
  { id: 2, name: 'Jane', email: 'jane@example.com' }
];

// controllers/userController.js
exports.getAllUsers = (req, res) => {
  res.json({ success: true, data: users });
};

exports.getUser = (req, res, next) => {
  const user = users.find(u => u.id === parseInt(req.params.id));

  if (!user) {
    return next(new AppError('User not found', 404));
  }

  res.json({ success: true, data: user });
};

exports.createUser = (req, res) => {
  const newUser = {
    id: users.length + 1,
    name: req.body.name,
    email: req.body.email
  };

  users.push(newUser);
  res.status(201).json({ success: true, data: newUser });
};

exports.updateUser = (req, res, next) => {
  const user = users.find(u => u.id === parseInt(req.params.id));

  if (!user) {
    return next(new AppError('User not found', 404));
  }

  user.name = req.body.name || user.name;
  user.email = req.body.email || user.email;

  res.json({ success: true, data: user });
};

exports.deleteUser = (req, res, next) => {
  const index = users.findIndex(u => u.id === parseInt(req.params.id));

  if (index === -1) {
    return next(new AppError('User not found', 404));
  }

  users.splice(index, 1);
  res.status(204).send();
};

// routes/users.js
const express = require('express');
const router = express.Router();
const userController = require('../controllers/userController');

router
  .route('/')
  .get(userController.getAllUsers)
  .post(userController.createUser);

router
  .route('/:id')
  .get(userController.getUser)
  .patch(userController.updateUser)
  .delete(userController.deleteUser);

module.exports = router;

// app.js
const express = require('express');
const userRoutes = require('./routes/users');

const app = express();

app.use(express.json());
app.use('/api/users', userRoutes);

// Error handler
app.use((err, req, res, next) => {
  res.status(err.statusCode || 500).json({
    success: false,
    error: err.message
  });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

## 6.7 Practical Tasks

### Task 1: CRUD API with Validation
```javascript
// task1.js - User CRUD with validation
const express = require('express');
const { body, validationResult } = require('express-validator');

const app = express();
app.use(express.json());

let users = [];
let nextId = 1;

// Validation middleware
const validateUser = [
  body('name').trim().notEmpty().withMessage('Name is required'),
  body('email').isEmail().withMessage('Invalid email'),
  body('age').optional().isInt({ min: 18 }).withMessage('Must be 18+')
];

// Create user
app.post('/users', validateUser, (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }

  const user = {
    id: nextId++,
    ...req.body
  };
  users.push(user);
  res.status(201).json(user);
});

// Get all users
app.get('/users', (req, res) => {
  res.json(users);
});

// Get user by ID
app.get('/users/:id', (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});

// Update user
app.patch('/users/:id', validateUser, (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ errors: errors.array() });
  }

  const user = users.find(u => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).json({ error: 'User not found' });

  Object.assign(user, req.body);
  res.json(user);
});

// Delete user
app.delete('/users/:id', (req, res) => {
  const index = users.findIndex(u => u.id === parseInt(req.params.id));
  if (index === -1) return res.status(404).json({ error: 'User not found' });

  users.splice(index, 1);
  res.status(204).send();
});

app.listen(3000);
```

### Task 2: Logging Middleware
```javascript
// task2.js - Custom logging middleware
const express = require('express');
const fs = require('fs').promises;

const app = express();

// Logger middleware
const logger = (req, res, next) => {
  const start = Date.now();

  res.on('finish', async () => {
    const duration = Date.now() - start;
    const log = `[${new Date().toISOString()}] ${req.method} ${req.url} - ${res.statusCode} - ${duration}ms\n`;

    console.log(log.trim());
    await fs.appendFile('access.log', log);
  });

  next();
};

app.use(logger);

app.get('/', (req, res) => {
  res.send('Hello World');
});

app.listen(3000);
```

## 6.8 Interview Questions

### Basic Level:
1. **What is Express.js?**
2. **What is middleware in Express?**
3. **How do you handle errors in Express?**
4. **What's the difference between app.use() and app.get()?**

### Intermediate Level:
1. **Explain the request-response cycle in Express.**
2. **What's the difference between app-level and router-level middleware?**
3. **How do you handle async errors in Express?**
4. **What are route parameters vs query parameters?**

### Advanced Level:
1. **How would you implement rate limiting?**
2. **Explain middleware execution order and next().**
3. **How do you structure a large Express application?**
4. **Implement custom error handling with different error types.**

## 6.9 Key Takeaways

- Express simplifies Node.js web development
- Middleware functions have access to req, res, and next
- Error handlers must have 4 parameters
- Use router for organizing routes
- Always handle async errors with try-catch or .catch()
- RESTful APIs follow resource-based URL patterns
- Built-in middleware: express.json(), express.static()

## Next Chapter Preview

In Chapter 7, we'll explore Database Integration with MongoDB, PostgreSQL, and ORMs like Mongoose and Sequelize.

---

**Pro Tip**: Study middleware execution order thoroughly - it's a common interview topic!
