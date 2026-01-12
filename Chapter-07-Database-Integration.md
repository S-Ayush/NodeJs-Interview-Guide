# Chapter 7: Database Integration

## 7.1 Database Types

```
┌─────────────────────────────────────────┐
│        Database Types                    │
│                                          │
│  SQL (Relational)    NoSQL (Non-Relational│
│  ├─ PostgreSQL      ├─ MongoDB          │
│  ├─ MySQL           ├─ Redis            │
│  └─ SQLite          └─ Cassandra        │
└─────────────────────────────────────────┘
```

## 7.2 MongoDB with Mongoose

### Installation:
```bash
npm install mongoose
```

### Connection:
```javascript
const mongoose = require('mongoose');

mongoose.connect('mongodb://localhost:27017/myapp', {
  useNewUrlParser: true,
  useUnifiedTopology: true
})
  .then(() => console.log('MongoDB connected'))
  .catch(err => console.error('Connection error:', err));
```

### Schema & Model:
```javascript
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  age: { type: Number, min: 18 },
  createdAt: { type: Date, default: Date.now }
});

const User = mongoose.model('User', userSchema);
```

### CRUD Operations:
```javascript
// Create
const user = await User.create({
  name: 'John',
  email: 'john@example.com',
  age: 25
});

// Read
const users = await User.find();
const oneUser = await User.findById(id);
const filtered = await User.find({ age: { $gte: 18 } });

// Update
await User.findByIdAndUpdate(id, { name: 'Jane' }, { new: true });
await User.updateMany({ age: { $lt: 18 } }, { verified: false });

// Delete
await User.findByIdAndDelete(id);
await User.deleteMany({ verified: false });
```

### Relationships:
```javascript
// One-to-Many (Reference)
const postSchema = new mongoose.Schema({
  title: String,
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User' }
});

// Population
const post = await Post.findById(id).populate('author');

// Embedded documents
const userSchema = new mongoose.Schema({
  name: String,
  addresses: [{
    street: String,
    city: String
  }]
});
```

## 7.3 PostgreSQL with node-postgres

### Installation:
```bash
npm install pg
```

### Connection Pool:
```javascript
const { Pool } = require('pg');

const pool = new Pool({
  user: 'postgres',
  host: 'localhost',
  database: 'myapp',
  password: 'password',
  port: 5432,
  max: 20,
  idleTimeoutMillis: 30000
});
```

### Queries:
```javascript
// Simple query
const result = await pool.query('SELECT * FROM users');
console.log(result.rows);

// Parameterized query (prevents SQL injection)
const user = await pool.query(
  'SELECT * FROM users WHERE id = $1',
  [userId]
);

// Transaction
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('INSERT INTO users(name) VALUES($1)', ['John']);
  await client.query('INSERT INTO logs(action) VALUES($1)', ['user_created']);
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release();
}
```

## 7.4 Connection Pooling

```
┌────────────────────────────────────────┐
│        Connection Pool                  │
│  ┌────┐  ┌────┐  ┌────┐  ┌────┐       │
│  │Conn│  │Conn│  │Conn│  │Conn│       │
│  │ 1  │  │ 2  │  │ 3  │  │ 4  │       │
│  └─┬──┘  └─┬──┘  └─┬──┘  └─┬──┘       │
│    │       │       │       │           │
│    └───────┴───────┴───────┘           │
│              │                          │
│         Available for                   │
│         requests                        │
└────────────────────────────────────────┘
```

## 7.5 Interview Questions

### Basic:
1. **What's the difference between SQL and NoSQL?**
2. **What is Mongoose?**
3. **How do you prevent SQL injection?**

### Intermediate:
1. **Explain connection pooling and its benefits.**
2. **What are transactions and when to use them?**
3. **How do you handle relationships in MongoDB?**

### Advanced:
1. **Design database schema for a social media app.**
2. **Implement optimistic locking in MongoDB.**
3. **Explain ACID properties with examples.**

## 7.6 Key Takeaways

- Use connection pooling for better performance
- Always use parameterized queries to prevent SQL injection
- Mongoose provides schema validation and middleware
- Transactions ensure data consistency
- Choose database based on use case (SQL vs NoSQL)

**Next:** Chapter 8 - Authentication & Security
