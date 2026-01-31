# Deployment Pipeline

## Vercel Deployment Flow

SourceShan uses **Vercel** for automated deployments with git-based workflow.

```mermaid
flowchart LR
    subgraph Development["👨‍💻 Development"]
        LOCAL["Local Dev<br/>npm run dev"]
        COMMIT["Git Commit"]
    end
    
    subgraph GitHub["📦 GitHub"]
        PUSH["Push to Branch"]
        PR["Pull Request"]
        MAIN["Merge to Main"]
    end
    
    subgraph Vercel["▲ Vercel Platform"]
        DETECT["Webhook Trigger"]
        BUILD["Build Process<br/>next build"]
        DEPLOY_PREVIEW["Preview Deploy<br/>pr-123.vercel.app"]
        DEPLOY_PROD["Production Deploy<br/>sourceshan.vercel.app"]
    end
    
    subgraph PostDeploy["✅ Post-Deploy"]
        EDGE["Edge Network<br/>Global CDN"]
        FUNCTIONS["Serverless Functions<br/>API Routes"]
        MONITOR["Monitoring<br/>Analytics"]
    end
    
    LOCAL --> COMMIT
    COMMIT --> PUSH
    PUSH --> PR
    PUSH --> DETECT
    PR --> MAIN
    MAIN --> DETECT
    
    DETECT --> BUILD
    BUILD --> |"PR Branch"| DEPLOY_PREVIEW
    BUILD --> |"Main Branch"| DEPLOY_PROD
    
    DEPLOY_PROD --> EDGE
    DEPLOY_PROD --> FUNCTIONS
    DEPLOY_PROD --> MONITOR
    
    classDef dev fill:#e3f2fd,stroke:#1976d2
    classDef git fill:#f3e5f5,stroke:#7b1fa2
    classDef vercel fill:#000,stroke:#fff,color:#fff
    classDef post fill:#e8f5e9,stroke:#388e3c
    
    class LOCAL,COMMIT dev
    class PUSH,PR,MAIN git
    class DETECT,BUILD,DEPLOY_PREVIEW,DEPLOY_PROD vercel
    class EDGE,FUNCTIONS,MONITOR post
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Pipeline Stages Detail

### 1. Local Development

```bash
# Start development server
npm run dev

# Run linting
npm run lint

# Type checking
npm run type-check
```

### 2. Git Workflow

```mermaid
gitGraph
    commit id: "Initial"
    branch feature/new-component
    checkout feature/new-component
    commit id: "Add component"
    commit id: "Add tests"
    checkout main
    merge feature/new-component id: "PR #42"
    commit id: "Deploy" type: HIGHLIGHT
```

### 3. Vercel Build Process

```mermaid
flowchart TB
    subgraph Build["Build Phase (~60s)"]
        INSTALL["npm install<br/>Install dependencies"]
        CACHE["Cache Check<br/>node_modules, .next/cache"]
        COMPILE["next build<br/>Compile + Optimize"]
        STATIC["Static Generation<br/>Pre-render pages"]
    end
    
    subgraph Output["Build Output"]
        HTML["Static HTML<br/>Pre-rendered pages"]
        CHUNKS["JS Chunks<br/>Code-split bundles"]
        API["API Functions<br/>Serverless handlers"]
        MIDDLEWARE["Edge Middleware<br/>Auth verification"]
    end
    
    INSTALL --> CACHE
    CACHE --> |"Cache Hit"| COMPILE
    CACHE --> |"Cache Miss"| COMPILE
    COMPILE --> STATIC
    STATIC --> HTML
    STATIC --> CHUNKS
    STATIC --> API
    STATIC --> MIDDLEWARE
```

### 4. Deployment Types

| Branch | Deploy Type | URL | Visibility |
|--------|-------------|-----|------------|
| `main` | Production | `sourceshan.vercel.app` | Public |
| PR branches | Preview | `pr-xxx.vercel.app` | Team link |
| Feature branches | Preview | `feature-xxx.vercel.app` | Team link |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Environment Configuration

### Environment Variables

```mermaid
flowchart LR
    subgraph Secrets["🔐 Vercel Secrets"]
        direction TB
        MONGO["MONGODB_URI"]
        JWT["JWT_SECRET"]
        REFRESH["REFRESH_TOKEN_SECRET"]
        GH_APP["GITHUB_APP_*"]
    end
    
    subgraph Scopes["📍 Environment Scopes"]
        DEV["Development<br/>Local .env"]
        PREVIEW["Preview<br/>PR deployments"]
        PROD["Production<br/>Live site"]
    end
    
    Secrets --> DEV
    Secrets --> PREVIEW
    Secrets --> PROD
```

### Variable Management

| Variable | Development | Preview | Production |
|----------|-------------|---------|------------|
| `MONGODB_URI` | Local/Dev Atlas | Dev Atlas | Prod Atlas |
| `JWT_SECRET` | Dev secret | Dev secret | Prod secret |
| `GITHUB_APP_ID` | Test App | Test App | Prod App |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Rollback Strategy

```mermaid
flowchart LR
    ISSUE["🚨 Issue Detected"]
    IDENTIFY["Identify Bad Deploy"]
    ROLLBACK["Instant Rollback<br/>Promote Previous"]
    VERIFY["Verify Recovery"]
    HOTFIX["Optional: Hotfix<br/>New Deploy"]
    
    ISSUE --> IDENTIFY
    IDENTIFY --> ROLLBACK
    ROLLBACK --> VERIFY
    VERIFY --> |"Issue Persists"| IDENTIFY
    VERIFY --> |"Resolved"| HOTFIX
```

### Rollback Methods

1. **Instant Rollback (Vercel Dashboard)**
   - Click "Promote to Production" on previous deployment
   - Takes effect in ~10 seconds
   - Zero downtime

2. **Git Revert**
   ```bash
   git revert HEAD
   git push origin main
   # Triggers new deploy with reverted code
   ```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Monitoring & Alerts

```mermaid
flowchart TB
    subgraph Vercel["Vercel Monitoring"]
        ANALYTICS["Web Analytics<br/>Page views, Vitals"]
        LOGS["Function Logs<br/>API execution"]
        SPEED["Speed Insights<br/>Core Web Vitals"]
    end
    
    subgraph External["External Monitoring"]
        UPTIME["Uptime Robot<br/>Availability check"]
        SENTRY["Sentry<br/>Error tracking"]
    end
    
    subgraph Alerts["Alert Channels"]
        EMAIL["Email"]
        DISCORD["Discord Webhook"]
    end
    
    ANALYTICS --> EMAIL
    LOGS --> EMAIL
    SPEED --> EMAIL
    UPTIME --> DISCORD
    SENTRY --> EMAIL
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Build Performance

| Metric | Value | Notes |
|--------|-------|-------|
| Install time | ~15s | With cache |
| Build time | ~45s | Full rebuild |
| Deploy time | ~10s | Edge propagation |
| **Total** | **~70s** | Main branch to live |

### Optimization Techniques

1. **Dependency Caching**: `node_modules` cached between builds
2. **Build Caching**: `.next/cache` persisted
3. **Incremental Builds**: Only changed pages rebuilt
4. **Edge Functions**: <20ms cold start


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## CI/CD Checklist

- [x] Automatic deploys on push
- [x] Preview deployments for PRs
- [x] Environment variable management
- [x] Instant rollback capability
- [x] Build caching enabled
- [ ] Automated tests pre-deploy (TODO)
- [ ] Lighthouse CI integration (TODO)
- [ ] Slack/Discord notifications (TODO)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Related: [Solution Architecture](../docs/03-solution-architecture.md) | [Deployment Docs](../docs/09-deployment.md)*
