# GCP Credit Card Transactions Solution (Goldman Sachs)

End-to-end design for credit card transaction processing: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Transaction ingest, authorization, fraud detection, settlement, compliance.

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
│   └── prj-cc-dev
├── QA
│   └── prj-cc-qa
└── Prod
    └── prj-cc-prod
```

**Why**: PCI-DSS requires strict isolation. Separate projects enforce segmentation; Prod has VPC SC perimeter for cardholder data. Dev/QA use tokenized or synthetic data only.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Data** | Synthetic / tokenized | Tokenized | Real (encrypted, restricted) |
| **Throughput** | 1K TPS | 10K TPS | 100K+ TPS |
| **VPC SC** | No | Yes (test perimeter) | Yes (full perimeter) |
| **DLP** | Audit | Enforce | Enforce + block |
| **Binary Auth** | Audit | Enforce | Enforce |
| **CMEK** | Optional | Yes | Yes (HSM-backed) |

---

## 2. Network Design

### 2.1 Ingress

- **Global HTTP(S) LB** → **Cloud Armor** (rate limit, geo, OWASP) for authorization API
- **Private endpoint** for acquirer/issuer connections (no public)
- **IAP** for admin; no direct SSH
- **Why**: Card data must not traverse public internet for internal systems. Authorization API protected by WAF.

### 2.2 Egress

- **Cloud NAT** for outbound (e.g., network authorization)
- **PSC** for Vertex (fraud model), BigQuery, Storage
- **Firewall**: Deny unknown egress; allow only approved acquirer/network IPs
- **Why**: Exfiltration prevention; card data cannot leave perimeter.

### 2.3 East-West

- **Pub/Sub** → **Dataflow** → **Vertex** (fraud): Private VPC; no public IPs
- **GKE** (authorization service): Network Policy; pod-to-pod restricted
- **VPC SC**: Block copy of cardholder data to projects outside perimeter
- **Why**: PCI segmentation; authorization path isolated from analytics.

### 2.4 On-Prem to GCP

- **Dedicated Interconnect** for core banking / issuer systems (10/100 Gbps)
- **HA VPN** for backup
- **Why**: Low latency for authorization; BGP failover.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Cloud Build | Block if PAN, CVV in repo |
| **Container scan** | Artifact Registry | Block critical CVEs |
| **Image signing** | KMS + attestation | Binary Auth enforce |
| **PCI scan** | Custom | Block if PAN in image |
| **IAC** | Terraform + Policy | Block public storage, missing encryption |

**Pipeline**: Build → Scan → Sign → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Authorization API**: GKE Ingress canary-weight 5% → 25% → 100%
- **Fraud model**: Vertex endpoint 5% traffic to new model
- **Settlement batch**: New job; compare output with stable before cutover
- **Why**: Limit impact of bad auth logic or model; financial risk mitigation.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Pub/Sub** | Transaction event bus | Decouple; scale; ordering |
| **Dataflow** | Stream processing | Real-time fraud; settlement batch |
| **Vertex AI** | Fraud model | Low-latency inference |
| **Cloud SQL / Spanner** | Transaction store | Strong consistency; PCI |
| **BigQuery** | Analytics (tokenized) | Reporting; no PAN |
| **VPC SC** | Perimeter | Cardholder data exfiltration prevention |
| **CMEK** | Encryption | PCI; key control |
| **DLP** | PAN detection | Block accidental exposure |
