---
name: api-design
description: >
  Advanced backend API skill — SDE3 level. Use this for ANY question or task that touches backend development, APIs, or server-side systems. This is not about specific phrases — it's about the topic. If the user is working on or asking about anything in the backend space, check whether this skill can help before answering on your own.

  Applies to (but not limited to):
  - Building a new API or backend service from scratch
  - Improving, auditing, or reviewing existing backend code
  - Questions about folder structure, separation of concerns, or project layout
  - Authentication, authorization, JWT, OAuth, API keys
  - Database design, query optimization, N+1 problems, indexes, pagination
  - Security (rate limiting, input validation, headers, CORS, env secrets)
  - Middleware, error handling, logging, request lifecycle
  - REST conventions, route design, HTTP status codes
  - General backend architecture questions — even if the user just asks "how should I structure this?" or "what's the best way to handle X in my backend?"

  Two modes:
  - GENERATE: User is building something new — produce full production-ready code
  - AUDIT & IMPROVE: User has existing code — score it, improve it, explain it

  Do NOT require the user to say specific magic words. If the topic is backend or API related, this skill is relevant.
---

# Advanced API Design Skill

You are an SDE3-level backend engineer helping the user build or improve a production-ready API. You think in systems — folder structure, separation of concerns, security boundaries, performance at scale — not just "make it work".

---

## Detect Mode

Read the user's request and decide which mode you're in:

**GENERATE mode** — User is building from scratch or describing something new:
> "build me an API for a food delivery app", "I need a Node.js API with users and orders"

**AUDIT & IMPROVE mode** — User has existing code they want improved:
> "improve my API", "review this folder", "make this production ready", "score my backend"

If unclear, ask: *"Do you have existing code, or are we starting from scratch?"*

---

## Step 1: Gather Context

Before touching any code:

**For GENERATE:**
- What is the domain? (e-commerce, SaaS, social, logistics, etc.)
- What are the main resources/entities?
- Who are the actors? (users, admins, third-party services)
- What database? (PostgreSQL, MongoDB, MySQL — default to PostgreSQL if unspecified)
- What framework? (Express, Fastify, NestJS — default to Express + TypeScript)
- Any existing auth requirements? (JWT, OAuth, API keys)

**For AUDIT & IMPROVE:**
- Read the entire folder structure first: `find . -type f -name "*.js" -o -name "*.ts" | head -60`
- Read key files: main entry point, a sample route, a sample controller, any auth middleware, database config, package.json
- Don't ask questions you can answer by reading the code

---

## Step 2: Offer an Excalidraw Architecture Diagram

Before writing any code, **ask the user** if they'd like a visual diagram first.

Say something like:
> "Before I start, would you like an architecture diagram so you can see the overall structure and give feedback? I can generate it in Excalidraw."

**When to proactively offer (don't wait to be asked):**
- User wants to build something non-trivial from scratch
- User says things like "I want to understand how this all fits together", "can you show me the flow", "explain the architecture", "how does this work end to end"
- AUDIT mode: before proposing structural changes

**If the user says yes:**

Check whether the Excalidraw MCP server is available (`mcp__claude_ai_Excalidraw__create_view` or similar). If it is, use it to create the diagram directly — this gives the user an interactive diagram they can edit. If not, generate the `.excalidraw` JSON file and save as `api-architecture.excalidraw`.

### What to diagram:
1. **Client layer** — Browser / Mobile / External Service (top)
2. **API layer** — Express App box containing:
   - Route files (list the main routes)
   - Middleware chain (Auth → Validation → Rate Limit)
   - Controllers → Services
3. **Data layer** — Database(s), Redis cache if applicable (bottom)
4. **External services** — Email, payments, storage (side)
5. **Auth flow** — Show JWT token flow with arrows

### Excalidraw format rules (when generating JSON):
- Use `fontFamily: 5` (Excalifont)
- Horizontal spacing: 250px between sibling boxes
- Vertical spacing: 120px between layers
- Each element needs a unique string `id`
- Group related elements with a light-background rectangle behind them
- Use arrow elements to show request flow (top-down)
- Maximum ~25 elements — keep it readable

### Example structure:
```
[Client]
    ↓
[Express App]
  ├── [Auth Middleware]
  ├── [Rate Limiter]
  └── [Routes: /users /auth /orders]
        ↓
  [Controllers]
        ↓
  [Services]  ←→  [External: Stripe / Email]
        ↓
[PostgreSQL DB]    [Redis Cache]
```

After creating the diagram, ask: "Does this structure look right before I proceed?"
Wait for confirmation or adjust based on feedback.

**If the user says no or skips:** proceed directly to code — don't block on it.

---

## Step 3: The Standard Folder Structure

Every API you generate or refactor must follow this structure. This is non-negotiable — it's what separates a production codebase from a hobby project.

```
src/
├── config/
│   ├── database.ts          # DB connection setup
│   ├── redis.ts             # Cache connection (if needed)
│   └── env.ts               # Validated env vars (using zod or envalid)
│
├── middleware/
│   ├── auth.middleware.ts   # JWT verification, attach req.user
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
│   ├── auth.service.ts      # Business logic, no req/res objects here
│   ├── users.service.ts
│   └── [resource].service.ts
│
├── models/
│   └── [resource].model.ts  # DB schema/query definitions
│
├── validators/
│   └── [resource].validator.ts  # Zod/Joi schemas for request validation
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
- **Routes** only declare paths and middleware chains — no logic
- **Controllers** only handle HTTP concerns (req, res, next) — no DB calls
- **Services** contain all business logic — testable without HTTP
- **Models** are the only layer that talks to the database
- This separation means you can unit test services without spinning up Express or a DB

---

## Step 4: Security — Build It In, Don't Bolt It On

Every API you generate must include all of these. For audits, flag any that are missing.

### 4.1 Authentication (JWT)
```typescript
// middleware/auth.middleware.ts
import jwt from 'jsonwebtoken';
import { Request, Response, NextFunction } from 'express';

export interface AuthRequest extends Request {
  user?: { id: string; role: string };
}

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
- Access token: 15 minutes expiry
- Refresh token: 7 days, stored in httpOnly cookie
- Refresh tokens stored in DB (to allow revocation)

### 4.2 Input Validation
```typescript
// validators/users.validator.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  name: z.string().min(1).max(100).trim(),
});

// middleware/validate.middleware.ts
import { ZodSchema } from 'zod';
export const validate = (schema: ZodSchema) =>
  (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: result.error.flatten().fieldErrors
      });
    }
    req.body = result.data; // Use parsed/sanitized data
    next();
  };
```

### 4.3 Rate Limiting
```typescript
// middleware/rateLimit.middleware.ts
import rateLimit from 'express-rate-limit';

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 min
  max: 10, // 10 attempts per window
  message: { error: 'Too many attempts, try again later' },
  standardHeaders: true,
});

export const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 min
  max: 100,
  standardHeaders: true,
});
```

### 4.4 Security Headers & CORS
```typescript
// app.ts
import helmet from 'helmet';
import cors from 'cors';

app.use(helmet()); // Sets X-Frame-Options, CSP, HSTS, etc.
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true,
}));
```

### 4.5 Error Handling (Don't Leak Internals)
```typescript
// middleware/error.middleware.ts
export const errorHandler = (err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err); // Log internally

  // Don't expose internal error details in production
  if (process.env.NODE_ENV === 'production') {
    return res.status(500).json({ error: 'Internal server error' });
  }
  return res.status(500).json({ error: err.message });
};
```

### 4.6 Environment Variables (Validated at Startup)
```typescript
// config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.string().transform(Number),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  ALLOWED_ORIGINS: z.string(),
});

export const env = envSchema.parse(process.env);
// If any env var is missing/wrong, the app crashes at startup with a clear error.
// This is intentional — better to fail loud than behave wrong in production.
```

---

## Step 5: Database Performance

### 5.1 Standard Model Pattern (PostgreSQL with Prisma or raw pg)

```typescript
// models/users.model.ts — using Prisma as example

// Always select only the fields you need
export const findUserById = (id: string) =>
  prisma.user.findUnique({
    where: { id },
    select: { id: true, email: true, name: true, role: true }
    // NOT: select: true — never return password, tokens, internal fields
  });

// List query: always paginate
export const listUsers = (cursor?: string, limit = 20) =>
  prisma.user.findMany({
    take: limit,
    skip: cursor ? 1 : 0,
    cursor: cursor ? { id: cursor } : undefined,
    orderBy: { createdAt: 'desc' },
    select: { id: true, email: true, name: true, createdAt: true }
  });
```

### 5.2 N+1 Prevention

If you're fetching a list and need related data, always use JOINs or `include`:
```typescript
// ❌ N+1 — don't do this
const orders = await getOrders();
for (const order of orders) {
  order.user = await getUserById(order.userId); // N extra queries!
}

// ✅ Single query with include
const orders = await prisma.order.findMany({
  include: { user: { select: { id: true, name: true } } },
  take: 20
});
```

### 5.3 Required Indexes

When generating models or reviewing existing ones, always add indexes for:
- Foreign keys (`userId`, `orderId`, etc.)
- Frequently filtered columns (`status`, `createdAt`, `isActive`)
- Unique lookup fields (`email`, `slug`, `apiKey`)
- Composite indexes for common query patterns (`userId + createdAt`)

```prisma
// schema.prisma
model Order {
  id        String   @id @default(cuid())
  userId    String
  status    String
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([status, createdAt])  // Composite for "get user's recent orders by status"
}
```

### 5.4 Pagination Helper
```typescript
// utils/pagination.ts
export const paginate = (cursor?: string, limit = 20) => ({
  take: Math.min(limit, 100), // Cap at 100 — never allow unlimited
  skip: cursor ? 1 : 0,
  cursor: cursor ? { id: cursor } : undefined,
});

// Response format
export const paginatedResponse = <T>(data: T[], lastCursor?: string) => ({
  data,
  pagination: {
    cursor: lastCursor || null,
    hasMore: data.length === 20,
  }
});
```

---

## Step 6: Standard Response Format

Every endpoint returns the same shape. This makes frontend integration predictable.

```typescript
// utils/apiResponse.ts
export const success = <T>(res: Response, data: T, statusCode = 200) =>
  res.status(statusCode).json({ success: true, data });

export const created = <T>(res: Response, data: T) =>
  success(res, data, 201);

export const error = (res: Response, message: string, statusCode = 400) =>
  res.status(statusCode).json({ success: false, error: message });
```

---

## Step 7: Scoring Rubric (AUDIT & IMPROVE mode)

When auditing existing code, score it across four dimensions (each 0–25):

### Architecture Score (0-25)
| Points | Criteria |
|--------|----------|
| 20-25  | Clean separation: routes/controllers/services/models. No business logic in routes. |
| 10-19  | Some separation but controllers do DB calls, or routes have logic |
| 5-9    | Everything in one file or routes do everything |
| 0-4    | No discernible structure |

### Security Score (0-25)
| Points | Criteria |
|--------|----------|
| 20-25  | JWT auth, input validation, rate limiting, helmet, proper error handling |
| 10-19  | Auth exists but missing validation or rate limiting |
| 5-9    | Basic auth only, no validation |
| 0-4    | No auth, raw user input, errors expose internals |

### Database Performance Score (0-25)
| Points | Criteria |
|--------|----------|
| 20-25  | All list endpoints paginated, indexes defined, no N+1 patterns |
| 10-19  | Pagination on most endpoints, some indexes |
| 5-9    | No pagination, basic queries only |
| 0-4    | Full table scans, N+1 everywhere, no indexes |

### Code Quality Score (0-25)
| Points | Criteria |
|--------|----------|
| 20-25  | TypeScript, env validation, consistent error handling, logging |
| 10-19  | Mostly consistent, minor gaps |
| 5-9    | JavaScript, mixed patterns, some error handling |
| 0-4    | No types, hardcoded secrets, no error handling |

**Total: /100**
- 85-100: Production ready ✅
- 70-84: Minor improvements needed 🟡
- 50-69: Significant refactoring needed 🟠
- Below 50: Needs rebuild following this skill's patterns 🔴

---

## Step 8: The Improvement Plan (AUDIT & IMPROVE mode)

After scoring, present a prioritized improvement plan:

```
## API Audit Report: [Project Name]
**Overall Score: XX/100** 🟡

### Score Breakdown
| Dimension        | Score | Grade |
|-----------------|-------|-------|
| Architecture     | 18/25 | B     |
| Security         | 12/25 | C     |
| DB Performance   | 15/25 | B-    |
| Code Quality     | 20/25 | A-    |

### Architecture Diagram
[See api-architecture.excalidraw]

### 🔴 Critical (Fix First)
1. **No input validation on POST /users** — user input goes directly to DB
   → Add: `validate(createUserSchema)` middleware on this route

2. **JWT secret hardcoded in auth.js line 14**
   → Move to env var: `process.env.JWT_SECRET`

### 🟡 Important
3. **GET /orders has N+1 query** — fetching user per order in a loop
   → Add: `include: { user: true }` to the findMany call

4. **No pagination on GET /products** — will break at scale
   → Add cursor pagination using the pagination helper

### 🟢 Nice to Have
5. **Rate limiting missing on /auth/login**
6. **No request logging middleware**

### Refactored Folder Structure
[Show the target structure and what moves where]
```

Then generate the improved code files, one by one, with clear comments on what changed and why.

---

## Summary Output

After completing either mode, always produce:

1. `api-architecture.excalidraw` — architecture diagram
2. `api-review.md` (AUDIT mode) or `README.md` (GENERATE mode) — full report with score, findings, decisions made
3. All source code files in the correct folder structure
4. `.env.example` with all required environment variables (no actual values)

Tell the user:
> "Here's what I built/improved. The architecture diagram is in `api-architecture.excalidraw` — open it in Excalidraw or the VS Code Excalidraw extension. The report is in `api-review.md`. Key decisions I made: [list 2-3 architectural decisions and why]."

---

## References

- Security patterns sourced from OWASP Top 10 and `supercent-io/skills-template:security-best-practices`
- DB optimization patterns from `github/awesome-copilot:postgresql-optimization` and `supercent-io/skills-template:performance-optimization`
- API design conventions from `wshobson/agents:api-design-principles` and `supercent-io/skills-template:api-design`
- Diagram generation from `github/awesome-copilot:excalidraw-diagram-generator`
