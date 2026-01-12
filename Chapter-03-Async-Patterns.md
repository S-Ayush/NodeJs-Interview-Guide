# Chapter 3: Asynchronous Programming Patterns

## 3.1 Evolution of Async Programming in Node.js

```
Callbacks (2009)
    │
    ▼
Promises (ES6/2015)
    │
    ▼
Async/Await (ES2017)
    │
    ▼
Top-level await (ES2022)
```

## 3.2 Callbacks - The Foundation

Callbacks are functions passed as arguments to be executed after an asynchronous operation completes.

### Error-First Callback Pattern

```javascript
// Standard Node.js callback pattern
function asyncOperation(param, callback) {
  // callback signature: (error, result)
  setTimeout(() => {
    if (!param) {
      callback(new Error('Parameter required'), null);
    } else {
      callback(null, `Processed: ${param}`);
    }
  }, 1000);
}

// Usage
asyncOperation('test', (err, result) => {
  if (err) {
    console.error('Error:', err.message);
    return;
  }
  console.log('Success:', result);
});
```

### Why Error-First?
```
Callback Parameters:
┌─────────────────────────────────┐
│ callback(error, result)         │
│          ▲       ▲             │
│          │       │             │
│    First param   Second param   │
│    (Error obj)   (Success data) │
└─────────────────────────────────┘

Benefits:
- Consistent error handling
- Easy to check for errors first
- null/undefined indicates success
```

### Callback Hell (Pyramid of Doom)

```javascript
// BAD - Nested callbacks
getData(function(a){
  getMoreData(a, function(b){
    getMoreData(b, function(c){
      getMoreData(c, function(d){
        getMoreData(d, function(e){
          // Finally do something with e
          console.log(e);
        });
      });
    });
  });
});

// Visualization:
//           getData
//              │
//              ▼
//          getMoreData
//              │
//              ▼
//          getMoreData
//              │
//              ▼
//          getMoreData   ◄── Hard to read
//              │              Hard to maintain
//              ▼              Hard to debug
//          getMoreData
//              │
//              ▼
//            result
```

### Solutions to Callback Hell:

#### 1. Named Functions
```javascript
// BETTER - Named functions
function handleE(e) {
  console.log(e);
}

function handleD(d) {
  getMoreData(d, handleE);
}

function handleC(c) {
  getMoreData(c, handleD);
}

function handleB(b) {
  getMoreData(b, handleC);
}

function handleA(a) {
  getMoreData(a, handleB);
}

getData(handleA);
```

#### 2. Promises (Better solution)
```javascript
// BEST - Using Promises
getData()
  .then(a => getMoreData(a))
  .then(b => getMoreData(b))
  .then(c => getMoreData(c))
  .then(d => getMoreData(d))
  .then(e => console.log(e))
  .catch(err => console.error(err));
```

## 3.3 Promises - The Modern Approach

A Promise represents the eventual completion (or failure) of an asynchronous operation.

### Promise States:

```
┌──────────────────────────────────────────┐
│          Promise Lifecycle               │
│                                          │
│  Pending ──────┐                        │
│                │                         │
│                ├──▶ Fulfilled (resolved) │
│                │                         │
│                └──▶ Rejected             │
│                                          │
│  Note: Once settled (fulfilled/rejected),│
│  a promise cannot change state           │
└──────────────────────────────────────────┘
```

### Creating Promises:

```javascript
// 1. Basic Promise
const myPromise = new Promise((resolve, reject) => {
  const success = true;

  setTimeout(() => {
    if (success) {
      resolve('Operation successful!');
    } else {
      reject(new Error('Operation failed!'));
    }
  }, 1000);
});

// 2. Using Promise
myPromise
  .then(result => {
    console.log(result);
    return 'Next step';
  })
  .then(result => {
    console.log(result);
  })
  .catch(error => {
    console.error('Error:', error);
  })
  .finally(() => {
    console.log('Cleanup');
  });
```

### Promise Methods:

#### Promise.all() - Wait for All
```javascript
// Executes in parallel, waits for all to complete
const promise1 = fetchUser(1);
const promise2 = fetchUser(2);
const promise3 = fetchUser(3);

Promise.all([promise1, promise2, promise3])
  .then(([user1, user2, user3]) => {
    console.log('All users:', user1, user2, user3);
  })
  .catch(err => {
    // If ANY promise rejects, catch is called
    console.error('One failed:', err);
  });

// Diagram:
// promise1 ────▶ (resolves)
// promise2 ────▶ (resolves)  ──▶ All succeed ──▶ then()
// promise3 ────▶ (resolves)

// promise1 ────▶ (resolves)
// promise2 ────▶ (REJECTS)   ──▶ One fails ──▶ catch()
// promise3 ────▶ (resolves)
```

#### Promise.allSettled() - Wait for All (No Fail)
```javascript
// ES2020 - Returns results of all promises
Promise.allSettled([promise1, promise2, promise3])
  .then(results => {
    results.forEach(result => {
      if (result.status === 'fulfilled') {
        console.log('Success:', result.value);
      } else {
        console.log('Failed:', result.reason);
      }
    });
  });

// Always resolves, never rejects
// Returns: [
//   { status: 'fulfilled', value: result },
//   { status: 'rejected', reason: error }
// ]
```

#### Promise.race() - First to Finish
```javascript
// Returns first promise that settles (resolve or reject)
Promise.race([
  fetchWithTimeout(5000),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error('Timeout')), 3000)
  )
])
  .then(result => console.log('First:', result))
  .catch(err => console.error('First failed:', err));

// Diagram:
// promise1 ─────────────▶ (slow)
// promise2 ────▶ (fast)  ──▶ Returns first ──▶ then()
// promise3 ───────▶ (medium)
```

#### Promise.any() - First Success
```javascript
// ES2021 - Returns first fulfilled promise
Promise.any([promise1, promise2, promise3])
  .then(result => {
    console.log('First success:', result);
  })
  .catch(err => {
    // Only if ALL promises reject (AggregateError)
    console.error('All failed:', err);
  });

// Diagram:
// promise1 ────▶ (rejects)
// promise2 ────▶ (resolves) ──▶ First success ──▶ then()
// promise3 ────▶ (pending...)
```

### Promise Chaining:

```javascript
// Each .then() returns a new promise
fetch('/api/user')
  .then(response => response.json())     // Return promise
  .then(user => fetch(`/api/posts/${user.id}`))  // Return promise
  .then(response => response.json())     // Return promise
  .then(posts => {
    console.log('User posts:', posts);
    return posts.length;  // Return value (wrapped in resolved promise)
  })
  .then(count => {
    console.log('Post count:', count);
  })
  .catch(err => {
    console.error('Any step failed:', err);
  });

// Chain flow:
// Step 1 ──▶ Step 2 ──▶ Step 3 ──▶ Step 4
//   │         │         │         │
//   └─────────┴─────────┴─────────┴──▶ catch (if any fail)
```

## 3.4 Async/Await - Syntactic Sugar

Async/await makes asynchronous code look and behave like synchronous code.

### Basic Syntax:

```javascript
// Function marked as async always returns a Promise
async function fetchUserData(userId) {
  try {
    const response = await fetch(`/api/users/${userId}`);
    const user = await response.json();
    return user;  // Wrapped in Promise.resolve()
  } catch (error) {
    console.error('Error fetching user:', error);
    throw error;  // Wrapped in Promise.reject()
  }
}

// Usage
fetchUserData(1)
  .then(user => console.log(user))
  .catch(err => console.error(err));

// Or with async/await
async function main() {
  try {
    const user = await fetchUserData(1);
    console.log(user);
  } catch (err) {
    console.error(err);
  }
}
```

### Comparison: Promise vs Async/Await

```javascript
// PROMISES
function getUserPosts(userId) {
  return fetch(`/api/users/${userId}`)
    .then(res => res.json())
    .then(user => fetch(`/api/posts?userId=${user.id}`))
    .then(res => res.json())
    .then(posts => {
      console.log('Posts:', posts);
      return posts;
    })
    .catch(err => {
      console.error('Error:', err);
      throw err;
    });
}

// ASYNC/AWAIT (Same logic, cleaner code)
async function getUserPosts(userId) {
  try {
    const userRes = await fetch(`/api/users/${userId}`);
    const user = await userRes.json();

    const postsRes = await fetch(`/api/posts?userId=${user.id}`);
    const posts = await postsRes.json();

    console.log('Posts:', posts);
    return posts;
  } catch (err) {
    console.error('Error:', err);
    throw err;
  }
}
```

### Sequential vs Parallel Execution:

```javascript
// SEQUENTIAL (Slow) - One after another
async function sequential() {
  const user1 = await fetchUser(1);  // Wait 1s
  const user2 = await fetchUser(2);  // Wait 1s
  const user3 = await fetchUser(3);  // Wait 1s
  return [user1, user2, user3];      // Total: 3s
}

// PARALLEL (Fast) - All at once
async function parallel() {
  const [user1, user2, user3] = await Promise.all([
    fetchUser(1),  // Start all together
    fetchUser(2),  // All running in parallel
    fetchUser(3)   // Total: 1s
  ]);
  return [user1, user2, user3];
}

// Visual:
// Sequential:
// fetchUser(1) ──▶ fetchUser(2) ──▶ fetchUser(3)  [3 seconds]

// Parallel:
// fetchUser(1) ────▶
// fetchUser(2) ────▶  [All complete in 1 second]
// fetchUser(3) ────▶
```

### Error Handling Patterns:

```javascript
// 1. Try-Catch (Recommended)
async function withTryCatch() {
  try {
    const result = await riskyOperation();
    return result;
  } catch (error) {
    console.error('Error:', error);
    throw error; // Re-throw if needed
  }
}

// 2. .catch() Method
async function withCatch() {
  const result = await riskyOperation().catch(err => {
    console.error('Error:', err);
    return null; // Provide fallback
  });
  return result;
}

// 3. Multiple Operations
async function multipleOperations() {
  try {
    const result1 = await operation1();
    const result2 = await operation2();
    const result3 = await operation3();
    return [result1, result2, result3];
  } catch (error) {
    // Catches error from ANY operation
    console.error('One operation failed:', error);
    throw error;
  }
}

// 4. Specific Error Handling
async function specificErrors() {
  try {
    const result = await operation();
    return result;
  } catch (error) {
    if (error.code === 'ENOENT') {
      console.error('File not found');
    } else if (error.code === 'EACCES') {
      console.error('Permission denied');
    } else {
      console.error('Unknown error:', error);
    }
    throw error;
  }
}
```

## 3.5 Converting Callbacks to Promises

### Using util.promisify:

```javascript
const util = require('util');
const fs = require('fs');

// Old callback style
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) throw err;
  console.log(data);
});

// Convert to Promise
const readFilePromise = util.promisify(fs.readFile);

// Use with async/await
async function readFile() {
  try {
    const data = await readFilePromise('file.txt', 'utf8');
    console.log(data);
  } catch (err) {
    console.error(err);
  }
}

// Or use fs.promises (built-in)
const fsPromises = require('fs').promises;

async function readFile2() {
  const data = await fsPromises.readFile('file.txt', 'utf8');
  console.log(data);
}
```

### Manual Promisification:

```javascript
// Callback function
function callbackStyle(param, callback) {
  setTimeout(() => {
    if (param) {
      callback(null, `Result: ${param}`);
    } else {
      callback(new Error('No param'));
    }
  }, 1000);
}

// Convert to Promise
function promisifiedFunction(param) {
  return new Promise((resolve, reject) => {
    callbackStyle(param, (err, result) => {
      if (err) {
        reject(err);
      } else {
        resolve(result);
      }
    });
  });
}

// Usage
async function main() {
  try {
    const result = await promisifiedFunction('test');
    console.log(result);
  } catch (err) {
    console.error(err);
  }
}
```

## 3.6 Practical Tasks

### Task 1: Converting Callback Hell to Async/Await
```javascript
// task1-before.js - Callback hell
const fs = require('fs');

fs.readFile('file1.txt', 'utf8', (err, data1) => {
  if (err) throw err;
  fs.readFile('file2.txt', 'utf8', (err, data2) => {
    if (err) throw err;
    fs.readFile('file3.txt', 'utf8', (err, data3) => {
      if (err) throw err;
      console.log(data1 + data2 + data3);
    });
  });
});

// task1-after.js - With async/await
const fs = require('fs').promises;

async function readFiles() {
  try {
    const data1 = await fs.readFile('file1.txt', 'utf8');
    const data2 = await fs.readFile('file2.txt', 'utf8');
    const data3 = await fs.readFile('file3.txt', 'utf8');
    console.log(data1 + data2 + data3);
  } catch (err) {
    console.error(err);
  }
}

readFiles();
```

### Task 2: Promise.all() Practice
```javascript
// task2.js
async function fetchMultipleUsers() {
  try {
    const userIds = [1, 2, 3, 4, 5];

    // Parallel fetching
    const users = await Promise.all(
      userIds.map(id => fetchUser(id))
    );

    console.log('All users:', users);
  } catch (err) {
    console.error('Failed to fetch users:', err);
  }
}

function fetchUser(id) {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve({ id, name: `User ${id}` });
    }, 1000);
  });
}

fetchMultipleUsers();
```

### Task 3: Error Handling Patterns
```javascript
// task3.js
async function robustDataFetch() {
  try {
    // Attempt primary source
    const data = await fetchFromPrimary();
    return data;
  } catch (primaryError) {
    console.warn('Primary failed, trying backup:', primaryError.message);

    try {
      // Attempt backup source
      const data = await fetchFromBackup();
      return data;
    } catch (backupError) {
      console.error('Both sources failed:', backupError.message);

      // Return cached data as last resort
      return await getCachedData();
    }
  }
}

function fetchFromPrimary() {
  return Promise.reject(new Error('Primary source down'));
}

function fetchFromBackup() {
  return Promise.resolve({ data: 'Backup data' });
}

function getCachedData() {
  return Promise.resolve({ data: 'Cached data' });
}

robustDataFetch().then(data => console.log('Final data:', data));
```

### Task 4: Race Condition with Timeout
```javascript
// task4.js
function fetchWithTimeout(url, timeout = 5000) {
  return Promise.race([
    fetch(url),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Request timeout')), timeout)
    )
  ]);
}

async function getData() {
  try {
    const response = await fetchWithTimeout('https://api.example.com/data', 3000);
    const data = await response.json();
    console.log(data);
  } catch (err) {
    if (err.message === 'Request timeout') {
      console.error('Request took too long');
    } else {
      console.error('Request failed:', err);
    }
  }
}
```

### Task 5: Retry Logic
```javascript
// task5.js
async function retry(fn, maxAttempts = 3, delay = 1000) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const result = await fn();
      return result;
    } catch (err) {
      if (attempt === maxAttempts) {
        throw err;
      }
      console.log(`Attempt ${attempt} failed, retrying in ${delay}ms...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Usage
async function unreliableOperation() {
  if (Math.random() > 0.7) {
    return 'Success!';
  }
  throw new Error('Random failure');
}

retry(() => unreliableOperation(), 5, 1000)
  .then(result => console.log('Final result:', result))
  .catch(err => console.error('All attempts failed:', err));
```

## 3.7 Common Mistakes & Best Practices

### Mistake 1: Forgetting await
```javascript
// WRONG - Returns Promise, not value
async function wrong() {
  const data = fetchData(); // Missing await!
  console.log(data); // Promise { <pending> }
}

// CORRECT
async function correct() {
  const data = await fetchData();
  console.log(data); // Actual data
}
```

### Mistake 2: Not Returning in Promise Chain
```javascript
// WRONG - Breaking the chain
fetchUser()
  .then(user => {
    fetchPosts(user.id); // Not returned!
  })
  .then(posts => {
    console.log(posts); // undefined!
  });

// CORRECT
fetchUser()
  .then(user => {
    return fetchPosts(user.id); // Return the promise
  })
  .then(posts => {
    console.log(posts); // Actual posts
  });
```

### Mistake 3: Sequential Instead of Parallel
```javascript
// WRONG - Slow (3 seconds total)
async function slow() {
  const user1 = await fetchUser(1); // 1s
  const user2 = await fetchUser(2); // 1s
  const user3 = await fetchUser(3); // 1s
}

// CORRECT - Fast (1 second total)
async function fast() {
  const [user1, user2, user3] = await Promise.all([
    fetchUser(1),
    fetchUser(2),
    fetchUser(3)
  ]); // All in parallel - 1s
}
```

### Mistake 4: Not Handling Errors
```javascript
// WRONG - Unhandled promise rejection
async function risky() {
  const data = await fetchData(); // Might fail
  return data;
}

// CORRECT
async function safe() {
  try {
    const data = await fetchData();
    return data;
  } catch (err) {
    console.error('Error:', err);
    throw err; // Or handle appropriately
  }
}
```

## 3.8 Interview Questions

### Basic Level:
1. **What is a callback? What is callback hell?**
2. **What are the three states of a Promise?**
3. **What does async/await do?**
4. **How do you handle errors in async/await?**

### Intermediate Level:
1. **What's the difference between Promise.all() and Promise.allSettled()?**
2. **How do you convert a callback function to a Promise?**
3. **Explain Promise chaining with an example.**
4. **What happens if you don't await an async function?**

### Advanced Level:
1. **How would you implement Promise.all() from scratch?**
2. **What's the difference between sequential and parallel async operations?**
3. **How do you implement retry logic with exponential backoff?**
4. **Explain how async/await works under the hood.**

### Coding Challenges:
```javascript
// Challenge 1: What's the output?
async function challenge1() {
  console.log(1);
  await Promise.resolve(2).then(console.log);
  console.log(3);
}
challenge1();
console.log(4);

// Challenge 2: Fix this code
async function buggy() {
  const users = [1, 2, 3];
  users.forEach(async (id) => {
    const user = await fetchUser(id);
    console.log(user);
  });
  console.log('Done'); // Prints before users!
}
```

## 3.9 Key Takeaways

- Callbacks are the foundation but lead to callback hell
- Promises provide better error handling and chaining
- Async/await makes async code look synchronous
- Always handle errors with try/catch or .catch()
- Use Promise.all() for parallel operations
- Use util.promisify() to convert callbacks to Promises
- Understand sequential vs parallel execution
- Return promises in .then() chains
- Don't forget await keyword in async functions

## Next Chapter Preview

In Chapter 4, we'll explore Node.js Core Modules including fs, path, os, and util. You'll learn how to work with the file system, handle paths across platforms, and use essential built-in utilities.

---

**Practice Tip**: Refactor old callback-based code to Promises and then to async/await. This practice solidifies understanding of all three patterns.
