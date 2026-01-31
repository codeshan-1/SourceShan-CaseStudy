# 02 - Problem Statement

## The Challenges That Shaped SourceShan


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Introduction

Before building a solution, we need to understand the problem deeply. This document explores the challenges that drove the design of SourceShan—from practical pain points to technical constraints.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Core Problem

### Manual Portfolio Management Doesn't Scale

When I started building portfolios for clients, the workflow was straightforward:

1. Client describes what they want
2. I build the portfolio
3. I hand it over (deployed on Vercel, code on GitHub)
4. Client uses it

But then came the requests:

- *"Can you add this new project?"*
- *"Can you update my bio?"*
- *"Can you swap these images?"*

Each request required me to:
1. Switch context from whatever I was working on
2. Pull the client's repo
3. Make the changes
4. Push and deploy
5. Confirm with the client

For one client, this was manageable. For multiple clients, it became unsustainable.

### The Impact

| Who | Impact |
|-----|--------|
| **Clients** | Had to wait for me. Often days for simple changes. |
| **Me** | Constant context switching. Low-value work. |
| **Business** | Didn't scale. Revenue per client was artificially capped. |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Problem Dimensions

The solution needed to address several interconnected problems:

### 1. User Independence Problem

**Symptom:** Clients can't update their portfolios without developer help.

**Root Cause:** Portfolios were static code, not managed content.

**Requirement:** A system where non-technical clients can:
- Add/edit/delete portfolio items
- Upload and manage images
- Reorder content
- Preview changes
- Publish updates

**Constraint:** Must be intuitive enough for non-technical users who have never used a CMS.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 2. Multi-Tenancy Problem

**Symptom:** Each client needs their own isolated environment.

**Root Cause:** Sharing a platform requires robust data isolation.

**Requirement:** A system where:
- Multiple clients use the same platform
- Each client sees only their data
- One client cannot access another's content
- Admin can oversee all clients

**Constraint:** Isolation must be enforced at every layer—UI, API, and database.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 3. Security Problem

**Symptom:** Opening a system to clients introduces security risks.

**Root Cause:** More users = larger attack surface.

**Requirements:**

| Security Requirement | Why It Matters |
|---------------------|----------------|
| Strong authentication | Prevent unauthorized access |
| Session security | Prevent session hijacking |
| Role-based access | Different permissions for admins vs. clients |
| Token rotation | Minimize damage if tokens are stolen |
| Input validation | Prevent injection attacks |

**Constraint:** Security cannot compromise user experience. Silent token refresh is a must.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 4. Serverless Infrastructure Problem

**Symptom:** Vercel's serverless architecture has unique constraints.

**Root Causes:**
- Each request may spin up a new Lambda instance
- No persistent connections between requests
- Cold starts affect performance

**Specific Challenges:**

```
┌─────────────────────────────────────────────────────────────┐
│                  SERVERLESS CHALLENGES                       │
├─────────────────────────────────────────────────────────────┤
│  • Connection Pooling: 100+ connections quickly exhausted   │
│  • Cold Starts: First request in a Lambda is slow          │
│  • State: No shared memory between invocations             │
│  • Edge Limitations: Fewer APIs available in Edge Runtime  │
└─────────────────────────────────────────────────────────────┘
```

**Requirement:** Architecture must work within these constraints, not fight them.

**Constraint:** Must use Vercel (client portfolios already deployed there).


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 5. Data Consistency Problem

**Symptom:** Portfolio updates often involve multiple files.

**Root Cause:** A single "save" might update:
- `projects.json` (metadata)
- `public/images/project-1.webp` (new image)
- `public/images/old-image.webp` (deleted)

**The Risk:**

```
UPDATE SEQUENCE (Traditional Approach):
1. Update projects.json        ✅ Success
2. Upload new image            ✅ Success
3. Delete old image            ❌ FAILURE!

RESULT: Data is now inconsistent
- projects.json references new image
- New image exists
- Old image still exists (orphaned)
- Client sees broken state
```

**Requirement:** All changes must be atomic—all succeed or none do.

**Constraint:** GitHub's standard API is per-file, not atomic.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 6. Schema Evolution Problem

**Symptom:** Portfolio requirements change over time.

**Examples:**
- Client wants to add testimonials section
- New field needed for video links
- Different portfolio might need different fields

**The Traditional Problem:**

```
Traditional Approach:
1. Client requests new field
2. Update database schema
3. Update API endpoints
4. Update editor UI component
5. Update validation
6. Deploy everything

Time: Hours to days
Risk: High (multiple changes)
```

**Requirement:** Adding new fields should be a configuration change, not a code change.

**Constraint:** Must work for different portfolio structures (each client is unique).


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### 7. User Experience Problem

**Symptom:** Enterprise tools are often complex and ugly.

**Root Cause:** Developer tools prioritize function over form.

**Requirements:**

| UX Requirement | Rationale |
|---------------|-----------|
| Intuitive navigation | Non-technical users must figure it out |
| Visual feedback | Animations, toasts, loading states |
| Bilingual support | Arabic (RTL) and English (LTR) content |
| Mobile responsive | Clients check on their phones |
| Premium aesthetic | Reflects the quality of their portfolio |

**Constraint:** Aesthetic cannot sacrifice performance.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Intersection of Problems

These problems don't exist in isolation—they interact:

```
                    ┌─────────────────┐
                    │   Multi-Tenant  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌───────────┐  ┌───────────┐  ┌───────────┐
        │ Security  │  │  Data     │  │   UX      │
        │           │◄─┤ Isolation │─►│           │
        └─────┬─────┘  └───────────┘  └─────┬─────┘
              │                             │
              │    ┌───────────────┐        │
              └───►│  Performance  │◄───────┘
                   └───────┬───────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
     ┌───────────┐  ┌───────────────┐  ┌───────────┐
     │Serverless │  │  Atomicity   │  │  Schema   │
     │Constraints│  │              │  │ Evolution │
     └───────────┘  └───────────────┘  └───────────┘
```

**Key Intersections:**

- **Security + Multi-Tenancy**: RBAC at every layer
- **UX + Serverless**: Performance despite cold starts
- **Atomicity + Schema**: Complex saves must be atomic
- **Security + UX**: Token refresh must be invisible


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## What Success Looks Like

Before diving into the solution, let's define success criteria:

### Must Have

- [ ] Clients can log in securely
- [ ] Clients can only see their portfolios
- [ ] Clients can update content without code
- [ ] Updates are atomic (no partial saves)
- [ ] System works on Vercel serverless

### Should Have

- [ ] Admins can manage clients
- [ ] Schema changes don't require code changes
- [ ] Editor feels premium (smooth animations)
- [ ] Mobile-responsive design
- [ ] Bilingual support

### Nice to Have

- [ ] Preview before publish
- [ ] Version history
- [ ] Analytics integration
- [ ] Email notifications


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Problem Prioritization

Based on impact and dependencies, the problems were tackled in this order:

| Priority | Problem | Rationale |
|----------|---------|-----------|
| 1 | Security | Foundation. Gets harder to add later. |
| 2 | Multi-Tenancy | Core architecture decision. |
| 3 | Serverless Constraints | Must work within infrastructure. |
| 4 | Atomicity | Data integrity is non-negotiable. |
| 5 | Schema Evolution | Scalability depends on this. |
| 6 | User Experience | Important, but built on top. |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Constraints Summary

The solution must operate within these constraints:

### Technical Constraints

| Constraint | Source |
|------------|--------|
| Must use Vercel | Existing infrastructure |
| Must use GitHub for content | Version control benefit |
| Must work in Edge Runtime | Performance requirement |
| Must use MongoDB | Existing database choice |

### Non-Technical Constraints

| Constraint | Source |
|------------|--------|
| Solo developer | Resource limitation |
| Clients are non-technical | UX requirement |
| Arabic + English content | Market requirement |
| No forced registration | UX philosophy |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Path Forward

Understanding these problems deeply shaped every architectural decision in SourceShan. The next document, **[Solution Architecture](03-solution-architecture.md)**, explains how each problem was addressed.

Key takeaways:
- Problems interact—solving one can create or solve another
- Constraints are features—they force creative solutions
- Security first—retrofitting is harder than building in


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*[← Back to Overview](01-overview.md) | [Continue to Solution Architecture →](03-solution-architecture.md)*
