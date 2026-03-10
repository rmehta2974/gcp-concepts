# AWS AI/ML Solution (SageMaker, Bedrock, Training, Inference)

End-to-end design for AI/ML workloads: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. SageMaker training, inference, model registry, Bedrock, MLOps.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Account Structure

```
Organization
├── ml-dev-account
├── ml-qa-account
└── ml-prod-account
```

**Why**: Separate accounts isolate model artifacts and training costs. Dev/QA use smaller GPU quotas; Prod has SCP for model/data perimeter.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **SageMaker Training** | 1–2 GPUs | 4–8 GPUs | 16+ GPUs |
| **SageMaker Endpoint** | Dev | QA | Prod |
| **Bedrock** | Dev quota | QA quota | Prod quota |
| **Macie** | Scan | Alert | Block + alert |
| **SCP** | Relaxed | Stricter | Full |

---

## 2. Network Design

### 2.1 Ingress

- **API Gateway** → **WAF** for inference API
- **SageMaker Studio** via VPC-only; no public access
- **VPC Endpoint** for SageMaker API (private)
- **Why**: Inference API protected by WAF; Studio via private path only.

### 2.2 Egress

- **VPC Endpoints** for SageMaker, S3, Redshift (no NAT for AWS APIs)
- **NAT Gateway** for training jobs (e.g., pip install) if needed
- **Security Group**: Allow AWS endpoints; deny unknown egress
- **Why**: Model and data stay within AWS network; no exfiltration.

### 2.3 East-West

- **SageMaker Training** → **ECR** → **SageMaker Endpoint**: Private subnets
- **Lambda** (inference proxy) → **SageMaker**: VPC Endpoint
- **Macie**: Monitor S3 for model artifacts
- **Why**: Micro-segmentation; models cannot leak.

### 2.4 On-Prem to AWS

- **Direct Connect** for large dataset transfer from DC
- **SageMaker** datasets from on-prem via DataSync
- **Why**: Hybrid training data; batch upload.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | CodeBuild | Block if keys in repo |
| **ECR scan** | Image scanning | Block critical CVEs |
| **Model validation** | SageMaker Pipeline | Block if metrics below threshold |
| **SageMaker** | IAM role | Least privilege |

**Pipeline**: Train → Evaluate → Register → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **SageMaker Endpoint**: New variant; route 10% traffic via config
- **Lambda**: Weighted alias 10% to new inference logic
- **ALB**: Target group 90/10 for inference API
- **Why**: Limit impact of bad model; monitor latency and errors.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **SageMaker** | Training, inference, registry | Managed; GPU; MLOps |
| **SageMaker Pipelines** | MLOps orchestration | Reproducible; lineage |
| **Bedrock** | LLM | Multimodal; API |
| **ECR** | Container images | Training images; scanned |
| **S3** | Training data | Feature store; analytics |
| **VPC Endpoints** | Private AWS APIs | No internet egress |
| **Macie** | Sensitive data | Model/data exfiltration detection |
