---
name: pino-http
description: HTTP request/response logging with pino-http including middleware setup, correlation IDs, custom log levels, and request context. Use when setting up HTTP logging for Express, Fastify, or Node.js HTTP servers with Pino.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: pino
  category: http
---

# Pino HTTP Logging Skills

This skill covers HTTP request/response logging using pino-http middleware for Express, Fastify, and Node.js HTTP servers.

## Installation

```bash
npm install pino-http
```

For pretty printing in development:

```bash
npm install -D pino-pretty
```

## Basic Setup

### Node.js HTTP Server

```typescript
import http from 'http';
import pinoHttp from 'pino-http';

const logger = pinoHttp();

const server = http.createServer((req, res) => {
  // Attach middleware - logs request/response automatically
  logger(req, res);
  
  // Access child logger on request object
  req.log.info('Processing request');
  
  res.end('Hello World');
});

server.listen(3000);
```

### Express Integration

```typescript
import express from 'express';
import pinoHttp from 'pino-http';

const app = express();

// Basic setup
app.use(pinoHttp());

// With custom logger instance
const logger = require('pino')();
app.use(pinoHttp({ logger }));

app.get('/', (req, res) => {
  req.log.info('Handling GET request');
  res.send('Hello World');
});

app.listen(3000);
```

### Fastify Integration

```typescript
import Fastify from 'fastify';
import pino from 'pino';

const logger = pino({
  transport: { target: 'pino-pretty' },
});

const fastify = Fastify({ logger });

// Fastify has built-in Pino support - just pass the logger
fastify.get('/', async (request, reply) => {
  request.log.info('Handling GET request');
  return { hello: 'world' };
});

await fastify.listen({ port: 3000 });
```

## Configuration Options

### Complete Options Object

```typescript
import pinoHttp, { Options } from 'pino-http';

const logger = pinoHttp({
  // === Logger Instance ===
  logger: undefined,              // Reuse existing pino instance
  
  // === Request ID ===
  genReqId: (req, res) => {       // Custom request ID generator
    const existingID = req.headers['x-request-id'];
    if (existingID) return existingID;
    const id = require('crypto').randomUUID();
    res.setHeader('X-Request-ID', id);
    return id;
  },
  
  // === Log Level Settings ===
  useLevel: 'info',               // Default log level for successful responses
  customLogLevel: (req, res, err) => {  // Custom level based on status/error
    if (res.statusCode >= 500 || err) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return null;                  // Use default useLevel
  },
  
  // === Auto Logging ===
  autoLogging: {                  // Enable/disable auto logging
    enabled: true,                // Default: true
    path: '/health',              // Path to exclude from auto logging
    ignore: ['^/health'],         // Regex patterns to ignore
  },
  
  // === Custom Messages ===
  customSuccessMessage: (req, res) => {
    return `${req.method} completed`;
  },
  customErrorMessage: (req, res, err) => {
    return `request errored with status ${res.statusCode}`;
  },
  
  // === Custom Attributes ===
  customAttributeKeys: [          // Customize property names in log output
    'req',                        // Default: 'req'
    'res',                        // Default: 'res'
    'err',                        // Default: 'err'
    'responseTime',               // Default: 'responseTime'
  ],
  
  customAttributes: (req, res, err) => {  // Add custom properties to logs
    return {
      userId: req.user?.id,
      requestId: req.id,
      responseTimeMs: res.responseTime,
    };
  },
  
  // === Serializers ===
  serializers: {                  // Custom serializers for HTTP objects
    err: require('pino').stdSerializers.err,
    req: require('pino').stdSerializers.req,
    res: require('pino').stdSerializers.res,
  },
  
  // === Response Time Formatting ===
  customReceivedObject: (req) => {  // Customize received object
    return { msg: 'request received', method: req.method };
  },
  
  // === Other Options ===
  quietReqLogger: false,          // Don't create child logger if not needed
  buffer: false,                  // Buffer logs for async flush
});
```

## Correlation ID Pattern

Correlation IDs allow tracking requests across services and logs.

### Basic Implementation

```typescript
import pinoHttp from 'pino-http';
import crypto from 'crypto';

const logger = pinoHttp({
  genReqId: (req, res) => {
    // Use existing ID from headers if present
    const existingID = req.headers['x-request-id'] as string;
    if (existingID) return existingID;
    
    // Generate new UUID
    const id = crypto.randomUUID();
    
    // Add to response headers for client
    res.setHeader('X-Request-ID', id);
    
    return id;
  },
});

// In middleware or services, access the ID via req.id
app.use((req, res, next) => {
  console.log(`Request ID: ${req.id}`);
  next();
});
```

### Advanced Correlation with Child Logger

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp({
  genReqId: () => crypto.randomUUID(),
  customProps: (req, res) => ({
    requestId: req.id,
    correlationId: req.headers['x-correlation-id'] as string || req.id,
  }),
});

// Every log from this request will include the correlation ID
app.use((req, res, next) => {
  // Access child logger with all bindings
  const requestLogger = req.log.child({ 
    userId: req.user?.id,
    action: 'process-payment'
  });
  
  requestLogger.info('Processing payment');
  // Output includes: requestId, correlationId, userId, action
  
  next();
});
```

## Custom Log Levels Based on Status Code

### Basic Custom Level

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp({
  customLogLevel: (req, res, err) => {
    // 5xx errors -> error level
    if (res.statusCode >= 500 || err) return 'error';
    
    // 4xx errors -> warn level
    if (res.statusCode >= 400) return 'warn';
    
    // Everything else -> info level (default)
    return null;
  },
});

// Output examples:
// {"level":50,"msg":"POST /api/users returned 500 in 45ms"}  // Error
// {"level":40,"msg":"GET /api/missing returned 404 in 12ms"} // Warn
// {"level":30,"msg":"GET /api/users returned 200 in 23ms"}   // Info
```

### Custom Level with Different Loggers

```typescript
import pinoHttp from 'pino-http';
import pino from 'pino';

const logger = pino();

const httpLogger = pinoHttp({
  logger,
  customLogLevel: (req, res) => {
    if (res.statusCode >= 503) return 'fatal';  // Service unavailable
    if (res.statusCode >= 500) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return 'info';
  },
});
```

## Auto Logging Configuration

### Enable/Disable Auto Logging

```typescript
import pinoHttp from 'pino-http';

// Disable auto logging completely
const logger = pinoHttp({ autoLogging: false });

// Manually log requests
app.use((req, res, next) => {
  logger(req, res);  // Manual logging
  next();
});
```

### Exclude Paths from Auto Logging

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp({
  autoLogging: {
    enabled: true,
    path: '/health',           // Single path to exclude
    ignore: [                 // Multiple patterns to ignore
      '^/health',
      '^/metrics',
      '^/favicon.ico',
    ],
  },
});

// Or in Express middleware
app.use(pinoHttp({ autoLogging: { ignore: [/^\/health/] } }));
```

### Custom Auto Logging Callbacks

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp({
  autoLogging: {
    initialize: false,        // Don't create req.log by default
    error: (req, res, err) => {  // Custom error handler
      console.error('Auto logging error:', err);
    },
  },
});
```

## Custom Properties and Attributes

### Adding Custom Properties to Logs

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp({
  customProps: (req, res) => {
    // Add properties based on request data
    return {
      userId: req.user?.id,
      tenantId: req.tenant?.id,
      apiKey: req.headers['x-api-key'],
    };
  },
});

// Every log entry will include these custom properties
```

### Custom Attribute Keys

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp({
  customAttributeKeys: {
    req: 'request',           // Change 'req' to 'request'
    res: 'response',          // Change 'res' to 'response'
    err: 'error',             // Change 'err' to 'error'
    responseTime: 'duration', // Change 'responseTime' to 'duration'
  },
});

// Output: {"request":{...},"response":{...},"error":{...},"duration":45}
```

### Custom Response Time Format

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp({
  customAttributeKeys: {
    responseTime: 'timeTaken',
  },
  customAttributes: (req, res) => ({
    timeTakenMs: res.responseTime,
    timeTakenHuman: `${res.responseTime.toFixed(2)}ms`,
  }),
});
```

## Serializers

### Standard HTTP Serializers

```typescript
import pinoHttp from 'pino-http';
import pino from 'pino';

const logger = pinoHttp({
  serializers: {
    // Serialize Error objects with stack trace
    err: pino.stdSerializers.err,
    
    // Serialize HTTP request object
    req: pino.stdSerializers.req,
    
    // Serialize HTTP response object
    res: pino.stdSerializers.res,
  },
});

// The standard serializers include:
// - err: { type, message, stack }
// - req: { method, url, headers, remoteAddress, ... }
// - res: { statusCode, headers, ... }
```

### Custom Serializers

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp({
  serializers: {
    // Customize request serialization (hide sensitive headers)
    req: (req) => ({
      method: req.method,
      url: req.url,
      headers: {
        host: req.headers.host,
        // Don't include authorization headers
      },
      remoteAddress: req.ip,
      userId: req.user?.id,
    }),
    
    // Customize response serialization
    res: (res) => ({
      statusCode: res.statusCode,
      headers: res.getHeaders ? res.getHeaders() : res.headers,
      responseTime: res.responseTime,
    }),
  },
});
```

## Request Logger Access

### Using req.log

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp();

app.use((req, res, next) => {
  // req.log is a child logger with request bindings
  req.log.info('Request received');
  req.log.debug({ headers: req.headers }, 'Request details');
  
  try {
    // Process request
    res.send('OK');
  } catch (err) {
    req.log.error(err, 'Request failed');
    next(err);
  }
});
```

### Creating Child Loggers from req.log

```typescript
import pinoHttp from 'pino-http';

const logger = pinoHttp();

app.use((req, res, next) => {
  // Create more specific child loggers
  const userLogger = req.log.child({ userId: req.user?.id });
  const actionLogger = userLogger.child({ action: 'update-profile' });
  
  actionLogger.info('Profile update started');
  // Output includes: requestId, method, url, userId, action
  
  next();
});
```

## Complete Express Example

```typescript
import express from 'express';
import pinoHttp from 'pino-http';
import pino from 'pino';
import crypto from 'crypto';

// Create base logger
const baseLogger = pino({
  name: 'my-app',
  transport: process.env.NODE_ENV !== 'production'
    ? { target: 'pino-pretty' }
    : undefined,
});

// Create HTTP middleware with custom configuration
const httpLogger = pinoHttp({
  logger: baseLogger.child({ name: 'http' }),
  
  // Correlation ID
  genReqId: (req, res) => {
    const id = (req.headers['x-request-id'] as string) || crypto.randomUUID();
    res.setHeader('X-Request-ID', id);
    return id;
  },
  
  // Custom log levels
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500 || err) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return 'info';
  },
  
  // Auto logging exclusions
  autoLogging: {
    ignore: [/^\/health/, /^\/metrics/],
  },
  
  // Custom properties
  customProps: (req, res) => ({
    userId: req.user?.id,
    tenantId: req.tenant?.id,
  }),
  
  // Custom attributes
  customAttributes: (req, res) => ({
    responseTimeMs: res.responseTime,
  }),
});

const app = express();

// Use HTTP logger middleware
app.use(httpLogger);

// Health check endpoint (excluded from auto logging)
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

// Protected route with request-specific logging
app.get('/api/users/:id', (req, res) => {
  const userLogger = req.log.child({ 
    userId: req.params.id,
    action: 'get-user'
  });
  
  userLogger.info('Fetching user');
  
  // Simulate database query
  setTimeout(() => {
    userLogger.debug('User found in database');
    res.json({ id: req.params.id, name: 'John Doe' });
  }, 100);
});

app.listen(3000);
```

## Best Practices

### 1. Always Use Correlation IDs

```typescript
const logger = pinoHttp({
  genReqId: (req, res) => {
    return req.headers['x-request-id'] || crypto.randomUUID();
  },
});
```

### 2. Set Custom Log Levels for Errors

```typescript
const logger = pinoHttp({
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500 || err) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return null;
  },
});
```

### 3. Exclude Health Check Endpoints

```typescript
const logger = pinoHttp({
  autoLogging: {
    ignore: [/^\/health/, /^\/metrics/],
  },
});
```

### 4. Use Child Loggers for Context-Specific Logging

```typescript
app.use((req, res, next) => {
  const requestLogger = req.log.child({ 
    userId: req.user?.id,
    action: 'process-payment'
  });
  
  requestLogger.info('Processing payment');
  next();
});
```

### 5. Redact Sensitive Data in Request Logs

```typescript
const logger = pinoHttp({
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      // Don't include sensitive headers
      headers: { host: req.headers.host },
    }),
  },
});
```

## References

- [pino-http Documentation](https://github.com/pinojs/pino-http)
- [Pino HTTP Middleware](https://getpino.io/#/docs/web)
