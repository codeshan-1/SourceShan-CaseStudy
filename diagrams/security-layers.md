# Security Layers Diagram

## Defense in Depth Architecture

```mermaid
flowchart TB
    subgraph Request["Incoming Request"]
        Browser["Browser"]
    end

    subgraph Layer1["LAYER 1: Edge Middleware"]
        Edge["JWT Verification (JOSE)<br/>─────────<br/>• Verify access token<br/>• Silent refresh if expired<br/>• Zero compute for invalid tokens"]
    end

    subgraph Layer2["LAYER 2: HTTPS/Cookies"]
        Cookies["Cookie Security<br/>─────────<br/>• HttpOnly (no JS access)<br/>• Secure (HTTPS only)<br/>• SameSite (CSRF protection)"]
    end

    subgraph Layer3["LAYER 3: Role-Based Access"]
        RBAC["RBAC Enforcement<br/>─────────<br/>• Admin routes require admin role<br/>• Client can't access admin API<br/>• Route-level permissions"]
    end

    subgraph Layer4["LAYER 4: API Validation"]
        APICheck["API Route Checks<br/>─────────<br/>• Additional token verification<br/>• Input validation<br/>• Request sanitization"]
    end

    subgraph Layer5["LAYER 5: Data Scoping"]
        DataScope["Query Scoping<br/>─────────<br/>• Users only see their portfolios<br/>• Ownership verified before ops<br/>• No cross-tenant access"]
    end

    subgraph Protected["Protected Resources"]
        DB[("MongoDB")]
        GH[("GitHub")]
    end

    Browser --> Layer1
    Layer1 -->|Valid| Layer2
    Layer1 -.->|Invalid| Browser
    
    Layer2 --> Layer3
    Layer3 -->|Authorized| Layer4
    Layer3 -.->|Forbidden| Browser
    
    Layer4 --> Layer5
    Layer5 --> Protected

    style Layer1 fill:#e74c3c,color:#fff
    style Layer2 fill:#e67e22,color:#fff
    style Layer3 fill:#f1c40f,color:#000
    style Layer4 fill:#2ecc71,color:#fff
    style Layer5 fill:#3498db,color:#fff
```

## Layer Details

### Layer 1: Edge Middleware (First Defense)

```mermaid
flowchart LR
    Request["Request"] --> Extract["Extract Tokens<br/>from Cookies"]
    Extract --> Verify["Verify with JOSE"]
    
    Verify -->|Valid| Continue["Continue"]
    Verify -->|Expired| Refresh["Try Refresh"]
    Verify -->|Invalid| Reject["Reject (401)"]
    
    Refresh -->|Success| Continue
    Refresh -->|Fail| Reject
```

**Key Insight:** Invalid tokens are rejected at the CDN edge with zero server compute cost.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### Layer 2: Cookie Security

| Protection | HttpOnly | Secure | SameSite |
|------------|----------|--------|----------|
| XSS Token Theft | ✅ | - | - |
| Network Interception | - | ✅ | - |
| CSRF Attacks | - | - | ✅ |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### Layer 3: Role-Based Access Control

```mermaid
flowchart TB
    Token["Token Payload"]
    Token --> Role{"role?"}
    
    Role -->|admin| AdminRoutes["Admin Routes<br/>/api/admin/*<br/>/admin/*"]
    Role -->|client| ClientRoutes["Client Routes<br/>/api/portfolio/*<br/>/editor/*"]
    
    AdminRoutes -.->|if client| Forbidden["403 Forbidden"]
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### Layer 4: API Route Validation

Example checks in each API route:
- Token presence verification
- Payload structure validation
- Input sanitization
- Type checking (TypeScript enforced)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


### Layer 5: Data Scoping

```mermaid
flowchart LR
    Query["Query Request"]
    Query --> Extract["Extract User ID<br/>from Token"]
    Extract --> Scope["Scope Query<br/>userId: token.id"]
    Scope --> DB[("MongoDB")]
```

**Key Insight:** Even if an attacker bypasses outer layers, queries are scoped to the user's own data.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Attack Mitigation

| Attack Type | Mitigating Layer | How |
|-------------|-----------------|-----|
| Invalid Token | Layer 1 (Edge) | JOSE verification |
| Stolen Token | Layer 2 (Cookies) | HttpOnly prevents XSS |
| Privilege Escalation | Layer 3 (RBAC) | Role checks |
| SQL/NoSQL Injection | Layer 4 (API) | Input validation |
| Data Exfiltration | Layer 5 (Scoping) | Query scoping |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Related: [Feature: Edge Authentication](../docs/04-key-features.md#feature-1-edge-fortress-authentication)*
