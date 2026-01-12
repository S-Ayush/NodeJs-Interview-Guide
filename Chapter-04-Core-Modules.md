# Chapter 4: Core Modules & File System

## 4.1 Node.js Core Modules Overview

Core modules are built into Node.js and can be used without installation.

```
┌────────────────────────────────────────────┐
│         Node.js Core Modules               │
│                                            │
│  File System      Network        Utility   │
│  ├─ fs           ├─ http         ├─ util  │
│  ├─ path         ├─ https        ├─ url   │
│  └─ fs/promises  ├─ net          └─ querystring │
│                  └─ dns                    │
│                                            │
│  Process          Crypto         OS        │
│  ├─ child_process ├─ crypto      ├─ os    │
│  ├─ cluster       └─ zlib        └─ process│
│  └─ worker_threads                         │
│                                            │
│  Streams          Events         Others    │
│  ├─ stream       ├─ events       ├─ buffer│
│  └─ readline     └─ assert       └─ timers│
└────────────────────────────────────────────┘
```

### Importing Core Modules:

```javascript
// CommonJS (traditional)
const fs = require('fs');
const path = require('path');

// ES Modules (modern)
import fs from 'fs';
import path from 'path';

// Named imports
import { readFile, writeFile } from 'fs/promises';
```

## 4.2 File System (fs) Module

The fs module provides file system operations.

### File System Architecture:

```
┌────────────────────────────────────────────┐
│           fs Module                        │
│                                            │
│  ┌──────────────┐  ┌──────────────┐      │
│  │  Synchronous │  │ Asynchronous │      │
│  │   (Blocking) │  │ (Callback)   │      │
│  └──────────────┘  └──────────────┘      │
│                                            │
│  ┌──────────────────────────────────┐    │
│  │   fs/promises                     │    │
│  │   (Promise-based)                 │    │
│  └──────────────────────────────────┘    │
└────────────────────────────────────────────┘
```

### 4.2.1 Reading Files

```javascript
const fs = require('fs');
const fsPromises = require('fs/promises');

// 1. SYNCHRONOUS (Blocking - avoid in production)
try {
  const data = fs.readFileSync('file.txt', 'utf8');
  console.log(data);
} catch (err) {
  console.error(err);
}

// 2. ASYNCHRONOUS with Callbacks
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(data);
});

// 3. PROMISE-BASED (Recommended)
fsPromises.readFile('file.txt', 'utf8')
  .then(data => console.log(data))
  .catch(err => console.error(err));

// 4. ASYNC/AWAIT (Best)
async function readFile() {
  try {
    const data = await fsPromises.readFile('file.txt', 'utf8');
    console.log(data);
  } catch (err) {
    console.error(err);
  }
}
```

### 4.2.2 Writing Files

```javascript
const fsPromises = require('fs/promises');

// Write (overwrites existing file)
async function writeFile() {
  try {
    await fsPromises.writeFile('output.txt', 'Hello World', 'utf8');
    console.log('File written successfully');
  } catch (err) {
    console.error('Error writing file:', err);
  }
}

// Append to file
async function appendFile() {
  try {
    await fsPromises.appendFile('output.txt', '\nNew line', 'utf8');
    console.log('Content appended');
  } catch (err) {
    console.error('Error appending:', err);
  }
}

// Write with flags
async function writeWithFlags() {
  try {
    // 'w' = write (default), 'a' = append, 'r+' = read/write
    await fsPromises.writeFile('file.txt', 'Data', { flag: 'a' });
  } catch (err) {
    console.error(err);
  }
}
```

### 4.2.3 File Operations

```javascript
const fsPromises = require('fs/promises');
const fs = require('fs');

// Check if file exists
async function fileExists(filepath) {
  try {
    await fsPromises.access(filepath);
    return true;
  } catch {
    return false;
  }
}

// Get file stats
async function getFileInfo(filepath) {
  try {
    const stats = await fsPromises.stat(filepath);
    console.log({
      size: stats.size,
      isFile: stats.isFile(),
      isDirectory: stats.isDirectory(),
      created: stats.birthtime,
      modified: stats.mtime
    });
  } catch (err) {
    console.error(err);
  }
}

// Copy file
async function copyFile(source, destination) {
  try {
    await fsPromises.copyFile(source, destination);
    console.log('File copied');
  } catch (err) {
    console.error('Copy failed:', err);
  }
}

// Rename/Move file
async function renameFile(oldPath, newPath) {
  try {
    await fsPromises.rename(oldPath, newPath);
    console.log('File renamed');
  } catch (err) {
    console.error('Rename failed:', err);
  }
}

// Delete file
async function deleteFile(filepath) {
  try {
    await fsPromises.unlink(filepath);
    console.log('File deleted');
  } catch (err) {
    console.error('Delete failed:', err);
  }
}
```

### 4.2.4 Directory Operations

```javascript
const fsPromises = require('fs/promises');

// Create directory
async function createDir(dirpath) {
  try {
    // recursive: true creates parent directories if needed
    await fsPromises.mkdir(dirpath, { recursive: true });
    console.log('Directory created');
  } catch (err) {
    console.error(err);
  }
}

// Read directory contents
async function readDir(dirpath) {
  try {
    const files = await fsPromises.readdir(dirpath);
    console.log('Files:', files);

    // With file types
    const entries = await fsPromises.readdir(dirpath, { withFileTypes: true });
    entries.forEach(entry => {
      console.log(`${entry.name} - ${entry.isDirectory() ? 'DIR' : 'FILE'}`);
    });
  } catch (err) {
    console.error(err);
  }
}

// Remove directory
async function removeDir(dirpath) {
  try {
    // recursive: true removes directory and contents
    await fsPromises.rm(dirpath, { recursive: true, force: true });
    console.log('Directory removed');
  } catch (err) {
    console.error(err);
  }
}

// Read directory recursively
async function readDirRecursive(dirpath) {
  try {
    const files = await fsPromises.readdir(dirpath, {
      recursive: true,  // Node 18.17.0+
      withFileTypes: true
    });
    return files;
  } catch (err) {
    console.error(err);
  }
}
```

### 4.2.5 Watch for File Changes

```javascript
const fs = require('fs');
const fsPromises = require('fs/promises');

// Method 1: fs.watch (low-level, OS-dependent)
const watcher = fs.watch('./watched-dir', (eventType, filename) => {
  console.log(`Event: ${eventType}, File: ${filename}`);
});

// Stop watching
// watcher.close();

// Method 2: fs.watchFile (polling-based, consistent but slower)
fs.watchFile('file.txt', (curr, prev) => {
  console.log(`File changed: ${prev.mtime} -> ${curr.mtime}`);
});

// Method 3: FSWatcher (event emitter)
const watcher2 = fs.watch('./watched-dir');
watcher2.on('change', (eventType, filename) => {
  console.log(`Changed: ${filename}`);
});
watcher2.on('error', (err) => {
  console.error('Watcher error:', err);
});
```

## 4.3 Path Module

The path module provides utilities for working with file and directory paths.

```javascript
const path = require('path');

// Platform-specific separator
console.log(path.sep);        // '/' on Unix, '\\' on Windows
console.log(path.delimiter);  // ':' on Unix, ';' on Windows

// Join paths (handles separators correctly)
const fullPath = path.join('/users', 'john', 'documents', 'file.txt');
// Output: '/users/john/documents/file.txt'

// Resolve to absolute path
const absolute = path.resolve('docs', 'file.txt');
// Output: '/current/working/directory/docs/file.txt'

// Get directory name
const dirname = path.dirname('/users/john/file.txt');
// Output: '/users/john'

// Get base name (file name with extension)
const basename = path.basename('/users/john/file.txt');
// Output: 'file.txt'

// Get base name without extension
const nameOnly = path.basename('/users/john/file.txt', '.txt');
// Output: 'file'

// Get extension
const ext = path.extname('/users/john/file.txt');
// Output: '.txt'

// Parse path into components
const parsed = path.parse('/users/john/file.txt');
/* Output:
{
  root: '/',
  dir: '/users/john',
  base: 'file.txt',
  ext: '.txt',
  name: 'file'
}
*/

// Format path from components
const formatted = path.format({
  dir: '/users/john',
  base: 'file.txt'
});
// Output: '/users/john/file.txt'

// Normalize path (resolves '..' and '.')
const normalized = path.normalize('/users/john/../jane/./file.txt');
// Output: '/users/jane/file.txt'

// Check if path is absolute
console.log(path.isAbsolute('/users/john'));     // true
console.log(path.isAbsolute('docs/file.txt'));   // false

// Get relative path between two paths
const relative = path.relative('/users/john', '/users/jane/file.txt');
// Output: '../jane/file.txt'
```

### Path Module Diagram:

```
Path: /users/john/documents/report.pdf

┌─────────────────────────────────────────┐
│  path.parse()                           │
├─────────────────────────────────────────┤
│  root: '/'                              │
│  dir:  '/users/john/documents'          │
│  base: 'report.pdf'                     │
│  name: 'report'                         │
│  ext:  '.pdf'                           │
└─────────────────────────────────────────┘

Methods:
- path.dirname()  → '/users/john/documents'
- path.basename() → 'report.pdf'
- path.extname()  → '.pdf'
```

## 4.4 OS Module

The os module provides operating system-related utility methods.

```javascript
const os = require('os');

// System information
console.log('Platform:', os.platform());      // 'darwin', 'linux', 'win32'
console.log('Architecture:', os.arch());      // 'x64', 'arm64'
console.log('CPU cores:', os.cpus().length);
console.log('Total memory:', os.totalmem());
console.log('Free memory:', os.freemem());
console.log('Uptime:', os.uptime(), 'seconds');

// CPU information
const cpus = os.cpus();
cpus.forEach((cpu, index) => {
  console.log(`CPU ${index}:`, cpu.model);
});

// Network interfaces
const networkInterfaces = os.networkInterfaces();
console.log('Network interfaces:', Object.keys(networkInterfaces));

// User info
console.log('Username:', os.userInfo().username);
console.log('Home directory:', os.homedir());

// Temp directory
console.log('Temp directory:', os.tmpdir());

// Host information
console.log('Hostname:', os.hostname());

// Line ending for the platform
console.log('EOL:', JSON.stringify(os.EOL));  // '\n' or '\r\n'

// Load average (Unix only)
if (os.platform() !== 'win32') {
  console.log('Load average:', os.loadavg());
}
```

## 4.5 Util Module

The util module provides utility functions.

```javascript
const util = require('util');

// 1. Promisify - Convert callback to Promise
const fs = require('fs');
const readFile = util.promisify(fs.readFile);

async function read() {
  const data = await readFile('file.txt', 'utf8');
  console.log(data);
}

// 2. Callbackify - Convert Promise to callback (rare)
function promiseFunction() {
  return Promise.resolve('result');
}
const callbackVersion = util.callbackify(promiseFunction);
callbackVersion((err, result) => {
  console.log(result);
});

// 3. Format strings
const formatted = util.format('%s:%d', 'Server', 3000);
// Output: 'Server:3000'

// 4. Inspect objects (better than JSON.stringify)
const obj = { name: 'John', nested: { deep: true } };
console.log(util.inspect(obj, { depth: null, colors: true }));

// 5. Type checking
console.log(util.types.isDate(new Date()));          // true
console.log(util.types.isPromise(Promise.resolve())); // true
console.log(util.types.isRegExp(/abc/));              // true

// 6. Deprecation warnings
function oldFunction() {
  console.log('This is deprecated');
}
const deprecated = util.deprecate(oldFunction, 'oldFunction() is deprecated');

// 7. Inherits (for class inheritance in older Node)
function Parent() {}
function Child() {}
util.inherits(Child, Parent);

// 8. Debug logging
const debuglog = util.debuglog('myapp');
debuglog('This only shows when NODE_DEBUG=myapp');
```

## 4.6 Practical Tasks

### Task 1: File Processing Pipeline
```javascript
// task1.js - Read, process, and write file
const fsPromises = require('fs/promises');

async function processFile(inputFile, outputFile) {
  try {
    // Read file
    const content = await fsPromises.readFile(inputFile, 'utf8');

    // Process (convert to uppercase)
    const processed = content.toUpperCase();

    // Write result
    await fsPromises.writeFile(outputFile, processed, 'utf8');

    console.log('Processing complete');
  } catch (err) {
    console.error('Error:', err);
  }
}

processFile('input.txt', 'output.txt');
```

### Task 2: Directory Tree Listing
```javascript
// task2.js - List all files in directory tree
const fsPromises = require('fs/promises');
const path = require('path');

async function listFilesRecursive(dirPath, indent = '') {
  try {
    const entries = await fsPromises.readdir(dirPath, { withFileTypes: true });

    for (const entry of entries) {
      const fullPath = path.join(dirPath, entry.name);

      if (entry.isDirectory()) {
        console.log(`${indent}📁 ${entry.name}/`);
        await listFilesRecursive(fullPath, indent + '  ');
      } else {
        console.log(`${indent}📄 ${entry.name}`);
      }
    }
  } catch (err) {
    console.error('Error:', err);
  }
}

listFilesRecursive('./project');
```

### Task 3: File Backup Utility
```javascript
// task3.js - Create timestamped backup
const fsPromises = require('fs/promises');
const path = require('path');

async function createBackup(filePath) {
  try {
    // Check if file exists
    await fsPromises.access(filePath);

    // Parse path
    const parsed = path.parse(filePath);

    // Create backup filename with timestamp
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const backupName = `${parsed.name}_backup_${timestamp}${parsed.ext}`;
    const backupPath = path.join(parsed.dir, backupName);

    // Copy file
    await fsPromises.copyFile(filePath, backupPath);

    console.log(`Backup created: ${backupPath}`);
    return backupPath;
  } catch (err) {
    console.error('Backup failed:', err);
    throw err;
  }
}

createBackup('./important-file.txt');
```

### Task 4: System Information Reporter
```javascript
// task4.js - Generate system report
const os = require('os');
const fsPromises = require('fs/promises');

async function generateSystemReport() {
  const report = {
    timestamp: new Date().toISOString(),
    platform: os.platform(),
    architecture: os.arch(),
    hostname: os.hostname(),
    cpus: os.cpus().length,
    totalMemory: `${(os.totalmem() / 1024 / 1024 / 1024).toFixed(2)} GB`,
    freeMemory: `${(os.freemem() / 1024 / 1024 / 1024).toFixed(2)} GB`,
    uptime: `${(os.uptime() / 3600).toFixed(2)} hours`,
    userInfo: os.userInfo().username,
    nodeVersion: process.version
  };

  const reportText = JSON.stringify(report, null, 2);

  await fsPromises.writeFile('system-report.json', reportText);
  console.log('System report generated');
  console.log(reportText);
}

generateSystemReport();
```

### Task 5: File Watcher with Logging
```javascript
// task5.js - Watch directory and log changes
const fs = require('fs');
const fsPromises = require('fs/promises');
const path = require('path');

async function logChange(message) {
  const timestamp = new Date().toISOString();
  const logEntry = `[${timestamp}] ${message}\n`;
  await fsPromises.appendFile('file-changes.log', logEntry);
  console.log(logEntry.trim());
}

function watchDirectory(dirPath) {
  console.log(`Watching directory: ${dirPath}`);

  const watcher = fs.watch(dirPath, async (eventType, filename) => {
    if (filename) {
      const fullPath = path.join(dirPath, filename);
      const message = `${eventType.toUpperCase()}: ${fullPath}`;
      await logChange(message);
    }
  });

  watcher.on('error', (err) => {
    console.error('Watcher error:', err);
  });

  return watcher;
}

const watcher = watchDirectory('./watched-folder');

// Stop after 30 seconds
setTimeout(() => {
  watcher.close();
  console.log('Watcher stopped');
}, 30000);
```

## 4.7 Interview Questions

### Basic Level:
1. **What's the difference between fs.readFile() and fs.readFileSync()?**
2. **How do you create a directory in Node.js?**
3. **What does path.join() do and why use it?**
4. **Name 5 core Node.js modules.**

### Intermediate Level:
1. **How do you convert callback-based fs methods to Promises?**
2. **What's the difference between path.resolve() and path.join()?**
3. **How do you check if a file exists without throwing an error?**
4. **Explain fs.watch() vs fs.watchFile().**

### Advanced Level:
1. **How would you implement a file upload progress tracker?**
2. **What are the performance implications of readFileSync() in production?**
3. **How do you handle file operations in a clustered environment?**
4. **Implement a function to safely delete a directory recursively.**

## 4.8 Key Takeaways

- Use fs/promises for modern async file operations
- path module handles cross-platform path differences
- Always use path.join() for constructing file paths
- os module provides system information
- util.promisify() converts callbacks to Promises
- Avoid synchronous file operations in production
- Use fs.watch() for file system monitoring
- Handle errors properly in file operations

## Next Chapter Preview

In Chapter 5, we'll explore Streams and Buffers - essential for handling large files and real-time data efficiently. You'll learn about readable, writable, duplex, and transform streams.

---

**Practice Reminder**: Build a small file management utility using the concepts from this chapter to solidify your understanding.
