# Hybrid Data Storage Pattern

## Context

**Feature:** Architectural Data Strategy  
**Complexity:** Medium (Conceptual)  
**Technologies Used:** MongoDB, GitHub API, Octokit

**The Philosophy:**
Use the right storage for each data type - databases for secrets and relationships, Git repositories for content and assets.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

Conventional wisdom says "put everything in the database":

```
Traditional Approach:
┌───────────────────────────────────────────────────────────────┐
│                      SINGLE DATABASE                           │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ users             │ credentials, roles, sessions         │  │
│  │ documents         │ JSON blobs of content                │  │
│  │ images            │ Binary blobs or S3 paths             │  │
│  │ audit_logs        │ Change history                       │  │
│  │ versions          │ Previous document states             │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘

Problems:
- Binary storage is inefficient in databases
- Version history is manually implemented
- No deployment pipeline for content
- Backup strategies differ by data type
```

**The Question:**
What if we used Git for what Git does best?


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Pattern

**"Credentials in Database, Content in Git"**

```
Hybrid Approach:
┌─────────────────────────────────────────────────────────────────┐
│  MONGODB (Secure, Fast)          GITHUB REPOS (Versioned, CDN)  │
│  ┌───────────────────────┐       ┌───────────────────────────┐ │
│  │ users                 │       │ user1/portfolio           │ │
│  │ ├─ credentials        │       │ ├─ data/                  │ │
│  │ ├─ roles              │       │ │   ├─ projects.json      │ │
│  │ └─ portfolio_links ───┼───────┼─│   └─ about.json         │ │
│  │                       │       │ └─ public/images/         │ │
│  │ settings              │       │     ├─ hero.webp          │ │
│  │ └─ app configuration  │       │     └─ project1.webp      │ │
│  └───────────────────────┘       └───────────────────────────┘ │
│                                                                 │
│  Perfect for:                    Perfect for:                   │
│  - Authentication                - Portfolio content            │
│  - Authorization                 - Images and assets            │
│  - Fast lookups                  - Version history              │
│  - Sensitive data                - Auto-deployment              │
└─────────────────────────────────────────────────────────────────┘
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Why This Works

### Git as a Database

Git is actually a **content-addressable file system** with:
- Built-in versioning (every change is tracked)
- Diff capabilities (see what changed)
- Merge capabilities (handle conflicts)
- Distributed backup (multiple remotes)
- Webhooks (trigger actions on change)

### Database Where It Matters

Keep in the database:
- **Credentials** (hashed passwords)
- **Relationships** (user → portfolios)
- **Session tokens** (if not JWT)
- **App settings** (configuration)

### Repository Where It Shines

Keep in Git repos:
- **Content** (JSON documents, markdown)
- **Assets** (images, videos)
- **History** (free with Git)
- **Deployable output** (static sites)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        APPLICATION                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     API ROUTES                                │  │
│  │                                                               │  │
│  │  /api/auth/*  ──────────→  MongoDB                            │  │
│  │    - Login                   - Verify credentials             │  │
│  │    - Refresh                 - Get user roles                 │  │
│  │                                                               │  │
│  │  /api/portfolio/*  ─────→  GitHub API                         │  │
│  │    - Get content             - Fetch from repo                │  │
│  │    - Save content            - Commit to repo                 │  │
│  │    - Upload image            - Create blob + commit           │  │
│  │                                                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ↓                       ↓                       ↓
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│   MongoDB     │       │  GitHub Repo  │       │  GitHub Repo  │
│   (Atlas)     │       │  (User 1)     │       │  (User 2)     │
│               │       │               │       │               │
│ users:        │       │ data/         │       │ data/         │
│  - user1 ─────┼──────→│  projects.json│       │  projects.json│
│  - user2 ─────┼───────┼───────────────┼──────→│  about.json   │
│               │       │ public/       │       │ public/       │
└───────────────┘       │  images/      │       │  images/      │
                        └───────────────┘       └───────────────┘
                                │
                                ↓
                        ┌───────────────┐
                        │ Auto-Deploy   │
                        │ (Vercel/      │
                        │  Netlify)     │
                        └───────────────┘
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Benefits

### 1. Free Version Control

```
Git History:
commit abc123 - "Update project description"
commit def456 - "Add new project"
commit ghi789 - "Initial portfolio setup"

↓ Free rollback, diff, blame
```

Compare to implementing this in a database:
- Custom versions table
- Manual snapshot logic
- Storage costs for each version

### 2. Auto-Deployment Pipeline

```
Content Change → Git Commit → Webhook → Vercel Build → Live Site
                     ↑
              No custom pipeline needed!
```

### 3. Content Delivery

GitHub raw files can be served via CDN:
```
https://raw.githubusercontent.com/user/portfolio/main/public/images/hero.webp
                                                          ↑
                                    Direct CDN access to assets
```

### 4. Multi-Tenant Isolation

Each user's content in separate repository:
- Complete data isolation
- Independent access control
- Easy export (clone the repo)
- Clean deletion (delete the repo)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Trade-offs

### What You Gain

1. **Version History for Free**
   - Every change is a Git commit
   - Full audit trail
   
2. **Auto-Deployment**
   - Git push triggers deploy
   - No custom CI/CD for content
   
3. **Tenant Isolation**
   - Separate repos per user
   - Clean separation of concerns
   
4. **Developer Experience**
   - Familiar Git workflow
   - Can edit content via GitHub UI

### What You Give Up

1. **Query Flexibility**
   ```
   // ❌ Can't do in Git:
   SELECT * FROM portfolios 
   WHERE category = 'web' 
   ORDER BY created_at DESC
   
   // Must: Load JSON, filter in code
   ```

2. **Speed for Writes**
   ```
   Git commit: ~200-500ms
   Database write: ~10-50ms
   
   Acceptable for content, not for sessions
   ```

3. **Rate Limits**
   ```
   GitHub API: 5,000 requests/hour (app)
   MongoDB: Unlimited
   
   Need to be thoughtful about API usage
   ```

4. **Complexity**
   ```
   Two data stores = two failure modes
   Two backup strategies
   Two monitoring setups
   ```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## When to Use This Pattern

| Use Case | Recommendation |
|----------|----------------|
| Portfolio/Blog content | ✅ Perfect fit |
| Documentation sites | ✅ Great fit |
| CMS with deployable output | ✅ Good fit |
| E-commerce products | ⚠️ Consider (high update frequency) |
| Real-time data | ❌ Don't use Git |
| High-frequency updates | ❌ Git is too slow |
| Complex queries | ❌ Need real database |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation Checklist

### MongoDB (Credentials Store)
- [ ] User credentials (hashed passwords)
- [ ] Role assignments (admin/user)
- [ ] Portfolio mappings (user → repo)
- [ ] App settings and configuration

### GitHub (Content Store)
- [ ] Portfolio JSON files
- [ ] Image and media assets
- [ ] Schema definitions
- [ ] Auto-deploy via Vercel/Netlify

### Application Logic
- [ ] Auth routes use MongoDB
- [ ] Content routes use GitHub API
- [ ] Batch commits for multi-file updates
- [ ] Caching for frequently accessed content


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Samples

- [GitHub Batch Commits](../backend/github-batch-commits.md) - How content is saved
- [Serverless Connection Pool](../backend/serverless-connection-pool.md) - MongoDB handling


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [Git as a NoSQL Database](https://www.git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Jamstack Architecture](https://jamstack.org/)
- [GitHub as a CMS](https://www.netlify.com/blog/2020/04/14/use-github-as-a-cms/)
- [Polyglot Persistence](https://martinfowler.com/bliki/PolyglotPersistence.html)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Pattern complexity: Medium | Education value: High | Interview relevance: High*
