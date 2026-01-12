# Chapter 9: Testing & Debugging

## 9.1 Testing Pyramid

```
        ┌────────┐
        │   E2E  │  Few, slow, expensive
        ├────────┤
        │  Integration │  Some, medium speed
        ├────────┤
        │   Unit Tests   │  Many, fast, cheap
        └────────┘
```

## 9.2 Unit Testing with Jest

### Installation:
```bash
npm install --save-dev jest supertest
```

### Basic Test:
```javascript
// math.js
function add(a, b) {
  return a + b;
}

function multiply(a, b) {
  return a * b;
}

module.exports = { add, multiply };

// math.test.js
const { add, multiply } = require('./math');

describe('Math functions', () => {
  describe('add', () => {
    test('adds two positive numbers', () => {
      expect(add(2, 3)).toBe(5);
    });

    test('adds negative numbers', () => {
      expect(add(-2, -3)).toBe(-5);
    });

    test('handles zero', () => {
      expect(add(5, 0)).toBe(5);
    });
  });

  describe('multiply', () => {
    test('multiplies two numbers', () => {
      expect(multiply(2, 3)).toBe(6);
    });

    test('returns zero when multiplying by zero', () => {
      expect(multiply(5, 0)).toBe(0);
    });
  });
});
```

### Async Testing:
```javascript
// api.js
async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

// api.test.js
test('fetches user successfully', async () => {
  const user = await fetchUser(1);
  expect(user).toHaveProperty('id', 1);
  expect(user).toHaveProperty('name');
});

// With Promise
test('fetches user with promise', () => {
  return fetchUser(1).then(user => {
    expect(user.id).toBe(1);
  });
});

// With async/await and error
test('handles error when user not found', async () => {
  await expect(fetchUser(999)).rejects.toThrow('User not found');
});
```

### Mocking:
```javascript
// userService.js
const db = require('./database');

async function getUserById(id) {
  return await db.query('SELECT * FROM users WHERE id = $1', [id]);
}

// userService.test.js
jest.mock('./database');
const db = require('./database');
const { getUserById } = require('./userService');

describe('UserService', () => {
  test('gets user by id', async () => {
    const mockUser = { id: 1, name: 'John' };
    db.query.mockResolvedValue([mockUser]);

    const user = await getUserById(1);

    expect(db.query).toHaveBeenCalledWith(
      'SELECT * FROM users WHERE id = $1',
      [1]
    );
    expect(user).toEqual([mockUser]);
  });
});

// Mock functions
const mockFn = jest.fn();
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue('async result');
mockFn.mockRejectedValue(new Error('error'));

// Spy on functions
const spy = jest.spyOn(obj, 'method');
expect(spy).toHaveBeenCalled();
expect(spy).toHaveBeenCalledWith('arg1', 'arg2');
```

## 9.3 API Testing with Supertest

```javascript
// app.js
const express = require('express');
const app = express();

app.use(express.json());

let users = [{ id: 1, name: 'John' }];

app.get('/users', (req, res) => {
  res.json(users);
});

app.post('/users', (req, res) => {
  const user = { id: users.length + 1, ...req.body };
  users.push(user);
  res.status(201).json(user);
});

module.exports = app;

// app.test.js
const request = require('supertest');
const app = require('./app');

describe('User API', () => {
  test('GET /users returns all users', async () => {
    const response = await request(app)
      .get('/users')
      .expect(200)
      .expect('Content-Type', /json/);

    expect(response.body).toBeInstanceOf(Array);
  });

  test('POST /users creates a new user', async () => {
    const newUser = { name: 'Jane' };

    const response = await request(app)
      .post('/users')
      .send(newUser)
      .expect(201);

    expect(response.body).toHaveProperty('id');
    expect(response.body.name).toBe('Jane');
  });

  test('POST /users without name returns 400', async () => {
    await request(app)
      .post('/users')
      .send({})
      .expect(400);
  });
});
```

## 9.4 Test Coverage

```bash
# Run tests with coverage
npm test -- --coverage

# Coverage report shows:
# - Statement coverage
# - Branch coverage
# - Function coverage
# - Line coverage
```

```javascript
// jest.config.js
module.exports = {
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

## 9.5 Debugging Techniques

### 1. Console Logging
```javascript
console.log('Value:', value);
console.error('Error:', error);
console.table([{ name: 'John', age: 30 }]);
console.time('operation');
// ... code
console.timeEnd('operation');

console.trace('Trace stack');  // Shows call stack
```

### 2. Node.js Debugger
```javascript
// Add debugger statement
function process Data(data) {
  debugger;  // Execution pauses here
  return data * 2;
}

// Run with:
// node inspect app.js
// node --inspect app.js (Chrome DevTools)
```

### 3. VS Code Debugging
```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug App",
      "program": "${workspaceFolder}/app.js",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Jest Tests",
      "program": "${workspaceFolder}/node_modules/.bin/jest",
      "args": ["--runInBand"],
      "console": "integratedTerminal"
    }
  ]
}
```

### 4. Debug Module
```javascript
const debug = require('debug')('app:server');

debug('Server starting...');
debug('Connected to database');

// Run with: DEBUG=app:* node app.js
```

### 5. Error Handling & Stack Traces
```javascript
// Custom error with stack trace
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    Error.captureStackTrace(this, this.constructor);
  }
}

try {
  throw new AppError('Something went wrong', 500);
} catch (err) {
  console.error(err.stack);  // Full stack trace
}
```

## 9.6 Performance Profiling

### Node.js Built-in Profiler
```bash
# CPU profiling
node --prof app.js

# Process the log
node --prof-process isolate-0xnnnnnnnnnnnn-v8.log > processed.txt
```

### Chrome DevTools
```bash
# Start with inspector
node --inspect app.js

# Open chrome://inspect
# Click "inspect" under Remote Target
```

### Clinic.js
```bash
npm install -g clinic

# Doctor (overall health)
clinic doctor -- node app.js

# Flame (CPU profiling)
clinic flame -- node app.js

# Bubbleprof (async operations)
clinic bubbleprof -- node app.js
```

## 9.7 Memory Leak Detection

```javascript
// Monitor memory usage
setInterval(() => {
  const used = process.memoryUsage();
  console.log({
    rss: `${Math.round(used.rss / 1024 / 1024)}MB`,
    heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)}MB`,
    heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)}MB`,
    external: `${Math.round(used.external / 1024 / 1024)}MB`
  });
}, 5000);

// Common memory leak causes:
// 1. Global variables
// 2. Forgotten timers/callbacks
// 3. Event listeners not removed
// 4. Closures holding references
// 5. Large object caches

// Example leak
const leaks = [];
setInterval(() => {
  leaks.push(new Array(1000000).fill('leak'));  // Memory leak!
}, 100);

// Fix: Clear or limit
const cache = new Map();
const MAX_CACHE_SIZE = 1000;

function addToCache(key, value) {
  if (cache.size >= MAX_CACHE_SIZE) {
    const firstKey = cache.keys().next().value;
    cache.delete(firstKey);
  }
  cache.set(key, value);
}
```

## 9.8 Testing Best Practices

### 1. AAA Pattern
```javascript
test('adds user to database', async () => {
  // Arrange
  const newUser = { name: 'John', email: 'john@example.com' };

  // Act
  const result = await createUser(newUser);

  // Assert
  expect(result).toHaveProperty('id');
  expect(result.name).toBe('John');
});
```

### 2. Test Isolation
```javascript
describe('User tests', () => {
  beforeEach(async () => {
    // Setup: Clean database
    await User.deleteMany({});
  });

  afterEach(async () => {
    // Cleanup
    await User.deleteMany({});
  });

  test('creates user', async () => {
    // Test is isolated
  });
});
```

### 3. Test Naming
```javascript
// Good
test('returns 404 when user not found', () => {});
test('throws error when email is invalid', () => {});

// Bad
test('test1', () => {});
test('should work', () => {});
```

### 4. Don't Test Implementation
```javascript
// Bad: Testing implementation
test('calls database.query', () => {
  // Tests how it works
});

// Good: Testing behavior
test('returns user data', async () => {
  // Tests what it does
  const user = await getUser(1);
  expect(user).toHaveProperty('name');
});
```

## 9.9 E2E Testing with Playwright/Puppeteer

```javascript
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.goto('http://localhost:3000');
  await page.fill('#username', 'testuser');
  await page.fill('#password', 'password');
  await page.click('button[type="submit"]');

  await page.waitForSelector('.dashboard');
  const title = await page.textContent('h1');
  console.log('Title:', title);

  await browser.close();
})();
```

## 9.10 Interview Questions

### Basic:
1. **What are the different types of testing?**
2. **What is mocking and why is it useful?**
3. **How do you debug Node.js applications?**

### Intermediate:
1. **Explain the testing pyramid.**
2. **How do you test async code?**
3. **What's test coverage and what percentage is ideal?**

### Advanced:
1. **How do you detect and fix memory leaks?**
2. **Design a testing strategy for a microservices app.**
3. **Explain TDD and its benefits.**

## 9.11 Key Takeaways

- Write tests before or alongside code (TDD)
- Focus on unit tests (fast, cheap, many)
- Use mocking to isolate units
- Aim for 80%+ code coverage
- Test behavior, not implementation
- Use debugger and profiling tools
- Monitor memory usage in production
- Automate testing in CI/CD pipeline

**Next:** Chapter 10 - Performance & Scalability
