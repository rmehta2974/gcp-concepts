# Photo Processing App & Infrastructure Design

Architecture for a photo processing application with large-scale streaming ingest and storage.

---

## Overview

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        Mobile[Mobile App]
        Web[Web App]
        API[Third-Party API]
    end
    
    subgraph Ingest["Streaming Ingest"]
        LB[Load Balancer]
        Run[Cloud Run]
        PubSub[Pub/Sub]
    end
    
    subgraph Process["Processing"]
        CF[Cloud Functions]
        Dataflow[Dataflow]
        Vision[Vision API]
    end
    
    subgraph Store["Storage"]
        GCS[Cloud Storage]
        BQ[BigQuery]
    end
    
    Clients --> LB
    LB --> Run
    Run --> PubSub
    PubSub --> CF
    PubSub --> Dataflow
    CF --> Vision
    Dataflow --> GCS
    Dataflow --> BQ
```

---

## 1. Application Design

### 1.1 High-Level Architecture

```mermaid
flowchart LR
    subgraph Frontend["Frontend"]
        Web[Web]
        Mobile[Mobile]
    end
    
    subgraph API["API Layer"]
        Gateway[API Gateway]
        Auth[Auth]
    end
    
    subgraph Backend["Backend"]
        Upload[Upload Service]
        Process[Process Service]
        Search[Search Service]
    end
    
    subgraph Data["Data"]
        GCS[Storage]
        DB[Database]
    end
    
    Frontend --> Gateway
    Gateway --> Auth
    Auth --> Upload
    Auth --> Process
    Auth --> Search
    Upload --> GCS
    Process --> GCS
    Search --> DB
```

---

### 1.2 Upload Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant GCS
    participant PubSub
    
    Client->>API: POST /upload (multipart)
    API->>GCS: Signed URL or direct write
    Client->>GCS: Upload bytes (streaming)
    GCS->>PubSub: Object finalize event
    PubSub->>API: Trigger processing
```

**Options for large uploads:**

| Method | Use Case | Max Size | Pros |
|--------|----------|----------|------|
| **Direct to GCS** | Resumable, large | 5 TB | No proxy; resumable |
| **Signed URL** | Client upload | 5 TB | No server bandwidth |
| **Chunked upload** | Resumable | 5 TB | Resume on failure |
| **Stream through API** | Small–medium | ~100 MB | Simple; single request |

---

### 1.3 Processing Pipeline

```mermaid
flowchart TB
    subgraph Ingest["Ingest"]
        Event[Object Finalize]
        PubSub[Pub/Sub]
    end
    
    subgraph Process["Process"]
        Resize[Resize / Thumbnail]
        Enhance[Enhance / Filter]
        Extract[Metadata Extract]
        Vision[Vision API - Labels]
    end
    
    subgraph Output["Output"]
        Thumbs[Thumbnails GCS]
        Meta[Metadata DB]
        Search[Search Index]
    end
    
    Event --> PubSub
    PubSub --> Resize
    Resize --> Enhance
    Enhance --> Extract
    Extract --> Vision
    Vision --> Thumbs
    Vision --> Meta
    Meta --> Search
```

---

## 2. Large Streaming Ingest

### 2.1 Streaming Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        C1[Client 1]
        C2[Client 2]
        Cn[Client N]
    end
    
    subgraph Edge["Edge"]
        LB[Global HTTP/S LB]
        CDN[Cloud CDN]
    end
    
    subgraph Ingest["Ingest"]
        Run[Cloud Run]
        GCS[Cloud Storage]
        Eventarc[Eventarc]
    end
    
    subgraph Queue["Queue"]
        PubSub[Pub/Sub]
    end
    
    C1 --> CDN
    C2 --> CDN
    Cn --> CDN
    CDN --> LB
    LB --> Run
    Run --> GCS
    GCS --> Eventarc
    Eventarc --> PubSub
```

---

### 2.2 Resumable Upload (Client → GCS)

```mermaid
flowchart LR
    Client[Client] -->|1. Request signed URL| API[API]
    API -->|2. Return upload URL| Client
    Client -->|3. Chunked PUT| GCS[GCS]
    GCS -->|4. Finalize event| PubSub[Pub/Sub]
```

**Resumable upload flow:**
1. Client requests upload URL from API
2. API returns signed resumable upload URL (or initiates session)
3. Client uploads in chunks (e.g., 256 KB–5 MB)
4. On failure, client resumes from last byte
5. GCS emits `OBJECT_FINALIZE` → triggers processing

---

### 2.3 High-Throughput Ingest

| Component | Role |
|-----------|------|
| **Cloud Run** | Stateless; scales to 0; handles auth + URL generation |
| **Cloud Storage** | Receives bytes; no proxy; multi-region |
| **Eventarc** | GCS → Pub/Sub on object finalize |
| **Pub/Sub** | Decouples ingest from processing; backpressure |
| **Cloud CDN** | Cache metadata; reduce origin load |

---

### 2.4 Scaling for Large Streams

```mermaid
flowchart TB
    subgraph Scale["Scaling"]
        LB[Load Balancer]
        Run[Cloud Run - 1000+ instances]
        GCS[GCS - unlimited]
        PubSub[Pub/Sub - 10M+ msg/s]
    end
    
    LB --> Run
    Run --> GCS
    GCS --> PubSub
```

| Service | Scale | Tuning |
|---------|-------|--------|
| **Cloud Run** | 0–1000+ | Concurrency; min instances for cold start |
| **Cloud Storage** | Unlimited | Prefix sharding; avoid hot keys |
| **Pub/Sub** | 10M msg/s | Ordering; ack deadline; push vs pull |
| **Dataflow** | Horizontal | Worker count; autoscaling |

---

## 3. Storage Design

### 3.1 Storage Layout

```mermaid
flowchart TB
    subgraph Bucket["Bucket: photos-{env}"]
        subgraph Raw["raw/"]
            R1[user_id/year/month/day/photo_id.raw]
        end
        subgraph Processed["processed/"]
            P1[user_id/photo_id/thumb_256.jpg]
            P2[user_id/photo_id/thumb_1024.jpg]
            P3[user_id/photo_id/full.jpg]
        end
        subgraph Temp["temp/"]
            T1[upload_id/chunk]
        end
    end
```

**Naming convention:**
```
gs://photos-prod/raw/{user_id}/{year}/{month}/{day}/{photo_id}.{ext}
gs://photos-prod/processed/{user_id}/{photo_id}/thumb_256.jpg
gs://photos-prod/processed/{user_id}/{photo_id}/thumb_1024.jpg
gs://photos-prod/processed/{user_id}/{photo_id}/full.jpg
```

---

### 3.2 Lifecycle & Tiering

```mermaid
flowchart LR
    Standard[Standard] -->|30 days no access| Nearline[Nearline]
    Nearline -->|90 days no access| Coldline[Coldline]
    Coldline -->|1 year no access| Archive[Archive]
```

| Tier | Use Case | Cost |
|------|----------|------|
| **Standard** | Active; recent uploads | Highest |
| **Nearline** | Thumbnails; occasional view | Medium |
| **Coldline** | Archive; compliance | Low |
| **Archive** | Long-term; rarely accessed | Lowest |

---

### 3.3 Capacity Planning

| Scenario | Est. Size | Approach |
|----------|-----------|----------|
| **1M users, 100 photos each** | ~10 TB | Single bucket; prefix by user |
| **10M users, 1000 photos each** | ~1 PB | Multi-bucket; shard by user hash |
| **100M+ photos** | 10+ PB | Multiple buckets; lifecycle rules |

---

## 4. End-to-End Infrastructure

```mermaid
flowchart TB
    subgraph Clients["Clients"]
        Web[Web]
        Mobile[Mobile]
    end
    
    subgraph GCP["GCP"]
        subgraph Ingest["Ingest"]
            LB[Load Balancer]
            Run[Cloud Run]
            GCS[Cloud Storage]
            Eventarc[Eventarc]
            PubSub[Pub/Sub]
        end
        
        subgraph Process["Process"]
            CF[Cloud Functions]
            Vision[Vision API]
        end
        
        subgraph Store["Store"]
            GCS2[Cloud Storage]
            BQ[BigQuery]
            Firestore[Firestore]
        end
    end
    
    Web --> LB
    Mobile --> LB
    LB --> Run
    Run --> GCS
    GCS --> Eventarc
    Eventarc --> PubSub
    PubSub --> CF
    CF --> Vision
    Vision --> GCS2
    Vision --> BQ
    CF --> Firestore
```

---

## 5. Component Summary

| Component | Service | Purpose |
|-----------|---------|---------|
| **Upload API** | Cloud Run | Auth; signed URLs; metadata |
| **Streaming storage** | Cloud Storage | Raw + processed; resumable upload |
| **Event trigger** | Eventarc + Pub/Sub | Object finalize → process |
| **Processing** | Cloud Functions / Dataflow | Resize; enhance; extract |
| **Vision** | Vision API | Labels; faces; objects |
| **Metadata** | Firestore / BigQuery | Search; queries |
| **CDN** | Cloud CDN | Cache thumbnails; reduce latency |

---

## 6. AWS Equivalent (Reference)

| GCP | AWS |
|-----|-----|
| Cloud Storage | S3 |
| Cloud Run | Lambda / ECS Fargate |
| Pub/Sub | SQS / SNS / EventBridge |
| Eventarc | S3 Event Notifications |
| Vision API | Rekognition |
| Firestore | DynamoDB |
| Cloud CDN | CloudFront |
