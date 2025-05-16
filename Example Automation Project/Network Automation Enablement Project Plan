# 🌐 Network Automation Enablement Project Plan

*From Manual to Fully Automated Network Operations*

---

## 🚀 Executive Summary

This phased project guides the transformation of traditional manual router and switch configuration workflows into a fully automated, scalable, and source-of-truth (SoT)-driven network infrastructure. The implementation is vendor-agnostic and adaptable to mixed environments (Cisco, Juniper, Arista, Aruba). The objective is to unify network configuration, lifecycle management, and compliance through modern tools like **NetBox**, **Nautobot**, or **Infrahub** with automation powered by **Python**, **Ansible**, **Terraform**, and **PowerShell**.

---

## 🧡 Project Objectives

### OKRs – Project-Wide

| Objective                                        | Key Results                                                     |
| ------------------------------------------------ | --------------------------------------------------------------- |
| Build a reliable automation foundation           | Deploy SoT with device data for 100% of target routers/switches |
| Replace manual config with repeatable automation | Automate 80% of CLI-driven changes via templates/API            |
| Enable multi-vendor network management           | Integrate at least 3 different vendor platforms                 |
| Ensure ongoing compliance and governance         | Validate configs against baseline templates monthly             |

---

## 📊 Phase-by-Phase Plan

### 🛠️ **Phase 1: Manual Configuration Baseline**

**Goal:** Document the current environment, standardize configuration baselines, and prepare for automation.

**Activities:**

* Inventory routers and switches manually.
* Capture running configurations.
* Normalize configurations into standardized baseline templates per vendor.
* Identify key CLI config deltas across sites.

**Tools:**

* CLI (SSH/Telnet)
* Text comparison (VS Code, Notepad++, diff tools)

**OKRs & KPIs:**

| Objective                           | KPI                                                                 |
| ----------------------------------- | ------------------------------------------------------------------- |
| Document current network config     | 100% of target devices have reviewed and version-controlled configs |
| Identify vendor-specific variations | Create at least 3 reusable baseline templates                       |

---

### 🧩 **Phase 2: Introduce Source of Truth (SoT)**

**Goal:** Choose and deploy a SoT platform, populate network data, and define key data models.

**Tool Options & Comparison:**

| Tool         | Strengths                                                   | Weaknesses                                         |
| ------------ | ----------------------------------------------------------- | -------------------------------------------------- |
| **NetBox**   | Strong DCIM and IPAM, mature plugin ecosystem               | API requires care when modeling custom config data |
| **Nautobot** | Active plugin development, extensible with jobs and GraphQL | Slightly heavier than NetBox in setup              |
| **Infrahub** | Git-native config-driven SoT                                | Less UI-friendly, newer in maturity                |

**Example Models to Populate:**

* Device inventory (hostname, model, site)
* Interfaces (name, type, VLANs)
* IP addresses and subnets
* VLANs and VRFs
* Platform-specific metadata (OS type, API endpoint)

**Actions:**

* Deploy SoT (e.g., Nautobot via Docker).
* Import data from spreadsheets or scripts.
* Validate accuracy and establish SoT ownership model.

**OKRs & KPIs:**

| Objective               | KPI                                                           |
| ----------------------- | ------------------------------------------------------------- |
| Establish reliable SoT  | 100% of devices modeled with valid interface/IP/VLAN metadata |
| Validate SoT usefulness | 1 successful config rendered using SoT data                   |

---

### ⚙️ **Phase 3: Semi-Automated Configuration Generation**

**Goal:** Generate configurations from SoT models using Jinja2 templates and validate changes offline.

**Technologies:**

* **Python** + **Jinja2** for config rendering
* **Ansible** for playbook-driven device config staging
* **Terraform** (optional) for supporting cloud or overlay network setup

**Examples:**

**Python + Jinja2 Rendering:**

```python
from jinja2 import Template

template = Template("interface {{ interface }}\n ip address {{ ip }}/{{ mask }}")
print(template.render(interface="GigabitEthernet0/1", ip="10.1.1.1", mask="24"))
```

**Why:** Enables dry-run config generation before pushing to devices. Decouples data from logic.

**API-Based Push (if vendor supports):**

* **Cisco DNAC / Catalyst Center:** REST API to push templates
* **Aruba Central:** HTTP POST JSON body to `configuration_template`
* **Juniper/Arista:** NETCONF/RESTCONF available depending on version

**Ansible Example:**

```yaml
- name: Push BGP config to Cisco IOS
  ios_config:
    lines:
      - router bgp 65000
      - neighbor 10.0.0.1 remote-as 65001
    parents: interface GigabitEthernet1
```

**OKRs & KPIs:**

| Objective                     | KPI                                                 |
| ----------------------------- | --------------------------------------------------- |
| Generate valid config per SoT | 90% match between rendered config and golden config |
| Limit manual CLI intervention | 50% of changes now generated from SoT               |

---

### 🤖 **Phase 4: Fully Automated Lifecycle Workflows**

**Goal:** Enable zero-touch configuration deployment and auditing using SoT + CI/CD-like automation.

**Workflow Overview:**

1. User submits change request via GitHub/Nautobot Job/CI Form.
2. SoT is updated with change request.
3. Automation tool generates and pushes config (Ansible/Python/Terraform).
4. Post-checks validate compliance.
5. Logs are stored in ELK/Grafana for auditing.

**Optional Integrations:**

* **Controller-Based Management:**

  * Aruba Central
  * Cisco Catalyst Center
* **PowerShell** (Windows server automation if needed)
* **GitHub Actions** for pipeline-driven changes

**Security & Governance:**

* RBAC on SoT changes
* Git-backed audit trails
* Drift detection using periodic config pulls

**OKRs & KPIs:**

| Objective                           | KPI                                                 |
| ----------------------------------- | --------------------------------------------------- |
| Full automation of repeatable tasks | 80% of port/VLAN/IP updates done without manual CLI |
| Reduce config errors                | 90% decrease in post-deployment incidents           |
| Compliance baseline enforcement     | 100% monthly validation of config drift             |

---

## 📈 Tooling Evaluation Summary

| Tool           | Use Case                        | Pros                               | Cons                                 |
| -------------- | ------------------------------- | ---------------------------------- | ------------------------------------ |
| **Ansible**    | Multi-vendor config push        | Readable, modular, large ecosystem | Can be verbose, slow for large-scale |
| **Python**     | Custom workflows, render logic  | Flexible, reusable libraries       | Needs more scaffolding               |
| **Terraform**  | Cloud/VPN overlay (e.g. SD-WAN) | Good for stateful infra-as-code    | Not ideal for CLI-based net config   |
| **PowerShell** | Windows/DC automation           | Strong on Windows-native platforms | Weak multi-vendor CLI support        |

---

## 🏁 Conclusion

This project plan provides a **practical, progressive approach to network automation** suitable for enterprises with mixed vendor infrastructure. By focusing on vendor-agnostic **source-of-truth modeling** and pairing it with flexible automation tools like **Ansible**, **Python**, and **Terraform**, teams can confidently modernize their network operations.

---

## 📚 References & Resources

* [Nautobot](https://github.com/nautobot/nautobot)
* [NetBox](https://github.com/netbox-community/netbox)
* [Infrahub](https://infrahub.dev/)
* [Cisco Catalyst Center API Docs](https://developer.cisco.com/docs/dnac/)
* [Aruba Central API Docs](https://developer.arubanetworks.com/aruba-central/docs/overview)
* [Ansible Network Guide](https://docs.ansible.com/network/)
* [Jinja2 Templates](https://jinja.palletsprojects.com/)
