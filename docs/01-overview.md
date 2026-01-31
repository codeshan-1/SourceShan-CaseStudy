# 01 - Project Overview

## SourceShan: Multi-Tenant Portfolio Management Platform


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Executive Summary

**SourceShan** is a sophisticated multi-tenant portfolio management system that enables clients to independently manage their portfolio websites without touching code. Built on Next.js 15+ with a security-first architecture, it implements Edge-level authentication, schema-driven UI generation, and atomic GitHub commits for portfolio content management.

### At a Glance

| Attribute | Value |
|-----------|-------|
| **Project Type** | Multi-tenant SaaS Platform |
| **Primary Technology** | Next.js 16 (App Router) |
| **Database** | MongoDB (users) + GitHub (content) |
| **Authentication** | Edge Middleware with dual-token rotation |
| **Codebase Size** | ~6,500 lines of TypeScript |
| **Component Count** | 53 TypeScript/TSX files |
| **Architecture** | Layered with Edge-first security |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Origin Story

Every project has an origin story. SourceShan's began with a simple but revealing question from a client:

> *"Do I have to come back to you every time I want to add a project?"*

At the time, I was building portfolios for clients manually. Every edit—adding a project, changing a bio, uploading new images—had to go through me. The workflow looked like this:

1. Client sends me a request via WhatsApp/Email
2. I edit the portfolio code
3. I push to GitHub
4. Vercel deploys
5. I confirm with the client

This workflow had several problems:

- **I became the bottleneck** - Clients had to wait for me to be available
- **Context switching** - Small changes interrupted my flow on bigger projects
- **Error-prone** - Manual edits are error-prone, especially for content
- **Not scalable** - What happens when I have 10 clients? 50?

That question sparked an idea: **What if clients could update their own portfolios?** Not just update content, but truly manage it—add projects, reorder sections, upload images—all through a control panel that's as intuitive as updating a social media profile.

SourceShan was born.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Project Goals

### Primary Goals

1. **Client Independence**  
   Clients should be able to manage their portfolio content without developer intervention.

2. **Multi-Tenant Architecture**  
   Multiple clients should be able to use the same platform with complete data isolation.

3. **Security First**  
   With multiple clients managing sensitive content, security cannot be an afterthought.

4. **Schema Extensibility**  
   Portfolio structures should be evolvable without code changes to the editor.

### Secondary Goals

5. **Premium User Experience**  
   The editor should feel premium—smooth animations, intuitive interactions, no page reloads.

6. **Low Operational Overhead**  
   Once set up, clients should be largely self-sufficient.

7. **Developer Ergonomics**  
   The codebase should be maintainable, type-safe, and well-organized.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem Space

Building a client-facing portfolio management system isn't as simple as adding CRUD endpoints. Several challenges emerge:

### Security Challenges

| Challenge | Description |
|-----------|-------------|
| **Data Isolation** | Client A must never see Client B's data |
| **Token Security** | Session hijacking would expose client content |
| **Role Enforcement** | Admins vs. Clients have different permissions |
| **Attack Surface** | Every API endpoint is a potential vulnerability |

### Technical Challenges

| Challenge | Description |
|-----------|-------------|
| **Serverless Constraints** | Connection pooling in Lambda environments |
| **Atomic Operations** | Multiple file updates must be all-or-nothing |
| **Schema Evolution** | New fields shouldn't require editor code changes |
| **Edge Limitations** | Auth at Edge requires different libraries |

### User Experience Challenges

| Challenge | Description |
|-----------|-------------|
| **Non-Technical Users** | Clients aren't developers—UX must be intuitive |
| **Bilingual Content** | Supporting both Arabic and English content |
| **Image Management** | Uploads, previews, and batch processing |
| **Mobile Experience** | Many clients check on mobile |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Solution

SourceShan addresses these challenges through a layered architecture with several innovative patterns:

### Edge-First Security

Authentication happens at Vercel's Edge network—before requests even reach the Node.js runtime. Using JOSE for JWT verification, invalid tokens are rejected at the CDN level with zero compute cost.

### Dual-Token Rotation

A dual-token system balances security with user experience:
- **Access Token**: 15-minute lifetime for security
- **Refresh Token**: 7-day lifetime for convenience
- **Silent Rotation**: Tokens refresh automatically without interruption

### Schema-Driven Forms

The editor UI is generated dynamically from JSON schemas. When a new field is added to the portfolio schema, the editor automatically renders the appropriate input component—no code changes required.

### Atomic GitHub Commits

Portfolio content is stored in GitHub repositories (one per client). Updates use GitHub's Git Trees API for atomic batch commits—all changes succeed or none do.

### Global Connection Pooling

A singleton pattern for MongoDB connections survives Lambda cold starts, preventing the "Too Many Connections" problem common in serverless environments.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Technical Stack Overview

```
┌──────────────────────────────────────────────────────────────┐
│                        FRONTEND                              │
│  Next.js 16 • React 19 • Framer Motion • Tailwind CSS v4    │
│  Geist Font • Lucide Icons • CVA • clsx                      │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                     EDGE LAYER                               │
│  JOSE • Edge Middleware • RBAC • Token Rotation              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                       BACKEND                                │
│  Next.js API Routes • jsonwebtoken • bcryptjs                │
│  Mongoose • Octokit • Sharp                                  │
└──────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌────────────────────┐           ┌────────────────────┐
│     MongoDB        │           │   GitHub Repos     │
│  (Users & Auth)    │           │ (Portfolio Data)   │
└────────────────────┘           └────────────────────┘
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Key Innovations

### 1. "Zero Server Wake-Up" Security
Invalid tokens are rejected at the Edge before Node.js even starts. This means:
- No compute cost for attack attempts
- Faster rejection response times
- Reduced attack surface

### 2. "The Database Writes the Frontend"
Schema-driven forms mean the database schema dictates what the editor shows. Add a field to the schema, it appears in the editor automatically.

### 3. "One Commit to Rule Them All"
Batch commits using Git Trees API ensure atomic operations. Whether updating 1 file or 10, it's a single commit—or nothing.

### 4. "Paranoid Security" Philosophy
Every layer assumes the previous layer might have been bypassed:
- Edge verifies tokens
- API routes re-verify
- Database queries are scoped
- RBAC is enforced


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Project Outcomes

| Outcome | Description |
|---------|-------------|
| **Client Independence** | Clients update portfolios without developer help |
| **Security Posture** | 5-layer security with Edge-first architecture |
| **Developer Experience** | Type-safe, well-organized, maintainable codebase |
| **Extensibility** | New features require minimal code changes |
| **Performance** | Edge auth + connection pooling for optimal response times |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## What's Next?

This documentation dives deep into each aspect of SourceShan:

1. **[Problem Statement](02-problem-statement.md)** - Detailed breakdown of challenges
2. **[Solution Architecture](03-solution-architecture.md)** - How the system is designed
3. **[Key Features](04-key-features.md)** - Feature-by-feature deep dive
4. **[Technical Decisions](05-technical-decisions.md)** - Why we made specific choices

Each document provides the technical depth needed to understand both the "what" and the "why" of this project.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*[Continue to Problem Statement →](02-problem-statement.md)*
