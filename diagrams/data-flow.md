# Data Flow Diagram

## Portfolio Save Operation

```mermaid
flowchart LR
    subgraph Editor["Editor (Client)"]
        Form["Form State"]
        Pending["PendingMediaContext<br/>Staged Images"]
        Delete["Deleted Paths"]
    end

    subgraph FormData["FormData Construction"]
        JSON["JSON Data"]
        Images["Image Files"]
        Deletions["Deletion List"]
    end

    subgraph API["batch-save API Route"]
        Parse["Parse FormData"]
        Process["Process Images<br/>(Sharp → WebP)"]
        Build["Build Operations[]"]
    end

    subgraph GitHub["GitHub API"]
        Blobs["Create Blobs"]
        Tree["Build Tree"]
        Commit["Create Commit"]
        Ref["Update Ref"]
    end

    Form --> JSON
    Pending --> Images
    Delete --> Deletions
    
    JSON --> Parse
    Images --> Parse
    Deletions --> Parse
    
    Parse --> Process
    Process --> Build
    
    Build --> Blobs
    Blobs --> Tree
    Tree --> Commit
    Commit --> Ref
```

## Detailed Batch Commit Flow

```mermaid
sequenceDiagram
    participant Editor
    participant API as batch-save API
    participant Sharp
    participant GH as GitHub API

    Editor->>API: POST FormData<br/>(data + images + deletions)
    
    API->>Sharp: Convert images to WebP
    Sharp-->>API: WebP buffers
    
    API->>API: Build operations array
    
    Note over API,GH: Git Trees API Flow
    
    API->>GH: GET /git/ref/heads/main
    GH-->>API: currentCommitSha
    
    API->>GH: GET /git/commits/{sha}
    GH-->>API: currentTreeSha
    
    loop For each file
        API->>GH: POST /git/blobs
        GH-->>API: blobSha
    end
    
    API->>GH: POST /git/trees<br/>{base_tree, tree entries}
    GH-->>API: newTreeSha
    
    API->>GH: POST /git/commits<br/>{tree, parents, message}
    GH-->>API: newCommitSha
    
    API->>GH: PATCH /git/refs/heads/main<br/>{sha: newCommitSha}
    GH-->>API: Updated ref
    
    API-->>Editor: 200 OK<br/>{commitSha}
```

## Data Persistence Split

```mermaid
flowchart TB
    subgraph App["SourceShan Application"]
        Auth["Authentication"]
        Editor["Portfolio Editor"]
    end

    subgraph MongoDB["MongoDB Atlas"]
        Users[("users collection")]
        UserDoc["User Document<br/>─────────<br/>_id<br/>username<br/>password (hashed)<br/>role<br/>portfolios[]"]
    end

    subgraph GitHub["GitHub Repositories"]
        Repo1["client1/portfolio"]
        Repo2["client2/portfolio"]
        
        subgraph RepoContent["Repository Structure"]
            Data["data/<br/>├── projects.json<br/>├── about.json<br/>└── skills.json"]
            Public["public/<br/>└── images/<br/>    ├── project1.webp<br/>    └── profile.webp"]
        end
    end

    Auth --> Users
    Editor --> GitHub
    
    Users --> UserDoc
    Repo1 --> RepoContent
    Repo2 --> RepoContent
```

## Key Points

| Data Type | Stored In | Reason |
|-----------|-----------|--------|
| User credentials | MongoDB | Sensitive, needs encryption |
| User portfolios list | MongoDB | Links users to repos |
| Portfolio content | GitHub | Version control, auto-deploy |
| Images | GitHub | Part of content, served via CDN |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Related: [Solution Architecture](../docs/03-solution-architecture.md)*
