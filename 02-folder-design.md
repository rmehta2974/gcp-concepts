# GCP Folder Design

## Overview

Folders provide logical grouping, policy inheritance, and IAM boundaries. A clear folder design is essential for governance and cost allocation.

---

## Resource Hierarchy

```mermaid
flowchart TB
    Org[Organization]
    
    subgraph Platform["Platform (Central)"]
        P1[Logging Project]
        P2[Security Project]
        P3[Network Host Project]
    end
    
    subgraph Workloads["Workloads"]
        subgraph Dev["Dev"]
            D1[App A Dev]
            D2[App B Dev]
        end
        subgraph Stage["Stage"]
            S1[App A Stage]
            S2[App B Stage]
        end
        subgraph Prod["Prod"]
            PR1[App A Prod]
            PR2[App B Prod]
        end
    end
    
    subgraph Sandbox["Sandbox"]
        SB[Experimentation]
    end
    
    Org --> Platform
    Org --> Workloads
    Org --> Sandbox
    Workloads --> Dev
    Workloads --> Stage
    Workloads --> Prod
```

---

## Folder Design Choices

### Choice 1: By Environment (Recommended for most)

```
Organization
├── Platform
│   ├── logging
│   ├── security
│   └── network
├── Dev
│   ├── project-app-a-dev
│   └── project-app-b-dev
├── Stage
│   ├── project-app-a-stage
│   └── project-app-b-stage
├── Prod
│   ├── project-app-a-prod
│   └── project-app-b-prod
└── Sandbox
```

**When to use**: Standard dev/stage/prod; environment-based policies.

---

### Choice 2: By Business Unit

```
Organization
├── Platform
├── BU-Finance
│   ├── Dev
│   ├── Stage
│   └── Prod
├── BU-Marketing
│   ├── Dev
│   └── Prod
└── BU-Engineering
    ├── Dev
    ├── Stage
    └── Prod
```

**When to use**: Cost allocation by BU; BU-specific policies.

---

### Choice 3: By Product / Team

```
Organization
├── Platform
├── Product-Core
│   ├── dev
│   ├── stage
│   └── prod
├── Product-Analytics
│   ├── dev
│   └── prod
└── Product-Data
    ├── dev
    └── prod
```

**When to use**: Product-centric ownership; team autonomy.

---

## Decision Matrix

| Factor | By Environment | By BU | By Product |
|--------|----------------|-------|------------|
| Cost allocation | By env | By BU | By product |
| Policy inheritance | Env-based | BU-based | Product-based |
| IAM complexity | Lower | Medium | Medium |
| VPC SC perimeters | Per env | Per BU | Per product |
| Best for | Most orgs | Large enterprises | Product-led orgs |

---

## Folder Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Folder | `lowercase-with-hyphens` | `platform`, `prod-apps` |
| Project | `{env}-{app}-{purpose}` | `prj-prod-app-a`, `prj-dev-data` |
| Max depth | 4–5 levels | Org → Folder → Subfolder → Project |

---

## IAM at Folder Level

- **Platform folder**: Central SAs, logging, security roles
- **Environment folders**: Environment-specific roles (e.g., prod approvers)
- **Project level**: Application-specific SAs, workload identity

```mermaid
flowchart LR
    subgraph IAM
        A[Org Admin] --> B[Folder Admin]
        B --> C[Project Owner]
        C --> D[Workload SA]
    end
```

---

## Diagram: Full Hierarchy Example

```mermaid
flowchart TB
    O[Organization]
    O --> P[Platform]
    O --> W[Workloads]
    O --> S[Sandbox]
    
    P --> P1[logging-prj]
    P --> P2[security-prj]
    P --> P3[network-host-prj]
    
    W --> Dev[Dev]
    W --> Stg[Stage]
    W --> Prd[Prod]
    
    Dev --> D1[prj-app-a-dev]
    Dev --> D2[prj-app-b-dev]
    Stg --> S1[prj-app-a-stage]
    Prd --> PR1[prj-app-a-prod]
    Prd --> PR2[prj-app-b-prod]
    
    S --> SB[prj-sandbox]
```

---

## Next Steps

- [04-network-design.md](./04-network-design.md) — Network design
- [09-centralized-logging-iam.md](./09-centralized-logging-iam.md) — Central logging project
