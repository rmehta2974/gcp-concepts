# GCP Real-Time & ETL Solution

End-to-end design for real-time event streaming and ETL: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Pub/Sub, Dataflow, BigQuery, Cloud Functions.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Folder and Project Structure

```
Organization
├── Platform
│   ├── prj-logging
│   └── prj-network-host
├── Dev
│   └── prj-realtime-dev
├── QA
│   └── prj-realtime-qa
└── Prod
    └── prj-realtime-prod
```

**Why**: Separate projects isolate event throughput and BigQuery costs. Dev/QA use smaller Pub/Sub quotas; Prod has VPC SC for data perimeter.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Pub/Sub** | 10K msg/s | 100K msg/s | 1M+ msg/s |
| **Dataflow** | 5 workers | 20 workers | 100+ workers |
| **BigQuery** | On-demand, small | Reserved slots | Reserved slots, large |
| **VPC SC** | No | Optional | Yes |
| **Binary Auth** | Audit | Enforce | Enforce |

---

## 2. Network Design

### 2.1 Ingress

- **Global HTTP(S) LB** → **Cloud Armor** for event ingestion APIs
- **Pub/Sub push** from external sources (webhook) → IAP or API key
- **Eventarc** for direct event ingestion
- **Why**: Centralized ingress; rate limit protects from event floods.

### 2.2 Egress

- **Cloud NAT** for Dataflow workers; **PSC** for BigQuery, Pub/Sub, Storage
- **Firewall**: Allow Google APIs; deny unknown egress
- **Why**: All data stays within Google network; no exfiltration.

### 2.3 East-West

- **Pub/Sub** → **Dataflow** → **BigQuery**: Private VPC; no public IPs
- **Cloud Functions** (push): Invoked via private VPC connector
- **VPC SC**: Block copy to projects outside perimeter
- **Why**: Micro-segmentation; event data cannot leak.

### 2.4 On-Prem to GCP

- **Interconnect** for high-volume event ingest from DC
- **Pub/Sub** over private path (PSC)
- **Transfer Service** for batch ETL from on-prem DB
- **Why**: Hybrid event sources; batch backfill.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Cloud Build | Block if keys in repo |
| **Container scan** | Artifact Registry | Block critical CVEs |
| **Dataflow template** | Validation | Block invalid config |
| **BigQuery** | IAM | Least privilege for pipeline SA |

**Pipeline**: Build → Scan → Deploy Dataflow template → Deploy Dev → (approval) → QA → (approval) → Prod

---

## 4. Canary

- **Dataflow**: New job with new template; run in parallel; compare output
- **Pub/Sub**: New subscription for canary consumer; 10% message filter
- **Cloud Functions**: New revision; 10% traffic via Cloud Run traffic split
- **Why**: Limit impact of bad pipeline or consumer logic.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Pub/Sub** | Event bus | Decouple; scale; ordering |
| **Dataflow** | Stream/batch ETL | Beam; horizontal scale |
| **BigQuery** | Warehouse | Real-time insert; analytics |
| **Cloud Functions** | Serverless consumer | Push subscription; scale to zero |
| **Eventarc** | Event routing | Source → destination |
| **PSC** | Private APIs | No public egress |
| **VPC SC** | Perimeter | Exfiltration prevention |
