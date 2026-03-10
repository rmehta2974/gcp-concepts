# Azure Real-Time & ETL Solution

End-to-end design for real-time event streaming and ETL: prod/dev/QA, pipeline security, canary, ingress/egress/east-west, on-prem. Event Hubs, Data Factory, Synapse, Functions.

---

## 1. Environment Strategy (Prod, Dev, QA)

### 1.1 Subscription Structure

```
Management Group
├── realtime-dev-subscription
├── realtime-qa-subscription
└── realtime-prod-subscription
```

**Why**: Separate subscriptions isolate event throughput and Synapse costs. Dev/QA use smaller Event Hubs; Prod has Purview for data perimeter.

### 1.2 Environment Differences

| Aspect | Dev | QA | Prod |
|--------|-----|-----|------|
| **Event Hubs** | 1 TU | 10 TU | 20+ TU |
| **Data Factory / Synapse** | Small | Medium | Large, auto-scale |
| **Synapse** | Serverless | Dedicated | Dedicated, large |
| **Purview** | Scan | Classify | Enforce |
| **Defender** | Foundational | Standard | Standard |

---

## 2. Network Design

### 2.1 Ingress

- **API Management** → **WAF** for event ingestion APIs
- **Event Hubs** via Private Endpoint (private)
- **Event Grid** for SaaS events
- **Why**: Centralized ingress; rate limit protects from event floods.

### 2.2 Egress

- **NAT Gateway** for Data Factory/Synapse egress
- **Private Endpoints** for Event Hubs, Synapse, Blob (no NAT for Azure APIs)
- **NSG**: Allow Azure endpoints; deny unknown egress
- **Why**: All data stays within Azure backbone; no exfiltration.

### 2.3 East-West

- **Event Hubs** → **Functions** → **Synapse**: Private subnets; Private Endpoints
- **Data Factory** → **Blob** → **Synapse**: Private; no public access
- **Purview**: Classify sensitive data
- **Why**: Micro-segmentation; event data cannot leak.

### 2.4 On-Prem to Azure

- **ExpressRoute** for high-volume event ingest from DC
- **Event Hubs** over Private Link
- **Data Factory** for batch ETL from on-prem DB
- **Why**: Hybrid event sources; batch backfill.

---

## 3. Security in Pipeline

| Gate | Component | Action |
|------|-----------|--------|
| **Secret scan** | Azure DevOps | Block if keys in repo |
| **ACR scan** | Image scanning | Block critical CVEs |
| **Data Factory** | Managed Identity | Least privilege |
| **Synapse** | RBAC + encryption | Enforce at workspace |

**Pipeline**: Build → Scan → Deploy pipeline → Deploy Dev → (approval) → QA → (approval) → Prod

---

## 4. Canary

- **Data Factory**: New pipeline; run in parallel; compare output
- **Event Hubs**: New consumer group; 10% via filter
- **Functions**: Slot swap or deployment slot 10%
- **Why**: Limit impact of bad pipeline or consumer logic.

---

## 5. Component Rationale

| Component | What | Why |
|-----------|------|-----|
| **Event Hubs** | Event stream | Decouple; scale; Kafka compat |
| **Data Factory / Synapse** | ETL | Spark; horizontal scale |
| **Synapse** | Warehouse | Real-time load; analytics |
| **Functions** | Serverless consumer | Event Hubs trigger; scale |
| **Event Grid** | Event routing | SaaS → Azure |
| **Private Endpoints** | Private Azure APIs | No internet egress |
| **Purview** | Data governance | Exfiltration prevention |
