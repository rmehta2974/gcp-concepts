# GCP Connectivity Patterns

## Overview

Connectivity covers project-to-project, organization-level, on-premises, and cloud-to-cloud patterns. Choose based on isolation, cost, and compliance.

---

## Connectivity Matrix

| Source | Destination | Option | Use Case |
|--------|-------------|--------|----------|
| Project A | Project B (same org) | Shared VPC | Same VPC; no peering |
| Project A | Project B (same org) | VPC Peering | Different VPCs |
| Project A | Project B (different org) | VPC Peering (if allowed) | Cross-org |
| On-prem | GCP | Cloud VPN / Interconnect | Hybrid |
| GCP | AWS/Azure | Partner Interconnect / VPN | Multi-cloud |
| GCP | Google APIs | PSC / Public | Private vs public API access |

---

## Project-to-Project (Same Org)

```mermaid
flowchart LR
    subgraph SharedVPC["Shared VPC Host Project"]
        VPC[VPC]
    end
    
    subgraph SP1["Service Project A"]
        A[Workload A]
    end
    
    subgraph SP2["Service Project B"]
        B[Workload B]
    end
    
    A --> VPC
    B --> VPC
    VPC --> A
    VPC --> B
```

**Mechanism**: Shared VPC — service projects attach subnets to host project's VPC. No peering; same L2 domain.

---

## VPC Peering (Different VPCs)

```mermaid
flowchart LR
    subgraph VPC1["VPC A"]
        A[Workload A]
    end
    
    subgraph VPC2["VPC B"]
        B[Workload B]
    end
    
    A <-->|Peering| B
```

**Limitations**: No transitive peering; CIDR must not overlap. Use for cross-project when Shared VPC not used.

---

## Organization-Level Connectivity

```mermaid
flowchart TB
    Org[Organization]
    Org --> F1[Folder 1]
    Org --> F2[Folder 2]
    F1 --> P1[Project A]
    F2 --> P2[Project B]
    
    P1 --> V1[VPC A]
    P2 --> V2[VPC B]
    V1 <-->|Peering| V2
```

**Consider**: Org policies can restrict peering. Use Shared VPC across folders when possible.

---

## On-Premises to GCP

```mermaid
flowchart TB
    subgraph OnPrem["On-Premises"]
        DC[Data Center]
        VPN1[Cloud VPN Gateway]
    end
    
    subgraph GCP["GCP"]
        VPN2[Cloud VPN Gateway]
        VPC[VPC]
    end
    
    DC --> VPN1
    VPN1 <-->|IPsec| VPN2
    VPN2 --> VPC
```

| Option | Latency | Throughput | Cost | Use Case |
|--------|---------|------------|------|----------|
| **Cloud VPN** | Higher | Up to 3 Gbps/tunnel | Lower | Dev, small prod |
| **Dedicated Interconnect** | Lower | 10/100 Gbps | Higher | Production, high throughput |
| **Partner Interconnect** | Medium | 50 Mbps–10 Gbps | Medium | Colo, partner DC |

---

## Cloud-to-Cloud (GCP ↔ AWS/Azure)

```mermaid
flowchart LR
    subgraph GCP["GCP"]
        VPC1[VPC]
    end
    
    subgraph AWS["AWS"]
        VPC2[VPC]
    end
    
    VPC1 <-->|VPN / Partner Interconnect| VPC2
```

**Options**:
- **VPN over internet**: Site-to-site VPN (AWS VPN / Azure VPN Gateway ↔ Cloud VPN)
- **Partner Interconnect**: Via Equinix, Megaport, etc.
- **Anthos / Multi-Cloud**: For container workloads

---

## Private Data Access

| Mechanism | Purpose |
|-----------|---------|
| **Private Google Access** | VMs without public IP reach Google APIs via private path |
| **Private Service Connect (PSC)** | Route to Google APIs (Storage, Pub/Sub) via private IP |
| **Private Service Access (PSA)** | Route to managed services (Cloud SQL, Memorystore) via private IP |

---

## Diagram: Full Connectivity Overview

```mermaid
flowchart TB
    subgraph OnPrem["On-Premises"]
        DC[Data Center]
    end
    
    subgraph GCP["GCP Org"]
        subgraph Proj1["Project 1"]
            V1[VPC]
        end
        subgraph Proj2["Project 2"]
            V2[VPC]
        end
        PSC[PSC Endpoint]
        PSA[PSA Peering]
    end
    
    subgraph Google["Google APIs / Managed"]
        APIs[APIs]
        CloudSQL[Cloud SQL]
    end
    
    DC <-->|VPN/Interconnect| V1
    V1 <-->|Shared VPC or Peering| V2
    V1 --> PSC
    V1 --> PSA
    PSC --> APIs
    PSA --> CloudSQL
```

---

## Next Steps

- [06-private-access-endpoints.md](./06-private-access-endpoints.md) — PSC, PSA details
- [09-centralized-logging-iam.md](./09-centralized-logging-iam.md) — Cross-project access
