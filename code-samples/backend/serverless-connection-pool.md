# Serverless Connection Pooling Pattern

## Context

**Feature:** Database Connectivity  
**Complexity:** Medium  
**Technologies Used:** Mongoose, MongoDB, TypeScript Global Augmentation, Singleton Pattern

**The Challenge:**
Prevent "Too Many Connections" errors in serverless environments where each Lambda function invocation might create a new database connection.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

Serverless functions have a unique lifecycle that breaks traditional connection patterns:

```
Traditional Server:
App Start → Create Connection → Handle 10,000 requests → Connection reused

Serverless Lambda:
Cold Start #1 → Create Connection #1 → Handle request → Sleep
Cold Start #2 → Create Connection #2 → Handle request → Sleep
Cold Start #3 → Create Connection #3 → Handle request → Sleep
...
Result: 100 concurrent users = 100+ database connections!
```

**Requirements:**
- Prevent connection exhaustion (MongoDB has limits)
- Reuse connections within the same Lambda instance
- Handle development hot module reloading (HMR)
- Support concurrent requests to same Lambda instance

**Constraints:**
- Serverless functions are stateless by design
- Lambda instances can be frozen and thawed
- Multiple Lambda instances exist concurrently
- No persistent process between cold starts


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Approach

The key insight: **Lambda instances reuse the same global scope between invocations**.

When a Lambda "warms" (stays alive between requests), variables in the global scope persist. We can exploit this to cache the database connection.

### Why This Pattern?

```
With Global Singleton:
Cold Start #1 → Create Connection → Cache in global → Handle request
Warm Request  → Check global → Connection exists! → Handle request
Warm Request  → Check global → Connection exists! → Handle request
```

Result: One connection per Lambda instance, not per request.

### Alternatives Considered

1. **New Connection Per Request**
   - Pros: Simple, stateless
   - Cons: Connection exhaustion, slow (100ms+ per connection)

2. **External Connection Pooler (PgBouncer/ProxySQL)**
   - Pros: Professional solution, highly scalable
   - Cons: Additional infrastructure, cost, complexity

3. **Global Singleton Pattern** ✅
   - Pros: No infrastructure, simple, effective
   - Trade-offs: One connection per Lambda (not per request)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation (Simplified)

```typescript
/**
 * DISCLAIMER: This is a simplified version for demonstration purposes.
 * The actual implementation includes:
 * - Retry logic for initial connection failures
 * - Graceful shutdown handling
 * - Connection health checks
 * - Timeout configurations
 */

import mongoose from 'mongoose';

// ═══════════════════════════════════════════════════════════════════
// STEP 1: TypeScript Global Augmentation
// ═══════════════════════════════════════════════════════════════════
// Extend the global namespace to include our cache
declare global {
  // Must use 'var' for global scope in TypeScript
  // eslint-disable-next-line no-var
  var mongooseCache: {
    conn: mongoose.Connection | null;
    promise: Promise<mongoose.Connection> | null;
  } | undefined;
}

// ═══════════════════════════════════════════════════════════════════
// STEP 2: Initialize the Cache
// ═══════════════════════════════════════════════════════════════════
// If cache exists (warm Lambda), use it
// If not (cold start), create new empty cache
let cached = global.mongooseCache ?? { conn: null, promise: null };

// Store in global for subsequent invocations
if (!global.mongooseCache) {
  global.mongooseCache = cached;
}

// ═══════════════════════════════════════════════════════════════════
// STEP 3: Connection Function with Caching
// ═══════════════════════════════════════════════════════════════════
const MONGODB_URI = process.env.MONGODB_URI!;

if (!MONGODB_URI) {
  throw new Error('MONGODB_URI environment variable not defined');
}

/**
 * Get a database connection, creating one only if needed
 * 
 * This function is safe to call from anywhere - concurrent calls
 * will share the same connection (or connection promise).
 */
async function connectToDatabase(): Promise<mongoose.Connection> {
  // FAST PATH: Return cached connection if available
  if (cached.conn) {
    console.log('[MongoDB] Returning cached connection');
    return cached.conn;
  }

  // SLOW PATH: Create new connection
  // Check if connection is already being created (race condition prevention)
  if (!cached.promise) {
    console.log('[MongoDB] Creating new connection...');
    
    const options = {
      bufferCommands: false,  // Fail fast if not connected
      maxPoolSize: 10,        // Connection pool within the connection
    };

    // Store the PROMISE, not the connection
    // This prevents race conditions when multiple requests
    // arrive during initial connection
    cached.promise = mongoose.connect(MONGODB_URI, options).then((m) => {
      console.log('[MongoDB] Connected successfully');
      return m.connection;
    });
  }

  // Wait for connection to complete
  try {
    cached.conn = await cached.promise;
  } catch (error) {
    // Reset promise so next attempt can try again
    cached.promise = null;
    throw error;
  }

  return cached.conn;
}

export default connectToDatabase;
```

### Key Techniques Used

#### 1. TypeScript Global Augmentation

```typescript
// Extending the global object safely in TypeScript
declare global {
  var mongooseCache: CacheType | undefined;
}

// Why 'var' and not 'let/const'?
// Only 'var' declarations are added to globalThis
// let/const create block-scoped variables, not global
```

*Why it matters:* Type-safe global state without TypeScript errors.

#### 2. Promise Caching (Race Condition Prevention)

```typescript
// Problem: Two requests arrive, both see conn = null
// Request 1: Creates connection...
// Request 2: Also creates connection! (duplicate!)

// Solution: Cache the PROMISE, not just the result
if (!cached.promise) {
  cached.promise = mongoose.connect(/*...*/);
}
// Both requests will await the SAME promise
const conn = await cached.promise;
```

*Why it matters:* Prevents duplicate connections during concurrent cold-start requests.

#### 3. Error Recovery with Promise Reset

```typescript
try {
  cached.conn = await cached.promise;
} catch (error) {
  // Connection failed - reset promise so next request 
  // can try connecting again instead of re-throwing
  cached.promise = null;
  throw error;
}
```

*Why it matters:* Transient errors don't permanently break the connection.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Usage Example

```typescript
// In any API route - call connectToDatabase() before queries
import connectToDatabase from '@/lib/mongodb';
import User from '@/models/User';

export async function GET(request: Request) {
  // First request: Creates connection
  // Subsequent requests: Returns cached connection
  await connectToDatabase();
  
  // Now safe to query
  const users = await User.find({}).lean();
  return Response.json(users);
}
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Approach

```typescript
describe('connectToDatabase', () => {
  beforeEach(() => {
    // Reset global cache before each test
    global.mongooseCache = undefined;
  });

  it('should reuse connection on subsequent calls', async () => {
    // First call
    const conn1 = await connectToDatabase();
    
    // Second call
    const conn2 = await connectToDatabase();
    
    // Should be the exact same object
    expect(conn1).toBe(conn2);
    
    // Mongoose connect should only be called once
    expect(mongoose.connect).toHaveBeenCalledTimes(1);
  });

  it('should handle concurrent initial calls gracefully', async () => {
    // Simulate concurrent requests during cold start
    const [conn1, conn2, conn3] = await Promise.all([
      connectToDatabase(),
      connectToDatabase(),
      connectToDatabase(),
    ]);
    
    // All should get the same connection
    expect(conn1).toBe(conn2);
    expect(conn2).toBe(conn3);
    
    // Only one actual connection created
    expect(mongoose.connect).toHaveBeenCalledTimes(1);
  });

  it('should retry after connection failure', async () => {
    // First attempt fails
    (mongoose.connect as jest.Mock).mockRejectedValueOnce(new Error('Network error'));
    
    await expect(connectToDatabase()).rejects.toThrow('Network error');
    
    // Second attempt succeeds
    (mongoose.connect as jest.Mock).mockResolvedValueOnce(mockConnection);
    
    const conn = await connectToDatabase();
    expect(conn).toBe(mockConnection);
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Considerations

**Without Connection Caching:**
- Connection time: ~100-500ms per request
- Connections created: Equal to request count
- Risk: "Too Many Connections" error after ~100 concurrent users

**With Connection Caching:**
- Connection time: ~100ms for first request only
- Connections created: One per Lambda instance
- Risk: Minimal (bounded by Lambda concurrency)

| Metric | Without Cache | With Cache | Improvement |
|--------|---------------|------------|-------------|
| Cold start latency | +100-500ms | +100-500ms | Same (unavoidable) |
| Warm request latency | +100-500ms | ~0ms | 100% improvement |
| Connection count | = Request count | = Lambda count | 90%+ reduction |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Lessons Learned

### What Worked Well
- **Simple pattern** with big impact on scalability
- **No external infrastructure** required
- **Works across all serverless providers** (Vercel, AWS, etc.)
- **Development HMR** handled gracefully

### What I'd Do Differently
- Add **connection health checks** for long-running Lambda
- Implement **graceful shutdown** for Lambda termination
- Consider **read replicas** for read-heavy workloads
- Add **Prometheus metrics** for connection monitoring

### Key Takeaways
1. Serverless doesn't mean stateless - global scope persists
2. Cache the Promise, not just the result (race conditions!)
3. This pattern works for any "expensive to create" resource
4. Always handle connection errors with promise reset


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Common Pitfalls

### 1. Using `let` Instead of `var`

```typescript
// ❌ BAD: let is block-scoped, not global
declare global {
  let mongooseCache: CacheType;  // TypeScript error!
}

// ✅ GOOD: var adds to globalThis
declare global {
  var mongooseCache: CacheType;
}
```

### 2. Not Caching the Promise

```typescript
// ❌ BAD: Race condition risk
if (!cached.conn) {
  cached.conn = await mongoose.connect(/*...*/);
}

// ✅ GOOD: Promise caching prevents duplicates
if (!cached.promise) {
  cached.promise = mongoose.connect(/*...*/);
}
cached.conn = await cached.promise;
```

### 3. Not Resetting on Error

```typescript
// ❌ BAD: Failed promise stays in cache forever
cached.promise = mongoose.connect(/*...*/);
cached.conn = await cached.promise; // If this throws, promise never resets

// ✅ GOOD: Reset on failure
try {
  cached.conn = await cached.promise;
} catch (e) {
  cached.promise = null;  // Allow retry
  throw e;
}
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Edge Middleware Auth](./edge-middleware-auth.md) - Where auth happens before DB access
- [GitHub Batch Commits](./github-batch-commits.md) - Alternative data storage approach


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [MongoDB Connection Pooling](https://www.mongodb.com/docs/drivers/node/current/fundamentals/connection/connection-options/#connection-pool-options)
- [Mongoose Connection Guide](https://mongoosejs.com/docs/connections.html)
- [Vercel Serverless Functions](https://vercel.com/docs/functions/serverless-functions)
- [AWS Lambda Execution Context](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-context.html)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Sample complexity: Medium | Education value: High | Interview relevance: High*
