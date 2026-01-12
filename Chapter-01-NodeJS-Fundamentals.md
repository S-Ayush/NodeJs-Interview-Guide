# Chapter 1: Node.js Fundamentals & Architecture

## 1.1 What is Node.js?

Node.js is a JavaScript runtime built on Chrome's V8 JavaScript engine. It's not a language, framework, or library - it's a runtime environment that allows you to execute JavaScript code outside of a web browser.

### Key Characteristics:
- **Server-side JavaScript**: Run JS on the server
- **Single-threaded**: Uses one main thread for JavaScript execution
- **Non-blocking I/O**: Asynchronous operations don't block the main thread
- **Event-driven**: Powered by an event loop
- **Cross-platform**: Runs on Windows, Linux, macOS

## 1.2 Node.js Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Your JavaScript Code                  │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│                    Node.js Bindings                     │
│  ┌────────────────────┐    ┌───────────────────────┐    │
│  │   V8 JavaScript    │    │   Node.js Core APIs   │    │
│  │      Engine        │    │  (fs, http, crypto)   │    │
│  └────────────────────┘    └───────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│                        libuv                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  Event Loop  │  │ Thread Pool  │  │   Async I/O  │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│              Operating System (OS APIs)                 │
└─────────────────────────────────────────────────────────┘
```

### Components Breakdown:

#### 1. **V8 Engine**
- Open-source JavaScript engine written in C++
- Compiles JavaScript to native machine code
- Created by Google for Chrome browser
- Handles memory allocation and garbage collection
- Just-In-Time (JIT) compilation for performance

#### 2. **libuv**
- Cross-platform C library
- Provides the Event Loop
- Handles asynchronous I/O operations
- Manages thread pool (default: 4 threads)
- Abstracts OS-specific operations

#### 3. **Node.js Bindings**
- Bridge between JavaScript and C++ libraries
- Exposes low-level functionality to JavaScript
- Wraps C++ functionality in JavaScript APIs

#### 4. **Core Modules**
- Written in JavaScript and C++
- Provide essential functionality (fs, http, etc.)
- No need to install separately

## 1.3 How Node.js Works

```
┌──────────────────────┐
│  JavaScript Code     │
│  (Single Thread)     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐     ┌─────────────────────┐
│                      │     │   Thread Pool       │
│    Event Loop        │────▶│   (4 threads)       │
│  (Non-blocking I/O)  │     │                     │
│                      │◀────│  - File I/O         │
└──────────┬───────────┘     │  - DNS Lookup       │
           │                 │  - Compression      │
           ▼                 │  - Crypto           │
┌──────────────────────┐     └─────────────────────┘
│   Callback Queue     │
│  (Completed Tasks)   │
└──────────────────────┘
```

### Execution Flow:

1. **Code Initialization**: Your script starts executing
2. **Synchronous Code**: Runs line by line on the main thread
3. **Async Operations**: Delegated to libuv
4. **Thread Pool**: Heavy operations handled by worker threads
5. **Event Loop**: Monitors completion of async operations
6. **Callbacks**: Executed when operations complete

## 1.4 Single-Threaded vs Multi-Threaded

### Traditional Multi-Threaded (e.g., Apache):
```
Request 1 ───▶ Thread 1 (blocked during I/O)
Request 2 ───▶ Thread 2 (blocked during I/O)
Request 3 ───▶ Thread 3 (blocked during I/O)
Request 4 ───▶ Thread 4 (blocked during I/O)
Request 5 ───▶ ⏳ Waiting... (no available threads)
```

### Node.js Single-Threaded:
```
Request 1 ───┐
Request 2 ───┼──▶ Event Loop ──▶ Delegates I/O ──▶ Thread Pool
Request 3 ───┤    (Non-blocking)                    (Background)
Request 4 ───┘                                            │
                                                          ▼
                                                    Callbacks
                                                     Executed
```

### Advantages:
- **No Thread Context Switching**: Faster execution
- **Lower Memory Footprint**: One thread vs many threads
- **No Deadlocks**: Single thread eliminates race conditions
- **High Concurrency**: Handle thousands of connections

### Limitations:
- **CPU-Intensive Tasks**: Block the event loop
- **Not for Heavy Computation**: Use Worker Threads or offload
- **Error Handling**: One unhandled error can crash the process

## 1.5 libuv and Thread Pool

### libuv Architecture:
```
┌───────────────────────────────────────────────┐
│                Event Loop                     │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐       │
│  │Timers│  │ I/O  │  │Check │  │Close │       │
│  └──────┘  └──────┘  └──────┘  └──────┘       │
└───────────────────────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌───────────────┐    ┌────────────────┐
│  Thread Pool  │    │  Async I/O     │
│  (UV_THREADPOOL)   │  (OS Level)    │
│  - File I/O   │    │  - Network     │
│  - DNS        │    │  - TCP/UDP     │
│  - Crypto     │    │  - Pipes       │
│  - Compression│    └────────────────┘
└───────────────┘
```

### Thread Pool Operations:
1. **File System Operations**: All fs methods except `fs.watch()`
2. **DNS Lookups**: `dns.lookup()`
3. **Crypto Operations**: `crypto.pbkdf2()`, `crypto.randomBytes()`
4. **Compression**: zlib operations

### Configuring Thread Pool:
```javascript
// Set thread pool size (must be done before any operations)
process.env.UV_THREADPOOL_SIZE = 8; // Default is 4
```

## 1.6 V8 Engine Deep Dive

### V8 Components:
```
┌──────────────────────────────────────┐
│          V8 Engine                   │
│  ┌────────────────────────────────┐  │
│  │       Parser                   │  │
│  │  (JavaScript → AST)            │  │
│  └────────────┬───────────────────┘  │
│               ▼                      │
│  ┌────────────────────────────────┐  │
│  │      Ignition                  │  │
│  │  (Interpreter - Bytecode)      │  │
│  └────────────┬───────────────────┘  │
│               ▼                      │
│  ┌────────────────────────────────┐  │
│  │      TurboFan                  │  │
│  │  (Optimizing Compiler)         │  │
│  └────────────┬───────────────────┘  │
│               ▼                      │
│  ┌────────────────────────────────┐  │
│  │    Machine Code                │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### Memory Management:
```
┌──────────────────────────────────────┐
│           V8 Heap                    │
│  ┌────────────────────────────────┐  │
│  │   New Space (Young Generation) │  │
│  │   - Short-lived objects        │  │
│  │   - 1-8 MB                     │  │
│  │   - Fast GC (Scavenge)         │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │   Old Space (Old Generation)   │  │
│  │   - Long-lived objects         │  │
│  │   - Larger memory              │  │
│  │   - Slower GC (Mark-Sweep)     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

## 1.7 Practical Tasks

### Task 1: Understanding Single-Threaded Behavior
```javascript
// task1.js
console.log('Start');

setTimeout(() => {
  console.log('Timeout 1');
}, 0);

setTimeout(() => {
  console.log('Timeout 2');
}, 0);

console.log('End');

// Question: What's the output order and why?
// Run: node task1.js
```

**Expected Output:**
```
Start
End
Timeout 1
Timeout 2
```

**Why?** Synchronous code runs first, then event loop processes callbacks.

### Task 2: Thread Pool Visualization
```javascript
// task2.js
const crypto = require('crypto');

const start = Date.now();

// This will use thread pool
crypto.pbkdf2('password', 'salt', 100000, 512, 'sha512', () => {
  console.log('1:', Date.now() - start);
});

crypto.pbkdf2('password', 'salt', 100000, 512, 'sha512', () => {
  console.log('2:', Date.now() - start);
});

crypto.pbkdf2('password', 'salt', 100000, 512, 'sha512', () => {
  console.log('3:', Date.now() - start);
});

crypto.pbkdf2('password', 'salt', 100000, 512, 'sha512', () => {
  console.log('4:', Date.now() - start);
});

crypto.pbkdf2('password', 'salt', 100000, 512, 'sha512', () => {
  console.log('5:', Date.now() - start);
});

// Question: Notice the timing difference between 1-4 and 5?
// This demonstrates the default thread pool size of 4
```

### Task 3: Blocking vs Non-Blocking
```javascript
// task3-blocking.js
const fs = require('fs');

console.log('Start');

// Blocking - stops execution until file is read
const data = fs.readFileSync('./large-file.txt', 'utf8');
console.log('File read complete');

console.log('End');
```

```javascript
// task3-non-blocking.js
const fs = require('fs');

console.log('Start');

// Non-blocking - continues execution immediately
fs.readFile('./large-file.txt', 'utf8', (err, data) => {
  if (err) throw err;
  console.log('File read complete');
});

console.log('End');

// Question: What's the output difference?
```

### Task 4: Checking Node.js Version and V8 Version
```javascript
// task4.js
console.log('Node.js version:', process.version);
console.log('V8 version:', process.versions.v8);
console.log('libuv version:', process.versions.uv);
console.log('Platform:', process.platform);
console.log('Architecture:', process.arch);
```

## 1.8 Interview Questions

### Basic Level:
1. **What is Node.js and how is it different from JavaScript?**
2. **Is Node.js single-threaded? Explain.**
3. **What is V8 engine?**
4. **What is libuv and what is its role?**

### Intermediate Level:
1. **Explain the thread pool in Node.js. When is it used?**
2. **How does Node.js handle thousands of concurrent connections with a single thread?**
3. **What operations use the thread pool vs OS async I/O?**
4. **How can you increase the thread pool size?**

### Advanced Level:
1. **Explain V8's garbage collection mechanisms.**
2. **When would you NOT use Node.js? What are its limitations?**
3. **How does Node.js achieve non-blocking I/O?**
4. **Explain the difference between blocking and non-blocking code with examples.**

## 1.9 Key Takeaways

- Node.js = V8 Engine + libuv + Core Modules
- Single-threaded for JavaScript execution, but uses thread pool for heavy operations
- Non-blocking I/O allows high concurrency
- Event-driven architecture processes callbacks asynchronously
- libuv provides the event loop and abstracts OS operations
- Not suitable for CPU-intensive tasks without Worker Threads
- Thread pool (default 4) handles fs, dns, crypto, compression operations

## Next Chapter Preview

In Chapter 2, we'll dive deep into the Event Loop - the heart of Node.js's asynchronous nature. You'll learn about event loop phases, microtasks vs macrotasks, and how to write efficient async code.

---

**Practice Makes Perfect**: Complete all tasks before moving to the next chapter. Understanding these fundamentals is crucial for advanced topics.
