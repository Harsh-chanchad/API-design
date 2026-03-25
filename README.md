# api-design

A senior backend dev Claude skill for building and auditing production-ready backend APIs. Covers architecture, security, database performance, and code quality — not just "make it work."

---

## Table of Contents

- [Overview](#overview)
- [Modes](#modes)
- [Workflow](#workflow)
- [Folder Structure](#folder-structure)
- [Security Patterns](#security-patterns)
- [Database Performance](#database-performance)
- [Response Format](#response-format)
- [Audit Scoring Rubric](#audit-scoring-rubric)
- [Improvement Plan Format](#improvement-plan-format)
- [Outputs](#outputs)
- [Eval Scenarios](#eval-scenarios)
- [References](#references)

---

## Overview

This skill activates automatically for any backend or API-related task. It does not require magic trigger words — if the topic involves server-side development, this skill is relevant.

**Applies to:**
- Building a new API or backend service from scratch
- Improving, auditing, or reviewing existing backend code
- Folder structure, separation of concerns, and project layout
- Authentication, authorization (JWT, OAuth, API keys)
- Database design, query optimization, N+1 problems, indexes, pagination
- Security: rate limiting, input validation, headers, CORS, env secrets
- Middleware, error handling, logging, request lifecycle
- REST conventions, route design, HTTP status codes

**Default stack:** Express.js + TypeScript + PostgreSQL (via Prisma)

---

## Modes

### GENERATE
Triggered when the user is building from scratch.

> *"Build me an API for a food delivery app"*
> *"I need a Node.js backend with users and orders"*

Produces: full folder structure, all source files, `.env.example`, architecture diagram, README.

### AUDIT & IMPROVE
Triggered when the user has existing code.

> *"Review my API"*, *"Make this production ready"*, *"Score my backend"*

Produces: score breakdown (0–100), critical findings, refactored code, architecture diagram, `api-review.md`.

If unclear, Claude will ask: *"Do you have existing code, or are we starting from scratch?"*

---

## Workflow

### Step 1 — Gather Context

**GENERATE mode asks:**
- Domain (e-commerce, SaaS, logistics, social, etc.)
- Main resources/entities
- Actors (users, admins, third-party services)
- Database preference (default: PostgreSQL)
- Framework preference (default: Express + TypeScript)
- Auth requirements (JWT, OAuth, API keys)

**AUDIT mode reads first:**
- Full folder structure scan
- Entry point, sample route, sample controller, auth middleware, DB config, `package.json`
- Does not ask questions answerable from the code

---

### Step 2 — Architecture Diagram (Excalidraw)

Before writing code, Claude offers an Excalidraw architecture diagram for feedback. The diagram shows:

```
[Client: Browser / Mobile / External Service]
              ↓
       [Express App]
    ┌──────────────────────┐
    │  Auth Middleware      │
    │  Rate Limiter         │
    │  Routes: /users /auth │
    │  Controllers          │
    │  Services         ←→  │── [Stripe / Email / Storage]
    └──────────────────────┘
              ↓
   [PostgreSQL DB]   [Redis Cache]
```

The diagram is saved as `api-architecture.excalidraw` — openable in Excalidraw or the VS Code Excalidraw extension.

---

### Step 3 — Folder Structure

Every API generated or refactored follows this layout:

```
src/
├── config/
│   ├── database.ts          # DB connection setup
│   ├── redis.ts             # Cache connection (if needed)
│   └── env.ts               # Validated env vars (Zod)
│
├── middleware/
│   ├── auth.middleware.ts   # JWT verification, attaches req.user
│   ├── validate.middleware.ts  # Request body/query/param validation
│   ├── rateLimit.middleware.ts # Per-route rate limiting
│   ├── error.middleware.ts  # Global error handler
│   └── logger.middleware.ts # Request logging (morgan or pino)
│
├── routes/
│   ├── index.ts             # Mounts all routers
│   ├── auth.routes.ts
│   ├── users.routes.ts
│   └── [resource].routes.ts
│
├── controllers/
│   ├── auth.controller.ts   # Handles req/res, calls services
│   ├── users.controller.ts
│   └── [resource].controller.ts
│
├── services/
│   ├── auth.service.ts      # Business logic — no req/res objects
│   ├── users.service.ts
│   └── [resource].service.ts
│
├── models/
│   └── [resource].model.ts  # DB schema/query definitions
│
├── validators/
│   └── [resource].validator.ts  # Zod schemas
│
├── utils/
│   ├── jwt.ts               # Token sign/verify helpers
│   ├── hash.ts              # Password hashing (bcrypt)
│   ├── pagination.ts        # Cursor/offset pagination helpers
│   └── apiResponse.ts       # Standardized response format
│
├── types/
│   └── index.ts             # Shared TypeScript interfaces
│
└── app.ts                   # Express app setup (no listen() here)
index.ts                     # Entry point (listen())
```

**Why this structure:**

| Layer | Responsibility |
|-------|---------------|
| Routes | Declare paths and middleware chains only — no logic |
| Controllers | Handle HTTP concerns (req, res, next) — no DB calls |
| Services | Business logic — fully testable without HTTP or a DB |
| Models | The only layer that talks to the database |

---

## Security Patterns

All six patterns are included in every generated API. In AUDIT mode, missing patterns are flagged as findings.

### 1. JWT Authentication

```typescript
// middleware/auth.middleware.ts
export const authenticate = (req: AuthRequest, res: Response, next: NextFunction) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token provided' });

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as { id: string; role: string };
    req.user = payload;
    next();
  } catch {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
};

export const authorize = (...roles: string[]) =>
  (req: AuthRequest, res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
```

**Token strategy:**
- Access token: 15-minute expiry
- Refresh token: 7-day expiry, stored in `httpOnly` cookie
- Refresh tokens stored in DB (allows revocation)

---

### 2. Input Validation (Zod)

```typescript
// validators/users.validator.ts
export const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  name: z.string().min(1).max(100).trim(),
});

// middleware/validate.middleware.ts
export const validate = (schema: ZodSchema) =>
  (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: result.error.flatten().fieldErrors
      });
    }
    req.body = result.data; // Use sanitized data
    next();
  };
```

---

### 3. Rate Limiting

```typescript
// middleware/rateLimit.middleware.ts
export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 min
  max: 10,                   // 10 attempts per window
  message: { error: 'Too many attempts, try again later' },
  standardHeaders: true,
});

export const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 min
  max: 100,
  standardHeaders: true,
});
```

---

### 4. Security Headers & CORS

```typescript
// app.ts
app.use(helmet()); // Sets X-Frame-Options, CSP, HSTS, etc.
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true,
}));
```

---

### 5. Error Handling (No Internal Leaks)

```typescript
// middleware/error.middleware.ts
export const errorHandler = (err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err); // Log internally

  if (process.env.NODE_ENV === 'production') {
    return res.status(500).json({ error: 'Internal server error' });
  }
  return res.status(500).json({ error: err.message });
};
```

---

### 6. Environment Variable Validation

```typescript
// config/env.ts
const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.string().transform(Number),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  ALLOWED_ORIGINS: z.string(),
});

export const env = envSchema.parse(process.env);
// App crashes at startup if any env var is missing or invalid.
// Intentional — fail loud rather than behave wrong in production.
```

---

## Database Performance

### Selective Field Projection

Never return passwords, tokens, or internal fields from queries.

```typescript
export const findUserById = (id: string) =>
  prisma.user.findUnique({
    where: { id },
    select: { id: true, email: true, name: true, role: true }
    // NOT select: true — never return sensitive fields
  });
```

### Cursor Pagination

All list endpoints are paginated. Default limit: 20. Hard cap: 100.

```typescript
// utils/pagination.ts
export const paginate = (cursor?: string, limit = 20) => ({
  take: Math.min(limit, 100),
  skip: cursor ? 1 : 0,
  cursor: cursor ? { id: cursor } : undefined,
});

export const paginatedResponse = <T>(data: T[], lastCursor?: string) => ({
  data,
  pagination: {
    cursor: lastCursor || null,
    hasMore: data.length === 20,
  }
});
```

### N+1 Prevention

```typescript
// ❌ N+1 — one query per order
const orders = await getOrders();
for (const order of orders) {
  order.user = await getUserById(order.userId);
}

// ✅ Single query with JOIN
const orders = await prisma.order.findMany({
  include: { user: { select: { id: true, name: true } } },
  take: 20
});
```

### Index Recommendations

Always index:
- Foreign keys (`userId`, `orderId`)
- Frequently filtered columns (`status`, `createdAt`, `isActive`)
- Unique lookup fields (`email`, `slug`, `apiKey`)
- Composite indexes for common query patterns

```prisma
model Order {
  id        String   @id @default(cuid())
  userId    String
  status    String
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([status, createdAt])  // Composite: "get user's recent orders by status"
}
```

---

## Response Format

Every endpoint returns the same shape for predictable frontend integration.

```typescript
// utils/apiResponse.ts
export const success = <T>(res: Response, data: T, statusCode = 200) =>
  res.status(statusCode).json({ success: true, data });

export const created = <T>(res: Response, data: T) =>
  success(res, data, 201);

export const error = (res: Response, message: string, statusCode = 400) =>
  res.status(statusCode).json({ success: false, error: message });
```

**Success:**
```json
{ "success": true, "data": { ... } }
```

**Error:**
```json
{ "success": false, "error": "Validation failed", "details": { ... } }
```

---

## Audit Scoring Rubric

Code is scored across four dimensions (each 0–25):

### Architecture (0–25)

| Score | Criteria |
|-------|----------|
| 20–25 | Clean separation: routes → controllers → services → models. No business logic in routes. |
| 10–19 | Some separation but controllers do DB calls, or routes contain logic. |
| 5–9 | Everything in one file or routes handle everything. |
| 0–4 | No discernible structure. |

### Security (0–25)

| Score | Criteria |
|-------|----------|
| 20–25 | JWT auth, input validation, rate limiting, Helmet, safe error handling. |
| 10–19 | Auth exists but missing validation or rate limiting. |
| 5–9 | Basic auth only, no validation. |
| 0–4 | No auth, raw user input used in queries, errors expose stack traces. |

### Database Performance (0–25)

| Score | Criteria |
|-------|----------|
| 20–25 | All list endpoints paginated, indexes defined, no N+1 patterns. |
| 10–19 | Pagination on most endpoints, some indexes. |
| 5–9 | No pagination, basic queries only. |
| 0–4 | Full table scans, N+1 everywhere, no indexes. |

### Code Quality (0–25)

| Score | Criteria |
|-------|----------|
| 20–25 | TypeScript, env validation, consistent error handling, logging. |
| 10–19 | Mostly consistent, minor gaps. |
| 5–9 | JavaScript, mixed patterns, some error handling. |
| 0–4 | No types, hardcoded secrets, no error handling. |

### Overall

| Score | Status |
|-------|--------|
| 85–100 | Production ready |
| 70–84 | Minor improvements needed |
| 50–69 | Significant refactoring needed |
| < 50 | Needs rebuild |

---

## Improvement Plan Format

Audit reports follow this structure:

```
## API Audit Report: [Project Name]
Overall Score: XX/100

### Score Breakdown
| Dimension        | Score | Grade |
|-----------------|-------|-------|
| Architecture     | 18/25 | B     |
| Security         | 12/25 | C     |
| DB Performance   | 15/25 | B-    |
| Code Quality     | 20/25 | A-    |

### Critical (Fix First)
1. No input validation on POST /users — user input goes directly to DB
   Fix: Add validate(createUserSchema) middleware on this route

2. JWT secret hardcoded in auth.js line 14
   Fix: Move to env var — process.env.JWT_SECRET

### Important
3. GET /orders has N+1 query — fetching user per order in a loop
   Fix: Add include: { user: true } to the findMany call

4. No pagination on GET /products — will break at scale
   Fix: Add cursor pagination using the pagination helper

### Nice to Have
5. Rate limiting missing on /auth/login
6. No request logging middleware

### Refactored Folder Structure
[Target structure showing what moves where]
```

Followed by the full refactored source code with comments explaining what changed and why.

---

## Outputs

Both modes always produce:

| File | Description |
|------|-------------|
| `api-architecture.excalidraw` | Interactive architecture diagram |
| `README.md` *(GENERATE)* | Summary of decisions made |
| `api-review.md` *(AUDIT)* | Full score, findings, refactored code |
| `.env.example` | All required env vars with no actual values |
| `src/` | Full production-ready source tree |

---

## Eval Scenarios

### 1. Build from scratch — Food Delivery App

> *"I'm building a food delivery app. I need a Node.js + Express + TypeScript backend with users, restaurants, menu items, and orders. Users can browse restaurants, add items to cart, and place orders. Restaurants can manage their menus. Build me the full API structure with authentication, proper folder structure, and make it production-ready."*

**Expected output:** Full folder structure with routes/controllers/services/models, JWT auth middleware, Zod validation, rate limiting, Excalidraw diagram, `.env.example`, README.

---

### 2. Audit — Vulnerable single-file Express API

> *Single-file `server.js` with SQL injection via string interpolation, hardcoded JWT secret, N+1 query in `/orders`, `SELECT *` returning passwords, no pagination.*

**Expected output:** Score ~25–35/100, identification of SQL injection, hardcoded secret, N+1, missing pagination and auth middleware. Full refactored code in proper structure.

---

### 3. Audit — Multi-workspace SaaS API

> *Node.js API with workspaces, projects, tasks, and team members. Focus on workspace-level authorization (IDOR prevention), index recommendations, and preventing user enumeration.*

**Expected output:** Architecture diagram, score, IDOR vulnerability analysis, composite index recommendations (`status + assigneeId`, `workspaceId`), improved workspace-scoped auth middleware.

---

## References

- Security patterns — [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- DB optimization — PostgreSQL performance best practices
- API design conventions — REST naming and HTTP status code standards
- Diagram generation — Excalidraw JSON format specification
