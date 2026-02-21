# 🛡️ SourceShan — Edge-First Multi-Tenant Portfolio Management

> **Protecting at the Edge before the server even wakes up.**

![SourceShan Hero Banner](screenshots/banner.webp)
*SourceShan's admin interface — a multi-tenant portfolio control panel with edge-first security.*

---

## 📖 Project Overview

**SourceShan** is a multi-tenant portfolio management system that enables multiple clients to independently manage their own portfolio websites through a unified control panel. Born from the real pain of manually editing client portfolios for every small change, SourceShan transforms a freelancer's workflow from "client requests → developer edits code → manual deploy" into "client logs in → edits content → changes are live."

The platform's defining engineering achievement is its **Edge Fortress Authentication** — JWT verification happens at Vercel's CDN layer using the JOSE library, meaning invalid tokens are rejected before the Node.js runtime even wakes up. This results in zero compute cost for unauthorized requests and dramatically reduces the attack surface. The system uses a dual-token architecture (15-minute access + 7-day refresh) with silent rotation, ensuring users never experience session interruptions.

SourceShan also features a **Schema-Driven Form Engine** that generates dynamic editing forms from JSON schemas. When a new field is added to the portfolio schema, the UI updates automatically — zero additional code required. Portfolio updates are committed atomically to GitHub repositories using the Git Trees API, ensuring data integrity across multiple file operations in a single commit.

---

## ❓ Problem Statement

As a freelance developer building portfolios for clients, a recurring workflow emerged:

> *"Do I have to come back to you every time I want to add a project?"*

Every edit — adding a project, swapping an image, changing a bio — required the developer to manually edit code and redeploy. This created four critical challenges:

**1. Security at Scale**
Multiple clients sharing one platform means a single vulnerability could compromise everyone's data. Complete data isolation is non-negotiable.

**2. Serverless Connection Exhaustion**
Vercel's Lambda architecture creates a new database connection per request. Under load, this quickly exhausts MongoDB's connection pool limits, causing cascading failures.

**3. Atomic Multi-File Operations**
A portfolio update often involves multiple files simultaneously (JSON data + images). GitHub's Contents API creates N commits for N files — not atomic. A partial failure leaves the portfolio in a broken state.

**4. Edge Runtime Constraints**
The industry-standard `jsonwebtoken` library doesn't work in Vercel's Edge Runtime. Security verification — the most frequent operation — needs to run at the edge for performance, but the tooling wasn't designed for it.

---

## 🚀 Solution & Approach

SourceShan is built around a **5-Layer Security Architecture** that protects data at every level from the CDN edge to the database.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     🛡️ EDGE LAYER (Vercel CDN)                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           middleware.ts (JOSE JWT Verification)            │  │
│  │  • Token validation before Node.js wakes up               │  │
│  │  • Automatic token refresh during request lifecycle        │  │
│  │  • RBAC enforcement (admin/client roles)                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    💻 APPLICATION LAYER                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ API Routes  │  │   React     │  │    Service Layer        │ │
│  │ /api/auth/* │  │ Components  │  │ • lib/github.ts         │ │
│  │ /api/admin/*│  │ • MainEditor│  │ • lib/auth.ts           │ │
│  │ /api/portfolio│ │ • AdminPanel│ │ • lib/mongodb.ts        │ │
│  └──────┬──────┘  └─────────────┘  └──────────┬──────────────┘ │
└─────────┼─────────────────────────────────────┼─────────────────┘
          │                                     │
          ▼                                     ▼
┌──────────────────────┐              ┌──────────────────────┐
│      MongoDB         │              │    GitHub Repos      │
│  • Users collection  │              │  • Portfolio JSON    │
│  • Auth data         │              │  • Images/assets     │
│  • Portfolio refs    │              │  • Version history   │
└──────────────────────┘              └──────────────────────┘
```

### The 5 Security Layers

| Layer | Technology | Purpose |
|:-----:|:-----------|:--------|
| **1. Edge** | JOSE JWT in Vercel Middleware | Reject invalid tokens at CDN — zero server compute |
| **2. API** | Route-level auth guards | Validate permissions per endpoint |
| **3. RBAC** | Admin/Client role enforcement | Data isolation between tenants |
| **4. Bcrypt** | 10-round password hashing | Industry-standard credential storage |
| **5. HttpOnly** | Secure cookie configuration | XSS and CSRF prevention |

### Key Technical Decisions

**Edge JWT with JOSE + jsonwebtoken Hybrid:**
`jsonwebtoken` doesn't work in Edge Runtime. Solution: use `jose` for Edge verification (hot path — most frequent), `jsonwebtoken` for Node.js signing (cold path — less frequent). Best of both worlds.

**GitHub Batch Commit with Git Trees API:**
Instead of N commits for N files, the `batchCommit()` function reconstructs the entire Git tree and creates a single atomic commit. All changes land together, or none do.

**Global Connection Pooling:**
TypeScript global augmentation caches the MongoDB connection promise across Lambda invocations. Survives cold starts, HMR, and race conditions. No more "Too Many Connections" errors.

---

## ✨ Features

- **🛡️ Edge Fortress Authentication** — JWT verification at Vercel's CDN using JOSE. Invalid tokens rejected before Node.js wakes up — zero compute cost for bad requests.
- **📐 Schema-Driven Form Engine** — Dynamic forms generated from JSON schemas with 7+ specialized field renderers. New schema fields appear in the UI automatically.
- **🔄 GitHub Batch Commit System** — Atomic multi-file operations using Git Trees API. Single commit for all changes, full tree reconstruction for deletions.
- **🔗 Global Connection Pooling** — Serverless-safe MongoDB connections via TypeScript global augmentation. Promise caching prevents race conditions across Lambda invocations.
- **🔐 Dual-Token Authentication** — 15-minute access + 7-day refresh tokens with silent rotation. Users never experience session interruptions.
- **👥 Multi-Tenant Data Isolation** — Complete separation between client portfolios with RBAC enforcement at every layer.
- **🖱️ Drag-and-Drop Reordering** — Framer Motion-powered reordering of portfolio sections and projects.
- **🌍 Bilingual Support** — Arabic/English content editing with proper RTL layout.
- **📸 Staged Image Uploads** — PendingMediaContext stages uploads before commit, preventing orphaned assets.
- **🖼️ Sharp Image Processing** — Server-side image optimization before storage.
- **📋 Admin Dashboard** — User management, portfolio oversight, and system monitoring for administrators.

---

## 🛠️ Technologies Used

| Category | Technologies |
|:---------|:-------------|
| **Frontend** | Next.js 16 (App Router), React 19, TypeScript (Strict Mode) |
| **Styling** | Tailwind CSS v4, Framer Motion (Drag & Drop) |
| **Auth & Security** | JOSE (Edge JWT), jsonwebtoken (Node.js), Bcrypt (10 rounds), HttpOnly Cookies |
| **Database** | MongoDB Atlas, Global Connection Pooling |
| **Storage** | GitHub API (Octokit), Git Trees API (Atomic Commits) |
| **Image Processing** | Sharp (Server-side optimization) |
| **Deployment** | Vercel (Edge Functions + Serverless) |
| **Auth Provider** | GitHub App (Installation-based auth) |

---

## 📸 Screenshots / Visuals

![Login Page](screenshots/login.webp)
*Secure Login — Edge-authenticated entry point with dual-token session management.*

![Admin Dashboard](screenshots/admin-dashboard.webp)
*Admin Dashboard — Multi-tenant overview with user management and portfolio monitoring.*

![Portfolio Editor](screenshots/editor.webp)
*Schema-Driven Editor — Dynamic forms generated from JSON schemas with drag-and-drop reordering.*

![User Management](screenshots/user-management.webp)
*User Management — RBAC-enforced administration with complete tenant data isolation.*

---

## 🧪 How to Use / Demo

### Live Demo
👉 Visit **[sourceshan.vercel.app](https://sourceshan.vercel.app)** to view the platform in action.

### Admin Workflow
1. **Login** — Authenticate with admin credentials. JWT tokens are verified at the Edge before reaching the server.
2. **Manage Users** — Create client accounts, assign roles (admin/client), and configure portfolio access.
3. **Edit Portfolio** — Select a client's portfolio. The schema-driven form engine renders the appropriate editing interface.
4. **Modify Content** — Edit text fields, upload images, reorder sections via drag-and-drop. All changes are staged locally.
5. **Commit Changes** — Click "Save" to atomically commit all modifications to the client's GitHub repository in a single commit.

### Client Workflow
1. **Login** — Client authenticates and sees only their own portfolio data (tenant isolation).
2. **Edit Content** — Update bio, add projects, change images using the dynamic form interface.
3. **Publish** — Changes commit atomically to GitHub, triggering automatic deployment of the portfolio site.

---

## 📊 Impact / Results

### Technical Achievements

| Metric | Value |
|:------:|:-----:|
| Lines of Code | **~6,500 TypeScript** |
| Type Coverage | **100% Strict Mode** |
| Components | **53 Modular Files** |
| Security Layers | **5 (Edge → API → RBAC → Bcrypt → HttpOnly)** |
| Token TTL | **15 min Access / 7 day Refresh** |

### Security Improvements
- ✅ **Zero server compute** for invalid tokens — Edge rejection eliminates unnecessary Lambda invocations
- ✅ **15-minute attack window** — Short-lived access tokens minimize exposure
- ✅ **HttpOnly cookies** — Complete XSS protection for token storage
- ✅ **Complete data isolation** — Multi-tenant RBAC prevents cross-client access
- ✅ **Atomic commits** — Git Trees API prevents data corruption from partial failures

### Business Impact
- **Client independence** — No more manual code edits for content updates. Clients manage their own portfolios.
- **Scalability** — Multi-tenant architecture supports unlimited clients on a single deployment.
- **Data integrity** — Atomic batch commits guarantee portfolio consistency across all file operations.
- **Cost efficiency** — Edge rejection of bad requests means zero compute cost for unauthorized access attempts.

---

## 🎓 Conclusion / Takeaways

SourceShan demonstrates that **enterprise-grade security patterns can be implemented in serverless architectures** without compromising developer experience or user flow. The Edge Fortress pattern — verifying JWTs at the CDN before the runtime starts — represents a fundamental shift in how authentication should work in modern web applications.

**Key Insights:**
- **Edge verification is the future of auth** — Moving JWT verification to the CDN edge eliminates compute cost for invalid requests and reduces latency for valid ones.
- **JOSE + jsonwebtoken is the right hybrid** — Using the right tool for each runtime (Edge vs. Node.js) delivers the best of both worlds.
- **Git Trees API enables true atomicity** — GitHub's Contents API is insufficient for multi-file operations. The lower-level Trees API provides the atomic guarantees production systems require.
- **Schema-driven UIs eliminate boilerplate** — Generating forms from JSON schemas means new fields appear automatically, reducing maintenance burden to zero.
- **Global connection pooling is essential for serverless** — Without it, every Lambda cold start creates a new connection, quickly exhausting database limits.

SourceShan is a production-grade case study in building **secure, scalable, multi-tenant SaaS** on modern serverless infrastructure.

---

## 🔗 References / Links

- 🌐 **Live Demo:** [sourceshan.vercel.app](https://sourceshan.vercel.app)
- 🌐 **Portfolio:** [codeshan.vercel.app](https://codeshan.vercel.app)
- 🐙 **GitHub:** [github.com/codeshan-1](https://github.com/codeshan-1)
- 💼 **LinkedIn:** [linkedin.com/in/codeshan](https://www.linkedin.com/in/codeshan/)
- 📚 **Full Documentation:** [Case Study Docs](docs/01-overview.md)

---

*Built with 💜 by **CodeShan***
