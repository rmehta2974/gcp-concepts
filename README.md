# GCP Best Practices & Reference Architecture

Comprehensive documentation for Google Cloud Platform design, security, and operations—covering landing zones, GKE, networking, data, and migration.

---

## Documentation Index

### 1. Foundation & Governance

| Document | Description |
|----------|-------------|
| [01-landing-zone-best-practices.md](./01-landing-zone-best-practices.md) | Landing zone design, workflow, and governance |
| [02-folder-design.md](./02-folder-design.md) | Resource hierarchy, folder structure, clear choices |
| [03-cloud-adoption-framework.md](./03-cloud-adoption-framework.md) | CAF alignment, strategy, and readiness |

### 2. Networking

| Document | Description |
|----------|-------------|
| [04-network-design.md](./04-network-design.md) | VPC design, subnets, load balancers, choices |
| [05-connectivity-patterns.md](./05-connectivity-patterns.md) | Project-to-project, org, on-prem, cloud-to-cloud |
| [06-private-access-endpoints.md](./06-private-access-endpoints.md) | PSC, PSA, private service endpoints |

### 3. Security

| Document | Description |
|----------|-------------|
| [07-security-design-zerotrust.md](./07-security-design-zerotrust.md) | Zero trust principles, security design |
| [08-vpc-service-controls.md](./08-vpc-service-controls.md) | VPC SC, Security Command Center |
| [09-centralized-logging-iam.md](./09-centralized-logging-iam.md) | Centralized logging, cross-project service accounts |

### 4. GKE (Kubernetes)

| Document | Description |
|----------|-------------|
| [10-gke-versions-offerings.md](./10-gke-versions-offerings.md) | GKE versions, Autopilot vs Standard, regional vs zonal |
| [22-gke-detailed.md](./22-gke-detailed.md) | GKE private/public, scalability, upgrades, release channels |
| [11-gke-security-zerotrust.md](./11-gke-security-zerotrust.md) | GKE security, binary authorization, zero trust |
| [12-gke-ingress-service-mesh.md](./12-gke-ingress-service-mesh.md) | Multitenant ingress, service mesh designs |
| [13-gke-tracing-observability.md](./13-gke-tracing-observability.md) | Tracing, observability, binary authorization |

### 5. Data & Messaging

| Document | Description |
|----------|-------------|
| [14-pubsub-data-ingest.md](./14-pubsub-data-ingest.md) | Pub/Sub, data ingest practices |
| [15-databases.md](./15-databases.md) | Database design, Cloud SQL, Spanner, choices |

### 6. Migration

| Document | Description |
|----------|-------------|
| [16-vmware-migration.md](./16-vmware-migration.md) | VMware to GCP migration |
| [17-application-migration.md](./17-application-migration.md) | Application migration strategies |

### 7. Architecture Scenarios & Solutions

| Document | Description |
|----------|-------------|
| [18-architecture-scenarios-gcp-aws.md](./18-architecture-scenarios-gcp-aws.md) | Real-time, ETL, infrastructure, AI scenarios |
| [19-healthcare-imaging-ai-pipeline.md](./19-healthcare-imaging-ai-pipeline.md) | 5 PB imaging, Vertex, Gemini, patient+image+disease |
| [20-photo-processing-app-infrastructure.md](./20-photo-processing-app-infrastructure.md) | Photo app, streaming, storage |
| [21-photo-app-multicloud-secure-design.md](./21-photo-app-multicloud-secure-design.md) | Photo app multi-cloud, zero trust |

### 8. Cloud Solutions (GCP, AWS, Azure)

| Folder | Description |
|--------|-------------|
| [cloud-solutions/](./cloud-solutions/) | End-to-end solutions per cloud (prod/dev/QA, pipeline, canary, traffic control) |
| [cloud-solutions/README.md](./cloud-solutions/README.md) | Full index of all scenario solutions |

### 9. Diagrams & Decision Trees

| Document | Description |
|----------|-------------|
| [diagrams/00-master-architecture.md](./diagrams/00-master-architecture.md) | End-to-end architecture, connectivity, security |
| [diagrams/01-decision-trees.md](./diagrams/01-decision-trees.md) | Load balancer, GKE, database, folder, migration decisions |

---

## Quick Reference

- **Landing Zone**: Start with folder design → network design → security baseline
- **GKE**: Choose Autopilot vs Standard → Regional vs Zonal → Ingress/Mesh
- **Connectivity**: VPC peering → Interconnect → PSC/PSA for private access
- **Security**: Zero trust → VPC SC → Binary Authorization → Security Center

---

*Generated for GCP architecture reference. Align with your organization's Cloud Adoption Framework and compliance requirements.*
"# gcp-concepts" 
