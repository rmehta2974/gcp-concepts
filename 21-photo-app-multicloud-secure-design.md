# Photo App: Multi-Cloud Secure Design (AWS, Azure, GCP)

Full-stack design for 1–10 MB photo ingest, authentication, storage, categorization, inference, browsing, sharing, analytics, and secure zero-trust infrastructure across AWS, Azure, and GCP.

---

## 1. Requirements Summary

| Requirement | Description |
|-------------|-------------|
| **Photo size** | 1–10 MB; rapid ingest |
| **Auth** | User authentication |
| **Frontend** | Web/mobile app |
| **Storage** | Photo storage; categorization |
| **Inference** | AI/ML on photos |
| **Browsing** | User photo gallery |
| **Sharing** | Share photos with others |
| **Analytics** | Data warehouse for management |
| **Security** | Zero trust; exfiltration prevention |
| **Availability** | HA within region; across regions |
| **Observability** | Metrics; logs; troubleshooting |

---

## 2. High-Level Architecture (Cloud-Agnostic)

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        Web[Web App]
        Mobile[Mobile App]
    end
    
    subgraph Edge["Edge"]
        CDN[CDN]
        WAF[WAF]
    end
    
    subgraph Auth["Auth"]
        IdP[Identity Provider]
    end
    
    subgraph API["API Layer"]
        Gateway[API Gateway]
        Services[Microservices]
    end
    
    subgraph Data["Data"]
        Storage[Object Storage]
        DB[Database]
        DW[Data Warehouse]
    end
    
    subgraph AI["AI"]
        Inference[Inference]
    end
    
    Clients --> CDN
    CDN --> WAF
    WAF --> IdP
    IdP --> Gateway
    Gateway --> Services
    Services --> Storage
    Services --> DB
    Services --> Inference
    Services --> DW
```

---

## 3. Per-Cloud Architecture

### 3.1 GCP Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        Web[Web]
        Mobile[Mobile]
    end
    
    subgraph GCP["GCP"]
        subgraph Edge["Edge"]
            LB[Global HTTP/S LB]
            CDN[Cloud CDN]
            IAP[Identity-Aware Proxy]
        end
        
        subgraph Auth["Auth"]
            Firebase[Firebase Auth]
            IAM[IAM]
        end
        
        subgraph API["API"]
            Run[Cloud Run]
            Functions[Cloud Functions]
        end
        
        subgraph Storage["Storage"]
            GCS[Cloud Storage]
            Firestore[Firestore]
        end
        
        subgraph AI["AI"]
            Vertex[Vertex AI Vision]
        end
        
        subgraph DW["Data Warehouse"]
            BQ[BigQuery]
        end
        
        subgraph Security["Security"]
            VPCSC[VPC Service Controls]
            SCC[Security Command Center]
        end
    end
    
    Clients --> LB
    LB --> CDN
    CDN --> IAP
    IAP --> Firebase
    Firebase --> Run
    Run --> GCS
    Run --> Firestore
    Run --> Vertex
    Run --> BQ
    VPCSC --> GCS
    VPCSC --> BQ
```

---

### 3.2 AWS Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        Web[Web]
        Mobile[Mobile]
    end
    
    subgraph AWS["AWS"]
        subgraph Edge["Edge"]
            CloudFront[CloudFront]
            WAF[AWS WAF]
            Cognito[Cognito]
        end
        
        subgraph API["API"]
            APIGW[API Gateway]
            Lambda[Lambda]
            ECS[ECS Fargate]
        end
        
        subgraph Storage["Storage"]
            S3[S3]
            DynamoDB[DynamoDB]
        end
        
        subgraph AI["AI"]
            Rekognition[Rekognition]
            SageMaker[SageMaker]
        end
        
        subgraph DW["Data Warehouse"]
            Redshift[Redshift]
        end
        
        subgraph Security["Security"]
            Macie[Macie]
            GuardDuty[GuardDuty]
        end
    end
    
    Clients --> CloudFront
    CloudFront --> WAF
    WAF --> Cognito
    Cognito --> APIGW
    APIGW --> Lambda
    APIGW --> ECS
    Lambda --> S3
    Lambda --> DynamoDB
    Lambda --> Rekognition
    Lambda --> Redshift
```

---

### 3.3 Azure Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        Web[Web]
        Mobile[Mobile]
    end
    
    subgraph Azure["Azure"]
        subgraph Edge["Edge"]
            FrontDoor[Azure Front Door]
            WAF[Azure WAF]
            Entra[Entra ID]
        end
        
        subgraph API["API"]
            APIM[API Management]
            Functions[Azure Functions]
            Container[Container Apps]
        end
        
        subgraph Storage["Storage"]
            Blob[Blob Storage]
            CosmosDB[Cosmos DB]
        end
        
        subgraph AI["AI"]
            Vision[Computer Vision]
            ML[Azure ML]
        end
        
        subgraph DW["Data Warehouse"]
            Synapse[Synapse Analytics]
        end
        
        subgraph Security["Security"]
            Purview[Purview]
            Defender[Defender for Cloud]
        end
    end
    
    Clients --> FrontDoor
    FrontDoor --> WAF
    WAF --> Entra
    Entra --> APIM
    APIM --> Functions
    APIM --> Container
    Functions --> Blob
    Functions --> CosmosDB
    Functions --> Vision
    Functions --> Synapse
```

---

## 4. Service Mapping (All Three Clouds)

| Capability | GCP | AWS | Azure |
|------------|-----|-----|-------|
| **Auth** | Firebase Auth, IAP | Cognito | Entra ID, B2C |
| **API** | Cloud Run, API Gateway | API Gateway, Lambda | API Management, Functions |
| **Object storage** | Cloud Storage | S3 | Blob Storage |
| **NoSQL** | Firestore | DynamoDB | Cosmos DB |
| **Data warehouse** | BigQuery | Redshift | Synapse Analytics |
| **Vision/AI** | Vertex AI Vision | Rekognition | Computer Vision |
| **CDN** | Cloud CDN | CloudFront | Front Door + CDN |
| **WAF** | Cloud Armor | AWS WAF | Azure WAF |
| **Exfiltration control** | VPC SC | Macie, SCP | Purview, Defender |

---

## 5. Data Flow: 1–10 MB Rapid Ingest

```mermaid
flowchart TB
    subgraph Ingest["Ingest Path"]
        Client[Client]
        SignedURL[Signed URL]
        Storage[Object Storage]
        Event[Object Event]
        Queue[Message Queue]
        Process[Process Worker]
    end
    
    Client -->|1. Request URL| SignedURL
    SignedURL -->|2. Return URL| Client
    Client -->|3. Direct upload| Storage
    Storage -->|4. Event| Event
    Event -->|5. Publish| Queue
    Queue -->|6. Consume| Process
```

| Cloud | Signed URL | Event | Queue |
|-------|------------|-------|-------|
| **GCP** | Cloud Storage signed URL | Eventarc (GCS trigger) | Pub/Sub |
| **AWS** | S3 presigned URL | S3 Event Notifications | SQS |
| **Azure** | Blob SAS URL | Event Grid | Service Bus |

---

## 6. Zero Trust & Security Design

### 6.1 Zero Trust Layers

```mermaid
flowchart TB
    subgraph Identity["Identity"]
        MFA[MFA]
        Token[JWT / OAuth]
        SA[Service Accounts]
    end
    
    subgraph Network["Network"]
        Private[Private Endpoints]
        FW[Firewall]
        Segment[Micro-segmentation]
    end
    
    subgraph Data["Data"]
        Encrypt[Encryption at rest]
        CMEK[Customer-managed keys]
        DLP[DLP]
    end
    
    subgraph Exfil["Exfiltration Prevention"]
        VPCSC[VPC SC / SCP / Purview]
        Audit[Audit logs]
    end
    
    Identity --> Network
    Network --> Data
    Data --> Exfil
```

---

### 6.2 Exfiltration Prevention

| Cloud | Control | Purpose |
|-------|---------|---------|
| **GCP** | VPC Service Controls | Block copy to projects outside perimeter; restrict exports |
| **GCP** | DLP | Detect PII; block sensitive data egress |
| **AWS** | SCP (Service Control Policies) | Restrict cross-account data movement |
| **AWS** | Macie | Detect sensitive data; alert on exposure |
| **Azure** | Purview | Data governance; egress policies |
| **Azure** | Defender for Storage | Threat detection; exfiltration alerts |

---

### 6.3 Secure Infrastructure Checklist

- [ ] Private endpoints for storage, DB, APIs (no public IP)
- [ ] Identity-based access (no network-based trust)
- [ ] Encryption at rest (CMEK where required)
- [ ] Encryption in transit (TLS 1.3)
- [ ] VPC SC / SCP / Purview perimeter
- [ ] DLP for PII/PHI
- [ ] Audit logging; immutable logs
- [ ] Least privilege IAM; no broad roles

---

## 7. High Availability Design

### 7.1 Within Region (Single Region HA)

```mermaid
flowchart TB
    subgraph ZoneA["Zone A"]
        App1[App Instance]
        DB1[DB Primary]
    end
    
    subgraph ZoneB["Zone B"]
        App2[App Instance]
        DB2[DB Replica]
    end
    
    LB[Load Balancer] --> App1
    LB --> App2
    App1 --> DB1
    App2 --> DB2
    DB1 -->|Sync| DB2
```

| Component | GCP | AWS | Azure |
|-----------|-----|-----|-------|
| **LB** | Regional External LB | ALB (multi-AZ) | Load Balancer (multi-AZ) |
| **Compute** | GKE / Cloud Run (multi-zone) | EKS / ECS (multi-AZ) | AKS / Container Apps (multi-zone) |
| **DB** | Cloud SQL HA, Firestore | RDS Multi-AZ, DynamoDB | SQL DB, Cosmos DB |
| **Storage** | GCS (multi-zone) | S3 (multi-AZ) | Blob (RA-GRS) |

---

### 7.2 Across Regions (Multi-Region HA)

```mermaid
flowchart TB
    subgraph Region1["Region 1 (Primary)"]
        LB1[Global LB]
        App1[App]
        DB1[DB]
    end
    
    subgraph Region2["Region 2 (Secondary)"]
        LB2[Global LB]
        App2[App]
        DB2[DB Replica]
    end
    
    Users[Users] --> LB1
    LB1 --> App1
    LB1 --> App2
    App1 --> DB1
    App2 --> DB2
    DB1 -->|Async replicate| DB2
```

| Component | GCP | AWS | Azure |
|-----------|-----|-----|-------|
| **Global LB** | Global HTTP(S) LB | CloudFront + ALB | Front Door |
| **DB** | Spanner, Firestore | DynamoDB Global, Aurora Global | Cosmos DB (multi-region) |
| **Storage** | GCS (multi-region) | S3 Cross-Region Replication | Blob GRS, Geo-redundant |
| **Failover** | Traffic Director, health checks | Route 53 failover | Traffic Manager |

---

## 8. Observability & Troubleshooting

### 8.1 Metrics to Collect

| Category | Metrics | Purpose |
|----------|---------|---------|
| **Ingest** | Upload rate, latency P50/P99, errors | Throughput; bottlenecks |
| **Processing** | Queue depth, processing time, failures | Backlog; retries |
| **Storage** | Object count, size, tier distribution | Capacity; cost |
| **API** | Request rate, latency, 4xx/5xx | Availability; errors |
| **Auth** | Login success/fail, token refresh | Security; UX |
| **Inference** | Model latency, throughput | AI performance |
| **DB** | Connections, query latency | Database health |

---

### 8.2 Observability Stack (Per Cloud)

| Capability | GCP | AWS | Azure |
|------------|-----|-----|-------|
| **Metrics** | Cloud Monitoring | CloudWatch | Azure Monitor |
| **Logs** | Cloud Logging | CloudWatch Logs | Log Analytics |
| **Traces** | Cloud Trace | X-Ray | Application Insights |
| **Dashboards** | Looker Studio, Grafana | CloudWatch, Grafana | Azure Dashboards, Grafana |
| **Alerting** | Alerting policies | CloudWatch Alarms | Action Groups |
| **APM** | Cloud Profiler | X-Ray, CodeGuru | Application Insights |

---

### 8.3 Log Analysis & Troubleshooting

```mermaid
flowchart TB
    subgraph Sources["Log Sources"]
        App[Application]
        API[API Gateway]
        Storage[Storage]
        Auth[Auth]
    end
    
    subgraph Ingest["Ingest"]
        Logging[Central Logging]
    end
    
    subgraph Analysis["Analysis"]
        Query[Log Query]
        Alert[Alerts]
        Dashboard[Dashboard]
    end
    
    Sources --> Logging
    Logging --> Query
    Logging --> Alert
    Logging --> Dashboard
```

**Troubleshooting queries (example – GCP Logging):**
```
# Upload failures
resource.type="cloud_run_revision"
severity>=ERROR
textPayload=~"upload"

# Slow inference
metric.type="run.googleapis.com/request_latencies"
metric.labels.response_code!="200"

# Auth failures
protoPayload.serviceName="firebaseauth.googleapis.com"
protoPayload.methodName="SignIn"
protoPayload.status.code!=0
```

---

### 8.4 Troubleshooting Tools Matrix

| Issue | GCP | AWS | Azure |
|-------|-----|-----|-------|
| **Trace request** | Cloud Trace | X-Ray | Application Insights |
| **Log search** | Log Explorer | CloudWatch Logs Insights | Log Analytics KQL |
| **Metric drill-down** | Metrics Explorer | CloudWatch Metrics | Metrics Explorer |
| **Error budget** | SLO Monitoring | CloudWatch SLO | Azure SLO |
| **Incident** | Incident Manager | Incident Manager | Azure Monitor Alerts |

---

## 9. End-to-End Feature Flow

### 9.1 Upload → Store → Categorize → Infer

```mermaid
flowchart LR
    Upload[Upload] --> Store[Store]
    Store --> Event[Event]
    Event --> Categorize[Categorize]
    Categorize --> Infer[Infer]
    Infer --> Meta[Metadata DB]
    Infer --> DW[Data Warehouse]
```

### 9.2 Browse & Share

```mermaid
flowchart TB
    User[User] --> Auth[Auth]
    Auth --> API[API]
    API --> Gallery[Gallery Service]
    Gallery --> Storage[Storage]
    Gallery --> DB[DB]
    
    User --> Share[Share]
    Share --> API
    API --> ShareLink[Share Link]
    ShareLink --> DB
```

### 9.3 Analytics (Management Data Warehouse)

```mermaid
flowchart TB
    subgraph Sources["Sources"]
        App[App Events]
        Storage[Storage Events]
        Auth[Auth Events]
    end
    
    subgraph ETL["ETL"]
        Pipeline[Pipeline]
    end
    
    subgraph DW["Data Warehouse"]
        Tables[Tables]
        Reports[Reports]
    end
    
    Sources --> Pipeline
    Pipeline --> Tables
    Tables --> Reports
```

| Cloud | ETL | Data Warehouse |
|-------|-----|----------------|
| **GCP** | Dataflow, BigQuery Transfer | BigQuery |
| **AWS** | Glue, DMS | Redshift |
| **Azure** | Data Factory, Synapse pipelines | Synapse Analytics |

---

## 10. Multi-Cloud Design Summary

| Aspect | GCP | AWS | Azure |
|--------|-----|-----|-------|
| **Auth** | Firebase, IAP | Cognito | Entra, B2C |
| **Upload** | Signed URL, GCS | Presigned URL, S3 | SAS, Blob |
| **Process** | Cloud Functions, Dataflow | Lambda, SQS | Functions, Service Bus |
| **Inference** | Vertex AI Vision | Rekognition | Computer Vision |
| **Storage** | GCS, Firestore | S3, DynamoDB | Blob, Cosmos DB |
| **DW** | BigQuery | Redshift | Synapse |
| **Exfiltration** | VPC SC | Macie, SCP | Purview |
| **HA (region)** | Multi-zone | Multi-AZ | Multi-zone |
| **HA (global)** | Global LB, Spanner | CloudFront, DynamoDB Global | Front Door, Cosmos DB |
| **Observability** | Cloud Ops | CloudWatch | Azure Monitor |

---

## 11. Applying This Design to Other Solutions

Use this multi-cloud, secure, HA pattern for:

| Solution | Doc | Same Pattern |
|----------|-----|--------------|
| **Healthcare imaging (5 PB)** | 19-healthcare-imaging-ai-pipeline.md | Add GCP/AWS/Azure mapping; VPC SC/SCP/Purview; multi-region HA |
| **Architecture scenarios** | 18-architecture-scenarios-gcp-aws.md | Extend to Azure; add observability matrix |
| **Photo processing (earlier)** | 20-photo-processing-app-infrastructure.md | Superseded by this doc (21) |

**Template for any solution:**
1. Define GCP, AWS, Azure service mapping
2. Add zero trust + exfiltration controls per cloud
3. Design HA within region (multi-zone/AZ)
4. Design HA across regions (global LB, global DB)
5. Define observability stack (metrics, logs, traces, alerts)
6. Add troubleshooting queries and dashboards
