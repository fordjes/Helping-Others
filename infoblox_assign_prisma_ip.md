# Integrating Infoblox with Palo Alto Prisma SD-WAN ION via Ansible and Python

## Objective

Create an Ansible-driven workflow that:

1. Queries Infoblox for available IP address data (IP prefixes, DNS entries, DHCP scopes)
2. Formats this data to match the requirements of Palo Alto Prisma SD-WAN ION device configuration
3. Uses a Python script to push the formatted data into the Prisma SD-WAN API (as Ansible lacks a native collection for Prisma)

---

## 1. Prerequisites

### Tools Required

* Ansible
* Infoblox NIOS Ansible Collection ([Ansible Galaxy Link](https://galaxy.ansible.com/infoblox/nios_modules))
* Python 3.x
* `requests` or `httpx` Python module
* Prisma SD-WAN API access ([https://pan.dev/sdwan/api/](https://pan.dev/sdwan/api/))
* Credentials for both Infoblox and Prisma SD-WAN

---

## 2. Variables Required

This section defines the necessary variables to orchestrate the automation workflow between Infoblox and Palo Alto Prisma SD-WAN. These variables are typically stored in `group_vars/all.yml`, `host_vars/<host>.yml`, or passed via the CLI. They provide the dynamic inputs required for subnet filtering, DNS record creation, IP allocation, and API authentication.

Key categories include:

* Infoblox API connection settings
* Subnet metadata for IP filtering
* DNS zone configuration
* Prisma SD-WAN API settings

### Example Variable File (`vars.yml`)

```yaml
# Infoblox Connection and Network Context
infoblox_server: "infoblox.company.com"
infoblox_username: "apiuser"
infoblox_password: "password"
infoblox_network_view: "default"

# Subnet selection variables (used in subnet filtering logic)
site: "branch01"          # Logical site or location label
function: "wan-edge"      # Role or purpose of the interface (e.g., management, loopback, etc.)
zone: "dmz"               # Logical network zone (e.g., trusted, untrusted, guest)

# DNS zone used for forward record creation
dns_zone: "branch01.company.com"

# Comment template for DHCP scope naming
# This could be used if building DHCP scopes in the future
dhcp_scope_comment: "{{ site | upper }} {{ function | upper }} Scope"

# Optional hardcoded fallback subnet (overridden if dynamic lookup succeeds)
subnet_prefix: ""

# Prisma SD-WAN API details
prisma_base_url: "https://api.prisma.io"
prisma_token: "{{ vault_prisma_token }}"   # Stored securely in Ansible Vault
site_id: "12345"  # The ID of the Prisma site to which the device/interface belongs

# This will be dynamically populated at runtime
assigned_ip: ""
```

### Additional Notes

* The variables `site`, `function`, and `zone` serve as metadata tags and are critical for Infoblox subnet filtering.
* `dns_zone` should align with your internal DNS naming conventions.
* `vault_prisma_token` should be encrypted using `ansible-vault`.
* `subnet_prefix` allows a fallback path if no dynamic subnet match is found (not commonly used in scalable automation).
* You may optionally define `hostname` or `ion_id` variables to further automate naming or downstream API calls.

Using structured and parameterized variables enables multi-site support, repeatability, and dynamic provisioning—all essential for scalable network automation.

```yaml
# group_vars/all.yml or vars.yml
infoblox_server: "infoblox.company.com"
infoblox_username: "apiuser"
infoblox_password: "password"
infoblox_network_view: "default"
site: "branch01"
function: "wan-edge"
zone: "dmz"
dns_zone: "branch01.company.com"

dhcp_scope_comment: "{{ site | upper }} {{ function | upper }} Scope"

# The subnet_prefix can be derived dynamically based on site/function/zone from Infoblox
# Or hardcoded if needed (this will be overridden by dynamic lookup if used)
subnet_prefix: ""

prisma_base_url: "https://api.prisma.io"
prisma_token: "{{ vault_prisma_token }}"
site_id: "12345"  # Target Prisma site ID

# This will be built dynamically by the playbook
assigned_ip: ""
```

---

## 3. Enhanced Ansible Playbook to Gather Data from Infoblox

This playbook automates the process of querying Infoblox to allocate an IP address from a dynamically identified subnet based on descriptive metadata such as `site`, `function`, and `zone`. This information is typically embedded in the `comment` field or as extensible attributes on network objects in Infoblox. The result is a fully parameterized and reusable automation pattern.

Key steps include:

* Dynamic subnet selection using regex filter matching
* Retrieval of the next available IP
* DNS registration for traceability
* Handoff of the assigned IP to a Python script for device configuration in Prisma

```yaml
- name: Pull IP and DNS data from Infoblox
  hosts: localhost
  gather_facts: false
  vars_files:
    - vars.yml
  tasks:
    # Step 1: Search for a subnet matching the site/function/zone metadata in the Infoblox comment field
    - name: Search for subnet based on site/function/zone in comment
      infoblox.nios_modules.nios_search:
        object: network
        return_fields:
          - network
          - comment
        filter:
          - "comment~=site:{{ site }}.*function:{{ function }}.*zone:{{ zone }}"
        provider:
          host: "{{ infoblox_server }}"
          username: "{{ infoblox_username }}"
          password: "{{ infoblox_password }}"
          ssl_verify: false
      register: subnet_search_result

    # Step 2: Ensure a subnet was found, fail early if not to prevent misconfiguration
    - name: Validate subnet search result
      fail:
        msg: "No subnet found for site={{ site }}, function={{ function }}, zone={{ zone }}"
      when: subnet_search_result.result | length == 0

    # Step 3: Capture the subnet found as a new fact for future use
    - name: Set subnet_prefix fact
      set_fact:
        subnet_prefix: "{{ subnet_search_result.result[0].network }}"

    # Step 4: Use Infoblox to retrieve the next available IP address within the selected subnet
    - name: Get next available IP in subnet
      infoblox.nios_modules.nios_next_ip:
        network: "{{ subnet_prefix }}"
        network_view: "{{ infoblox_network_view }}"
        provider:
          host: "{{ infoblox_server }}"
          username: "{{ infoblox_username }}"
          password: "{{ infoblox_password }}"
          ssl_verify: false
      register: next_ip_result

    # Step 5: Store the IP address as an Ansible fact for later use in DNS and API tasks
    - name: Set assigned IP fact
      set_fact:
        assigned_ip: "{{ next_ip_result.ip_address }}"

    # Step 6: Register the IP address as a DNS A record for hostname resolution in your DNS system
    - name: Create DNS A record for ION device
      infoblox.nios_modules.nios_a_record:
        name: "ion01.{{ dns_zone }}"
        ipv4addr: "{{ assigned_ip }}"
        view: "{{ infoblox_network_view }}"
        state: present
        provider:
          host: "{{ infoblox_server }}"
          username: "{{ infoblox_username }}"
          password: "{{ infoblox_password }}"
          ssl_verify: false

    # Step 7: Call the external Python script to configure the Prisma ION interface with the assigned IP
    - name: Call Python script to configure Prisma SD-WAN
      command: >
        python3 configure_prisma.py
        --ip {{ assigned_ip }}
        --site {{ site_id }}
        --token {{ prisma_token }}
        --base_url {{ prisma_base_url }}
      args:
        chdir: "/path/to/scripts"
```

### Additional Notes:

* This structure ensures the logic remains modular and portable across different locations and use cases.
* The `fail` task adds robustness, halting execution if no matching subnet is found.
* You may replace the `comment` field filtering with extensible attributes for more scalable and structured tagging.
* Adjust `dns_zone` and hostname generation logic to reflect your DNS naming standards.
* Consider registering a PTR record in Infoblox using `nios_ptr_record` if reverse DNS is required.

This Ansible playbook sets the stage for seamless IP address provisioning, ensuring the assigned address and DNS entry are well-documented and then pushing the configuration into the target Prisma ION device via API.

This playbook automates the process of querying Infoblox to allocate an IP address from a dynamically identified subnet based on descriptive metadata such as `site`, `function`, and `zone`. This information is typically embedded in the `comment` field or as extensible attributes on network objects in Infoblox. The result is a fully parameterized and reusable automation pattern.

Key steps include:

* Dynamic subnet selection using regex filter matching
* Retrieval of the next available IP
* DNS registration for traceability
* Handoff of the assigned IP to a Python script for device configuration in Prisma

```yaml
- name: Pull IP and DNS data from Infoblox
  hosts: localhost
  gather_facts: false
  vars_files:
    - vars.yml
  tasks:
    - name: Search for subnet based on site/function/zone in comment
      infoblox.nios_modules.nios_search:
        object: network
        return_fields:
          - network
          - comment
        filter:
          - "comment~=site:{{ site }}.*function:{{ function }}.*zone:{{ zone }}"
        provider:
          host: "{{ infoblox_server }}"
          username: "{{ infoblox_username }}"
          password: "{{ infoblox_password }}"
          ssl_verify: false
      register: subnet_search_result

    - name: Validate subnet search result
      fail:
        msg: "No subnet found for site={{ site }}, function={{ function }}, zone={{ zone }}"
      when: subnet_search_result.result | length == 0

    - name: Set subnet_prefix fact
      set_fact:
        subnet_prefix: "{{ subnet_search_result.result[0].network }}"

    - name: Get next available IP in subnet
      infoblox.nios_modules.nios_next_ip:
        network: "{{ subnet_prefix }}"
        network_view: "{{ infoblox_network_view }}"
        provider:
          host: "{{ infoblox_server }}"
          username: "{{ infoblox_username }}"
          password: "{{ infoblox_password }}"
          ssl_verify: false
      register: next_ip_result

    - name: Set assigned IP fact
      set_fact:
        assigned_ip: "{{ next_ip_result.ip_address }}"

    - name: Create DNS A record for ION device
      infoblox.nios_modules.nios_a_record:
        name: "ion01.{{ dns_zone }}"
        ipv4addr: "{{ assigned_ip }}"
        view: "{{ infoblox_network_view }}"
        state: present
        provider:
          host: "{{ infoblox_server }}"
          username: "{{ infoblox_username }}"
          password: "{{ infoblox_password }}"
          ssl_verify: false

    - name: Call Python script to configure Prisma SD-WAN
      command: >
        python3 configure_prisma.py
        --ip {{ assigned_ip }}
        --site {{ site_id }}
        --token {{ prisma_token }}
        --base_url {{ prisma_base_url }}
      args:
        chdir: "/path/to/scripts"
```

### Additional Notes:

* This structure ensures the logic remains modular and portable across different locations and use cases.
* The `fail` task adds robustness, halting execution if no matching subnet is found.
* You may replace the `comment` field filtering with extensible attributes for more scalable and structured tagging.
* Adjust `dns_zone` and hostname generation logic to reflect your DNS naming standards.
* Consider registering a PTR record in Infoblox using `nios_ptr_record` if reverse DNS is required.

This Ansible playbook sets the stage for seamless IP address provisioning, ensuring the assigned address and DNS entry are well-documented and then pushing the configuration into the target Prisma ION device via API.

```yaml
- name: Pull IP and DNS data from Infoblox
  hosts: localhost
  gather_facts: false
  vars_files:
    - vars.yml
  tasks:
    - name: Search for subnet based on site/function/zone
      infoblox.nios_modules.nios_search:
        object: network
        return_fields:
          - network
          - comment
        filter:
          - "comment~=site:{{ site }}.*function:{{ function }}.*zone:{{ zone }}"
        provider:
          host: "{{ infoblox_server }}"
          username: "{{ infoblox_username }}"
          password: "{{ infoblox_password }}"
          ssl_verify: false
      register: subnet_search_result

    - name: Set subnet_prefix fact
      set_fact:
        subnet_prefix: "{{ subnet_search_result.result[0].network }}"
      when: subnet_search_result.result | length > 0

    - name: Get next available IP in subnet
      infoblox.nios_modules.nios_next_ip:
        network: "{{ subnet_prefix }}"
        network_view: "{{ infoblox_network_view }}"
        provider:
          host: "{{ infoblox_server }}"
          username: "{{ infoblox_username }}"
          password: "{{ infoblox_password }}"
          ssl_verify: false
      register: next_ip_result

    - name: Set assigned IP fact
      set_fact:
        assigned_ip: "{{ next_ip_result.ip_address }}"

    - name: Create DNS A record
      infoblox.nios_modules.nios_a_record:
        name: "ion01.{{ dns_zone }}"
        ipv4addr: "{{ assigned_ip }}"
        view: "{{ infoblox_network_view }}"
        state: present
        provider:
          host: "{{ infoblox_server }}"
          username: "{{ infoblox_username }}"
          password: "{{ infoblox_password }}"
          ssl_verify: false

    - name: Call Python script to configure Prisma SD-WAN
      command: >
        python3 configure_prisma.py
        --ip {{ assigned_ip }}
        --site {{ site_id }}
        --token {{ prisma_token }}
        --base_url {{ prisma_base_url }}
```

---

## 4. Python Script: `configure_prisma.py`

This Python script is designed to be executed externally by an Ansible playbook, maintaining a separation of responsibilities where Ansible manages orchestration and credential/context injection, and Python handles API logic. This is ideal for teams who standardize on Ansible workflows but need to integrate with platforms that don’t yet support native collections.

### How Argument Passing Works

Arguments are passed to the script via the command line using the `command` or `shell` module in Ansible. The script uses Python’s `argparse` to receive and parse these arguments.

For example, the following Ansible task calls the script:

```yaml
- name: Configure Prisma ION using external Python script
  command: >
    python3 configure_prisma.py
    --ip {{ assigned_ip }}
    --site {{ site_id }}
    --token {{ prisma_token }}
    --base_url {{ prisma_base_url }}
  args:
    chdir: "/path/to/scripts"  # Where the Python script resides
```

Each argument passed (like `--ip`) corresponds to a value that will be accessed within the script as `args.ip`, using the `argparse` module.

### Python Script Logic Breakdown

```python
"""
This script assigns a static IP address to a specific WAN interface of a Palo Alto Prisma SD-WAN ION device.
It is executed by an Ansible playbook, which passes required parameters like IP, site ID, and API token.

Steps:
1. Authenticate with Prisma API using Bearer token
2. Retrieve the target ION device(s) under a site
3. Identify the WAN interface (e.g., 'wan0')
4. Push static IP configuration to the WAN interface
"""

import argparse
import requests

parser = argparse.ArgumentParser(
    description="Assign a static IP to a specific Prisma SD-WAN ION device's WAN interface"
)
parser.add_argument('--ip', required=True, help='IP address to assign')
parser.add_argument('--site', required=True, help='Prisma SD-WAN site ID')
parser.add_argument('--token', required=True, help='Bearer token for Prisma API')
parser.add_argument('--base_url', required=True, help='Prisma API base URL')
args = parser.parse_args()

# Construct the required headers for API authentication
headers = {
    "Authorization": f"Bearer {args.token}",
    "Content-Type": "application/json"
}

# Define static IP payload
# Create the IP configuration payload
# This assumes a /24 netmask and a known gateway address — adjust if needed
payload = {
    "ipv4_config": {
        "config_type": "STATIC",
        "static_config": {
            "ip": args.ip,
            "netmask": "255.255.255.0",
            "gateway": "10.10.10.1"
        }
    }
}

# Step 1: Get ION devices at the site
# Step 1: Build the endpoint to list ION elements assigned to the site
endpoint = f"{args.base_url}/sdwan/v2/sites/{args.site}/elements"
response = requests.get(endpoint, headers=headers)
response.raise_for_status()
# Extract device list from API response
device_list = response.json().get('data', [])

if not device_list:
    raise ValueError("No ION devices found for site")

# Step 2: Identify WAN interface
device_id = device_list[0]['id']
intf_endpoint = f"{args.base_url}/sdwan/v2/elements/{device_id}/interfaces"
interfaces = requests.get(intf_endpoint, headers=headers).json().get('data', [])
wan_if = next((i for i in interfaces if i['name'] == 'wan0'), None)

if not wan_if:
    raise Exception("WAN interface 'wan0' not found")

# Step 3: Push config
apply_endpoint = f"{intf_endpoint}/{wan_if['id']}"
put_resp = requests.put(apply_endpoint, headers=headers, json=payload)
put_resp.raise_for_status()
print(f"Successfully assigned {args.ip} to ION {device_id} interface {wan_if['name']}")
```

### Benefits of This Pattern

* Keeps Prisma logic in Python for easier maintenance as API evolves
* Ansible team stays in control of orchestration, variable injection, and vaults
* Python can be tested independently from playbooks

This model is ideal for hybrid automation environments and allows extension without restructuring the Ansible ecosystem.

This Python script is responsible for pushing static IP configuration to a Palo Alto Prisma SD-WAN ION device using the Prisma API. Since Prisma SD-WAN does not yet provide an official Ansible collection, this external script is referenced directly from Ansible using the `command` module, allowing teams with Ansible-centric workflows to maintain their process.

The script does the following:

* Takes input variables for the IP, Prisma site ID, API token, and base URL.
* Authenticates using a bearer token.
* Fetches all ION devices assigned to the specified site.
* Identifies the WAN interface to target (e.g., `wan0`).
* Applies the static IP configuration to the selected interface.

```python
import argparse
import requests

parser = argparse.ArgumentParser(description="Assign IP to Prisma SD-WAN ION site")
parser.add_argument('--ip', required=True, help='IP address to assign')
parser.add_argument('--site', required=True, help='Prisma SD-WAN site ID')
parser.add_argument('--token', required=True, help='Bearer token for Prisma API')
parser.add_argument('--base_url', required=True, help='Prisma API base URL')
args = parser.parse_args()

headers = {
    "Authorization": f"Bearer {args.token}",
    "Content-Type": "application/json"
}

# Define static IP payload - netmask and gateway should be adapted as needed
payload = {
    "ipv4_config": {
        "config_type": "STATIC",
        "static_config": {
            "ip": args.ip,
            "netmask": "255.255.255.0",
            "gateway": "10.10.10.1"
        }
    }
}

# Step 1: Get ION devices under the given site
endpoint = f"{args.base_url}/sdwan/v2/sites/{args.site}/elements"
response = requests.get(endpoint, headers=headers)
response.raise_for_status()
device_list = response.json().get('data', [])

if not device_list:
    raise ValueError("No ION devices found for site")

# Step 2: Select the first device and locate its WAN interface
device_id = device_list[0]['id']
# Step 2: Retrieve interface list for the selected ION device
wan_config_endpoint = f"{args.base_url}/sdwan/v2/elements/{device_id}/interfaces"
interfaces = requests.get(wan_config_endpoint, headers=headers).json().get('data', [])
wan_if = next((i for i in interfaces if i['name'] == 'wan0'), None)

if not wan_if:
    raise Exception("WAN interface 'wan0' not found")

# Step 3: Apply configuration
# Step 3: Build the endpoint and push the static IP configuration
apply_endpoint = f"{wan_config_endpoint}/{wan_if['id']}"
put_resp = requests.put(apply_endpoint, headers=headers, json=payload)
put_resp.raise_for_status()
print(f"Successfully assigned {args.ip} to ION {device_id} interface {wan_if['name']}")
```

### Calling the Script from Ansible

To keep integration with Ansible seamless, we use the `command` module to pass the required arguments dynamically:

```yaml
- name: Call Python script to configure Prisma SD-WAN
  command: >
    python3 configure_prisma.py
    --ip {{ assigned_ip }}
    --site {{ site_id }}
    --token {{ prisma_token }}
    --base_url {{ prisma_base_url }}
  args:
    chdir: "/path/to/scripts"  # optional: specify directory where script is stored
```

This method:

* Ensures your IP logic remains in one automation workflow
* Uses variable substitution to pass facts directly from previous Ansible tasks
* Keeps the Ansible structure clean and allows for modular script updates

### Example Output

```
Successfully assigned 10.10.10.45 to ION d3f123456789 interface wan0
```

You can expand this logic to:

* Loop over multiple interfaces
* Use interface metadata to dynamically determine which interface to configure
* Apply templates for routing profiles, BGP configuration, or NAT settings via additional API calls

This approach lets you bridge the gap between Ansible workflows and Prisma SD-WAN’s powerful API until a native collection is released.

```python
import argparse
import requests

parser = argparse.ArgumentParser(description="Assign IP to Prisma SD-WAN ION site")
parser.add_argument('--ip', required=True, help='IP address to assign')
parser.add_argument('--site', required=True, help='Prisma SD-WAN site ID')
parser.add_argument('--token', required=True, help='Bearer token for Prisma API')
parser.add_argument('--base_url', required=True, help='Prisma API base URL')
args = parser.parse_args()

headers = {
    "Authorization": f"Bearer {args.token}",
    "Content-Type": "application/json"
}

# Example: Assign IP to an interface on a WAN edge ION device
payload = {
    "ipv4_config": {
        "config_type": "STATIC",
        "static_config": {
            "ip": args.ip,
            "netmask": "255.255.255.0",
            "gateway": "10.10.10.1"
        }
    }
}

endpoint = f"{args.base_url}/sdwan/v2/sites/{args.site}/elements"

# Step 1: Get the ION device(s) at this site
response = requests.get(endpoint, headers=headers)
response.raise_for_status()
device_list = response.json().get('data', [])

if not device_list:
    raise ValueError("No ION devices found for site")

# Step 2: Apply IP config to the first ION device’s WAN interface (example only)
device_id = device_list[0]['id']
wan_config_endpoint = f"{args.base_url}/sdwan/v2/elements/{device_id}/interfaces"

# You may need to fetch and identify the right interface first
interfaces = requests.get(wan_config_endpoint, headers=headers).json().get('data', [])
wan_if = next((i for i in interfaces if i['name'] == 'wan0'), None)

if not wan_if:
    raise Exception("WAN interface 'wan0' not found")

# Apply configuration
apply_endpoint = f"{wan_config_endpoint}/{wan_if['id']}"
put_resp = requests.put(apply_endpoint, headers=headers, json=payload)
put_resp.raise_for_status()
print(f"Successfully assigned {args.ip} to ION {device_id} interface {wan_if['name']}")
```

---

## 5. Workflow Summary

This section outlines the complete step-by-step integration flow that ties Infoblox IPAM automation with Palo Alto Prisma SD-WAN device configuration via Ansible and an external Python script.

1. **Query Infoblox**:

   * Ansible uses the `nios_search` module to query the Infoblox database.
   * The search is refined using variables such as `site`, `function`, and `zone` passed to the playbook.
   * Subnet metadata (typically in the `comment` field or extensible attributes) is used to identify the correct network prefix.

2. **Get Next Available IP**:

   * Once the subnet is identified, `nios_next_ip` is used to request the next free address within that subnet.
   * The IP address is registered as a fact using `set_fact` for use later in the playbook.

3. **Create DNS A Record**:

   * The playbook uses the `nios_a_record` module to register a DNS A record for the newly assigned IP address.
   * This supports network traceability and name resolution for the ION device being deployed.

4. **Pass Variables to Python Script**:

   * The Ansible playbook calls the Python script using the `command` module.
   * Variables such as the assigned IP address, Prisma site ID, API base URL, and token are passed as CLI arguments.
   * These are mapped within the Python script via the `argparse` module.

5. **Push Configuration to Prisma SD-WAN**:

   * The Python script connects to the Prisma SD-WAN API.
   * It authenticates using the provided token and retrieves the list of ION devices at the target site.
   * It identifies the correct WAN interface (e.g., `wan0`) and pushes the static IP configuration via a `PUT` request.

### Example Workflow Output

```yaml
TASK [Set assigned IP fact]
ok: [localhost] => {
    "assigned_ip": "10.10.10.45"
}

TASK [Create DNS A record]
ok: [localhost] => {
    "changed": true,
    "msg": "Created A record for ion01.branch01.company.com"
}

TASK [Call Python script to configure Prisma SD-WAN]
ok: [localhost] => {
    "stdout": "Successfully assigned 10.10.10.45 to ION d3f123456789 interface wan0"
}
```

This modular workflow allows network engineers and automation teams to:

* Maintain Ansible as the primary orchestration layer
* Delegate non-Ansible-integrated tasks (like Prisma configuration) to Python
* Maintain clear variable passing and consistent traceability throughout the flow

Optional enhancements include:

* Adding rollback steps
* Creating audit logs
* Creating reverse DNS entries
* Validating IP assignment against routing policies or network access controls

1. **Query Infoblox** for subnet based on structured metadata (site, function, zone).
2. **Get next available IP** from selected subnet.
3. **Create DNS A record** using Ansible module.
4. **Pass variables** to a Python script that configures the IP on the Prisma SD-WAN device via API.
5. **Push configuration** to Prisma ION using a combination of API calls:

   * List site devices
   * List device interfaces
   * PUT updated IP config on selected interface

---

## 6. Benefits of Variable-Based Filtering

Using variables such as `site`, `function`, and `zone` to search Infoblox improves several aspects of your automation pipeline:

* **Modularity**: By externalizing inputs like site name or zone role, your playbooks become reusable for any number of deployments or locations.
* **Scalability**: Rather than hardcoding subnets into logic, you dynamically query based on business-defined parameters, allowing your workflow to grow across sites and environments.
* **Accuracy**: Narrowing a search with meaningful attributes like zone or purpose (e.g., "dmz" or "management") ensures you're not provisioning from the wrong network space.
* **Auditability and Tagging**: Adding tags to Infoblox objects makes it easier to trace when, where, and how an address was allocated.

### Example Metadata Tags in Infoblox Comments

These tags are embedded in the `comment` field on each subnet or network object and are matched using regex in the Ansible `nios_search` task:

```text
Comment = "site:branch01 function:wan-edge zone:dmz"
Comment = "site:nyc-office function:management zone:trusted"
Comment = "site:london-dc function:loopback zone:infra"
```

This method enables flexible and clear querying across a large inventory of subnets. You can filter dynamically with:

```yaml
filter:
  - "comment~=site:{{ site }}.*function:{{ function }}.*zone:{{ zone }}"
```

### Alternate Filter Strategies

1. **Using Extensible Attributes**: Infoblox supports structured metadata via extensible attributes (EA). These are more structured than free-text comments and can be queried more efficiently:

```yaml
filter:
  - "extattrs.site=={{ site }}"
  - "extattrs.function=={{ function }}"
```

This provides a more reliable search mechanism and allows better enforcement of naming and tagging consistency.

2. **Prefix-Based Mapping**: If you're aligning site codes with IP schema (e.g., `10.11.x.x` for branch11), you can parse IP ranges from Ansible variables:

```yaml
subnet_prefix: "10.{{ branch_id }}.0.0/16"
```

3. **Network Containers**: Infoblox can group networks in containers. For example, if all WAN prefixes are under `10.10.0.0/16`, you can search by parent container:

```yaml
nios_search:
  object: network
  filter:
    - "network_container~=10.10."
```

4. **Custom YAML Mapping File**: Define a local lookup table of site/function/zone to subnet. This is useful as a fallback or override:

```yaml
subnet_map:
  branch01:
    wan-edge: "10.10.10.0/24"
    loopback: "192.168.10.0/30"
```

Then use it with `set_fact` to pull the correct prefix if dynamic search fails.

### Best Practices for Filtering

* Always validate that exactly one subnet is returned; if not, fail the playbook with a clear error.
* Consider combining both comments and extensible attributes for backward compatibility.
* Keep all metadata case-consistent (e.g., lowercase for all values) to simplify regex logic.

By designing your filtering with reusable variables, your automation becomes more portable, less error-prone, and easier to audit and extend across diverse environments.

Using variables such as `site`, `function`, and `zone` to search Infoblox improves:

* **Modularity**: Easily reuse logic across different locations
* **Scalability**: Avoid hardcoding subnet details into playbooks
* **Accuracy**: Ensures the right address space is chosen for the right role and zone
* **Auditability**: Leverages Infoblox tagging for traceability

### Example Metadata Tags in Infoblox

```
Comment = "site:branch01 function:wan-edge zone:dmz"
Comment = "site:nyc-office function:management zone:trusted"
```

These tags enable precise subnet selection using Infoblox’s regex filter capabilities.

### Alternate Filter Options

* Use custom extensible attributes in Infoblox instead of comments
* Filter by DHCP scope names or network containers
* Return multiple subnets and prioritize based on size or hierarchy

---

## Notes and Tips

This section highlights additional implementation and security considerations to ensure your automation is robust and production-ready.

* **Prisma API Version**: This guide uses API version `v2`, aligned with [https://pan.dev/sdwan/api/](https://pan.dev/sdwan/api/). Be sure your tenant supports this version.

* **Scope and Permissions**: Ensure that the API token provided to the Python script has the necessary scopes to read site and element info, and update interface configurations. Lack of scope will result in 403 errors.

* **Ansible Vault**: Always use `ansible-vault` to encrypt and manage sensitive variables like Infoblox and Prisma credentials. Example usage:

  ```bash
  ansible-vault encrypt_string 'my_password' --name 'infoblox_password'
  ```

* **Error Handling**: The current Python script uses `raise_for_status()` for basic error catching. In production, consider adding retries, detailed logging, and output handling to capture API errors and audit results.

* **DNS Reversals**: While the guide creates forward A records, it’s advisable to also create reverse PTR records in Infoblox. This can be done using the `nios_ptr_record` module if reverse DNS zones are in place.

* **Extending Interface Logic**: If your ION devices use more than one WAN interface or require loopbacks and LAN configs, extend the script logic to:

  * Loop through all interfaces
  * Identify via name or metadata
  * Apply corresponding IP logic

* **Testing Without Impact**: You can test Infoblox logic and DNS creation without running the Prisma configuration by commenting out the final command task. This lets you validate IP allocation and DNS updates first.

* **Tagging Strategy**: Use consistent, searchable tags in subnet comments or Infoblox extensible attributes such as:

  ```
  site=branch01, function=wan-edge, zone=dmz
  environment=prod, service=vpn
  ```

  Tags should be predictable so Ansible regex filters return deterministic matches.

* **Performance Consideration**: If your Infoblox instance contains thousands of networks, pre-filter your search scope by using Infoblox extensible attributes or containers to reduce result set size.

* **Idempotency**: Ansible tasks that register facts and interact with Infoblox are idempotent when properly filtered. Ensure your filters match only one network to avoid runtime ambiguity.

* **Documentation and GitHub Use**: Consider splitting your solution into:

  * `README.md` for overview and how-to
  * `roles/infoblox/tasks/main.yml` for IP + DNS logic
  * `files/configure_prisma.py`
  * `group_vars/` for regional site-specific variables

This structure supports version control, collaboration, and reuse across teams. API version used: `v2` (as per [https://pan.dev/sdwan/api/](https://pan.dev/sdwan/api/))

* Ensure Prisma tenant has correct scopes assigned to API token.
* Use `ansible-vault` to secure sensitive credentials.
* Add logging and error handling in production script.
* Adjust Python logic if configuring multiple interfaces or needing DHCP instead of static.

---

Let me know if you’d like this packaged into a role or extended to manage DNS reverse records, DHCP scopes, or Prisma BGP configuration.

---

## Visual Workflow Diagram (ASCII)

Below is a high-level visual representation of the overall workflow that illustrates the integration flow between Ansible, Infoblox, and Prisma SD-WAN. This diagram includes each key step and helps clarify where logic, decision points, and automation boundaries occur.

```text
+-------------------------------+
|        Ansible Playbook      |
|-----------------------------|
| Loads vars.yml              |
| Runs tasks sequentially     |
+---------------+-------------+
                |
                v
+-------------------------------+
|  Query Infoblox using        |
|  nios_search with site/zone |
|  metadata (e.g. via comment) |
+---------------+-------------+
                |
                v
+-------------------------------+
| Validate subnet found       |
| Register subnet_prefix fact |
+---------------+-------------+
                |
                v
+-------------------------------+
| Get next available IP       |
| using nios_next_ip          |
| Register assigned_ip fact   |
+---------------+-------------+
                |
                v
+-------------------------------+
| Create DNS A record using   |
| nios_a_record (ion01.site)  |
+---------------+-------------+
                |
                v
+-------------------------------+
| Call external Python script |
| with IP, token, site_id     |
| via Ansible command module  |
+---------------+-------------+
                |
                v
+-------------------------------+
|     Python Script Logic     |
|-----------------------------|
| Parse arguments             |
| Build auth header           |
| Call Prisma API to get     |
|  site + ION device list     |
| Get WAN interface details   |
| Construct static IP payload |
| PUT to /interfaces endpoint |
+---------------+-------------+
                |
                v
+-------------------------------+
| Prisma SD-WAN Controller    |
| Accepts static IP config    |
| Applies to WAN interface    |
+-------------------------------+
```

### Legend

* **Ansible Logic**: top-down orchestration including Infoblox and Python
* **Infoblox API**: dynamic data lookup and IP lifecycle management
* **Python Script**: translates playbook values into Prisma-compatible API calls
* **Prisma SD-WAN**: cloud-native controller handling configuration application

This visual flow allows new users to follow the logic end-to-end, map each task to its functional boundary, and identify where they might extend, override, or debug the automation.

Below is a high-level visual representation of the overall workflow that illustrates the integration flow between Ansible, Infoblox, and Prisma SD-WAN:

```text
+------------------------+
|    Ansible Playbook    |
+----------+-------------+
           |
           v
+------------------------+
|   Query Infoblox for   |
| subnet via metadata    |
|  (site/function/zone)  |
+----------+-------------+
           |
           v
+------------------------+
|   Get next available   |
|         IP             |
+----------+-------------+
           |
           v
+------------------------+
|  Register DNS A record |
|     (Infoblox DNS)     |
+----------+-------------+
           |
           v
+------------------------+
|  Call Python script to |
|  configure Prisma ION  |
+----------+-------------+
           |
           v
+------------------------+
|  Prisma API authenticates|
| and fetches ION device  |
+----------+-------------+
           |
           v
+------------------------+
| Assign static IP to WAN|
|  interface via API PUT |
+------------------------+
```

This diagram helps summarize the logical steps of the automation workflow and provides a clear visualization of how each system interacts in sequence.
