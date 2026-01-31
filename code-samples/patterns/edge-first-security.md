# Edge-First Security Pattern

## Context

**Feature:** Architectural Security Strategy  
**Complexity:** High (Conceptual)  
**Technologies Used:** Vercel Edge Middleware, JOSE, JWT, CDN

**The Philosophy:**
Reject unauthorized requests at the CDN edge, before they consume any server resources.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

Traditional serverless authentication has a hidden cost:

```
Traditional Flow (Serverless):
┌─────────────────────────────────────────────────────────────────┐
│ Request → CDN → Lambda Cold Start (500ms) → Auth Check → Reject │
│                      ↑                                          │
│           Server resources consumed for INVALID request          │
└─────────────────────────────────────────────────────────────────┘

With 10,000 invalid requests/hour:
- Compute time: 10,000 × 500ms = 1,388 hours of Lambda time
- Cost: Significant (varies by provider)
- Latency for valid users: Worse (shared resources)
```

**The Question:**
Why does the server even wake up if the request is going to be rejected?


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Pattern

**"Reject at the Edge"** - Authentication at CDN level, before origin server.

```
Edge-First Flow:
┌───────────────────────────────────────────────────────────────────┐
│ Request → CDN Edge (verify JWT in <20ms) → Reject                 │
│                 ↑                                                 │
│     Zero server resources consumed for invalid requests           │
└───────────────────────────────────────────────────────────────────┘

With 10,000 invalid requests/hour:
- Compute time: ~0 (Edge execution is essentially free)
- Cost: Minimal (Edge millicompute)
- Latency for valid users: Better (server not wasted on invalid)
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Why This Matters

### Cost Reduction

| Metric | Traditional | Edge-First |
|--------|-------------|------------|
| Invalid request cost | Full Lambda invocation | ~0 |
| Cold start for invalid | 100-500ms | 0ms |
| Origin server load | 100% of requests | Only valid requests |

### Security Improvement

| Threat | Traditional | Edge-First |
|--------|-------------|------------|
| DDoS impact | Server overloaded | Edge absorbs |
| Brute force | Each attempt hits server | Blocked at Edge |
| Token testing | Server does work | Near-zero cost blocking |

### Performance Gains

| Metric | Traditional | Edge-First |
|--------|-------------|------------|
| Invalid request latency | 100-500ms | 5-20ms |
| Valid request latency | Same | Better (less contention) |
| Geographic latency | To origin | To nearest CDN PoP |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     VERCEL EDGE NETWORK                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    EDGE MIDDLEWARE                          │ │
│  │  1. Extract tokens from cookies                             │ │
│  │  2. Verify JWT with JOSE (Edge-compatible)                  │ │
│  │  3. Check token expiry                                      │ │
│  │  4. Silent refresh if needed                                │ │
│  │  5. RBAC check (role-based access)                          │ │
│  │  6. Decision: Continue or Reject                            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                           │                                      │
│           ┌───────────────┼───────────────┐                      │
│           ↓               ↓               ↓                      │
│      [rejected]      [continued]     [refreshed]                 │
│           │               │               │                      │
│           ↓               │               │                      │
│    401/403 Response       │               │                      │
│    (never hits origin)    │               │                      │
│                           ↓               ↓                      │
│                      ┌─────────────────────────┐                 │
│                      │    ORIGIN SERVER        │                 │
│                      │  (API Routes/Pages)     │                 │
│                      │                         │                 │
│                      │  Only processes valid   │                 │
│                      │  authenticated requests │                 │
│                      └─────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Key Principles

### 1. Edge Runtime Constraints = Security Benefits

```typescript
// Edge Runtime limitations:
// ❌ No Node.js crypto module
// ❌ No file system access
// ❌ Limited npm package compatibility

// These constraints FORCE:
// ✅ Minimal attack surface
// ✅ Fast execution (no heavy dependencies)
// ✅ Predictable performance
```

### 2. Different Libraries for Different Contexts

```typescript
// Node.js Runtime (API Routes) - for signing
import jwt from 'jsonwebtoken';  // Full-featured
const token = jwt.sign(payload, secret);

// Edge Runtime (Middleware) - for verification
import { jwtVerify } from 'jose';  // Edge-compatible
const { payload } = await jwtVerify(token, secret);
```

*Why:* jsonwebtoken uses Node.js crypto; jose uses Web Crypto API.

### 3. Fail Closed, Not Open

```typescript
// ❌ BAD: Fail open (allow if error)
try {
  await verifyToken(token);
  return NextResponse.next();
} catch (error) {
  // Verification failed, but let it through anyway
  return NextResponse.next();  // DANGEROUS!
}

// ✅ GOOD: Fail closed (reject if error)
try {
  await verifyToken(token);
  return NextResponse.next();
} catch (error) {
  // Any error = reject
  return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
}
```

### 4. Silent Refresh at the Edge

```typescript
// Token refresh INSIDE the Edge, not as separate request
// 
// Before: Client detects 401 → Makes refresh request → Retries
// After:  Middleware detects expired → Refreshes → Continues with new token
//
// Result: Zero interruption for users
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Decision Matrix

When to use Edge-First Security:

| Condition | Recommendation |
|-----------|----------------|
| Serverless architecture | ✅ Strongly recommended |
| High invalid request volume | ✅ Will save significant cost |
| Global user base | ✅ Edge PoPs reduce latency |
| Stateless JWT auth | ✅ Perfect fit |
| Session-based auth | ⚠️ May need Redis at Edge |
| Complex auth logic | ⚠️ Keep Edge simple, complex in API |
| Monolithic server | ❌ Less benefit (server always running) |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Trade-offs

### What You Gain

1. **Zero-Cost Invalid Request Handling**
   - Server never wakes up for bad tokens
   
2. **Global Low-Latency Auth**
   - Auth happens at nearest CDN edge (30+ locations)
   
3. **DDoS Resilience**
   - Edge network absorbs attack traffic
   
4. **Cleaner Origin Server Code**
   - API routes trust that requests are authenticated

### What You Give Up

1. **Library Compatibility**
   - Many npm packages don't work in Edge Runtime
   - Must use jose instead of jsonwebtoken for verification
   
2. **Debugging Complexity**
   - Edge logs are separate from server logs
   - Need to check both for full picture
   
3. **State Limitations**
   - No database queries in Edge (too slow)
   - No file system access
   - Limited memory

4. **Cold Start Still Exists for Valid Requests**
   - Edge auth doesn't eliminate server cold starts
   - It just ensures server ONLY handles valid requests


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## When NOT to Use This Pattern

1. **Complex Authorization Rules**
   ```typescript
   // ❌ TOO COMPLEX FOR EDGE:
   // "User can access document if they have 'read' permission 
   //  OR they're in the same organization as the creator
   //  OR they're an admin of any parent organization"
   
   // Instead: Do basic auth at Edge, complex authz in API
   ```

2. **Database-Dependent Auth**
   ```typescript
   // ❌ NEEDS DATABASE:
   // "Check if token is in blocklist"
   // "Check if user's subscription is active"
   
   // Instead: Use JWT claims for basic checks,
   // verify subscription in API route
   ```

3. **Very Short Token Expiry**
   ```typescript
   // ⚠️ CONSIDER CAREFULLY:
   // Token expires every 60 seconds
   // 
   // This means constant refresh requests,
   // which defeats some Edge benefits
   ```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation Checklist

- [ ] Use JOSE library for Edge JWT operations
- [ ] Secrets as `TextEncoder().encode()` for JOSE
- [ ] Fail closed (reject on any error)
- [ ] Implement silent token refresh in middleware
- [ ] Define clear public paths (login, static assets)
- [ ] Keep Edge logic simple (move complex logic to API)
- [ ] Test with invalid tokens to ensure rejection
- [ ] Monitor Edge execution time (<50ms target)
- [ ] Set up separate logging for Edge middleware


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Edge Middleware Auth](../backend/edge-middleware-auth.md) - Implementation details
- [Token Rotation System](../backend/token-rotation-system.md) - How tokens are created


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [Vercel Edge Functions](https://vercel.com/docs/functions/edge-functions)
- [Cloudflare Workers Security](https://developers.cloudflare.com/workers/platform/security/)
- [Zero Trust Architecture - NIST](https://www.nist.gov/publications/zero-trust-architecture)
- [Edge Computing Security Patterns](https://www.cloudflare.com/learning/security/glossary/what-is-edge-computing/)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Pattern complexity: High | Education value: Very High | Interview relevance: Very High*
