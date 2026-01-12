# Chapter 8: Authentication & Security

## 8.1 Authentication vs Authorization

```
┌────────────────────────────────────────┐
│  Authentication: Who are you?          │
│  └─▶ Login, JWT, OAuth                 │
│                                         │
│  Authorization: What can you do?        │
│  └─▶ Permissions, Roles, Access Control│
└────────────────────────────────────────┘
```

## 8.2 Password Hashing with bcrypt

```javascript
const bcrypt = require('bcrypt');

// Hash password
async function hashPassword(password) {
  const salt = await bcrypt.genSalt(10);
  return await bcrypt.hash(password, salt);
}

// Verify password
async function verifyPassword(password, hash) {
  return await bcrypt.compare(password, hash);
}

// Registration
app.post('/register', async (req, res) => {
  const { email, password } = req.body;
  const hashedPassword = await hashPassword(password);

  const user = await User.create({
    email,
    password: hashedPassword
  });

  res.status(201).json({ message: 'User created' });
});

// Login
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });

  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const isValid = await verifyPassword(password, user.password);
  if (!isValid) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Generate token
  const token = generateToken(user);
  res.json({ token });
});
```

## 8.3 JWT (JSON Web Tokens)

### JWT Structure:
```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJ1c2VySWQiOiIxMjMiLCJpYXQiOjE2MTYyMzkwMjJ9.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Implementation:
```javascript
const jwt = require('jsonwebtoken');

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

// Generate token
function generateToken(user) {
  return jwt.sign(
    { userId: user._id, email: user.email },
    JWT_SECRET,
    { expiresIn: '24h' }
  );
}

// Verify token middleware
const authMiddleware = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1]; // Bearer TOKEN

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Protected route
app.get('/profile', authMiddleware, async (req, res) => {
  const user = await User.findById(req.user.userId);
  res.json(user);
});

// Refresh token pattern
function generateTokens(user) {
  const accessToken = jwt.sign(
    { userId: user._id },
    JWT_SECRET,
    { expiresIn: '15m' }  // Short-lived
  );

  const refreshToken = jwt.sign(
    { userId: user._id },
    REFRESH_SECRET,
    { expiresIn: '7d' }  // Long-lived
  );

  return { accessToken, refreshToken };
}
```

## 8.4 OAuth 2.0

```
┌──────────┐                               ┌───────────┐
│  Client  │                               │  Server   │
└────┬─────┘                               └─────┬─────┘
     │                                           │
     │  1. Authorization Request                 │
     ├──────────────────────────────────────────▶│
     │                                           │
     │  2. Authorization Grant                   │
     │◀──────────────────────────────────────────┤
     │                                           │
     │  3. Access Token Request                  │
     ├──────────────────────────────────────────▶│
     │                                           │
     │  4. Access Token                          │
     │◀──────────────────────────────────────────┤
     │                                           │
     │  5. Protected Resource Request            │
     ├──────────────────────────────────────────▶│
     │                                           │
     │  6. Protected Resource                    │
     │◀──────────────────────────────────────────┤
```

### Google OAuth Example:
```javascript
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackURL: '/auth/google/callback'
  },
  async (accessToken, refreshToken, profile, done) => {
    // Find or create user
    let user = await User.findOne({ googleId: profile.id });

    if (!user) {
      user = await User.create({
        googleId: profile.id,
        email: profile.emails[0].value,
        name: profile.displayName
      });
    }

    done(null, user);
  }
));

// Routes
app.get('/auth/google',
  passport.authenticate('google', { scope: ['profile', 'email'] })
);

app.get('/auth/google/callback',
  passport.authenticate('google', { failureRedirect: '/login' }),
  (req, res) => {
    res.redirect('/dashboard');
  }
);
```

## 8.5 Security Best Practices

### 1. Helmet (Security Headers)
```javascript
const helmet = require('helmet');
app.use(helmet());

// Sets various HTTP headers:
// - X-DNS-Prefetch-Control
// - X-Frame-Options
// - Strict-Transport-Security
// - X-Content-Type-Options
```

### 2. CORS (Cross-Origin Resource Sharing)
```javascript
const cors = require('cors');

app.use(cors({
  origin: 'https://yourdomain.com',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### 3. Rate Limiting
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,  // Limit each IP to 100 requests per window
  message: 'Too many requests from this IP'
});

app.use('/api/', limiter);
```

### 4. Input Validation
```javascript
const { body, validationResult } = require('express-validator');

app.post('/user',
  body('email').isEmail().normalizeEmail(),
  body('password').isLength({ min: 8 }),
  body('age').optional().isInt({ min: 18 }),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // Process request
  }
);
```

### 5. XSS Protection
```javascript
const xss = require('xss-clean');
app.use(xss());  // Sanitizes user input

// Manual sanitization
const sanitize = require('sanitize-html');
const clean = sanitize(userInput, {
  allowedTags: [],
  allowedAttributes: {}
});
```

### 6. SQL Injection Prevention
```javascript
// VULNERABLE
const query = `SELECT * FROM users WHERE id = ${userId}`;

// SAFE (Parameterized queries)
const query = 'SELECT * FROM users WHERE id = $1';
await pool.query(query, [userId]);
```

### 7. CSRF Protection
```javascript
const csrf = require('csurf');
const csrfProtection = csrf({ cookie: true });

app.get('/form', csrfProtection, (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});

app.post('/process', csrfProtection, (req, res) => {
  // Process form
});
```

### 8. Environment Variables
```javascript
// .env file
JWT_SECRET=your-super-secret-key
DB_PASSWORD=your-database-password

// Load with dotenv
require('dotenv').config();

// Access
const secret = process.env.JWT_SECRET;
```

## 8.6 Common Vulnerabilities (OWASP Top 10)

```
1. Injection (SQL, NoSQL, Command)
2. Broken Authentication
3. Sensitive Data Exposure
4. XML External Entities (XXE)
5. Broken Access Control
6. Security Misconfiguration
7. Cross-Site Scripting (XSS)
8. Insecure Deserialization
9. Using Components with Known Vulnerabilities
10. Insufficient Logging & Monitoring
```

## 8.7 Practical Example: Complete Auth System

```javascript
// auth.controller.js
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.register = async (req, res) => {
  try {
    const { email, password, name } = req.body;

    // Check if user exists
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ error: 'Email already registered' });
    }

    // Hash password
    const hashedPassword = await bcrypt.hash(password, 10);

    // Create user
    const user = await User.create({
      email,
      password: hashedPassword,
      name
    });

    // Generate token
    const token = jwt.sign(
      { userId: user._id },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.status(201).json({
      message: 'User registered successfully',
      token,
      user: { id: user._id, email: user.email, name: user.name }
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;

    // Find user
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Verify password
    const isValid = await bcrypt.compare(password, user.password);
    if (!isValid) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Generate token
    const token = jwt.sign(
      { userId: user._id },
      process.env.JWT_SECRET,
      { expiresIn: '24h' }
    );

    res.json({
      message: 'Login successful',
      token,
      user: { id: user._id, email: user.email, name: user.name }
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

// middleware/auth.js
const jwt = require('jsonwebtoken');

exports.protect = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return res.status(401).json({ error: 'Not authorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

exports.restrictTo = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
};
```

## 8.8 Interview Questions

### Basic:
1. **What's the difference between authentication and authorization?**
2. **Why should you hash passwords?**
3. **What is JWT?**

### Intermediate:
1. **Explain the JWT structure and how it works.**
2. **How do you implement password reset functionality?**
3. **What's the difference between access and refresh tokens?**

### Advanced:
1. **Design a secure authentication system with 2FA.**
2. **How do you prevent timing attacks in authentication?**
3. **Explain OAuth 2.0 flow in detail.**

## 8.9 Key Takeaways

- Never store plain text passwords - always hash with bcrypt
- Use JWT for stateless authentication
- Implement rate limiting to prevent brute force attacks
- Sanitize user input to prevent XSS and injection attacks
- Use HTTPS in production
- Keep dependencies updated
- Implement proper error handling without leaking information
- Use environment variables for secrets

**Next:** Chapter 9 - Testing & Debugging
