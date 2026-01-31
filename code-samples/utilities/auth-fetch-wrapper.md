# Authenticated Fetch Wrapper

## Context

**Feature:** Automatic Token Handling  
**Complexity:** Low  
**Technologies Used:** Fetch API, Promise Chaining, HTTP Cookies

**The Challenge:**
Create a fetch wrapper that automatically handles 401 errors by refreshing the access token and retrying the original request, completely transparent to the calling code.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

With dual-token authentication, access tokens expire every 15 minutes:

```typescript
// Without wrapper - every API call needs this:
try {
  const res = await fetch('/api/data');
  if (res.status === 401) {
    await fetch('/api/auth/refresh', { method: 'POST' });
    // Retry original request
    const retryRes = await fetch('/api/data');
    return retryRes.json();
  }
  return res.json();
} catch (e) {
  // Handle error
}
```

**Requirements:**
- Transparent token refresh (caller doesn't know it happened)
- Retry original request with new token
- Only attempt refresh once (prevent infinite loops)
- Work with all HTTP methods and request bodies

**Constraints:**
- Must preserve all original request options
- Must handle cases where refresh fails (truly logged out)
- Should not modify successful responses


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Approach

**Intercept → Refresh → Retry** - Wrap fetch to handle 401s automatically.

### Why This Pattern?

The calling code stays simple:
```typescript
// With wrapper - just use authFetch:
const data = await authFetch('/api/data').then(r => r.json());
// Token refresh happens invisibly if needed
```

### Alternatives Considered

1. **Axios with Interceptors**
   - Pros: Built-in interceptor support, familiar API
   - Cons: Additional dependency, larger bundle

2. **React Query/SWR with Auth Handling**
   - Pros: Caching, deduplication built-in
   - Cons: Overkill for simple REST calls, learning curve

3. **Custom Fetch Wrapper** ✅
   - Pros: Zero dependencies, small footprint, full control
   - Trade-offs: Manual implementation of features


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation (Simplified)

```typescript
/**
 * DISCLAIMER: This is a simplified version for demonstration purposes.
 * The actual implementation includes:
 * - Request queuing during refresh
 * - Retry configuration
 * - Logging for debugging
 */

/**
 * Wrapper around fetch that automatically handles token refresh
 * 
 * Usage:
 *   const res = await authFetch('/api/protected-resource', {
 *     method: 'POST',
 *     body: JSON.stringify(data),
 *   });
 * 
 * If the request returns 401:
 * 1. Attempts to refresh the access token via /api/auth/refresh
 * 2. If refresh succeeds, retries the original request
 * 3. If refresh fails, throws the original 401 error
 */
async function authFetch(
  url: string | URL | Request,
  options?: RequestInit
): Promise<Response> {
  // First attempt
  const response = await fetch(url, {
    ...options,
    credentials: 'include',  // Always include cookies
  });

  // If not 401, return as-is (success or other error)
  if (response.status !== 401) {
    return response;
  }

  // ═══════════════════════════════════════════════════════════════
  // REFRESH: Access token expired, attempt refresh
  // ═══════════════════════════════════════════════════════════════
  
  console.log('[authFetch] 401 received, attempting token refresh');

  const refreshResponse = await fetch('/api/auth/refresh', {
    method: 'POST',
    credentials: 'include',  // Send refresh token cookie
  });

  // Refresh failed - truly unauthorized
  if (!refreshResponse.ok) {
    console.log('[authFetch] Refresh failed, user needs to login');
    // Return original 401 response
    return response;
  }

  // ═══════════════════════════════════════════════════════════════
  // RETRY: Refresh succeeded, retry original request
  // ═══════════════════════════════════════════════════════════════
  
  console.log('[authFetch] Token refreshed, retrying original request');

  const retryResponse = await fetch(url, {
    ...options,
    credentials: 'include',
  });

  return retryResponse;
}

export default authFetch;
```

### Usage Examples

```typescript
// Simple GET request
async function fetchUserProfile() {
  const response = await authFetch('/api/user/profile');
  if (!response.ok) throw new Error('Failed to fetch profile');
  return response.json();
}

// POST with body
async function saveDocument(data: object) {
  const response = await authFetch('/api/documents', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!response.ok) throw new Error('Failed to save');
  return response.json();
}

// File upload
async function uploadImage(file: File) {
  const formData = new FormData();
  formData.append('image', file);

  const response = await authFetch('/api/upload', {
    method: 'POST',
    body: formData,
    // Note: Don't set Content-Type - browser will set it with boundary
  });
  return response.json();
}
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Key Techniques Used

#### 1. Credentials: Include

```typescript
fetch(url, {
  credentials: 'include',  // Always send cookies
});

// Without this, HttpOnly cookies won't be sent
// and refresh token cookie won't be included
```

*Why it matters:* HttpOnly cookies are only sent if credentials: 'include' is set.

#### 2. Single Retry (No Infinite Loop)

```typescript
// Don't use authFetch for the refresh call itself!
const refreshResponse = await fetch('/api/auth/refresh', { ... });
//                           ^^^^^ Regular fetch, not authFetch

// If we used authFetch and refresh returned 401,
// we'd get an infinite loop
```

*Why it matters:* Prevents 401 → refresh → 401 → refresh → ...

#### 3. Preserve Response Object

```typescript
// Return the Response object, not json()
return retryResponse;  // ✅ Response object

// Let caller decide what to do with it:
const data = await authFetch(url).then(r => r.json());
const text = await authFetch(url).then(r => r.text());
const blob = await authFetch(url).then(r => r.blob());
```

*Why it matters:* Flexible for JSON, text, binary, or stream responses.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Advanced: Request Queuing

For production use, handle concurrent requests during refresh:

```typescript
let refreshPromise: Promise<void> | null = null;

async function authFetch(url: string, options?: RequestInit): Promise<Response> {
  const response = await fetch(url, { ...options, credentials: 'include' });
  
  if (response.status !== 401) return response;

  // ═══════════════════════════════════════════════════════════════
  // QUEUE: If refresh is already in progress, wait for it
  // ═══════════════════════════════════════════════════════════════
  
  if (refreshPromise) {
    await refreshPromise;
    // After refresh completes, retry
    return fetch(url, { ...options, credentials: 'include' });
  }

  // ═══════════════════════════════════════════════════════════════
  // REFRESH: Start refresh and let others wait
  // ═══════════════════════════════════════════════════════════════
  
  refreshPromise = (async () => {
    const refreshRes = await fetch('/api/auth/refresh', {
      method: 'POST',
      credentials: 'include',
    });
    if (!refreshRes.ok) {
      throw new Error('Refresh failed');
    }
  })();

  try {
    await refreshPromise;
    // Retry after successful refresh
    return fetch(url, { ...options, credentials: 'include' });
  } finally {
    refreshPromise = null;  // Reset for future requests
  }
}
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Approach

```typescript
describe('authFetch', () => {
  const mockFetch = jest.spyOn(global, 'fetch');

  beforeEach(() => {
    mockFetch.mockReset();
  });

  it('should return successful response without refresh', async () => {
    mockFetch.mockResolvedValueOnce(new Response('{"data": "ok"}', { status: 200 }));

    const response = await authFetch('/api/data');
    
    expect(response.status).toBe(200);
    expect(mockFetch).toHaveBeenCalledTimes(1);
  });

  it('should refresh and retry on 401', async () => {
    // First request: 401
    mockFetch.mockResolvedValueOnce(new Response('Unauthorized', { status: 401 }));
    // Refresh: 200 (success)
    mockFetch.mockResolvedValueOnce(new Response('OK', { status: 200 }));
    // Retry: 200
    mockFetch.mockResolvedValueOnce(new Response('{"data": "ok"}', { status: 200 }));

    const response = await authFetch('/api/data');

    expect(response.status).toBe(200);
    expect(mockFetch).toHaveBeenCalledTimes(3);
    expect(mockFetch).toHaveBeenNthCalledWith(2, '/api/auth/refresh', 
      expect.objectContaining({ method: 'POST' })
    );
  });

  it('should return 401 if refresh fails', async () => {
    // First request: 401
    mockFetch.mockResolvedValueOnce(new Response('Unauthorized', { status: 401 }));
    // Refresh: 401 (also failed)
    mockFetch.mockResolvedValueOnce(new Response('Unauthorized', { status: 401 }));

    const response = await authFetch('/api/data');

    expect(response.status).toBe(401);
    expect(mockFetch).toHaveBeenCalledTimes(2);  // No retry
  });

  it('should include credentials in all requests', async () => {
    mockFetch.mockResolvedValueOnce(new Response('OK', { status: 200 }));

    await authFetch('/api/data');

    expect(mockFetch).toHaveBeenCalledWith('/api/data', 
      expect.objectContaining({ credentials: 'include' })
    );
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Considerations

**Overhead:**
- Successful requests: ~0ms overhead (pass-through)
- 401 with refresh: One additional round-trip (~50-100ms)
- Retry: One additional round-trip (~50-100ms)

**Total on refresh scenario: ~100-200ms additional latency**

This is acceptable because:
1. It only happens once every 15 minutes max
2. User doesn't have to re-login
3. Alternative (manual re-auth) would take 5-30 seconds


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Lessons Learned

### What Worked Well
- **Transparency** makes calling code clean
- **Single retry** prevents infinite loops
- **Cookie handling** works automatically
- **Small footprint** (< 30 lines of core logic)

### What I'd Do Differently
- Add **request queuing** for concurrent 401s
- Include **configurable retry delays**
- Add **logging/analytics** for auth failures
- Consider **background refresh** before expiry

### Key Takeaways
1. Wrapper pattern keeps auth concerns out of business logic
2. Always use regular fetch for the refresh call itself
3. Return Response object for flexibility
4. credentials: 'include' is essential for cookie auth


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Common Pitfalls

### 1. Using authFetch for Refresh

```typescript
// ❌ BAD: Infinite loop risk
const refreshResponse = await authFetch('/api/auth/refresh');

// ✅ GOOD: Regular fetch
const refreshResponse = await fetch('/api/auth/refresh', { method: 'POST' });
```

### 2. Forgetting credentials: include

```typescript
// ❌ BAD: Cookies not sent by default on cross-origin
fetch('/api/auth/refresh');

// ✅ GOOD: Explicitly include credentials
fetch('/api/auth/refresh', { credentials: 'include' });
```

### 3. Consuming Response Body Twice

```typescript
// ❌ BAD: Response body can only be read once
const response = await authFetch('/api/data');
console.log(await response.json());  // Works
console.log(await response.json());  // Error!

// ✅ GOOD: Store the result
const data = await response.json();
console.log(data);
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Token Rotation System](../backend/token-rotation-system.md) - Server-side auth
- [Edge Middleware Auth](../backend/edge-middleware-auth.md) - Server-side refresh


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [Fetch API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [Using Fetch with Credentials](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#sending_a_request_with_credentials_included)
- [HTTP 401 Handling Best Practices](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/401)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Sample complexity: Low | Education value: Medium | Interview relevance: Medium*
