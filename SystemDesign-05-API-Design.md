# Chapter 5: API Design & Communication

## 5.1 API Design Fundamentals

APIs (Application Programming Interfaces) are the contracts between different parts of your system and external consumers.

### Good API Design Principles

```
┌────────────────────────────────────────────────┐
│ 1. Consistency                                 │
│    - Predictable naming conventions            │
│    - Uniform error handling                    │
│    - Consistent data formats                   │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 2. Simplicity                                  │
│    - Clear, intuitive endpoints                │
│    - Minimal required parameters               │
│    - Self-documenting design                   │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 3. Flexibility                                 │
│    - Pagination for large datasets             │
│    - Filtering and sorting options             │
│    - Versioning for backward compatibility     │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 4. Security                                    │
│    - Authentication and authorization          │
│    - Rate limiting                             │
│    - Input validation                          │
└────────────────────────────────────────────────┘
```

---

## 5.2 REST API Design

**REST (Representational State Transfer)** is the most common API architectural style.

### RESTful Principles

**1. Resource-Based URLs**
```
Good:
GET    /users              - Get all users
GET    /users/123          - Get user 123
POST   /users              - Create user
PUT    /users/123          - Update user 123
DELETE /users/123          - Delete user 123

Bad:
GET    /getUsers
POST   /createUser
POST   /updateUser
POST   /deleteUser
```

**2. HTTP Methods Mapping**
```
┌────────────┬──────────────────┬─────────────────┐
│ Method     │ Action           │ Response        │
├────────────┼──────────────────┼─────────────────┤
│ GET        │ Read             │ 200 OK          │
│ POST       │ Create           │ 201 Created     │
│ PUT        │ Update (full)    │ 200 OK          │
│ PATCH      │ Update (partial) │ 200 OK          │
│ DELETE     │ Delete           │ 204 No Content  │
└────────────┴──────────────────┴─────────────────┘
```

**3. Status Codes**
```
2xx Success:
200 OK                  - Request succeeded
201 Created             - Resource created
204 No Content          - Success but no body

4xx Client Errors:
400 Bad Request         - Invalid input
401 Unauthorized        - Authentication required
403 Forbidden           - No permission
404 Not Found           - Resource doesn't exist
429 Too Many Requests   - Rate limit exceeded

5xx Server Errors:
500 Internal Server Error - Server crashed
503 Service Unavailable   - Server overloaded
```

### REST API Implementation

```javascript
const express = require('express');
const router = express.Router();

// GET /api/users - List users with pagination
router.get('/users', async (req, res) => {
  try {
    const { page = 1, limit = 20, sort = '-createdAt' } = req.query;

    const users = await User.find()
      .sort(sort)
      .limit(limit * 1)
      .skip((page - 1) * limit)
      .select('-password'); // Exclude sensitive fields

    const count = await User.countDocuments();

    res.json({
      users,
      totalPages: Math.ceil(count / limit),
      currentPage: page,
      total: count
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET /api/users/:id - Get single user
router.get('/users/:id', async (req, res) => {
  try {
    const user = await User.findById(req.params.id).select('-password');

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// POST /api/users - Create user
router.post('/users', async (req, res) => {
  try {
    const { email, name, password } = req.body;

    // Validation
    if (!email || !password) {
      return res.status(400).json({
        error: 'Email and password are required'
      });
    }

    // Check if exists
    const existing = await User.findOne({ email });
    if (existing) {
      return res.status(409).json({
        error: 'Email already registered'
      });
    }

    // Create user
    const user = await User.create({ email, name, password });

    res.status(201).json({
      id: user._id,
      email: user.email,
      name: user.name
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// PUT /api/users/:id - Update user (full update)
router.put('/users/:id', async (req, res) => {
  try {
    const { email, name } = req.body;

    const user = await User.findByIdAndUpdate(
      req.params.id,
      { email, name },
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// PATCH /api/users/:id - Update user (partial update)
router.patch('/users/:id', async (req, res) => {
  try {
    const updates = req.body;

    // Only allow certain fields to be updated
    const allowedUpdates = ['name', 'email', 'bio'];
    const actualUpdates = Object.keys(updates)
      .filter(key => allowedUpdates.includes(key))
      .reduce((obj, key) => {
        obj[key] = updates[key];
        return obj;
      }, {});

    const user = await User.findByIdAndUpdate(
      req.params.id,
      actualUpdates,
      { new: true, runValidators: true }
    ).select('-password');

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// DELETE /api/users/:id - Delete user
router.delete('/users/:id', async (req, res) => {
  try {
    const user = await User.findByIdAndDelete(req.params.id);

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.status(204).send();
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Advanced REST Patterns

#### 1. Nested Resources

```javascript
// GET /api/users/:userId/posts - Get user's posts
router.get('/users/:userId/posts', async (req, res) => {
  const posts = await Post.find({ userId: req.params.userId });
  res.json(posts);
});

// POST /api/users/:userId/posts - Create post for user
router.post('/users/:userId/posts', async (req, res) => {
  const post = await Post.create({
    userId: req.params.userId,
    ...req.body
  });
  res.status(201).json(post);
});

// GET /api/posts/:postId/comments - Get post's comments
router.get('/posts/:postId/comments', async (req, res) => {
  const comments = await Comment.find({ postId: req.params.postId });
  res.json(comments);
});
```

#### 2. Filtering and Searching

```javascript
// GET /api/products?category=electronics&minPrice=100&maxPrice=500
router.get('/products', async (req, res) => {
  const {
    category,
    minPrice,
    maxPrice,
    inStock,
    search,
    page = 1,
    limit = 20
  } = req.query;

  // Build query
  const query = {};

  if (category) {
    query.category = category;
  }

  if (minPrice || maxPrice) {
    query.price = {};
    if (minPrice) query.price.$gte = Number(minPrice);
    if (maxPrice) query.price.$lte = Number(maxPrice);
  }

  if (inStock === 'true') {
    query.stock = { $gt: 0 };
  }

  if (search) {
    query.$text = { $search: search };
  }

  const products = await Product.find(query)
    .limit(limit * 1)
    .skip((page - 1) * limit);

  const count = await Product.countDocuments(query);

  res.json({
    products,
    totalPages: Math.ceil(count / limit),
    currentPage: page
  });
});
```

#### 3. Field Selection (Sparse Fieldsets)

```javascript
// GET /api/users?fields=name,email
router.get('/users', async (req, res) => {
  const { fields } = req.query;

  let query = User.find();

  if (fields) {
    // Convert "name,email" to "name email"
    const selectedFields = fields.split(',').join(' ');
    query = query.select(selectedFields);
  }

  const users = await query;
  res.json(users);
});
```

#### 4. HATEOAS (Hypermedia)

```javascript
// Include links to related resources
router.get('/users/:id', async (req, res) => {
  const user = await User.findById(req.params.id);

  res.json({
    id: user._id,
    name: user.name,
    email: user.email,
    _links: {
      self: { href: `/api/users/${user._id}` },
      posts: { href: `/api/users/${user._id}/posts` },
      followers: { href: `/api/users/${user._id}/followers` },
      following: { href: `/api/users/${user._id}/following` }
    }
  });
});
```

---

## 5.3 GraphQL

**GraphQL** is a query language for APIs that allows clients to request exactly what they need.

### GraphQL vs REST

```
┌──────────────────────────────────────────────────┐
│ REST Problems:                                   │
│ - Over-fetching (get more data than needed)     │
│ - Under-fetching (need multiple requests)       │
│ - Versioning complexity                          │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ GraphQL Solutions:                               │
│ - Request exactly what you need                  │
│ - Single request for multiple resources          │
│ - Strong typing and schema                       │
└──────────────────────────────────────────────────┘
```

**REST Example (Multiple Requests):**
```
GET /api/users/123              → User data
GET /api/users/123/posts        → User's posts
GET /api/posts/456/comments     → Post's comments

3 HTTP requests, potential over-fetching
```

**GraphQL Example (Single Request):**
```graphql
query {
  user(id: "123") {
    name
    email
    posts(limit: 5) {
      title
      comments(limit: 3) {
        text
        author {
          name
        }
      }
    }
  }
}

1 HTTP request, exact data needed
```

### GraphQL Schema Definition

```javascript
const { ApolloServer, gql } = require('apollo-server');

// Define schema
const typeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
    followers: [User!]!
    following: [User!]!
  }

  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    comments: [Comment!]!
    likes: Int!
    createdAt: String!
  }

  type Comment {
    id: ID!
    text: String!
    author: User!
    post: Post!
    createdAt: String!
  }

  type Query {
    user(id: ID!): User
    users(limit: Int, offset: Int): [User!]!
    post(id: ID!): Post
    posts(limit: Int, offset: Int): [Post!]!
  }

  type Mutation {
    createUser(name: String!, email: String!, password: String!): User!
    createPost(title: String!, content: String!): Post!
    likePost(postId: ID!): Post!
    createComment(postId: ID!, text: String!): Comment!
  }

  type Subscription {
    postCreated: Post!
    commentAdded(postId: ID!): Comment!
  }
`;

// Resolvers
const resolvers = {
  Query: {
    user: async (parent, { id }, context) => {
      return await User.findById(id);
    },

    users: async (parent, { limit = 20, offset = 0 }, context) => {
      return await User.find().skip(offset).limit(limit);
    },

    post: async (parent, { id }, context) => {
      return await Post.findById(id);
    },

    posts: async (parent, { limit = 20, offset = 0 }, context) => {
      return await Post.find().skip(offset).limit(limit);
    }
  },

  Mutation: {
    createUser: async (parent, { name, email, password }, context) => {
      const user = await User.create({ name, email, password });
      return user;
    },

    createPost: async (parent, { title, content }, context) => {
      // Requires authentication
      if (!context.user) {
        throw new Error('Not authenticated');
      }

      const post = await Post.create({
        title,
        content,
        authorId: context.user.id
      });

      // Publish subscription
      pubsub.publish('POST_CREATED', { postCreated: post });

      return post;
    },

    likePost: async (parent, { postId }, context) => {
      const post = await Post.findById(postId);
      post.likes += 1;
      await post.save();
      return post;
    },

    createComment: async (parent, { postId, text }, context) => {
      if (!context.user) {
        throw new Error('Not authenticated');
      }

      const comment = await Comment.create({
        postId,
        text,
        authorId: context.user.id
      });

      // Publish subscription
      pubsub.publish('COMMENT_ADDED', {
        commentAdded: comment,
        postId
      });

      return comment;
    }
  },

  // Field resolvers
  User: {
    posts: async (user) => {
      return await Post.find({ authorId: user.id });
    },

    followers: async (user) => {
      const follows = await Follow.find({ followingId: user.id });
      const followerIds = follows.map(f => f.followerId);
      return await User.find({ _id: { $in: followerIds } });
    }
  },

  Post: {
    author: async (post) => {
      return await User.findById(post.authorId);
    },

    comments: async (post) => {
      return await Comment.find({ postId: post.id });
    }
  },

  Comment: {
    author: async (comment) => {
      return await User.findById(comment.authorId);
    },

    post: async (comment) => {
      return await Post.findById(comment.postId);
    }
  },

  Subscription: {
    postCreated: {
      subscribe: () => pubsub.asyncIterator(['POST_CREATED'])
    },

    commentAdded: {
      subscribe: (parent, { postId }) =>
        pubsub.asyncIterator(['COMMENT_ADDED'])
    }
  }
};

// Create server
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => {
    // Add user to context from JWT
    const token = req.headers.authorization || '';
    const user = getUserFromToken(token);
    return { user };
  }
});

server.listen().then(({ url }) => {
  console.log(`Server ready at ${url}`);
});
```

### GraphQL N+1 Problem

**Problem:** Inefficient database queries.

```javascript
// This query:
{
  posts {
    title
    author {
      name
    }
  }
}

// Naive resolver causes N+1 queries:
// 1 query for posts
// N queries for each author (one per post)

const resolvers = {
  Query: {
    posts: async () => {
      return await Post.find(); // 1 query
    }
  },
  Post: {
    author: async (post) => {
      return await User.findById(post.authorId); // N queries! 😱
    }
  }
};
```

**Solution: DataLoader (Batching)**

```javascript
const DataLoader = require('dataloader');

// Batch load users
const userLoader = new DataLoader(async (userIds) => {
  const users = await User.find({ _id: { $in: userIds } });

  // Return in same order as requested
  return userIds.map(id =>
    users.find(user => user.id.toString() === id.toString())
  );
});

// Use in resolver
const resolvers = {
  Post: {
    author: async (post, args, { loaders }) => {
      return await loaders.user.load(post.authorId);
    }
  }
};

// Context with loaders
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: () => ({
    loaders: {
      user: new DataLoader(batchLoadUsers),
      post: new DataLoader(batchLoadPosts)
    }
  })
});
```

### When to Use GraphQL

```
✓ Mobile apps (reduce data transfer)
✓ Multiple clients with different needs
✓ Rapid frontend development
✓ Complex, interconnected data
✓ Need real-time subscriptions

✗ Simple CRUD APIs
✗ Heavy caching requirements
✗ File uploads (better with REST)
✗ Public APIs (REST more familiar)
```

---

## 5.4 gRPC

**gRPC** is a high-performance RPC framework using Protocol Buffers.

### gRPC vs REST vs GraphQL

```
┌──────────────┬───────────┬──────────┬──────────┐
│ Feature      │ REST      │ GraphQL  │ gRPC     │
├──────────────┼───────────┼──────────┼──────────┤
│ Protocol     │ HTTP/1.1  │ HTTP/1.1 │ HTTP/2   │
│ Format       │ JSON      │ JSON     │ Protobuf │
│ Performance  │ Good      │ Good     │ Excellent│
│ Browser      │ ✓         │ ✓        │ Limited  │
│ Streaming    │ ✗         │ ✓        │ ✓        │
│ Learning     │ Easy      │ Medium   │ Hard     │
└──────────────┴───────────┴──────────┴──────────┘
```

### Protocol Buffers Definition

```protobuf
// user.proto
syntax = "proto3";

package user;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc StreamUsers(StreamUsersRequest) returns (stream User);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}

message GetUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
}

message StreamUsersRequest {
  string filter = 1;
}
```

### gRPC Server Implementation

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

// Load proto file
const packageDefinition = protoLoader.loadSync('user.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

// Implement service
const userService = {
  async getUser(call, callback) {
    try {
      const user = await User.findById(call.request.id);

      if (!user) {
        callback({
          code: grpc.status.NOT_FOUND,
          message: 'User not found'
        });
        return;
      }

      callback(null, {
        id: user._id.toString(),
        name: user.name,
        email: user.email,
        created_at: user.createdAt.getTime()
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message
      });
    }
  },

  async listUsers(call, callback) {
    try {
      const { page = 1, limit = 20 } = call.request;

      const users = await User.find()
        .skip((page - 1) * limit)
        .limit(limit);

      const total = await User.countDocuments();

      callback(null, {
        users: users.map(u => ({
          id: u._id.toString(),
          name: u.name,
          email: u.email,
          created_at: u.createdAt.getTime()
        })),
        total
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message
      });
    }
  },

  async createUser(call, callback) {
    try {
      const { name, email, password } = call.request;

      const user = await User.create({ name, email, password });

      callback(null, {
        id: user._id.toString(),
        name: user.name,
        email: user.email,
        created_at: user.createdAt.getTime()
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message
      });
    }
  },

  // Server streaming
  streamUsers(call) {
    const stream = User.find().cursor();

    stream.on('data', (user) => {
      call.write({
        id: user._id.toString(),
        name: user.name,
        email: user.email,
        created_at: user.createdAt.getTime()
      });
    });

    stream.on('end', () => {
      call.end();
    });

    stream.on('error', (error) => {
      call.destroy(error);
    });
  }
};

// Create and start server
const server = new grpc.Server();
server.addService(userProto.UserService.service, userService);
server.bindAsync(
  '0.0.0.0:50051',
  grpc.ServerCredentials.createInsecure(),
  () => {
    console.log('gRPC server running on port 50051');
    server.start();
  }
);
```

### gRPC Client

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync('user.proto');
const userProto = grpc.loadPackageDefinition(packageDefinition).user;

const client = new userProto.UserService(
  'localhost:50051',
  grpc.credentials.createInsecure()
);

// Unary RPC
client.getUser({ id: '123' }, (error, user) => {
  if (error) {
    console.error(error);
    return;
  }
  console.log('User:', user);
});

// Server streaming
const stream = client.streamUsers({ filter: 'active' });

stream.on('data', (user) => {
  console.log('Received user:', user);
});

stream.on('end', () => {
  console.log('Stream ended');
});

stream.on('error', (error) => {
  console.error('Stream error:', error);
});
```

### When to Use gRPC

```
✓ Microservices communication (internal)
✓ Real-time streaming
✓ Low latency requirements
✓ Polyglot services (many languages)
✓ Binary protocol efficiency

✗ Browser clients (limited support)
✗ Public APIs (less familiar)
✗ Simple CRUD (REST is easier)
```

---

## 5.5 WebSockets & Real-Time Communication

WebSockets enable bidirectional, real-time communication.

### WebSocket vs HTTP

```
HTTP (Request-Response):
Client ──Request──► Server
Client ◄─Response── Server
(Connection closes)

WebSocket (Persistent Connection):
Client ══════════► Server
       ◄══════════
       (Connection stays open)
       Both can send anytime
```

### WebSocket Server

```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

// Store connected clients
const clients = new Map();

wss.on('connection', (ws, req) => {
  const userId = getUserIdFromRequest(req);

  // Store client
  clients.set(userId, ws);
  console.log(`User ${userId} connected`);

  // Handle messages
  ws.on('message', (message) => {
    const data = JSON.parse(message);

    switch (data.type) {
      case 'chat':
        handleChatMessage(userId, data);
        break;
      case 'typing':
        handleTypingIndicator(userId, data);
        break;
      default:
        console.log('Unknown message type:', data.type);
    }
  });

  // Handle disconnection
  ws.on('close', () => {
    clients.delete(userId);
    console.log(`User ${userId} disconnected`);
  });

  // Send welcome message
  ws.send(JSON.stringify({
    type: 'connected',
    message: 'Welcome to the chat!'
  }));
});

function handleChatMessage(senderId, data) {
  const { recipientId, text } = data;

  // Save to database
  const message = await Message.create({
    senderId,
    recipientId,
    text,
    timestamp: Date.now()
  });

  // Send to recipient if online
  const recipientWs = clients.get(recipientId);
  if (recipientWs && recipientWs.readyState === WebSocket.OPEN) {
    recipientWs.send(JSON.stringify({
      type: 'chat',
      from: senderId,
      text,
      timestamp: message.timestamp
    }));
  }
}

function handleTypingIndicator(senderId, data) {
  const { recipientId, isTyping } = data;

  const recipientWs = clients.get(recipientId);
  if (recipientWs && recipientWs.readyState === WebSocket.OPEN) {
    recipientWs.send(JSON.stringify({
      type: 'typing',
      from: senderId,
      isTyping
    }));
  }
}
```

### WebSocket Client

```javascript
const ws = new WebSocket('ws://localhost:8080');

ws.onopen = () => {
  console.log('Connected to server');

  // Send message
  ws.send(JSON.stringify({
    type: 'chat',
    recipientId: '456',
    text: 'Hello!'
  }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);

  switch (data.type) {
    case 'chat':
      displayMessage(data);
      break;
    case 'typing':
      showTypingIndicator(data.from, data.isTyping);
      break;
    case 'connected':
      console.log(data.message);
      break;
  }
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

ws.onclose = () => {
  console.log('Disconnected from server');
  // Reconnect logic
  setTimeout(connect, 5000);
};
```

### Socket.IO (Enhanced WebSockets)

```javascript
const io = require('socket.io')(3000);

// Namespaces (separate channels)
const chatNamespace = io.of('/chat');
const notificationNamespace = io.of('/notifications');

chatNamespace.on('connection', (socket) => {
  console.log('User connected:', socket.id);

  // Join room
  socket.on('join-room', (roomId) => {
    socket.join(roomId);
    console.log(`User ${socket.id} joined room ${roomId}`);

    // Notify others in room
    socket.to(roomId).emit('user-joined', {
      userId: socket.id
    });
  });

  // Send message to room
  socket.on('send-message', (data) => {
    const { roomId, message } = data;

    // Broadcast to everyone in room except sender
    socket.to(roomId).emit('receive-message', {
      from: socket.id,
      message,
      timestamp: Date.now()
    });
  });

  // Private message
  socket.on('private-message', (data) => {
    const { recipientId, message } = data;

    // Send to specific socket
    io.to(recipientId).emit('receive-message', {
      from: socket.id,
      message,
      timestamp: Date.now()
    });
  });

  // Disconnect
  socket.on('disconnect', () => {
    console.log('User disconnected:', socket.id);
  });
});

// Client
const socket = io('http://localhost:3000/chat');

socket.emit('join-room', 'room1');

socket.on('receive-message', (data) => {
  console.log(`Message from ${data.from}: ${data.message}`);
});

socket.emit('send-message', {
  roomId: 'room1',
  message: 'Hello everyone!'
});
```

### Real-World: Slack-like Chat System

```javascript
class ChatService {
  constructor() {
    this.io = require('socket.io')(3000);
    this.setupHandlers();
  }

  setupHandlers() {
    this.io.on('connection', (socket) => {
      const userId = socket.handshake.auth.userId;

      // User authentication
      if (!userId) {
        socket.disconnect();
        return;
      }

      // Join user's personal room
      socket.join(`user:${userId}`);

      // Load user's channels
      this.loadUserChannels(userId).then(channels => {
        channels.forEach(channel => {
          socket.join(`channel:${channel.id}`);
        });
      });

      // Handle channel messages
      socket.on('message:send', async (data) => {
        const { channelId, text, attachments } = data;

        // Save message
        const message = await Message.create({
          channelId,
          userId,
          text,
          attachments,
          timestamp: Date.now()
        });

        // Broadcast to channel
        this.io.to(`channel:${channelId}`).emit('message:new', {
          id: message.id,
          channelId,
          userId,
          user: await this.getUser(userId),
          text,
          attachments,
          timestamp: message.timestamp
        });

        // Send push notifications to offline users
        this.notifyOfflineUsers(channelId, userId, text);
      });

      // Handle typing indicator
      socket.on('typing:start', (channelId) => {
        socket.to(`channel:${channelId}`).emit('typing:user', {
          userId,
          channelId
        });
      });

      socket.on('typing:stop', (channelId) => {
        socket.to(`channel:${channelId}`).emit('typing:stop', {
          userId,
          channelId
        });
      });

      // Handle reactions
      socket.on('reaction:add', async (data) => {
        const { messageId, emoji } = data;

        await Reaction.create({ messageId, userId, emoji });

        const message = await Message.findById(messageId);

        this.io.to(`channel:${message.channelId}`).emit('reaction:added', {
          messageId,
          userId,
          emoji
        });
      });

      // Handle presence
      this.updateUserPresence(userId, 'online');

      socket.on('disconnect', () => {
        this.updateUserPresence(userId, 'offline');
      });
    });
  }

  async loadUserChannels(userId) {
    return await Channel.find({
      members: userId
    });
  }

  async updateUserPresence(userId, status) {
    await User.findByIdAndUpdate(userId, {
      status,
      lastSeen: Date.now()
    });

    // Broadcast presence update
    this.io.emit('presence:update', {
      userId,
      status,
      lastSeen: Date.now()
    });
  }

  async notifyOfflineUsers(channelId, senderId, text) {
    const channel = await Channel.findById(channelId);
    const offlineUsers = await User.find({
      _id: { $in: channel.members, $ne: senderId },
      status: 'offline'
    });

    for (const user of offlineUsers) {
      await this.sendPushNotification(user, {
        title: `New message in ${channel.name}`,
        body: text
      });
    }
  }
}
```

---

## 5.6 Message Queues & Async Communication

Message queues enable asynchronous, decoupled communication between services.

### Why Message Queues?

```
Without Queue (Synchronous):
Client → Service A → Service B → Service C
         (waits)    (waits)    (waits)
         All must be online and fast

With Queue (Asynchronous):
Client → Service A → Queue
         ↓ (returns immediately)
         Done!

         Queue → Service B → Process
                Service C → Process
                (process when ready)
```

### Common Message Queue Patterns

#### 1. Work Queue (Task Distribution)

```
Producer → [Queue] → Consumer 1
                  → Consumer 2
                  → Consumer 3

Multiple workers process tasks in parallel
```

**Implementation with RabbitMQ:**

```javascript
const amqp = require('amqplib');

// Producer
class TaskProducer {
  async connect() {
    this.connection = await amqp.connect('amqp://localhost');
    this.channel = await this.connection.createChannel();
    await this.channel.assertQueue('tasks', { durable: true });
  }

  async sendTask(task) {
    this.channel.sendToQueue(
      'tasks',
      Buffer.from(JSON.stringify(task)),
      { persistent: true }
    );
    console.log('Task sent:', task);
  }
}

// Consumer
class TaskConsumer {
  async connect() {
    this.connection = await amqp.connect('amqp://localhost');
    this.channel = await this.connection.createChannel();
    await this.channel.assertQueue('tasks', { durable: true });

    // Fair dispatch (one task at a time)
    this.channel.prefetch(1);
  }

  async start() {
    this.channel.consume('tasks', async (msg) => {
      const task = JSON.parse(msg.content.toString());

      console.log('Processing task:', task);

      try {
        await this.processTask(task);

        // Acknowledge completion
        this.channel.ack(msg);
      } catch (error) {
        console.error('Task failed:', error);

        // Reject and requeue
        this.channel.nack(msg, false, true);
      }
    });
  }

  async processTask(task) {
    // Simulate work
    await new Promise(resolve => setTimeout(resolve, 1000));
    console.log('Task completed:', task);
  }
}

// Usage
const producer = new TaskProducer();
await producer.connect();
await producer.sendTask({ type: 'email', to: 'user@example.com' });

const consumer = new TaskConsumer();
await consumer.connect();
await consumer.start();
```

#### 2. Pub/Sub (Broadcast)

```
Publisher → [Topic] → Subscriber 1
                   → Subscriber 2
                   → Subscriber 3

All subscribers receive all messages
```

**Implementation with Redis:**

```javascript
const Redis = require('ioredis');

// Publisher
class EventPublisher {
  constructor() {
    this.redis = new Redis();
  }

  publish(channel, event) {
    this.redis.publish(channel, JSON.stringify(event));
    console.log(`Published to ${channel}:`, event);
  }
}

// Subscriber
class EventSubscriber {
  constructor() {
    this.redis = new Redis();
  }

  subscribe(channel, handler) {
    this.redis.subscribe(channel);

    this.redis.on('message', (ch, message) => {
      if (ch === channel) {
        const event = JSON.parse(message);
        handler(event);
      }
    });
  }
}

// Usage
const publisher = new EventPublisher();
const subscriber1 = new EventSubscriber();
const subscriber2 = new EventSubscriber();

subscriber1.subscribe('user.created', (event) => {
  console.log('Subscriber 1 received:', event);
  // Send welcome email
});

subscriber2.subscribe('user.created', (event) => {
  console.log('Subscriber 2 received:', event);
  // Create user profile
});

publisher.publish('user.created', {
  userId: '123',
  email: 'user@example.com'
});
```

#### 3. Request-Reply Pattern

```
Client → [Request Queue] → Server
       ← [Reply Queue]   ←
```

**Implementation:**

```javascript
class RPCClient {
  constructor() {
    this.pendingRequests = new Map();
  }

  async connect() {
    this.connection = await amqp.connect('amqp://localhost');
    this.channel = await this.connection.createChannel();

    // Create reply queue
    const { queue } = await this.channel.assertQueue('', { exclusive: true });
    this.replyQueue = queue;

    // Listen for replies
    this.channel.consume(this.replyQueue, (msg) => {
      const correlationId = msg.properties.correlationId;
      const resolve = this.pendingRequests.get(correlationId);

      if (resolve) {
        const result = JSON.parse(msg.content.toString());
        resolve(result);
        this.pendingRequests.delete(correlationId);
      }

      this.channel.ack(msg);
    });
  }

  async call(method, params) {
    const correlationId = generateUUID();

    return new Promise((resolve) => {
      this.pendingRequests.set(correlationId, resolve);

      this.channel.sendToQueue(
        'rpc_queue',
        Buffer.from(JSON.stringify({ method, params })),
        {
          correlationId,
          replyTo: this.replyQueue
        }
      );

      // Timeout
      setTimeout(() => {
        if (this.pendingRequests.has(correlationId)) {
          this.pendingRequests.delete(correlationId);
          resolve({ error: 'Timeout' });
        }
      }, 5000);
    });
  }
}

// RPC Server
class RPCServer {
  async connect() {
    this.connection = await amqp.connect('amqp://localhost');
    this.channel = await this.connection.createChannel();
    await this.channel.assertQueue('rpc_queue');
  }

  async start() {
    this.channel.consume('rpc_queue', async (msg) => {
      const { method, params } = JSON.parse(msg.content.toString());

      try {
        const result = await this.handleMethod(method, params);

        this.channel.sendToQueue(
          msg.properties.replyTo,
          Buffer.from(JSON.stringify(result)),
          { correlationId: msg.properties.correlationId }
        );
      } catch (error) {
        this.channel.sendToQueue(
          msg.properties.replyTo,
          Buffer.from(JSON.stringify({ error: error.message })),
          { correlationId: msg.properties.correlationId }
        );
      }

      this.channel.ack(msg);
    });
  }

  async handleMethod(method, params) {
    switch (method) {
      case 'add':
        return params.a + params.b;
      case 'multiply':
        return params.a * params.b;
      default:
        throw new Error('Unknown method');
    }
  }
}

// Usage
const client = new RPCClient();
await client.connect();

const result = await client.call('add', { a: 5, b: 3 });
console.log('Result:', result); // 8
```

### Real-World: E-Commerce Order Processing

```javascript
class OrderProcessingSystem {
  constructor() {
    this.producer = new EventPublisher();
    this.setupConsumers();
  }

  // User places order
  async placeOrder(orderData) {
    // 1. Save order
    const order = await Order.create({
      ...orderData,
      status: 'pending'
    });

    // 2. Publish event (async processing)
    this.producer.publish('order.created', {
      orderId: order.id,
      userId: order.userId,
      items: order.items,
      total: order.total
    });

    // 3. Return immediately
    return {
      orderId: order.id,
      status: 'pending',
      message: 'Order is being processed'
    };
  }

  setupConsumers() {
    // Payment service
    const paymentService = new EventSubscriber();
    paymentService.subscribe('order.created', async (event) => {
      console.log('Processing payment for order:', event.orderId);

      try {
        await this.processPayment(event);

        this.producer.publish('payment.completed', {
          orderId: event.orderId
        });
      } catch (error) {
        this.producer.publish('payment.failed', {
          orderId: event.orderId,
          error: error.message
        });
      }
    });

    // Inventory service
    const inventoryService = new EventSubscriber();
    inventoryService.subscribe('payment.completed', async (event) => {
      console.log('Reserving inventory for order:', event.orderId);

      try {
        await this.reserveInventory(event.orderId);

        this.producer.publish('inventory.reserved', {
          orderId: event.orderId
        });
      } catch (error) {
        this.producer.publish('inventory.failed', {
          orderId: event.orderId,
          error: error.message
        });

        // Compensating transaction: refund payment
        this.producer.publish('payment.refund', {
          orderId: event.orderId
        });
      }
    });

    // Shipping service
    const shippingService = new EventSubscriber();
    shippingService.subscribe('inventory.reserved', async (event) => {
      console.log('Creating shipment for order:', event.orderId);

      await this.createShipment(event.orderId);

      this.producer.publish('shipment.created', {
        orderId: event.orderId
      });
    });

    // Notification service
    const notificationService = new EventSubscriber();
    notificationService.subscribe('shipment.created', async (event) => {
      const order = await Order.findById(event.orderId);

      await this.sendEmail(order.userId, {
        subject: 'Order shipped!',
        body: `Your order ${order.id} has been shipped.`
      });
    });
  }

  async processPayment(event) {
    // Payment processing logic
    await new Promise(resolve => setTimeout(resolve, 1000));
  }

  async reserveInventory(orderId) {
    // Inventory reservation logic
    await new Promise(resolve => setTimeout(resolve, 500));
  }

  async createShipment(orderId) {
    // Shipment creation logic
    await new Promise(resolve => setTimeout(resolve, 800));
  }
}
```

---

## 5.7 API Gateway

API Gateway is a single entry point for all client requests, routing to appropriate microservices.

### API Gateway Responsibilities

```
┌────────────────────────────────────────────────┐
│ 1. Routing                                     │
│    - Route requests to correct service         │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 2. Authentication & Authorization              │
│    - Verify tokens                             │
│    - Check permissions                         │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 3. Rate Limiting                               │
│    - Prevent abuse                             │
│    - Throttle requests                         │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 4. Request/Response Transformation             │
│    - Format conversion                         │
│    - Aggregation                               │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 5. Load Balancing                              │
│    - Distribute load                           │
│    - Health checks                             │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 6. Caching                                     │
│    - Cache responses                           │
│    - Reduce backend load                       │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│ 7. Logging & Monitoring                        │
│    - Centralized logging                       │
│    - Metrics collection                        │
└────────────────────────────────────────────────┘
```

### API Gateway Architecture

```
                    ┌──────────────┐
                    │   Clients    │
                    │ (Web/Mobile) │
                    └──────┬───────┘
                           │
                           ↓
                    ┌──────────────┐
                    │ API Gateway  │
                    │ - Auth       │
                    │ - Rate Limit │
                    │ - Routing    │
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ↓                ↓                ↓
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │  User    │    │ Product  │    │  Order   │
    │ Service  │    │ Service  │    │ Service  │
    └──────────┘    └──────────┘    └──────────┘
```

### API Gateway Implementation

```javascript
const express = require('express');
const httpProxy = require('http-proxy');
const redis = require('redis');

class APIGateway {
  constructor() {
    this.app = express();
    this.proxy = httpProxy.createProxyServer();
    this.cache = redis.createClient();

    this.services = {
      user: 'http://localhost:3001',
      product: 'http://localhost:3002',
      order: 'http://localhost:3003'
    };

    this.setupMiddleware();
    this.setupRoutes();
  }

  setupMiddleware() {
    // 1. Logging
    this.app.use((req, res, next) => {
      console.log(`${req.method} ${req.path} from ${req.ip}`);
      next();
    });

    // 2. Authentication
    this.app.use(async (req, res, next) => {
      // Public routes
      if (req.path.startsWith('/auth/')) {
        return next();
      }

      const token = req.headers.authorization?.split(' ')[1];

      if (!token) {
        return res.status(401).json({ error: 'No token provided' });
      }

      try {
        const user = await this.verifyToken(token);
        req.user = user;
        next();
      } catch (error) {
        res.status(401).json({ error: 'Invalid token' });
      }
    });

    // 3. Rate Limiting
    this.app.use(async (req, res, next) => {
      const key = `rate_limit:${req.ip}`;
      const limit = 100; // requests per minute
      const window = 60; // seconds

      const current = await this.cache.incr(key);

      if (current === 1) {
        await this.cache.expire(key, window);
      }

      if (current > limit) {
        return res.status(429).json({
          error: 'Too many requests',
          retryAfter: await this.cache.ttl(key)
        });
      }

      res.setHeader('X-RateLimit-Limit', limit);
      res.setHeader('X-RateLimit-Remaining', limit - current);

      next();
    });
  }

  setupRoutes() {
    // Route to User Service
    this.app.all('/api/users/*', async (req, res) => {
      await this.proxyRequest(req, res, 'user');
    });

    // Route to Product Service
    this.app.all('/api/products/*', async (req, res) => {
      // Check cache for GET requests
      if (req.method === 'GET') {
        const cached = await this.cache.get(req.path);
        if (cached) {
          return res.json(JSON.parse(cached));
        }
      }

      await this.proxyRequest(req, res, 'product');
    });

    // Route to Order Service
    this.app.all('/api/orders/*', async (req, res) => {
      // Check permissions
      if (!req.user || !req.user.canAccessOrders) {
        return res.status(403).json({ error: 'Forbidden' });
      }

      await this.proxyRequest(req, res, 'order');
    });

    // Aggregated endpoint (multiple services)
    this.app.get('/api/dashboard', async (req, res) => {
      try {
        const [user, orders, recommendations] = await Promise.all([
          this.callService('user', `/users/${req.user.id}`),
          this.callService('order', `/orders/user/${req.user.id}`),
          this.callService('product', `/recommendations/${req.user.id}`)
        ]);

        res.json({
          user,
          recentOrders: orders.slice(0, 5),
          recommendations
        });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
  }

  async proxyRequest(req, res, serviceName) {
    const target = this.services[serviceName];

    if (!target) {
      return res.status(404).json({ error: 'Service not found' });
    }

    this.proxy.web(req, res, { target }, (error) => {
      console.error(`Proxy error for ${serviceName}:`, error);
      res.status(503).json({ error: 'Service unavailable' });
    });
  }

  async callService(serviceName, path) {
    const target = this.services[serviceName];
    const response = await fetch(`${target}${path}`);

    if (!response.ok) {
      throw new Error(`${serviceName} service error: ${response.statusText}`);
    }

    return await response.json();
  }

  async verifyToken(token) {
    // JWT verification logic
    return { id: '123', canAccessOrders: true };
  }

  start(port = 3000) {
    this.app.listen(port, () => {
      console.log(`API Gateway running on port ${port}`);
    });
  }
}

const gateway = new APIGateway();
gateway.start();
```

---

## 5.8 Rate Limiting & Throttling

Rate limiting prevents abuse and ensures fair usage.

### Rate Limiting Algorithms

#### 1. Fixed Window

```
Window: 1 minute

00:00 - 00:59 → Allow 100 requests
01:00 - 01:59 → Reset, allow 100 requests

Problem: Burst at window boundaries
00:59 → 100 requests
01:00 → 100 requests
(200 requests in 2 seconds!)
```

**Implementation:**

```javascript
class FixedWindowRateLimiter {
  constructor(redis, limit, windowSeconds) {
    this.redis = redis;
    this.limit = limit;
    this.windowSeconds = windowSeconds;
  }

  async isAllowed(userId) {
    const now = Date.now();
    const windowStart = Math.floor(now / (this.windowSeconds * 1000));
    const key = `rate_limit:${userId}:${windowStart}`;

    const current = await this.redis.incr(key);

    if (current === 1) {
      await this.redis.expire(key, this.windowSeconds);
    }

    return current <= this.limit;
  }
}
```

#### 2. Sliding Window

```
More accurate, considers rolling time window

At 00:30 with 1-minute window:
Count requests from 23:30 to 00:30

Prevents burst at boundaries
```

**Implementation:**

```javascript
class SlidingWindowRateLimiter {
  constructor(redis, limit, windowMs) {
    this.redis = redis;
    this.limit = limit;
    this.windowMs = windowMs;
  }

  async isAllowed(userId) {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    const key = `rate_limit:${userId}`;

    // Add current request
    await this.redis.zadd(key, now, `${now}-${Math.random()}`);

    // Remove old requests
    await this.redis.zremrangebyscore(key, 0, windowStart);

    // Count requests in window
    const count = await this.redis.zcard(key);

    // Set expiry
    await this.redis.expire(key, Math.ceil(this.windowMs / 1000));

    return count <= this.limit;
  }
}
```

#### 3. Token Bucket

```
Bucket has N tokens
- Each request consumes 1 token
- Tokens refill at fixed rate
- Allows bursts (up to bucket size)

Example:
Bucket size: 10 tokens
Refill rate: 1 token/second

Can handle burst of 10 requests immediately
Then 1 request/second sustained
```

**Implementation:**

```javascript
class TokenBucketRateLimiter {
  constructor(redis, capacity, refillRate) {
    this.redis = redis;
    this.capacity = capacity;
    this.refillRate = refillRate; // tokens per second
  }

  async isAllowed(userId) {
    const key = `token_bucket:${userId}`;
    const now = Date.now();

    // Get bucket state
    const bucket = await this.redis.hgetall(key);

    let tokens = bucket.tokens ? parseFloat(bucket.tokens) : this.capacity;
    let lastRefill = bucket.lastRefill ? parseInt(bucket.lastRefill) : now;

    // Refill tokens
    const timePassed = (now - lastRefill) / 1000; // seconds
    const tokensToAdd = timePassed * this.refillRate;
    tokens = Math.min(this.capacity, tokens + tokensToAdd);

    // Check if request allowed
    if (tokens >= 1) {
      tokens -= 1;

      // Update bucket
      await this.redis.hset(key, {
        tokens: tokens.toString(),
        lastRefill: now.toString()
      });
      await this.redis.expire(key, 3600);

      return true;
    }

    return false;
  }
}
```

#### 4. Leaky Bucket

```
Bucket processes requests at fixed rate
Excess requests queue up (or dropped)

Like water dripping from bucket at constant rate
```

**Implementation:**

```javascript
class LeakyBucketRateLimiter {
  constructor(redis, capacity, leakRate) {
    this.redis = redis;
    this.capacity = capacity;
    this.leakRate = leakRate; // requests per second
  }

  async isAllowed(userId) {
    const key = `leaky_bucket:${userId}`;
    const now = Date.now();

    // Lua script for atomic operation
    const script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local leakRate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])

      local bucket = redis.call('HGETALL', key)
      local count = 0
      local lastLeak = now

      if #bucket > 0 then
        count = tonumber(bucket[2])
        lastLeak = tonumber(bucket[4])
      end

      -- Leak
      local timePassed = (now - lastLeak) / 1000
      local leaked = math.floor(timePassed * leakRate)
      count = math.max(0, count - leaked)

      -- Add request
      if count < capacity then
        count = count + 1
        redis.call('HSET', key, 'count', count, 'lastLeak', now)
        redis.call('EXPIRE', key, 3600)
        return 1
      else
        return 0
      end
    `;

    const result = await this.redis.eval(
      script,
      1,
      key,
      this.capacity,
      this.leakRate,
      now
    );

    return result === 1;
  }
}
```

### Rate Limiting Strategies

```
┌──────────────────┬────────────┬────────────────┐
│ Algorithm        │ Burst      │ Memory         │
├──────────────────┼────────────┼────────────────┤
│ Fixed Window     │ Poor       │ Low            │
│ Sliding Window   │ Good       │ High           │
│ Token Bucket     │ Excellent  │ Low            │
│ Leaky Bucket     │ No         │ Low            │
└──────────────────┴────────────┴────────────────┘
```

---

## 5.9 Key Takeaways

### API Style Selection

```
Choose REST when:
✓ Simple CRUD operations
✓ Public APIs
✓ Caching important
✓ Standard HTTP tooling

Choose GraphQL when:
✓ Multiple client types
✓ Complex data requirements
✓ Rapid frontend iteration
✓ Real-time subscriptions

Choose gRPC when:
✓ Internal microservices
✓ Low latency critical
✓ Streaming needed
✓ Polyglot environment
```

### Communication Patterns

```
Synchronous (REST/gRPC):
✓ Immediate response needed
✓ Simple request-reply
✗ Tight coupling
✗ Availability dependency

Asynchronous (Message Queue):
✓ Decoupled services
✓ High throughput
✓ Resilience to failures
✗ Eventual consistency
✗ Complex debugging
```

---

## 5.10 Interview Questions

### Basic:
1. What are the main principles of REST API design?
2. Explain the difference between PUT and PATCH.
3. What is rate limiting and why is it needed?

### Intermediate:
1. Compare REST, GraphQL, and gRPC. When would you use each?
2. What is the N+1 problem in GraphQL and how do you solve it?
3. Explain different rate limiting algorithms.

### Advanced:
1. Design an API Gateway for a microservices architecture.
2. How would you implement real-time notifications for 10M users?
3. Design the API for a WhatsApp-like messaging system.

---

**Next Chapter:** [Chapter 6: Real-World Case Studies](./SystemDesign-06-Case-Studies.md)

In the next chapter, we'll design complete systems:
- URL Shortener (like Bitly)
- Social Media Platform (like Twitter)
- Video Streaming Service (like Netflix)
- Ride-Sharing Platform (like Uber)
- Messaging System (like WhatsApp)
