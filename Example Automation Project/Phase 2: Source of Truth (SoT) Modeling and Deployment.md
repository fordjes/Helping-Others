
## Objective

Establish a centralized, authoritative, and structured data platform to represent the current and intended state of the network. This Source of Truth (SoT) will drive configuration generation, device onboarding, and long-term automation workflows across multiple vendors.

## What is a Source of Truth (SoT)?

A **Source of Truth (SoT)** is a centralized system or platform that serves as the **single authoritative reference** for network and infrastructure data. It provides a structured model of your environment—devices, interfaces, IPs, VLANs, and more—that other systems and processes rely on to make accurate decisions and enforce consistency.

### A SoT **is**:

* A **central repository** of device and configuration metadata
* A **structured database** that defines your network's current and desired state
* A **source for generating configurations** using templating (Jinja2, etc.)
* A **governance layer** that supports change control and auditing
* A **reference for automation tools** (e.g., Ansible, Terraform, custom scripts)

### A SoT **is not**:

* A real-time telemetry system (that would be observability platforms like Grafana or Splunk)
* A configuration backup tool (though it may reference current config states)
* A monitoring or SNMP polling system
* A replacement for all IT systems—it should integrate with other authoritative sources

## Bridging the Gap: From Spreadsheets to SoT

For many teams, network documentation and configuration data live in **spreadsheets, tribal knowledge, emails**, and handwritten notes. These methods are common but present significant risks:

* Data is **scattered and inconsistent**
* No central change history or version control
* Difficult to share and update collaboratively
* Not machine-readable or suitable for automation

### Common Spreadsheet-Based Approaches

| Spreadsheet Tab     | Purpose                            | Limitations                              |
| ------------------- | ---------------------------------- | ---------------------------------------- |
| `Device List`       | Hostnames, IPs, vendor, site       | Manual updates lead to stale data        |
| `IP Plan`           | Subnets, VLANs, allocations        | Lacks validation or hierarchy awareness  |
| `Interface Map`     | Port-to-port cabling, descriptions | Hard to update during moves/adds/changes |
| `Routing Protocols` | OSPF/BGP configs per site          | No link to interface/IP definitions      |

> ⚠️ These spreadsheets often become **semi-authoritative**, yet no one is fully confident in them. In meetings, engineers will often say, "Let me double-check the config on the device." That’s a red flag for data drift.

### What the SoT Changes

By transitioning this spreadsheet-based structure into a proper SoT platform like Nautobot or NetBox, teams:

* Convert tribal and spreadsheet knowledge into structured, queryable data
* Gain **role-based access control**, version control, and validation
* Enable **machine-readable data models** that power automation
* Break down silos between teams (e.g., Network, Security, DNS, Systems)

**Example Transition Mapping:**

| Spreadsheet Column | SoT Field (Nautobot/NetBox)        |
| ------------------ | ---------------------------------- |
| `Hostname`         | Device -> Name                     |
| `Loopback0 IP`     | Interface -> IP Address (Primary)  |
| `Site`             | Site model -> Region + Location    |
| `Interface VLAN`   | Interface -> Tagged/Untagged VLANs |
| `OSPF Area`        | VRF/Role + Custom Field            |

> 📘 **Beginner Advice**: Start by loading just the `Device List` and `IP Plan`. Use these to validate your process and build confidence in the platform before importing more complex relationships.

> 🚀 **Experienced Teams**: Tie your SoT data to Git workflows and CI/CD pipelines, replacing error-prone spreadsheets with dynamic, validated, and versioned infrastructure data.

## How SoT Integrates with Other Authoritative Systems

In large enterprises, you already have **data scattered across various systems** that act as authoritative sources for specific domains:

| System               | Domain Authority            | Role in SoT Integration                                 |
| -------------------- | --------------------------- | ------------------------------------------------------- |
| **Infoblox**         | IPAM/DNS                    | Feed IP address plans, subnet allocations, DNS mappings |
| **ServiceNow**       | Asset/CMDB                  | Feed device metadata, ownership, site location          |
| **Orion/SolarWinds** | Interface state, monitoring | Inform SoT about live interfaces and link statuses      |
| **Spreadsheets**     | Tribal or legacy knowledge  | Often manually imported into SoT as a starting point    |

These systems can be **sources of truth for specific data domains** (e.g., IPAM or asset inventory), but the goal in Phase 2 is to **centralize and normalize** that data into a unified model that automation tools can reliably consume.

> 🔄 In some organizations, the SoT becomes the master, and tools like Infoblox or ServiceNow sync from it.
>
> 🔗 In others, the SoT is a federated consumer—pulling validated data from trusted systems and assembling it into an automation-ready model.

> 🧠 **Teaching Tip**: Imagine building a house. Each contractor (electrician, plumber, architect) has their own blueprint—but the general contractor keeps the master plan. Your SoT is that master blueprint for the network.

---

## Goals

* Select and deploy a suitable SoT platform (NetBox, Nautobot, Infrahub)
* Ingest validated inventory and configuration data from Phase 1
* Normalize and enrich data to fit the SoT data model
* Define fields and relationships to support configuration generation
* Validate SoT integrity through proof-of-concept config rendering

> 🔍 **Tip for Beginners**: Think of the SoT as the brain of the automation system—it should always reflect the desired state of your network.

> 💡 **For Experienced Practitioners**: Consider using GitOps integrations, CI pipelines, or plugin frameworks to enhance the SoT’s responsiveness and governance.

---

## Tool Selection and Evaluation

| Platform | Strengths                                              | Limitations                                  |
| -------- | ------------------------------------------------------ | -------------------------------------------- |
| NetBox   | Mature, rich DCIM and IPAM, large community            | Less extensible for workflow orchestration   |
| Nautobot | Powerful plugin/job framework, GraphQL, REST APIs      | Slightly more complex to deploy and maintain |
| Infrahub | Git-native, schema-defined, great for IaC-driven teams | Minimal UI; ideal for Git-centric workflows  |

**Selection Criteria:**

* Existing team skillset (Python/Django, Git, REST APIs)
* Integration needs (Ansible, Terraform, Jenkins)
* Source format preference (GUI vs GitOps)
* Licensing, support model (open-source vs enterprise options)

---

## Activities

### 1. Platform Installation and Configuration

**Purpose:** Stand up the SoT platform and secure access.

**Steps:**

* Install SoT on-prem or in a containerized environment (Docker/Kubernetes)
* Configure authentication (local, LDAP, SSO)
* Secure API access and web front-end

**Example (Nautobot with Docker):**

```bash
git clone https://github.com/nautobot/nautobot-docker-compose
cd nautobot-docker-compose
make start
```

**Deliverables:**

* Working SoT instance accessible via web and API
* Admin credentials and initial config backup

---

### 2. Data Model Planning and Mapping

**Purpose:** Align existing inventory and configuration data with the SoT schema.

**Core Objects to Map:**

* Sites and Regions
* Devices (hostname, model, role, serial)
* Interfaces and LAGs
* IP addresses and subnets
* VLANs and VRFs
* Platform metadata (vendor, OS, config style)

**Tips:**

* Use data dictionaries or spreadsheets to track field mappings
* Identify required vs optional fields per object type

**Advanced Fields (for later phases):**

* Custom fields for API endpoints, login methods, automation tags
* Relationships (e.g., site to device, device to interface, VLAN to L2 domain)

---

### 3. Data Ingestion and Validation

**Purpose:** Load inventory and config-derived metadata into the SoT.

**Steps:**

* Clean Phase 1 outputs (YAML, CSV, JSON) to match SoT import formats
* Use built-in import scripts or APIs (Python or Ansible)
* Validate accuracy of loaded data against real devices

**Example (NetBox CSV import):**

* Devices: hostname, role, site, model, serial
* Interfaces: device name, interface name, type, IP address

**Deliverables:**

* Populated SoT with validated inventory for pilot site or region
* Documented import method (manual, script, plugin)

---

### 4. Define Automation-Centric Data Models

**Purpose:** Extend the SoT schema to support templated configuration generation.

**Fields to Include:**

* Loopback IPs
* OSPF/BGP process IDs
* Interface descriptions and roles
* Tagged/unTagged VLANs
* Authentication/AAA groups

**How to Use:**

* Jinja2 templates reference SoT data objects
* Python scripts or Ansible jobs query SoT and populate variables

**Example:**

```jinja
interface {{ interface.name }}
 description {{ interface.description }}
 ip address {{ interface.ip }}
```

**Deliverables:**

* Jinja2 templates with mapped variables
* Documentation of required data fields for each automation task

---

### 5. Proof-of-Concept (PoC) Rendering and Feedback

**Purpose:** Validate that the SoT can be used to drive network config generation.

**Steps:**

* Select 2–3 representative devices (different vendors/sites)
* Render configuration using Jinja2 + Python or Ansible
* Compare output to device baseline configs (Phase 1)
* Review with operations/engineering teams

**Deliverables:**

* Sample rendered config per device
* Feedback summary and improvement plan

---

## Tools & Ecosystem

| Purpose            | Tool/Framework                    |
| ------------------ | --------------------------------- |
| SoT Platform       | NetBox, Nautobot, Infrahub        |
| Data Ingestion     | Python, Ansible, CSV Import       |
| Validation & Diff  | PyATS, Git diff, VS Code, Batfish |
| Config Generation  | Jinja2, Python scripts, Ansible   |
| Visualization/Docs | draw\.io, Mermaid, Confluence     |

---

## OKRs & KPIs

### OKRs

| Objective                | Key Results                                         |
| ------------------------ | --------------------------------------------------- |
| Deploy functioning SoT   | SoT deployed and populated with 100% pilot data     |
| Normalize inventory      | All imported devices match live topology attributes |
| Enable config generation | Configs rendered from SoT for 3+ vendor templates   |

### KPIs

| Metric                     | Target                            |
| -------------------------- | --------------------------------- |
| Inventory match accuracy   | >95% match to known device data   |
| Data field completeness    | >90% of required fields populated |
| Config render success rate | 100% render without syntax errors |

---

## Risks and Mitigations

| Risk                 | Mitigation                                                 |
| -------------------- | ---------------------------------------------------------- |
| Data inconsistencies | Run pre-import validation scripts or manual QA             |
| Schema misalignment  | Extend SoT schema or create custom fields                  |
| Skill gaps in team   | Pair junior engineers with senior mentors or run workshops |

---

## Deliverables

* Fully deployed SoT platform
* Populated device, interface, IP, and VLAN data
* Jinja2 config templates linked to SoT model
* PoC rendered configurations and test results
* Documentation of SoT field mapping and ingestion logic

---

## Next Steps

* Expand SoT data across all sites and device types
* Integrate SoT with automation tools (Phase 3)
* Use SoT to validate and propose network changes before deployment
* Begin version-controlling SoT changes for audit trail and CI/CD workflows
