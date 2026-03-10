# VMware to GCP Migration

## Overview

Migration from VMware to GCP: lift-and-shift (Compute Engine) or modernize (GKE, serverless). Use Migrate for Compute Engine (formerly Velostrata) for VM migration.

---

## Migration Approaches

```mermaid
flowchart TB
    VMware[VMware Workloads]
    VMware --> L1[Lift & Shift]
    VMware --> L2[Replatform]
    VMware --> L3[Refactor]
    
    L1 --> GCE[Compute Engine]
    L2 --> GKE[GKE]
    L3 --> Run[Cloud Run / Serverless]
```

---

## Migrate for Compute Engine

- **What**: Agent-based or agentless migration of VMs to GCE
- **Process**: Replicate → Cutover
- **Use**: VMware, AWS, Azure → GCP

---

## Migration Phases

| Phase | Activities |
|-------|------------|
| **Assess** | Inventory VMs; identify dependencies; right-size |
| **Plan** | Network design; project structure; migration waves |
| **Pilot** | Migrate non-critical VMs; validate |
| **Migrate** | Wave-based migration; cutover |
| **Optimize** | Rightsize; consider GKE/serverless |

---

## Network Considerations

- **VPC**: Design Shared VPC or per-project; align with landing zone
- **Connectivity**: VPN or Interconnect for migration traffic
- **DNS**: Plan DNS cutover; consider Cloud DNS

---

## Diagram: VMware Migration Flow

```mermaid
flowchart LR
    subgraph VMware["VMware"]
        VM1[VM 1]
        VM2[VM 2]
    end
    
    subgraph Migrate["Migrate for Compute"]
        Replicate[Replicate]
        Cutover[Cutover]
    end
    
    subgraph GCP["GCP"]
        GCE1[GCE VM 1]
        GCE2[GCE VM 2]
    end
    
    VM1 --> Replicate
    VM2 --> Replicate
    Replicate --> Cutover
    Cutover --> GCE1
    Cutover --> GCE2
```
