# GCP Landing Zone Best Practices

## Overview

A **landing zone** is a well-architected, secure, scalable foundation that enables workloads to land on GCP with governance, networking, and security built in from day one.

---

## Landing Zone Workflow

```mermaid
flowchart TB
    subgraph Phase1["Phase 1: Foundation"]
        A[Organization Setup] --> B[Folder Hierarchy]
        B --> C[IAM Baseline]
        C --> D[Billing]
    end
    
    subgraph Phase2["Phase 2: Networking"]
        D --> E[Shared VPC]
        E --> F[Firewall Baseline]
        F --> G[Private Access]
    end
    
    subgraph Phase3["Phase 3: Security"]
        G --> H[VPC Service Controls]
        H --> I[Security Command Center]
        I --> J[Logging Sink]
    end
    
    subgraph Phase4["Phase 4: Workloads"]
        J --> K[Project Factory]
        K --> L[Workload Deployment]
    end
```

---

## Core Principles

| Principle | Description |
|-----------|-------------|
| **Separation of duties** | Central team owns foundation; app teams own workloads |
| **Least privilege** | IAM at folder/project level; avoid org-level broad roles |
| **Defense in depth** | Network + IAM + VPC SC + Binary Auth |
| **Automation first** | Terraform/Pulumi for all foundation; no manual drift |
| **Environment parity** | Dev/stage/prod share same patterns; different isolation |

---

## Landing Zone Components

```mermaid
flowchart LR
    subgraph Org
        O[Organization]
    end
    
    subgraph Folders
        F1[Platform]
        F2[Workloads]
        F3[Sandbox]
    end
    
    subgraph Projects
        P1[Logging]
        P2[Security]
        P3[Network]
        P4[Shared Services]
    end
    
    O --> Folders
    Folders --> Projects
```

---

## Design Scenarios

### Scenario A: Single Business Unit

- **Use case**: One team, limited blast radius
- **Structure**: Org → Environment folders (dev/stage/prod) → Projects
- **Network**: Single Shared VPC or per-project VPC

### Scenario B: Multi-BU, Central Platform

- **Use case**: Platform team provides shared services; BUs own apps
- **Structure**: Org → Platform folder + BU folders → Projects
- **Network**: Shared VPC host project; service projects attach

### Scenario C: Regulated / Multi-Tenant

- **Use case**: Strong isolation (compliance, tenants)
- **Structure**: Org → Tenant/Environment folders → Projects
- **Network**: Per-tenant VPCs; VPC SC per perimeter
- **Security**: VPC SC, Binary Auth, CMEK

---

## Workflow: New Project Onboarding

```mermaid
sequenceDiagram
    participant AppTeam
    participant ProjectFactory
    participant Terraform
    participant GCP
    
    AppTeam->>ProjectFactory: Request project (form/API)
    ProjectFactory->>Terraform: Generate project config
    Terraform->>GCP: Create project, IAM, attach to Shared VPC
    GCP->>Terraform: Project created
    Terraform->>ProjectFactory: Output project ID
    ProjectFactory->>AppTeam: Project ready
```

---

## Checklist: Landing Zone Readiness

- [ ] Organization created with appropriate org policy
- [ ] Folder hierarchy defined (platform vs workloads)
- [ ] Billing accounts linked; budget alerts set
- [ ] Central logging project with org-level sink
- [ ] Shared VPC (or design decision documented)
- [ ] IAM baseline: groups, roles, conditions
- [ ] VPC Service Controls perimeter (if required)
- [ ] Security Command Center enabled
- [ ] Project factory / automation in place

---

## Next Steps

- [02-folder-design.md](./02-folder-design.md) — Folder structure and choices
- [04-network-design.md](./04-network-design.md) — Network design
- [07-security-design-zerotrust.md](./07-security-design-zerotrust.md) — Security baseline
