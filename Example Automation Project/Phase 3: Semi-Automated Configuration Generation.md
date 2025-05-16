
## Objective

Enable repeatable, reliable, and partially automated configuration generation by leveraging the Source of Truth (SoT) platform. This phase focuses on transitioning teams from manually typed CLI commands to rendered and reviewed configurations using automation tools such as Jinja2, Python, and Ansible.

## Context and Why This Matters

In earlier phases, we captured the current network state and built a structured data model using a Source of Truth (SoT) platform. Most teams, however, are still used to managing change via CLI sessions, tribal knowledge, and spreadsheets. In this phase, we introduce semi-automated config generation—an approach that reduces human error, accelerates delivery, and starts shifting teams toward infrastructure-as-code (IaC).

**Why this is important:**

* Manual changes introduce inconsistency and often rely on memory or copy-paste methods, especially across diverse vendors (Cisco, Juniper, Aruba, etc.).
* Teams often lack an enforceable standard and must rely on peer review or ad hoc processes to ensure quality.
* As networks grow and evolve, scaling manual efforts becomes infeasible and unsustainable.

This phase gives teams the ability to:

* **Render repeatable, standardized configurations** based on known-good templates
* **Visualize and validate changes** before applying them to production
* **Connect abstract network intent** (stored in SoT) to concrete, vendor-ready configuration commands
* **Iterate safely** through dry-run reviews and version control, building trust in automation workflows

> 👩‍💻 **For Junior Engineers**: This is your chance to learn how automation fits into network operations. You’ll still need to know CLI, but now you’ll write fewer commands and make fewer mistakes.

> 👨‍🏫 **For Senior Engineers**: You can scale your best practices across the entire team. You won’t lose control—you’ll gain repeatability, visibility, and auditability.

By the end of this phase, teams should be able to generate configs for multiple vendors from shared templates and structured data, review those configs before deployment, and prepare for full automation in the next phase.
By this point, the SoT platform now holds structured information about devices, interfaces, IPs, VLANs, and more. However, most changes are still executed manually via CLI. This phase introduces automation tools to generate configurations dynamically based on SoT data, allowing:

* Reduced human error in config creation
* Enforcement of config standards across sites/vendors
* Faster delivery of routine changes
* Repeatability for lab → staging → production deployments

> 🧠 **Teaching Tip**: Think of the SoT as your recipe book, and config generation as the chef preparing dishes exactly to spec—consistently and without guesswork.

---

## Goals

* Develop reusable Jinja2 configuration templates for core services
* Build Python and/or Ansible jobs to render configurations from SoT
* Enable dry-run preview of changes before deployment
* Validate rendered configs against device-specific constraints

---

## Key Concepts

### What is Config Rendering?

**Config rendering** is the process of transforming structured data—typically pulled from a Source of Truth (SoT)—into device-ready configurations using templating logic. It bridges the gap between abstract intent and concrete implementation.

Rather than writing CLI configs line-by-line, engineers define **templates** with placeholders (like `{{ ip_address }}`), and feed in device-specific values from structured data (YAML, JSON, or directly from APIs).

This allows teams to:

* **Enforce standards**: Every interface, VLAN, or BGP session follows the same template.
* **Avoid errors**: No more typos, misaligned indentation, or forgotten NTP servers.
* **Scale efficiently**: Generate configs for dozens—or thousands—of devices automatically.
* **Enable review and version control**: Treat configurations like code (GitOps).

**Example Scenario:**
You're provisioning a new access switch at a branch office. The interface IPs, VLANs, descriptions, and routing requirements are already stored in your SoT (e.g., Nautobot).

**Step-by-Step Flow:**

1. SoT holds: interface = `GigabitEthernet1/0/1`, description = "Uplink to Core", IP = `10.1.10.1/24`
2. Jinja2 template:

   ```jinja
   interface {{ interface }}
    description {{ description }}
    ip address {{ ip }}
   ```
3. Python or Ansible loads SoT values into the template.
4. Rendered output:

   ```
   interface GigabitEthernet1/0/1
    description Uplink to Core
    ip address 10.1.10.1 255.255.255.0
   ```

**Why It Matters:**

* **Junior engineers** get a repeatable, mistake-resistant way to generate changes without memorizing CLI syntax.
* **Senior engineers** can ensure consistency across platforms and reduce troubleshooting time caused by misconfiguration.
* **Cross-functional teams** can work from shared data models, simplifying handoffs.

> 🔍 **Tip**: Templating logic (if/else, loops) makes it easy to handle conditional configs—for example, add a second IP if a certain variable exists, or omit QoS for devices that don't support it.

> 🧪 **Advanced Use**: Teams can layer in logic to generate complete router configurations based on device role, site region, and peer data—fully automating what once took hours manually.?
> Config rendering is the process of using structured data (from SoT) and templates (Jinja2) to generate device-specific configurations. This bridges the gap between abstract network design and concrete CLI-ready commands.

**Example:**
SoT Data:

```yaml
interface: GigabitEthernet0/1
ip: 10.1.1.1
mask: 255.255.255.0
description: Uplink to Core
```

Jinja2 Template:

```jinja
interface {{ interface }}
 description {{ description }}
 ip address {{ ip }} {{ mask }}
```

Rendered Output:

```
interface GigabitEthernet0/1
 description Uplink to Core
 ip address 10.1.1.1 255.255.255.0
```

---

## Tools for Config Generation

| Tool        | Role                                 | Strengths                                               |
| ----------- | ------------------------------------ | ------------------------------------------------------- |
| **Jinja2**  | Templating engine                    | Lightweight, readable, integrates with Python & Ansible |
| **Python**  | Scripted rendering and validation    | Great for custom logic and pipelines                    |
| **Ansible** | Job orchestration, config preview    | Vendor modules, parallelism, extensibility              |
| **Git**     | Version control of generated configs | Rollback, audit, team collaboration                     |

---

## Activities

### 1. Template Design and Standardization

**Purpose:** Create consistent templates for interfaces, routing, services, etc.

**Steps:**

* Start with one platform (e.g., Cisco IOS)
* Identify repeatable config patterns: NTP, interfaces, BGP, VLANs
* Translate each block into Jinja2 templates
* Group templates per vendor/family (IOS, NX-OS, Junos, EOS)

**Tips:**

* Use Jinja conditionals (`{% if %}`) to support optional settings
* Abstract site- or device-specific values into variables

**Deliverables:**

* Folder of Jinja2 templates, categorized by config domain

---

### 2. Data Extraction from SoT

**Purpose:** Retrieve structured, automation-ready configuration data from your Source of Truth (SoT) platform to serve as the input for rendering device configurations using Jinja2 templates. This is where we turn raw inventory into usable variables.

**Why This Matters:**

* SoT holds the current and desired state of the network in a structured format (e.g., site, device, interface, IP, VLAN, routing metadata).
* Automation depends on accurate and complete input—your rendered configuration is only as good as your source data.
* Connecting the SoT to your automation pipeline enables consistency, scalability, and a reduction in manual data entry errors.

**How To Extract the Data:**
Depending on your SoT platform (e.g., Nautobot, NetBox, Infrahub), there are multiple methods to access the structured data:

**Option 1: REST API (NetBox/Nautobot)**

* Easy to integrate with Python scripts or Ansible modules
* Supports token-based auth and filtering

**Option 2: GraphQL (Nautobot)**

* Efficient for complex queries and retrieving nested objects
* Ideal when fetching related data such as interfaces tied to devices

**Option 3: CSV or JSON Export (Infrahub, NetBox)**

* Useful for snapshots, testing, or offline workflows

**Option 4: Plugin Integration or Scheduled Job**

* Use Nautobot Jobs or NetBox scripts to extract and push to Git/CI pipelines

**Example Python Script (NetBox REST):**

```python
import requests
url = "https://netbox.local/api/dcim/devices/?site=dc1"
headers = {"Authorization": "Token abcd1234"}
response = requests.get(url, headers=headers)
devices = response.json()['results']
for device in devices:
    print(device['name'], device['role']['name'], device['primary_ip'])
```

**Example GraphQL Query (Nautobot):**

```graphql
{
  devices(site: "branch-a") {
    name
    device_type { model }
    interfaces {
      name
      ip_addresses { address }
    }
  }
}
```

**For Junior Engineers:**

* Think of SoT data like filling out a form—hostname, site, IP, role. This data feeds into a config template like values filling blanks in a letter.
* Practice running these queries and exploring what fields are available in your SoT system.

**For Senior Engineers:**

* Evaluate data completeness and quality: Are descriptions meaningful? Are IPs assigned consistently? Are all mandatory fields populated?
* Build validation scripts that pre-check the data before passing it to rendering workflows.

**Deliverables:**

* Python scripts or Ansible playbooks to query SoT
* Sample JSON/YAML exports of device data
* Validated variable sets ready for input to config templates
* Documentation on query logic and authentication practices
  **Purpose:** Pull structured data from SoT to feed into templates

**Approaches:**

* Use SoT API (e.g., Nautobot GraphQL, NetBox REST) to query device/interface data
* Optionally cache results in local JSON/YAML for testing

**Python Example:**

```python
import requests
r = requests.get("https://nautobot/api/dcim/devices/", headers={"Authorization": "Token xyz"})
devices = r.json()['results']
```

**Deliverables:**

* Data query scripts
* Example JSON datasets for rendering tests

---

### 3. Render and Validate Configs

**Purpose:** Generate full configuration files per device using SoT data and templates, then review and validate those outputs to ensure they meet operational standards, vendor syntax, and intended design. This step serves as the technical quality gate before any configuration is proposed for change control or deployment.

**Why This Matters:**

* Configs must be accurate to avoid downtime or misconfigurations.
* Validating output ensures trust in automation workflows.
* Junior engineers gain confidence by reviewing output before pushing.
* Senior engineers can enforce consistency across the environment.

**Steps:**

1. **Combine SoT data with Templates:**

   * Use Jinja2 templates and Python or Ansible to generate device-specific configurations.
   * Store rendered configs in version-controlled directories named after device hostnames, sites, or roles (e.g., `configs/branch-a/sw1.conf`).

2. **Perform Syntactic Validation:**

   * Run simple regex checks to validate expected structure (e.g., `hostname` line exists, IP address formats are correct).
   * Use CLI emulators or linting tools to flag invalid or unsupported syntax.

3. **Run Platform-Specific Validators:**

   * Use tools like `pyATS validate`, `ciscoconfparse`, or mock configuration loaders.
   * For Juniper: dry-parse config using `commit check` on a staging device.

4. **Compare to Baseline Configs:**

   * Use `difflib`, `Git`, or GUI diff tools (e.g., Meld, VS Code) to compare rendered configs to baseline or previously deployed versions.
   * Highlight unexpected deviations (e.g., missing NTP or unplanned OSPF changes).

5. **Log and Resolve Issues:**

   * Capture errors to a structured log (CSV, Markdown, or JSON).
   * Tag template or SoT records for correction.

**Example Python Snippet (Validation):**

```python
from difflib import unified_diff
with open('new_config.txt') as new, open('baseline.txt') as base:
    diff = unified_diff(base.readlines(), new.readlines())
    for line in diff:
        print(line)
```

**Example pyATS Usage:**

```bash
pyats validate --testbed testbed.yaml --config configs/sw1.conf --platform iosxe
```

**Tips for All Levels:**

* **Junior Engineers:** Focus on understanding how each variable from the SoT maps into the rendered output. Practice modifying SoT records and rerunning the render.
* **Senior Engineers:** Focus on validation quality and ensuring templates cover corner cases like unused interfaces, VRFs, or site-specific overrides.

**Deliverables:**

* Fully rendered `.conf` or `.txt` files per device
* Validation results: pass/fail logs, linting reports
* Change diffs: side-by-side comparison outputs
* Feedback loop: error report with links to SoT fields or templates that need fixing
  **Purpose:** Generate full config files per device, review and validate

**Steps:**

* Combine SoT data with templates using Python or Ansible
* Render into `.txt` or `.conf` files
* Perform basic linting and syntax validation (e.g., regex, `ciscoconfparse`, `pyats validate`)

**Deliverables:**

* Rendered configuration files
* Error logs or diff outputs if validations fail

---

### 4. Dry-Run Change Reviews

**Purpose:** Let engineers preview changes before deployment

**Approaches:**

* Use `--check` mode in Ansible to show intent
* Git diffs between previous and new rendered configs
* Visual review in code editor or pull request

**Deliverables:**

* Sample dry-run output
* Change approval documentation template

---

### 5. Team Enablement and SOP Updates

**Purpose:** Prepare engineering and operations teams to adopt and own the new automation workflows by providing practical training, clear documentation, and updated operational procedures.

This step bridges the human gap between traditional CLI-based operations and the automation-powered, template-driven approach. Without alignment and understanding, even the most powerful automation pipelines can be misused, ignored, or even blocked.

**Why This Is Critical:**

* Senior engineers need confidence that automation won’t break their network.
* Junior engineers need clarity and guided practice to build trust in the tools.
* All teams need a shared understanding of responsibilities and processes.

**Steps:**

1. **Introduce the Why:**

   * Present common configuration issues found during manual operations (e.g., mismatched MTUs, missing SNMP servers).
   * Highlight how automation tools eliminate those problems.

2. **Hands-On Workshops:**

   * Walkthrough: "Here’s a device in the SoT → Here’s its config generated via Jinja2 → Here’s how we review and deploy."
   * Have engineers edit a Jinja2 template and rerun rendering.
   * Practice dry-runs using Git or Ansible check mode.

3. **Update Operational Runbooks and SOPs:**

   * Integrate new roles like **template reviewer**, **SoT data owner**, and **change submitter**.
   * Include examples of:

     * What fields to fill in the SoT when requesting a new VLAN
     * How to verify a rendered config before pushing
     * Where rendered configs are stored in Git and how to read diffs

4. **Build Feedback Loops:**

   * Use retrospectives or surveys to gather input from users adopting automation.
   * Log requests for new template features, config nuances, or visual feedback needs.

5. **Create Peer Mentorship Opportunities:**

   * Match senior engineers who know the network with junior staff fluent in Python/Ansible.
   * Have pairs work together to build or troubleshoot a rendered config pipeline.

**Example Scenarios to Train On:**

* Adding a new interface description via SoT and seeing it reflected in the config
* Correcting a bad IP in the SoT and rerunning the template to see the fix
* Submitting a new branch to Git with a proposed configuration change for review

**Deliverables:**

* Versioned internal SOP documentation
* Slide decks and recorded demos of config rendering lifecycle
* Live or self-paced workshop materials with step-by-step labs
* Template library changelog and ownership map
* Feedback form or automation backlog for ongoing improvement
  **Purpose:** Train teams on new tools, document standard operating procedures

**Steps:**

* Host hands-on workshops on config rendering
* Update internal runbooks and ticket workflows
* Define team roles for template maintenance, data ownership

**Deliverables:**

* Internal SOP documentation
* Training materials or recorded walkthroughs

---

## OKRs & KPIs

### OKRs

| Objective                           | Key Results                                |
| ----------------------------------- | ------------------------------------------ |
| Enable structured config generation | 3+ Jinja2 templates per vendor family      |
| Validate consistency of output      | 90% of configs render without syntax error |
| Train engineering staff             | 100% of ops engineers attend 1 workshop    |

### KPIs

| Metric                   | Target                                  |
| ------------------------ | --------------------------------------- |
| Rendered config coverage | ≥ 80% of routers/switches               |
| Template reuse rate      | ≥ 70% of changes generated via template |
| Config rendering errors  | < 5% per run                            |

---

## Risks and Mitigations

| Risk                                  | Mitigation                                                      |
| ------------------------------------- | --------------------------------------------------------------- |
| Templates are too rigid or incomplete | Iterate with small pilots and refine per use case               |
| Inaccurate SoT data breaks render     | Add data validation scripts and schema enforcement              |
| Teams resist change                   | Provide workshops, side-by-side comparisons with manual methods |

---

## Deliverables

* Jinja2 template library
* Scripts for SoT data extraction
* Rendered configs and validation logs
* SOP documentation for config generation
* Workshop training materials

---

## Next Steps

* Integrate rendered config into deployment workflows (Phase 4)
* Begin storing rendered configs in Git for history/audit
* Expand template coverage to new vendors, services, and site types
* Implement validation pipelines to catch errors before deployment
