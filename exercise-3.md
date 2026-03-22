# Exercise 3 — Design Constraints

## 3a) Constraints from Connecting to Medical Devices

Connecting to a CE-marked medical device like Aletta introduces several regulatory and technical constraints.

### Regulatory Constraints

1. **EU Medical Device Regulation**  
   The cloud infrastructure that connects to Aletta® may itself be classified as part of the medical device system. Any software that influences device behavior, such as OTA updates or configuration changes, must follow a validated software development lifecycle. This means changes to cloud components that interact with the device cannot be deployed casually; they require documented risk analysis, verification, and validation before release.

2. **Post-Market Surveillance (PMS)**  
   MDR mandates that manufacturers continuously collect and analyze performance and safety data from devices in the field. The cloud infrastructure must support structured collection, retention, and reporting of this data. Incident detection and reporting timelines are legally defined, and serious incidents must be reported to competent authorities within strict timeframes.

3. **Data Protection (GDPR)**  
   Although Aletta primarily collects operational data, some data may be linked to individual procedures or patients, even indirectly. GDPR requires explicit data processing agreements with hospitals, data minimization, defined retention periods, and support for data subject rights where applicable. In practice, those rights must be balanced against medical, legal, and quality-system retention obligations.

4. **Audit Trail and Traceability**  
   Every action on or related to a medical device must be traceable. The cloud must maintain immutable audit logs of all device interactions, including configuration changes, software updates, remote commands, and data access. These logs must be retained for the lifetime of the device plus the applicable regulatory retention period, which under MDR is typically measured in many years rather than months.

### Technical Constraints

6. **Software Update Validation**  
   OTA updates to a medical device cannot be a “push and hope” process. Each update must be validated, the device must verify the update’s integrity and authenticity through code signing, and there must be a reliable rollback mechanism if an update fails. Update deployment should also be gradual, never to all devices simultaneously.

7. **Network Isolation**  
   Hospital IT departments have strict network security policies. Aletta devices will likely be placed on isolated network segments, such as VLANs, with limited outbound access. The cloud architecture must work within these constraints through outbound-only connections, restricted ports and protocols, and compatibility with hospital proxy and firewall configurations.

8. **Strict Safety / Cloud Boundary**  
   The cloud monitors, reports, and orchestrates updates, but it must never be in the safety-critical execution path. Venipuncture execution, needle control, safety interlocks, and emergency stop logic run entirely on the device with zero cloud dependency. If the cloud is unreachable, the device must continue to perform blood draws safely using its local software. This boundary is non-negotiable: no cloud latency, outage, or misconfiguration may ever affect patient safety. The cloud infrastructure design must enforce this by treating all cloud-to-device interactions as advisory, such as desired state or configuration suggestions, rather than imperative commands that control real-time device behavior.

---

## 3b) Intermittent Connectivity: Impact on Design

### Impact on Cloud Infrastructure Design

When the device is not always connected, the cloud cannot assume real-time data availability or instant command delivery. This fundamentally shapes the architecture:

- **Eventual Consistency Model**: The cloud must tolerate gaps in telemetry data and process data as it arrives, not as it is generated. Dashboards and analytics must clearly indicate data freshness and the last-seen time for each device.
- **Asynchronous Command Delivery**: Cloud-to-device interactions must be designed for delayed delivery. Azure IoT Hub offers two mechanisms, and the choice depends on intent:
  - **Device twin desired properties** are used for durable desired state, such as configuration changes or target software version. The twin persists the desired state indefinitely until the device comes online and reconciles.
  - **Cloud-to-device (C2D) messages** are used for short-lived, one-time commands, such as “upload logs for procedure X now.” These expire after a configurable TTL if the device does not connect in time.  
    Mixing these up would cause either lost commands or stale state, so the distinction matters.
- **Idempotent Operations**: Since messages may be delivered more than once, for example if a device reconnects mid-transmission, all device-side operations must be idempotent. Applying the same command twice must not cause errors or unintended side effects.
- **Health Monitoring Adaptation**: A disconnected device is not necessarily a failed device. Alerting rules must distinguish between “device offline” and “device unexpectedly unreachable.” This requires understanding expected connectivity patterns per device and site.

### Additional Software Needed on the Device

To support intermittent connectivity, the device needs the following edge software components:

| Component | Purpose |
|---|---|
| **Local Data Store** | A durable, encrypted on-device database or file system that buffers all telemetry, events, and procedure data generated while offline. Must handle several days’ worth of data without loss. |
| **Store-and-Forward Agent** | Queues outbound messages and retransmits them when connectivity is restored. Handles message ordering, deduplication, and delivery confirmation. Azure IoT Edge’s built-in store-and-forward capability provides this. |
| **Sync Manager** | Coordinates upload of buffered data when the connection is available. Prioritizes: (1) critical alerts/errors, (2) procedure summaries and telemetry, (3) large raw data files. Respects available bandwidth. |
| **Local Configuration Cache** | Stores the last-known-good device configuration locally so the device operates correctly even if it cannot reach the cloud for an extended period. |
| **OTA Update Agent** | Downloads, verifies via signature check, and applies software updates. Must support resumable downloads and validated rollback if an update fails after installation. |
| **Health Self-Monitor** | Runs locally to monitor device health and logs anomalies even while offline. Flags issues for immediate cloud upload when connectivity resumes. |

**Recommended Implementation:** Azure IoT Edge runtime deployed on the device’s embedded Linux system. IoT Edge provides a containerized runtime with built-in store-and-forward, module management, and device-twin synchronization, reducing the amount of custom software Vitestro must build and maintain.

---

## 3c) Bandwidth Constraint: Gigabytes of Data per Procedure

### The Problem

Each Aletta blood draw generates several gigabytes of data, including high-resolution images, 3D vein models, real-time motor telemetry, and complete service logs. Typical hospital internet connections cannot handle transmitting all of this data, especially when multiple devices are operating concurrently.

### Strategy: Tiered Data with Edge Processing

The solution is to categorize data by priority and process it at the edge so that only essential information is transmitted in real time, while full raw data is handled through alternative channels.

#### Tier 1 — Real-Time (transmit immediately, small footprint)

- Device health status and heartbeat
- Procedure outcome summary (success/failure, key metrics)
- Error events and alerts
- Aggregated motor performance statistics (minimum, maximum, mean — not raw time series)

**Size:** Easily fits within any connection.

#### Tier 2 — Near Real-Time (transmit within hours, compressed)

- Compressed procedure logs
- Summarized sensor data (downsampled time series)
- Thumbnail images
- Key 3D model features extracted at the edge

**Size:** Transmitted during low-usage periods.

#### Tier 3 — Batch / On-Demand (transmit overnight or on request)

- Full-resolution images and 3D models
- Complete raw motor telemetry
- Detailed debug logs

**Size:** Several GB per procedure.

Options for transfer:

- **Scheduled batch upload**: Automatically sync during off-peak hours, such as nights and weekends, when hospital bandwidth is available. The sync manager throttles uploads to avoid impacting hospital network performance.
- **On-demand retrieval**: Vitestro support can request specific raw data from a specific procedure when debugging an issue. Only the needed data is transmitted.
- **Physical data transfer**: For exceptional cases where network transfer is impractical, data can be exported to encrypted portable storage and shipped under an approved chain-of-custody process. This should remain the exception, not the default operating model.

#### Edge Processing to Reduce Data Volume

The edge agent performs several processing steps before transmission:

- **Compression**: Apply lossless compression such as zstd or gzip to logs and telemetry. Use lossy compression, such as reduced JPEG quality, for images where full fidelity is not required for monitoring purposes.
- **Deduplication**: Avoid sending data that has not changed, such as static configuration data or identical error logs.
- **Feature Extraction**: Instead of transmitting raw 3D models, extract key measurements and metrics on the device and send only those. The raw model stays on the device for on-demand retrieval.
- **Retention Policy**: Automatically purge raw data from the device after a defined retention period, such as 30 to 90 days, or after confirmation that the data has been successfully uploaded to the cloud or is no longer needed.

#### Cloud-Side Design Implications

- **Azure Blob Storage with cool/archive tiers**: Raw data that is infrequently accessed is stored in cool or archive tiers to minimize cost.
- **Resumable uploads**: The upload protocol supports resuming interrupted transfers. Azure Blob Storage supports block blob uploads that can be resumed.
- **Bandwidth-aware scheduling**: The cloud-side ingest service monitors per-device upload rates and adjusts scheduling to avoid saturating the hospital’s connection.
- **Data catalog**: All data, whether already transmitted or still on-device, is cataloged in a central registry so Vitestro engineers can see what exists and request specific files when needed.

#### Purpose-Based Data Retention

Not all data has the same lifecycle. Retention should be driven by purpose, not just age. A practical example policy could look like this:

| Data Category | Default Upload | Retention (device) | Retention (cloud) | Trigger for Deletion |
|---|---|---|---|---|
| Procedure summaries & health metrics (Tier 1) | Automatic, immediate | 7 days | 5+ years (MDR PMS) | Regulatory retention period expires |
| Compressed logs & thumbnails (Tier 2) | Automatic, delayed | 30 days | 2 years | Cloud upload confirmed + retention period |
| Raw images & 3D models (Tier 3) | **Not uploaded by default** | 90 days | Uploaded only on demand | Support case closed or 90-day device purge, whichever is later |
| Full motor telemetry (Tier 3) | **Not uploaded by default** | 60 days | Uploaded only on demand | Same as above |
| Debug/crash logs | Automatic on error event | 30 days | 1 year | Cloud upload confirmed + retention period |

Key design decisions:

- Tier 3 raw data is **not uploaded by default**. It remains on the device and is pulled to the cloud only when a support case or R&D request requires it. This is the biggest bandwidth saving.
- Once uploaded to the cloud, data moves through storage tiers automatically (`hot → cool → archive`) based on age.
- Deletion from the device happens only after either:
  - confirmed cloud upload, or
  - the on-device retention window expires,  
    whichever provides the stronger guarantee that the data is not lost prematurely.

The exact retention periods should be finalized with Vitestro’s quality, regulatory, legal, and security teams.
