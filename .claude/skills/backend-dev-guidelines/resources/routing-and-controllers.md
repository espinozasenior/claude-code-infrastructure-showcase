# Routing and Controllers - Best Practices

Complete guide to clean route definitions and controller patterns.

## Table of Contents

- [Routes: Routing Only](#routes-routing-only)
- [BaseController Pattern](#basecontroller-pattern)
- [Good Examples](#good-examples)
- [Anti-Patterns](#anti-patterns)
- [Refactoring Guide](#refactoring-guide)
- [Error Handling](#error-handling)
- [HTTP Status Codes](#http-status-codes)

---

## Routes: Routing Only

### The Golden Rule

**Routes should ONLY:**
- ✅ Define route paths
- ✅ Register middleware
- ✅ Delegate to controllers

**Routes should NEVER:**
- ❌ Contain business logic
- ❌ Access database directly
- ❌ Implement validation logic (use Zod + controller)
- ❌ Format complex responses
- ❌ Handle complex error scenarios

### Clean Route Pattern

```typescript
// routes/userRoutes.ts
import { Hono } from 'hono';
import { UserController } from '../controllers/UserController';
import { SSOMiddlewareClient } from '../middleware/SSOMiddleware';
import { auditMiddleware } from '../middleware/auditMiddleware';

const app = new Hono();
const controller = new UserController();

// ✅ CLEAN: Route definition only
app.get('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (c) => controller.getUser(c)
);

app.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (c) => controller.createUser(c)
);

app.put('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (c) => controller.updateUser(c)
);

export default app;
```

**Key Points:**
- Each route: method, path, middleware chain, controller delegation
- No try-catch needed (controller handles errors)
- Clean, readable, maintainable
- Easy to see all endpoints at a glance
- Use `c` (Context) instead of separate `req, res` objects

---

## BaseController Pattern

### Why BaseController?

**Benefits:**
- Consistent error handling across all controllers
- Automatic Sentry integration
- Standardized response formats
- Reusable helper methods
- Performance tracking utilities
- Logging and breadcrumb helpers

### BaseController Pattern (Template)

**File:** `/email/src/controllers/BaseController.ts`

```typescript
import * as Sentry from '@sentry/node';
import { Context } from 'hono';
import { HTTPException } from 'hono/http-exception';

export abstract class BaseController {
    /**
     * Handle errors with Sentry integration
     */
    protected handleError(
        c: Context,
        error: unknown,
        context: string,
        statusCode = 500
    ): Response {
        Sentry.withScope((scope) => {
            scope.setTag('controller', this.constructor.name);
            scope.setTag('operation', context);
            scope.setUser({ id: c.var?.claims?.userId });

            if (error instanceof Error) {
                scope.setContext('error_details', {
                    message: error.message,
                    stack: error.stack,
                });
            }

            Sentry.captureException(error);
        });

        c.status(statusCode);
        return c.json({
            success: false,
            error: {
                message: error instanceof Error ? error.message : 'An error occurred',
                code: statusCode,
            },
        });
    }

    /**
     * Handle success responses
     */
    protected handleSuccess<T>(
        c: Context,
        data: T,
        message?: string,
        statusCode = 200
    ): Response {
        c.status(statusCode);
        return c.json({
            success: true,
            message,
            data,
        });
    }

    /**
     * Performance tracking wrapper
     */
    protected async withTransaction<T>(
        name: string,
        operation: string,
        callback: () => Promise<T>
    ): Promise<T> {
        return await Sentry.startSpan(
            { name, op: operation },
            callback
        );
    }

    /**
     * Validate required fields
     */
    protected validateRequest(
        c: Context,
        required: string[],
        actual: Record<string, any>
    ): boolean {
        const missing = required.filter((field) => !actual[field]);

        if (missing.length > 0) {
            Sentry.captureMessage(
                `Missing required fields: ${missing.join(', ')}`,
                'warning'
            );

            throw new HTTPException(400, {
                res: c.json({
                    success: false,
                    error: {
                        message: 'Missing required fields',
                        code: 'VALIDATION_ERROR',
                        details: { missing },
                    },
                }),
            });
        }
        return true;
    }

    /**
     * Logging helpers
     */
    protected logInfo(message: string, context?: Record<string, any>): void {
        Sentry.addBreadcrumb({
            category: this.constructor.name,
            message,
            level: 'info',
            data: context,
        });
    }

    protected logWarning(message: string, context?: Record<string, any>): void {
        Sentry.captureMessage(message, {
            level: 'warning',
            tags: { controller: this.constructor.name },
            extra: context,
        });
    }

    /**
     * Add Sentry breadcrumb
     */
    protected addBreadcrumb(
        message: string,
        category: string,
        data?: Record<string, any>
    ): void {
        Sentry.addBreadcrumb({ message, category, level: 'info', data });
    }

    /**
     * Capture custom metric
     */
    protected captureMetric(name: string, value: number, unit: string): void {
        Sentry.metrics.gauge(name, value, { unit });
    }
}
```

### Using BaseController

```typescript
// controllers/UserController.ts
import { Context } from 'hono';
import { BaseController } from './BaseController';
import { UserService } from '../services/userService';
import { createUserSchema } from '../validators/userSchemas';

export class UserController extends BaseController {
    private userService: UserService;

    constructor() {
        super();
        this.userService = new UserService();
    }

    async getUser(c: Context): Promise<Response> {
        try {
            this.logInfo('Fetching user', { userId: c.req.param('id') });

            const user = await this.userService.findById(c.req.param('id'));

            if (!user) {
                return this.handleError(
                    c,
                    new Error('User not found'),
                    'getUser',
                    404
                );
            }

            return this.handleSuccess(c, user);
        } catch (error) {
            return this.handleError(c, error, 'getUser');
        }
    }

    async createUser(c: Context): Promise<Response> {
        try {
            // Validate input
            const validated = createUserSchema.parse(await c.req.json());

            // Track performance
            const user = await this.withTransaction(
                'user.create',
                'db.query',
                () => this.userService.create(validated)
            );

            return this.handleSuccess(c, user, 'User created successfully', 201);
        } catch (error) {
            return this.handleError(c, error, 'createUser');
        }
    }

    async updateUser(c: Context): Promise<Response> {
        try {
            const validated = updateUserSchema.parse(await c.req.json());
            const user = await this.userService.update(c.req.param('id'), validated);
            return this.handleSuccess(c, user, 'User updated');
        } catch (error) {
            return this.handleError(c, error, 'updateUser');
        }
    }
}
```

**Benefits:**
- Consistent error handling
- Automatic Sentry integration
- Performance tracking
- Clean, readable code
- Easy to test

---

## Good Examples

### Example 1: Email Notification Routes (Excellent ✅)

**File:** `/email/src/routes/notificationRoutes.ts`

```typescript
import { Hono } from 'hono';
import { NotificationController } from '../controllers/NotificationController';
import { SSOMiddlewareClient } from '../middleware/SSOMiddleware';

const app = new Hono();
const controller = new NotificationController();

// ✅ EXCELLENT: Clean delegation
app.get('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (c) => controller.getNotifications(c)
);

app.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (c) => controller.createNotification(c)
);

app.put('/:id/read',
    SSOMiddlewareClient.verifyLoginStatus,
    async (c) => controller.markAsRead(c)
);

export default app;
```

**What Makes This Excellent:**
- Zero business logic in routes
- Clear middleware chain
- Consistent pattern
- Easy to understand

### Example 2: Proxy Routes with Validation (Good ✅)

**File:** `/form/src/routes/proxyRoutes.ts`

```typescript
import { Hono } from 'hono';
import { z } from 'zod';

const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

app.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (c) => {
        try {
            const validated = createProxySchema.parse(await c.req.json());
            const proxy = await proxyService.createProxyRelationship(validated);
            c.status(201);
            return c.json({ success: true, data: proxy });
        } catch (error) {
            return handler.handleException(c, error);
        }
    }
);
```

**What Makes This Good:**
- Zod validation
- Delegates to service
- Proper HTTP status codes
- Error handling

**Could Be Better:**
- Move validation to controller
- Use BaseController

---

## Anti-Patterns

### Anti-Pattern 1: Business Logic in Routes (Bad ❌)

**File:** `/form/src/routes/responseRoutes.ts` (actual production code)

```typescript
// ❌ ANTI-PATTERN: 200+ lines of business logic in route
app.post('/:formID/submit', async (c) => {
    try {
        const username = c.var.claims.preferred_username;
        const body = await c.req.json();
        const responses = body.responses;
        const stepInstanceId = body.stepInstanceId;

        // ❌ Permission checking in route
        const userId = await userProfileService.getProfileByEmail(username).then(p => p.id);
        const canComplete = await permissionService.canCompleteStep(userId, stepInstanceId);
        if (!canComplete) {
            c.status(403);
            return c.json({ error: 'No permission' });
        }

        // ❌ Workflow logic in route
        const { createWorkflowEngine, CompleteStepCommand } = require('../workflow/core/WorkflowEngineV3');
        const engine = await createWorkflowEngine();
        const command = new CompleteStepCommand(
            stepInstanceId,
            userId,
            responses,
            additionalContext
        );
        const events = await engine.executeCommand(command);

        // ❌ Impersonation handling in route
        if (c.var.isImpersonating) {
            impersonationContextStore.storeContext(stepInstanceId, {
                originalUserId: c.var.originalUserId,
                effectiveUserId: userId,
            });
        }

        // ❌ Response processing in route
        const post = await PrismaService.main.post.findUnique({
            where: { id: postData.id },
            include: { comments: true },
        });

        // ❌ Permission check in route
        await checkPostPermissions(post, userId);

        // ... 100+ more lines of business logic

        return c.json({ success: true, data: result });
    } catch (e) {
        return handler.handleException(c, e);
    }
});
```

**Why This Is Terrible:**
- 200+ lines of business logic
- Hard to test (requires HTTP mocking)
- Hard to reuse (tied to route)
- Mixed responsibilities
- Difficult to debug
- Performance tracking difficult

### How to Refactor (Step-by-Step)

**Step 1: Create Controller**

```typescript
// controllers/PostController.ts
export class PostController extends BaseController {
    private postService: PostService;

    constructor() {
        super();
        this.postService = new PostService();
    }

    async createPost(req: Request, res: Response): Promise<void> {
        try {
            const validated = createPostSchema.parse({
                ...req.body,
            });

            const result = await this.postService.createPost(
                validated,
                res.locals.userId
            );

            this.handleSuccess(res, result, 'Post created successfully');
        } catch (error) {
            this.handleError(error, res, 'createPost');
        }
    }
}
```

**Step 2: Create Service**

```typescript
// services/postService.ts
export class PostService {
    async createPost(
        data: CreatePostDTO,
        userId: string
    ): Promise<PostResult> {
        // Permission check
        const canCreate = await permissionService.canCreatePost(userId);
        if (!canCreate) {
            throw new ForbiddenError('No permission to create post');
        }

        // Execute workflow
        const engine = await createWorkflowEngine();
        const command = new CompleteStepCommand(/* ... */);
        const events = await engine.executeCommand(command);

        // Handle impersonation if needed
        if (context.isImpersonating) {
            await this.handleImpersonation(data.stepInstanceId, context);
        }

        // Synchronize roles
        await this.synchronizeRoles(events, userId);

        return { events, success: true };
    }

    private async handleImpersonation(stepInstanceId: number, context: any) {
        impersonationContextStore.storeContext(stepInstanceId, {
            originalUserId: context.originalUserId,
            effectiveUserId: context.effectiveUserId,
        });
    }

    private async synchronizeRoles(events: WorkflowEvent[], userId: string) {
        // Role synchronization logic
    }
}
```

**Step 3: Update Route**

```typescript
// routes/postRoutes.ts
import { PostController } from '../controllers/PostController';

const router = Router();
const controller = new PostController();

// ✅ CLEAN: Just routing
router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.createPost(req, res)
);
```

**Result:**
- Route: 8 lines (was 200+)
- Controller: 25 lines (request handling)
- Service: 50 lines (business logic)
- Testable, reusable, maintainable!

---

## Error Handling

### Controller Error Handling

```typescript
async createUser(req: Request, res: Response): Promise<void> {
    try {
        const result = await this.userService.create(req.body);
        this.handleSuccess(res, result, 'User created', 201);
    } catch (error) {
        // BaseController.handleError automatically:
        // - Captures to Sentry with context
        // - Sets appropriate status code
        // - Returns formatted error response
        this.handleError(error, res, 'createUser');
    }
}
```

### Custom Error Status Codes

```typescript
async getUser(req: Request, res: Response): Promise<void> {
    try {
        const user = await this.userService.findById(req.params.id);

        if (!user) {
            // Custom 404 status
            return this.handleError(
                new Error('User not found'),
                res,
                'getUser',
                404  // Custom status code
            );
        }

        this.handleSuccess(res, user);
    } catch (error) {
        this.handleError(error, res, 'getUser');
    }
}
```

### Validation Errors

```typescript
async createUser(req: Request, res: Response): Promise<void> {
    try {
        const validated = createUserSchema.parse(req.body);
        const user = await this.userService.create(validated);
        this.handleSuccess(res, user, 'User created', 201);
    } catch (error) {
        // Zod errors get 400 status
        if (error instanceof z.ZodError) {
            return this.handleError(error, res, 'createUser', 400);
        }
        this.handleError(error, res, 'createUser');
    }
}
```

---

## HTTP Status Codes

### Standard Codes

| Code | Use Case | Example |
|------|----------|---------|
| 200 | Success (GET, PUT) | User retrieved, Updated |
| 201 | Created (POST) | User created |
| 204 | No Content (DELETE) | User deleted |
| 400 | Bad Request | Invalid input data |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | No permission |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource |
| 422 | Unprocessable Entity | Validation failed |
| 500 | Internal Server Error | Unexpected error |

### Usage Examples

```typescript
// 200 - Success (default)
this.handleSuccess(res, user);

// 201 - Created
this.handleSuccess(res, user, 'Created', 201);

// 400 - Bad Request
this.handleError(error, res, 'operation', 400);

// 404 - Not Found
this.handleError(new Error('Not found'), res, 'operation', 404);

// 403 - Forbidden
this.handleError(new ForbiddenError('No permission'), res, 'operation', 403);
```

---

## Refactoring Guide

### Identify Routes Needing Refactoring

**Red Flags:**
- Route file > 100 lines
- Multiple try-catch blocks in one route
- Direct database access (Prisma calls)
- Complex business logic (if statements, loops)
- Permission checks in routes

**Check your routes:**
```bash
# Find large route files
wc -l form/src/routes/*.ts | sort -n

# Find routes with Prisma usage
grep -r "PrismaService" form/src/routes/
```

### Refactoring Process

**1. Extract to Controller:**
```typescript
// Before: Route with logic
router.post('/action', async (req, res) => {
    try {
        // 50 lines of logic
    } catch (e) {
        handler.handleException(res, e);
    }
});

// After: Clean route
router.post('/action', (req, res) => controller.performAction(req, res));

// New controller method
async performAction(req: Request, res: Response): Promise<void> {
    try {
        const result = await this.service.performAction(req.body);
        this.handleSuccess(res, result);
    } catch (error) {
        this.handleError(error, res, 'performAction');
    }
}
```

**2. Extract to Service:**
```typescript
// Controller stays thin
async performAction(req: Request, res: Response): Promise<void> {
    try {
        const validated = actionSchema.parse(req.body);
        const result = await this.actionService.execute(validated);
        this.handleSuccess(res, result);
    } catch (error) {
        this.handleError(error, res, 'performAction');
    }
}

// Service contains business logic
export class ActionService {
    async execute(data: ActionDTO): Promise<Result> {
        // All business logic here
        // Permission checks
        // Database operations
        // Complex transformations
        return result;
    }
}
```

**3. Add Repository (if needed):**
```typescript
// Service calls repository
export class ActionService {
    constructor(private actionRepository: ActionRepository) {}

    async execute(data: ActionDTO): Promise<Result> {
        // Business logic
        const entity = await this.actionRepository.findById(data.id);
        // More logic
        return await this.actionRepository.update(data.id, changes);
    }
}

// Repository handles data access
export class ActionRepository {
    async findById(id: number): Promise<Entity | null> {
        return PrismaService.main.entity.findUnique({ where: { id } });
    }

    async update(id: number, data: Partial<Entity>): Promise<Entity> {
        return PrismaService.main.entity.update({ where: { id }, data });
    }
}
```

---

**Related Files:**
- [SKILL.md](SKILL.md) - Main guide
- [services-and-repositories.md](services-and-repositories.md) - Service layer details
- [complete-examples.md](complete-examples.md) - Full refactoring examples
