# GCP Photo App Solution (1–10 MB, Auth, Inference, Analytics)

End-to-end design for photo app: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. 1–10 MB rapid ingest, auth, storage, categorization, Vertex Vision inference, browsing, sharing, BigQuery analytics.

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
│   └── prj-photo-dev
├── QA
│   └── prj-photo-qa
└── Prod
    └── prj-photo-prod
```

**Why**: Separate projects per environment; Dev/QA use smaller quotas; Prod has VPC SC for exfiltration prevention.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Cloud Run** | Min 0, max 10 | Min 1, max 50 | Min 5, max 500 |
| **Storage** | 1 TB | 10 TB | Unlimited |
| **Vertex Vision** | Dev quota | QA quota | Prod quota |
| **VPC SC** | No | Optional | Yes |
| **Binary Auth** | Audit | Enforce | Enforce |

---

## 2. Network Design

### 2.1 Ingress

- **Global HTTP(S) LB** → **Cloud Armor** (rate limit, geo) → **Cloud Run** / **GKE**
- **Signed URL** for direct client → GCS upload (no proxy)
- **IAP** for admin dashboards
- **Why**: WAF protects APIs; signed URL avoids bandwidth on app tier.

### 2.2 Egress

- **Cloud NAT** for Cloud Run egress (e.g., external APIs)
- **PSC** for Vertex, BigQuery, Storage (no public API calls)
- **Firewall**: Allow Google APIs; restrict third-party to approved IPs
- **Why**: Centralized egress; no exfiltration to unknown destinations.

### 2.3 East-West

- **Cloud Run** → **Firestore** → **Vertex**: Private VPC; Serverless VPC Access connector
- **GKE** (if used): Network Policy; Workload Identity
- **VPC SC**: Block copy to projects outside perimeter
- **Why**: Micro-segmentation; user data cannot leak across services.

### 2.4 On-Prem to GCP

- **HA VPN** or **Interconnect** for admin/backup from DC
- **Why**: Optional; primary users are internet. On-prem for enterprise sync if needed.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Cloud Build | Block if keys in repo |
| **Container scan** | Artifact Registry | Block critical CVEs |
| **Image signing** | KMS | Binary Auth in prod |
| **Firebase** | Auth config | No prod credentials in dev |

**Pipeline**: Build → Scan → Sign → Deploy Dev → (approval) → QA → (approval) → Canary Prod → Full Prod

---

## 4. Canary

- **Cloud Run**: New revision; 10% traffic via `--no-traffic` then gradual increase
- **GKE**: Ingress canary-weight 10% → 50% → 100%
- **Why**: Limit impact of bad release on photo upload/processing.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Cloud Run** | Upload API | Scale to zero; stateless |
| **Cloud Storage** | Photo storage | 1–10 MB; resumable upload |
| **Eventarc** | GCS → Pub/Sub | Trigger processing on upload |
| **Vertex Vision** | Labels, objects | Categorization |
| **Firestore** | Metadata, user data | Low latency; real-time |
| **BigQuery** | Analytics DW | Management reports |
| **Firebase Auth** | User auth | Social, email, MFA |
| **VPC SC** | Perimeter | Exfiltration prevention |
