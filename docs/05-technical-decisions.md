# 05 - Technical Decisions

## Architecture Decision Records


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Introduction

This document captures major technical decisions in an ADR (Architecture Decision Record) style format. Each decision includes context, options considered, the final decision, and consequences.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Decision 1: Edge-First Authentication with JOSE

### Status
**Accepted** ✅

### Context

Authentication is the foundation of the security model. Traditional approaches verify tokens in the application layer (Node.js runtime), meaning:
- Every request, valid or invalid, spins up a Lambda
- Invalid tokens still consume compute resources
- Authentication latency is tied to cold start times

Vercel offers Edge Middleware that runs at the CDN level before reaching the application.

### Options Considered

#### Option A: Traditional Route Protection
- Use Next.js `getServerSideProps` or API route checks
- **Pros:** Simple, familiar pattern
- **Cons:** Every request reaches Node.js, compute cost for invalid requests

#### Option B: NextAuth.js
- Use NextAuth with JWT strategy
- **Pros:** Full-featured, well-documented
- **Cons:** Heavy dependency, middleware support was limited at the time, less control

#### Option C: Custom Edge Middleware with JOSE
- Verify JWTs at the Edge using JOSE library
- **Pros:** Zero compute for invalid tokens, full control, lightweight
- **Cons:** More custom code, two JWT libraries needed (JOSE for Edge, jsonwebtoken for Node.js)

### Decision

**Option C: Custom Edge Middleware with JOSE**

The control and performance benefits outweighed the additional complexity. Invalid tokens are now rejected at the CDN level with zero server compute.

### Consequences

**Positive:**
- Invalid requests rejected at Edge (zero compute cost)
- Faster response times for valid requests
- Full control over token structure and refresh logic

**Negative:**
- Must maintain two JWT libraries (JOSE for Edge, jsonwebtoken for Node.js signing)
- Custom code means more maintenance responsibility
- Need to handle Edge Runtime limitations

**Risks & Mitigation:**
- Risk: JOSE updates could break compatibility
- Mitigation: Pin versions, comprehensive testing


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Decision 2: Dual-Token Authentication Strategy

### Status
**Accepted** ✅

### Context

Session management needs to balance:
- **Security:** Short-lived tokens minimize damage from theft
- **UX:** Long session lifetimes prevent constant re-authentication
- **Simplicity:** Fewer token types means simpler implementation

### Options Considered

#### Option A: Single Long-Lived Token
- One JWT with 7-day expiry
- **Pros:** Simple implementation
- **Cons:** If stolen, attacker has 7 days of access

#### Option B: Single Short-Lived Token + Database Sessions
- 15-minute JWT with server-side session storage
- **Pros:** Immediate revocation possible
- **Cons:** Database lookup on every request, complexity

#### Option C: Dual-Token (Access + Refresh)
- 15-minute access token + 7-day refresh token
- **Pros:** Short attack window, good UX, no database lookup for most requests
- **Cons:** More complex token management

### Decision

**Option C: Dual-Token Strategy**

This approach provides the best balance of security and user experience. The 15-minute access token limits exposure while the 7-day refresh token enables seamless sessions.

### Consequences

**Positive:**
- Attack window limited to 15 minutes for stolen access tokens
- Users stay logged in for 7 days without friction
- No database lookup needed for most authenticated requests

**Negative:**
- Two tokens to manage (issue, store, refresh)
- Silent refresh logic adds complexity
- More cookie management

**Implementation Details:**
- Both tokens stored as HttpOnly cookies
- Middleware handles silent refresh
- Refresh endpoint validates refresh token and issues new access token


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Decision 3: GitHub as Content Storage

### Status
**Accepted** ✅

### Context

Portfolio content (projects, images, JSON data) needs persistent storage. Options include:
- Traditional database (MongoDB, PostgreSQL)
- Cloud storage (S3, Cloudflare R2)
- Git repositories

Client portfolios are already deployed on Vercel from GitHub repos.

### Options Considered

#### Option A: MongoDB for Everything
- Store content as documents in MongoDB
- **Pros:** Single data store, familiar querying
- **Cons:** No version history, images need separate storage

#### Option B: S3 for Content
- JSON files and images in S3
- **Pros:** Scalable, cheap storage
- **Cons:** No version history, separate deployment trigger needed

#### Option C: GitHub Repositories
- JSON files and images in GitHub repos
- **Pros:** Version control, Vercel auto-deploys on push, one source of truth
- **Cons:** GitHub API complexity, rate limits

### Decision

**Option C: GitHub Repositories**

Portfolios already live in GitHub repos. Using GitHub as the content store means:
- Single source of truth (no sync issues)
- Free version history
- Automatic Vercel deployment on commit

### Consequences

**Positive:**
- Version history for free (Git history)
- Automatic portfolio deployment (Vercel watches repo)
- Rollback is simple (git revert)
- No additional storage costs

**Negative:**
- GitHub API is complex (especially for batch operations)
- Rate limits (5000 requests/hour for Apps)
- More latency than direct database queries

**Mitigation:**
- Installation ID caching reduces API calls
- Batch commits reduce request count
- Rate limit monitoring (not implemented yet)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Decision 4: Schema-Driven Form Generation

### Status
**Accepted** ✅

### Context

The editor needs to render forms for different portfolio structures. Portfolios have different fields (projects, skills, testimonials, etc.). Traditional approach: build a form component for each field type.

### Options Considered

#### Option A: Static Form Components
- Build a React component for each form type
- **Pros:** Full control, optimized per form
- **Cons:** New component for every new field, high maintenance

#### Option B: Use Form Library (React Hook Form + Zod)
- Generate forms from Zod schemas
- **Pros:** Well-tested, validation built-in
- **Cons:** Still need component mapping, another dependency

#### Option C: Custom Schema-Driven Engine
- Build a form engine that reads JSON schemas
- **Pros:** Add field = add to schema (no code), maximum flexibility
- **Cons:** Upfront investment, custom code to maintain

### Decision

**Option C: Custom Schema-Driven Engine**

The upfront investment creates long-term leverage. Adding new fields becomes a configuration change, not a code change.

### Consequences

**Positive:**
- Zero code changes for new fields
- Consistent rendering across all forms
- Schema is the single source of truth

**Negative:**
- More complex initial implementation
- Custom code to maintain
- Edge cases need special handling (bilingual fields, content blocks)

**Implementation Details:**
- `FieldRenderer` dispatches based on schema type
- Specialized components for complex types (ImageInput, DetailedContentEditor)
- `x-format` schema extension for custom behaviors


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Decision 5: Atomic Batch Commits via Git Trees API

### Status
**Accepted** ✅

### Context

Portfolio saves often involve multiple files:
- Update `data/projects.json`
- Upload new images
- Delete replaced images

Using GitHub's standard Contents API means N API calls for N changes. Each call is a separate commit. If one fails, data is left inconsistent.

### Options Considered

#### Option A: Sequential Contents API Calls
- One commit per file change
- **Pros:** Simple API, well-documented
- **Cons:** Not atomic, messy commit history

#### Option B: GraphQL API with Multiple Mutations
- Batch changes via GraphQL
- **Pros:** Single request
- **Cons:** Still not truly atomic, complex mutation syntax

#### Option C: Git Trees API (Low-Level)
- Use Blobs, Trees, and Commits endpoints
- **Pros:** True atomic commits, clean history
- **Cons:** Complex implementation, more API calls upfront

### Decision

**Option C: Git Trees API**

True atomicity is worth the implementation complexity. All changes succeed or none do.

### Consequences

**Positive:**
- Atomic operations (data integrity guaranteed)
- Clean commit history (one commit per save)
- Fewer total API calls (batch > sequential)

**Negative:**
- Complex implementation (5-step process)
- Need to handle tree reconstruction for deletes
- More code to maintain

**Implementation Details:**
- Pre-upload large images as blobs
- Build tree with all changes
- Create single commit
- Update ref atomically


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Decision 6: MongoDB Connection Pooling Pattern

### Status
**Accepted** ✅

### Context

Vercel runs on serverless Lambdas. Each Lambda invocation could create a new MongoDB connection. Under load, connections multiply rapidly, causing "Too Many Connections" errors.

### Options Considered

#### Option A: Connect Per Request
- Create connection in each API route
- **Pros:** Simple, isolated
- **Cons:** Connection exhaustion, slow (handshake per request)

#### Option B: Connection Pooling via Mongoose Configuration
- Configure Mongoose pool settings
- **Pros:** Built-in feature
- **Cons:** Doesn't solve Lambda instance creation

#### Option C: Global Singleton Pattern
- Cache connection in global scope
- **Pros:** Reuses connection within Lambda instance
- **Cons:** Custom code, relies on Lambda behavior

### Decision

**Option C: Global Singleton Pattern**

The global scope persists across invocations within the same Lambda instance. This is the recommended pattern for serverless MongoDB.

### Consequences

**Positive:**
- Stable connections (one per Lambda instance)
- Faster requests (no handshake overhead)
- No connection exhaustion under normal load

**Negative:**
- Relies on Lambda global scope behavior
- Connection can go stale (needs reconnection logic)
- Still creates one connection per Lambda instance

**Implementation Details:**
- Global augmentation for TypeScript
- Promise caching prevents race conditions
- Connection reuse across invocations


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Decision 7: Tailwind CSS v4 with Custom Properties

### Status
**Accepted** ✅

### Context

The editor needs a premium aesthetic with glassmorphism effects, custom colors, and responsive design. Style system needed to be:
- Fast to iterate
- Consistent across components
- Support dark mode and custom theming

### Options Considered

#### Option A: CSS Modules
- Scoped CSS per component
- **Pros:** Familiar, no runtime
- **Cons:** Verbose, harder to share utilities

#### Option B: Styled-Components
- CSS-in-JS with runtime
- **Pros:** Dynamic theming, component-scoped
- **Cons:** Runtime overhead, Next.js SSR complexity

#### Option C: Tailwind CSS with Custom Properties
- Utility-first CSS with CSS variables for theming
- **Pros:** Fast iteration, utility-first, custom properties for theming
- **Cons:** Long class strings, learning curve

### Decision

**Option C: Tailwind CSS with Custom Properties**

Tailwind's utility-first approach accelerates development. CSS custom properties enable theming (glassmorphism colors, brand colors) without JavaScript.

### Consequences

**Positive:**
- Rapid prototyping and iteration
- Consistent spacing, colors, typography
- Custom properties enable easy theming
- No runtime CSS-in-JS overhead

**Negative:**
- Long class strings can be hard to read
- Need discipline to extract repeated patterns
- Glassmorphism effects require custom CSS

**Implementation Details:**
- CSS variables defined in `globals.css`
- Tailwind configured to use CSS variables
- Custom utilities for glassmorphism patterns


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Summary

These decisions create a coherent architecture:

| Decision | Trade-off | Outcome |
|----------|-----------|---------|
| Edge Auth | More JWT libraries | Zero-cost rejection |
| Dual Tokens | Token complexity | Security + UX |
| GitHub Storage | API complexity | Version control |
| Schema-Driven | Upfront investment | Future flexibility |
| Batch Commits | Implementation work | Data integrity |
| Connection Pooling | Global state | Serverless stability |
| Tailwind + Variables | Class verbosity | Rapid iteration |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*[← Back to Key Features](04-key-features.md) | [Continue to Challenges & Solutions →](06-challenges-solutions.md)*
