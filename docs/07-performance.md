# 07 - Performance

## Optimization Strategies and Results


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Introduction

Performance is a first-class concern in SourceShan. This document covers the performance optimizations implemented, the reasoning behind them, and the results achieved.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Philosophy

### Principles

1. **Measure First** - Don't optimize what you haven't measured
2. **Hot Path Focus** - Optimize the most common paths first
3. **Edge Over Origin** - Do as much as possible at the Edge
4. **Reduce Round Trips** - Batch operations when possible
5. **Lazy Load** - Only load what's needed when it's needed


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Optimization 1: Edge Authentication

### The Optimization

Move JWT verification from Node.js runtime to Edge middleware.

### Before

```
Request → CDN → Node.js Lambda (cold start) → Verify JWT → Handle request
         └───────────────────────────────────────────────────────────┘
                              ~200-1500ms for cold start
```

### After

```
Request → Edge Middleware → Verify JWT → Continue or Reject
         └─────────────────────────────┘
                     ~5-20ms

If valid:
          → Node.js Lambda → Handle request
```

### Impact

| Metric | Before | After |
|--------|--------|-------|
| **Invalid Request Latency** | 200-1500ms | <50ms |
| **Invalid Request Compute** | Full Lambda | Zero |
| **Valid Request Auth Latency** | Part of request | Parallel with cold start |

### Why This Works

- Edge runs at 30+ global locations
- No cold start penalty for Edge functions
- Invalid tokens never reach the server
- Valid tokens are verified before Lambda even starts warming


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Optimization 2: Connection Pooling

### The Optimization

Cache MongoDB connections across Lambda invocations within the same instance.

### Before

```
Request 1 → Connect (100ms) → Query → Response
Request 2 → Connect (100ms) → Query → Response
Request 3 → Connect (100ms) → Query → Response
```

### After

```
Request 1 → Connect (100ms) → Query → Response (cache connection)
Request 2 → Use cache → Query → Response
Request 3 → Use cache → Query → Response
```

### Impact

| Metric | Before | After |
|--------|--------|-------|
| **Connection Time** | Every request | First request only |
| **Per-Request Overhead** | ~100ms | ~0ms |
| **Connection Count** | One per request | One per Lambda |

### Implementation

```typescript
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


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Optimization 3: Batch GitHub Commits

### The Optimization

Combine multiple file operations into a single atomic commit.

### Before (Per-File Commits)

```
Save portfolio with 3 images:
  1. Update JSON      → API call → Commit #1
  2. Upload image 1   → API call → Commit #2
  3. Upload image 2   → API call → Commit #3
  4. Upload image 3   → API call → Commit #4
  5. Delete old image → API call → Commit #5

Total: 5 API calls, 5 commits
```

### After (Batch Commit)

```
Save portfolio with 3 images:
  1. Create blobs for all content → 4 API calls (parallel)
  2. Build tree with all changes  → 1 API call
  3. Create commit                → 1 API call
  4. Update ref                   → 1 API call

Total: 7 API calls (parallelized), 1 commit
```

### Impact

| Metric | Before | After |
|--------|--------|-------|
| **Commits per Save** | N (one per file) | 1 |
| **API Latency** | Sequential | Parallelized |
| **Data Consistency** | At risk | Guaranteed |
| **Commit History** | Cluttered | Clean |

### Why This Works

- Blob creation can be parallelized
- Single commit means atomic operation
- Fewer sequential API calls
- Clean git history aids debugging


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Optimization 4: Image Processing with Sharp

### The Optimization

Convert uploaded images to WebP format with optimal compression.

### Before

```
User uploads: photo.jpg (2.5 MB)
Result: photo.jpg stored as-is (2.5 MB)
```

### After

```
User uploads: photo.jpg (2.5 MB)
Process: Sharp converts to WebP (quality: 100)
Result: photo.webp stored (~400 KB)
```

### Impact

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Image Size** | 2.5 MB | ~400 KB | 84% reduction |
| **Load Time** | Slower | Faster | Proportional to size |
| **Git Repo Size** | Grows fast | Grows slower | Long-term benefit |

### Implementation

```typescript
const webpBuffer = await sharp(Buffer.from(arrayBuffer))
  .webp({ quality: 100 })
  .toBuffer();
```

### Trade-offs

- **Pro:** Smaller files, faster loads
- **Con:** Processing time during save (~100-500ms per image)
- **Decision:** Processing time is acceptable for better long-term performance


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Optimization 5: Installation ID Caching

### The Optimization

Cache GitHub App installation IDs to reduce API calls.

### Before

```
Every GitHub API request:
  1. List all installations → Find matching owner
  2. Use installation ID for authentication
  
If 10 requests: 10 installation lookups
```

### After

```
First request:
  1. List installations → Find matching owner
  2. Cache installation ID with TTL

Subsequent requests:
  1. Check cache → Use cached ID
  
If 10 requests: 1 installation lookup + 9 cache hits
```

### Impact

| Metric | Before | After |
|--------|--------|-------|
| **API Calls per Request** | +1 (installation lookup) | 0 (cached) |
| **Rate Limit Usage** | Higher | Lower |
| **Latency per Request** | +100-200ms | ~0ms |

### Implementation

```typescript
const installationCache = new Map<string, { 
  id: number; 
  expiresAt: number 
}>();

async function getInstallationId(owner: string): Promise<number> {
  const cached = installationCache.get(owner);
  if (cached && cached.expiresAt > Date.now()) {
    return cached.id;
  }
  
  const id = await fetchInstallationId(owner);
  installationCache.set(owner, {
    id,
    expiresAt: Date.now() + 3600000 // 1 hour
  });
  return id;
}
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Optimization 6: Lazy Component Loading

### The Optimization

Use Next.js dynamic imports for heavy components.

### Before

```javascript
import MainEditor from '@/components/editor/MainEditor';

// MainEditor loaded even if user doesn't visit editor page
```

### After

```javascript
import dynamic from 'next/dynamic';

const MainEditor = dynamic(
  () => import('@/components/editor/MainEditor'),
  { loading: () => <EditorSkeleton /> }
);

// MainEditor loaded only when needed
```

### Impact

| Metric | Before | After |
|--------|--------|-------|
| **Initial Bundle Size** | Larger | Smaller |
| **Time to Interactive** | Slower | Faster |
| **Editor Load** | Part of initial | On-demand |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Optimization 7: Framer Motion Bundle Optimization

### The Optimization

Use LazyMotion and only import needed features.

### Standard Import

```javascript
import { motion, AnimatePresence } from 'framer-motion';
// Loads entire framer-motion bundle (~50KB gzipped)
```

### Optimized Import

```javascript
import { LazyMotion, domAnimation, m } from 'framer-motion';

// Wrap app with LazyMotion
<LazyMotion features={domAnimation} strict>
  {children}
</LazyMotion>

// Use m instead of motion
<m.div animate={{ opacity: 1 }} />

// Bundle: ~20KB gzipped (60% reduction)
```

### Impact

| Metric | Standard | Optimized |
|--------|----------|-----------|
| **Bundle Size** | ~50KB | ~20KB |
| **Features** | All | DOM only |
| **Animation Quality** | Same | Same |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Metrics to Track

### Current State

Based on code analysis, these metrics should be measured:

| Category | Metric | Expected Value | To Measure |
|----------|--------|----------------|------------|
| **Auth** | Edge verify latency | <20ms | ✅ Estimated |
| **Auth** | Invalid request cost | $0 | ✅ Designed |
| **DB** | Connection reuse rate | >90% | ⏳ To measure |
| **GitHub** | Batch commit time | <3s | ⏳ To measure |
| **Images** | WebP conversion time | <500ms | ⏳ To measure |
| **Initial Load** | Time to Interactive | <3s | ⏳ To measure |

### Recommended Monitoring

1. **Vercel Analytics** - Basic performance metrics
2. **Custom Logging** - API route execution times
3. **MongoDB Atlas** - Connection and query metrics
4. **GitHub API** - Rate limit monitoring


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Budget

### Proposed Limits

| Metric | Target | Max |
|--------|--------|-----|
| **API Response (p50)** | <200ms | 500ms |
| **API Response (p95)** | <500ms | 1000ms |
| **Initial Page Load (LCP)** | <2.5s | 4s |
| **JavaScript Bundle** | <200KB | 300KB |
| **Time to Interactive** | <3s | 5s |

### Enforcement

- Bundle analyzer in CI
- Lighthouse CI for page metrics
- Custom alerts for API latency


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Future Optimizations

### Planned

1. **Redis Cache** - Cache frequently accessed schemas
2. **CDN for Images** - Serve images from CDN instead of GitHub raw
3. **Streaming Responses** - Stream large datasets incrementally

### Under Consideration

1. **ISR for Static Pages** - Incremental Static Regeneration
2. **Edge Caching** - Cache common API responses
3. **WebP AVIF Migration** - Even smaller images


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Summary

| Optimization | Target | Mechanism |
|--------------|--------|-----------|
| Edge Auth | Latency | Move verification to Edge |
| Connection Pool | DB overhead | Global singleton |
| Batch Commits | API efficiency | Git Trees API |
| Sharp Processing | Image size | WebP conversion |
| Installation Cache | Rate limits | In-memory cache |
| Lazy Loading | Bundle size | Dynamic imports |
| Framer Motion | Bundle size | LazyMotion |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*[← Back to Challenges & Solutions](06-challenges-solutions.md) | [Continue to Testing & Quality →](08-testing-quality.md)*
