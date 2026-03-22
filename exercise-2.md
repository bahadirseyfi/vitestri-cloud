# Exercise 2

## Overview

The architecture supports monitoring and managing multiple Aletta devices deployed across multiple hospitals, in multiple countries, over a wide geographical region. It is designed around three layers: **Edge (Device)**, **Cloud Platform**, and **Operations & Observability**.

> See `architecture-overview.png` or `README.md` for the visual architecture diagram.

---

## Platform Choice: Microsoft Azure

### Why Azure?

| Criterion | Azure Advantage |
|---|---|
| **IoT Ecosystem** | Azure IoT Hub + IoT Edge provide device management, bi-directional communication, OTA updates, and store-and-forward capabilities that are purpose-built for this use case. |
| **Healthcare Compliance** | Azure holds several certifications, reducing the effort needed to meet Vitestro’s regulatory obligations. |
| **European Presence** | Regions in West Europe (Netherlands), North Europe (Ireland), and other EU regions support GDPR data residency requirements. |
| **US Expansion** | US East and West regions are readily available when Vitestro enters the US market. |
| **Edge Computing** | Azure IoT Edge allows deployment of containerized modules directly on the device for local processing, offline operation, and data filtering. |
| **Enterprise Integration** | Many hospitals already use Microsoft ecosystems, easing integration and security alignment. |

### Why Azure over AWS?

I should note upfront that I chose Azure for this assignment because it is the platform I work with daily and know most deeply. Vitestro uses AWS, and I am familiar with the AWS service ecosystem as well but I believe an assignment like this benefits from designing with the tools I can reason about most confidently, rather than producing a surface-level AWS architecture.

Beyond personal familiarity, Azure does have genuine strengths for this use case:

1. **IoT Edge maturity**  
   Azure IoT Edge offers a more tightly integrated store-and-forward runtime with native device twin synchronization, which directly addresses the intermittent connectivity constraint described in Exercise 3b.

2. **Healthcare ecosystem in Europe**  
   Many European hospitals operate within Microsoft ecosystems such as Active Directory and Microsoft 365. Azure Entra ID integration simplifies identity federation with hospital IT departments, reducing friction during device onboarding at new customer sites.

3. **EU data residency**  
   Azure’s Netherlands region (West Europe) aligns directly with Vitestro’s headquarters in Utrecht, minimizing latency and simplifying GDPR compliance. AWS has EU regions in Ireland and Frankfurt, but not in the Netherlands specifically.

That said, portability is still worth preserving. Application logic is containerized, infrastructure is defined in Terraform rather than a cloud-specific IaC language, and the data tier uses standard PostgreSQL. This does not make a provider switch free, but it keeps the migration path realistic if business requirements change.

---

## Architecture Layers

### Edge (Device & Hospital Site)

Each Aletta device runs an **edge agent** responsible for:

- **Local data storage**: Buffering procedure data such as images, 3D models, motor telemetry, and logs on the device’s local storage.
- **Data filtering & aggregation**: Extracting key performance metrics and summaries locally, reducing the data volume that needs to be transmitted to the cloud.
- **Store-and-forward**: Queuing telemetry and events when cloud connectivity is unavailable, then automatically syncing when the connection is restored.
- **OTA update client**: Receiving, validating, and applying software updates pushed from the cloud. Updates are verified with digital signatures before installation.
- **Secure communication**: Connecting to Azure IoT Hub using MQTT or AMQP over TLS 1.2+, authenticated with per-device X.509 certificates.

**Technology:** Azure IoT Edge runtime with custom containerized modules deployed on the device’s embedded Linux system.

**Hospital network:** The device connects through the hospital’s network. A firewall rule allows only outbound MQTT(S) and AMQP(S) traffic to Azure IoT Hub endpoints, and no inbound ports are opened.

---

### Key Components

#### Device Connectivity & Management

- **Azure IoT Hub**  
  Central message broker for device-to-cloud (D2C) telemetry and cloud-to-device (C2D) commands. It manages device twins (desired/reported state) and direct methods for real-time device interaction.

- **Device Provisioning Service (DPS)**  
  Enables zero-touch, secure provisioning of new Aletta® devices. When a device is manufactured, it receives X.509 certificates and is automatically registered on first connection.

- **Device Update for IoT Hub**  
  Manages OTA software update campaigns by targeting specific device groups, rolling out updates gradually, and tracking update compliance across the fleet.

#### Data Ingestion & Processing

- **Event Hub**  
  Receives high-throughput telemetry streams routed from IoT Hub. It is chosen over Event Grid for the primary data path because the telemetry flow is continuous and high-volume. Event Grid could be added later for specific event-driven triggers, such as device error webhooks.

- **Azure Functions**  
  Processes incoming telemetry by calculating aggregates, detecting anomalies, triggering alerts, and transforming data before storage. It is chosen over Stream Analytics because Functions provide greater flexibility for custom logic at Vitestro’s current scale. Stream Analytics would become relevant once the fleet grows enough to justify dedicated stream processing infrastructure.

- **Azure Blob Storage**  
  Stores large raw data files such as images, 3D models, and detailed logs uploaded from devices via batch sync. Data is organized using a hierarchical namespace:

  ```text
  /{customer}/{hospital}/{device_id}/{date}/{procedure_id}/
  ```

- **Azure Database for PostgreSQL**  
  Stores structured telemetry, device metadata, procedure summaries, and maintenance records. PostgreSQL is selected for its relational query strength, mature ecosystem, and broad team familiarity. A globally distributed database such as Cosmos DB is not justified at this stage and would only be considered if Vitestro later needs sub-10 ms reads across both the US and EU.

#### Operations & Observability

- **Azure Monitor + Log Analytics**  
  Provides centralized monitoring for cloud resources and device health metrics. Diagnostic logs from IoT Hub, storage, databases, and network components flow into a shared Log Analytics workspace for fleet dashboards, alerting, and audit retention.

- **Application Insights**  
  Monitors the operations portal and customer-facing APIs running on App Service, including request latency, exceptions, dependency failures, and end-user performance. This helps distinguish device-side incidents from backend or portal issues.

- **App Service**  
  Hosts the Vitestro operations portal and customer-facing APIs. Vitestro engineers and hospital administrators access device data through this web application, with authorization scoped per tenant.

#### Security

- **Azure Key Vault**  
  Manages encryption keys, device certificates, and secrets. Certificate rotation is automated.

- **Microsoft Entra ID (Azure AD)**  
  Serves as the identity provider for all human users, including Vitestro engineers and support staff. It enforces MFA, RBAC, and conditional access policies.

- **Azure Policy**  
  Enforces baseline controls such as approved regions, required diagnostic settings, minimum TLS versions, and private access patterns where applicable.

- **Microsoft Defender for Cloud**  
  Continuously assesses misconfigurations, vulnerability exposure, and compliance posture across the environment.

---

## Multi-Tenancy & Multi-Region

### Multi-Tenancy (Customers & Hospital Sites)

Per-customer infrastructure isolation would be introduced only if a customer’s regulatory or contractual requirements demand it. Until then, logical isolation provides a more efficient balance between security, manageability, and cost.

Logical isolation is enforced at three levels:

1. **Device layer**  
   Each device is tagged in IoT Hub with its `customer_id` and `site_id`. Device groups restrict which devices a query or update campaign can target. A support engineer at Vitestro can interact only with devices assigned to their authorized customer scope.

2. **Data layer**  
   Blob Storage uses a hierarchical path structure:

   ```text
   /{customer_id}/{site_id}/{device_id}/...
   ```

   Azure RBAC and SAS tokens are scoped per customer prefix. PostgreSQL uses a `customer_id` column together with row-level security (RLS) policies, ensuring that queries never cross customer boundaries.

3. **Portal layer**  
   The customer portal authenticates hospital administrators via Microsoft Entra External ID or federation with the hospital’s own identity provider. Each logged-in user can see only their organization’s devices and data. Vitestro internal users are managed through Entra ID with role-based access, such as `support engineer – OLVG` or `R&D – all devices`.

### Multi-Region

The deployment follows an **active-passive** model:

- **West Europe (Netherlands) — Active**  
  All device traffic, data ingestion, and processing runs here. This is the primary region, aligned with Vitestro’s headquarters and current customer base.

- **North Europe (Ireland) — Passive / DR**  
  Storage is geo-replicated to this region. In a regional outage, IoT Hub can fail over to a pre-provisioned secondary hub in North Europe, and DNS routing switches device traffic accordingly.  
  **RTO target:** < 4 hours  
  **RPO target:** near-zero for replicated telemetry, and minutes for in-flight processing.

- **US East — Future**  
  A separate, independent deployment with its own IoT Hub, storage, and database will be provisioned when Vitestro begins US commercialization after FDA approval. EU devices never route through the US stack, and US devices never route through the EU stack, ensuring data residency by design.

Each regional stack is deployed from the same Terraform codebase using region-specific variable files, ensuring consistency across environments.

---

## Security Aspects

| Security Aspect | Implementation |
|---|---|
| **Device Identity** | Per-device X.509 certificates issued during manufacturing and registered via DPS. No shared keys. |
| **Transport Encryption** | TLS 1.2+ for all device-to-cloud and cloud-to-cloud communication. |
| **Data Encryption at Rest** | Encryption for all storage accounts, databases, and backups, with customer-managed keys via Key Vault where required. |
| **Network Security** | Backend PaaS services are integrated with a virtual network and accessed through private endpoints where supported. Field devices connect outbound-only over TLS through hospital firewalls; no inbound connectivity to devices is required. |
| **Access Control** | Entra ID with MFA for all human access. RBAC follows least-privilege principles. No shared accounts. |
| **Compliance Guardrails** | Azure Policy enforces approved regions, minimum TLS versions, mandatory diagnostic settings, and private access patterns where applicable. |
| **Audit Logging** | All access and configuration changes are logged to immutable audit trails through Azure Monitor and Log Analytics, and retained according to regulatory requirements. |
| **Security Posture Management** | Microsoft Defender for Cloud continuously assesses misconfigurations, surfaces vulnerabilities, and supports compliance reporting. |
| **OTA Update Security** | Updates are code-signed. Devices verify digital signatures before applying them. Rollback is supported if an update fails. |
| **Vulnerability Management** | Regular security scanning of infrastructure and container images. Patch management SLAs are defined. |
| **Incident Response** | A documented incident response plan covers breach notification timelines, including 72 hours for GDPR and 60 days for HIPAA where applicable. Integration with Azure Sentinel supports threat detection. |
| **Data Residency** | Region-pinned deployments ensure data does not leave the applicable legal jurisdiction. |

---
