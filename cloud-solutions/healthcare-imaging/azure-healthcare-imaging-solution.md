# Azure Healthcare Imaging Solution (5 PB, Azure ML, OpenAI)

End-to-end design for healthcare imaging: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. 5 PB image ETL, metadata cataloging, Azure ML inference, OpenAI, LLM accuracy monitoring, patient+image+disease training.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Subscription Structure

```
Management Group
├── imaging-dev-subscription
├── imaging-qa-subscription
└── imaging-prod-subscription
```

**Why**: Separate subscriptions isolate PHI; Azure Policy enforces encryption and private endpoints. Dev/QA use sample data; Prod holds full 5 PB.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Storage** | 100 TB (Blob) | 500 TB | 5 PB |
| **Data Factory / Synapse** | Small | Medium | Large, auto-scale |
| **Azure ML endpoint** | Dev | QA | Prod |
| **Purview** | Scan only | Classify | Block + classify |
| **Defender** | Foundational | Standard | Standard + Storage |

---

## 2. Network Design

### 2.1 Ingress

- **Front Door** → **WAF** for admin UIs
- **PACS ingest**: On-prem → ExpressRoute → Private Endpoint (no public)
- **Why**: PHI must not traverse public internet.

### 2.2 Egress

- **NAT Gateway** for Data Factory/Synapse workers
- **Private Endpoints** for Blob, Azure ML, OpenAI (no NAT for Azure APIs)
- **NSG**: Deny unknown egress; allow only Azure endpoints and approved IPs
- **Why**: Exfiltration prevention; Azure traffic stays on backbone.

### 2.3 East-West

- **Data Factory** → **Azure ML** → **Synapse**: Private subnets; Private Endpoints
- **AKS** (if used): Network Policy; NSG
- **Purview**: Classify PHI; enforce policies
- **Why**: Micro-segmentation; PHI cannot leak.

### 2.4 On-Prem to Azure

- **ExpressRoute** for PACS bulk transfer (10/100 Gbps)
- **Site-to-Site VPN** for backup
- **Data Box** / **AzCopy** for batch migration
- **Why**: High throughput; BGP failover.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Azure DevOps | Block if PHI in repo |
| **ACR scan** | Image scanning | Block critical CVEs |
| **Managed Identity** | No secrets | OIDC for GitHub |
| **Azure Policy** | Block public Blob | Enforce at MG level |

**Pipeline**: Build → Scan → Push ACR → Deploy Dev → (approval) → QA → (approval) → Prod

---

## 4. Canary

- **Data Factory**: New pipeline; compare output with stable
- **Azure ML**: New deployment; route 10% via Traffic Manager
- **AKS**: Application Gateway backend pool 90/10 split
- **Why**: Limit impact of bad model or pipeline.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Data Factory / Synapse** | ETL | Scale for 5 PB; Spark |
| **Azure ML** | Inference | Managed; GPU; model monitor |
| **OpenAI** | LLM | Report coding; multimodal |
| **Purview** | Catalog | PHI tagging; governance |
| **Defender for Storage** | Threat detection | Detect PHI exposure |
| **Private Endpoints** | Private Azure APIs | No internet egress |
| **ExpressRoute** | On-prem | Throughput for 5 PB |
