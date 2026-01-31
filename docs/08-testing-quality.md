# 08 - Testing & Quality

## Testing Approach and Code Quality Standards


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Introduction

This document covers the testing strategy, code quality standards, and quality assurance practices in SourceShan.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Current Testing State

### Honest Assessment

Based on the codebase analysis:

| Test Type | Status | Files |
|-----------|--------|-------|
| Unit Tests | ❌ Not implemented | 0 |
| Integration Tests | ❌ Not implemented | 0 |
| E2E Tests | ❌ Not implemented | 0 |

**Coverage: 0%**

### Why No Tests (Yet)?

This is a solo developer project where:
- Speed to market was prioritized
- Manual testing during development was sufficient
- Test infrastructure was deferred

**This is a known gap that should be addressed.**


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Philosophy

### Recommended Approach

If implementing tests, the recommended strategy is:

```
┌─────────────────────────────────────────────────────────────────┐
│                      TESTING PYRAMID                             │
│                                                                  │
│                          ╱▲╲                                     │
│                         ╱  ╲                                     │
│                        ╱ E2E╲                                    │
│                       ╱──────╲                                   │
│                      ╱        ╲                                  │
│                     ╱Integration╲                                │
│                    ╱──────────────╲                              │
│                   ╱                ╲                             │
│                  ╱    Unit Tests    ╲                            │
│                 ╱────────────────────╲                           │
│                                                                  │
│  More tests at the bottom, fewer at the top                     │
│  Bottom tests run fast, top tests run slow                      │
└─────────────────────────────────────────────────────────────────┘
```

### Priority Order

1. **E2E Tests First** - Highest ROI for existing untested code
2. **Integration Tests** - API route testing
3. **Unit Tests** - Utility functions, complex logic


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Recommended Test Coverage

### E2E Tests (Priority 1)

| Flow | Priority | Description |
|------|----------|-------------|
| Login Flow | Critical | Username/password → Dashboard |
| Editor Save | Critical | Edit content → Save → Verify |
| Admin User CRUD | High | Create/Edit/Delete users |
| Token Refresh | High | Expired token → Silent refresh |

**Recommended Tool:** Playwright

```typescript
// Example: Login flow test
test('user can login and access editor', async ({ page }) => {
  await page.goto('/login');
  
  await page.fill('[name="username"]', 'testuser');
  await page.fill('[name="password"]', 'testpass');
  await page.click('button[type="submit"]');
  
  await expect(page).toHaveURL('/editor');
  await expect(page.locator('h1')).toContainText('المحرر');
});
```

### Integration Tests (Priority 2)

| API Route | Priority | Test Cases |
|-----------|----------|------------|
| `/api/auth/login` | Critical | Valid creds, invalid creds, missing fields |
| `/api/auth/refresh` | Critical | Valid token, expired token, missing token |
| `/api/portfolio/batch-save` | High | Single file, multiple files, with deletions |
| `/api/admin/users` | High | CRUD operations, RBAC enforcement |

**Recommended Tool:** Jest + Next.js testing utilities

```typescript
// Example: Login API test
describe('/api/auth/login', () => {
  it('returns tokens for valid credentials', async () => {
    const response = await POST('/api/auth/login', {
      body: { username: 'admin', password: 'correct' }
    });
    
    expect(response.status).toBe(200);
    expect(response.cookies).toHaveProperty('accessToken');
    expect(response.cookies).toHaveProperty('refreshToken');
  });

  it('returns 401 for invalid credentials', async () => {
    const response = await POST('/api/auth/login', {
      body: { username: 'admin', password: 'wrong' }
    });
    
    expect(response.status).toBe(401);
  });
});
```

### Unit Tests (Priority 3)

| Module | Functions to Test |
|--------|-------------------|
| `lib/auth.ts` | `signAccessToken`, `verifyRefreshToken` |
| `lib/github.ts` | `normalizePath`, helper functions |
| `lib/mongodb.ts` | Connection caching logic |

```typescript
// Example: Auth utility tests
describe('auth utilities', () => {
  it('signAccessToken creates valid JWT', () => {
    const token = signAccessToken({
      username: 'test',
      role: 'client',
      id: '123',
      portfolioCount: 1
    });
    
    expect(typeof token).toBe('string');
    expect(token.split('.')).toHaveLength(3);
  });

  it('verifyRefreshToken throws for invalid token', () => {
    expect(() => verifyRefreshToken('invalid')).toThrow();
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Code Quality Standards

### TypeScript Configuration

```json
// tsconfig.json highlights
{
  "compilerOptions": {
    "strict": true,           // All strict checks enabled
    "noEmit": true,           // Next.js handles compilation
    "esModuleInterop": true,  // CommonJS/ESM interop
    "skipLibCheck": true      // Performance optimization
  }
}
```

### ESLint Configuration

```javascript
// eslint.config.mjs
import { defineConfig, globalIgnores } from "eslint/config";
import nextVitals from "eslint-config-next/core-web-vitals";
import nextTs from "eslint-config-next/typescript";

const eslintConfig = defineConfig([
  ...nextVitals,    // Next.js best practices
  ...nextTs,        // TypeScript rules
  globalIgnores([
    ".next/**",
    "out/**",
    "build/**",
    "next-env.d.ts",
  ]),
]);
```

### Rules Enforced

| Category | Rules |
|----------|-------|
| **TypeScript** | strict mode, no-any (warning) |
| **React** | hooks rules, jsx-key |
| **Next.js** | no-html-link-for-pages, no-sync-scripts |
| **Web Vitals** | Core Web Vitals related rules |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Code Quality Metrics

### From Analysis

| Metric | Value | Assessment |
|--------|-------|------------|
| **Lines of Code** | ~6,500 | Manageable |
| **Component Count** | 53 files | Well-organized |
| **TypeScript** | 100% | Full coverage |
| **ESLint** | Configured | Active |
| **Test Coverage** | 0% | Gap to address |

### File Organization

```
src/
├── components/          # React components
│   ├── admin/          # Feature: Admin
│   ├── editor/         # Feature: Editor
│   ├── layout/         # Layout components
│   ├── common/         # Shared components
│   └── ui/             # UI primitives
├── context/            # React contexts
├── lib/                # Utility libraries
└── models/             # Database models
```

**Assessment:** Well-structured feature-based organization.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Error Handling

### Pattern Used

```typescript
// API Route pattern
export async function POST(req: Request) {
  try {
    // Business logic
    const result = await doSomething();
    return NextResponse.json(result);
    
  } catch (error: unknown) {
    console.error('Operation Error:', error);
    
    const errorMessage = error instanceof Error 
      ? error.message 
      : String(error);
      
    return NextResponse.json(
      { message: 'Operation failed', error: errorMessage },
      { status: 500 }
    );
  }
}
```

### Strengths

- ✅ All API routes wrapped in try-catch
- ✅ Errors logged to console
- ✅ Structured error responses
- ✅ TypeScript unknown type for errors

### Gaps

- ❌ No centralized error handler
- ❌ No error tracking service (Sentry)
- ❌ Basic console.error logging

### Recommendations

1. Add Sentry for production error tracking
2. Create error utility for consistent formatting
3. Add request ID for log correlation


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Security Practices

### Implemented

| Practice | Implementation |
|----------|---------------|
| **Password Hashing** | bcryptjs with 10 rounds |
| **JWT Security** | Short-lived access tokens |
| **Cookie Security** | HttpOnly, Secure, SameSite |
| **RBAC** | Role checks in middleware |
| **Input Validation** | Basic checks in API routes |

### Not Implemented

| Practice | Recommendation |
|----------|---------------|
| **Rate Limiting** | Add rate limits to auth endpoints |
| **CSRF Protection** | Consider for state-changing requests |
| **Content Security Policy** | Add CSP headers |
| **Audit Logging** | Log security-relevant events |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Recommended Testing Setup

### Test Framework Selection

```json
// package.json additions
{
  "devDependencies": {
    "@playwright/test": "^1.40.0",
    "jest": "^29.0.0",
    "@types/jest": "^29.0.0",
    "ts-jest": "^29.0.0",
    "@testing-library/react": "^14.0.0"
  },
  "scripts": {
    "test": "jest",
    "test:e2e": "playwright test",
    "test:coverage": "jest --coverage"
  }
}
```

### Directory Structure

```
tests/
├── unit/               # Unit tests
│   ├── lib/
│   │   ├── auth.test.ts
│   │   └── github.test.ts
│   └── components/
├── integration/        # API tests
│   ├── auth.test.ts
│   └── portfolio.test.ts
└── e2e/               # Playwright tests
    ├── login.spec.ts
    └── editor.spec.ts
```

### CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
      - run: pnpm install
      - run: pnpm test

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
      - run: pnpm install
      - run: pnpm exec playwright install
      - run: pnpm test:e2e
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Quality Improvement Roadmap

### Phase 1: Foundation (Week 1-2)

- [ ] Set up Jest for unit tests
- [ ] Add tests for `lib/auth.ts`
- [ ] Set up Playwright for E2E
- [ ] Add login flow E2E test

### Phase 2: Critical Paths (Week 3-4)

- [ ] E2E: Editor save flow
- [ ] E2E: Admin user management
- [ ] Integration: Auth API routes
- [ ] Integration: Portfolio API routes

### Phase 3: Comprehensive (Week 5-6)

- [ ] Unit tests for form engine
- [ ] Integration tests for GitHub operations
- [ ] Add Sentry for error tracking
- [ ] Set up coverage reporting

### Phase 4: Automation (Week 7-8)

- [ ] CI/CD pipeline for tests
- [ ] Coverage thresholds
- [ ] Pre-commit hooks
- [ ] Lighthouse CI for performance


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Summary

### Current State

| Aspect | Status |
|--------|--------|
| **TypeScript** | ✅ Strict mode |
| **ESLint** | ✅ Configured |
| **Testing** | ❌ Not implemented |
| **Error Tracking** | ❌ Not implemented |
| **Code Organization** | ✅ Well-structured |

### Priority Actions

1. **Add E2E tests** for critical flows (login, editor)
2. **Set up error tracking** with Sentry
3. **Add rate limiting** to auth endpoints
4. **Create testing infrastructure** (Jest + Playwright)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*[← Back to Performance](07-performance.md) | [Continue to Deployment →](09-deployment.md)*
