# 04 - Key Features

## Feature-by-Feature Deep Dive


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Introduction

This document provides detailed documentation for each major feature in SourceShan. For each feature, we'll cover the user story, technical implementation, challenges, and outcomes.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Feature 1: Edge Fortress Authentication

### Overview

Authentication that runs at the Edge (CDN level) before requests reach the Node.js runtime. Invalid tokens are rejected immediately with zero compute cost.

### User Story

**As a** platform administrator  
**I need to** ensure only authenticated users access protected resources  
**So that** client data remains secure and server resources aren't wasted on invalid requests

### Technical Implementation

#### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         REQUEST                                  │
│                            │                                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   Edge Middleware                         │   │
│  │                                                          │   │
│  │  ┌────────────────┐    ┌────────────────┐               │   │
│  │  │ Extract Token  │───►│ Verify (JOSE)  │               │   │
│  │  └────────────────┘    └───────┬────────┘               │   │
│  │                               │                          │   │
│  │            ┌──────────────────┼──────────────────┐       │   │
│  │            ▼                  ▼                  ▼       │   │
│  │     ┌───────────┐     ┌───────────┐     ┌───────────┐   │   │
│  │     │   Valid   │     │  Expired  │     │  Invalid  │   │   │
│  │     │   Token   │     │   Token   │     │   Token   │   │   │
│  │     └─────┬─────┘     └─────┬─────┘     └─────┬─────┘   │   │
│  │           │                 │                 │          │   │
│  │           │                 ▼                 │          │   │
│  │           │         ┌───────────────┐         │          │   │
│  │           │         │ Try Refresh   │         │          │   │
│  │           │         │    Token      │         │          │   │
│  │           │         └───────┬───────┘         │          │   │
│  │           │                 │                 │          │   │
│  │           │      ┌──────────┼──────────┐     │          │   │
│  │           │      ▼          ▼          │     │          │   │
│  │           │   ┌──────┐ ┌──────────┐   │     │          │   │
│  │           │   │ OK   │ │ Refresh  │   │     │          │   │
│  │           │   │      │ │ Failed   │   │     │          │   │
│  │           │   └───┬──┘ └────┬─────┘   │     │          │   │
│  │           │       │         │         │     │          │   │
│  │           ▼       ▼         ▼         ▼     ▼          │   │
│  │     ┌───────────────┐ ┌────────────────────────┐        │   │
│  │     │   Continue    │ │   Redirect to Login    │        │   │
│  │     │   to Server   │ │   or 401 Response      │        │   │
│  │     └───────────────┘ └────────────────────────┘        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

#### Key Technologies

| Technology | Purpose |
|------------|---------|
| **JOSE** | JWT verification in Edge Runtime |
| **Next.js Middleware** | Intercept requests at Edge |
| **HttpOnly Cookies** | Secure token storage |

#### Data Flow

1. Request arrives at Edge
2. Middleware extracts `accessToken` and `refreshToken` from cookies
3. Attempt to verify access token with JOSE
4. If valid → continue to API route
5. If expired → attempt refresh using refresh token
6. If refresh succeeds → set new access token, continue
7. If all fails → redirect to login or return 401

#### Code Highlights

```typescript
// Edge-compatible token verification
async function verifyOrRefreshToken(req: NextRequest) {
  const accessToken = req.cookies.get('accessToken')?.value;
  const refreshToken = req.cookies.get('refreshToken')?.value;

  // Try access token first (most common case)
  if (accessToken) {
    try {
      const { payload } = await jwtVerify(accessToken, accessSecretKey);
      return { payload, newAccessToken: null };
    } catch {
      // Token expired or invalid - try refresh
    }
  }

  // Attempt silent refresh
  if (refreshToken) {
    try {
      const { payload } = await jwtVerify(refreshToken, refreshSecretKey);
      // Create new access token
      const newAccessToken = await new SignJWT({...payload})
        .setProtectedHeader({ alg: 'HS256' })
        .setExpirationTime('15m')
        .sign(accessSecretKey);
      return { payload, newAccessToken };
    } catch {
      // Refresh token also invalid
    }
  }

  return { payload: null, newAccessToken: null };
}
```

### Challenges

**Challenge:** jsonwebtoken doesn't work in Edge Runtime.  
**Solution:** Used JOSE library which is designed for modern JavaScript environments.

**Challenge:** Need to refresh tokens without user interaction.  
**Solution:** Middleware checks refresh token when access token expires and issues new access token automatically.

### Impact

| Metric | Value |
|--------|-------|
| **Invalid Request Cost** | Zero compute (rejected at Edge) |
| **Auth Latency** | Sub-millisecond (Edge network) |
| **Security Posture** | 5-layer defense starting at Edge |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Feature 2: Dual-Token Rotation System

### Overview

A security architecture using short-lived access tokens (15 minutes) for security and long-lived refresh tokens (7 days) for user experience, with automatic silent rotation.

### User Story

**As a** user  
**I need to** stay logged in across sessions without constant re-authentication  
**So that** my workflow isn't interrupted while maintaining security

### Technical Implementation

#### Token Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     TOKEN LIFECYCLE                              │
│                                                                  │
│  LOGIN                        15 MINUTES                         │
│    │                              │                               │
│    ▼                              ▼                               │
│  ┌────────────────┐         ┌────────────────┐                   │
│  │ Access Token   │────────►│ Access Token   │─ ─ ─ ┐            │
│  │ Created        │         │ Expires        │      │            │
│  └────────────────┘         └───────┬────────┘      │            │
│                                     │               │            │
│                              ┌──────┴──────┐        │            │
│                              ▼             ▼        │            │
│                        ┌──────────┐  ┌──────────┐   │            │
│                        │ Refresh  │  │  Force   │   │            │
│                        │ (Silent) │  │  Login   │◄─ ┘            │
│                        └────┬─────┘  └──────────┘  (if no        │
│                             │                       refresh)     │
│                             ▼                                    │
│                      ┌────────────────┐                          │
│                      │ New Access     │                          │
│                      │ Token Created  │                          │
│                      └────────────────┘                          │
│                                                                  │
│  ┌────────────────┐                      ┌────────────────┐      │
│  │ Refresh Token  │─────────────────────►│ Refresh Token  │      │
│  │ Created        │       7 DAYS         │ Expires        │      │
│  └────────────────┘                      └───────┬────────┘      │
│                                                  │               │
│                                                  ▼               │
│                                          ┌────────────────┐      │
│                                          │ Must Re-Login  │      │
│                                          └────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

#### Token Payload

```typescript
interface TokenPayload {
  username: string;
  role: 'admin' | 'client';
  id: string;
  portfolioCount: number;
  firstPortfolioId?: string;
}
```

#### Cookie Configuration

| Cookie | HttpOnly | Secure | SameSite | MaxAge |
|--------|----------|--------|----------|--------|
| accessToken | Yes | Yes (prod) | None (prod) / Lax (dev) | 15 min |
| refreshToken | Yes | Yes (prod) | None (prod) / Lax (dev) | 7 days |

### Challenges

**Challenge:** Keeping sessions alive without security risks.  
**Solution:** 15-minute access tokens limit the window if a token is stolen, while 7-day refresh tokens provide seamless experience.

**Challenge:** Token refresh must be invisible to users.  
**Solution:** Middleware handles refresh automatically; users never see authentication prompts during active sessions.

### Impact

| Outcome | Description |
|---------|-------------|
| **Session Continuity** | Users stay logged in for 7 days |
| **Attack Window** | Limited to 15 minutes for stolen access tokens |
| **User Friction** | Zero login interruptions during active use |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Feature 3: Schema-Driven Form Engine

### Overview

A dynamic form generation system that reads JSON schemas and renders appropriate form components automatically. Adding new fields to the schema automatically updates the editor UI.

### User Story

**As a** developer  
**I need to** add new portfolio fields without writing editor code  
**So that** I can iterate quickly and minimize bugs

### Technical Implementation

#### Component Hierarchy

```
┌────────────────────────────────────────────────────────────────┐
│                        MainEditor                               │
│                            │                                    │
│                            ▼                                    │
│                     ┌─────────────┐                             │
│                     │ DynamicForm │                             │
│                     └──────┬──────┘                             │
│                            │                                    │
│              ┌─────────────┼─────────────┐                      │
│              ▼             ▼             ▼                      │
│       ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│       │FieldRenderer│ │FieldRenderer│ │FieldRenderer│          │
│       └──────┬──────┘ └──────┬──────┘ └──────┬──────┘          │
│              │               │               │                  │
│    ┌─────────┴────┐    ┌─────┴─────┐   ┌─────┴─────┐           │
│    ▼              ▼    ▼           ▼   ▼           ▼           │
│ ┌──────┐    ┌──────────┐ ┌──────────┐ ┌──────────────────┐     │
│ │Input │    │Bilingual │ │ImageInput│ │DetailedContent   │     │
│ │      │    │TextField │ │          │ │Editor            │     │
│ └──────┘    └──────────┘ └──────────┘ └──────────────────┘     │
└────────────────────────────────────────────────────────────────┘
```

#### Schema to Component Mapping

| Schema Type | x-format | Component Rendered |
|-------------|----------|-------------------|
| `string` | - | Text Input |
| `object` | - | Bilingual TextField (if has `ar`/`en` keys) |
| `array` | `detailed-content` | DetailedContentEditor |
| `array` | `simple-list` | StringArrayEditor |
| `array` | `bilingual-list` | BilingualListEditor |
| `string` | `image` | ImageInput |

#### Schema Example

```json
{
  "title": "Projects",
  "type": "object",
  "properties": {
    "name": {
      "type": "object",
      "properties": {
        "ar": { "type": "string", "title": "الاسم" },
        "en": { "type": "string", "title": "Name" }
      }
    },
    "image": {
      "type": "string",
      "format": "image",
      "x-upload-path": "public/images/projects"
    },
    "gallery": {
      "type": "array",
      "items": { "type": "string", "format": "image" }
    },
    "story": {
      "type": "array",
      "x-format": "detailed-content"
    }
  }
}
```

#### FieldRenderer Logic

```typescript
function FieldRenderer({ field, value, onChange }) {
  // Bilingual object (has ar/en keys)
  if (field.type === 'object' && field.properties?.ar && field.properties?.en) {
    return <BilingualTextField {...props} />;
  }

  // Image field
  if (field.format === 'image') {
    return <ImageInput {...props} />;
  }

  // Detailed content array
  if (field.type === 'array' && field['x-format'] === 'detailed-content') {
    return <DetailedContentEditor {...props} />;
  }

  // Default: simple text input
  return <input type="text" {...props} />;
}
```

### Challenges

**Challenge:** Complex nested schemas with multiple field types.  
**Solution:** Recursive FieldRenderer that dispatches to specialized components.

**Challenge:** Image uploads need staging before save.  
**Solution:** PendingMediaContext manages pending uploads separately from form state.

### Impact

| Metric | Before | After |
|--------|--------|-------|
| **Time to add new field** | Hours (code + deploy) | Minutes (schema update) |
| **Code changes for new field** | Multiple files | Zero |
| **Bug risk** | High (manual UI) | Low (tested engine) |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Feature 4: GitHub Batch Commit System

### Overview

Atomic multi-file operations using GitHub's Git Trees API. All changes (creates, updates, deletes) happen in a single commit.

### User Story

**As a** user  
**I need to** save portfolio changes atomically  
**So that** my data is never left in an inconsistent state

### Technical Implementation

#### Traditional vs. Batch Approach

```
TRADITIONAL (Contents API):
┌─────────────────────────────────────────────────────────────────┐
│  Update projects.json  ─────► Commit #1                         │
│  Upload image1.webp    ─────► Commit #2                         │
│  Upload image2.webp    ─────► Commit #3                         │
│  Delete old-image.webp ─────► Commit #4                         │
│                                                                  │
│  Result: 4 separate commits                                      │
│  Risk: If commit #3 fails, data is inconsistent                 │
└─────────────────────────────────────────────────────────────────┘

BATCH (Git Trees API):
┌─────────────────────────────────────────────────────────────────┐
│  Build tree with ALL changes ─────► Single Commit               │
│                                                                  │
│  Result: 1 atomic commit                                         │
│  Guarantee: All succeed or none do                               │
└─────────────────────────────────────────────────────────────────┘
```

#### Git Trees API Flow

```
┌────────────────────────────────────────────────────────────────┐
│                    BATCH COMMIT FLOW                            │
│                                                                 │
│  1. Get current commit SHA                                      │
│     GET /repos/{owner}/{repo}/git/ref/heads/main               │
│                          │                                      │
│                          ▼                                      │
│  2. Get current tree SHA                                        │
│     GET /repos/{owner}/{repo}/git/commits/{sha}                │
│                          │                                      │
│                          ▼                                      │
│  3. Create blobs for new content                                │
│     POST /repos/{owner}/{repo}/git/blobs                       │
│     (one per new/updated file)                                  │
│                          │                                      │
│                          ▼                                      │
│  4. Create new tree with changes                                │
│     POST /repos/{owner}/{repo}/git/trees                       │
│     { base_tree: currentTreeSha, tree: changes }               │
│                          │                                      │
│                          ▼                                      │
│  5. Create commit with new tree                                 │
│     POST /repos/{owner}/{repo}/git/commits                     │
│     { tree: newTreeSha, parents: [currentCommitSha] }          │
│                          │                                      │
│                          ▼                                      │
│  6. Update branch ref                                           │
│     PATCH /repos/{owner}/{repo}/git/refs/heads/main            │
│     { sha: newCommitSha }                                      │
│                          │                                      │
│                          ▼                                      │
│                     ✅ DONE                                     │
└────────────────────────────────────────────────────────────────┘
```

#### Operation Types

```typescript
interface BatchFileOperation {
  path: string;
  content: string | Buffer | null;  // null = delete
  sha?: string;  // For pre-uploaded blobs
}
```

| Operation | Content | Effect |
|-----------|---------|--------|
| Create | String/Buffer | File created with content |
| Update | String/Buffer | File updated with content |
| Delete | `null` | File removed from tree |
| Blob Reference | Empty + SHA | Use pre-uploaded blob |

### Challenges

**Challenge:** Deleting files requires full tree reconstruction.  
**Solution:** Recursively walk tree, excluding files marked for deletion, then rebuild.

**Challenge:** Large images should not be base64 encoded in request.  
**Solution:** Pre-upload images as blobs, reference by SHA in tree.

### Impact

| Aspect | Before | After |
|--------|--------|-------|
| **Commits per save** | N (one per file) | 1 |
| **Data consistency** | At risk | Guaranteed |
| **API calls** | O(n) commits | O(n) blobs + 3 commits |
| **Rollback** | Complex | Single git revert |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Feature 5: Pending Media Context

### Overview

Client-side state management for staging image uploads and deletions before committing to GitHub.

### User Story

**As a** user  
**I need to** preview images before saving  
**So that** I can verify my changes before they're permanent

### Technical Implementation

#### State Structure

```typescript
interface PendingMediaState {
  pendingImages: Map<string, {
    file: File;
    previewUrl: string;  // blob: URL
    targetPath: string;  // where it will be uploaded
  }>;
  deletedPaths: Set<string>;  // paths to delete on save
}
```

#### Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    PENDING MEDIA FLOW                            │
│                                                                  │
│  USER SELECTS IMAGE                                              │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────────────┐                                         │
│  │ Create blob: URL    │                                         │
│  │ for preview         │                                         │
│  └──────────┬──────────┘                                         │
│             │                                                    │
│             ▼                                                    │
│  ┌─────────────────────┐                                         │
│  │ Add to pendingImages│                                         │
│  │ in context          │                                         │
│  └──────────┬──────────┘                                         │
│             │                                                    │
│             ▼                                                    │
│  ┌─────────────────────┐                                         │
│  │ ImageInput shows    │                                         │
│  │ blob: URL preview   │                                         │
│  └──────────┬──────────┘                                         │
│             │                                                    │
│  USER CLICKS SAVE      │                                         │
│             │                                                    │
│             ▼                                                    │
│  ┌─────────────────────┐                                         │
│  │ Collect pending     │                                         │
│  │ images + deletions  │                                         │
│  └──────────┬──────────┘                                         │
│             │                                                    │
│             ▼                                                    │
│  ┌─────────────────────┐                                         │
│  │ batch-save API      │                                         │
│  │ (FormData submit)   │                                         │
│  └──────────┬──────────┘                                         │
│             │                                                    │
│             ▼                                                    │
│  ┌─────────────────────┐                                         │
│  │ Clear pending state │                                         │
│  │ Revoke blob URLs    │                                         │
│  └─────────────────────┘                                         │
└─────────────────────────────────────────────────────────────────┘
```

### Challenges

**Challenge:** Blob URLs need to be revoked to prevent memory leaks.  
**Solution:** Context tracks all blob URLs and revokes them on save or clear.

**Challenge:** Deleted images should be removed from GitHub.  
**Solution:** deletedPaths set tracks removals; batch-save includes DELETE operations.

### Impact

| Feature | Benefit |
|---------|---------|
| **Instant Preview** | Users see changes before commit |
| **Batch Upload** | Efficient single request for all images |
| **Undo Support** | Clear pending state = undo changes |
| **Memory Safety** | Blob URLs properly cleaned up |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Feature 6: Global Connection Pooling

### Overview

Singleton pattern for MongoDB connections that survives Lambda cold starts and prevents connection exhaustion.

### User Story

**As a** system operator  
**I need to** prevent database connection exhaustion  
**So that** the system remains stable under load

### Technical Implementation

#### The Problem

```
WITHOUT POOLING:
┌─────────────────────────────────────────────────────────────────┐
│  Request 1 → Lambda Instance A → New MongoDB Connection         │
│  Request 2 → Lambda Instance B → New MongoDB Connection         │
│  Request 3 → Lambda Instance C → New MongoDB Connection         │
│  Request 4 → Lambda Instance A → New MongoDB Connection (!)     │
│                                                                  │
│  Result: Connections multiply rapidly                            │
│  Error: "Too Many Connections"                                  │
└─────────────────────────────────────────────────────────────────┘

WITH POOLING:
┌─────────────────────────────────────────────────────────────────┐
│  Request 1 → Lambda Instance A → New MongoDB Connection         │
│  Request 2 → Lambda Instance B → New MongoDB Connection         │
│  Request 3 → Lambda Instance C → New MongoDB Connection         │
│  Request 4 → Lambda Instance A → Reuse Existing Connection ✓    │
│                                                                  │
│  Result: One connection per Lambda instance                      │
│  Stable: Connections bounded by Lambda concurrency              │
└─────────────────────────────────────────────────────────────────┘
```

#### Implementation Pattern

```typescript
// TypeScript global augmentation
declare global {
  var mongoose: {
    conn: mongoose.Connection | null;
    promise: Promise<mongoose.Connection> | null;
  };
}

let cached = global.mongoose || { conn: null, promise: null };

async function connectToDatabase() {
  // Return cached connection if available
  if (cached.conn) {
    return cached.conn;
  }

  // Create connection promise if not exists
  if (!cached.promise) {
    cached.promise = mongoose.connect(MONGODB_URI, {
      bufferCommands: false,
    });
  }

  // Wait for connection
  cached.conn = await cached.promise;
  
  // Store in global for next invocation
  global.mongoose = cached;
  
  return cached.conn;
}
```

### Key Concepts

| Concept | Explanation |
|---------|-------------|
| **global** | Persists across Lambda invocations (same instance) |
| **Promise caching** | Prevents multiple concurrent connection attempts |
| **Connection reuse** | Same connection used for all requests in instance |

### Impact

| Metric | Without Pooling | With Pooling |
|--------|----------------|--------------|
| **Connections/Instance** | Many (per request) | 1 |
| **Connection Errors** | Frequent under load | None |
| **Cold Start Penalty** | Every request | First request only |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Summary

These features work together to create a secure, efficient, and user-friendly system:

| Feature | Primary Benefit |
|---------|----------------|
| Edge Authentication | Zero-cost rejection of invalid requests |
| Dual-Token Rotation | Security without UX friction |
| Schema-Driven Forms | Zero code for new fields |
| Batch Commits | Atomic data operations |
| Pending Media | Preview before commit |
| Connection Pooling | Serverless stability |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*[← Back to Solution Architecture](03-solution-architecture.md) | [Continue to Technical Decisions →](05-technical-decisions.md)*
