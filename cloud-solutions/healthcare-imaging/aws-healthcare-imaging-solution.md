# AWS Healthcare Imaging Solution (5 PB, SageMaker, Bedrock)

End-to-end design for healthcare imaging: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. 5 PB image ETL, metadata cataloging, SageMaker inference, Bedrock, LLM accuracy monitoring, patient+image+disease training.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Account and OU Structure

```
Organization
├── Workloads (OU)
│   ├── imaging-dev-account
│   ├── imaging-qa-account
│   └── imaging-prod-account
```

**Why**: Separate accounts isolate PHI; SCPs restrict cross-account data movement. Dev/QA use sample data; Prod holds full 5 PB.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Storage** | 100 TB (S3) | 500 TB | 5 PB |
| **Glue/EMR** | Small cluster | Medium | Large, auto-scale |
| **SageMaker endpoint** | Dev | QA | Prod |
| **Macie** | Scan only | Alert | Block + alert |
| **SCP** | Relaxed | Stricter | Full (no public S3) |

---

## 2. Network Design

### 2.1 Ingress

- **CloudFront** → **WAF** for admin UIs
- **PACS ingest**: On-prem → Direct Connect → Private VPC endpoint (no public)
- **Why**: PHI must not traverse public internet.

### 2.2 Egress

- **NAT Gateway** for Glue/EMR workers
- **VPC Endpoints** for S3, SageMaker, Bedrock (no NAT for AWS APIs)
- **Security Group**: Deny unknown egress; allow only AWS endpoints and approved IPs
- **Why**: Exfiltration prevention; AWS traffic stays on backbone.

### 2.3 East-West

- **Glue** → **SageMaker** → **Redshift**: Private subnets; VPC endpoints
- **EKS** (if used): Network Policy; Security Groups
- **Macie**: Monitor S3 for sensitive data exposure
- **Why**: Micro-segmentation; PHI cannot leak.

### 2.4 On-Prem to AWS

- **Direct Connect** for PACS bulk transfer (10/100 Gbps)
- **Site-to-Site VPN** for backup
- **DataSync** / **Transfer Family** for batch migration
- **Why**: High throughput; BGP failover.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | CodeBuild | Block if PHI in repo |
| **ECR scan** | Image scanning | Block critical CVEs |
| **IAM** | No long-lived keys | OIDC for GitHub/GitLab |
| **SCP** | Block public S3 | Enforce at org level |

**Pipeline**: Build → Scan → Push ECR → Deploy Dev → (approval) → QA → (approval) → Prod

---

## 4. Canary

- **Glue**: New job version; compare output with stable
- **SageMaker**: New endpoint; route 10% via Lambda/ALB weight
- **EKS**: ALB target group 90/10 split
- **Why**: Limit impact of bad model or pipeline.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Glue / EMR** | ETL | Scale for 5 PB; Spark |
| **SageMaker** | Inference | Managed; GPU; model monitor |
| **Bedrock** | LLM | Report coding; multimodal |
| **Lake Formation** | Catalog | PHI tagging; access control |
| **Macie** | Sensitive data | Detect PHI exposure |
| **VPC Endpoints** | Private AWS APIs | No internet egress |
| **Direct Connect** | On-prem | Throughput for 5 PB |
