# Dual-Token Rotation System

## Context

**Feature:** Session Management & Authentication  
**Complexity:** High  
**Technologies Used:** jsonwebtoken, jose, HttpOnly Cookies, JWT

**The Challenge:**
Implement a secure authentication system that balances security (short-lived tokens) with user experience (long sessions without frequent logins).


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

Traditional session management approaches have trade-offs:

```
Single Long-Lived Token:
✅ Good UX (user stays logged in)
❌ High risk (stolen token = long compromise window)

Single Short-Lived Token:
✅ Low risk (short compromise window)
❌ Bad UX (frequent re-logins, session interruption)
```

**Requirements:**
- Minimize damage from token theft
- Maintain long sessions (7+ days) without re-login
- Support seamless session extension
- Work with Edge Runtime (limited APIs)

**Constraints:**
- Different JWT libraries needed for Edge vs Node.js
- Tokens must be in HttpOnly cookies (XSS protection)
- Silent rotation must be invisible to users
- Must handle concurrent requests during rotation


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Approach

**Dual-Token Strategy:**

| Token | Lifespan | Purpose | Security Level |
|-------|----------|---------|----------------|
| Access Token | 15 minutes | API access | Higher exposure (every request) |
| Refresh Token | 7 days | Session continuity | Lower exposure (only for refresh) |

### Why This Pattern?

1. **If Access Token Stolen:** Only 15 minutes of access
2. **If Refresh Token Stolen:** Can be revoked, requires cookie theft
3. **User Experience:** Seamless 7-day sessions without login

### Alternatives Considered

1. **Single Long-Lived Token (7 days)**
   - Pros: Simple implementation
   - Cons: Long compromise window, no revocation strategy

2. **Session-Based Authentication (Server State)**
   - Pros: Easy revocation, full control
   - Cons: Requires database lookups per request, scaling issues

3. **Dual-Token Rotation** ✅
   - Pros: Short exposure window, long sessions, stateless verification
   - Trade-offs: Requires two JWT libraries (Node + Edge)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation (Simplified)

### Token Signing (Node.js Runtime)

```typescript
/**
 * DISCLAIMER: This is a simplified version for demonstration purposes.
 * The actual implementation includes:
 * - Additional claims (portfolio count, permissions)
 * - Token fingerprinting for binding
 * - Refresh token families for rotation detection
 */

import jwt from 'jsonwebtoken';

// Environment secrets
const ACCESS_SECRET = process.env.JWT_SECRET!;
const REFRESH_SECRET = process.env.REFRESH_TOKEN_SECRET!;

// Token expiration times
const ACCESS_TOKEN_EXPIRY = '15m';    // 15 minutes
const REFRESH_TOKEN_EXPIRY = '7d';    // 7 days

interface TokenPayload {
  userId: string;
  username: string;
  role: 'admin' | 'user';
}

/**
 * Sign a new access token
 * Short-lived, used for every API request
 */
function signAccessToken(payload: TokenPayload): string {
  return jwt.sign(
    {
      userId: payload.userId,
      username: payload.username,
      role: payload.role,
    },
    ACCESS_SECRET,
    { expiresIn: ACCESS_TOKEN_EXPIRY }
  );
}

/**
 * Sign a new refresh token
 * Long-lived, used only for getting new access tokens
 */
function signRefreshToken(payload: TokenPayload): string {
  return jwt.sign(
    {
      userId: payload.userId,
      username: payload.username,
      role: payload.role,
    },
    REFRESH_SECRET,
    { expiresIn: REFRESH_TOKEN_EXPIRY }
  );
}

/**
 * Verify refresh token and return payload
 */
function verifyRefreshToken(token: string): TokenPayload {
  const payload = jwt.verify(token, REFRESH_SECRET);
  return payload as TokenPayload;
}

export { signAccessToken, signRefreshToken, verifyRefreshToken };
```

### Login API Route

```typescript
import { NextResponse } from 'next/server';
import bcrypt from 'bcryptjs';
import { signAccessToken, signRefreshToken } from '@/lib/auth';

export async function POST(request: Request) {
  const { username, password } = await request.json();

  // 1. Find user in database
  await connectToDatabase();
  const user = await User.findOne({ username });

  if (!user) {
    return NextResponse.json(
      { error: 'Invalid credentials' },
      { status: 401 }
    );
  }

  // 2. Verify password
  const isValid = await bcrypt.compare(password, user.password);
  if (!isValid) {
    return NextResponse.json(
      { error: 'Invalid credentials' },
      { status: 401 }
    );
  }

  // 3. Generate both tokens
  const tokenPayload = {
    userId: user._id.toString(),
    username: user.username,
    role: user.role,
  };

  const accessToken = signAccessToken(tokenPayload);
  const refreshToken = signRefreshToken(tokenPayload);

  // 4. Create response with cookies
  const response = NextResponse.json({
    message: 'Login successful',
    user: { username: user.username, role: user.role },
  });

  // 5. Set HttpOnly cookies
  response.cookies.set('accessToken', accessToken, {
    httpOnly: true,           // Not accessible via JavaScript
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',          // CSRF protection
    path: '/',
    maxAge: 15 * 60,          // 15 minutes
  });

  response.cookies.set('refreshToken', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: 7 * 24 * 60 * 60, // 7 days
  });

  return response;
}
```

### Token Refresh API Route

```typescript
import { NextResponse } from 'next/server';
import { signAccessToken, verifyRefreshToken } from '@/lib/auth';

export async function POST(request: Request) {
  // 1. Get refresh token from cookies
  const refreshToken = request.cookies.get('refreshToken')?.value;

  if (!refreshToken) {
    return NextResponse.json(
      { error: 'No refresh token provided' },
      { status: 401 }
    );
  }

  try {
    // 2. Verify refresh token
    const payload = verifyRefreshToken(refreshToken);

    // 3. Generate new access token
    const newAccessToken = signAccessToken({
      userId: payload.userId,
      username: payload.username,
      role: payload.role,
    });

    // 4. Return new access token in cookie
    const response = NextResponse.json({ message: 'Token refreshed' });
    
    response.cookies.set('accessToken', newAccessToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      path: '/',
      maxAge: 15 * 60,
    });

    return response;
  } catch (error) {
    // Refresh token invalid or expired
    return NextResponse.json(
      { error: 'Invalid refresh token' },
      { status: 401 }
    );
  }
}
```

### Key Techniques Used

#### 1. Different Secrets for Each Token

```typescript
// Access and refresh tokens use DIFFERENT secrets
const accessToken = jwt.sign(payload, ACCESS_SECRET);
const refreshToken = jwt.sign(payload, REFRESH_SECRET);

// Why? If one secret is compromised, the other isn't
// Also allows independent rotation of secrets
```

*Why it matters:* Defense in depth - compromising one doesn't compromise the other.

#### 2. Cookie Expiry Matches Token Expiry

```typescript
// Access token: 15 minutes in JWT + 15 minutes in cookie
{ expiresIn: '15m' }
{ maxAge: 15 * 60 }

// Refresh token: 7 days in JWT + 7 days in cookie
{ expiresIn: '7d' }
{ maxAge: 7 * 24 * 60 * 60 }
```

*Why it matters:* Browser automatically removes expired cookies; double protection.

#### 3. Silent Refresh in Middleware

```typescript
// In Edge Middleware - happens before API routes
if (accessTokenExpired && refreshTokenValid) {
  const newAccessToken = await createNewAccessToken();
  response.cookies.set('accessToken', newAccessToken);
  // User never sees the refresh happen!
}
```

*Why it matters:* Seamless UX - users don't experience session interruption.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Token Lifecycle Diagram

```
             ┌─────────────────────────────────────────────────────┐
             │                     7 DAYS                          │
             │◄──────────────────────────────────────────────────►│
             │                                                     │
             │    15m      15m      15m      15m      15m          │
             │   ┌───┐    ┌───┐    ┌───┐    ┌───┐    ┌───┐   ...  │
             │   │AT1│    │AT2│    │AT3│    │AT4│    │AT5│        │
             │   └───┘    └───┘    └───┘    └───┘    └───┘        │
             │     ↑        ↑        ↑        ↑        ↑          │
             │     └──────┬─┴──────┬─┴──────┬─┴──────┬─┘          │
             │            │        │        │        │            │
             │      Silent Refresh (automatic, invisible)          │
             │                                                     │
             │   ┌─────────────────────────────────────────────┐   │
             │   │               REFRESH TOKEN                 │   │
             │   │                 (7 days)                    │   │
             │   └─────────────────────────────────────────────┘   │
             │                                                     │
             └─────────────────────────────────────────────────────┘
                                      │
                                      ▼
                              User must re-login
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Approach

```typescript
describe('Token Rotation System', () => {
  describe('Access Token', () => {
    it('should expire after 15 minutes', async () => {
      const token = signAccessToken(mockPayload);
      
      // Fast-forward time by 16 minutes
      jest.advanceTimersByTime(16 * 60 * 1000);
      
      expect(() => jwt.verify(token, ACCESS_SECRET)).toThrow('jwt expired');
    });
  });

  describe('Refresh Token', () => {
    it('should create new access token when refresh is valid', async () => {
      // Arrange
      const refreshToken = signRefreshToken(mockPayload);
      const request = mockRequest('/api/auth/refresh', {
        cookies: { refreshToken }
      });

      // Act
      const response = await POST(request);

      // Assert
      expect(response.status).toBe(200);
      expect(response.cookies.get('accessToken')).toBeDefined();
    });

    it('should reject expired refresh token', async () => {
      // Create expired token
      const expiredToken = jwt.sign(mockPayload, REFRESH_SECRET, {
        expiresIn: '-1d' // Already expired
      });

      const request = mockRequest('/api/auth/refresh', {
        cookies: { refreshToken: expiredToken }
      });

      const response = await POST(request);
      expect(response.status).toBe(401);
    });
  });

  describe('Cookie Security', () => {
    it('should set HttpOnly flag on both cookies', async () => {
      const response = await login('testuser', 'password');
      
      const accessCookie = response.cookies.get('accessToken');
      const refreshCookie = response.cookies.get('refreshToken');
      
      expect(accessCookie.httpOnly).toBe(true);
      expect(refreshCookie.httpOnly).toBe(true);
    });
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Considerations

**Token Verification Time:**
- Access token verify: ~0.5ms
- Refresh token verify: ~0.5ms
- Token signing: ~1ms

**Network Overhead:**
- Refresh request: ~50-100ms (only when access token expires)
- Silent refresh: Invisible to user (happens in middleware)

| Scenario | User Experience |
|----------|-----------------|
| Valid access token | Instant access |
| Expired access + valid refresh | ~5ms delay (Edge) |
| Both tokens expired | Redirect to login |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Lessons Learned

### What Worked Well
- **15-minute access tokens** are short enough to limit damage but long enough to avoid constant refreshes
- **Silent rotation in Edge** is truly invisible to users
- **HttpOnly cookies** eliminate XSS token theft risk
- **Separate secrets** add defense in depth

### What I'd Do Differently
- Implement **refresh token rotation** (new refresh token on each use)
- Add **token families** to detect refresh token theft
- Consider **sliding window** for refresh token expiry
- Add **device fingerprinting** for suspicious access detection

### Key Takeaways
1. Dual tokens balance security and UX effectively
2. Edge middleware is perfect for silent rotation
3. HttpOnly cookies are non-negotiable for tokens
4. Different secrets for different tokens is good practice


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Security Considerations

### Token Theft Scenarios

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Access token stolen | 15 min access | Short expiry limits damage |
| Refresh token stolen | Session takeover | HttpOnly prevents JS theft |
| Both tokens stolen | Full compromise | Use refresh rotation + fingerprinting |

### Best Practices Implemented

- ✅ Short-lived access tokens (15m)
- ✅ HttpOnly cookies (XSS protection)
- ✅ Secure flag in production (HTTPS only)
- ✅ SameSite attribute (CSRF protection)
- ✅ Separate secrets per token type
- ⏳ Token rotation (recommended addition)
- ⏳ Refresh token families (recommended addition)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Edge Middleware Auth](./edge-middleware-auth.md) - Token verification at Edge
- [Auth Fetch Wrapper](../utilities/auth-fetch-wrapper.md) - Client-side token handling


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [JWT Best Practices - Auth0](https://auth0.com/blog/jwt-security-best-practices/)
- [Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Sample complexity: High | Education value: High | Interview relevance: Very High*
