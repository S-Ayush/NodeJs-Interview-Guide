# Chapter 5: Streams & Buffers

## 5.1 What are Buffers?

Buffers are temporary storage spots for data being transferred. They handle binary data directly.

```
┌──────────────────────────────────────────┐
│         Buffer Concept                    │
│                                           │
│  Source ──▶ Buffer ──▶ Destination      │
│  (Data)     (Temp)     (Process)         │
│                                           │
│  Example: Reading large file              │
│  File ──▶ Buffer ──▶ Memory              │
└──────────────────────────────────────────┘
```

### Why Buffers?
- JavaScript traditionally didn't handle binary data well
- Buffers allow working with binary streams (files, network, etc.)
- More memory-efficient than strings for binary data
- Direct memory access without V8 heap

### Buffer Representation:

```
Buffer: <Buffer 48 65 6c 6c 6f>
         │      │  │  │  │  │
         │      H  e  l  l  o
         │
         Hexadecimal representation
```

## 5.2 Working with Buffers

### Creating Buffers:

```javascript
// 1. From string
const buf1 = Buffer.from('Hello', 'utf8');
console.log(buf1);  // <Buffer 48 65 6c 6c 6f>

// 2. Allocate empty buffer (filled with zeros)
const buf2 = Buffer.alloc(10);  // Safe, filled with 0x00
console.log(buf2);  // <Buffer 00 00 00 00 00 00 00 00 00 00>

// 3. Allocate uninitialized buffer (faster but unsafe)
const buf3 = Buffer.allocUnsafe(10);  // Contains old memory data
buf3.fill(0);  // Manually clear it

// 4. From array
const buf4 = Buffer.from([1, 2, 3, 4, 5]);
console.log(buf4);  // <Buffer 01 02 03 04 05>

// 5. From another buffer
const buf5 = Buffer.from(buf1);
```

### Buffer Operations:

```javascript
const buf = Buffer.from('Hello World');

// 1. Length
console.log(buf.length);  // 11 (bytes, not characters)

// 2. Convert to string
console.log(buf.toString());          // 'Hello World'
console.log(buf.toString('hex'));     // '48656c6c6f20576f726c64'
console.log(buf.toString('base64'));  // 'SGVsbG8gV29ybGQ='

// 3. Access individual bytes
console.log(buf[0]);  // 72 (ASCII code for 'H')

// 4. Write to buffer
const writeBuf = Buffer.alloc(10);
writeBuf.write('Hello');
console.log(writeBuf.toString());  // 'Hello'

// 5. Compare buffers
const buf1 = Buffer.from('ABC');
const buf2 = Buffer.from('ABD');
console.log(buf1.compare(buf2));  // -1 (buf1 comes before buf2)
console.log(Buffer.compare(buf1, buf2));  // -1

// 6. Concatenate buffers
const buf3 = Buffer.from('Hello');
const buf4 = Buffer.from(' World');
const result = Buffer.concat([buf3, buf4]);
console.log(result.toString());  // 'Hello World'

// 7. Slice buffer (creates a view, not a copy)
const sliced = buf.slice(0, 5);
console.log(sliced.toString());  // 'Hello'

// 8. Copy buffer
const copyBuf = Buffer.alloc(5);
buf.copy(copyBuf, 0, 0, 5);
console.log(copyBuf.toString());  // 'Hello'

// 9. Fill buffer
const fillBuf = Buffer.alloc(10);
fillBuf.fill('ab');
console.log(fillBuf.toString());  // 'ababababab'
```

### Buffer Encoding:

```javascript
// Supported encodings:
// utf8, utf16le, latin1, base64, hex, ascii, binary, ucs2

const text = 'Hello';

const utf8Buf = Buffer.from(text, 'utf8');
const hexBuf = Buffer.from(text, 'hex');  // Treats as hex string
const base64Buf = Buffer.from(text, 'base64');

console.log(utf8Buf.toString('hex'));     // '48656c6c6f'
console.log(utf8Buf.toString('base64'));  // 'SGVsbG8='
```

## 5.3 What are Streams?

Streams are collections of data that might not be available all at once. They allow processing data piece by piece.

```
┌────────────────────────────────────────────┐
│          Stream vs Regular Read            │
│                                            │
│  Regular Read:                             │
│  ┌─────────────────┐                      │
│  │   Wait for      │                      │
│  │   entire file   │ ──▶ Process all     │
│  └─────────────────┘                      │
│  Memory: 100% file size                    │
│                                            │
│  Stream:                                   │
│  ┌────┐  ┌────┐  ┌────┐  ┌────┐         │
│  │Chunk1│─▶│Chunk2│─▶│Chunk3│─▶│Chunk4│ │
│  └────┘  └────┘  └────┘  └────┘         │
│  Memory: Only 1 chunk at a time            │
└────────────────────────────────────────────┘
```

### Benefits of Streams:
1. **Memory Efficiency**: Process large files without loading into memory
2. **Time Efficiency**: Start processing immediately
3. **Composability**: Pipe streams together
4. **Backpressure**: Automatic flow control

## 5.4 Types of Streams

```
┌────────────────────────────────────────────┐
│         Stream Types                       │
│                                            │
│  1. Readable   ──▶  Read data              │
│     (Source)        fs.createReadStream    │
│                                            │
│  2. Writable   ──▶  Write data             │
│     (Destination)   fs.createWriteStream   │
│                                            │
│  3. Duplex     ──▶  Read & Write           │
│     (Both ways)     TCP socket             │
│                                            │
│  4. Transform  ──▶  Modify data            │
│     (Process)       zlib, crypto           │
└────────────────────────────────────────────┘
```

### Visual Flow:

```
Readable Stream
┌──────────┐     data event
│  Source  │ ───────────────▶ Consumer
└──────────┘

Writable Stream
            write method
Provider ───────────────▶ ┌─────────────┐
                          │ Destination │
                          └─────────────┘

Duplex Stream
         write              read
Input ──────────▶ ┌──────┐ ──────────▶ Output
                  │Duplex│
Output ◀────────── └──────┘ ◀────────── Input

Transform Stream
         write                          read
Input ──────────▶ ┌───────────┐ ──────────▶ Output
                  │ Transform │  (modified)
                  │ (Process) │
                  └───────────┘
```

## 5.5 Readable Streams

### Creating Readable Streams:

```javascript
const fs = require('fs');
const { Readable } = require('stream');

// 1. File read stream
const readStream = fs.createReadStream('large-file.txt', {
  encoding: 'utf8',
  highWaterMark: 64 * 1024  // 64KB chunks (default 16KB)
});

// 2. Custom readable stream
class MyReadable extends Readable {
  constructor(options) {
    super(options);
    this.counter = 0;
  }

  _read() {
    if (this.counter < 5) {
      this.push(`Data chunk ${this.counter}\n`);
      this.counter++;
    } else {
      this.push(null);  // Signal end of stream
    }
  }
}

const myStream = new MyReadable();
```

### Reading from Streams:

```javascript
const fs = require('fs');

// Method 1: Data event (flowing mode)
const readStream1 = fs.createReadStream('file.txt');
readStream1.on('data', (chunk) => {
  console.log('Received chunk:', chunk.length, 'bytes');
});
readStream1.on('end', () => {
  console.log('No more data');
});
readStream1.on('error', (err) => {
  console.error('Error:', err);
});

// Method 2: Readable event (paused mode)
const readStream2 = fs.createReadStream('file.txt');
readStream2.on('readable', () => {
  let chunk;
  while ((chunk = readStream2.read()) !== null) {
    console.log('Received chunk:', chunk.length, 'bytes');
  }
});

// Method 3: Async iterator (modern, clean)
const readStream3 = fs.createReadStream('file.txt');
(async () => {
  for await (const chunk of readStream3) {
    console.log('Received chunk:', chunk.length, 'bytes');
  }
})();

// Method 4: Pipeline (recommended)
const { pipeline } = require('stream/promises');
await pipeline(
  fs.createReadStream('file.txt'),
  async function* (source) {
    for await (const chunk of source) {
      console.log('Processing chunk:', chunk.length);
      yield chunk;
    }
  },
  fs.createWriteStream('output.txt')
);
```

## 5.6 Writable Streams

### Creating Writable Streams:

```javascript
const fs = require('fs');
const { Writable } = require('stream');

// 1. File write stream
const writeStream = fs.createWriteStream('output.txt', {
  encoding: 'utf8',
  flags: 'a'  // 'w' for write, 'a' for append
});

// 2. Custom writable stream
class MyWritable extends Writable {
  _write(chunk, encoding, callback) {
    console.log('Writing:', chunk.toString());
    callback();  // Signal completion
  }

  _final(callback) {
    console.log('Stream ending');
    callback();
  }
}

const myWriteStream = new MyWritable();
```

### Writing to Streams:

```javascript
const fs = require('fs');
const writeStream = fs.createWriteStream('output.txt');

// Write data
writeStream.write('Line 1\n');
writeStream.write('Line 2\n');
writeStream.write('Line 3\n');

// End stream
writeStream.end('Final line\n');

// Handle events
writeStream.on('finish', () => {
  console.log('All data written');
});

writeStream.on('error', (err) => {
  console.error('Write error:', err);
});

// Handle backpressure
function writeMillionRecords() {
  let i = 0;

  function write() {
    let ok = true;
    while (i < 1000000 && ok) {
      ok = writeStream.write(`Record ${i}\n`);
      i++;
    }

    if (i < 1000000) {
      // Buffer is full, wait for drain event
      writeStream.once('drain', write);
    } else {
      writeStream.end();
    }
  }

  write();
}
```

## 5.7 Piping Streams

Piping connects a readable stream to a writable stream automatically.

```
Source (Readable) ──pipe()──▶ Destination (Writable)
```

### Basic Piping:

```javascript
const fs = require('fs');

// Simple pipe
const readStream = fs.createReadStream('input.txt');
const writeStream = fs.createWriteStream('output.txt');

readStream.pipe(writeStream);

writeStream.on('finish', () => {
  console.log('Copy complete');
});

// Chaining pipes (transform streams)
const zlib = require('zlib');

fs.createReadStream('input.txt')
  .pipe(zlib.createGzip())  // Compress
  .pipe(fs.createWriteStream('input.txt.gz'));

// Multiple destinations
const readStream2 = fs.createReadStream('input.txt');
const writeStream1 = fs.createWriteStream('output1.txt');
const writeStream2 = fs.createWriteStream('output2.txt');

readStream2.pipe(writeStream1);
readStream2.pipe(writeStream2);
```

### Modern Pipeline (with error handling):

```javascript
const { pipeline } = require('stream/promises');
const fs = require('fs');
const zlib = require('zlib');

// With error handling
async function compressFile(input, output) {
  try {
    await pipeline(
      fs.createReadStream(input),
      zlib.createGzip(),
      fs.createWriteStream(output)
    );
    console.log('Pipeline successful');
  } catch (err) {
    console.error('Pipeline failed:', err);
  }
}

compressFile('large-file.txt', 'large-file.txt.gz');
```

## 5.8 Transform Streams

Transform streams modify data as it passes through.

```javascript
const { Transform } = require('stream');
const fs = require('fs');

// 1. Uppercase transform
class UpperCaseTransform extends Transform {
  _transform(chunk, encoding, callback) {
    const upperChunk = chunk.toString().toUpperCase();
    this.push(upperChunk);
    callback();
  }
}

// Usage
fs.createReadStream('input.txt')
  .pipe(new UpperCaseTransform())
  .pipe(fs.createWriteStream('output.txt'));

// 2. CSV to JSON transform
class CsvToJsonTransform extends Transform {
  constructor(options) {
    super(options);
    this.isFirstLine = true;
    this.headers = [];
  }

  _transform(chunk, encoding, callback) {
    const lines = chunk.toString().split('\n');

    for (const line of lines) {
      if (this.isFirstLine) {
        this.headers = line.split(',');
        this.isFirstLine = false;
      } else if (line.trim()) {
        const values = line.split(',');
        const obj = {};
        this.headers.forEach((header, i) => {
          obj[header.trim()] = values[i]?.trim();
        });
        this.push(JSON.stringify(obj) + '\n');
      }
    }
    callback();
  }
}

// 3. Built-in transforms
const zlib = require('zlib');
const crypto = require('crypto');

// Compression
fs.createReadStream('file.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('file.txt.gz'));

// Encryption
fs.createReadStream('secret.txt')
  .pipe(crypto.createCipheriv('aes-256-cbc', key, iv))
  .pipe(fs.createWriteStream('secret.enc'));
```

## 5.9 Backpressure

Backpressure occurs when the writable stream can't keep up with the readable stream.

```
┌────────────────────────────────────────┐
│        Backpressure                     │
│                                         │
│  Fast Reader ──▶  Slow Writer          │
│      ┌───┐         ┌───┐               │
│      │ ▶ │ ═════▶  │ ▶ │               │
│      └───┘  Buffer └───┘               │
│              Full!                      │
│                                         │
│  Solution: Pause reading until          │
│            writer is ready              │
└────────────────────────────────────────┘
```

### Handling Backpressure:

```javascript
const fs = require('fs');

function copyWithBackpressure(source, dest) {
  const readStream = fs.createReadStream(source);
  const writeStream = fs.createWriteStream(dest);

  readStream.on('data', (chunk) => {
    const canContinue = writeStream.write(chunk);

    if (!canContinue) {
      // Buffer is full, pause reading
      console.log('Pausing read stream');
      readStream.pause();
    }
  });

  writeStream.on('drain', () => {
    // Buffer emptied, resume reading
    console.log('Resuming read stream');
    readStream.resume();
  });

  readStream.on('end', () => {
    writeStream.end();
  });
}

// Better: Use pipe() or pipeline() - they handle backpressure automatically
const readStream = fs.createReadStream('source.txt');
const writeStream = fs.createWriteStream('dest.txt');
readStream.pipe(writeStream);  // Automatic backpressure handling
```

## 5.10 Practical Tasks

### Task 1: Large File Processing
```javascript
// task1.js - Process large log file
const fs = require('fs');
const readline = require('readline');

async function processLargeLogFile(filePath) {
  const readStream = fs.createReadStream(filePath);
  const rl = readline.createInterface({
    input: readStream,
    crlfDelay: Infinity
  });

  let errorCount = 0;
  let warningCount = 0;

  for await (const line of rl) {
    if (line.includes('ERROR')) errorCount++;
    if (line.includes('WARNING')) warningCount++;
  }

  console.log(`Errors: ${errorCount}, Warnings: ${warningCount}`);
}

processLargeLogFile('app.log');
```

### Task 2: File Compression Utility
```javascript
// task2.js - Compress/decompress files
const fs = require('fs');
const zlib = require('zlib');
const { pipeline } = require('stream/promises');

async function compressFile(input, output) {
  await pipeline(
    fs.createReadStream(input),
    zlib.createGzip(),
    fs.createWriteStream(output)
  );
  console.log('Compression complete');
}

async function decompressFile(input, output) {
  await pipeline(
    fs.createReadStream(input),
    zlib.createGunzip(),
    fs.createWriteStream(output)
  );
  console.log('Decompression complete');
}

compressFile('large-file.txt', 'large-file.txt.gz');
```

### Task 3: Stream-Based CSV Parser
```javascript
// task3.js - Parse CSV efficiently
const fs = require('fs');
const { Transform } = require('stream');
const readline = require('readline');

class CsvParser extends Transform {
  constructor() {
    super({ objectMode: true });
    this.headers = null;
  }

  _transform(line, encoding, callback) {
    if (!this.headers) {
      this.headers = line.split(',').map(h => h.trim());
    } else {
      const values = line.split(',');
      const obj = {};
      this.headers.forEach((header, i) => {
        obj[header] = values[i]?.trim();
      });
      this.push(obj);
    }
    callback();
  }
}

async function parseCsv(filePath) {
  const readStream = fs.createReadStream(filePath);
  const rl = readline.createInterface({
    input: readStream,
    crlfDelay: Infinity
  });

  const parser = new CsvParser();
  const writeStream = fs.createWriteStream('output.json');

  rl.on('line', (line) => {
    parser.write(line);
  });

  parser.on('data', (obj) => {
    writeStream.write(JSON.stringify(obj) + '\n');
  });

  rl.on('close', () => {
    parser.end();
  });

  parser.on('end', () => {
    writeStream.end();
    console.log('CSV parsing complete');
  });
}

parseCsv('data.csv');
```

### Task 4: Stream Progress Tracker
```javascript
// task4.js - Track file processing progress
const fs = require('fs');
const { Transform } = require('stream');

class ProgressTransform extends Transform {
  constructor(totalSize) {
    super();
    this.totalSize = totalSize;
    this.processedSize = 0;
  }

  _transform(chunk, encoding, callback) {
    this.processedSize += chunk.length;
    const progress = ((this.processedSize / this.totalSize) * 100).toFixed(2);
    process.stdout.write(`\rProgress: ${progress}%`);
    this.push(chunk);
    callback();
  }

  _flush(callback) {
    process.stdout.write('\n');
    callback();
  }
}

async function copyWithProgress(source, dest) {
  const stats = await fs.promises.stat(source);
  const progressTransform = new ProgressTransform(stats.size);

  fs.createReadStream(source)
    .pipe(progressTransform)
    .pipe(fs.createWriteStream(dest))
    .on('finish', () => console.log('Copy complete!'));
}

copyWithProgress('large-file.txt', 'copy.txt');
```

## 5.11 Interview Questions

### Basic Level:
1. **What is a Buffer and why is it needed?**
2. **What are the four types of streams?**
3. **What's the difference between pipe() and pipeline()?**
4. **How do you create a readable stream?**

### Intermediate Level:
1. **Explain backpressure in streams.**
2. **What's the difference between flowing and paused mode?**
3. **How do you handle errors in piped streams?**
4. **What is highWaterMark in streams?**

### Advanced Level:
1. **Implement a custom transform stream for CSV parsing.**
2. **How would you process a 10GB file efficiently?**
3. **Explain the internal workings of stream buffering.**
4. **How do you test stream-based code?**

## 5.12 Key Takeaways

- Buffers handle binary data efficiently
- Streams process data in chunks for memory efficiency
- Four stream types: Readable, Writable, Duplex, Transform
- Use pipe() or pipeline() for automatic backpressure handling
- Streams are EventEmitters (emit 'data', 'end', 'error')
- Always handle errors in streams
- pipeline() provides better error handling than pipe()
- Use streams for large file processing

## Next Chapter Preview

In Chapter 6, we'll explore Express.js and Web Frameworks. You'll learn middleware, routing, error handling, and building RESTful APIs.

---

**Memory Tip**: Streams = Memory efficient, Buffers = Binary data handling
