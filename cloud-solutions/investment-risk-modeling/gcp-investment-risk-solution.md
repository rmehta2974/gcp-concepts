# GCP Investment & Risk Modeling Solution (Goldman Sachs)

End-to-end design for investment division and risk modeling: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Market data ingest, risk models (VaR, stress testing), portfolio analytics, trading signals, compliance.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Folder and Project Structure

```
Organization
├── Platform
│   ├── prj-logging
│   ├── prj-security
│   └── prj-network-host
├── Dev
│   └── prj-inv-risk-dev
├── QA
│   └── prj-inv-risk-qa
└── Prod
    └── prj-inv-risk-prod
```

**Why**: Investment and risk systems handle sensitive market data and models. Separate projects enforce segmentation; Prod has VPC SC for model and position data. Dev/QA use anonymized or synthetic market data.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Market data** | Delayed / synthetic | Near real-time (test) | Real-time |
| **Vertex / BigQuery** | Small quota | Medium | Large; reserved slots |
| **VPC SC** | No | Optional | Yes |
| **CMEK** | Optional | Yes | Yes (HSM-backed) |
| **Binary Auth** | Audit | Enforce | Enforce |
| **Audit** | 30 days | 90 days | 7 years (regulatory) |

---

## 2. Network Design

### 2.1 Ingress

- **Global HTTP(S) LB** → **Cloud Armor** for analytics/trading APIs
- **Private endpoint** for market data feeds (Bloomberg, Reuters) via PSC or partner
- **IAP** for risk dashboards, model management
- **Why**: Market data and positions must not traverse public internet. API protected by WAF.

### 2.2 Egress

- **Cloud NAT** for outbound to market data providers (approved IPs only)
- **PSC** for Vertex (risk models), BigQuery, Storage
- **Firewall**: Deny unknown egress; allow only approved vendor IPs
- **Why**: Exfiltration prevention; models and positions cannot leave perimeter.

### 2.3 East-West

- **Pub/Sub** → **Dataflow** → **Vertex** (risk models): Private VPC
- **BigQuery** (portfolio analytics): Private; no public access
- **GKE** (trading signals): Network Policy; pod-to-pod restricted
- **VPC SC**: Block copy of models/positions to projects outside perimeter
- **Why**: Micro-segmentation; risk models isolated from trading; audit trail.

### 2.4 On-Prem to GCP

- **Dedicated Interconnect** for core trading systems, market data (10/100 Gbps)
- **HA VPN** for backup
- **Why**: Low latency for real-time risk; BGP failover.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Cloud Build | Block if keys in repo |
| **Container scan** | Artifact Registry | Block critical CVEs |
| **Image signing** | KMS + attestation | Binary Auth enforce |
| **Model validation** | Vertex Pipeline | Block if backtest fails |
| **IAC** | Terraform + Policy | Block public buckets |

**Pipeline**: Build → Scan → Sign → Train → Evaluate → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Risk model**: Vertex endpoint 5% traffic to new model; compare VaR vs stable
- **Trading signals API**: GKE Ingress canary-weight 5% → 25% → 100%
- **Dataflow** (risk calc): New job; run in parallel; compare output
- **Why**: Limit impact of bad model; financial risk mitigation.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Pub/Sub** | Market data stream | Decouple; scale; ordering |
| **Dataflow** | Risk calc; VaR; stress | Horizontal scale; Beam |
| **Vertex AI** | Risk models; ML | VaR; stress; signals |
| **BigQuery** | Portfolio analytics | Historical; reporting |
| **Spanner** | Position store | Strong consistency; global |
| **VPC SC** | Perimeter | Model/position exfiltration prevention |
| **CMEK** | Encryption | Regulatory; key control |
| **Cloud Storage** | Model artifacts | Versioned; lineage |
| **Vertex Pipelines** | MLOps | Reproducible; audit |
