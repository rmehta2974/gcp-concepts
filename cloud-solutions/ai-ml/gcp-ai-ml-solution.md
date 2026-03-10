# GCP AI/ML Solution (Vertex, Gemini, Training, Inference)

End-to-end design for AI/ML workloads: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Vertex AI training, inference, model registry, Gemini, MLOps.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Folder and Project Structure

```
Organization
├── Platform
│   ├── prj-logging
│   └── prj-network-host
├── Dev
│   └── prj-ml-dev
├── QA
│   └── prj-ml-qa
└── Prod
    └── prj-ml-prod
```

**Why**: Separate projects isolate model artifacts and training costs. Dev/QA use smaller GPU quotas; Prod has VPC SC for model/data perimeter.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Vertex Training** | 1–2 GPUs | 4–8 GPUs | 16+ GPUs |
| **Vertex Endpoint** | Dev model | QA model | Prod model |
| **Gemini** | Dev quota | QA quota | Prod quota |
| **VPC SC** | No | Optional | Yes |
| **Binary Auth** | Audit | Enforce | Enforce |
| **Model Registry** | Dev registry | QA registry | Prod registry |

---

## 2. Network Design

### 2.1 Ingress

- **Global HTTP(S) LB** → **Cloud Armor** for inference API
- **IAP** for Vertex AI Notebooks, ML pipelines UI
- **Private endpoint** for Vertex API (PSC)
- **Why**: Inference API protected by WAF; notebooks via IAP only.

### 2.2 Egress

- **PSC** for Vertex, BigQuery, Storage (no public API calls)
- **Cloud NAT** for training jobs (e.g., pip install) if needed
- **Firewall**: Allow Google APIs; deny unknown egress
- **Why**: Model and data stay within Google network; no exfiltration.

### 2.3 East-West

- **Vertex Training** → **Artifact Registry** → **Vertex Endpoint**: Private VPC
- **Cloud Run** (inference) → **Vertex**: Private; Serverless VPC Access
- **VPC SC**: Block copy of models/data to projects outside perimeter
- **Why**: Micro-segmentation; models cannot leak.

### 2.4 On-Prem to GCP

- **Interconnect** for large dataset transfer from DC
- **Vertex AI** datasets from on-prem via Transfer Service
- **Why**: Hybrid training data; batch upload.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Cloud Build | Block if keys in repo |
| **Container scan** | Artifact Registry | Block critical CVEs |
| **Model validation** | Vertex Pipeline | Block if metrics below threshold |
| **Data lineage** | Vertex MLMD | Audit trail |

**Pipeline**: Train → Evaluate → Register → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Vertex Endpoint**: New model deployment; route 10% traffic via config
- **Cloud Run** (inference proxy): Revision traffic split 10% → 50% → 100%
- **A/B test**: Vertex AI supports traffic split by model version
- **Why**: Limit impact of bad model; monitor latency and errors.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Vertex AI** | Training, inference, registry | Managed; GPU; MLOps |
| **Vertex Pipelines** | MLOps orchestration | Reproducible; lineage |
| **Gemini** | LLM | Multimodal; API |
| **Artifact Registry** | Container images | Training images; signed |
| **BigQuery** | Training data | Feature store; analytics |
| **PSC** | Private APIs | No public egress |
| **VPC SC** | Perimeter | Model/data exfiltration prevention |
