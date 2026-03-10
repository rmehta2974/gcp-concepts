# Architecture Scenarios: GCP & AWS

Real-world architecture patterns for real-time services, ETL, infrastructure, and AI workloads on both GCP and AWS.

---

## 1. Real-Time Services

### 1.1 Real-Time Event Streaming (Pub/Sub vs Kinesis)

**GCP Architecture**

```mermaid
flowchart TB
    subgraph Sources["Event Sources"]
        API[API / Webhooks]
        IoT[IoT Devices]
        App[Applications]
    end
    
    subgraph GCP["GCP"]
        PubSub[Cloud Pub/Sub]
        Dataflow[Dataflow]
        BQ[BigQuery]
        Run[Cloud Run]
    end
    
    Sources --> PubSub
    PubSub --> Dataflow
    PubSub --> Run
    Dataflow --> BQ
```

| Component | GCP | AWS |
|-----------|-----|-----|
| **Messaging** | Cloud Pub/Sub | Amazon Kinesis Data Streams / SQS |
| **Stream processing** | Dataflow (Apache Beam) | Kinesis Data Analytics / Flink |
| **Serverless consumer** | Cloud Functions / Cloud Run | Lambda |
| **Real-time DB** | Firestore / Bigtable | DynamoDB |

---

**AWS Architecture**

```mermaid
flowchart TB
    subgraph Sources["Event Sources"]
        API[API Gateway]
        IoT[IoT Core]
        App[Applications]
    end
    
    subgraph AWS["AWS"]
        Kinesis[Kinesis Data Streams]
        Lambda[Lambda]
        KDA[Kinesis Data Analytics]
        DynamoDB[DynamoDB]
    end
    
    Sources --> Kinesis
    Kinesis --> Lambda
    Kinesis --> KDA
    Lambda --> DynamoDB
```

---

### 1.2 Real-Time Analytics Dashboard

**GCP**

```mermaid
flowchart LR
    PubSub[Pub/Sub] --> Dataflow[Dataflow]
    Dataflow --> BQ[BigQuery]
    BQ --> Looker[Looker Studio]
    BQ --> BI[Looker / Metabase]
```

**AWS**

```mermaid
flowchart LR
    Kinesis[Kinesis] --> KDA[KDA / Glue Streaming]
    KDA --> Redshift[Redshift]
    Redshift --> QuickSight[QuickSight]
```

---

### 1.3 Real-Time Fraud Detection

**GCP**

```mermaid
flowchart TB
    Txn[Transaction Events] --> PubSub[Pub/Sub]
    PubSub --> Dataflow[Dataflow]
    Dataflow --> Vertex[Vertex AI Model]
    Dataflow --> BQ[BigQuery]
    Vertex --> Alert[Alert / Block]
```

**AWS**

```mermaid
flowchart TB
    Txn[Transaction Events] --> Kinesis[Kinesis]
    Kinesis --> Lambda[Lambda]
    Lambda --> SageMaker[SageMaker Endpoint]
    SageMaker --> Alert[Alert / Block]
```

---

## 2. ETL Architectures

### 2.1 Batch ETL (Scheduled)

**GCP**

```mermaid
flowchart TB
    subgraph Sources["Sources"]
        GCS[Cloud Storage]
        OnPrem[On-Prem DB]
        SaaS[SaaS APIs]
    end
    
    subgraph ETL["ETL"]
        Scheduler[Cloud Scheduler]
        CF[Cloud Functions]
        Dataflow[Dataflow]
    end
    
    subgraph Dest["Destinations"]
        BQ[BigQuery]
        GCS2[Cloud Storage]
    end
    
    Scheduler --> CF
    CF --> Dataflow
    Sources --> Dataflow
    Dataflow --> BQ
    Dataflow --> GCS2
```

**AWS**

```mermaid
flowchart TB
    subgraph Sources["Sources"]
        S3[S3]
        RDS[RDS]
        APIs[APIs]
    end
    
    subgraph ETL["ETL"]
        EventBridge[EventBridge]
        Lambda[Lambda]
        Glue[Glue ETL]
    end
    
    subgraph Dest["Destinations"]
        Redshift[Redshift]
        S3Out[S3]
    end
    
    EventBridge --> Lambda
    Lambda --> Glue
    Sources --> Glue
    Glue --> Redshift
    Glue --> S3Out
```

---

### 2.2 CDC (Change Data Capture) ETL

**GCP**

```mermaid
flowchart LR
    CloudSQL[Cloud SQL] --> Datastream[Datastream]
    Datastream --> BQ[BigQuery]
    Datastream --> PubSub[Pub/Sub]
```

**AWS**

```mermaid
flowchart LR
    RDS[RDS / Aurora] --> DMS[DMS]
    DMS --> Redshift[Redshift]
    DMS --> Kinesis[Kinesis]
```

---

### 2.3 Streaming ETL

**GCP**

```mermaid
flowchart TB
    PubSub[Pub/Sub] --> Dataflow[Dataflow]
    Dataflow --> BQ[BigQuery]
    Dataflow --> Bigtable[Bigtable]
    Dataflow --> GCS[Cloud Storage]
```

**AWS**

```mermaid
flowchart TB
    Kinesis[Kinesis] --> KDA[Kinesis Data Analytics]
    KDA --> Redshift[Redshift]
    KDA --> S3[S3]
    KDA --> OpenSearch[OpenSearch]
```

---

## 3. Infrastructure Scenarios

### 3.1 Multi-Region Active-Active

**GCP**

```mermaid
flowchart TB
    LB[Global HTTP/S LB]
    LB --> R1[us-east4]
    LB --> R2[us-west2]
    LB --> R3[europe-west1]
    
    R1 --> GKE1[GKE]
    R2 --> GKE2[GKE]
    R3 --> GKE3[GKE]
    
    GKE1 --> Spanner[Spanner]
    GKE2 --> Spanner
    GKE3 --> Spanner
```

**AWS**

```mermaid
flowchart TB
    CloudFront[CloudFront]
    CloudFront --> ALB1[ALB us-east-1]
    CloudFront --> ALB2[ALB us-west-2]
    
    ALB1 --> EKS1[EKS]
    ALB2 --> EKS2[EKS]
    
    EKS1 --> DynamoDB[DynamoDB Global]
    EKS2 --> DynamoDB
```

---

### 3.2 Hybrid (On-Prem + Cloud)

**GCP**

```mermaid
flowchart TB
    subgraph OnPrem["On-Premises"]
        DC[Data Center]
    end
    
    subgraph GCP["GCP"]
        Interconnect[Cloud Interconnect]
        VPC[Shared VPC]
        GKE[GKE]
    end
    
    DC <--> Interconnect
    Interconnect --> VPC
    VPC --> GKE
```

**AWS**

```mermaid
flowchart TB
    subgraph OnPrem["On-Premises"]
        DC[Data Center]
    end
    
    subgraph AWS["AWS"]
        DirectConnect[Direct Connect]
        VPC[VPC]
        EKS[EKS]
    end
    
    DC <--> DirectConnect
    DirectConnect --> VPC
    VPC --> EKS
```

---

### 3.3 Multi-Cloud (GCP + AWS)

```mermaid
flowchart TB
    subgraph GCP["GCP"]
        GKE[GKE]
        BQ[BigQuery]
    end
    
    subgraph AWS["AWS"]
        EKS[EKS]
        S3[S3]
    end
    
    subgraph Connectivity["Connectivity"]
        VPN[Site-to-Site VPN]
        Interconnect[Partner Interconnect]
    end
    
    GKE <--> VPN
    EKS <--> VPN
    GKE --> BQ
    EKS --> S3
```

---

## 4. AI / ML Architectures

### 4.1 Batch Inference Pipeline

**GCP**

```mermaid
flowchart TB
    GCS[Cloud Storage] --> Vertex[Vertex AI]
    BQ[BigQuery] --> Vertex
    Vertex --> Training[Training Job]
    Training --> Model[Model Registry]
    Model --> BatchPred[Batch Prediction]
    BatchPred --> BQOut[BigQuery]
```

**AWS**

```mermaid
flowchart TB
    S3[S3] --> SageMaker[SageMaker]
    Redshift[Redshift] --> SageMaker
    SageMaker --> Training[Training Job]
    Training --> Model[Model Registry]
    Model --> BatchPred[Batch Transform]
    BatchPred --> S3Out[S3]
```

---

### 4.2 Real-Time Inference

**GCP**

```mermaid
flowchart TB
    API[API / App] --> Vertex[Vertex AI Endpoint]
    Vertex --> Model[Model]
    Model --> Response[Response]
    
    subgraph Scaling["Auto-scaling"]
        Vertex
    end
```

**AWS**

```mermaid
flowchart TB
    API[API Gateway] --> SageMaker[SageMaker Endpoint]
    SageMaker --> Model[Model]
    Model --> Response[Response]
    
    subgraph Scaling["Auto-scaling"]
        SageMaker
    end
```

---

### 4.3 Generative AI (LLM) Architecture

**GCP**

```mermaid
flowchart TB
    subgraph GCP["GCP"]
        App[Application]
        Vertex[Vertex AI]
        Gemini[Gemini API]
        Embed[Embedding API]
        VectorDB[Vertex AI Vector Search]
    end
    
    App --> Vertex
    Vertex --> Gemini
    Vertex --> Embed
    Embed --> VectorDB
    Gemini --> VectorDB
```

**AWS**

```mermaid
flowchart TB
    subgraph AWS["AWS"]
        App[Application]
        Bedrock[Amazon Bedrock]
        SageMaker[SageMaker]
        OpenSearch[OpenSearch Vector]
    end
    
    App --> Bedrock
    App --> SageMaker
    Bedrock --> OpenSearch
    SageMaker --> OpenSearch
```

---

### 4.4 MLOps Pipeline

**GCP**

```mermaid
flowchart LR
    Data[Data] --> Vertex[Vertex AI Pipelines]
    Vertex --> Train[Train]
    Train --> Eval[Evaluate]
    Eval --> Deploy[Deploy]
    Deploy --> Endpoint[Endpoint]
```

**AWS**

```mermaid
flowchart LR
    Data[Data] --> SageMaker[SageMaker Pipelines]
    SageMaker --> Train[Train]
    Train --> Eval[Evaluate]
    Eval --> Deploy[Deploy]
    Deploy --> Endpoint[Endpoint]
```

---

## 5. Scenario Comparison Matrix

| Scenario | GCP Primary | AWS Primary |
|----------|-------------|-------------|
| **Real-time events** | Pub/Sub + Dataflow | Kinesis + Lambda |
| **Batch ETL** | Dataflow, Cloud Composer | Glue, Step Functions |
| **CDC** | Datastream | DMS |
| **Streaming ETL** | Dataflow | Kinesis Data Analytics |
| **Real-time analytics** | BigQuery + Looker | Redshift + QuickSight |
| **ML training** | Vertex AI | SageMaker |
| **LLM / GenAI** | Vertex AI, Gemini | Bedrock, SageMaker |
| **Multi-region** | Global LB + Spanner | CloudFront + DynamoDB Global |
| **Hybrid** | Interconnect | Direct Connect |

---

## 6. Quick Reference: Service Mapping

| Capability | GCP | AWS |
|------------|-----|-----|
| **Messaging** | Pub/Sub | SQS, SNS, Kinesis |
| **Stream processing** | Dataflow | Kinesis Data Analytics, EMR |
| **Data warehouse** | BigQuery | Redshift |
| **Orchestration** | Cloud Composer (Airflow) | Step Functions, MWAA |
| **ML platform** | Vertex AI | SageMaker |
| **GenAI** | Vertex AI, Gemini | Bedrock |
| **Containers** | GKE | EKS |
| **Serverless** | Cloud Run, Functions | Lambda |
| **Object storage** | Cloud Storage | S3 |
