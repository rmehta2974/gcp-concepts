# GKE Ingress & Service Mesh

## Overview

Multitenant ingress and service mesh design choices for GKE: GKE Ingress, Anthos Service Mesh (ASM), or open-source Istio. Clear trade-offs for each.

---

## Ingress Options

```mermaid
flowchart TB
    Ingress[Ingress Options]
    Ingress --> GKEIng[GKE Ingress]
    Ingress --> ASM[Anthos Service Mesh]
    Ingress --> NGINX[NGINX Ingress]
    
    GKEIng --> |"Native, simple"| Use1[Standard apps]
    ASM --> |"mTLS, advanced"| Use2[Zero trust, multi-cluster]
    NGINX --> |"Custom config"| Use3[Specific needs]
```

---

## GKE Ingress (Native)

- **Backend**: Envoy-based; uses proxy-only subnet
- **Features**: L7 routing, TLS, backend auth
- **Multitenant**: One Ingress per tenant or shared with host-based routing

### Multitenant Ingress Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Host-based** | One Ingress; different hosts (tenant1.example.com) | Shared LB; tenant isolation by host |
| **Path-based** | One Ingress; paths (/tenant1/, /tenant2/) | Shared LB; path routing |
| **Per-tenant Ingress** | Separate Ingress per tenant | Strong isolation; separate certs |

---

## Service Mesh Choices

```mermaid
flowchart TD
    Q1{Need mTLS?}
    Q1 -->|No| GKE[GKE Ingress only]
    Q1 -->|Yes| Q2{Multi-cluster?}
    
    Q2 -->|Yes| ASM[Anthos Service Mesh]
    Q2 -->|No| Q3{Managed or self-hosted?}
    
    Q3 -->|Managed| ASM
    Q3 -->|Self-hosted| Istio[Istio OSS]
```

---

## Anthos Service Mesh (ASM) vs Istio OSS

| Aspect | ASM | Istio OSS |
|--------|-----|-----------|
| **Management** | Google-managed control plane | Self-managed |
| **Multi-cluster** | Native | Complex |
| **Integration** | GKE, Cloud Run | Any K8s |
| **Cost** | Per vCPU | Free (ops cost) |
| **Support** | Google support | Community |

---

## Service Mesh Design

```mermaid
flowchart TB
    subgraph Mesh["Service Mesh"]
        CP[Control Plane]
        SP[Sidecar Proxies]
    end
    
    subgraph Services["Services"]
        S1[Service A]
        S2[Service B]
        S3[Service C]
    end
    
    CP --> SP
    S1 --> SP
    S2 --> SP
    S3 --> SP
    SP <-->|mTLS| SP
```

**Benefits**: mTLS, traffic management, observability, policy (authz).

---

## Multitenant Ingress Diagram

```mermaid
flowchart TB
    LB[Global HTTP/S LB]
    
    subgraph Ingress["GKE Ingress"]
        I[Ingress Controller]
    end
    
    subgraph Namespaces["Namespaces (Tenants)"]
        NS1[tenant-a]
        NS2[tenant-b]
        NS3[tenant-c]
    end
    
    LB --> I
    I --> NS1
    I --> NS2
    I --> NS3
```

**Host-based**: `tenant-a.example.com` → `tenant-a` namespace; `tenant-b.example.com` → `tenant-b`.

---

## Clear Choices Summary

| Need | Choice |
|------|--------|
| Simple L7 routing | GKE Ingress |
| mTLS, zero trust | ASM or Istio |
| Multi-cluster mesh | ASM |
| Cost-sensitive, single cluster | Istio OSS |
| Multitenant | Host-based Ingress + Network Policy per namespace |
