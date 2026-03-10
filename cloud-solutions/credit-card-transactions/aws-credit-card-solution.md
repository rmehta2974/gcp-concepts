# AWS Credit Card Transactions Solution (Goldman Sachs)

End-to-end design for credit card transaction processing: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Transaction ingest, authorization, fraud detection, settlement, compliance.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Account Structure

```
Organization
├── cc-dev-account
├── cc-qa-account
└── cc-prod-account
```

**Why**: PCI-DSS requires strict isolation. Separate accounts enforce SCPs; Prod has Macie and SCP blocking public S3. Dev/QA use tokenized or synthetic data only.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Data** | Synthetic / tokenized | Tokenized | Real (encrypted, restricted) |
| **Throughput** | 1K TPS | 10K TPS | 100K+ TPS |
| **Macie** | Scan | Alert | Block + alert |
| **SCP** | Relaxed | Stricter | Full (no public S3, no cross-account) |
| **KMS** | Default | CMK | CMK + HSM |
| **GuardDuty** | Enabled | Enabled | Enabled + S3 protection |

---

## 2. Network Design

### 2.1 Ingress

- **CloudFront** → **WAF** (rate limit, geo, OWASP) for authorization API
- **PrivateLink** for acquirer/issuer connections (no public)
- **VPC Endpoint** for internal APIs
- **Why**: Card data must not traverse public internet. Authorization API protected by WAF.

### 2.2 Egress

- **NAT Gateway** for outbound (e.g., network authorization)
- **VPC Endpoints** for SageMaker (fraud), Redshift, S3
- **Security Group**: Deny unknown egress; allow only approved acquirer/network IPs
- **Why**: Exfiltration prevention; card data cannot leave perimeter.

### 2.3 East-West

- **Kinesis** → **Lambda** → **SageMaker** (fraud): Private subnets; VPC Endpoints
- **ECS/EKS** (authorization): Network Policy; Security Groups
- **Macie**: Monitor S3 for PAN exposure
- **Why**: PCI segmentation; authorization path isolated from analytics.

### 2.4 On-Prem to AWS

- **Direct Connect** for core banking / issuer systems (10/100 Gbps)
- **Site-to-Site VPN** for backup
- **Why**: Low latency for authorization; BGP failover.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | CodeBuild | Block if PAN, CVV in repo |
| **ECR scan** | Image scanning | Block critical CVEs |
| **IAM** | OIDC | No long-lived keys |
| **PCI scan** | Custom | Block if PAN in image |
| **SCP** | Block public S3 | Enforce at org level |

**Pipeline**: Build → Scan → Push ECR → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Authorization API**: ALB target group 5% → 25% → 100%
- **Fraud model**: SageMaker endpoint 5% traffic to new variant
- **Settlement batch**: New Glue job; compare output before cutover
- **Why**: Limit impact of bad auth logic or model; financial risk mitigation.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Kinesis** | Transaction event stream | Decouple; scale; ordering |
| **Lambda / Glue** | Stream processing | Real-time fraud; settlement batch |
| **SageMaker** | Fraud model | Low-latency inference |
| **RDS / Aurora** | Transaction store | Strong consistency; PCI |
| **Redshift** | Analytics (tokenized) | Reporting; no PAN |
| **Macie** | Sensitive data | PAN exposure detection |
| **KMS** | Encryption | PCI; key control |
| **VPC Endpoints** | Private AWS APIs | No internet egress |
