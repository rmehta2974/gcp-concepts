# Azure Investment & Risk Modeling Solution (Goldman Sachs)

End-to-end design for investment division and risk modeling: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Market data ingest, risk models (VaR, stress testing), portfolio analytics, trading signals, compliance.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Subscription Structure

```
Management Group
├── inv-risk-dev-subscription
├── inv-risk-qa-subscription
└── inv-risk-prod-subscription
```

**Why**: Investment and risk systems handle sensitive market data and models. Separate subscriptions enforce Azure Policy; Prod has Purview and Defender. Dev/QA use anonymized or synthetic market data.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Market data** | Delayed / synthetic | Near real-time (test) | Real-time |
| **Azure ML / Synapse** | Small | Medium | Large; reserved |
| **Purview** | Scan | Classify | Enforce |
| **Defender** | Foundational | Standard | Standard |
| **Key Vault** | Standard | Premium | Premium + HSM |
| **Log retention** | 30 days | 90 days | 7 years (regulatory) |

---

## 2. Network Design

### 2.1 Ingress

- **Front Door** → **WAF** for analytics/trading APIs
- **Private Endpoint** for market data feeds (Bloomberg, Reuters)
- **Why**: Market data and positions must not traverse public internet. API protected by WAF.

### 2.2 Egress

- **NAT Gateway** for outbound to market data providers (approved IPs only)
- **Private Endpoints** for Azure ML, Synapse, Blob
- **NSG**: Deny unknown egress; allow only approved vendor IPs
- **Why**: Exfiltration prevention; models and positions cannot leave perimeter.

### 2.3 East-West

- **Event Hubs** → **Functions** → **Azure ML** (risk models): Private subnets
- **Synapse** (portfolio analytics): Private; no public access
- **AKS/Container Apps** (trading signals): Network Policy; NSG
- **Purview**: Classify model/position data; enforce policies
- **Why**: Micro-segmentation; risk models isolated from trading; audit trail.

### 2.4 On-Prem to Azure

- **ExpressRoute** for core trading systems, market data (10/100 Gbps)
- **Site-to-Site VPN** for backup
- **Why**: Low latency for real-time risk; BGP failover.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Azure DevOps | Block if keys in repo |
| **ACR scan** | Image scanning | Block critical CVEs |
| **Managed Identity** | No secrets | OIDC for GitHub |
| **Model validation** | Azure ML Pipeline | Block if backtest fails |
| **Azure Policy** | Block public Blob | Enforce at MG level |

**Pipeline**: Build → Scan → Train → Evaluate → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Risk model**: Azure ML endpoint 5% traffic to new deployment; compare VaR vs stable
- **Trading signals API**: Application Gateway backend pool 5% → 25% → 100%
- **Data Factory** (risk calc): New pipeline; run in parallel; compare output
- **Why**: Limit impact of bad model; financial risk mitigation.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Event Hubs** | Market data stream | Decouple; scale; ordering |
| **Functions / Data Factory** | Risk calc; VaR; stress | Horizontal scale |
| **Azure ML** | Risk models; ML | VaR; stress; signals |
| **Synapse** | Portfolio analytics | Historical; reporting |
| **Cosmos DB / Azure SQL** | Position store | Strong consistency |
| **Purview** | Data governance | Model/position classification |
| **Key Vault** | Encryption | Regulatory; key control |
| **Blob** | Model artifacts | Versioned; lineage |
| **Azure ML Pipelines** | MLOps | Reproducible; audit |
