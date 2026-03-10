# AWS Photo App Solution (1–10 MB, Auth, Inference, Analytics)

End-to-end design for photo app: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. 1–10 MB rapid ingest, auth, storage, categorization, Rekognition inference, browsing, sharing, Redshift analytics.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Account Structure

```
Organization
├── photo-dev-account
├── photo-qa-account
└── photo-prod-account
```

**Why**: Separate accounts per environment; SCPs restrict public S3, cross-account copy. Dev/QA use smaller quotas.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Lambda / ECS** | Small concurrency | Medium | High, multi-AZ |
| **S3** | 1 TB | 10 TB | Unlimited |
| **Rekognition** | Dev quota | QA quota | Prod quota |
| **Macie** | Scan | Alert | Block + alert |
| **SCP** | Relaxed | Stricter | Full |

---

## 2. Network Design

### 2.1 Ingress

- **CloudFront** → **WAF** (rate limit, geo) → **ALB** → **ECS** / **Lambda**
- **Presigned URL** for direct client → S3 upload (no proxy)
- **Cognito** for auth
- **Why**: WAF protects APIs; presigned URL avoids bandwidth on app tier.

### 2.2 Egress

- **NAT Gateway** for Lambda/ECS egress
- **VPC Endpoints** for S3, Rekognition, DynamoDB (no NAT for AWS APIs)
- **Security Group**: Allow AWS endpoints; restrict third-party to approved IPs
- **Why**: Centralized egress; no exfiltration to unknown destinations.

### 2.3 East-West

- **Lambda** → **DynamoDB** → **Rekognition**: Private subnets; VPC Endpoints
- **EKS** (if used): Network Policy; Security Groups
- **Macie**: Monitor S3 for sensitive data
- **Why**: Micro-segmentation; user data cannot leak.

### 2.4 On-Prem to AWS

- **Site-to-Site VPN** or **Direct Connect** for admin/backup from DC
- **Why**: Optional; primary users are internet.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | CodeBuild | Block if keys in repo |
| **ECR scan** | Image scanning | Block critical CVEs |
| **IAM** | OIDC | No long-lived keys |
| **Cognito** | Auth config | No prod credentials in dev |

**Pipeline**: Build → Scan → Push ECR → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **ECS**: CodeDeploy blue/green; 10% → 50% → 100%
- **Lambda**: Weighted alias; 10% to new version
- **ALB**: Target group 90/10 split
- **Why**: Limit impact of bad release.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Lambda / ECS** | Upload API | Scale; stateless |
| **S3** | Photo storage | 1–10 MB; presigned upload |
| **EventBridge / S3 Events** | S3 → Lambda | Trigger processing |
| **Rekognition** | Labels, objects | Categorization |
| **DynamoDB** | Metadata, user data | Low latency |
| **Redshift** | Analytics DW | Management reports |
| **Cognito** | User auth | Social, email, MFA |
| **Macie** | Sensitive data | Exfiltration detection |
