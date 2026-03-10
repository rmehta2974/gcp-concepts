# Centralized Logging & Cross-Project Service Accounts

## Overview

Centralized logging collects logs from all projects into one place. Cross-project service accounts enable workloads in project A to act in project B (e.g., write logs, access resources).

---

## Centralized Logging Architecture

```mermaid
flowchart TB
    subgraph Projects["Source Projects"]
        P1[Project A]
        P2[Project B]
        P3[Project C]
    end
    
    subgraph Sinks["Log Sinks"]
        S1[Org Sink]
    end
    
    subgraph Dest["Destination"]
        LP[Logging Project]
        BQ[BigQuery]
        Pub[Pub/Sub]
    end
    
    P1 --> S1
    P2 --> S1
    P3 --> S1
    S1 --> LP
    S1 --> BQ
    S1 --> Pub
```

---

## How Centralized Logging Works

1. **Log Sink** (org or folder level): Routes logs matching a filter to a destination
2. **Destination**: Logging project, BigQuery dataset, or Pub/Sub topic
3. **IAM**: Sink's writer identity needs `logging.sinks.create` and destination write permissions

### Sink Configuration Example

```hcl
resource "google_logging_folder_sink" "central" {
  name        = "central-logs-sink"
  folder      = "folders/123456"
  destination = "logging.googleapis.com/projects/prj-logging/logs/central"
  filter      = "NOT (protoPayload.serviceName = \"storage.googleapis.com\" AND protoPayload.methodName = \"storage.objects.get\")"
}
```

---

## Cross-Project Service Account Access

```mermaid
flowchart LR
    subgraph ProjA["Project A (Workload)"]
        SA[Service Account A]
        Workload[GKE / Cloud Run]
    end
    
    subgraph ProjB["Project B (Central)"]
        Logging[Logging]
        BQ[BigQuery]
    end
    
    Workload --> SA
    SA -->|"iam.serviceAccountTokenCreator"| ProjB
```

**Pattern**: Service Account in Project A is granted roles in Project B (e.g., `roles/logging.logWriter`, `roles/bigquery.dataEditor`).

---

## Central Project → Other Projects

| Use Case | Central Project | Other Projects |
|----------|-----------------|----------------|
| **Logging** | Logging project receives org sink | Source projects (no extra config) |
| **Security** | Security project runs SCC, audit | All projects send logs |
| **CI/CD** | CICD project has build SAs | Workload projects grant `actAs` |

---

## Service Account from Central to Workload Projects

**Scenario**: Central platform team's SA needs to deploy or manage resources in workload projects.

1. Create SA in central project (or org-level)
2. Grant SA roles in workload projects (e.g., `roles/container.developer`)
3. Use Workload Identity Federation for CI/CD (no keys)

---

## Diagram: Full Logging + IAM Flow

```mermaid
flowchart TB
    subgraph Org["Organization"]
        subgraph Workloads["Workload Projects"]
            W1[App A]
            W2[App B]
        end
        
        subgraph Central["Central Projects"]
            Log[Logging]
            Sec[Security]
        end
    end
    
    W1 -->|Log Sink| Log
    W2 -->|Log Sink| Log
    Log -->|Findings| Sec
    Sec -->|IAM grants| W1
    Sec -->|IAM grants| W2
```

---

## Best Practices

- **Logging project**: Dedicated project; no workloads
- **Retention**: Set per log type; compliance-driven
- **Exclusions**: Exclude high-volume, low-value logs (e.g., health checks)
- **Access**: Central security team has read; workload teams have limited access to their logs
