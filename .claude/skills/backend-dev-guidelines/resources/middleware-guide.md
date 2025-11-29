# Middleware Guide - Hono Middleware Patterns

Complete guide to creating and using middleware in backend microservices.

## Table of Contents

- [Authentication Middleware](#authentication-middleware)
- [Audit Middleware with AsyncLocalStorage](#audit-middleware-with-asynclocalstorage)
- [Error Boundary Middleware](#error-boundary-middleware)
- [Validation Middleware](#validation-middleware)
- [Composable Middleware](#composable-middleware)
- [Middleware Ordering](#middleware-ordering)

---

## Authentication Middleware

### SSOMiddleware Pattern

**File:** `/form/src/middleware/SSOMiddleware.ts`

```typescript
import { createMiddleware } from 'hono/factory';
import { Context } from 'hono';
import { getCookie } from 'hono/cookie';

export class SSOMiddlewareClient {
    static verifyLoginStatus = createMiddleware<{ Variables: { claims: any, effectiveUserId: string } }>(
        async (c: Context, next) => {
            // Get cookie using Hono's getCookie helper
            const token = getCookie(c, 'refresh_token');

            if (!token) {
                c.status(401);
                return c.json({ error: 'Not authenticated' });
            }

            try {
                const decoded = jwt.verify(token, config.tokens.jwt);
                c.set('claims', decoded);
                c.set('effectiveUserId', decoded.sub);
                // Continue to next middleware/handler
                await next();
            } catch (error) {
                c.status(401);
                return c.json({ error: 'Invalid token' });
            }
        }
    );
}
```

---

## Audit Middleware with AsyncLocalStorage

### Excellent Pattern from Blog API

**File:** `/form/src/middleware/auditMiddleware.ts`

```typescript
import { AsyncLocalStorage } from 'async_hooks';
import { createMiddleware } from 'hono/factory';
import { Context } from 'hono';

export interface AuditContext {
    userId: string;
    userName?: string;
    impersonatedBy?: string;
    sessionId?: string;
    timestamp: Date;
    requestId: string;
}

export const auditContextStorage = new AsyncLocalStorage<AuditContext>();

export const auditMiddleware = createMiddleware<{ Variables: { auditContext: AuditContext } }>(
    async (c: Context, next) => {
        const context: AuditContext = {
            userId: c.var.effectiveUserId || 'anonymous',
            userName: c.var.claims?.preferred_username,
            impersonatedBy: c.var.isImpersonating ? c.var.originalUserId : undefined,
            timestamp: new Date(),
            requestId: c.req.header('x-request-id') || uuidv4(),
        };

        c.set('auditContext', context);
        
        // Also store in AsyncLocalStorage for compatibility with services
        await auditContextStorage.run(context, async () => {
            await next();
        });
    }
);

// Getter for current context
export function getAuditContext(): AuditContext | null {
    return auditContextStorage.getStore() || null;
}
```

**Benefits:**
- Context propagates through entire request
- No need to pass context through every function
- Automatically available in services, repositories
- Type-safe context access via Hono's typed context variables
- AsyncLocalStorage for services that need automatic context

**Usage in Services:**
```typescript
import { getAuditContext } from '../middleware/auditMiddleware';

async function someOperation() {
    const context = getAuditContext();
    console.log('Operation by:', context?.userId);
}
```

---

## Error Boundary Middleware

### Comprehensive Error Handler

**File:** `/form/src/middleware/errorBoundary.ts`

```typescript
import { Context } from 'hono';
import { HTTPException } from 'hono/http-exception';
import * as Sentry from '@sentry/node';

// Use app.onError for global error handling
export const errorHandler = (err: Error, c: Context) => {
    // Determine status code
    const statusCode = getStatusCodeForError(err);

    // Capture to Sentry
    Sentry.withScope((scope) => {
        scope.setLevel(statusCode >= 500 ? 'error' : 'warning');
        scope.setTag('error_type', err.name);
        scope.setContext('error_details', {
            message: err.message,
            stack: err.stack,
        });
        Sentry.captureException(err);
    });

    // User-friendly response
    c.status(statusCode);
    return c.json({
        success: false,
        error: {
            message: getUserFriendlyMessage(err),
            code: err.name,
        },
        requestId: Sentry.getCurrentScope().getPropagationContext().traceId,
    });
};

// Usage in app setup:
// app.onError(errorHandler);

// Alternative: throw HTTPException directly in handlers
export function throwHttpError(statusCode: number, message: string) {
    throw new HTTPException(statusCode, {
        res: new Response(JSON.stringify({ error: message }), {
            status: statusCode,
            headers: { 'Content-Type': 'application/json' },
        }),
    });
}
```

---

## Composable Middleware

### withAuthAndAudit Pattern

```typescript
// In Hono, middleware is composed in route definitions directly
// Usage with multiple middleware
app.post('/:formID/submit',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (c) => controller.submit(c)
);

// Or create a helper for common middleware stacks
export function withAuthAndAudit(
    ...middlewares: any[]
) {
    return [...middlewares, auditMiddleware];
}

// Usage
app.post('/:formID/submit',
    ...withAuthAndAudit(SSOMiddlewareClient.verifyLoginStatus),
    async (c) => controller.submit(c)
);
```

---

## Middleware Ordering

### Critical Order (Must Follow)

```typescript
import { Hono } from 'hono';
import { logger } from 'hono/logger';
import * as Sentry from '@sentry/node';
import { errorHandler } from './middleware/errorBoundary';

const app = new Hono();

// 1. Global middleware (before routes)
// - Sentry request handler (FIRST)
app.use(Sentry.Handlers.requestHandler());

// - Request logging middleware
app.use(logger());

// - Auth middleware (cookies are auto-parsed by Hono)
app.use(SSOMiddlewareClient.verifyLoginStatus);

// - Audit middleware
app.use(auditMiddleware);

// 2. Routes registered here
app.route('/api/users', userRoutes);
app.route('/api/posts', postRoutes);

// 3. Global error handler (catches errors from routes and middleware)
app.onError(errorHandler);

// 4. Sentry error handler (AFTER onError so we can customize errors)
app.use(Sentry.Handlers.errorHandler());
```

**Key Points:**
- Hono automatically parses cookies (no separate middleware needed)
- Hono automatically parses JSON when you call `c.req.json()`
- Global middleware with `app.use()` applies to all routes
- Use `app.route()` to mount sub-applications for organized routing
- Use `app.onError()` for global error handling (called when handler/middleware throws)
- Middleware runs in order before `await next()`, then in reverse order after
- No separate body parsing middleware needed in Hono
- Error handlers are separate from middleware - they catch exceptions

---

**Related Files:**
- [SKILL.md](SKILL.md)
- [routing-and-controllers.md](routing-and-controllers.md)
- [async-and-errors.md](async-and-errors.md)
