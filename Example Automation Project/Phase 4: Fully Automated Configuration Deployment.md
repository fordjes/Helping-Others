
## Objective

Build upon the semi-automated foundation to create a fully automated, closed-loop workflow for network configuration deployment. This includes rendering, validation, delivery, compliance checks, and rollback mechanisms—eliminating the need for manual CLI entry while enhancing consistency, speed, and governance.

## Context and Why This Matters

In previous phases, we evolved from manual CLI changes to rendering validated configurations using a structured SoT model. Now we take the final step: enabling **automated deployment pipelines** that push validated configurations to devices via API, controller platforms, or CLI automation tools.

**Benefits:**

* **Consistency** across all device types and sites
* **Faster execution** with fewer delays between request and deployment
* **Automated compliance enforcement** (e.g., drift detection and rollback)
* **Improved collaboration** across teams with clearer workflows and auditability

> 🔄 **For Junior Engineers**: You’re not giving up learning CLI—you’re learning how to deliver it at scale.
>
> 🧠 **For Senior Engineers**: This phase preserves your intent while reducing manual effort and risk.

---

## Goals

* Automate the full lifecycle: render → validate → deploy → verify
* Integrate with SoT, Git, and CI/CD pipelines for control and traceability
* Build in validation gates and rollback strategies
* Support vendor diversity (CLI, API, controller-based platforms)

---

## Key Concepts

Understanding the different methods of automated deployment is critical to aligning the capabilities of your team with the technologies present in your network. Each deployment method offers unique trade-offs depending on vendor support, team familiarity, and integration complexity.

For engineers coming from traditional CLI backgrounds, this section serves as a bridge—translating familiar concepts (like copy-pasting config blocks into terminal sessions) into automated equivalents. For senior engineers with experience in tooling, this reinforces how to standardize and scale those workflows.

### Why Deployment Methods Matter

* **CLI Push** helps teams leverage existing knowledge while introducing automation tools like Ansible and Nornir.
* **API-Based Push** increases speed and reliability when platforms like Arista, Juniper, or Palo Alto provide robust REST or gRPC interfaces.
* **Controller Push** takes advantage of vendor-native orchestration, abstracting hardware details and enabling things like real-time telemetry and intent-based networking.

**Teaching Example for Juniors:**
Imagine deploying a BGP neighbor configuration. With CLI push, you use Ansible to send `router bgp` commands via SSH. With an API, you POST a JSON payload to the device. With a controller, you upload a config profile to DNAC and assign it to the device group. The result is the same—BGP comes online—but the method, reliability, and auditability differ.

**Design Consideration for Seniors:**
Choosing the right method is a balance of:

* Device capabilities (e.g., does NX-OS version X support RESTCONF?)
* Risk tolerance (e.g., controller rollback is safer but less transparent)
* Long-term maintainability (e.g., Git + API offers fine-grained control)

> 🧭 Reference: [Cisco DNAC Template Programming Guide](https://developer.cisco.com/docs/dna-center/#!template-programming/overview)
>
> 🔗 Reference: [Ansible Network Modules](https://docs.ansible.com/ansible/latest/network/index.html)

By understanding and selecting the right deployment method(s), teams align automation efforts with operational goals and technical realities—laying the groundwork for scalable, resilient network infrastructure.

### Deployment Methods

| Type                | Examples                             | Notes                                            |
| ------------------- | ------------------------------------ | ------------------------------------------------ |
| **CLI Push**        | Ansible, Nornir, Netmiko             | Best for CLI-native platforms (Cisco IOS, NX-OS) |
| **API-Based Push**  | Arista eAPI, Juniper REST, Palo Alto | Preferred when modern API access is available    |
| **Controller Push** | DNAC, Catalyst Center, Aruba Central | Allows abstracted orchestration + telemetry      |

> 🧩 Choose the method based on platform capabilities, team skillsets, and long-term scalability.

---

## Activities

In this section, we break down the core steps needed to implement a fully automated deployment system. Each activity reflects a critical piece of the puzzle—whether you're just getting started or refining an existing automation pipeline.

For **junior engineers**, this is an opportunity to connect CLI knowledge and config syntax to how those same commands can be generated, validated, and deployed programmatically. You'll gain hands-on understanding of the moving parts that automate a change.

For **senior engineers**, this is about elevating operational maturity. Each activity provides a structure to minimize manual errors, reduce time-to-implement, and increase control over complex infrastructure.

Throughout these activities, we aim to strike a balance between:

* **Control** (via validation and RBAC)
* **Speed** (through automation and CI/CD)
* **Safety** (enabled by drift detection and rollback support)

We'll also reference familiar tools like Git, Ansible, and Jenkins to help you apply these patterns with technologies your teams may already use.

### 1. Establish Deployment Gateways

**Purpose:** Secure and centralize access to deploy changes.

* Configure jump hosts or automation runners with necessary SSH/API credentials
* Use vaults (e.g., HashiCorp Vault, Ansible Vault) to store secrets
* Implement RBAC to protect who can trigger deployments

### 2. Build the Automation Pipeline

**Purpose:** Define the end-to-end deployment workflow.

**Stages:**

1. **Trigger** – Git commit, CI form, SoT webhook
2. **Render** – Use Jinja2 with SoT data
3. **Validate** – Run syntax check, linting, logic validation
4. **Deploy** – Push to devices via CLI/API
5. **Post-checks** – Confirm config success (e.g., BGP/OSPF up, interface state)
6. **Log & Audit** – Store job results, logs, and diffs in Git or a central dashboard

**Tools:** GitHub Actions, Jenkins, Ansible AWX, Nornir, Nautobot Jobs

### 3. Develop Rollback Strategies

**Purpose:** Ensure safe recovery from failed deployments.

* Pre-pull current config as backup
* Render rollback config using reverse template logic
* Validate rollback paths (e.g., interface disable, route retraction)

### 4. Post-Change Validation

**Purpose:** Validate deployment success using show commands or API checks.

Examples:

* Verify `show ip route` includes new prefixes
* Confirm `show bgp summary` peer status is Established
* Use SNMP, gNMI, or streaming telemetry where available

### 5. Compliance and Drift Detection

**Purpose:** Maintain alignment with config baselines.

* Periodically re-render config from SoT
* Compare against current device state
* Flag drift and trigger corrective workflows

---

## OKRs & KPIs

### OKRs

| Objective                         | Key Results                                  |
| --------------------------------- | -------------------------------------------- |
| Enable full deployment automation | 90% of standard changes handled via pipeline |
| Reduce change lead time           | Average deploy time reduced by 50%           |
| Enforce config compliance         | 100% of devices checked monthly for drift    |

### KPIs

| Metric                 | Target                                     |
| ---------------------- | ------------------------------------------ |
| Config drift incidents | < 5% deviation month over month            |
| Rollback success rate  | 100% of tested rollback scenarios succeed  |
| Post-check success     | ≥ 95% of deployments pass automated checks |

---

## Deliverables

* Config deployment pipeline (render → validate → push → verify)
* Git-based audit trail of changes
* Rollback library for common configuration failures
* Compliance check scripts and reporting dashboard
* Role-based access model for change initiation and approval

---

## Next Steps

* Expand automation to multi-domain (e.g., firewalls, load balancers)
* Integrate with change request systems (ServiceNow, Jira)
* Standardize DR/rollback testing for all change types
* Enable closed-loop remediation workflows with monitoring and event feedback
