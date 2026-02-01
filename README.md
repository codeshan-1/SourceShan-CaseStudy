<div align="center">

<!-- Header with SourceShan colors -->
![SourceShan](https://readme-typing-svg.demolab.com?font=Fira+Code&size=28&duration=3000&pause=1000&color=4A45EA&center=true&vCenter=true&width=550&lines=%F0%9F%9B%A1%EF%B8%8F+SourceShan+Case+Study;Edge-First+Security+Architecture;Next.js+16+%7C+MongoDB+%7C+GitHub+API)

<br/>

### 🔐 Multi-Tenant Portfolio Management System

> **Protecting at the Edge before the server even wakes up**

<p>
  <a href="https://sourceshan.vercel.app">
    <img src="https://img.shields.io/badge/🌐_Live_Demo-sourceshan.vercel.app-5042a5?style=for-the-badge"/>
  </a>
  <a href="#-documentation">
    <img src="https://img.shields.io/badge/📚_Docs-Explore-4a45ea?style=for-the-badge"/>
  </a>
  <a href="#-quick-stats">
    <img src="https://img.shields.io/badge/📊_Stats-View-33325c?style=for-the-badge"/>
  </a>
</p>

<br/>

![SourceShan Banner](screenshots/banner.webp)

<br/>

[![Tech Stack](https://img.shields.io/badge/Stack-Next.js%2016%20%7C%20React%2019%20%7C%20MongoDB-5042a5?style=flat-square)]()
[![Security](https://img.shields.io/badge/Security-Edge%20Auth%20%7C%20Dual%20Token-4a45ea?style=flat-square)]()
[![TypeScript](https://img.shields.io/badge/TypeScript-Strict%20Mode-3178C6?style=flat-square)]()
[![Status](https://img.shields.io/badge/Status-Production-33325c?style=flat-square)]()

</div>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## 📊 Quick Stats

<div align="center">

<table>
  <tr>
    <td align="center">
      <h3>🛡️</h3>
      <b>Security Layers</b><br/>
      <sub>5 Levels</sub>
    </td>
    <td align="center">
      <h3>👨‍💻</h3>
      <b>Role</b><br/>
      <sub>Solo Full-Stack</sub>
    </td>
    <td align="center">
      <h3>💻</h3>
      <b>Lines of Code</b><br/>
      <sub>~6,500 TypeScript</sub>
    </td>
    <td align="center">
      <h3>🧩</h3>
      <b>Components</b><br/>
      <sub>53 Files</sub>
    </td>
    <td align="center">
      <h3>⚡</h3>
      <b>Token TTL</b><br/>
      <sub>15 min Access</sub>
    </td>
  </tr>
</table>

</div>

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## 🛠️ Technology Stack

<div align="center">

**🌟 Frontend**
<p>
<img src="https://img.shields.io/badge/Next.js_16-060615?style=for-the-badge&logo=next.js&logoColor=white"/>
<img src="https://img.shields.io/badge/React_19-5042a5?style=for-the-badge&logo=react&logoColor=white"/>
<img src="https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white"/>
<img src="https://img.shields.io/badge/Tailwind_v4-4a45ea?style=for-the-badge&logo=tailwind-css&logoColor=white"/>
</p>

**🔐 Security & Auth**
<p>
<img src="https://img.shields.io/badge/JOSE-Edge_JWT-5042a5?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Bcrypt-Hashing-4a45ea?style=for-the-badge"/>
<img src="https://img.shields.io/badge/HttpOnly-Cookies-33325c?style=for-the-badge"/>
</p>

**💾 Backend & Storage**
<p>
<img src="https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white"/>
<img src="https://img.shields.io/badge/GitHub_API-181717?style=for-the-badge&logo=github&logoColor=white"/>
<img src="https://img.shields.io/badge/Octokit-5042a5?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Sharp-4a45ea?style=for-the-badge"/>
</p>

**☁️ Infrastructure**
<p>
<img src="https://img.shields.io/badge/Vercel-Edge_Functions-060615?style=for-the-badge&logo=vercel&logoColor=white"/>
<img src="https://img.shields.io/badge/GitHub_App-Secure_Auth-181717?style=for-the-badge&logo=github"/>
</p>

</div>

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## 🎯 The Story

### 🌍 The Context

I was building portfolios for clients manually. Every edit—adding a project, changing an image—had to go through me. Client sends a request, I edit the code, deploy. This wasn't scalable.

SourceShan was born from a simple question: *"Do I have to come back to you every time I want to add a project?"*

That sparked an entire platform—a control panel letting each client manage their portfolio independently.

### ⚡ The Challenge

Building multi-tenant with **complete data isolation** while maintaining seamless UX:

| Challenge | Description |
|:---------:|-------------|
| 🔐 **Security at Scale** | Multiple clients, one vulnerability could compromise everyone |
| ⚡ **Serverless Constraints** | Lambda architecture risks database connection exhaustion |
| 🔄 **Atomic Operations** | Portfolio updates involve multiple files (JSON + images) |
| 📐 **Schema Evolution** | New fields shouldn't require code changes |

### 🚀 The Solution

<table>
<tr>
<td width="33%" valign="top">

#### 🛡️ Edge Fortress
JWT verification at CDN level using JOSE. Invalid tokens rejected before Node.js wakes up. Zero compute cost for bad requests.

</td>
<td width="33%" valign="top">

#### 🔄 Dual-Token System
15-minute access + 7-day refresh tokens. Silent rotation maintains sessions. Users never experience interruptions.

</td>
<td width="33%" valign="top">

#### 📐 Schema-Driven UI
Forms generated dynamically from JSON schemas. New fields appear automatically—zero additional code.

</td>
</tr>
</table>

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     🛡️ EDGE LAYER (Vercel CDN)                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           middleware.ts (JOSE JWT Verification)            │  │
│  │  • Token validation before Node.js                        │  │
│  │  • Automatic token refresh                                │  │
│  │  • RBAC enforcement (admin/client)                        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    💻 APPLICATION LAYER                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ API Routes  │  │   React     │  │    Service Layer        │  │
│  │ /api/auth/* │  │ Components  │  │ • lib/github.ts         │  │
│  │ /api/admin/*│  │ • MainEditor│  │ • lib/auth.ts           │  │
│  │ /api/portfolio│ │ • Admin    │  │ • lib/mongodb.ts        │  │
│  └──────┬──────┘  └─────────────┘  └──────────┬──────────────┘  │
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

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## ✨ Key Features

<table>
<tr>
<td width="50%" valign="top">

### 🛡️ Edge Fortress Authentication
JWT verification at Vercel's Edge using JOSE. Invalid tokens rejected at CDN—zero server wake-up.

**Technical Highlights:**
- Custom `middleware.ts` in Edge Runtime
- JOSE for Edge-compatible JWT
- Automatic token refresh during request lifecycle
- RBAC with admin/client roles

**Impact:** Zero-cost rejection of invalid requests

</td>
<td width="50%" valign="top">

### 📐 Schema-Driven Form Engine
Dynamic forms generated from JSON schemas automatically.

**Technical Highlights:**
- 7+ specialized field renderers
- Framer Motion drag-and-drop reordering
- Bilingual support (Arabic/English) with RTL
- PendingMediaContext for staged uploads

**Impact:** Schema changes = automatic UI updates

</td>
</tr>
<tr>
<td width="50%" valign="top">

### 🔄 GitHub Batch Commit System
Atomic multi-file operations using Git Trees API.

**Technical Highlights:**
- Single commit for multiple file operations
- Full tree reconstruction for deletions
- Installation ID caching
- Support for text and binary content

**Impact:** Data integrity guaranteed, cleaner history

</td>
<td width="50%" valign="top">

### 🔗 Global Connection Pooling
Serverless-safe database connections.

**Technical Highlights:**
- TypeScript global augmentation
- Promise caching prevents race conditions
- Survives Lambda cold starts and HMR
- Automatic reconnection on failure

**Impact:** No "Too Many Connections" errors

</td>
</tr>
</table>

<br/>

### 🔐 Dual-Token Authentication

| Token | Duration | Storage | Protection |
|:-----:|:--------:|:-------:|:----------:|
| 🔑 **Access Token** | 15 minutes | HttpOnly cookie | XSS prevention |
| 🔄 **Refresh Token** | 7 days | HttpOnly cookie | CSRF prevention |

- Silent rotation maintains session without interruption
- Edge middleware handles refresh transparently

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## 💡 Challenges & Solutions

<table>
<tr>
<td align="center" width="25%">🌐</td>
<td><b>Edge Runtime Limitations</b></td>
</tr>
<tr>
<td colspan="2">

**Problem:** `jsonwebtoken` doesn't work in Edge Runtime.

**Solution:** Use `jose` for Edge verification, `jsonwebtoken` for Node.js signing. Verification is more frequent, so Edge handles the hot path.

**Result:** Best of both worlds—Edge speed, mature signing library.

</td>
</tr>
<tr>
<td align="center" width="25%">🔄</td>
<td><b>GitHub API Atomicity</b></td>
</tr>
<tr>
<td colspan="2">

**Problem:** Contents API creates N commits for N files—not atomic.

**Solution:** Implemented `batchCommit()` using Git Trees API. All changes in one commit, or none.

**Result:** True atomicity, cleaner history, fewer API calls.

</td>
</tr>
<tr>
<td align="center" width="25%">💾</td>
<td><b>Serverless Database Connections</b></td>
</tr>
<tr>
<td colspan="2">

**Problem:** Vercel Lambdas create new connections per request, exhausting limits.

**Solution:** Global singleton pattern caching connections across invocations.

**Result:** Stable connections, no exhaustion errors.

</td>
</tr>
</table>

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## 📊 Results & Impact

### ⚡ Technical Achievements

<div align="center">

| Metric | Value |
|:------:|:-----:|
| 📝 **Lines of Code** | ~6,500 TypeScript |
| 🔷 **Type Coverage** | 100% Strict Mode |
| 🧩 **Components** | 53 modular files |
| 🛡️ **Security Layers** | 5 (Edge → API → RBAC → Bcrypt → HttpOnly) |

</div>

### ✅ Security Improvements

- ✅ **Zero server compute** for invalid tokens (Edge rejection)
- ✅ **15-minute attack window** (short-lived access tokens)
- ✅ **HttpOnly cookies** (XSS protection)
- ✅ **Complete data isolation** (multi-tenant architecture)
- ✅ **Bcrypt hashing** (10 rounds, industry standard)

### 📈 Business Impact

- 📈 **Client independence** — No more manual edits for content updates
- 📈 **Scalability** — Multi-tenant supports unlimited clients
- 📈 **Data integrity** — Atomic commits prevent corruption

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## 📚 Documentation

<div align="center">

| Document | Description |
|:--------:|:------------|
| [📋 Overview](docs/01-overview.md) | Project context and vision |
| [❓ Problem Statement](docs/02-problem-statement.md) | The challenges solved |
| [🏗️ Solution Architecture](docs/03-solution-architecture.md) | System design deep dive |
| [✨ Key Features](docs/04-key-features.md) | Feature breakdown |
| [🔧 Technical Decisions](docs/05-technical-decisions.md) | ADR-style records |
| [💡 Challenges & Solutions](docs/06-challenges-solutions.md) | Engineering problem-solving |
| [⚡ Performance](docs/07-performance.md) | Optimization strategies |
| [🧪 Testing & Quality](docs/08-testing-quality.md) | QA approach |
| [🚀 Deployment](docs/09-deployment.md) | Infrastructure & CI/CD |
| [📊 Results & Impact](docs/10-results-impact.md) | Metrics and outcomes |

</div>

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## 🖼️ Screenshots

<div align="center">

| 🔐 Login | 🎛️ Admin Dashboard |
|:--------:|:------------------:|
| ![Login](screenshots/login.webp) | ![Dashboard](screenshots/admin-dashboard.webp) |

| 📝 Editor | 👥 User Management |
|:---------:|:------------------:|
| ![Editor](screenshots/editor.webp) | ![Users](screenshots/user-management.webp) |

</div>

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

## ⚠️ Repository Note

<div align="center">

> This is a **case study repository** showcasing technical approach and learnings.  
> The actual project code is proprietary and not publicly available.

</div>

<table align="center">
<tr>
<td align="center" valign="top" width="50%">

### ✅ What's Included
- 📐 Architectural diagrams
- 📝 Technical decision records
- ⚡ Code samples (conceptual)
- 📊 Security analysis

</td>
<td align="center" valign="top" width="50%">

### ❌ What's NOT Included
- 🔒 Actual source code
- 📊 Client data or credentials
- 🔐 Proprietary business logic

</td>
</tr>
</table>

<br/>

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<br/>

<div align="center">

## 🌐 Explore the Live Platform

<a href="https://sourceshan.vercel.app">
  <img src="https://img.shields.io/badge/🚀_Launch_Demo-sourceshan.vercel.app-5042a5?style=for-the-badge"/>
</a>

<br/><br/>

---

**Want to discuss this project?**

<p>
<a href="https://codeshan.vercel.app">
  <img src="https://img.shields.io/badge/Portfolio-codeshan.vercel.app-4a45ea?style=flat-square&logo=vercel"/>
</a>
<a href="https://github.com/codeshan-1">
  <img src="https://img.shields.io/badge/GitHub-codeshan--1-181717?style=flat-square&logo=github"/>
</a>
<a href="https://www.linkedin.com/in/codeshan/">
  <img src="https://img.shields.io/badge/LinkedIn-codeshan-0077B5?style=flat-square&logo=linkedin"/>
</a>
</p>

---

<br/>

Built with 💜 by **CodeShan**

⭐ Star this repo if you found it helpful!

<br/>

<img src="https://capsule-render.vercel.app/api?type=waving&color=5042a5,4a45ea,33325c&height=100&section=footer"/>

</div>
