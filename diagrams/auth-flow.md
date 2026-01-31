# Authentication Flow Diagram

## Complete Auth Flow with Token Refresh

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant Edge as Edge Middleware
    participant API as API Route
    participant DB as MongoDB

    Note over User,DB: Login Flow
    
    User->>Browser: Enter credentials
    Browser->>API: POST /api/auth/login
    API->>DB: Find user by username
    DB-->>API: User document
    API->>API: bcrypt.compare(password)
    
    alt Password Valid
        API->>API: Sign access token (15m)
        API->>API: Sign refresh token (7d)
        API-->>Browser: 200 OK + Set-Cookie (HttpOnly)
        Browser->>Browser: Store cookies
        Browser-->>User: Redirect to dashboard
    else Password Invalid
        API-->>Browser: 401 Unauthorized
        Browser-->>User: Show error
    end

    Note over User,DB: Protected Request (Token Valid)
    
    User->>Browser: Access protected page
    Browser->>Edge: GET /editor (with cookies)
    Edge->>Edge: Verify access token (JOSE)
    Edge-->>API: Continue (token valid)
    API-->>Browser: 200 OK + Page content
    Browser-->>User: Render page

    Note over User,DB: Protected Request (Token Expired)
    
    User->>Browser: Access protected page
    Browser->>Edge: GET /editor (with cookies)
    Edge->>Edge: Verify access token (JOSE)
    Edge->>Edge: Access token expired!
    Edge->>Edge: Verify refresh token (JOSE)
    
    alt Refresh Token Valid
        Edge->>Edge: Sign new access token
        Edge-->>API: Continue + Set new token cookie
        API-->>Browser: 200 OK + Set-Cookie
        Browser-->>User: Render page (seamless)
    else Refresh Token Invalid
        Edge-->>Browser: 302 Redirect to /login
        Browser-->>User: Login page
    end
```

## Token Lifecycle

```mermaid
stateDiagram-v2
    [*] --> NoTokens: Initial State
    
    NoTokens --> HasTokens: Login Success
    HasTokens --> AccessExpired: 15 minutes
    AccessExpired --> HasTokens: Silent Refresh
    AccessExpired --> NoTokens: Refresh Failed
    HasTokens --> NoTokens: Logout
    HasTokens --> NoTokens: 7 days (Refresh Expires)
    
    note right of HasTokens
        Access: Valid (15m)
        Refresh: Valid (7d)
    end note
    
    note right of AccessExpired
        Access: Expired
        Refresh: Still Valid
        (Silent refresh occurs)
    end note
```

## Cookie Configuration

| Cookie | HttpOnly | Secure | SameSite | Max-Age | Purpose |
|--------|----------|--------|----------|---------|---------|
| accessToken | ✅ | ✅ (prod) | None | 15 min | Short-lived auth |
| refreshToken | ✅ | ✅ (prod) | None | 7 days | Session continuity |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Related: [Feature: Dual-Token Rotation](../docs/04-key-features.md#feature-2-dual-token-rotation-system)*
