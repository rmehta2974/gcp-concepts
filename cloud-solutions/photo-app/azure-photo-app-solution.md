# Azure Photo App Solution (1–10 MB, Auth, Inference, Analytics)

End-to-end design for photo app: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. 1–10 MB rapid ingest, auth, storage, categorization, Computer Vision inference, browsing, sharing, Synapse analytics.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Subscription Structure

```
Management Group
├── photo-dev-subscription
├── photo-qa-subscription
└── photo-prod-subscription
```

**Why**: Separate subscriptions per environment; Azure Policy enforces private endpoints. Dev/QA use smaller quotas.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Container Apps / Functions** | Small scale | Medium | High, multi-zone |
| **Blob** | 1 TB | 10 TB | Unlimited |
| **Computer Vision** | Dev quota | QA quota | Prod quota |
| **Purview** | Scan | Classify | Enforce |
| **Defender** | Foundational | Standard | Standard |

---

## 2. Network Design

### 2.1 Ingress

- **Front Door** → **WAF** (rate limit, geo) → **Application Gateway** → **Container Apps** / **Functions**
- **SAS URL** for direct client → Blob upload (no proxy)
- **Entra ID** for auth
- **Why**: WAF protects APIs; SAS avoids bandwidth on app tier.

### 2.2 Egress

- **NAT Gateway** for Functions/Container Apps egress
- **Private Endpoints** for Blob, Computer Vision, Cosmos DB (no NAT for Azure APIs)
- **NSG**: Allow Azure endpoints; restrict third-party to approved IPs
- **Why**: Centralized egress; no exfiltration.

### 2.3 East-West

- **Functions** → **Cosmos DB** → **Computer Vision**: Private subnets; Private Endpoints
- **AKS** (if used): Network Policy; NSG
- **Purview**: Classify sensitive data
- **Why**: Micro-segmentation; user data cannot leak.

### 2.4 On-Prem to Azure

- **Site-to-Site VPN** or **ExpressRoute** for admin/backup from DC
- **Why**: Optional; primary users are internet.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Azure DevOps | Block if keys in repo |
| **ACR scan** | Image scanning | Block critical CVEs |
| **Managed Identity** | No secrets | OIDC for GitHub |
| **Entra** | Auth config | No prod credentials in dev |

**Pipeline**: Build → Scan → Push ACR → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Container Apps**: Revision traffic split 10% → 50% → 100%
- **AKS**: Application Gateway backend pool 90/10
- **Why**: Limit impact of bad release.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Container Apps / Functions** | Upload API | Scale; stateless |
| **Blob Storage** | Photo storage | 1–10 MB; SAS upload |
| **Event Grid** | Blob → Function | Trigger processing |
| **Computer Vision** | Labels, objects | Categorization |
| **Cosmos DB** | Metadata, user data | Low latency |
| **Synapse** | Analytics DW | Management reports |
| **Entra ID** | User auth | Social, email, MFA |
| **Purview** | Data governance | Exfiltration prevention |
