# Azure Credit Card Transactions Solution (Goldman Sachs)

End-to-end design for credit card transaction processing: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Transaction ingest, authorization, fraud detection, settlement, compliance.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Subscription Structure

```
Management Group
├── cc-dev-subscription
├── cc-qa-subscription
└── cc-prod-subscription
```

**Why**: PCI-DSS requires strict isolation. Separate subscriptions enforce Azure Policy; Prod has Purview and Defender. Dev/QA use tokenized or synthetic data only.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Data** | Synthetic / tokenized | Tokenized | Real (encrypted, restricted) |
| **Throughput** | 1K TPS | 10K TPS | 100K+ TPS |
| **Purview** | Scan | Classify | Enforce + block |
| **Defender** | Foundational | Standard | Standard + Storage |
| **Key Vault** | Standard | Premium | Premium + HSM |
| **Azure Policy** | Relaxed | Stricter | Full (no public Blob) |

---

## 2. Network Design

### 2.1 Ingress

- **Front Door** → **WAF** (rate limit, geo, OWASP) for authorization API
- **Private Endpoint** for acquirer/issuer connections (no public)
- **Why**: Card data must not traverse public internet. Authorization API protected by WAF.

### 2.2 Egress

- **NAT Gateway** for outbound (e.g., network authorization)
- **Private Endpoints** for Azure ML (fraud), Synapse, Blob
- **NSG**: Deny unknown egress; allow only approved acquirer/network IPs
- **Why**: Exfiltration prevention; card data cannot leave perimeter.

### 2.3 East-West

- **Event Hubs** → **Functions** → **Azure ML** (fraud): Private subnets; Private Endpoints
- **AKS/Container Apps** (authorization): Network Policy; NSG
- **Purview**: Classify PAN; enforce policies
- **Why**: PCI segmentation; authorization path isolated from analytics.

### 2.4 On-Prem to Azure

- **ExpressRoute** for core banking / issuer systems (10/100 Gbps)
- **Site-to-Site VPN** for backup
- **Why**: Low latency for authorization; BGP failover.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Azure DevOps | Block if PAN, CVV in repo |
| **ACR scan** | Image scanning | Block critical CVEs |
| **Managed Identity** | No secrets | OIDC for GitHub |
| **PCI scan** | Custom | Block if PAN in image |
| **Azure Policy** | Block public Blob | Enforce at MG level |

**Pipeline**: Build → Scan → Push ACR → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Authorization API**: Application Gateway backend pool 5% → 25% → 100%
- **Fraud model**: Azure ML endpoint 5% traffic to new deployment
- **Settlement batch**: New Data Factory pipeline; compare output before cutover
- **Why**: Limit impact of bad auth logic or model; financial risk mitigation.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Event Hubs** | Transaction event stream | Decouple; scale; ordering |
| **Functions / Data Factory** | Stream processing | Real-time fraud; settlement batch |
| **Azure ML** | Fraud model | Low-latency inference |
| **Azure SQL / Cosmos DB** | Transaction store | Strong consistency; PCI |
| **Synapse** | Analytics (tokenized) | Reporting; no PAN |
| **Purview** | Data governance | PAN classification; policies |
| **Key Vault** | Encryption | PCI; key control |
| **Private Endpoints** | Private Azure APIs | No internet egress |
