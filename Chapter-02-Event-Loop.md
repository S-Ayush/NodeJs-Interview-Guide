# Chapter 2: Event-Driven Architecture & Event Loop

## 2.1 Event-Driven Architecture

Event-driven architecture is a design pattern where the flow of the program is determined by events. In Node.js, everything revolves around events and callbacks.

```
┌──────────────────────────────────────────────────┐
│           Event-Driven Architecture              │
│                                                  │
│  Event Occurs  ──▶  Event Emitter  ──▶  Listener │
│                                           │      │
│                                           ▼      │
│                                      Callback    │
│                                      Executed    │
└──────────────────────────────────────────────────┘
```

### Core Concepts:
1. **Event Emitter**: Object that emits events
2. **Event Listener**: Function that responds to events
3. **Event Loop**: Mechanism that processes events
4. **Callback Queue**: Stores callbacks waiting to execute

## 2.2 The Event Loop - Heart of Node.js

The Event Loop is what allows Node.js to perform non-blocking I/O operations despite JavaScript being single-threaded.

### Event Loop Phases:

```
   ┌───────────────────────────┐
┌─▶│           timers          │  setTimeout(), setInterval()
│  └─────────────┬─────────────┘
│                │
│  ┌─────────────▼─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred
│  └─────────────┬─────────────┘
│                │
│  ┌─────────────▼─────────────┐
│  │       idle, prepare       │  Internal use only
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────▼─────────────┐      │   incoming:   │
│  │           poll            │◀─────┤  connections, │
│  │                           │      │   data, etc.  │
│  └─────────────┬─────────────┘      └───────────────┘
│                │
│  ┌─────────────▼─────────────┐
│  │           check           │  setImmediate()
│  └─────────────┬─────────────┘
│                │
│  ┌─────────────▼─────────────┐
│  │      close callbacks      │  socket.on('close', ...)
│  └─────────────┬─────────────┘
│                │
└────────────────┘
```

### Phase-by-Phase Breakdown:

#### 1. **Timers Phase**
- Executes callbacks scheduled by `setTimeout()` and `setInterval()`
- Timers specify the threshold after which a callback may execute
- Not guaranteed to run exactly at that time

```javascript
setTimeout(() => {
  console.log('Timer executed');
}, 1000);
```

#### 2. **Pending Callbacks Phase**
- Executes I/O callbacks deferred to the next loop iteration
- Handles some system operations (e.g., TCP errors)

#### 3. **Idle, Prepare Phase**
- Used internally by Node.js
- You don't interact with this phase

#### 4. **Poll Phase** (Most Important)
- Retrieves new I/O events
- Executes I/O related callbacks (except close, timers, setImmediate)
- Will block here when appropriate

**Poll Phase Logic:**
```
Is the poll queue empty?
  ├─ YES ──▶ Are there setImmediate() callbacks?
  │            ├─ YES ──▶ Go to check phase
  │            └─ NO ───▶ Are there timers ready?
  │                        ├─ YES ──▶ Go to timers phase
  │                        └─ NO ───▶ Wait for callbacks
  │
  └─ NO ───▶ Execute callbacks until queue empty or limit reached
```

#### 5. **Check Phase**
- Executes `setImmediate()` callbacks
- Allows code to execute immediately after poll phase

```javascript
setImmediate(() => {
  console.log('Immediate executed');
});
```

#### 6. **Close Callbacks Phase**
- Executes close event callbacks
- Example: `socket.on('close', ...)`

## 2.3 Microtasks vs Macrotasks

This is a CRITICAL interview topic. Understanding the difference is essential.

```
┌─────────────────────────────────────────────┐
│          Task Queue Priority                │
│                                             │
│  1. Microtasks (HIGHEST PRIORITY)           │
│     ├─ process.nextTick()                   │
│     └─ Promise callbacks (.then, .catch)    │
│                                             │
│  2. Macrotasks (LOWER PRIORITY)             │
│     ├─ setTimeout()                         │
│     ├─ setInterval()                        │
│     ├─ setImmediate()                       │
│     └─ I/O operations                       │
└─────────────────────────────────────────────┘
```

### Execution Order:
```
1. Execute synchronous code
2. Execute ALL microtasks (process.nextTick queue first, then Promise queue)
3. Execute ONE macrotask
4. Execute ALL microtasks again
5. Repeat steps 3-4
```

### Visual Flow:
```
┌──────────────────────────────────────────────────┐
│  Synchronous Code                                │
└───────────────┬──────────────────────────────────┘
                ▼
┌──────────────────────────────────────────────────┐
│  Microtask Queue                                 │
│  1. All process.nextTick() callbacks             │
│  2. All Promise callbacks                        │
└───────────────┬──────────────────────────────────┘
                ▼
┌──────────────────────────────────────────────────┐
│  Macrotask (ONE from queue)                      │
└───────────────┬──────────────────────────────────┘
                ▼
┌──────────────────────────────────────────────────┐
│  Check for Microtasks again                      │
│  (Execute ALL before next macrotask)             │
└───────────────┬──────────────────────────────────┘
                │
                └──────────────┐
                               ▼
                          Repeat
```

## 2.4 process.nextTick() vs setImmediate()

### process.nextTick()
- Executed after the current operation completes
- Before the event loop continues
- Highest priority in microtask queue

### setImmediate()
- Executed in the check phase of the event loop
- Macrotask
- Lower priority than nextTick

```javascript
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');

// Output:
// sync
// nextTick
// immediate
```

### Comparison Diagram:
```
Current Operation Complete
         │
         ▼
┌─────────────────────┐
│ process.nextTick()  │ ◀── Executes HERE (microtask)
└─────────────────────┘
         │
         ▼
┌─────────────────────┐
│   Event Loop        │
│   continues...      │
│   ┌──────────┐      │
│   │ Check    │      │
│   │ Phase    │      │ ◀── setImmediate() executes HERE
│   └──────────┘      │
└─────────────────────┘
```

## 2.5 Event Emitters

The EventEmitter class is the foundation of event-driven programming in Node.js.

```javascript
const EventEmitter = require('events');

// Create an emitter
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();

// Register listener
myEmitter.on('event', (data) => {
  console.log('Event occurred with data:', data);
});

// Emit event
myEmitter.emit('event', { name: 'test' });
```

### EventEmitter Architecture:
```
┌──────────────────────────────────────────┐
│         EventEmitter Instance            │
│                                          │
│  Event: 'data'                           │
│  ├─ Listener 1 ─────────┐                │
│  ├─ Listener 2 ─────────┼──────▶ Fire    │
│  └─ Listener 3 ─────────┘       All      │
│                                          │
│  Event: 'error'                          │
│  └─ Listener 1 ──────────▶ Fire          │
└──────────────────────────────────────────┘
```

### Common EventEmitter Methods:

```javascript
const emitter = new EventEmitter();

// 1. on() - Add listener
emitter.on('event', listener);

// 2. once() - Add one-time listener
emitter.once('event', listener);

// 3. emit() - Trigger event
emitter.emit('event', arg1, arg2);

// 4. off() / removeListener() - Remove listener
emitter.off('event', listener);

// 5. removeAllListeners() - Remove all
emitter.removeAllListeners('event');

// 6. listeners() - Get all listeners
const listeners = emitter.listeners('event');

// 7. listenerCount() - Count listeners
const count = emitter.listenerCount('event');
```

### Error Handling in EventEmitters:
```javascript
const emitter = new EventEmitter();

// ALWAYS handle error events
emitter.on('error', (err) => {
  console.error('Error occurred:', err);
});

// Without error listener, this crashes the process
emitter.emit('error', new Error('Something went wrong'));
```

## 2.6 Practical Tasks

### Task 1: Understanding Event Loop Order
```javascript
// task1.js
console.log('1: Sync');

setTimeout(() => console.log('2: setTimeout'), 0);

setImmediate(() => console.log('3: setImmediate'));

process.nextTick(() => console.log('4: nextTick'));

Promise.resolve().then(() => console.log('5: Promise'));

console.log('6: Sync');

// Question: What's the output order?
// Answer: 1, 6, 4, 5, 2/3, 3/2 (2 and 3 may vary)
```

### Task 2: Microtask vs Macrotask
```javascript
// task2.js
console.log('Start');

setTimeout(() => {
  console.log('Timeout 1');
  Promise.resolve().then(() => console.log('Promise in Timeout'));
}, 0);

Promise.resolve().then(() => {
  console.log('Promise 1');
  process.nextTick(() => console.log('NextTick in Promise'));
});

process.nextTick(() => {
  console.log('NextTick 1');
});

console.log('End');

// Analyze the output and explain why
```

### Task 3: EventEmitter Custom Class
```javascript
// task3.js
const EventEmitter = require('events');

class DataProcessor extends EventEmitter {
  processData(data) {
    console.log('Processing data...');

    // Simulate async processing
    setTimeout(() => {
      if (!data) {
        this.emit('error', new Error('No data provided'));
        return;
      }

      const result = data.toUpperCase();
      this.emit('processed', result);
    }, 1000);
  }
}

const processor = new DataProcessor();

processor.on('processed', (result) => {
  console.log('Result:', result);
});

processor.on('error', (err) => {
  console.error('Error:', err.message);
});

processor.processData('hello world');
```

### Task 4: setImmediate vs setTimeout Order
```javascript
// task4.js
// In I/O cycle, setImmediate is always executed first

const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);

  setImmediate(() => {
    console.log('immediate');
  });
});

// Output: immediate, timeout (always in this order inside I/O)
```

### Task 5: Event Loop Blocking
```javascript
// task5-blocking.js
setTimeout(() => console.log('Timeout should run'), 100);

// This blocks the event loop
const start = Date.now();
while (Date.now() - start < 2000) {
  // Busy waiting - blocks everything
}

console.log('Finished blocking');

// Notice: Timeout runs late because event loop was blocked
```

### Task 6: Multiple Listeners
```javascript
// task6.js
const EventEmitter = require('events');
const emitter = new EventEmitter();

emitter.on('data', (msg) => {
  console.log('Listener 1:', msg);
});

emitter.on('data', (msg) => {
  console.log('Listener 2:', msg);
});

emitter.once('data', (msg) => {
  console.log('Listener 3 (once):', msg);
});

console.log('Listener count:', emitter.listenerCount('data'));

emitter.emit('data', 'First emit');
console.log('---');
emitter.emit('data', 'Second emit');

// Notice: Listener 3 only runs once
```

## 2.7 Real-World Scenarios

### Scenario 1: HTTP Server Events
```javascript
const http = require('http');

const server = http.createServer();

// HTTP server is an EventEmitter
server.on('request', (req, res) => {
  console.log('Request received');
  res.end('Hello World');
});

server.on('connection', (socket) => {
  console.log('New connection');
});

server.on('close', () => {
  console.log('Server closed');
});

server.listen(3000);
```

### Scenario 2: Stream Events
```javascript
const fs = require('fs');

const readStream = fs.createReadStream('./large-file.txt');

// Streams are EventEmitters
readStream.on('data', (chunk) => {
  console.log('Received chunk:', chunk.length);
});

readStream.on('end', () => {
  console.log('Finished reading');
});

readStream.on('error', (err) => {
  console.error('Error:', err);
});
```

## 2.8 Common Pitfalls

### 1. Not Handling Errors
```javascript
// BAD - Will crash the process
const emitter = new EventEmitter();
emitter.emit('error', new Error('Oops'));

// GOOD - Always handle errors
emitter.on('error', (err) => {
  console.error('Handled error:', err);
});
```

### 2. Memory Leaks with Listeners
```javascript
// BAD - Creating listeners in a loop
for (let i = 0; i < 1000; i++) {
  emitter.on('event', () => {});
}
// Warning: MaxListenersExceededWarning

// GOOD - Use once() or removeListener()
emitter.once('event', () => {});
```

### 3. Blocking the Event Loop
```javascript
// BAD - Synchronous heavy computation
app.get('/compute', (req, res) => {
  let result = 0;
  for (let i = 0; i < 1e9; i++) {
    result += i;
  }
  res.send(result);
});

// GOOD - Use Worker Threads or offload
const { Worker } = require('worker_threads');
app.get('/compute', (req, res) => {
  const worker = new Worker('./compute.js');
  worker.on('message', (result) => res.send(result));
});
```

## 2.9 Interview Questions

### Basic Level:
1. **What is the Event Loop?**
2. **What are the phases of the Event Loop?**
3. **What is an EventEmitter?**
4. **What's the difference between on() and once()?**

### Intermediate Level:
1. **Explain microtasks vs macrotasks with examples.**
2. **What's the difference between process.nextTick() and setImmediate()?**
3. **In what order do these execute: setTimeout, Promise, process.nextTick?**
4. **How does Node.js handle concurrent requests with a single thread?**

### Advanced Level:
1. **Explain the poll phase and when the event loop blocks there.**
2. **Why does setImmediate() execute before setTimeout() inside an I/O callback?**
3. **How can you prevent blocking the event loop in CPU-intensive operations?**
4. **Explain how memory leaks can occur with EventEmitters.**

### Coding Challenges:
1. **Predict the output:**
```javascript
setTimeout(() => console.log('1'), 0);
Promise.resolve().then(() => console.log('2'));
process.nextTick(() => console.log('3'));
console.log('4');
```

2. **What happens here?**
```javascript
const emitter = new EventEmitter();
emitter.emit('event', 'data');
emitter.on('event', (data) => console.log(data));
// Will it print? Why or why not?
```

## 2.10 Key Takeaways

- The Event Loop has 6 phases: timers, pending callbacks, idle/prepare, poll, check, close
- Microtasks (nextTick, Promises) have higher priority than macrotasks
- process.nextTick() executes before all other asynchronous code
- EventEmitter is the foundation of Node.js's event-driven architecture
- Always handle 'error' events to prevent crashes
- Don't block the event loop with synchronous heavy operations
- setImmediate() is designed to execute after the poll phase
- Understanding execution order is crucial for writing correct async code

## Next Chapter Preview

In Chapter 3, we'll explore Asynchronous Programming Patterns in depth - from callbacks to Promises to async/await. You'll learn best practices, error handling, and how to avoid callback hell.

---

**Pro Tip**: Practice with the tasks above until you can predict outputs without running the code. This level of understanding is what interviewers look for.
