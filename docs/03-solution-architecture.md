# 03 - Solution Architecture

## How SourceShan is Designed


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Introduction

This document provides a comprehensive view of SourceShan's architecture. We'll explore the system design, component breakdown, data flow, and the reasoning behind key architectural choices.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Architectural Philosophy

Before diving into specifics, it's important to understand the guiding principles:

### 1. Edge-First Security
Authentication and authorization happen at the Edge (CDN level) before requests reach the server. This principle influences the entire architecture.

### 2. Layered Defense
Every layer assumes the previous layer might have been bypassed. Security is enforced at Edge, API, and database levels.

### 3. Schema as Truth
The database schema is the single source of truth for UI generation. Change the schema, the UI follows.

### 4. Atomic Operations
All state changes are atomic. Partial updates are never acceptable.

### 5. Serverless-Native
The architecture embraces serverless constraints rather than fighting them.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                     │
│                    (Browser / Mobile Browser)                            │
│                                                                          │
│   ┌────────────┐     ┌────────────┐     ┌────────────┐                  │
│   │   Admin    │     │   Client   │     │   Login    │                  │
│   │ Dashboard  │     │   Editor   │     │   Page     │                  │
│   └─────┬──────┘     └─────┬──────┘     └─────┬──────┘                  │
└─────────┼──────────────────┼──────────────────┼─────────────────────────┘
          │                  │                  │
          └──────────────────┴──────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         EDGE LAYER (Vercel Edge)                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                     middleware.ts                                  │  │
│  │                                                                   │  │
│  │  • JWT Verification (JOSE)          • Role-Based Access Control  │  │
│  │  • Automatic Token Refresh          • Route Protection           │  │
│  │  • Cookie Management                • Redirect Logic             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER (Node.js Runtime)                   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        API ROUTES                                │    │
│  │                                                                 │    │
│  │  /api/auth/*          /api/admin/*         /api/portfolio/*    │    │
│  │  ├── login            ├── users            ├── batch-save      │    │
│  │  ├── logout           └── [id]             ├── catalog         │    │
│  │  ├── refresh              ├── route.ts     ├── upload          │    │
│  │  └── profile              └── portfolios   └── section         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      SERVICE LAYER                               │    │
│  │                                                                 │    │
│  │  lib/auth.ts      lib/github.ts      lib/mongodb.ts            │    │
│  │  ├── signAccessToken  ├── getOctokit    ├── connectToDatabase  │    │
│  │  ├── signRefreshToken ├── getFileContent├── (singleton pool)   │    │
│  │  └── verifyToken      ├── batchCommit   │                       │    │
│  │                       └── deleteFile    │                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                       DATA LAYER                                 │    │
│  │                                                                 │    │
│  │  models/User.ts                                                 │    │
│  │  ├── username: String                                          │    │
│  │  ├── password: String (hashed)                                 │    │
│  │  ├── role: 'admin' | 'client'                                   │    │
│  │  └── portfolios: [{ id, name, path }]                          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
┌──────────────────────────┐   ┌──────────────────────────┐
│        MongoDB           │   │     GitHub Repos         │
│                          │   │                          │
│  Users Collection        │   │  Per-Client Repos        │
│  ├── _id                 │   │  ├── data/projects.json  │
│  ├── username            │   │  ├── data/about.json     │
│  ├── password (bcrypt)   │   │  ├── public/images/*     │
│  ├── role                │   │  └── (other assets)      │
│  └── portfolios[]        │   │                          │
│      ├── id              │   │  Accessed via:           │
│      ├── name            │   │  • GitHub App API        │
│      └── path (repo)     │   │  • Git Trees API         │
└──────────────────────────┘   └──────────────────────────┘
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Component Breakdown

### 1. Edge Layer (middleware.ts)

**Purpose:** Act as the first line of defense, verifying authentication and enforcing access control before any server-side code executes.

**Key Responsibilities:**
- JWT verification using JOSE (Edge-compatible)
- Automatic token refresh when access token expires
- Role-based route protection
- Redirect logic for unauthenticated users

**Technology Choice:** JOSE instead of jsonwebtoken because:
- jsonwebtoken uses Node.js APIs not available in Edge Runtime
- JOSE is designed for modern JavaScript environments including Edge

```typescript
// Conceptual flow in middleware
const { payload, newAccessToken } = await verifyOrRefreshToken(req);

if (!payload) {
  // Not authenticated - redirect or 401
  return redirectToLogin(req);
}

if (requiresAdmin(pathname) && payload.role !== 'admin') {
  // Wrong role - 403
  return forbiddenResponse();
}

// Authenticated and authorized - continue
return NextResponse.next();
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 2. API Routes

**Purpose:** Handle business logic and data operations for authenticated requests.

#### Authentication Routes (`/api/auth/*`)

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/auth/login` | POST | Validate credentials, issue tokens |
| `/api/auth/logout` | POST | Clear token cookies |
| `/api/auth/refresh` | POST | Issue new access token using refresh token |
| `/api/auth/profile/update` | POST | Update user profile (password, username) |

#### Admin Routes (`/api/admin/*`)

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/admin/users` | GET | List all users (admin only) |
| `/api/admin/users` | POST | Create new user (admin only) |
| `/api/admin/users/[id]` | PUT | Update user (admin only) |
| `/api/admin/users/[id]` | DELETE | Delete user (admin only) |
| `/api/admin/users/[id]/portfolios` | PUT | Assign portfolios to user |

#### Portfolio Routes (`/api/portfolio/*`)

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/portfolio/catalog` | GET | Get portfolio schema (catalog.json) |
| `/api/portfolio/section/[id]` | GET | Get specific section data |
| `/api/portfolio/batch-save/[id]` | PUT | Atomic save (data + images) |
| `/api/portfolio/upload` | POST | Pre-upload image blob to GitHub |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 3. Service Layer

#### `lib/auth.ts` - Token Management

```typescript
// Functions provided
signAccessToken(payload)   // Create 15-min access token
signRefreshToken(payload)  // Create 7-day refresh token
verifyAccessToken(token)   // Verify access token (Node.js)
verifyRefreshToken(token)  // Verify refresh token (Node.js)
```

**Why two verification systems?**
- Edge (middleware.ts) uses JOSE for verification
- Node.js (API routes) uses jsonwebtoken for verification + signing
- This split allows Edge optimization while using battle-tested library for signing


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


#### `lib/github.ts` - GitHub Integration

**Key Functions:**

| Function | Purpose |
|----------|---------|
| `getOctokit(owner, repo)` | Get authenticated Octokit instance |
| `getFileContent(owner, repo, path)` | Read file from repo |
| `updateFileContent(...)` | Update single file (simple API) |
| `deleteFile(owner, repo, path)` | Delete single file |
| `batchCommit(owner, repo, operations, message)` | Atomic multi-file operation |

**GitHub App vs. Personal Access Token:**
- Primary: GitHub App authentication (more secure, scoped per installation)
- Fallback: Personal Access Token (for development/debugging)

**Installation ID Caching:**
```typescript
const installationCache = new Map<string, { id: number; expiresAt: number }>();

// Cache installation IDs to reduce API calls
// Each repo lookup requires finding the installation ID
// Caching avoids repeated lookups within expiration window
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


#### `lib/mongodb.ts` - Connection Pooling

**The Problem:** Serverless = new instance per request = new connection per request = connection exhaustion.

**The Solution:** Global singleton pattern.

```typescript
// Simplified connection pooling pattern
declare global {
  var mongoose: { conn: Connection | null; promise: Promise<Connection> | null };
}

let cached = global.mongoose || { conn: null, promise: null };

async function connectToDatabase() {
  if (cached.conn) return cached.conn;
  
  if (!cached.promise) {
    cached.promise = mongoose.connect(MONGODB_URI);
  }
  
  cached.conn = await cached.promise;
  return cached.conn;
}
```

**Why this works:**
- `global` persists across invocations in the same Lambda instance
- First request creates connection
- Subsequent requests reuse it
- New Lambda instances create new connections (unavoidable)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 4. Frontend Architecture

#### Component Organization

```
src/components/
├── admin/                    # Admin-specific components
│   ├── ManageUsers.tsx       # User CRUD interface
│   ├── AdminPortfolios.tsx   # Portfolio management
│   └── PortfolioEditorModal.tsx
│
├── editor/                   # Editor components
│   ├── MainEditor.tsx        # Main editor container
│   └── form-engine/          # Dynamic form system
│       ├── DynamicForm.tsx
│       ├── FieldRenderer.tsx
│       ├── DetailedContentEditor.tsx
│       ├── BilingualListEditor.tsx
│       ├── StringArrayEditor.tsx
│       └── ImageInput.tsx
│
├── layout/                   # Layout components
│   ├── Header.tsx            # Navigation header
│   └── FooterWrapper.tsx     # Footer wrapper
│
├── common/                   # Shared components
│   └── SpaceBackground.tsx   # Animated background
│
└── ui/                       # UI primitives
    ├── Toast.tsx             # Toast notifications
    ├── Modal.tsx             # Confirmation modals
    └── ProfileModal.tsx      # Profile update modal
```

#### State Management

**Context API Usage:**

| Context | Purpose | Scope |
|---------|---------|-------|
| `UIContext` | Toasts, modals, profile state | Global |
| `PendingMediaContext` | Staged image uploads/deletions | Editor |

**Why Context API over Redux/Zustand?**
- App state is relatively simple
- Server Components handle most data fetching
- Context is sufficient for cross-component communication
- Less boilerplate, easier to reason about


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 5. Data Layer (MongoDB)

#### User Model

```typescript
interface IPortfolio {
  id: string;      // Unique identifier
  name: string;    // Display name
  path: string;    // GitHub repo path (owner/repo)
}

interface IUser {
  _id: ObjectId;
  username: string;
  password: string;  // Bcrypt hashed
  role: 'admin' | 'client';
  portfolios: IPortfolio[];
  createdAt?: Date;
  updatedAt?: Date;
}
```

**Key Design Decisions:**

1. **Embedded Portfolios:** Portfolios are embedded in User documents rather than a separate collection.
   - *Pro:* Single query gets user + portfolios
   - *Pro:* Atomic updates (user + portfolios in one operation)
   - *Con:* If portfolios grow large, document size increases

2. **Path as Reference:** The `path` field references a GitHub repo, not stored content.
   - MongoDB stores *who owns what*
   - GitHub stores *what the content is*
   - Clean separation of concerns


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Data Flow Diagrams

### Authentication Flow

```
┌────────┐     ┌──────────┐     ┌───────────┐     ┌──────────┐
│ Client │────►│ /login   │────►│ API Route │────►│ MongoDB  │
│        │     │ (page)   │     │ /api/auth │     │          │
└────────┘     └──────────┘     │ /login    │     └──────────┘
                                └─────┬─────┘
                                      │
    ┌─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Receive credentials (username, password)                │
│  2. Find user in MongoDB                                    │
│  3. Verify password with bcrypt.compare()                   │
│  4. Generate access token (15min) with jsonwebtoken         │
│  5. Generate refresh token (7d) with jsonwebtoken           │
│  6. Set tokens as HttpOnly cookies                          │
│  7. Return user role and portfolios                         │
└─────────────────────────────────────────────────────────────┘
```

### Protected Request Flow

```
┌────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│ Client │────►│   Edge    │────►│ API Route │────►│ Service   │
│        │     │ Middleware│     │           │     │ Layer     │
└────────┘     └─────┬─────┘     └───────────┘     └───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │ 1. Extract tokens     │
         │ 2. Verify with JOSE   │
         │ 3. If expired, try    │
         │    refresh token      │
         │ 4. If valid, attach   │
         │    user to request    │
         │ 5. Check route access │
         │ 6. Continue or reject │
         └───────────────────────┘
```

### Portfolio Save Flow (Batch Commit)

```
┌────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│ Editor │────►│ FormData  │────►│ batch-save│────►│ GitHub    │
│        │     │ Submit    │     │ API Route │     │ API       │
└────────┘     └───────────┘     └─────┬─────┘     └───────────┘
                                       │
    ┌──────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────────┐
│  1. Receive FormData (data JSON + images + deleted paths)     │
│  2. Process pending images with Sharp (WebP conversion)       │
│  3. Build operations array:                                   │
│     - CREATE for new images                                   │
│     - UPDATE for data.json                                    │
│     - DELETE for removed images                               │
│  4. Call batchCommit() which:                                 │
│     a. Gets current commit SHA                                │
│     b. Creates blobs for new/updated content                  │
│     c. Builds new tree with all changes                       │
│     d. Creates single commit referencing new tree             │
│     e. Updates branch ref to new commit                       │
│  5. Return success with commit SHA                            │
└────────────────────────────────────────────────────────────────┘
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Security Architecture

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: Edge Middleware                                       │
│  • JWT verification (JOSE)                                      │
│  • Token refresh                                                 │
│  • Route protection                                              │
│  "First line of defense - rejects invalid tokens at CDN"        │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2: API Route Verification                                │
│  • Re-verify token in sensitive operations                      │
│  • Additional validation                                        │
│  "Second check - don't trust middleware blindly"                │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3: Role-Based Access Control                             │
│  • Admin vs. Client roles                                       │
│  • Route-level permissions                                      │
│  "Right person, right action"                                   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 4: Data Scoping                                          │
│  • Queries scoped to user's portfolios                          │
│  • Ownership verification before operations                     │
│  "Users only see their own data"                                │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 5: HttpOnly Cookies                                      │
│  • Tokens stored in HttpOnly cookies                            │
│  • Not accessible to JavaScript                                 │
│  "Even XSS can't steal tokens"                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Token Strategy

| Token | Lifetime | Storage | Purpose |
|-------|----------|---------|---------|
| Access Token | 15 minutes | HttpOnly cookie | Short-lived for security |
| Refresh Token | 7 days | HttpOnly cookie | Long-lived for UX |

**Why this strategy?**
- If access token is somehow stolen, it expires quickly
- Refresh tokens enable long sessions without constant re-authentication
- HttpOnly cookies prevent JavaScript access (XSS protection)
- Silent refresh in middleware maintains seamless UX


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Scalability Considerations

### Current Architecture Limits

| Aspect | Current Approach | Limit | Mitigation |
|--------|-----------------|-------|------------|
| Users | Single MongoDB collection | ~10K users comfortably | Indexing on username, pagination |
| Portfolios per User | Embedded array | ~100 per user | Would need separate collection beyond |
| GitHub API | Rate limited (5000/hr for Apps) | Heavy concurrent use | Installation ID caching |
| Connection Pool | Global singleton | One per Lambda warm instance | Standard for serverless |

### Future Scaling Paths

1. **Read Replicas:** If read load increases, MongoDB read replicas
2. **Caching Layer:** Redis for frequently accessed schemas
3. **CDN for Images:** Serve images from CDN instead of GitHub raw
4. **Queue for Heavy Operations:** Background processing for large batch commits


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Technology Rationale Summary

| Technology | Why Chosen | Alternatives Considered |
|------------|------------|-------------------------|
| Next.js 16 | App Router, Edge support, unified build | Remix, Nuxt |
| JOSE | Edge Runtime compatible JWT | jsonwebtoken (Node.js only) |
| MongoDB | Flexible schema, embedding | PostgreSQL |
| GitHub API | Versioning, existing portfolios | S3, dedicated CMS |
| Framer Motion | Animation quality, Reorder component | react-spring, CSS |
| Tailwind CSS | Utility-first, rapid iteration | styled-components |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Summary

SourceShan's architecture is built on three pillars:

1. **Edge-First Security** - Authenticate and authorize before compute
2. **Schema-Driven Flexibility** - Let data structure drive UI
3. **Atomic Operations** - All changes succeed or none do

The architecture embraces serverless constraints through:
- Global connection pooling
- Edge-first authentication
- Stateless API routes
- External storage (GitHub) for content


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*[← Back to Problem Statement](02-problem-statement.md) | [Continue to Key Features →](04-key-features.md)*
