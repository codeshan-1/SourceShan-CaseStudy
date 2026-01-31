# 09 - Deployment

## CI/CD Pipeline and Infrastructure


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Introduction

This document covers the deployment infrastructure, CI/CD pipeline, and operational considerations for SourceShan.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Deployment Stack

### Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        DEPLOYMENT STACK                          │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Vercel Platform                         │  │
│  │                                                           │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│  │  │    Edge     │  │  Serverless │  │   Static    │       │  │
│  │  │  Functions  │  │   Lambda    │  │   Assets    │       │  │
│  │  │(middleware) │  │ (API routes)│  │  (CDN)      │       │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      Data Layer                            │  │
│  │                                                           │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐        │  │
│  │  │    MongoDB Atlas    │  │    GitHub Repos     │        │  │
│  │  │   (User Data)       │  │  (Portfolio Data)   │        │  │
│  │  └─────────────────────┘  └─────────────────────┘        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Provider | Purpose |
|-----------|----------|---------|
| **Edge Functions** | Vercel Edge | Authentication middleware |
| **Serverless Functions** | Vercel | API routes |
| **Static Assets** | Vercel CDN | CSS, JS, images |
| **Database** | MongoDB Atlas | User and auth data |
| **Content Storage** | GitHub | Portfolio JSON and images |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## CI/CD Pipeline

### Current Setup

Vercel provides automatic CI/CD:

```
┌─────────────────────────────────────────────────────────────────┐
│                      DEPLOYMENT FLOW                             │
│                                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────┐  │
│  │ git push│───►│ Vercel  │───►│  Build  │───►│  Deploy     │  │
│  │         │    │ Webhook │    │         │    │  (Atomic)   │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────────┘  │
│                                                                  │
│  Features:                                                       │
│  • Automatic on every push to main                              │
│  • Preview deployments for PRs                                   │
│  • Instant rollback available                                    │
│  • Zero-downtime deployments                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Deployment Triggers

| Trigger | Action |
|---------|--------|
| Push to `main` | Production deployment |
| Pull Request | Preview deployment |
| Manual | Dashboard redeploy |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Environment Configuration

### Required Environment Variables

| Variable | Description | Where Set |
|----------|-------------|-----------|
| `MONGODB_URI` | MongoDB connection string | Vercel Dashboard |
| `JWT_SECRET` | Access token signing key | Vercel Dashboard |
| `REFRESH_TOKEN_SECRET` | Refresh token signing key | Vercel Dashboard |
| `GITHUB_APP_ID` | GitHub App ID | Vercel Dashboard |
| `GITHUB_PRIVATE_KEY` | GitHub App private key | Vercel Dashboard |

### Environment Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENVIRONMENT STRATEGY                          │
│                                                                  │
│  Development (local)                                             │
│  • .env.local file (gitignored)                                  │
│  • Local MongoDB or Atlas                                        │
│  • Test GitHub credentials                                       │
│                                                                  │
│  Preview (Vercel)                                                │
│  • Same as production environment                                │
│  • PR-specific URLs                                              │
│                                                                  │
│  Production (Vercel)                                             │
│  • Production MongoDB                                            │
│  • Production GitHub App                                         │
│  • Custom domain                                                 │
└─────────────────────────────────────────────────────────────────┘
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Build Configuration

### next.config.ts

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    serverActions: {
      bodySizeLimit: '50mb',  // For large image uploads
    },
  },
};

export default nextConfig;
```

### Build Scripts

```json
{
  "scripts": {
    "dev": "next dev --turbo",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  }
}
```

| Script | Purpose |
|--------|---------|
| `dev --turbo` | Development with Turbopack (fast HMR) |
| `build` | Production build |
| `start` | Production server (for self-hosting) |
| `lint` | ESLint checks |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Deployment Checklist

### Pre-Deployment

- [ ] All environment variables set in Vercel
- [ ] MongoDB Atlas IP allowlist includes `0.0.0.0/0` (Vercel uses dynamic IPs)
- [ ] GitHub App installed on target organizations
- [ ] Domain configured (if custom)

### Post-Deployment Verification

- [ ] Login flow works
- [ ] Token refresh works
- [ ] Portfolio editing works
- [ ] Image uploads work
- [ ] Admin panel accessible to admins


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Rollback Procedure

### Via Vercel Dashboard

1. Go to Vercel Dashboard → Project → Deployments
2. Find the last working deployment
3. Click "..." → "Promote to Production"
4. Confirm rollback

### Rollback Time

- **Instant** - Vercel maintains all deployment artifacts
- **Zero downtime** - Atomic swap to previous version


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Monitoring & Observability

### Current State

| Capability | Status |
|------------|--------|
| **Deployment Logs** | ✅ Vercel Dashboard |
| **Runtime Logs** | ✅ Vercel Dashboard (limited) |
| **Error Tracking** | ❌ Not implemented |
| **Performance Metrics** | ❌ Not implemented |
| **Uptime Monitoring** | ❌ Not implemented |

### Recommended Additions

1. **Sentry** - Error tracking and alerting
2. **Vercel Analytics** - Core Web Vitals monitoring
3. **Better Uptime/Pingdom** - Uptime monitoring
4. **LogTail/Axiom** - Log aggregation


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Infrastructure Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         USER REQUEST                                  │
│                              │                                        │
└──────────────────────────────┼────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    CLOUDFLARE DNS (Optional)                          │
│                         sourceshan.vercel.app                         │
└──────────────────────────────┼────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        VERCEL EDGE NETWORK                            │
│                      (30+ Global Locations)                           │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                    Edge Middleware                               │ │
│  │                                                                 │ │
│  │  • JWT Verification (JOSE)                                      │ │
│  │  • Token Refresh                                                │ │
│  │  • RBAC Checks                                                  │ │
│  │                                                                 │ │
│  │  Result: Continue to Lambda OR Reject (401/403)                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │               Vercel Serverless Functions                        │ │
│  │                    (AWS Lambda)                                  │ │
│  │                                                                 │ │
│  │  • Next.js API Routes                                           │ │
│  │  • Server Components                                            │ │
│  │                                                                 │ │
│  │  Cold Start: ~100-500ms                                         │ │
│  │  Warm: ~10-50ms                                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
└──────────────────────────────┼────────────────────────────────────────┘
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐
│   MongoDB Atlas    │ │   GitHub API       │ │   Vercel CDN       │
│                    │ │                    │ │                    │
│ Region: Auto       │ │ api.github.com     │ │ Static Assets      │
│ Cluster: M0/M10    │ │                    │ │ _next/static/*     │
│                    │ │ Rate: 5000/hr      │ │                    │
└────────────────────┘ └────────────────────┘ └────────────────────┘
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Security Considerations

### Secrets Management

| Secret | How Stored | Rotation |
|--------|------------|----------|
| `JWT_SECRET` | Vercel env var | Manual (recommended: 90 days) |
| `REFRESH_TOKEN_SECRET` | Vercel env var | Manual (recommended: 90 days) |
| `GITHUB_PRIVATE_KEY` | Vercel env var | GitHub App regeneration |
| `MONGODB_URI` | Vercel env var | Database password rotation |

### Network Security

- MongoDB Atlas: IP allowlist + authentication
- GitHub App: Private key authentication
- Vercel: HTTPS enforced, edge security


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Scaling Considerations

### Current Limits

| Resource | Limit | Notes |
|----------|-------|-------|
| **Vercel Serverless** | 100 concurrent | Hobby plan |
| **MongoDB Connections** | 100 | M0 cluster |
| **GitHub API** | 5000/hour | App rate limit |

### Scaling Path

1. **Vercel Pro** - 1000 concurrent executions
2. **MongoDB M10+** - More connections, dedicated resources
3. **GitHub API** - Higher limits with paid GitHub plan


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Disaster Recovery

### Single Points of Failure

| Component | Risk | Mitigation |
|-----------|------|------------|
| MongoDB Atlas | Data loss | Automated backups (Atlas) |
| GitHub Repos | Content loss | Git history, can clone |
| Vercel | Deployment loss | Redeploy from Git |

### Backup Strategy

| Data | Backup Method | Frequency |
|------|---------------|-----------|
| User Data | MongoDB Atlas automated | Continuous |
| Portfolio Content | Git (distributed) | Every commit |
| Environment Config | Documented | Manual |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Summary

### Deployment Stack

| Layer | Technology |
|-------|------------|
| **CDN/Edge** | Vercel Edge Network |
| **Compute** | Vercel Serverless (AWS Lambda) |
| **Database** | MongoDB Atlas |
| **Storage** | GitHub Repositories |
| **CI/CD** | Vercel automatic deployments |

### Operational Readiness

| Capability | Status |
|------------|--------|
| **Automated Deployments** | ✅ Ready |
| **Instant Rollback** | ✅ Ready |
| **Environment Management** | ✅ Ready |
| **Monitoring** | ⚠️ Basic (needs improvement) |
| **Alerting** | ❌ Not implemented |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*[← Back to Testing & Quality](08-testing-quality.md) | [Continue to Results & Impact →](10-results-impact.md)*
