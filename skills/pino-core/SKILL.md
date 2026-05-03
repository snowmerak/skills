---
name: pino-core
description: Pino core logging concepts including logger creation, log levels, child loggers, bindings, and the complete logging API. Use when creating loggers, understanding log levels, using child loggers for context, or working with Pino's core API in NestJS applications.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: pino
  category: core
---

# Pino Core Logging Skills

This skill covers the fundamental concepts of Pino logging including logger creation, log levels, child loggers, and the complete logging API.

## Installation

```bash
npm install pino
```

For NestJS applications, use `nestjs-pino` instead (see drizzle-nestjs skill):

```bash
npm install nestjs-pino pino-http
```

## Logger Creation

### Basic Logger

```typescript
import pino from 'pino';

// Simple logger with default settings
const logger = pino();

logger.info('hello world');
logger.error('something bad happened');
```

### Logger with Options

```typescript
import pino from 'pino';

const logger = pino({
  level: 'info',           // Minimum log level
  name: 'my-app',          // Logger name (appears in every log line)
});

logger.info('message with name');
// Output: {"level":30,"time":"...","pid":...,"hostname":"...","name":"my-app","msg":"message with name"}
```

### Logger with Custom Destination

```typescript
import pino from 'pino';

// Write to file
const logger = pino(pino.destination('./logs/app.log'));

// Write to stdout (fd 1)
const logger = pino(pino.destination(1));

// Write to stderr (fd 2)
const logger = pino(pino.destination(2));

// Async buffered writing
const logger = pino(pino.destination({
  dest: './logs/app.log',
  minLength: 4096,    // Buffer size before flush
  sync: false,        // Asynchronous mode
}));
```

### Logger with Transport (v7+)

```typescript
import pino from 'pino';

// Single transport target
const logger = pino({
  transport: {
    target: 'pino-pretty',  // Human-readable output for development
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

## Log Levels

Pino defines 7 log levels with numeric values (lower number = more verbose):

| Level   | Value | Method              | Use Case                                    |
|---------|-------|---------------------|---------------------------------------------|
| fatal   | 60    | `logger.fatal()`    | Application cannot continue, must exit      |
| error   | 50    | `logger.error()`    | Error condition that should be investigated |
| warn    | 40    | `logger.warn()`     | Warning about potential issues              |
| info    | 30    | `logger.info()`     | General information about application flow  |
| debug   | 20    | `logger.debug()`    | Debugging information                       |
| trace   | 10    | `logger.trace()`    | Detailed tracing of execution               |
| silent  | 99    | -                   | Turn off all logging                        |

### Usage Examples

```typescript
import pino from 'pino';

const logger = pino({ level: 'info' });

// Info level
logger.info('Application started');
logger.info({ userId: 123 }, 'User logged in');

// Error level with Error object
try {
  await someOperation();
} catch (err) {
  logger.error(err, 'Operation failed');
}

// Fatal level - application should exit after this
try {
  criticalInit();
} catch (err) {
  logger.fatal(err, 'Application cannot start');
  process.exit(1);
}

// Debug level (only shown when level <= debug)
logger.debug({ data: expensiveComputation() }, 'Debug info');

// Silent - disables all logging
const silentLogger = pino({ level: 'silent' });
```

### Dynamic Level Change at Runtime

```typescript
import pino from 'pino';

const logger = pino({ level: 'info' });

logger.info('This will be logged'); // Logged

// Change level at runtime
logger.level = 'debug';

logger.debug('Now debug is enabled'); // Now logged

// Check if level is enabled (for expensive operations)
if (logger.isLevelEnabled('trace')) {
  logger.trace({ data: getExpensiveData() }, 'Trace info');
}
```

### Custom Log Levels

```typescript
import pino from 'pino';

const logger = pino({
  customLevels: {
    http: 35,      // Between info (30) and warn (40)
    security: 25,  // Between debug (20) and info (30)
  },
});

logger.http('HTTP request processed');
logger.security({ userId: 123 }, 'Security check passed');
```

### Use Only Custom Levels

```typescript
import pino from 'pino';

const logger = pino({
  customLevels: { foo: 35, bar: 25 },
  useOnlyCustomLevels: true,
  level: 'foo',
});

logger.foo('works');      // Works
logger.bar('also works'); // Works
// logger.info('throws error - info not available');
```

## Logger API Methods

### Standard Logging Methods

All methods accept the same signature patterns:

```typescript
import pino from 'pino';

const logger = pino();

// Pattern 1: (message)
logger.info('Simple message');

// Pattern 2: (object, message) - structured logging
logger.info({ userId: 123, action: 'login' }, 'User logged in');

// Pattern 3: (error, message) - error logging
logger.error(new Error('Connection failed'), 'Database connection error');

// Pattern 4: (message, ...interpolation) - string interpolation
logger.info('User %s logged in from %s', 'john', '192.168.1.1');

// Pattern 5: (object, message, ...interpolation)
logger.info({ userId: 123 }, 'User %s logged in', 'john');
```

### Error Logging Best Practices

```typescript
import pino from 'pino';

const logger = pino();

// GOOD: Pass Error object as first argument for proper serialization
try {
  await db.query();
} catch (err) {
  logger.error(err, 'Database query failed');
}

// GOOD: Include context with error
logger.error(
  { userId: 123, orderId: 456, operation: 'payment' },
  new Error('Payment processing failed')
);

// BAD: Stringify error manually (loses stack trace)
logger.error({ error: err.message });

// GOOD: Use fatal for unrecoverable errors
try {
  initCriticalService();
} catch (err) {
  logger.fatal(err, 'Critical service failed - application cannot continue');
  process.exit(1);
}
```

### Logger Instance Methods

```typescript
import pino from 'pino';

const logger = pino({ name: 'my-app' });

// Get current bindings (pid, hostname, custom fields)
const bindings = logger.bindings();
console.log(bindings); // { pid: 12345, hostname: '...', name: 'my-app' }

// Add additional bindings dynamically
logger.setBindings({ requestId: 'abc-123' });

// Flush buffered logs (for async transports)
await logger.flush();

// Listen for level changes
logger.on('level-change', (lvl, val, name) => {
  console.log(`Level changed to ${name} (${val})`);
});

// Access version
console.log(logger.version); // '9.x.x'
```

### Static Methods

```typescript
import pino from 'pino';

// Create destination stream
const dest = pino.destination('/path/to/file');
const stdout = pino.destination(1);
const stderr = pino.destination(2);

// Create transport
const transport = pino.transport({ target: 'pino-pretty' });

// Standard serializers (for custom use)
const serializedError = pino.stdSerializers.err(new Error('test'));
// { type: 'Error', message: 'test', stack: '...' }

// Standard time functions
const isoTime = pino.stdTimeFunctions.isoTime(); // ISO 8601 timestamp string
```

## Child Loggers

Child loggers inherit parent bindings and allow adding context-specific fields.

### Basic Child Logger

```typescript
import pino from 'pino';

const logger = pino({ name: 'app' });

// Create child with additional bindings
const userLogger = logger.child({ userId: 123, module: 'auth' });

userLogger.info('User authenticated');
// Output includes: {"userId":123,"module":"auth","name":"app",...}

userLogger.debug('Checking permissions'); // Also includes child bindings
```

### Nested Child Loggers

```typescript
import pino from 'pino';

const logger = pino();

// Parent logger
const requestLogger = logger.child({ requestId: 'abc-123' });

// Child of child
const userRequestLogger = requestLogger.child({ userId: 456, path: '/api/users' });

userRequestLogger.info('Processing request');
// Output includes: {"requestId":"abc-123","userId":456,"path":"/api/users",...}
```

### Child Logger with Custom Methods

```typescript
import pino from 'pino';

const logger = pino();

// Create child with custom logging methods
const httpLogger = logger.child({
  module: 'http',
}, {
  http: function (...args) {
    this.info(...args); // Map http level to info
  },
});

httpLogger.http('Request received');
```

### Removing Bindings from Child

```typescript
import pino from 'pino';

const logger = pino({ userId: 123 });

// Child inherits userId
const child = logger.child({ path: '/api/users' });
child.info('request'); // Includes userId: 123, path: '/api/users'

// Create grandchild that removes userId
const grandChild = child.child({}, { remove: ['userId'] });
grandChild.info('request'); // Only includes path: '/api/users', NOT userId
```

## Logger Bindings

### Base Bindings

By default, Pino adds `pid`, `hostname`, and optionally `name` to every log line:

```typescript
import pino from 'pino';

// Default - includes pid and hostname
const logger1 = pino();
logger1.info('test'); // {"pid":123,"hostname":"...","msg":"test"}

// Custom name added
const logger2 = pino({ name: 'my-app' });
logger2.info('test'); // {"pid":123,"hostname":"...","name":"my-app","msg":"test"}

// No base bindings (useful for containerized environments)
const logger3 = pino({ base: null });
logger3.info('test'); // {"msg":"test"}

// Custom base bindings
const logger4 = pino({ base: { service: 'my-service', version: '1.0.0' } });
logger4.info('test'); // {"service":"my-service","version":"1.0.0","msg":"test"}
```

### Dynamic Bindings with setBindings

```typescript
import pino from 'pino';

const logger = pino();

// Initial bindings
logger.info('initial log'); // { pid: ..., hostname: ... }

// Add more bindings dynamically
logger.setBindings({ requestId: 'abc-123', tenantId: 'tenant-1' });

// All subsequent logs include new bindings
logger.info('later log'); // { pid: ..., hostname: ..., requestId: 'abc-123', tenantId: 'tenant-1' }
```

## Best Practices

### 1. Use Structured Logging

```typescript
// GOOD: Log data as structured properties
logger.info({ userId: 123, action: 'login', ip: '192.168.1.1' }, 'User logged in');

// BAD: String interpolation for data (harder to query/analyze)
logger.info('User %s with IP %s logged in', userId, ip);
```

### 2. Use Child Loggers for Context

```typescript
// GOOD: Use child loggers for request-specific logging
const requestLogger = logger.child({ requestId: generateId(), path: req.path });
requestLogger.info('Processing request');

// BAD: Manually add context to every log
logger.info({ requestId: id, path: '/api/users' }, 'message 1');
logger.info({ requestId: id, path: '/api/users' }, 'message 2');
```

### 3. Set Appropriate Level for Environment

```typescript
const logger = pino({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
});
```

### 4. Check Level Before Expensive Operations

```typescript
if (logger.isLevelEnabled('debug')) {
  logger.debug({ data: getExpensiveData() }, 'Debug info');
}
```

### 5. Always Pass Error Objects Directly

```typescript
// GOOD: Pino serializes error properly with stack trace
logger.error(err, 'Operation failed');

// BAD: Loses stack trace and error details
logger.error({ errorMessage: err.message });
```

### 6. Use Fatal for Unrecoverable Errors

```typescript
try {
  initCriticalService();
} catch (err) {
  logger.fatal(err, 'Critical service failed');
  process.exit(1); // Exit after fatal log
}
```

## References

- [Pino Documentation](https://getpino.io/#/docs/api)
- [Child Loggers](https://getpino.io/#/docs/child-loggers)
- [Transports](https://getpino.io/#/docs/transports)
