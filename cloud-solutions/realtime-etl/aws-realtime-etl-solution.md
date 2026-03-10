# AWS Real-Time & ETL Solution

End-to-end design for real-time event streaming and ETL: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Kinesis, Glue, Lambda, Redshift.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Account Structure

```
Organization
├── realtime-dev-account
├── realtime-qa-account
└── realtime-prod-account
```

**Why**: Separate accounts isolate event throughput and Redshift costs. Dev/QA use smaller Kinesis shards; Prod has SCP for data perimeter.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Kinesis** | 2 shards | 10 shards | 50+ shards |
| **Glue / EMR** | Small cluster | Medium | Large, auto-scale |
| **Redshift** | Single node | Multi-node | Multi-node, RA3 |
| **Macie** | Scan | Alert | Block + alert |
| **SCP** | Relaxed | Stricter | Full |

---

## 2. Network Design

### 2.1 Ingress

- **API Gateway** → **WAF** for event ingestion APIs
- **Kinesis Data Streams** via VPC Endpoint (private)
- **EventBridge** for SaaS events
- **Why**: Centralized ingress; rate limit protects from event floods.

### 2.2 Egress

- **NAT Gateway** for Glue/Lambda egress
- **VPC Endpoints** for Kinesis, Redshift, S3 (no NAT for AWS APIs)
- **Security Group**: Allow AWS endpoints; deny unknown egress
- **Why**: All data stays within AWS network; no exfiltration.

### 2.3 East-West

- **Kinesis** → **Lambda** → **Redshift**: Private subnets; VPC Endpoints
- **Glue** → **S3** → **Redshift**: Private; no public access
- **Macie**: Monitor S3 for sensitive data
- **Why**: Micro-segmentation; event data cannot leak.

### 2.4 On-Prem to AWS

- **Direct Connect** for high-volume event ingest from DC
- **Kinesis Agent** or **Firehose** from on-prem
- **DMS** for batch ETL from on-prem DB
- **Why**: Hybrid event sources; batch backfill.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | CodeBuild | Block if keys in repo |
| **ECR scan** | Image scanning | Block critical CVEs |
| **Glue job** | IAM role | Least privilege |
| **Redshift** | IAM + encryption | Enforce at cluster |

**Pipeline**: Build → Scan → Deploy Glue job → Deploy Dev → (approval) → QA → (approval) → Prod

---

## 4. Canary

- **Glue**: New job version; run in parallel; compare output
- **Kinesis**: New consumer (Lambda); 10% via enhanced fan-out or filter
- **Lambda**: Weighted alias 10% to new version
- **Why**: Limit impact of bad pipeline or consumer logic.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Kinesis** | Event stream | Decouple; scale; ordering |
| **Glue / EMR** | ETL | Spark; horizontal scale |
| **Redshift** | Warehouse | Real-time load; analytics |
| **Lambda** | Serverless consumer | Kinesis trigger; scale |
| **EventBridge** | Event routing | SaaS → AWS |
| **VPC Endpoints** | Private AWS APIs | No internet egress |
| **Macie** | Sensitive data | Exfiltration detection |
