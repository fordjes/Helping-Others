

## Objective

Establish a clear understanding of the current network infrastructure by documenting device configurations and identifying configuration inconsistencies. This foundation will guide standardization and future automation.

## Goals

* **Create an authoritative inventory of all routers and switches**
  Before any automation efforts can succeed, we must know what hardware exists, where it is located, and how it is accessed. This includes gathering device hostnames, management IP addresses, software versions, and physical locations. Junior engineers may begin with manual entry into a spreadsheet or YAML file, while advanced teams might script inventory collection via CDP/LLDP neighbors or SNMP.

* **Capture and store running configurations from all devices**
  Capturing the current state of the network ensures we have a record of all configuration elements (interfaces, routing protocols, ACLs, etc.). This serves as a reference for developing standard templates. Use commands like `show running-config` (Cisco) or `show configuration | display set` (Juniper) to output the full configuration.

* **Normalize configurations across vendors and sites**
  A normalized configuration is one that removes device-specific or volatile elements (timestamps, random port numbers) and brings common patterns into reusable blocks. This step is important for identifying inconsistencies and setting the stage for templating in automation.

* **Develop baseline configuration templates for each vendor**
  These templates will represent the "ideal" or "standard" configuration per vendor. For example, all Cisco routers in a branch office might share the same NTP, SNMP, logging, and BGP base config. These baselines help define what is considered compliant.

* **Identify configuration drift and manual process gaps**
  Drift is any deviation from the expected baseline. It may be caused by ad-hoc changes, missed updates, or reactive troubleshooting. Identifying this drift helps reveal operational risks. For example, if SNMP traps are misconfigured on 20% of devices, it could impact monitoring effectiveness. Manual workflow gaps refer to places where documentation, approval, or validation is missing—crucial to correct before automation.

> 🔍 **Tip for Beginners**: Start small. Choose one device per vendor, document it thoroughly, and use it as a model for others.

> 💡 **For Experienced Practitioners**: Consider using tools like Batfish, PyATS, or Genie to validate that configuration intent matches actual state. You may also script config scraping using Nornir or Netmiko.

## Scope

* **Device Types**: Routers and switches across all sites
* **Vendors**: Cisco (IOS, IOS-XE, NX-OS), Juniper, Arista, Aruba
* **Interfaces**: CLI only (SSH, Telnet if legacy)
* **Out of Scope**: Firewalls, wireless controllers, SD-WAN appliances

---

## Activities

### 1. Device Inventory Collection

**Purpose:** Build a comprehensive and accurate inventory of all networking equipment.

**Steps:**

* Start with existing documentation (e.g., Excel spreadsheets, CMDBs, Visio diagrams).
* Log into network management systems (NMS) or SNMP-based tools to pull discovery data.
* Walk the network using `show cdp neighbors`, `show lldp neighbors`, or interface mappings.
* Validate the data: hostname, management IP, vendor/model, OS version, physical location, rack position.

**Tools & Examples:**

* CLI: `show version`, `show inventory`, `show interfaces`
* SNMP walk tools for structured polling (e.g., snmpwalk, LibreNMS, Observium)
* Scripted discovery using Netmiko/Nornir:

```python
from netmiko import ConnectHandler

cisco_device = {
    'device_type': 'cisco_ios',
    'host': '10.1.1.1',
    'username': 'admin',
    'password': 'password',
}

net_connect = ConnectHandler(**cisco_device)
print(net_connect.send_command("show version"))
```

**Deliverable:**

* A device inventory in CSV or YAML, structured by:

  * Hostname, IP, OS, Site, Model, Serial, Mgmt access

---

### 2. Configuration Backup

**Purpose:** Capture current configurations for analysis and standardization.

**Steps:**

* Manually or programmatically SSH into each device and run the relevant command.
* Use terminal log capture, `copy tftp`, or script output to files.
* Organize configs in a Git repo by vendor > site > hostname.

**Vendor Examples:**

* Cisco IOS: `show running-config`
* Juniper: `show configuration | display set`
* Arista: `show running-config`
* Aruba: `show running-config` or via REST API (optional)

**Tips:**

* Redact credentials or sensitive data before version control.
* Create a commit message format like `Backup - R1 - 2025-05-15`.

**Deliverable:**

* Local or Git-based repository with structured configs

---

### 3. Configuration Normalization

**Purpose:** Remove inconsistent or irrelevant elements to prepare configs for comparison.

**Steps:**

* Use scripts to remove timestamps, counters, and session-specific lines.
* Format configurations for line-by-line comparison.
* Identify reusable blocks (e.g., interface template, routing policy).

**Techniques:**

* Regex or text parsing using Python
* Tools like TextFSM, TTP (Text to Parse)

**Example Python Snippet:**

```python
import re
with open('config.txt') as f:
    raw = f.read()
cleaned = re.sub(r'! Last configuration.*', '', raw)
```

**Deliverables:**

* Cleaned configuration files
* Documentation of normalization logic used

---

### 4. Identify Configuration Drift

**Purpose:** Detect where devices deviate from baseline templates.

**Steps:**

* Create gold-standard configuration templates
* Compare each normalized config against the standard
* Generate drift reports highlighting missing, extra, or changed lines

**Tooling:**

* Python with `difflib`
* Batfish or PyATS config compare
* Meld, Beyond Compare for GUI diff

**Example Concept:**

* If all switches should have `ntp server 192.0.2.1`, and 3 do not, flag that in the report.

**Deliverables:**

* Drift analysis report (CSV, Markdown, or JSON)
* Prioritized list of remediations

---

### 5. Document Manual Workflows

**Purpose:** Understand current human-driven processes before introducing automation.

**Steps:**

* Interview operations staff
* Walk through change approval and implementation
* Document:

  * How changes are proposed (ticket, email, verbal)
  * How they’re reviewed, tested, and deployed
  * Rollback procedures (if any)

**Output Format:**

* Workflow diagrams (draw\.io, Lucidchart)
* Step-by-step written procedures
* Spreadsheet or table identifying gaps and risks

**Common Gaps:**

* No change control documentation
* Inconsistent validation post-change
* Lack of version control or backup

**Deliverables:**

* Visual workflow charts
* Text-based manual SOPs
* Recommendations for standardization

---

## Tools

| Function        | Tools & Examples                     |
| --------------- | ------------------------------------ |
| Terminal Access | PuTTY, SecureCRT, mRemoteNG, ssh     |
| Scripting       | Python (Netmiko, Nornir), PowerShell |
| Text Processing | Regex, TextFSM, TTP                  |
| Version Control | Git, GitHub, GitLab                  |
| Diff/Validation | VS Code, Meld, Batfish, PyATS        |
| Documentation   | Markdown, Confluence, Notion, Word   |

---

## Phase 1 OKRs & KPIs

### OKRs

| Objective                    | Key Results                                         |
| ---------------------------- | --------------------------------------------------- |
| Inventory all devices        | 100% of routers/switches listed with IPs and models |
| Standardize configurations   | 3+ vendor templates created                         |
| Identify configuration drift | Drift reported for 90% of inventory                 |

### KPIs

| Metric                 | Target                                  |
| ---------------------- | --------------------------------------- |
| Device coverage        | >95% reachable/config-backed up         |
| Baseline creation      | 100% of vendors have 1 working template |
| Config variance logged | ≥90% of devices analyzed for drift      |

---

## Risks and Mitigations

| Risk                             | Mitigation                                              |
| -------------------------------- | ------------------------------------------------------- |
| SSH/Telnet access unavailable    | Coordinate with local teams or use console access       |
| Configuration capture incomplete | Implement scripts to automate future collection         |
| No current inventory             | Build from MAC address tables, ARP, LLDP, CDP, NMS data |
| Human error in documentation     | Introduce peer review of inventory and workflows        |

---

## Deliverables

* Device Inventory File (CSV/YAML/JSON)
* Configuration Repository (organized by site/vendor)
* Normalized Configuration Templates (per vendor)
* Configuration Drift Report
* Workflow Documentation and Gaps Log

---

## Next Steps

* Select Source of Truth platform (NetBox, Nautobot, Infrahub)
* Begin mapping inventory and baseline data into the SoT model
* Define data fields needed to support automation (hostname, platform, IPs, interfaces, VLANs, etc.)
* Review data quality for consistency across sources

---

By thoroughly completing Phase 1, organizations gain visibility and control over their current network state. This work de-risks later automation efforts by grounding them in clean, validated, and structured data.
