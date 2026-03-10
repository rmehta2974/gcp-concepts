# AWS Investment & Risk Modeling Solution (Goldman Sachs)

End-to-end design for investment division and risk modeling: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Market data ingest, risk models (VaR, stress testing), portfolio analytics, trading signals, compliance.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Account Structure

```
Organization
├── inv-risk-dev-account
├── inv-risk-qa-account
└── inv-risk-prod-account
```

**Why**: Investment and risk systems handle sensitive market data and models. Separate accounts enforce SCPs; Prod has Macie and SCP. Dev/QA use anonymized or synthetic market data.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Market data** | Delayed / synthetic | Near real-time (test) | Real-time |
| **SageMaker / Redshift** | Small | Medium | Large; reserved |
| **Macie** | Scan | Alert | Block + alert |
| **SCP** | Relaxed | Stricter | Full |
| **KMS** | Default | CMK | CMK + HSM |
| **CloudTrail** | 30 days | 90 days | 7 years (regulatory) |

---

## 2. Network Design

### 2.1 Ingress

- **CloudFront** → **WAF** for analytics/trading APIs
- **PrivateLink** for market data feeds (Bloomberg, Reuters)
- **VPC Endpoint** for internal APIs
- **Why**: Market data and positions must not traverse public internet. API protected by WAF.

### 2.2 Egress

- **NAT Gateway** for outbound to market data providers (approved IPs only)
- **VPC Endpoints** for SageMaker, Redshift, S3
- **Security Group**: Deny unknown egress; allow only approved vendor IPs
- **Why**: Exfiltration prevention; models and positions cannot leave perimeter.

### 2.3 East-West

- **Kinesis** → **Lambda** → **SageMaker** (risk models): Private subnets
- **Redshift** (portfolio analytics): Private; no public access
- **ECS/EKS** (trading signals): Network Policy; Security Groups
- **Macie**: Monitor S3 for model/position exposure
- **Why**: Micro-segmentation; risk models isolated from trading; audit trail.

### 2.4 On-Prem to AWS

- **Direct Connect** for core trading systems, market data (10/100 Gbps)
- **Site-to-Site VPN** for backup
- **Why**: Low latency for real-time risk; BGP failover.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | CodeBuild | Block if keys in repo |
| **ECR scan** | Image scanning | Block critical CVEs |
| **IAM** | OIDC | No long-lived keys |
| **Model validation** | SageMaker Pipeline | Block if backtest fails |
| **SCP** | Block public S3 | Enforce at org level |

**Pipeline**: Build → Scan → Train → Evaluate → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Risk model**: SageMaker endpoint 5% traffic to new variant; compare VaR vs stable
- **Trading signals API**: ALB target group 5% → 25% → 100%
- **Glue** (risk calc): New job; run in parallel; compare output
- **Why**: Limit impact of bad model; financial risk mitigation.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Kinesis** | Market data stream | Decouple; scale; ordering |
| **Lambda / Glue** | Risk calc; VaR; stress | Horizontal scale |
| **SageMaker** | Risk models; ML | VaR; stress; signals |
| **Redshift** | Portfolio analytics | Historical; reporting |
| **DynamoDB / Aurora** | Position store | Strong consistency |
| **Macie** | Sensitive data | Model/position exposure |
| **KMS** | Encryption | Regulatory; key control |
| **S3** | Model artifacts | Versioned; lineage |
| **SageMaker Pipelines** | MLOps | Reproducible; audit |
