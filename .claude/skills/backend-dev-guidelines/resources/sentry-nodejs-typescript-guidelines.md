# Sentry Integration Guide for Node.js and TypeScript

This guide provides a comprehensive approach to implementing Sentry error monitoring and performance tracking in Node.js/TypeScript applications, specifically for Express.js services.

## Table of Contents

1. [Installation](#installation)
2. [Initialization](#initialization)
3. [Express Integration](#express-integration)
4. [Error Capture Strategies](#error-capture-strategies)
5. [Process-Level Error Handlers](#process-level-error-handlers)
6. [Configuration](#configuration)
7. [Best Practices](#best-practices)
8. [Complete Example](#complete-example)

---

## Installation

Install the required Sentry packages:

```bash
npm install @sentry/node @sentry/profiling-node
# or
pnpm add @sentry/node @sentry/profiling-node
# or
yarn add @sentry/node @sentry/profiling-node
```

**Packages:**
- `@sentry/node`: Core Sentry SDK for Node.js
- `@sentry/profiling-node`: Performance profiling (optional but recommended)

---

## Initialization

### 1. Create Sentry Initialization File

Create `src/shared/sentry.ts`:

```typescript
/**
 * Sentry initialization for your service
 * 
 * Initializes Sentry for error monitoring and performance tracking.
 */

import * as Sentry from '@sentry/node';
import { expressIntegration } from '@sentry/node';

// Try to import profiling, but make it optional
let nodeProfilingIntegration: any;
try {
  const profilingModule = require('@sentry/profiling-node');
  nodeProfilingIntegration = profilingModule.nodeProfilingIntegration;
} catch {
  // Profiling not available, that's okay
  nodeProfilingIntegration = null;
}

export function initializeSentry(serviceName: string = 'your-service'): void {
  const sentryDsn = process.env.SENTRY_DSN;
  
  if (!sentryDsn) {
    console.warn('⚠️  SENTRY_DSN not found - Sentry monitoring disabled');
    return;
  }

  try {
    const integrations: any[] = [
      expressIntegration(), // This replaces requestHandler, tracingHandler, and errorHandler
    ];
    if (nodeProfilingIntegration) {
      integrations.push(nodeProfilingIntegration());
    }

    // Default to 0.1 (10%) for production, 1.0 (100%) for development
    const defaultTracesSampleRate = process.env.NODE_ENV === 'production' ? '0.1' : '1.0';
    const defaultProfilesSampleRate = process.env.NODE_ENV === 'production' ? '0.1' : '1.0';

    Sentry.init({
      dsn: sentryDsn,
      environment: process.env.ENVIRONMENT || process.env.NODE_ENV || 'development',
      tracesSampleRate: parseFloat(process.env.SENTRY_TRACES_SAMPLE_RATE || defaultTracesSampleRate),
      profilesSampleRate: nodeProfilingIntegration ? parseFloat(process.env.SENTRY_PROFILES_SAMPLE_RATE || defaultProfilesSampleRate) : undefined,
      integrations,
      sendDefaultPii: false, // Don't send personally identifiable information
      release: process.env.RELEASE || '1.0.0',
      debug: process.env.SENTRY_DEBUG === 'true',
    });

    // Add global context
    Sentry.setContext('service', {
      name: serviceName,
      version: process.env.RELEASE || '1.0.0',
    });

    const environment = process.env.ENVIRONMENT || process.env.NODE_ENV || 'development';
    const release = process.env.RELEASE || '1.0.0';
    const atIndex = sentryDsn.indexOf('@');
    const dsnPrefix = atIndex > 0 ? sentryDsn.substring(0, atIndex) : '***';
    console.log(`✅ Sentry DSN confirmed - initialized for ${serviceName}`);
    console.log(`   Environment: ${environment}, Release: ${release}, DSN: ${dsnPrefix}@...`);
    console.log('🔍 Sentry initialized for error monitoring and performance tracking');
  } catch (error) {
    console.error('❌ Failed to initialize Sentry:', error);
    // Continue without Sentry - graceful degradation
  }
}

// Setup process-level error handlers
export function setupProcessErrorHandlers(): void {
  // Capture unhandled promise rejections
  process.on('unhandledRejection', (reason: any) => {
    const error = reason instanceof Error ? reason : new Error(String(reason));
    Sentry.captureException(error, {
      tags: { error_type: 'unhandledRejection' },
      extra: { reason: String(reason) },
    });
    console.error('Unhandled Promise Rejection:', reason);
  });

  // Capture uncaught exceptions
  process.on('uncaughtException', (error: Error) => {
    Sentry.captureException(error, {
      tags: { error_type: 'uncaughtException' },
    });
    console.error('Uncaught Exception:', error);
    // Exit after capturing - uncaught exceptions are fatal
    process.exit(1);
  });
}

// Re-export Sentry for direct use in catch blocks
export { Sentry };
```

### 2. Initialize in Your Entry Point

**CRITICAL: Initialize Sentry BEFORE any other imports**

```typescript
// src/index.ts

// Initialize Sentry FIRST (before any other imports that might throw)
import { initializeSentry, setupProcessErrorHandlers } from "./shared/sentry";
import * as Sentry from '@sentry/node';
initializeSentry("your-service-name");
setupProcessErrorHandlers();

// Now import everything else
import express from "express";
// ... rest of your imports
```

**Why initialize first?**
- Errors during module loading will be captured
- Sentry must be ready before any code runs
- Process-level handlers must be set up early

---

## Express Integration

### Automatic Error Handling

The `expressIntegration()` automatically handles:
- Request tracking
- Performance monitoring
- Error capture for unhandled Express errors

**No manual handlers needed in Sentry v8!**

### Custom Error Middleware (Optional)

Add a custom error handler as a fallback:

```typescript
// After all routes, before 404 handler
app.use((err: Error, _req: express.Request, res: express.Response, _next: express.NextFunction) => {
  console.error("Unhandled error:", err);
  
  // Add additional context if Sentry didn't catch it
  Sentry.captureException(err, {
    tags: {
      error_path: _req.path,
      error_method: _req.method,
    },
    extra: {
      url: _req.originalUrl,
    },
  });
  
  res.status(500).json({
    error: "Internal Server Error",
    message: process.env.NODE_ENV === "development" ? err.message : "Something went wrong",
  });
});

// 404 handler (must be last)
app.use("*", (req, res) => {
  res.status(404).json({
    error: "Not Found",
    message: `Route ${req.method} ${req.originalUrl} not found`,
  });
});
```

---

## Error Capture Strategies

### 1. Capture in Try-Catch Blocks

**Always capture errors in catch blocks:**

```typescript
import * as Sentry from '@sentry/node';

try {
  await riskyOperation();
} catch (error) {
  const err = error instanceof Error ? error : new Error(String(error));
  
  Sentry.captureException(err, {
    tags: {
      operation: "operation-name",
      errorType: err.constructor.name,
    },
    extra: {
      userId: user.id,
      requestId: req.id,
      // ... any relevant context
    },
  });
  
  // Handle error response
  res.status(500).json({ error: err.message });
}
```

### 2. Capture in Controllers

```typescript
// src/api/controllers/UserController.ts
import * as Sentry from '@sentry/node';

export class UserController {
  async getUser(req: Request, res: Response): Promise<void> {
    try {
      const user = await this.userService.getUser(req.params.id);
      res.json(user);
    } catch (error) {
      const err = error instanceof Error ? error : new Error(String(error));
      
      Sentry.captureException(err, {
        tags: { operation: "getUser" },
        extra: {
          userId: req.params.id,
          path: req.path,
        },
      });
      
      res.status(500).json({ error: 'Failed to get user' });
    }
  }
}
```

### 3. Capture in Middleware

```typescript
// src/middleware/auth.ts
import * as Sentry from '@sentry/node';

export function authMiddleware(req: Request, res: Response, next: NextFunction) {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) {
      throw new Error('No authorization token');
    }
    // ... validate token
    next();
  } catch (error) {
    const err = error instanceof Error ? error : new Error(String(error));
    
    Sentry.captureException(err, {
      tags: { operation: "auth-middleware" },
      extra: {
        path: req.path,
        method: req.method,
      },
    });
    
    res.status(401).json({ error: 'Unauthorized' });
  }
}
```

### 4. Capture in Service Layer

```typescript
// src/services/UserService.ts
import * as Sentry from '@sentry/node';

export class UserService {
  async createUser(data: CreateUserData): Promise<User> {
    try {
      return await this.repository.create(data);
    } catch (error) {
      const err = error instanceof Error ? error : new Error(String(error));
      
      Sentry.captureException(err, {
        tags: { operation: "createUser" },
        extra: {
          email: data.email,
          // ... relevant context
        },
      });
      
      throw err; // Re-throw if needed
    }
  }
}
```

---

## Process-Level Error Handlers

Process-level handlers capture errors that escape your try-catch blocks:

```typescript
// Already included in setupProcessErrorHandlers()

// Unhandled Promise Rejections
process.on('unhandledRejection', (reason: any) => {
  const error = reason instanceof Error ? reason : new Error(String(reason));
  Sentry.captureException(error, {
    tags: { error_type: 'unhandledRejection' },
    extra: { reason: String(reason) },
  });
});

// Uncaught Exceptions
process.on('uncaughtException', (error: Error) => {
  Sentry.captureException(error, {
    tags: { error_type: 'uncaughtException' },
  });
  process.exit(1); // Fatal - must exit
});
```

---

## Configuration

### Environment Variables

```bash
# Required
SENTRY_DSN=https://your-dsn@sentry.io/project-id

# Optional
ENVIRONMENT=development|staging|production
RELEASE=1.0.0  # or git commit SHA
SENTRY_TRACES_SAMPLE_RATE=0.1  # 10% in production
SENTRY_PROFILES_SAMPLE_RATE=0.1  # 10% in production
SENTRY_DEBUG=true  # Enable debug logging
```

### Sample Rates

**Production:**
- `tracesSampleRate: 0.1` (10% of transactions)
- `profilesSampleRate: 0.1` (10% of profiled transactions)

**Development:**
- `tracesSampleRate: 1.0` (100% for debugging)
- `profilesSampleRate: 1.0` (100% for debugging)

**Why lower in production?**
- Reduces Sentry quota usage
- Lowers performance overhead
- 10% is usually sufficient for error detection

---

## Best Practices

### 1. Initialize Early

```typescript
// ✅ CORRECT - Initialize first
import { initializeSentry } from "./shared/sentry";
initializeSentry("service-name");
import express from "express";

// ❌ WRONG - Initialize after imports
import express from "express";
import { initializeSentry } from "./shared/sentry";
initializeSentry("service-name");
```

### 2. Always Capture in Catch Blocks

```typescript
// ✅ CORRECT - Capture exception
try {
  await operation();
} catch (error) {
  Sentry.captureException(error, { tags: {...}, extra: {...} });
  // Handle error
}

// ❌ WRONG - Error escapes uncaught
try {
  await operation();
} catch (error) {
  res.status(500).json({ error: error.message });
  // Error not captured!
}
```

### 3. Add Context with Tags and Extra

```typescript
Sentry.captureException(error, {
  tags: {
    operation: "operation-name",  // For filtering in Sentry
    errorType: error.constructor.name,
    userId: user.id,  // If relevant
  },
  extra: {
    requestId: req.id,
    path: req.path,
    method: req.method,
    // ... any additional context
  },
});
```

### 4. Use Tags for Filtering

Tags are indexed and searchable in Sentry:
- `operation`: Operation name (e.g., "getUser", "createOrder")
- `errorType`: Error class name
- `service`: Service name
- `environment`: Environment name

### 5. Don't Send PII

```typescript
Sentry.init({
  sendDefaultPii: false, // Don't send personally identifiable information
  // ...
});

// Don't include sensitive data in extra
Sentry.captureException(error, {
  extra: {
    // ✅ OK
    userId: "user-123",
    orderId: "order-456",
    
    // ❌ BAD - Contains PII
    email: "user@example.com",
    creditCard: "1234-5678-9012-3456",
    password: "secret123",
  },
});
```

### 6. Graceful Degradation

```typescript
export function initializeSentry(serviceName: string): void {
  if (!process.env.SENTRY_DSN) {
    console.warn('⚠️  SENTRY_DSN not found - Sentry monitoring disabled');
    return; // Continue without Sentry
  }
  
  try {
    Sentry.init({ /* ... */ });
  } catch (error) {
    console.error('❌ Failed to initialize Sentry:', error);
    // Continue without Sentry - graceful degradation
  }
}
```

---

## Complete Example

### Project Structure

```
your-project/
├── src/
│   ├── shared/
│   │   └── sentry.ts          # Sentry initialization
│   ├── index.ts                # Entry point
│   ├── api/
│   │   ├── controllers/
│   │   │   └── UserController.ts
│   │   └── routes/
│   │       └── users.ts
│   └── middleware/
│       └── auth.ts
└── package.json
```

### src/shared/sentry.ts

```typescript
import * as Sentry from '@sentry/node';
import { expressIntegration } from '@sentry/node';

let nodeProfilingIntegration: any;
try {
  const profilingModule = require('@sentry/profiling-node');
  nodeProfilingIntegration = profilingModule.nodeProfilingIntegration;
} catch {
  nodeProfilingIntegration = null;
}

export function initializeSentry(serviceName: string = 'your-service'): void {
  const sentryDsn = process.env.SENTRY_DSN;
  
  if (!sentryDsn) {
    console.warn('⚠️  SENTRY_DSN not found - Sentry monitoring disabled');
    return;
  }

  try {
    const integrations: any[] = [expressIntegration()];
    if (nodeProfilingIntegration) {
      integrations.push(nodeProfilingIntegration());
    }

    const defaultTracesSampleRate = process.env.NODE_ENV === 'production' ? '0.1' : '1.0';
    const defaultProfilesSampleRate = process.env.NODE_ENV === 'production' ? '0.1' : '1.0';

    Sentry.init({
      dsn: sentryDsn,
      environment: process.env.ENVIRONMENT || process.env.NODE_ENV || 'development',
      tracesSampleRate: parseFloat(process.env.SENTRY_TRACES_SAMPLE_RATE || defaultTracesSampleRate),
      profilesSampleRate: nodeProfilingIntegration ? parseFloat(process.env.SENTRY_PROFILES_SAMPLE_RATE || defaultProfilesSampleRate) : undefined,
      integrations,
      sendDefaultPii: false,
      release: process.env.RELEASE || '1.0.0',
      debug: process.env.SENTRY_DEBUG === 'true',
    });

    Sentry.setContext('service', {
      name: serviceName,
      version: process.env.RELEASE || '1.0.0',
    });

    console.log(`✅ Sentry initialized for ${serviceName}`);
  } catch (error) {
    console.error('❌ Failed to initialize Sentry:', error);
  }
}

export function setupProcessErrorHandlers(): void {
  process.on('unhandledRejection', (reason: any) => {
    const error = reason instanceof Error ? reason : new Error(String(reason));
    Sentry.captureException(error, {
      tags: { error_type: 'unhandledRejection' },
      extra: { reason: String(reason) },
    });
    console.error('Unhandled Promise Rejection:', reason);
  });

  process.on('uncaughtException', (error: Error) => {
    Sentry.captureException(error, {
      tags: { error_type: 'uncaughtException' },
    });
    console.error('Uncaught Exception:', error);
    process.exit(1);
  });
}

export { Sentry };
```

### src/index.ts

```typescript
// Initialize Sentry FIRST
import { initializeSentry, setupProcessErrorHandlers } from "./shared/sentry";
import * as Sentry from '@sentry/node';
initializeSentry("your-service");
setupProcessErrorHandlers();

import express from "express";
import { userRoutes } from "./api/routes/users";

const app = express();
app.use(express.json());

app.use("/users", userRoutes);

// Error handler (after routes, before 404)
app.use((err: Error, _req: express.Request, res: express.Response, _next: express.NextFunction) => {
  Sentry.captureException(err, {
    tags: {
      error_path: _req.path,
      error_method: _req.method,
    },
    extra: {
      url: _req.originalUrl,
    },
  });
  
  res.status(500).json({
    error: "Internal Server Error",
    message: process.env.NODE_ENV === "development" ? err.message : "Something went wrong",
  });
});

// 404 handler
app.use("*", (req, res) => {
  res.status(404).json({
    error: "Not Found",
    message: `Route ${req.method} ${req.originalUrl} not found`,
  });
});

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

### src/api/controllers/UserController.ts

```typescript
import { Request, Response } from "express";
import * as Sentry from '@sentry/node';

export class UserController {
  async getUser(req: Request, res: Response): Promise<void> {
    try {
      const user = await this.userService.getUser(req.params.id);
      res.json(user);
    } catch (error) {
      const err = error instanceof Error ? error : new Error(String(error));
      
      Sentry.captureException(err, {
        tags: { operation: "getUser" },
        extra: {
          userId: req.params.id,
          path: req.path,
        },
      });
      
      res.status(500).json({ error: 'Failed to get user' });
    }
  }
}
```

---

## Verification

### 1. Check Initialization

Look for this in your logs:
```
✅ Sentry DSN confirmed - initialized for your-service
   Environment: development, Release: 1.0.0, DSN: https://...
🔍 Sentry initialized for error monitoring and performance tracking
```

### 2. Test Error Capture

Trigger a test error:
```typescript
// In a route handler
throw new Error("Test error for Sentry");
```

### 3. Check Sentry Dashboard

- Go to your Sentry project
- Filter by environment: `development`
- Look for the test error
- Verify it has:
  - Stack trace
  - Source file location
  - Tags and extra context

---

## Troubleshooting

### Errors Not Appearing in Sentry

1. **Check SENTRY_DSN is set**
   ```bash
   echo $SENTRY_DSN
   ```

2. **Check initialization order**
   - Sentry must be initialized before other imports

3. **Check error is actually thrown**
   - 404 responses don't throw exceptions
   - Validation errors might be handled gracefully

4. **Enable debug mode**
   ```bash
   SENTRY_DEBUG=true npm start
   ```

### Performance Impact

- Sample rates control overhead
- 10% in production is usually sufficient
- Profiling adds minimal overhead at 10% rate

---

## Summary

✅ **Do:**
- Initialize Sentry before other imports
- Capture errors in all catch blocks
- Add context with tags and extra data
- Use process-level error handlers
- Set appropriate sample rates

❌ **Don't:**
- Initialize Sentry after other imports
- Skip error capture in catch blocks
- Send PII (personally identifiable information)
- Use 100% sample rates in production
- Forget graceful degradation

---

## Additional Resources

- [Sentry Node.js Documentation](https://docs.sentry.io/platforms/javascript/guides/node/)
- [Sentry Express Integration](https://docs.sentry.io/platforms/javascript/guides/node/guides/express/)
- [Sentry Performance Monitoring](https://docs.sentry.io/product/performance/)

---

**Happy Error Monitoring! 🎉**

