# Bahadir Seyfi

# Exercise 1 — Cloud Strategy: Guiding Principles

Below are six guiding principles for successful cloud adoption at Vitestro.

---

## 1. Single Cloud, Multi-Region

Adopt a single cloud provider (**Azure**) deployed across multiple regions to balance operational simplicity with geographic reach.

**Rationale:**  
Vitestro is a ~70-person company entering its commercial phase. Operating multiple cloud providers would multiply operational complexity, require broader skill sets, increase the compliance surface, and duplicate baseline platform costs without clear benefit at this stage.

A single provider with multi-region deployments satisfies data residency requirements while keeping the team focused. Instead of paying the multi-cloud complexity premium up front, Vitestro should mitigate provider concentration risk through:

- multi-region disaster recovery,
- tested backups,
- documented data export procedures, and
- portable infrastructure definitions.

---

## 2. Managed Services First, Kubernetes Where Justified

Prefer managed services over self-managed Kubernetes clusters to reduce operational overhead and let the team focus on product development.

**Rationale:**  
Running Kubernetes demands dedicated platform engineering capacity that is better spent on Vitestro's core product. Managed services such as:

- Azure IoT Hub,
- Azure Functions, and
- Azure Database for PostgreSQL

provide built-in scaling and patching, and reduce the effort needed to meet compliance objectives, though regulatory responsibility remains with Vitestro.

Kubernetes (**AKS**) should be introduced only when specific workloads clearly justify the added complexity.

---

## 3. Security and Compliance by Design

Embed security, privacy, and regulatory compliance into every architectural decision from the start, not as an afterthought.

**Rationale:**  
Encryption at rest and in transit, RBAC, comprehensive audit logging, and data residency enforcement are baseline requirements. The cloud infrastructure must also support MDR post-market surveillance obligations, including:

- device traceability,
- performance reporting, and
- incident logging.

---

## 4. Edge-First Data Processing

Process data at the device edge wherever possible, transmitting only essential data to the cloud to handle bandwidth constraints and connectivity gaps.

**Rationale:**  
Each Aletta procedure generates several gigabytes of data, including images, 3D models, motor telemetry, and logs. Hospital internet connections cannot transfer all of this in real time.

The device should therefore perform local aggregation, filtering, and compression, sending prioritized summaries to the cloud and batch-syncing raw data during off-peak hours. This also ensures devices remain functional during network outages through local store-and-forward mechanisms.

---

## 5. Infrastructure as Code (IaC) and Automation

Define all infrastructure as code, automate deployments through CI/CD pipelines, and make every environment reproducible and auditable.

**Rationale:**  
In a regulated environment, auditability and reproducibility are critical. Every infrastructure change must be:

1. version-controlled,
2. peer-reviewed, and
3. deployed through automated pipelines.

**Terraform** is preferred over a cloud-specific IaC language such as **Bicep** because it preserves more portability if Vitestro ever needs to add a second cloud or change providers.

This approach eliminates configuration drift, provides a full audit trail, and enables consistent provisioning of new environments or regional deployments as Vitestro expands geographically.

---

## 6. Observability from Day One

Instrument all systems, devices, data pipelines, and cloud services for monitoring, alerting, and diagnostics from the initial deployment.

**Rationale:**  
For a medical device company, rapid detection and resolution of issues is directly tied to patient safety and customer trust. The following capabilities must be part of the core infrastructure from day one:

- centralized logging,
- real-time device health dashboards,
- anomaly detection, and
- proactive alerting.

Observability data also feeds into Vitestro's R&D loop and supports MDR post-market surveillance reporting.
