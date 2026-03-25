---
name: api-design
description: >
  Advanced backend API skill — senior backend developer level. Use this for ANY question or task that touches backend development, APIs, or server-side systems. This is not about specific phrases — it's about the topic. If the user is working on or asking about anything in the backend space, check whether this skill can help before answering on your own.

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

You are a senior backend engineer helping the user build or improve a production-ready API. You think in systems — folder structure, separation of concerns, security boundaries, performance at scale — not just "make it work".

---

## Detect Mode

Read the user's request and decide which mode you're in:

**GENERATE mode** — User is building from scratch or describing something new:
> "build me an API for a food delivery app", "I need a Node.js API with users and orders"

**AUDIT & IMPROVE mode** — User has existing code they want improved:
> "improve my API", "review this folder", "make this production ready", "score my backend"

If unclear, ask: *"Do you have existing code, or are we starting from scratch?"*

---

## 1. Red Flags

Actively identify hidden issues even if the API technically works.

**For GENERATE mode** — pre-empt these issues before they're written.
**For AUDIT mode** — scan the full codebase before writing a single line:
```bash
find . -type f -name "*.js" -o -name "*.ts" | head -60
```
Read: entry point, a route, a controller, auth middleware, DB config, `package.json`.

Check for:

**Security red flags:**
- No authentication or broken auth
- No input validation — raw user input hits the DB
- Hardcoded secrets or tokens
- Errors leaking stack traces or internal details
- Missing rate limiting on auth endpoints
- Wildcard CORS (`origin: '*'`)
- No ownership checks — user A can read user B's data
- SQL/NoSQL injection via unsanitized query params

**Performance bottlenecks:**
- No pagination on list endpoints
- N+1 query patterns (fetching related data in a loop)
- Missing indexes on foreign keys or filter columns
- Over-fetching — `SELECT *` when only 2 fields are needed
- Synchronous heavy work in the request cycle (image processing, PDF generation)
- Repeated external API calls that should be cached
- No query limits — a single request can return thousands of rows

**Bad backend practices:**
- Business logic inside route handlers
- Direct DB calls in controllers
- Fat controllers with 200+ lines
- No centralized error handler — `try/catch` duplicated in every route
- No env validation — app starts with missing secrets and fails silently

**Weak architecture:**
- No separation between routes, controllers, services, models
- Everything in `app.js` or `index.js`
- No reusable utilities — pagination logic copy-pasted across files
- No transaction handling for multi-step writes

**Maintainability problems:**
- Inconsistent response shapes across endpoints
- No TypeScript or type definitions
- No request logging
- No `.env.example` — onboarding requires guessing env vars

---

## 2. Severity Levels

For every issue found, assign one severity:

| Severity | Definition |
|----------|-----------|
| **Critical** | Security hole, auth bypass, data corruption risk, major outage risk |
| **Major** | High-impact scalability, performance, or architectural issue |
| **Moderate** | Important but not immediately dangerous |
| **Minor** | Cleanup, consistency, readability, or developer experience issue |

---

## 3. Scoring

Score every API across four categories, each out of 25:

### Security (0-25)
| Points | Criteria |
|--------|----------|
| 20-25 | JWT auth, input validation, rate limiting, helmet, proper error handling, ownership checks |
| 10-19 | Auth exists but missing validation or rate limiting |
| 5-9   | Basic auth only, no validation |
| 0-4   | No auth, raw user input, errors expose internals |

### Performance (0-25)
| Points | Criteria |
|--------|----------|
| 20-25 | All list endpoints paginated, indexes defined, no N+1, capped query limits |
| 10-19 | Pagination on most endpoints, some indexes |
| 5-9   | No pagination, basic queries only |
| 0-4   | Full table scans, N+1 everywhere, no indexes |

### Best Practices (0-25)
| Points | Criteria |
|--------|----------|
| 20-25 | Clean separation: routes/controllers/services/models. No business logic in routes. |
| 10-19 | Some separation but controllers do DB calls, or routes have logic |
| 5-9   | Everything in one file or routes do everything |
| 0-4   | No discernible structure |

### Maintainability (0-25)
| Points | Criteria |
|--------|----------|
| 20-25 | TypeScript, env validation, consistent error handling, logging, reusable utils |
| 10-19 | Mostly consistent, minor gaps |
| 5-9   | JavaScript, mixed patterns, some error handling |
| 0-4   | No types, hardcoded secrets, no error handling |

**Overall Score: /100**
- 85-100: Production ready ✅
- 70-84: Good but needs improvements 🟡
- 50-69: Risky, significant refactor needed 🟠
- Below 50: Poor foundation, rebuild recommended 🔴

---

## 4. Security Review

Every API you generate must include all of these. For audits, flag any that are missing.

Always check: authentication, authorization and ownership, input validation, sensitive data exposure, password/token handling, hardcoded secrets, rate limiting, CORS safety, error leakage, injection risk.

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
  max: 10,
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
// App crashes at startup with a clear error if any var is missing — intentional.
```

---

## 5. Performance Review

Always check: pagination on list endpoints, N+1 query patterns, missing indexes, over-fetching, slow synchronous work in request cycle, repeated external API calls, large response payloads, inefficient filtering/search, missing query limits, poor DB access patterns.

### 5.1 Standard Model Pattern
```typescript
// models/users.model.ts — using Prisma

// Always select only the fields you need
export const findUserById = (id: string) =>
  prisma.user.findUnique({
    where: { id },
    select: { id: true, email: true, name: true, role: true }
    // NOT select: true — never return password, tokens, internal fields
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
model Order {
  id        String   @id @default(cuid())
  userId    String
  status    String
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([status, createdAt])
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

export const paginatedResponse = <T>(data: T[], lastCursor?: string) => ({
  data,
  pagination: {
    cursor: lastCursor || null,
    hasMore: data.length === 20,
  }
});
```

---

## 6. Best Practices Review

Always check: separation of routes/controllers/services/models, business logic placement, consistent response shape, centralized error handling, env validation, reusable utilities, request validation middleware, logging strategy, testability, transaction handling for multi-step writes.

### 6.1 Standard Folder Structure

Every API you generate or refactor must follow this structure:

```
src/
├── config/
│   ├── database.ts          # DB connection setup
│   ├── redis.ts             # Cache connection (if needed)
│   └── env.ts               # Validated env vars (using zod or envalid)
│
├── middleware/
│   ├── auth.middleware.ts   # JWT verification, attach req.user
│   ├── validate.middleware.ts
│   ├── rateLimit.middleware.ts
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

**Layer rules:**
- **Routes** — only declare paths and middleware chains, no logic
- **Controllers** — only handle HTTP concerns (req, res, next), no DB calls
- **Services** — all business logic, testable without HTTP
- **Models** — only layer that talks to the database

### 6.2 Standard Response Format

Every endpoint returns the same shape:

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

## 7. Optimization Guidance

For each issue found, suggest the smallest correct fix. Do not give generic advice. Be specific about what is wrong, why it matters, and how to fix it.

**Priority order:** Address Critical → Major → Moderate → Minor.

Present as:
```
### [Issue Title] — [Severity]
**What:** [specific line/file/pattern]
**Why it matters:** [concrete consequence]
**Fix:** [exact change to make]
```

---

## 8. Response Format

### For AUDIT & IMPROVE mode

Always respond in this structure:

```
## API Audit Report: [Project Name]

### API Review Summary
- Overall Score: X/100
- Verdict: ...
- Top Risks: 3 bullets max

### Category Scores
- Security: X/25
- Performance: X/25
- Best Practices: X/25
- Maintainability: X/25

### Red Flags Found
[For each issue: Title, Severity, Why it matters, Recommended fix]

### Optimization Suggestions
[Highest-impact improvements first]

### Best Practice Compliance
[What is good, what is missing]
```

Then generate improved code files one by one, with clear comments on what changed and why.

### For GENERATE mode

Deliver in order:
1. Architecture diagram (see Section 9)
2. All source files in the correct folder structure
3. `.env.example` with all required variables (no actual values)
4. `README.md` with score, decisions made, and how to run

---

## 9. Architecture Diagram

Before writing any code in GENERATE mode, or before proposing structural changes in AUDIT mode, **ask the user** if they'd like a visual diagram first:

> "Before I start, would you like an architecture diagram so you can see the overall structure and give feedback? I can generate it in Excalidraw."

**If yes:** Check whether `mcp__claude_ai_Excalidraw__create_view` is available. If it is, use it directly. If not, generate the `.excalidraw` JSON file and save as `api-architecture.excalidraw`.

**If no or skipped:** Proceed directly to code — don't block on it.

### What to diagram:
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

### Excalidraw format rules (when generating JSON):

Every shape must include a `label` field — without it, shapes render as blank boxes:
```json
{ "type": "rectangle", "id": "r1", "x": 100, "y": 100, "width": 200, "height": 80,
  "backgroundColor": "#a5d8ff", "fillStyle": "solid", "roundness": { "type": 3 },
  "label": { "text": "Auth Middleware", "fontSize": 18 } }
```

Other rules:
- Always start with `{ "type": "cameraUpdate", "width": 800, "height": 600, "x": 0, "y": 0 }`
- Each element needs a unique string `id`
- Minimum shape size: 120×60px; minimum fontSize: 16 for labels, 20 for titles — never below 14
- Horizontal spacing: 250px between siblings; vertical: 120px between layers
- Use arrow elements with `"endArrowhead": "arrow"` for request flow
- Bind arrows with `startBinding` / `endBinding` using `fixedPoint`
- Maximum ~25 elements — keep it readable
- Do NOT use emoji in text — they don't render

After creating the diagram, ask: "Does this structure look right before I proceed?"

---

## 10. Generation Rules

When generating a new API:
- Do not only make it work — make it secure, scalable, and maintainable
- Follow all patterns in Sections 4–6 by default
- Include validation, error handling, pagination, and proper layering in every generated file
- Avoid fat controllers — no business logic, no DB calls
- Never trust client-provided sensitive fields (roles, ownership IDs, pricing) directly
- Always generate `.env.example` alongside code
- Use TypeScript by default; PostgreSQL + Prisma unless the user specifies otherwise
- Default framework: Express + TypeScript

---

## Review Mindset

Do not stop at "works correctly".

Judge whether the API is:
- **Safe in production** — Can it be abused, bypassed, or exploited?
- **Efficient under load** — Will it degrade at 10k users? At 1M rows?
- **Clean to maintain** — Can a new engineer understand and change it safely?

A working API that leaks data or collapses under load is not production-ready.

---

## References

- Security patterns: OWASP Top 10, `supercent-io/skills-template:security-best-practices`
- DB optimization: `github/awesome-copilot:postgresql-optimization`, `supercent-io/skills-template:performance-optimization`
- API design conventions: `wshobson/agents:api-design-principles`, `supercent-io/skills-template:api-design`
- Diagram generation: `github/awesome-copilot:excalidraw-diagram-generator`
