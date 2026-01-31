# Edge Middleware Authentication Pattern

## Context

**Feature:** Authentication & Authorization  
**Complexity:** High  
**Technologies Used:** Next.js Edge Runtime, JOSE Library, JWT, HttpOnly Cookies

**The Challenge:**
Implement authentication that runs **before** the Node.js server even wakes up, reducing compute costs for invalid requests and improving security by blocking bad actors at the CDN level.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

Traditional authentication in serverless environments has a critical flaw:

```
Traditional Flow:
Request → CDN → Lambda Cold Start (~500ms) → Auth Check → Reject
                      ↑
          Server does work BEFORE knowing if request is valid
```

We needed an authentication system that:

**Requirements:**
- Verify JWTs at the CDN Edge (before Node.js)
- Support automatic token refresh without user intervention
- Enforce role-based access control (RBAC) per route
- Handle multiple token states (valid, expired, missing, invalid)

**Constraints:**
- Edge Runtime has limited APIs (no Node.js crypto)
- Can't use `jsonwebtoken` library (Node.js only)
- Must be fast (<50ms for auth check)
- Maintain session continuity during token rotation


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Approach

The key insight: **Move authentication from the origin server to the Edge network.**

### Why This Pattern?

Vercel's Edge Middleware runs on the Cloudflare Network at 30+ global locations. By verifying JWTs here, we achieve:

1. **Zero Server Cost for Invalid Tokens** - Server never wakes up
2. **Faster Rejection** - Edge is geographically closer to users
3. **Global Rate Limiting** - Block at the edge, not the origin
4. **Seamless Rotation** - Refresh tokens during request lifecycle

### Alternatives Considered

1. **Standard API Route Authentication**
   - Pros: Simple, uses familiar Node.js libraries
   - Cons: Server wakes up for every request (even invalid ones)

2. **Third-Party Auth (NextAuth/Auth0)**
   - Pros: Feature-rich, maintained by teams
   - Cons: Less control over Edge behavior, potential vendor lock-in

3. **Edge Middleware with JOSE** ✅
   - Pros: Full control, zero invalid-request compute, Edge-native
   - Trade-offs: Need to learn JOSE API, two JWT libraries in project


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation (Simplified)

```typescript
/**
 * DISCLAIMER: This is a simplified version for demonstration purposes.
 * The actual implementation includes:
 * - More comprehensive error handling
 * - Logging for debugging
 * - Rate limiting integration
 * - Additional RBAC rules
 */

import { NextRequest, NextResponse } from 'next/server';
import { jwtVerify, SignJWT } from 'jose';

// Secrets as Uint8Array (required by JOSE)
const ACCESS_SECRET = new TextEncoder().encode(process.env.JWT_SECRET);
const REFRESH_SECRET = new TextEncoder().encode(process.env.REFRESH_SECRET);

// Token payload interface
interface TokenPayload {
  userId: string;
  username: string;
  role: 'admin' | 'user';
  exp?: number;
}

// Routes that don't require authentication
const PUBLIC_PATHS = ['/api/auth', '/login', '/_next', '/favicon.ico'];

/**
 * Check if path requires authentication
 */
function isProtectedPath(pathname: string): boolean {
  return !PUBLIC_PATHS.some(path => pathname.startsWith(path));
}

/**
 * Check if path requires admin role
 */
function requiresAdmin(pathname: string): boolean {
  return pathname.startsWith('/admin') || pathname.startsWith('/api/admin');
}

/**
 * Verify and optionally refresh tokens
 * 
 * Returns: { payload, newAccessToken? }
 * - payload: null if both tokens invalid
 * - newAccessToken: set only if refresh occurred
 */
async function verifyOrRefresh(
  accessToken: string | undefined,
  refreshToken: string | undefined
): Promise<{ payload: TokenPayload | null; newAccessToken: string | null }> {
  
  // Try access token first (happy path)
  if (accessToken) {
    try {
      const { payload } = await jwtVerify(accessToken, ACCESS_SECRET);
      return { payload: payload as TokenPayload, newAccessToken: null };
    } catch (error) {
      // Access token expired or invalid - try refresh
      console.log('[Edge] Access token invalid, attempting refresh');
    }
  }

  // Attempt silent refresh
  if (refreshToken) {
    try {
      const { payload: refreshPayload } = await jwtVerify(
        refreshToken, 
        REFRESH_SECRET
      );

      // Create new access token
      const newAccessToken = await new SignJWT({
        userId: refreshPayload.userId,
        username: refreshPayload.username,
        role: refreshPayload.role,
      })
        .setProtectedHeader({ alg: 'HS256' })
        .setExpirationTime('15m')
        .sign(ACCESS_SECRET);

      console.log('[Edge] Token refreshed successfully');
      return { 
        payload: refreshPayload as TokenPayload, 
        newAccessToken 
      };
    } catch (error) {
      // Refresh token also invalid
      console.log('[Edge] Refresh token invalid');
    }
  }

  return { payload: null, newAccessToken: null };
}

/**
 * Main middleware function - runs at the Edge
 */
export async function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl;

  // 1. Allow public paths
  if (!isProtectedPath(pathname)) {
    return NextResponse.next();
  }

  // 2. Extract tokens from cookies
  const accessToken = req.cookies.get('accessToken')?.value;
  const refreshToken = req.cookies.get('refreshToken')?.value;

  // 3. Verify or refresh
  const { payload, newAccessToken } = await verifyOrRefresh(
    accessToken, 
    refreshToken
  );

  // 4. Handle unauthenticated
  if (!payload) {
    // API routes: return 401
    if (pathname.startsWith('/api/')) {
      return NextResponse.json(
        { error: 'Authentication required' },
        { status: 401 }
      );
    }
    // Pages: redirect to login
    return NextResponse.redirect(new URL('/login', req.url));
  }

  // 5. Check RBAC
  if (requiresAdmin(pathname) && payload.role !== 'admin') {
    return NextResponse.json(
      { error: 'Admin access required' },
      { status: 403 }
    );
  }

  // 6. Continue with request
  const response = NextResponse.next();

  // 7. Set new access token if refreshed
  if (newAccessToken) {
    response.cookies.set('accessToken', newAccessToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      path: '/',
      maxAge: 15 * 60, // 15 minutes
    });
  }

  return response;
}

// Route configuration - apply to all routes except static assets
export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

### Key Techniques Used

#### 1. JOSE for Edge Compatibility

```typescript
// Node.js jsonwebtoken (NOT Edge compatible)
// ❌ const payload = jwt.verify(token, secret);

// JOSE library (Edge compatible)
// ✅ Uses Web Crypto API underneath
import { jwtVerify, SignJWT } from 'jose';

// Secret must be Uint8Array, not string
const secret = new TextEncoder().encode(process.env.JWT_SECRET);

const { payload } = await jwtVerify(token, secret);
```

*Why it matters:* Edge Runtime uses V8 isolates without Node.js APIs. JOSE uses Web Crypto API.

#### 2. Silent Token Rotation

```typescript
// Check access token first (fast path)
if (accessToken) {
  try {
    return await jwtVerify(accessToken, ACCESS_SECRET);
  } catch {
    // Fall through to refresh
  }
}

// Only attempt refresh if access token failed
if (refreshToken) {
  const newToken = await createNewAccessToken(refreshPayload);
  // Set in response cookies - user never notices
}
```

*Why it matters:* Users stay logged in for 7 days without manual re-authentication.

#### 3. Cookie Security Configuration

```typescript
response.cookies.set('accessToken', newAccessToken, {
  httpOnly: true,      // JavaScript can't access
  secure: true,        // HTTPS only in production
  sameSite: 'lax',     // CSRF protection
  path: '/',           // Available on all routes
  maxAge: 15 * 60,     // Expires with token
});
```

*Why it matters:* HttpOnly prevents XSS token theft; Secure prevents MITM attacks.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Approach

```typescript
// Example test (simplified)
describe('Edge Middleware', () => {
  it('should allow requests with valid access token', async () => {
    // Arrange
    const validToken = await createMockAccessToken({ role: 'user' });
    const request = mockRequest('/api/data', {
      cookies: { accessToken: validToken }
    });

    // Act
    const response = await middleware(request);

    // Assert
    expect(response.status).not.toBe(401);
    expect(response.headers.get('x-middleware-next')).toBe('1');
  });

  it('should refresh expired access token if refresh token valid', async () => {
    // Arrange
    const expiredAccess = await createMockAccessToken({ exp: -1 });
    const validRefresh = await createMockRefreshToken();
    const request = mockRequest('/api/data', {
      cookies: { 
        accessToken: expiredAccess,
        refreshToken: validRefresh 
      }
    });

    // Act
    const response = await middleware(request);

    // Assert
    expect(response.status).not.toBe(401);
    expect(response.cookies.get('accessToken')).toBeDefined();
  });

  it('should reject admin routes for non-admin users', async () => {
    // Arrange
    const userToken = await createMockAccessToken({ role: 'user' });
    const request = mockRequest('/admin/users', {
      cookies: { accessToken: userToken }
    });

    // Act
    const response = await middleware(request);

    // Assert
    expect(response.status).toBe(403);
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Considerations

**Before (Traditional Server Auth):**
- Invalid request latency: ~500-1500ms (includes cold start)
- Compute cost: Full Lambda invocation per request
- Geographic latency: Requests travel to origin server

**After (Edge Middleware Auth):**
- Invalid request latency: ~5-20ms
- Compute cost: Zero for invalid tokens
- Geographic latency: Auth at nearest CDN edge

**Metrics (Estimated):**
| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Auth latency (invalid) | 500ms | 15ms | 97% faster |
| Server invocations (invalid) | 100% | 0% | 100% reduction |
| Valid request overhead | 0ms | ~5ms | Acceptable |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Lessons Learned

### What Worked Well
- **Edge-first design** eliminated "wasted" server compute
- **Silent rotation** provides seamless UX for long sessions
- **JOSE library** is well-documented and performant
- **Separation of concerns** - auth logic isolated in middleware

### What I'd Do Differently
- Add **structured logging** for production debugging
- Implement **Edge-compatible rate limiting**
- Consider **JWT claims encryption** for sensitive payloads
- Add **automatic logout** after N failed refreshes

### Key Takeaways
1. Edge Runtime is more capable than expected for auth workloads
2. The `jose` library is the go-to for Edge JWT work
3. Token rotation should be invisible to users
4. Security at the edge is cheaper than security at the origin


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Token Rotation System](./token-rotation-system.md) - JWT signing in Node.js runtime
- [Auth Fetch Wrapper](../utilities/auth-fetch-wrapper.md) - Client-side token handling
- [Edge-First Security](../patterns/edge-first-security.md) - Philosophical overview


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [JOSE Library Documentation](https://github.com/panva/jose)
- [Next.js Middleware Documentation](https://nextjs.org/docs/app/building-your-application/routing/middleware)
- [Vercel Edge Runtime](https://vercel.com/docs/functions/edge-functions/edge-runtime)
- [JWT Best Practices - Auth0](https://auth0.com/blog/jwt-security-best-practices/)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Sample complexity: High | Education value: High | Interview relevance: High*
