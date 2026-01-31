# System Architecture Diagram

## SourceShan Multi-Tier Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        Browser["Browser / Mobile"]
    end

    subgraph Edge["Vercel Edge Network"]
        Middleware["middleware.ts<br/>JWT Verification (JOSE)<br/>Token Refresh<br/>RBAC"]
    end

    subgraph Application["Next.js Application Layer"]
        subgraph Routes["API Routes"]
            AuthRoutes["Auth Routes<br/>/api/auth/*"]
            AdminRoutes["Admin Routes<br/>/api/admin/*"]
            PortfolioRoutes["Portfolio Routes<br/>/api/portfolio/*"]
        end
        
        subgraph Services["Service Layer"]
            AuthLib["lib/auth.ts<br/>JWT Signing"]
            GitHubLib["lib/github.ts<br/>Octokit Integration"]
            MongoLib["lib/mongodb.ts<br/>Connection Pool"]
        end
    end

    subgraph Data["Data Layer"]
        MongoDB[("MongoDB Atlas<br/>Users & Auth")]
        GitHub[("GitHub Repos<br/>Portfolio Content")]
    end

    Browser --> Middleware
    Middleware -->|Valid Token| Routes
    Middleware -.->|Invalid Token| Browser
    
    AuthRoutes --> AuthLib
    AdminRoutes --> MongoLib
    PortfolioRoutes --> GitHubLib
    PortfolioRoutes --> MongoLib
    
    AuthLib --> MongoDB
    MongoLib --> MongoDB
    GitHubLib --> GitHub
```

## Component Description

| Layer | Component | Technology | Responsibility |
|-------|-----------|------------|----------------|
| **Edge** | Middleware | JOSE, Next.js Middleware | Auth verification, token refresh |
| **API** | Auth Routes | jsonwebtoken, bcrypt | Login, logout, token signing |
| **API** | Admin Routes | Mongoose | User CRUD, portfolio assignment |
| **API** | Portfolio Routes | Octokit | Content management, batch commits |
| **Services** | auth.ts | jsonwebtoken | Token creation and verification |
| **Services** | github.ts | Octokit | GitHub API interactions |
| **Services** | mongodb.ts | Mongoose | Connection pooling |
| **Data** | MongoDB | Atlas M0 | User credentials, portfolios |
| **Data** | GitHub | Repos | Portfolio JSON, images |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Related: [Solution Architecture](../docs/03-solution-architecture.md)*
