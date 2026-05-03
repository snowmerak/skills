---
name: pino-core
description: Pino core logging concepts including logger creation, log levels, child loggers, bindings, and the complete logging API. Use when creating loggers, understanding log levels, using child loggers for context, or working with Pino's core API in NestJS applications.
license: MIT
metadata:
  author: snowmerak
  version: '1.0'
  category: core
  tags: [pino, logging, core, structured-logging]
---

# Pino Core Logging

This skill covers fundamental Pino concepts: logger creation, log levels, child loggers, bindings, and the complete logging API.

## SOP: Step-by-Step Procedures

### SOP 1: Logger Creation

**Basic Logger:**

```typescript
import pino from 'pino';

const logger = pino();
logger.info('hello world');
logger.error('something bad happened');
```

**Logger with Options:**

```typescript
const logger = pino({
  level: 'info',       // Minimum log level
  name: 'my-app',      // Logger name (appears in every log line)
});
```

**Custom Destination:**

```typescript
// Write to file
const logger1 = pino(pino.destination('./logs/app.log'));

// Async buffered writing
const logger2 = pino(pino.destination({
  dest: './logs/app.log',
  minLength: 4096,    // Buffer size before flush
  sync: false,        // Asynchronous mode
}));
```

**Transport (v7+):**

```typescript
// Pretty print for development
const logger1 = pino({ transport: { target: 'pino-pretty' } });

// File transport
const logger2 = pino({
  transport: {
    target: 'pino/file',
    options: { destination: './logs/app.log' },
  },
});
```

### SOP 2: Log Levels

Pino defines 7 log levels (lower number = more verbose):

| Level   | Value | Method              | Use Case                                    |
|---------|-------|---------------------|---------------------------------------------|
| fatal   | 60    | `logger.fatal()`    | Application cannot continue, must exit      |
| error   | 50    | `logger.error()`    | Error condition that should be investigated |
| warn    | 40    | `logger.warn()`     | Warning about potential issues              |
| info    | 30    | `logger.info()`     | General information about application flow  |
| debug   | 20    | `logger.debug()`    | Debugging information                       |
| trace   | 10    | `logger.trace()`    | Detailed tracing of execution               |
| silent  | 99    | -                   | Turn off all logging                        |

**Usage Examples:**

```typescript
const logger = pino({ level: 'info' });

// Info level
logger.info('Application started');
logger.info({ userId: 123 }, 'User logged in');

// Error with Error object
try {
  await someOperation();
} catch (err) {
  logger.error(err, 'Operation failed');
}

// Fatal - application should exit after this
try {
  criticalInit();
} catch (err) {
  logger.fatal(err, 'Application cannot start');
  process.exit(1);
}

// Check if level is enabled (for expensive operations)
if (logger.isLevelEnabled('trace')) {
  logger.trace({ data: getExpensiveData() }, 'Trace info');
}
```

**Dynamic Level Change at Runtime:**

```typescript
const logger = pino({ level: 'info' });
logger.info('This will be logged'); // Logged

// Change level at runtime
logger.level = 'debug';
logger.debug('Now debug is enabled'); // Now logged
```

### SOP 3: Logging API Methods

All methods accept these signature patterns:

```typescript
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

**Error Logging Best Practices:**

```typescript
// GOOD: Pass Error object as first argument for proper serialization
try {
  await db.query();
} catch (err) {
  logger.error(err, 'Database query failed');
}

// GOOD: Include context with error
logger.error(
  { userId: 123, orderId: 456, operation: 'payment' },
  new Error('Payment processing failed'),
);

// BAD: Stringify error manually (loses stack trace)
logger.error({ error: err.message });
```

**Logger Instance Methods:**

```typescript
const logger = pino({ name: 'my-app' });

// Get current bindings
const bindings = logger.bindings(); // { pid, hostname, name }

// Add bindings dynamically
logger.setBindings({ requestId: 'abc-123' });

// Flush buffered logs (async transports)
await logger.flush();

// Listen for level changes
logger.on('level-change', (lvl, val, name) => {
  console.log(`Level changed to ${name} (${val})`);
});
```

### SOP 4: Child Loggers

Child loggers inherit parent bindings and allow adding context-specific fields.

**Basic Child Logger:**

```typescript
const logger = pino({ name: 'app' });

// Create child with additional bindings
const userLogger = logger.child({ userId: 123, module: 'auth' });

userLogger.info('User authenticated');
// Output includes: {"userId":123,"module":"auth","name":"app",...}
```

**Nested Child Loggers:**

```typescript
const requestLogger = logger.child({ requestId: 'abc-123' });
const userRequestLogger = requestLogger.child({ userId: 456, path: '/api/users' });

userRequestLogger.info('Processing request');
// Output includes: {"requestId":"abc-123","userId":456,"path":"/api/users",...}
```

**Removing Bindings from Child:**

```typescript
const logger = pino({ userId: 123 });
const child = logger.child({ path: '/api/users' });
child.info('request'); // Includes userId: 123, path: '/api/users'

// Grandchild removes userId
const grandChild = child.child({}, { remove: ['userId'] });
grandChild.info('request'); // Only includes path: '/api/users', NOT userId
```

### SOP 5: Logger Bindings

**Base Bindings:**

```typescript
// Default - includes pid and hostname
const logger1 = pino();
logger1.info('test'); // {"pid":123,"hostname":"...","msg":"test"}

// Custom name added
const logger2 = pino({ name: 'my-app' });
logger2.info('test'); // {"pid":123,"hostname":"...","name":"my-app","msg":"test"}

// No base bindings (containerized environments)
const logger3 = pino({ base: null });
logger3.info('test'); // {"msg":"test"}

// Custom base bindings
const logger4 = pino({ base: { service: 'my-service', version: '1.0.0' } });
logger4.info('test'); // {"service":"my-service","version":"1.0.0","msg":"test"}
```

**Dynamic Bindings:**

```typescript
const logger = pino();
logger.setBindings({ requestId: 'abc-123', tenantId: 'tenant-1' });
// All subsequent logs include new bindings
logger.info('later log'); // { pid, hostname, requestId: 'abc-123', tenantId: 'tenant-1' }
```

## Tool Integration

| Task | Tool | Usage |
|------|------|-------|
| Verify Pino installation | `run_command` | `npm list pino` |
| Find logger creation points | `search_files` | Search for `pino()` or `import pino from 'pino'` |
| Add child logger context | `edit_file` | Replace `logger.info(...)` with `logger.child({ ctx }).info(...)` |
| Inspect log output format | `run_command` + `read_file` | Check log file or stdout for JSON structure |

## Anti-Patterns & Guardrails

❌ **Never** stringify Error objects manually — always pass as first argument:
```typescript
// BAD - loses stack trace and error details
logger.error({ errorMessage: err.message });
// GOOD - Pino serializes properly with full stack
logger.error(err, 'Operation failed');
```

❌ **Never** use string interpolation for structured data:
```typescript
// BAD - harder to query/analyze in log aggregators
logger.info('User %s with IP %s logged in', userId, ip);
// GOOD - structured logging
logger.info({ userId, ip }, 'User logged in');
```

❌ **Never** use `logger.fatal()` without exiting the process:
```typescript
// BAD - app continues running after fatal error
logger.fatal(err, 'Critical failure');
// GOOD - exit after fatal
logger.fatal(err, 'Critical failure');
process.exit(1);
```

⚠️ **Always** use child loggers for request-scoped context instead of adding fields to every call.

⚠️ **Always** check `isLevelEnabled()` before expensive operations at debug/trace level.

## Best Practices

1. Use structured logging: `{ userId, action }` not string interpolation
2. Use child loggers (`logger.child({ requestId })`) for request-scoped context
3. Set appropriate level per environment: `production → info`, `development → debug`
4. Check `isLevelEnabled()` before expensive operations at lower levels
5. Always pass Error objects directly as first argument to `.error()` / `.fatal()`
6. Use `base: null` in containerized environments (orchestrator provides pid/hostname)

## References

- [Pino Documentation](https://getpino.io/#/docs/api)
- [Child Loggers](https://getpino.io/#/docs/child-loggers)
- [Transports](https://getpino.io/#/docs/transports)

---

**Skill successfully created:** `skills/pino-core/SKILL.md`

This skill is now ready. Please renew the Skill Index before using it.
