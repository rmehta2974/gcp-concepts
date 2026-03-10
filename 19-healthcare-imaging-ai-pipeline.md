# Healthcare Imaging AI Pipeline Architecture

5 PB image processing ETL, metadata cataloging, Vertex AI inference, Gemini integration, LLM accuracy monitoring, data cataloging, and patient–image–disease training pipeline on GCP.

---

## Overview

```mermaid
flowchart TB
    subgraph Ingest["1. Ingest & ETL"]
        Images[5 PB Images]
        Metadata[Metadata]
        Patient[Patient Data]
    end
    
    subgraph Catalog["2. Catalog & Unify"]
        DC[Data Catalog]
        Unified[Unified Schema]
    end
    
    subgraph Inference["3. Inference"]
        Vertex[Vertex AI]
        Gemini[Gemini]
    end
    
    subgraph Monitor["4. Monitor & Measure"]
        LLMEval[LLM Accuracy]
        Metrics[Metrics]
    end
    
    subgraph Train["5. Training"]
        Training[Model Training]
        Disease[Disease Category]
    end
    
    Ingest --> Catalog
    Catalog --> Inference
    Inference --> Monitor
    Catalog --> Train
    Train --> Inference
```

---

## 1. 5 PB Image Processing ETL

### Architecture

```mermaid
flowchart TB
    subgraph Sources["Sources"]
        PACS[PACS / DICOM]
        Storage[Cloud Storage]
        OnPrem[On-Prem Archive]
    end
    
    subgraph Ingest["Ingest Pipeline"]
        Transfer[Transfer Service]
        Eventarc[Eventarc]
        CF[Cloud Functions]
    end
    
    subgraph Process["Processing"]
        Dataflow[Dataflow]
        Vertex[Vertex AI Vision]
    end
    
    subgraph Output["Output"]
        GCS[Cloud Storage]
        BQ[BigQuery]
    end
    
    Sources --> Transfer
    Transfer --> GCS
    GCS --> Eventarc
    Eventarc --> CF
    CF --> Dataflow
    Dataflow --> Vertex
    Vertex --> GCS
    Dataflow --> BQ
```

### Design Choices for 5 PB Scale

| Concern | Solution | Rationale |
|---------|----------|-----------|
| **Storage** | Cloud Storage (Standard → Nearline/Archive) | Lifecycle rules; cost optimization |
| **Transfer** | Transfer Service (batch), Storage Transfer Service | Multi-threaded; parallel; resumable |
| **Processing** | Dataflow (Apache Beam) + Vertex AI Vision | Horizontal scale; GPU workers |
| **Chunking** | Process by study/series; shard by prefix | Avoid hotspots; parallelize |
| **Metadata** | Extract to BigQuery; DICOM tags | Queryable; joinable with patient |

### Dataflow Pipeline Pattern

```mermaid
flowchart LR
    GCS[GCS Bucket] --> Read[Read Images]
    Read --> Parse[Parse DICOM]
    Parse --> Extract[Extract Metadata]
    Extract --> Vision[Vertex AI Vision]
    Vision --> WriteMeta[Write to BigQuery]
    Vision --> WriteImg[Write Processed to GCS]
```

### Storage Tiering (5 PB)

| Tier | Use Case | Cost |
|------|----------|------|
| **Standard** | Hot / active processing | Highest |
| **Nearline** | Processed; occasional access | Medium |
| **Coldline** | Archive; compliance | Low |
| **Archive** | Long-term retention | Lowest |

---

## 2. Cataloging & Metadata Processing

### Architecture

```mermaid
flowchart TB
    subgraph Metadata["Metadata Sources"]
        DICOM[DICOM Tags]
        FHIR[FHIR / Patient]
        Custom[Custom Tags]
    end
    
    subgraph Catalog["Data Catalog"]
        DC[Dataplex]
        Tag[Tag Templates]
        Lineage[Lineage]
    end
    
    subgraph Schema["Unified Schema"]
        BQ[BigQuery Tables]
        PatientID[patient_id]
        StudyID[study_id]
        DiseaseCat[disease_category]
        ImagePath[image_path]
    end
    
    Metadata --> DC
    DC --> Tag
    DC --> Lineage
    DC --> BQ
    BQ --> Schema
```

### Unified Schema (Patient + Image + Disease)

| Field | Source | Description |
|-------|--------|-------------|
| `patient_id` | FHIR / HL7 | De-identified patient ID |
| `study_id` | DICOM | Study UID |
| `series_id` | DICOM | Series UID |
| `image_path` | GCS | `gs://bucket/patient/study/series/image.dcm` |
| `disease_category` | Labeling / Model | ICD-10, SNOMED, custom |
| `modality` | DICOM | CT, MRI, X-Ray |
| `body_part` | DICOM / AI | Anatomical region |
| `metadata_json` | Extracted | Full DICOM tags |

### Dataplex for Governance

- **Data Catalog**: Discover, tag, search
- **Dataplex**: Lakehouse; unified metadata
- **Tag templates**: PII, PHI, sensitivity
- **Lineage**: Image → Extract → BigQuery → Model

---

## 3. Vertex AI Inference

### Architecture

```mermaid
flowchart TB
    subgraph Trigger["Trigger"]
        PubSub[Pub/Sub]
        BQ[BQ Export]
    end
    
    subgraph Inference["Vertex AI"]
        Endpoint[Model Endpoint]
        Batch[Batch Prediction]
        Vision[Vertex AI Vision]
    end
    
    subgraph Output["Output"]
        BQOut[BigQuery]
        GCS[GCS]
    end
    
    PubSub --> Endpoint
    BQ --> Batch
    Endpoint --> BQOut
    Batch --> GCS
    Vision --> BQOut
```

### Real-Time vs Batch

| Mode | Use Case | Component |
|------|----------|-----------|
| **Real-time** | Live PACS; urgent reads | Vertex AI Endpoint (GPU) |
| **Batch** | Bulk historical; ETL | Batch Prediction |
| **Vision API** | Pre-built models (labels, objects) | Vertex AI Vision |

---

## 4. Gemini Integration

### Architecture

```mermaid
flowchart TB
    subgraph Input["Input"]
        Report[Radiology Report]
        Metadata[Image Metadata]
        Patient[Patient Context]
    end
    
    subgraph Gemini["Gemini"]
        Multimodal[Gemini Multimodal]
        Embed[Embedding API]
    end
    
    subgraph Output["Output"]
        Summary[Report Summary]
        Coding[ICD Coding]
        QA[Q&A]
    end
    
    Report --> Multimodal
    Metadata --> Multimodal
    Patient --> Multimodal
    Multimodal --> Summary
    Multimodal --> Coding
    Multimodal --> QA
```

### Use Cases

- **Report summarization**: Long report → concise summary
- **ICD/SNOMED coding**: Free text → structured codes
- **Multimodal**: Image + report → combined interpretation
- **Q&A**: Clinician questions over report + metadata

---

## 5. Watching & Measuring LLM Accuracy

### Architecture

```mermaid
flowchart TB
    subgraph Inference["LLM Output"]
        Gemini[Gemini]
        Vertex[Vertex AI]
    end
    
    subgraph GroundTruth["Ground Truth"]
        Labels[Human Labels]
        Gold[Gold Dataset]
    end
    
    subgraph Eval["Evaluation"]
        VertexEval[Vertex AI Evaluation]
        Custom[Custom Metrics]
    end
    
    subgraph Monitor["Monitoring"]
        Logging[Cloud Logging]
        Monitoring[Cloud Monitoring]
        Dashboard[Looker Dashboard]
    end
    
    Gemini --> VertexEval
    Vertex --> VertexEval
    GroundTruth --> VertexEval
    VertexEval --> Logging
    VertexEval --> Monitoring
    Monitoring --> Dashboard
```

### Metrics to Track

| Metric | Description | Tool |
|--------|-------------|------|
| **Exact match** | Output vs gold | Custom / Vertex Eval |
| **BLEU / ROUGE** | Text similarity | Vertex AI Evaluation |
| **Clinical accuracy** | Domain expert review | Human-in-loop |
| **Latency** | P50, P99 | Cloud Monitoring |
| **Drift** | Input/output distribution | Vertex AI Model Monitoring |
| **Hallucination rate** | Factual errors | Custom eval |

### Evaluation Pipeline

```mermaid
flowchart LR
    Gold[Gold Dataset] --> Eval[Evaluation Job]
    Model[Model Version] --> Eval
    Eval --> Report[Eval Report]
    Report --> BQ[BigQuery]
    Report --> Alert[Alert on Regression]
```

---

## 6. Data Cataloging

### Architecture

```mermaid
flowchart TB
    subgraph Assets["Assets"]
        GCS[Cloud Storage]
        BQ[BigQuery]
    end
    
    subgraph Catalog["Catalog"]
        Dataplex[Dataplex]
        DLP[DLP - PII/PHI]
        Tags[Tag Templates]
    end
    
    subgraph Governance["Governance"]
        Policy[Access Policy]
        Lineage[Lineage]
        Quality[Data Quality]
    end
    
    Assets --> Dataplex
    Dataplex --> DLP
    Dataplex --> Tags
    Dataplex --> Policy
    Dataplex --> Lineage
    Dataplex --> Quality
```

### Tag Templates

- **sensitivity**: PHI, PII, de-identified
- **disease_category**: Oncology, Cardiology, etc.
- **modality**: CT, MRI, X-Ray
- **retention**: Compliance retention period

---

## 7. Combining Patient + Image + Disease & Training

### Unified Training Dataset

```mermaid
flowchart TB
    subgraph Sources["Sources"]
        Patient[Patient Data - FHIR]
        Image[Image Metadata - BigQuery]
        Disease[Disease Labels]
    end
    
    subgraph Join["Join & Enrich"]
        BQ[BigQuery SQL]
        Spark[Dataproc / Spark]
    end
    
    subgraph Training["Training"]
        Vertex[Vertex AI Training]
        Custom[Custom Training]
    end
    
    subgraph Model["Model"]
        Registry[Model Registry]
        Endpoint[Endpoint]
    end
    
    Patient --> Join
    Image --> Join
    Disease --> Join
    Join --> Training
    Training --> Registry
    Registry --> Endpoint
```

### Training Data Schema

| Field | Source | Use |
|-------|--------|-----|
| `patient_id` | FHIR | Cohort; demographics |
| `age`, `sex` | FHIR | Demographics |
| `study_id`, `series_id` | DICOM | Image reference |
| `image_uri` | GCS | Training input |
| `disease_category` | Labels / ICD | Target variable |
| `modality`, `body_part` | DICOM | Features |
| `report_text` | Report | Multimodal input |

### Training Pipeline

```mermaid
flowchart LR
    BQ[BigQuery Dataset] --> Export[Export to GCS]
    Export --> Vertex[Vertex AI Training]
    Vertex --> Tune[Hyperparameter Tuning]
    Tune --> Eval[Evaluation]
    Eval --> Registry[Model Registry]
    Registry --> Endpoint[Deploy]
```

---

## 8. End-to-End Architecture

```mermaid
flowchart TB
    subgraph Ingest["Ingest"]
        PACS[PACS]
        Transfer[Transfer Service]
        GCS1[GCS 5 PB]
    end
    
    subgraph ETL["ETL"]
        Dataflow[Dataflow]
        Extract[Metadata Extract]
        BQ1[BigQuery]
    end
    
    subgraph Catalog["Catalog"]
        Dataplex[Dataplex]
        Unified[Patient+Image+Disease]
    end
    
    subgraph AI["AI"]
        Vertex[Vertex Inference]
        Gemini[Gemini]
        Train[Training]
    end
    
    subgraph Monitor["Monitor"]
        Eval[LLM Eval]
        Dashboard[Dashboard]
    end
    
    PACS --> Transfer
    Transfer --> GCS1
    GCS1 --> Dataflow
    Dataflow --> Extract
    Extract --> BQ1
    BQ1 --> Dataplex
    Dataplex --> Unified
    Unified --> Vertex
    Unified --> Gemini
    Unified --> Train
    Train --> Vertex
    Gemini --> Eval
    Eval --> Dashboard
```

---

## 9. Component Summary

| Component | GCP Service | Purpose |
|-----------|-------------|---------|
| **5 PB storage** | Cloud Storage | Images; lifecycle tiers |
| **Image ETL** | Dataflow, Transfer Service | Ingest; metadata extract |
| **Catalog** | Dataplex, Data Catalog | Governance; lineage |
| **Metadata** | BigQuery | Unified schema; queries |
| **Inference** | Vertex AI, Vision | Image models |
| **LLM** | Gemini API | Reports; coding; Q&A |
| **LLM accuracy** | Vertex AI Evaluation, custom | Metrics; alerts |
| **Training** | Vertex AI Training | Patient+image+disease models |
| **Monitoring** | Cloud Monitoring, Logging | Latency; errors; drift |

---

## 10. Compliance & Security

| Requirement | Implementation |
|-------------|----------------|
| **PHI/PII** | DLP; de-identification; CMEK |
| **HIPAA** | BAA; audit logs; access controls |
| **Retention** | Lifecycle rules; BigQuery TTL |
| **Access** | IAM; VPC SC; private endpoints |
