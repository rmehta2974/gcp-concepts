# GCP Healthcare Imaging Solution (5 PB, Vertex, Gemini)

End-to-end design for healthcare imaging: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. 5 PB image ETL, metadata cataloging, Vertex inference, Gemini, LLM accuracy monitoring, patient+image+disease training.

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
│   └── prj-imaging-dev
├── QA
│   └── prj-imaging-qa
└── Prod
    └── prj-imaging-prod
```

**Why**: Imaging workloads are PHI-heavy; separate projects enforce isolation and allow environment-specific VPC SC perimeters. Dev/QA use smaller datasets; Prod holds full 5 PB.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Storage** | 100 TB sample | 500 TB sample | 5 PB full |
| **Dataflow workers** | 5–10 | 20–50 | 100+ |
| **Vertex endpoint** | Dev model | QA model | Prod model |
| **VPC SC** | No | Optional | Yes (PHI perimeter) |
| **DLP** | Audit | Enforce | Enforce |
| **Binary Auth** | Audit | Enforce | Enforce |

---

## 2. Network Design

### 2.1 Ingress

- **Global HTTP(S) LB** → **Cloud Armor** → **IAP** (for admin UIs)
- **PACS/DICOM ingest**: On-prem → Interconnect → Private endpoint (no public ingress)
- **Why**: PHI must not traverse public internet. Admin access via IAP only.

### 2.2 Egress

- **Cloud NAT** for Dataflow workers; **Private Google Access** for Vertex/Gemini
- **PSC** for `*.googleapis.com` (Vertex, BigQuery, Storage)
- **Firewall**: Deny unknown egress; allow only Google APIs and approved SaaS
- **Why**: Exfiltration prevention; all API traffic stays private.

### 2.3 East-West

- **Dataflow** → **Vertex** → **BigQuery**: Private VPC; no public IPs
- **GKE** (if used): Network Policy; pod-to-pod restricted
- **VPC SC**: Block copy to projects outside perimeter
- **Why**: Micro-segmentation; PHI cannot leak across services.

### 2.4 On-Prem to GCP

- **Dedicated Interconnect** for PACS/DICOM bulk transfer (10/100 Gbps)
- **HA VPN** for backup and metadata sync
- **Cloud Storage Transfer Service** for batch migration
- **Why**: High throughput for 5 PB; BGP failover.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Cloud Build | Block if PHI/PII in repo |
| **Container scan** | Artifact Registry | Block critical CVEs |
| **Image signing** | KMS + attestation | Binary Auth enforce in prod |
| **IAC** | Terraform + Policy | Block public buckets, missing encryption |

**Pipeline**: Build → Scan → Sign → Deploy Dev → (approval) → Deploy QA → (approval) → Deploy Prod

---

## 4. Canary

- **Dataflow**: New pipeline version as separate job; compare output with stable
- **Vertex endpoint**: New model as separate endpoint; route 10% traffic via config
- **GKE** (inference API): Ingress canary-weight 10% → 50% → 100%
- **Why**: Limit impact of bad model or pipeline changes.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Dataflow** | Stream/batch ETL | Horizontal scale for 5 PB; Beam portability |
| **Vertex AI** | Inference | Managed; GPU; model monitoring |
| **Gemini** | LLM | Report coding; multimodal |
| **Dataplex** | Catalog | PHI tagging; lineage |
| **VPC SC** | Perimeter | Block PHI exfiltration |
| **DLP** | De-identify | Compliance; redaction |
| **PSC** | Private APIs | No public egress for Google APIs |
