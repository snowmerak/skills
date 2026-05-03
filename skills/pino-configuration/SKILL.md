---
name: pino-configuration
description: Pino configuration including all options, transports, serializers, formatters, hooks, redaction, and mixin patterns. Use when configuring Pino logger instances, setting up transports, customizing output format, or implementing security features like redaction.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: pino
  category: configuration
---

# Pino Configuration Skills

This skill covers comprehensive Pino configuration including options, transports, serializers, formatters, hooks, and security features.

## Installation

```bash
npm install pino
```

For development formatting:

```bash
npm install -D pino-pretty
```

## Complete Configuration Options

### Core Options Object

```typescript
import pino from 'pino';

const logger = pino({
  // === Level Settings ===
  level: 'info',                          // Minimum log level
  levelComparison: 'ASC',                 // 'ASC' | 'DESC' | custom function
  customLevels: { foo: 35 },              // Add custom levels
  useOnlyCustomLevels: false,             // Only use customLevels

  // === Output Settings ===
  name: 'my-app',                         // Logger name in every log line
  base: { pid: undefined, hostname: undefined },  // Base bindings (null to omit)
  messageKey: 'msg',                      // Key for log message
  nestedKey: null,                        // Nest all JSON under this key

  // === Timestamps ===
  timestamp: true,                        // Enable timestamps (default: true)
  
  // === Serializers ===
  serializers: {                          // Custom property serializers
    err: pino.stdSerializers.err,
    req: pino.stdSerializers.req,         // From pino-http
    res: pino.stdSerializers.res,         // From pino-http
  },

  // === Formatters ===
  formatters: {
    level(label, number) {                // Format the level field
      return { level: number };
    },
    bindings(pid, hostname) {             // Format base bindings
      return { pid, hostname };
    },
    log(object) {                         // Format entire log object
      return object;
    }
  },

  // === Hooks ===
  hooks: {
    logMethod(inputArgs, method, level) { // Manipulate args before logging
      return method.apply(this, inputArgs);
    },
    streamWrite(str) {                    // Manipulate stringified JSON
      return str.replaceAll('secret', 'XXX');
    }
  },

  // === Redaction ===
  redact: ['password', 'token'],          // Array of paths to redact
  
  // === Mixin ===
  mixin() {                               // Add dynamic fields to every log
    return { requestId: generateId() };
  },
  mixinMergeStrategy: undefined,          // Function(mergeObject, mixinObject) => object

  // === Depth Limits ===
  depthLimit: 5,                          // Max nesting depth for objects
  edgeLimit: 100,                         // Max properties per object

  // === Other ===
  enabled: true,                          // Enable/disable logging
});
```

## Transport Configuration (v7+)

Transports run in separate worker threads for non-blocking performance.

### Single Transport Target

```typescript
import pino from 'pino';

// Pretty output for development
const logger = pino({
  transport: {
    target: 'pino-pretty',
    options: {
      colorize: true,
      translateTime: 'HH:MM:ss l',
      ignore: 'pid,hostname',
    },
  },
});

// File transport
const logger = pino({
  transport: {
    target: 'pino/file',
    options: { destination: './logs/app.log' },
  },
});
```

### Multi-Transports (Multiple Destinations)

```typescript
import pino from 'pino';

// Log to multiple destinations with different levels
const logger = pino({
  transport: {
    targets: [
      // Console output for development (all info and above)
      { target: 'pino-pretty', level: 'info' },
      
      // File for all logs
      { 
        target: 'pino/file', 
        options: { destination: './logs/app.log' } 
      },
      
      // Errors only to stderr
      { 
        target: 'pino/file', 
        options: { destination: 2 }, 
        level: 'error' 
      },
      
      // External logging service (e.g., LogTail, Axiom)
      { target: '@logtail/pino', level: 'warn' },
    ],
  },
});
```

### Per-Target Level Filtering

```typescript
import pino from 'pino';

// Each transport can have its own log level
const logger = pino({
  transport: {
    targets: [
      { target: './my-transport.mjs', level: 'error' },   // Only errors
      { target: 'pino/file', options: { destination: '/dev/null' } },  // All (default: info)
    ],
  },
});
```

### Deduplication Mode

```typescript
import pino from 'pino';

// Log only to the highest-level target that matches
const logger = pino({
  transport: {
    targets: [
      { target: 'pino-pretty', level: 'info' },
      { target: '@logtail/pino', level: 'error' },
    ],
    dedupe: true,  // Error logs go only to @logtail/pino, not both
  },
});
```

### Transport Pipeline

```typescript
import pino from 'pino';

// Chain transports together (output of one becomes input of next)
const logger = pino({
  transport: {
    pipeline: [
      // First transform: add timestamp format
      { target: './transform-timestamp.js' },
      // Then send to file
      { target: 'pino/file', options: { destination: 1 } },
    ],
  },
});
```

### Notable Transports

| Package | Purpose | Installation |
|---------|---------|--------------|
| `pino-pretty` | Human-readable console output | `npm i -D pino-pretty` |
| `pino/file` | Write to file or file descriptor (built-in) | Built-in |
| `pino-elasticsearch` | Send logs to Elasticsearch | `npm i pino-elasticsearch` |
| `@logtail/pino` | LogTail cloud logging | `npm i @logtail/pino` |
| `@axiomhq/pino` | Axiom observability platform | `npm i @axiomhq/pino` |
| `pino-loki` | Grafana Loki integration | `npm i pino-loki` |
| `pino-roll` | Log file rotation | `npm i pino-roll` |
| `pino-logfmt` | Logfmt format output | `npm i pino-logfmt` |

## Serializers

Serializers customize how specific properties are serialized to JSON.

### Standard Serializers

```typescript
import pino from 'pino';

const logger = pino({
  serializers: {
    // Error serializer (includes stack trace)
    err: pino.stdSerializers.err,
    
    // HTTP request serializer (from pino-http)
    req: pino.stdSerializers.req,
    
    // HTTP response serializer (from pino-http)
    res: pino.stdSerializers.res,
  },
});

// Usage
logger.error(new Error('Something failed'), 'Error occurred');
// err property is automatically serialized with type, message, stack
```

### Custom Serializers

```typescript
import pino from 'pino';

const logger = pino({
  serializers: {
    // Custom serializer for user objects (hide sensitive data)
    user: (user) => {
      if (!user) return undefined;
      return {
        id: user.id,
        email: user.email,
        role: user.role,
        // password is NOT included
      };
    },
    
    // Custom serializer for arrays
    tags: (tags) => {
      return Array.isArray(tags) ? tags.join(',') : tags;
    },
  },
});

logger.info({ user: { id: 1, email: 'test@test.com', password: 'secret' } }, 'User action');
// Output only includes id, email, role - NOT password
```

### Override Default Serializers

```typescript
import pino from 'pino';

const logger = pino({
  serializers: {
    // Disable error serialization (not recommended)
    err: undefined,
    
    // Custom error serializer
    err: (err) => ({
      message: err.message,
      code: err.code,
    }),
  },
});
```

## Formatters

Formatters customize the structure of log output.

### Level Formatter

```typescript
import pino from 'pino';

// Default: { level: 30 }
const logger1 = pino();

// Custom: { level: 'INFO', label: 'INFO' }
const logger2 = pino({
  formatters: {
    level(label, number) {
      return { 
        level: label.toUpperCase(),
        label,
        value: number,
      };
    },
  },
});

logger2.info('test');
// Output: {"level":"INFO","label":"info","value":30,...}
```

### Bindings Formatter

```typescript
import pino from 'pino';

const logger = pino({
  formatters: {
    bindings(pid, hostname) {
      return { 
        pid, 
        hostname,
        service: 'my-service',  // Add custom field
      };
    },
  },
});
```

### Log Object Formatter

```typescript
import pino from 'pino';

const logger = pino({
  formatters: {
    log(object) {
      // Modify every log object before output
      const now = new Date().toISOString();
      return { ...object, timestamp: now };
    },
  },
});
```

## Hooks

Hooks allow manipulation of logging behavior.

### logMethod Hook

```typescript
import pino from 'pino';

// Manipulate arguments before logging
const logger = pino({
  hooks: {
    logMethod(inputArgs, method, level) {
      // Add timestamp to every message
      const [arg1, arg2, ...rest] = inputArgs;
      
      if (typeof arg1 === 'string') {
        // Simple message: logger.info('message')
        return method.call(this, { msg: arg1 }, arg2, ...rest);
      }
      
      // Object + message: logger.info({ userId: 1 }, 'message')
      return method.apply(this, inputArgs);
    },
  },
});

// Or filter certain messages
const logger = pino({
  hooks: {
    logMethod(inputArgs, method, level) {
      const msg = typeof inputArgs[0] === 'string' 
        ? inputArgs[0] 
        : inputArgs[1];
      
      if (msg && msg.includes('internal')) {
        return; // Skip internal messages
      }
      
      return method.apply(this, inputArgs);
    },
  },
});
```

### streamWrite Hook

```typescript
import pino from 'pino';

// Manipulate the final JSON string before writing
const logger = pino({
  hooks: {
    streamWrite(str) {
      // Redact sensitive data in output
      return str
        .replace(/"password":"[^"]*"/g, '"password":"[REDACTED]"')
        .replace(/"token":"[^"]*"/g, '"token":"[REDACTED]"');
    },
  },
});
```

## Redaction (Security)

Redact sensitive data from log output.

### Simple Array Paths

```typescript
import pino from 'pino';

const logger = pino({
  redact: ['password', 'token', 'apiSecret'],
});

logger.info({ 
  user: { password: 'secret123' }, 
  apiToken: 'abc-123' 
}, 'Login attempt');
// Output: {"user":{"password":"[REDACTED]"},"apiToken":"[REDACTED]",...}
```

### Object Configuration

```typescript
import pino from 'pino';

const logger = pino({
  redact: {
    paths: ['password', 'token', 'user.creditCard'],
    censor: '[REDACTED]',       // Replacement value (default)
    remove: false,              // Remove key entirely if true
  },
});

// With remove: true
const logger2 = pino({
  redact: {
    paths: ['password', 'token'],
    remove: true,  // Key is completely removed from output
  },
});
```

### Nested Paths and Wildcards

```typescript
import pino from 'pino';

const logger = pino({
  redact: {
    paths: [
      'user.password',           // Exact nested path
      'apiToken',                // Top-level key
      '*.creditCard',            // Any key named creditCard at any depth
      'payment.*',               // All keys under payment object
    ],
    censor: '[REDACTED]',
  },
});

logger.info({
  user: { password: 'secret', profile: { name: 'John' } },
  payment: { creditCard: '1234-5678-9012-3456', method: 'card' },
});
// Output: {"user":{"password":"[REDACTED]","profile":{"name":"John"}},"payment":{"creditCard":"[REDACTED]","method":"card"}}
```

### Custom Censor Function

```typescript
import pino from 'pino';

const logger = pino({
  redact: {
    paths: ['password', 'token'],
    censor(value, path) {
      // Custom logic based on value or path
      if (path.includes('password')) return '***';
      if (path.includes('token')) return '[TOKEN]';
      return '[REDACTED]';
    },
  },
});
```

## Mixin (Dynamic Fields)

Add dynamic fields to every log entry.

### Basic Mixin

```typescript
import pino from 'pino';

const logger = pino({
  mixin() {
    return { 
      environment: process.env.NODE_ENV,
      version: '1.0.0',
    };
  },
});

logger.info('message');
// Output includes: {"environment":"production","version":"1.0.0",...}
```

### Async Mixin

```typescript
import pino from 'pino';

const logger = pino({
  async mixin() {
    const requestId = await generateRequestId();
    return { requestId };
  },
});
```

### Mixin Merge Strategy

```typescript
import pino from 'pino';

const logger = pino({
  mixin() {
    return { timestamp: Date.now() };
  },
  mixinMergeStrategy(mergeObject, mixinObject) {
    // Custom merge logic - child bindings take priority
    return { ...mixinObject, ...mergeObject };
  },
});

const child = logger.child({ timestamp: 'custom' });
child.info('message');
// Uses custom merge strategy
```

## Depth Limits (Circular Objects)

Prevent infinite loops with circular references.

```typescript
import pino from 'pino';

const logger = pino({
  depthLimit: 5,      // Max nesting depth (default: 6)
  edgeLimit: 100,     // Max properties per object (default: 100)
});

// Objects beyond depthLimit are truncated to '[Circular]'
```

## Environment-Specific Configuration

### Development Configuration

```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'debug',
  transport: {
    target: 'pino-pretty',
    options: {
      colorize: true,
      translateTime: 'SYS:yyyy-mm-dd HH:MM:ss.l',
      ignore: 'pid,hostname',
      singleLine: false,
    },
  },
});
```

### Production Configuration

```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  timestamp: pino.stdTimeFunctions.isoTime,
  formatters: {
    level(label) {
      return { level: label.toUpperCase() };
    },
  },
  redact: {
    paths: ['password', 'token', 'apiSecret'],
    remove: true,
  },
});
```

### Docker/Container Configuration

```typescript
import pino from 'pino';

// No pid/hostname in containers (orchestrator provides this)
const logger = pino({
  base: null,
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV !== 'production'
    ? { target: 'pino-pretty' }
    : undefined,
});
```

## Best Practices

### 1. Use Transports for Production

```typescript
// GOOD: Non-blocking async logging
const logger = pino({
  transport: { target: 'pino/file', options: { destination: './logs/app.log' } },
});

// BAD: Sync logging blocks the event loop
const logger = pino(pino.destination({ sync: true }));
```

### 2. Redact Sensitive Data

```typescript
const logger = pino({
  redact: ['password', 'token', 'apiSecret'],
});
```

### 3. Use Appropriate Level per Environment

```typescript
const level = 
  process.env.NODE_ENV === 'production' ? 'info' : 
  process.env.NODE_ENV === 'staging' ? 'debug' : 'trace';

const logger = pino({ level });
```

### 4. Use Child Loggers for Request Context

```typescript
// In HTTP middleware
const requestLogger = logger.child({ 
  requestId: req.id, 
  method: req.method, 
  path: req.path 
});
```

### 5. Avoid Logging Large Objects in Production

```typescript
// GOOD
logger.info({ userId: 123 }, 'User action');

// BAD - may contain sensitive/large data
logger.info({ user: fullUserObjectWithAllFields }, 'User action');
```

## References

- [Pino Configuration Documentation](https://getpino.io/#/docs/api)
- [Transports](https://getpino.io/#/docs/transports)
- [Redaction Guide](https://getpino.io/#/docs/redaction)
