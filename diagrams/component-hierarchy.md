# Component Hierarchy

## SourceShan React Component Tree

```mermaid
graph TD
    subgraph App["🌐 Application Root"]
        RootLayout["RootLayout<br/>ThemeProvider + UIProvider"]
    end
    
    subgraph Auth["🔐 Auth Flow"]
        LoginPage["LoginPage"]
        LoginForm["LoginForm<br/>username/password"]
    end
    
    subgraph Public["🌍 Public Site (AstroShan)"]
        HomePage["HomePage"]
        HeroSection["HeroSection<br/>Animated Background"]
        ProjectsSection["ProjectsSection<br/>Featured Work"]
        SkillsSection["SkillsSection<br/>Tech Stack"]
        StorySection["StorySection<br/>Journey Timeline"]
        ContactSection["ContactSection<br/>Social Links"]
    end
    
    subgraph Editor["✏️ Editor System (SourceShan)"]
        EditorLayout["EditorLayout<br/>Sidebar + Content"]
        PortfolioSelector["PortfolioSelector<br/>Dropdown"]
        SectionList["SectionList<br/>Navigation"]
        EditorContent["EditorContent<br/>Form Container"]
    end
    
    subgraph FormEngine["📝 Schema-Driven Forms"]
        DynamicForm["DynamicForm<br/>Iterates Schema"]
        FieldRenderer["FieldRenderer<br/>Type Detection"]
    end
    
    subgraph FieldTypes["🧩 Field Components"]
        BilingualTextField["BilingualTextField<br/>AR/EN Side-by-Side"]
        ImageInput["ImageInput<br/>Upload + Preview"]
        BilingualListEditor["BilingualListEditor<br/>Drag-and-Drop"]
        DetailedContentEditor["DetailedContentEditor<br/>Content Blocks"]
        StringArrayEditor["StringArrayEditor<br/>Tag List"]
        Toggle["Toggle<br/>Boolean Fields"]
    end
    
    subgraph Contexts["⚡ React Contexts"]
        PendingMediaContext["PendingMediaContext<br/>Staged Uploads"]
        UIContext["UIContext<br/>Toasts + Modals"]
        ThemeContext["ThemeContext<br/>Dark/Light Mode"]
    end
    
    RootLayout --> Auth
    RootLayout --> Public
    RootLayout --> Editor
    RootLayout --> Contexts
    
    LoginPage --> LoginForm
    
    HomePage --> HeroSection
    HomePage --> ProjectsSection
    HomePage --> SkillsSection
    HomePage --> StorySection
    HomePage --> ContactSection
    
    EditorLayout --> PortfolioSelector
    EditorLayout --> SectionList
    EditorLayout --> EditorContent
    
    EditorContent --> DynamicForm
    DynamicForm --> FieldRenderer
    
    FieldRenderer --> BilingualTextField
    FieldRenderer --> ImageInput
    FieldRenderer --> BilingualListEditor
    FieldRenderer --> DetailedContentEditor
    FieldRenderer --> StringArrayEditor
    FieldRenderer --> Toggle
    
    ImageInput -.-> PendingMediaContext
    BilingualListEditor -.-> PendingMediaContext
    
    classDef page fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef layout fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef component fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef context fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    classDef field fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    
    class LoginPage,HomePage,EditorLayout page
    class RootLayout,EditorContent,DynamicForm layout
    class HeroSection,ProjectsSection,SkillsSection,StorySection,ContactSection,LoginForm,PortfolioSelector,SectionList component
    class PendingMediaContext,UIContext,ThemeContext context
    class FieldRenderer,BilingualTextField,ImageInput,BilingualListEditor,DetailedContentEditor,StringArrayEditor,Toggle field
```

## Component Categories

### 📄 Page Components
Top-level route components that define page structure.

| Component | Route | Description |
|-----------|-------|-------------|
| `LoginPage` | `/login` | Authentication entry point |
| `HomePage` | `/` | Public portfolio landing page |
| `EditorLayout` | `/editor/*` | Protected CMS interface |

### 🎨 Section Components
Major content sections of the public site.

| Component | Purpose | Key Features |
|-----------|---------|--------------|
| `HeroSection` | Introduction | Canvas stars, LED effects |
| `ProjectsSection` | Featured work | Modal previews, galleries |
| `SkillsSection` | Tech stack | Animated skill bars |
| `StorySection` | Journey timeline | Bilingual content |
| `ContactSection` | Social links | Footer with links |

### 📝 Form Components
Schema-driven form system components.

| Component | Detects | Renders |
|-----------|---------|---------|
| `FieldRenderer` | Field type from schema | Appropriate field component |
| `BilingualTextField` | `{ar, en}` object | Side-by-side RTL/LTR inputs |
| `ImageInput` | `format: "image"` | Upload button + preview |
| `BilingualListEditor` | `array + x-format` | Draggable AR/EN pairs |
| `DetailedContentEditor` | `x-format: "detailed-content"` | Content blocks |
| `StringArrayEditor` | `array + items: string` | Tag input |
| `Toggle` | `type: boolean` | On/off switch |

### ⚡ Context Providers
Cross-cutting concerns managed via React Context.

| Context | Scope | Provides |
|---------|-------|----------|
| `PendingMediaContext` | Editor | Image staging, deletion tracking |
| `UIContext` | Global | `showToast()`, `showConfirm()` |
| `ThemeContext` | Global | Dark/light mode toggle |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Data Flow Between Components

```mermaid
flowchart LR
    subgraph User["User Action"]
        Upload["Upload Image"]
        Save["Click Save"]
    end
    
    subgraph Components["React Components"]
        II["ImageInput"]
        PMC["PendingMediaContext"]
        DF["DynamicForm"]
    end
    
    subgraph API["API Layer"]
        Route["POST /api/portfolio/batch"]
        GitHub["GitHub API"]
    end
    
    Upload --> II
    II --> PMC
    PMC --> |"Blob URL"| II
    
    Save --> DF
    DF --> |"Form Data + Pending Images"| Route
    PMC --> |"Staged Files"| Route
    Route --> |"Batch Commit"| GitHub
    
    style PMC fill:#e8f5e9,stroke:#388e3c
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Related: [Schema-Driven Forms](../code-samples/frontend/schema-driven-forms.md) | [Pending Media Context](../code-samples/frontend/pending-media-context.md)*
