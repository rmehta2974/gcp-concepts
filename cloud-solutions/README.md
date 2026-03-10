# Cloud Solutions: End-to-End Designs

Standalone, cloud-native designs for GCP, AWS, and Azure. Each document is self-contained with no cross-cloud references. Every solution includes: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem, and component rationale.

---

## Foundational Infrastructure

| Cloud | Document | Description |
|-------|----------|-------------|
| **GCP** | [gcp-end-to-end-solution.md](./gcp-end-to-end-solution.md) | Base infra: prod/dev/QA, pipeline, canary, traffic control, on-prem |
| **AWS** | [aws-end-to-end-solution.md](./aws-end-to-end-solution.md) | Base infra: prod/dev/QA, pipeline, canary, traffic control, on-prem |
| **Azure** | [azure-end-to-end-solution.md](./azure-end-to-end-solution.md) | Base infra: prod/dev/QA, pipeline, canary, traffic control, on-prem |

---

## Healthcare Imaging (5 PB, Vertex/Gemini, Patient+Image+Disease)

| Cloud | Document |
|-------|----------|
| **GCP** | [healthcare-imaging/gcp-healthcare-imaging-solution.md](./healthcare-imaging/gcp-healthcare-imaging-solution.md) |
| **AWS** | [healthcare-imaging/aws-healthcare-imaging-solution.md](./healthcare-imaging/aws-healthcare-imaging-solution.md) |
| **Azure** | [healthcare-imaging/azure-healthcare-imaging-solution.md](./healthcare-imaging/azure-healthcare-imaging-solution.md) |

---

## Photo App (1–10 MB, Auth, Inference, Analytics)

| Cloud | Document |
|-------|----------|
| **GCP** | [photo-app/gcp-photo-app-solution.md](./photo-app/gcp-photo-app-solution.md) |
| **AWS** | [photo-app/aws-photo-app-solution.md](./photo-app/aws-photo-app-solution.md) |
| **Azure** | [photo-app/azure-photo-app-solution.md](./photo-app/azure-photo-app-solution.md) |

---

## Real-Time & ETL (Pub/Sub, Dataflow, BigQuery)

| Cloud | Document |
|-------|----------|
| **GCP** | [realtime-etl/gcp-realtime-etl-solution.md](./realtime-etl/gcp-realtime-etl-solution.md) |
| **AWS** | [realtime-etl/aws-realtime-etl-solution.md](./realtime-etl/aws-realtime-etl-solution.md) |
| **Azure** | [realtime-etl/azure-realtime-etl-solution.md](./realtime-etl/azure-realtime-etl-solution.md) |

---

## AI/ML (Vertex, SageMaker, Azure ML)

| Cloud | Document |
|-------|----------|
| **GCP** | [ai-ml/gcp-ai-ml-solution.md](./ai-ml/gcp-ai-ml-solution.md) |
| **AWS** | [ai-ml/aws-ai-ml-solution.md](./ai-ml/aws-ai-ml-solution.md) |
| **Azure** | [ai-ml/azure-ai-ml-solution.md](./ai-ml/azure-ai-ml-solution.md) |

---

## Credit Card Transactions (Goldman Sachs)

| Cloud | Document |
|-------|----------|
| **GCP** | [credit-card-transactions/gcp-credit-card-solution.md](./credit-card-transactions/gcp-credit-card-solution.md) |
| **AWS** | [credit-card-transactions/aws-credit-card-solution.md](./credit-card-transactions/aws-credit-card-solution.md) |
| **Azure** | [credit-card-transactions/azure-credit-card-solution.md](./credit-card-transactions/azure-credit-card-solution.md) |

---

## Investment & Risk Modeling (Goldman Sachs)

| Cloud | Document |
|-------|----------|
| **GCP** | [investment-risk-modeling/gcp-investment-risk-solution.md](./investment-risk-modeling/gcp-investment-risk-solution.md) |
| **AWS** | [investment-risk-modeling/aws-investment-risk-solution.md](./investment-risk-modeling/aws-investment-risk-solution.md) |
| **Azure** | [investment-risk-modeling/azure-investment-risk-solution.md](./investment-risk-modeling/azure-investment-risk-solution.md) |

---

## Migration (Data, Application, Server, VMware)

| Cloud | Document |
|-------|----------|
| **GCP** | [migration/gcp-migration-solution.md](./migration/gcp-migration-solution.md) |
| **AWS** | [migration/aws-migration-solution.md](./migration/aws-migration-solution.md) |
| **Azure** | [migration/azure-migration-solution.md](./migration/azure-migration-solution.md) |

---

## What Each Solution Covers

- **Environments**: Prod, Dev, QA (structure, differences, rationale)
- **Ingress**: WAF, load balancer, identity
- **Egress**: NAT, firewall, private endpoints
- **East-west**: Micro-segmentation, network policy
- **On-prem**: VPN, dedicated link, routing
- **Pipeline security**: Scanning, signing, no long-lived keys
- **Canary**: Traffic splitting, rollback
- **Components**: What and why for each choice
