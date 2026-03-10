# Azure AI/ML Solution (Azure ML, OpenAI, Training, Inference)

End-to-end design for AI/ML workloads: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Azure ML training, inference, model registry, OpenAI, MLOps.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Subscription Structure

```
Management Group
├── ml-dev-subscription
├── ml-qa-subscription
└── ml-prod-subscription
```

**Why**: Separate subscriptions isolate model artifacts and training costs. Dev/QA use smaller GPU quotas; Prod has Purview for model/data perimeter.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Azure ML Compute** | 1–2 GPUs | 4–8 GPUs | 16+ GPUs |
| **Azure ML Endpoint** | Dev | QA | Prod |
| **OpenAI** | Dev quota | QA quota | Prod quota |
| **Purview** | Scan | Classify | Enforce |
| **Defender** | Foundational | Standard | Standard |

---

## 2. Network Design

### 2.1 Ingress

- **Front Door** → **WAF** for inference API
- **Azure ML Studio** via Private Endpoint; no public access
- **Private Endpoint** for Azure ML API
- **Why**: Inference API protected by WAF; Studio via private path only.

### 2.2 Egress

- **Private Endpoints** for Azure ML, Blob, Synapse (no NAT for Azure APIs)
- **NAT Gateway** for training jobs (e.g., pip install) if needed
- **NSG**: Allow Azure endpoints; deny unknown egress
- **Why**: Model and data stay within Azure backbone; no exfiltration.

### 2.3 East-West

- **Azure ML Training** → **ACR** → **Azure ML Endpoint**: Private subnets
- **Container Apps** (inference proxy) → **Azure ML**: Private Endpoint
- **Purview**: Classify model artifacts
- **Why**: Micro-segmentation; models cannot leak.

### 2.4 On-Prem to Azure

- **ExpressRoute** for large dataset transfer from DC
- **Azure ML** datasets from on-prem via Data Factory
- **Why**: Hybrid training data; batch upload.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Azure DevOps | Block if keys in repo |
| **ACR scan** | Image scanning | Block critical CVEs |
| **Model validation** | Azure ML Pipeline | Block if metrics below threshold |
| **Azure ML** | Managed Identity | Least privilege |

**Pipeline**: Train → Evaluate → Register → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Azure ML Endpoint**: New deployment; route 10% traffic via Traffic Manager
- **Container Apps**: Revision traffic split 10% → 50% → 100%
- **AKS**: Application Gateway backend pool 90/10
- **Why**: Limit impact of bad model; monitor latency and errors.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Azure ML** | Training, inference, registry | Managed; GPU; MLOps |
| **Azure ML Pipelines** | MLOps orchestration | Reproducible; lineage |
| **OpenAI** | LLM | Multimodal; API |
| **ACR** | Container images | Training images; scanned |
| **Blob / Synapse** | Training data | Feature store; analytics |
| **Private Endpoints** | Private Azure APIs | No internet egress |
| **Purview** | Data governance | Model/data exfiltration prevention |
