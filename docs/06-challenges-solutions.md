# 06 - Challenges & Solutions

## Problems Faced and How We Solved Them


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Introduction

Every project encounters challenges. This document captures the significant problems faced during SourceShan's development and the solutions implemented.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Challenge 1: Edge Runtime Limitations

### The Problem

The Edge Runtime (Vercel Edge Middleware) is optimized for speed but has severe limitations:

- No access to most Node.js APIs
- Limited npm package support
- No file system access
- No native modules

The popular `jsonwebtoken` library depends on Node.js crypto APIs and **does not work** in Edge Runtime.

**Error encountered:**
```
Error: The edge runtime does not support Node.js 'crypto' module.
```

### The Investigation

Researched alternatives:
- `jose` - Designed for modern JavaScript, Edge-compatible
- `@peculiar/webcrypto` - WebCrypto polyfill
- Custom implementation - Using Web Crypto API directly

### The Solution

**Split JWT responsibilities between runtimes:**

| Operation | Runtime | Library | Rationale |
|-----------|---------|---------|-----------|
| Verify Access Token | Edge | JOSE | Frequent, needs Edge speed |
| Verify Refresh Token | Edge | JOSE | Silent refresh in middleware |
| Sign Access Token | Node.js | jsonwebtoken | Less frequent, needs full features |
| Sign Refresh Token | Node.js | jsonwebtoken | Login only, Node.js API route |

**Implementation:**

```typescript
// middleware.ts (Edge Runtime) - Verification only
import { jwtVerify, SignJWT } from 'jose';

const accessSecretKey = new TextEncoder().encode(ACCESS_SECRET);

async function verifyToken(token: string) {
  const { payload } = await jwtVerify(token, accessSecretKey);
  return payload;
}

// lib/auth.ts (Node.js Runtime) - Signing
import jwt from 'jsonwebtoken';

function signAccessToken(payload: TokenPayload): string {
  return jwt.sign(payload, ACCESS_SECRET, { expiresIn: '15m' });
}
```

### The Result

- ✅ JWT verification works at Edge
- ✅ JWT signing works in Node.js
- ✅ Two libraries, but clear separation of concerns
- ✅ Verification (hot path) is Edge-optimized


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Challenge 2: GitHub API Atomicity

### The Problem

When saving portfolio changes, multiple operations may be needed:

1. Update `data/projects.json`
2. Upload `public/images/project-1.webp`
3. Upload `public/images/project-2.webp`
4. Delete `public/images/old-image.webp`

GitHub's standard **Contents API** operates on single files:
- `PUT /repos/{owner}/{repo}/contents/{path}` - Create/Update
- `DELETE /repos/{owner}/{repo}/contents/{path}` - Delete

Each operation creates a **separate commit**. If operation 3 fails:
- `projects.json` is updated ✅
- `project-1.webp` is uploaded ✅
- `project-2.webp` is NOT uploaded ❌
- `old-image.webp` is NOT deleted ❌

**Result:** Data is inconsistent. `projects.json` references images that don't exist or shouldn't exist.

### The Investigation

Explored GitHub's API documentation:
- **Contents API** - Per-file, not atomic
- **Git Trees API** - Low-level, builds entire tree structure
- **GraphQL createCommitOnBranch** - Newer, but still per-file

### The Solution

**Use the Git Trees API for atomic batch commits.**

The Git Trees API operates at Git's internal level:
1. **Blobs** - Individual file contents
2. **Trees** - Directory structures pointing to blobs
3. **Commits** - Snapshots pointing to a tree

**Process:**

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: Get current state                                       │
│  GET /repos/{owner}/{repo}/git/ref/heads/main                   │
│  → Returns: currentCommitSha                                    │
│                                                                  │
│  STEP 2: Get current tree                                        │
│  GET /repos/{owner}/{repo}/git/commits/{sha}                    │
│  → Returns: currentTreeSha                                      │
│                                                                  │
│  STEP 3: Create blobs for new content                            │
│  POST /repos/{owner}/{repo}/git/blobs                           │
│  → For each CREATE/UPDATE operation                              │
│  → Returns: blobSha for each                                    │
│                                                                  │
│  STEP 4: Build new tree                                          │
│  POST /repos/{owner}/{repo}/git/trees                           │
│  {                                                               │
│    base_tree: currentTreeSha,                                    │
│    tree: [                                                       │
│      { path: "data/projects.json", sha: blobSha, mode: "100644" },│
│      { path: "public/images/new.webp", sha: blobSha, mode: "100644" },│
│      // Deletions handled by omitting from tree                  │
│    ]                                                             │
│  }                                                               │
│  → Returns: newTreeSha                                          │
│                                                                  │
│  STEP 5: Create commit                                           │
│  POST /repos/{owner}/{repo}/git/commits                         │
│  {                                                               │
│    tree: newTreeSha,                                            │
│    parents: [currentCommitSha],                                 │
│    message: "Update portfolio"                                   │
│  }                                                               │
│  → Returns: newCommitSha                                        │
│                                                                  │
│  STEP 6: Update ref                                              │
│  PATCH /repos/{owner}/{repo}/git/refs/heads/main                │
│  { sha: newCommitSha }                                          │
│  → DONE! Single atomic commit.                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Handling Deletions:**

Deletions are tricky because you can't just "delete" from a tree. You need to rebuild the tree without the deleted files.

```typescript
// For deletions, we build a complete new tree
// excluding the files to be deleted
const treeEntries = operations
  .filter(op => op.content !== null)  // Exclude deletions
  .map(op => ({
    path: op.path,
    mode: '100644' as const,
    type: 'blob' as const,
    sha: op.sha || calculatedBlobSha
  }));

// The new tree will not include deleted files
// They are effectively removed
```

### The Result

- ✅ All changes happen in one commit
- ✅ If any step fails, no changes are applied
- ✅ Clean commit history (one commit per save)
- ✅ Rollback is trivial (git revert one commit)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Challenge 3: Serverless Database Connections

### The Problem

Vercel's serverless architecture spins up new Lambda instances for each request (or reuses warm instances). Each Lambda creates a new connection context.

**Traditional connection pattern:**

```typescript
// This runs EVERY request
await mongoose.connect(MONGODB_URI);
```

**Result under load:**

```
Request 1 → Lambda A → Connection 1
Request 2 → Lambda B → Connection 2
Request 3 → Lambda C → Connection 3
Request 4 → Lambda A → Connection 4 (!!)
...
Request 100 → Connection limit reached
Error: "Too Many Connections"
```

### The Investigation

Researched serverless + MongoDB patterns:
- MongoDB Atlas connection pools
- Mongoose configuration options
- Global singleton pattern
- Vercel's serverless function documentation

### The Solution

**Global Singleton Pattern with TypeScript Augmentation**

The `global` object in Node.js/serverless persists across invocations within the same Lambda instance.

```typescript
// TypeScript global augmentation
declare global {
  var mongoose: {
    conn: mongoose.Connection | null;
    promise: Promise<mongoose.Connection> | null;
  } | undefined;
}

// Initialize cached state
let cached = global.mongoose || { conn: null, promise: null };
if (!global.mongoose) {
  global.mongoose = cached;
}

async function connectToDatabase() {
  // If we have a cached connection, return it
  if (cached.conn) {
    return cached.conn;
  }

  // If no promise exists, create connection
  if (!cached.promise) {
    cached.promise = mongoose.connect(MONGODB_URI);
  }

  // Wait for connection and cache it
  cached.conn = await cached.promise;
  return cached.conn;
}
```

**How it works:**

```
Request 1 → Lambda A (cold)
  • cached.conn = null
  • Create connection promise
  • Wait for connection
  • Cache in global

Request 2 → Lambda A (warm)
  • cached.conn exists!
  • Return cached connection immediately

Request 3 → Lambda B (cold)
  • Different instance, different global
  • Create new connection
  • Cache in Lambda B's global
```

### The Result

- ✅ One connection per Lambda instance
- ✅ Connection reused for all requests in that instance
- ✅ No "Too Many Connections" under normal load
- ✅ Promise caching prevents race conditions


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Challenge 4: Image Preview Before Upload

### The Problem

When users select images in the editor, they need to:
1. See a preview immediately
2. Not upload until they click "Save"
3. Be able to remove the image (undo) before saving

Traditional approach: Upload immediately, delete if user cancels. This is wasteful and complicated.

### The Investigation

Explored options:
- Immediate upload with rollback
- Local storage of image data
- Blob URLs for preview

### The Solution

**PendingMediaContext with Blob URLs**

```typescript
// When user selects image
function addPendingImage(fieldPath: string, file: File, targetPath: string) {
  const previewUrl = URL.createObjectURL(file);
  
  setPendingImages(prev => new Map(prev).set(fieldPath, {
    file,
    previewUrl,          // blob:// URL for instant preview
    targetPath           // Where it will be uploaded
  }));
}

// ImageInput shows blob URL
<img src={pendingImage?.previewUrl || existingImageUrl} />

// On save: collect pending images
function getPendingImages(): PendingImage[] {
  return Array.from(pendingImages.values());
}

// After save: clean up
function clearPendingImages() {
  pendingImages.forEach(img => URL.revokeObjectURL(img.previewUrl));
  setPendingImages(new Map());
}
```

**Flow:**

```
User selects image
      │
      ▼
Create blob: URL ──────► Show preview instantly
Add to pendingImages     (no upload yet)
      │
      ├── User clicks "Cancel" ──────► Clear pending, revoke blob URLs
      │
      └── User clicks "Save" ──────► Send files to batch-save API
                                     Upload to GitHub
                                     Clear pending, revoke blob URLs
```

### The Result

- ✅ Instant preview (no network delay)
- ✅ No wasted uploads for cancelled images
- ✅ Batch upload on save (efficient)
- ✅ Memory cleanup via `revokeObjectURL`


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Challenge 5: Bilingual Content Editing

### The Problem

Many clients need bilingual content (Arabic + English). The editor needs to:
- Render two inputs side-by-side
- Handle RTL (Arabic) and LTR (English) simultaneously
- Work for different field types (text, textarea, arrays)

### The Investigation

How to detect bilingual fields?
- Explicit flag in schema?
- Convention (naming pattern)?
- Structure detection (object with `ar`/`en` keys)?

### The Solution

**Structure Detection + Specialized Components**

```typescript
// Schema structure for bilingual fields
{
  "name": {
    "type": "object",
    "properties": {
      "ar": { "type": "string", "title": "الاسم" },
      "en": { "type": "string", "title": "Name" }
    }
  }
}

// FieldRenderer detection logic
function FieldRenderer({ field, value, onChange }) {
  // Detect bilingual fields by structure
  if (field.type === 'object' && 
      field.properties?.ar && 
      field.properties?.en) {
    return (
      <BilingualTextField
        arValue={value?.ar || ''}
        enValue={value?.en || ''}
        onChange={(lang, newValue) => 
          onChange({ ...value, [lang]: newValue })
        }
      />
    );
  }
  // ... other field types
}
```

**BilingualTextField component:**

```tsx
<div className="grid grid-cols-2 gap-4">
  {/* Arabic (RTL) */}
  <div dir="rtl" className="text-right">
    <label>{arTitle}</label>
    <input 
      value={arValue}
      onChange={(e) => onChange('ar', e.target.value)}
      className="text-right"
    />
  </div>
  
  {/* English (LTR) */}
  <div dir="ltr" className="text-left">
    <label>{enTitle}</label>
    <input
      value={enValue}
      onChange={(e) => onChange('en', e.target.value)}
      className="text-left"
    />
  </div>
</div>
```

### The Result

- ✅ Automatic detection of bilingual fields
- ✅ Side-by-side editing with correct text direction
- ✅ Works for text, textarea, and array fields
- ✅ No schema modification needed (convention-based)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Challenge 6: Preventing Admin Self-Lockout

### The Problem

Admin users can:
- Change user roles
- Delete users

If an admin accidentally:
- Changes their own role to "client"
- Deletes themselves

They could lock themselves out of the admin panel.

### The Investigation

Common patterns:
- Prevent editing currently logged-in user
- Require another admin for role changes
- Confirmation dialogs

### The Solution

**Filter current user from editable list + confirmation dialogs**

```typescript
// ManageUsers.tsx
function ManageUsers() {
  const currentUserId = getUserIdFromContext();
  
  // Filter out current user from the management list
  const manageableUsers = allUsers.filter(
    user => user._id !== currentUserId
  );
  
  return (
    <>
      {manageableUsers.map(user => (
        <UserRow key={user._id} user={user} />
      ))}
      
      {/* Current user shown separately (view only) */}
      <CurrentUserInfo userId={currentUserId} />
    </>
  );
}

// Delete confirmation
async function handleDelete(userId: string) {
  const confirmed = await showConfirm({
    title: "حذف المستخدم",
    message: "هل أنت متأكد من حذف هذا المستخدم؟ لا يمكن التراجع عن هذا الإجراء.",
    confirmLabel: "حذف",
    cancelLabel: "إلغاء"
  });
  
  if (confirmed) {
    await deleteUser(userId);
  }
}
```

### The Result

- ✅ Current admin cannot edit/delete themselves
- ✅ Clear separation between self-info and user management
- ✅ Confirmation dialogs prevent accidents
- ✅ System remains accessible


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Summary

| Challenge | Root Cause | Solution | Key Insight |
|-----------|------------|----------|-------------|
| Edge Runtime | Node.js APIs unavailable | JOSE library | Split responsibilities between runtimes |
| GitHub Atomicity | Per-file API | Git Trees API | Lower-level APIs offer more control |
| DB Connections | Serverless isolation | Global singleton | `global` persists within Lambda |
| Image Preview | Upload before confirm | Blob URLs | Client-side preview, server-side upload |
| Bilingual Content | RTL + LTR mixing | Structure detection | Convention over configuration |
| Admin Lockout | Self-modification risk | Filter current user | UI prevents dangerous actions |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*[← Back to Technical Decisions](05-technical-decisions.md) | [Continue to Performance →](07-performance.md)*
