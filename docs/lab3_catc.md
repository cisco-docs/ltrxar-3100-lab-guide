# Lab 3 — Catalyst Center as Code

In this lab you will use the **Catalyst Center as Code** Terraform module to provision a full SDA (Software-Defined Access) fabric. You will configure the site hierarchy, IP pools, AAA settings, fabric sites, L3 Virtual Networks, anycast gateways, and assign devices to fabric roles — all from YAML data files deployed through a GitLab CI/CD pipeline.

## Lab Objectives

After completing this lab, you will be able to:

- Navigate the Catalyst Center as Code data model for sites, network settings, fabric, and devices
- Understand the multi-fabric pattern (two distinct fabric sites managed by one Catalyst Center)
- Push the repository to GitLab and trigger the CI/CD pipeline
- Verify SDA fabric provisioning in the Catalyst Center GUI

## Lab Topology

This lab provisions two SDA fabrics managed by a single Catalyst Center instance:

| Fabric | Site | Devices | L3 VN | IP Pools |
|---|---|---|---|---|
| **Fabric A** | `Global/USA/Nevada/Las Vegas/Bld A` | BORDER, SDA-EDGE-01, SDA-EDGE-02 | Campus | EmployeesPool, ServersPool |
| **Fabric B** | `Global/USA/Nevada/Las Vegas/Bld B` | FIAB (all roles) | IoT | IoTPool |

Both fabrics share city-level AAA settings (ISE 3.3) and draw IP pool reservations from the same parent `Overlay` pool.

## Repository Structure

```
ltrxar-3100-catalystcenter/
├── main.tf                         # Terraform entry point
├── data/
│   ├── sites.nac.yaml              # Site hierarchy (areas, buildings, floors)
│   ├── network_settings.nac.yaml   # IP pools, AAA settings
│   ├── fabric.nac.yaml             # Fabric sites, L3 VNs, anycast gateways
│   └── devices.nac.yaml            # Device inventory and fabric role assignments
├── defaults/                       # Default values for the module
├── schemas/                        # JSON Schema for YAML validation
├── templates/                      # Jinja2 test templates for post-deploy checks
├── validation/                     # pytest-based semantic tests
└── .gitlab-ci.yml                  # CI/CD pipeline definition
```

## Step 1: Connect to the Windows Workstation

If you have closed the RDP session, reconnect:

- **IP:** `198.18.133.10`
- **Username:** `admin`
- **Password:** `C1sco12345`

Open **Visual Studio Code** and open a new terminal: **Terminal → New Terminal**.

## Step 2: Clone the Repository

Clone the Catalyst Center as Code repository from GitHub:

```bash
git clone https://github.com/cisco-docs/ltrxar-3100-catalystcenter.git
cd ltrxar-3100-catalystcenter
```

Open the folder in VS Code: **File → Open Folder** → select `ltrxar-3100-catalystcenter`. Trust the workspace when prompted.

## Step 3: Explore the Terraform Entry Point

Open `main.tf`:

```hcl
terraform {
  required_providers {
    catalystcenter = {
      source  = "CiscoDevNet/catalystcenter"
      version = "0.4.7"
    }
  }
  backend "http" {}
}

module "catalyst_center" {
  source  = "netascode/nac-catalystcenter/catalystcenter"
  version = "0.3.0"

  yaml_directories      = ["data/"]
  templates_directories = ["data/templates/"]

  use_bulk_api = true
}
```

**Key observations:**
- `use_bulk_api = true` — enables Catalyst Center's bulk provisioning APIs, which significantly speeds up large deployments by batching API calls
- `max_timeout = 600` — Catalyst Center operations like fabric provisioning can take several minutes; a high timeout prevents premature failures
- `backend "http" {}` — Terraform state is stored in GitLab (configured automatically by the pipeline)
- Provider credentials (`CC_URL`, `CC_USERNAME`, `CC_PASSWORD`) are injected by the pipeline as environment variables

## Step 4: Explore the Data Model

### Site Hierarchy (`sites.nac.yaml`)

Open `data/sites.nac.yaml`. This file defines the complete Catalyst Center site hierarchy from Global down to individual floors.

```yaml
catalyst_center:
  sites:
    areas:
      - name: Global
      - name: USA
        parent_name: Global
      - name: Nevada
        parent_name: Global/USA
      - name: Las Vegas
        parent_name: Global/USA/Nevada
        network_settings:
          aaa_servers: AAA_Settings      # ISE referenced by name
        ip_pools_reservations:
          - EmployeesPool
          - ServersPool
          - IoTPool
    buildings:
      - name: Bld A
        latitude: 36.1699
        longitude: -115.1398
        country: United States
        parent_name: Global/USA/Nevada/Las Vegas
      - name: Bld B
        latitude: 36.1699
        longitude: -115.1398
        country: United States
        parent_name: Global/USA/Nevada/Las Vegas
    floors:
      - name: FLOOR_1
        floor_number: 1
        parent_name: Global/USA/Nevada/Las Vegas/Bld A
```

**Key points:**
- The `ip_pools_reservations` at the `Las Vegas` level makes the pools available to **both** buildings — both Fabric A and Fabric B can reserve from the same parent
- Site paths use `/`-separated names (e.g., `Global/USA/Nevada/Las Vegas`) — these are used as references throughout all data files
- `aaa_servers: AAA_Settings` references the AAA server defined in `network_settings.nac.yaml`

### Network Settings (`network_settings.nac.yaml`)

Open `data/network_settings.nac.yaml`. This file defines global IP pools and AAA server settings.

```yaml
catalyst_center:
  network_settings:
    aaa_servers:
      - name: AAA_Settings
        client_and_endpoint_aaa:
          server_type: ISE
          protocol: RADIUS
          primary_ip: 198.18.133.30   # ISE 3.3

    ip_pools:
      - name: Overlay
        ip_address_space: IPv4
        ip_pool_cidr: 192.168.0.0/16
        ip_pools_reservations:
          - name: EmployeesPool
            prefix_length: 24
            subnet: 192.168.100.0
            gateway: 192.168.100.1
            dns_servers:
              - 198.18.130.11
            dhcp_servers:
              - 198.18.130.11
          - name: ServersPool
            prefix_length: 24
            subnet: 192.168.110.0
            gateway: 192.168.110.1
          - name: IoTPool
            prefix_length: 24
            subnet: 192.168.200.0
            gateway: 192.168.200.1
```

**Key points:**
- `Overlay` is the parent pool (`192.168.0.0/16`); the three child reservations are carved from it
- ISE (`198.18.133.30`) is configured as the RADIUS server for both client AAA (802.1X) and endpoint AAA (profiling)

### Fabric Configuration (`fabric.nac.yaml`)

Open `data/fabric.nac.yaml`. This is the most complex data file — it defines L3 Virtual Networks, fabric sites, and anycast gateways.

```yaml
catalyst_center:
  fabric:
    l3_virtual_networks:
      - name: Campus
      - name: IoT

    fabric_sites:
      # ─── Fabric A (Bld A) ───
      - name: Global/USA/Nevada/Las Vegas/Bld A
        pub_sub_enabled: true
        l3_virtual_networks:
          - Campus
        anycast_gateways:
          - ip_pool_name: EmployeesPool
            vlan_name: VLAN_EMPLOYEES
            traffic_type: DATA
            l3_virtual_network: Campus
            security_group_name: Employees      # SGT from ISE
            group_based_policy_enforcement_enabled: true
          - ip_pool_name: ServersPool
            vlan_name: VLAN_SERVERS
            traffic_type: DATA
            l3_virtual_network: Campus
            security_group_name: Servers
            group_based_policy_enforcement_enabled: true

      # ─── Fabric B (Bld B) ───
      - name: Global/USA/Nevada/Las Vegas/Bld B
        pub_sub_enabled: true
        l3_virtual_networks:
          - IoT
        anycast_gateways:
          - ip_pool_name: IoTPool
            vlan_name: VLAN_CAMERAS
            traffic_type: DATA
            l3_virtual_network: IoT
            security_group_name: Cameras
            group_based_policy_enforcement_enabled: true
```

**Key points:**
- `security_group_name: Employees` links the anycast gateway to the ISE TrustSec SGT you will configure in Lab 4 — Catalyst Center downloads the SGT-to-VLAN binding from ISE and programs it on fabric edge switches
- `group_based_policy_enforcement_enabled: true` enables fabric edge switches to enforce the ISE TrustSec policy matrix locally, without forwarding traffic to ISE for per-flow policy lookup
- The `Campus` VN will be the target of the L3 handoff to SD-WAN in Lab 5

### Device Inventory (`devices.nac.yaml`)

Open `data/devices.nac.yaml`. This file adds devices to Catalyst Center inventory and assigns them to fabric roles.

```yaml
catalyst_center:
  inventory:
    devices:
      - name: SDA-EDGE-01
        device_ip: 198.18.130.1
        pid: C9KV-UADP-8P
        site: Global/USA/Nevada/Las Vegas/Bld A
        fabric_site: Global/USA/Nevada/Las Vegas/Bld A
        fabric_roles:
          - EDGE_NODE
        port_assignments:
          - interface_name: GigabitEthernet1/0/2
            connected_device_type: USER_DEVICE
            data_vlan_name: VLAN_EMPLOYEES
            authenticate_template_name: "No Authentication"

      - name: FIAB
        device_ip: 198.18.130.11
        site: Global/USA/Nevada/Las Vegas/Bld B
        fabric_site: Global/USA/Nevada/Las Vegas/Bld B
        fabric_roles:
          - BORDER_NODE
          - CONTROL_PLANE_NODE
          - EDGE_NODE
```

**Key points:**
- `FIAB` (Fabric-in-a-Box) runs all three fabric roles on a single device — Border, Control Plane, and Edge — which is the standard simulation pattern for campus fabrics
- `BORDER` (Fabric A border) is defined in the `ltrxar-3100-multidomain` repository — its configuration depends on SD-WAN handoff data defined in Lab 5
- `port_assignments` map physical interfaces to VLANs — Catalyst Center pushes the access port configuration to the edge switch

## Step 5: Create a GitLab Project

Open a browser and navigate to the GitLab instance: `http://198.18.128.50`

Log in with `labuser` / `C1sco12345`.

Create a new project:

1. Click **New project → Create blank project**
2. Set the **Project name** to `ltrxar-3100-catalystcenter`
3. Set the **Namespace** to `md-as-code`
4. Set **Visibility level** to **Private**
5. **Uncheck** "Initialize repository with a README"
6. Click **Create project**

> **Note:** Catalyst Center credentials (`CC_URL`, `CC_USERNAME`, `CC_PASSWORD`) are pre-configured at the `md-as-code` group level.

## Step 6: Push to GitLab

In the VS Code terminal:

```bash
git remote add gitlab http://198.18.128.50/md-as-code/ltrxar-3100-catalystcenter.git
git push gitlab master
```

When prompted, enter `labuser` / `C1sco12345`.

## Step 7: Monitor the Pipeline

In GitLab, navigate to **Build → Pipelines** in your `ltrxar-3100-catalystcenter` project.

The pipeline runs through the standard stages:

| Stage | Job | What it does |
|---|---|---|
| **validate** | `validate` | Checks HCL formatting + validates YAML against Catalyst Center JSON Schema |
| **plan** | `plan` | `terraform plan` — shows all site, fabric, and device resources to be created |
| **deploy** | `deploy` | **Manual trigger** — `terraform apply` pushes configuration to Catalyst Center |
| **test** | `test` | `nac-test` verifies deployed state against Catalyst Center API |
| **notify** | `success` / `failure` | Sends Webex notification |

> **Note:** The `plan` stage downloads the `nac-catalystcenter` module from the Terraform registry. This may take 2–3 minutes on the first run.

Wait for **validate** and **plan** to complete. Review the plan — you should see resources for the full site hierarchy, IP pools, fabric sites, VNs, anycast gateways, and device assignments.

## Step 8: Trigger the Deploy and Verify

Click the **play button (▶)** next to the `deploy` job.

Catalyst Center fabric provisioning is long-running. The deploy job may take **10–15 minutes** to complete. This is expected — Catalyst Center must:
1. Create the site hierarchy
2. Configure IP pools and AAA settings
3. Provision fabric sites and L3 VNs
4. Create anycast gateways and push configs to fabric switches
5. Assign devices to inventory and fabric roles

Once the deploy job turns green, the **test** stage runs automatically.

**Verify in Catalyst Center:**

Open a browser and navigate to: `https://198.18.129.100`

Log in with `admin` / `C1sco12345`.

1. Navigate to **Design → Network Hierarchy**. Confirm `Bld A` and `Bld B` exist under `Global/USA/Nevada/Las Vegas`.
2. Navigate to **Design → Network Settings → IP Address Pools** for `Las Vegas`. Confirm `EmployeesPool`, `ServersPool`, and `IoTPool` are visible.
3. Navigate to **Provision → Fabric Sites**. Confirm `Bld A` and `Bld B` appear.
4. Click `Bld A` → **Virtual Networks**. Confirm the `Campus` VN with `VLAN_EMPLOYEES` and `VLAN_SERVERS` anycast gateways.
5. Navigate to **Provision → Inventory**. Confirm devices appear with their fabric role assignments.

![Catalyst Center Fabric](./assets/catc_fabric.png)

## Understanding the CI/CD Pipeline

Open `.gitlab-ci.yml`. The Catalyst Center pipeline follows the same structure as the previous labs, with variables specific to Catalyst Center:

```yaml
variables:
  CC_USERNAME: ...
  CC_PASSWORD: ...
  CC_URL: ...
  TF_HTTP_ADDRESS: "${GITLAB_API_URL}/projects/${CI_PROJECT_ID}/terraform/state/tfstate"
```

The `test` stage uses `nac-test` to verify the deployed configuration against the live Catalyst Center instance:

```yaml
test:
  script:
    - nac-test -d ./data -d ./defaults
        -t ./templates/catalyst_center/test
        -o ./tests/results/catalyst_center
```

The `destroy` stage is triggered by a Git tag matching the pattern `cleanup-catc-*` — this is a safety mechanism that prevents accidental teardown while still allowing automated cleanup for lab reset:

```yaml
destroy:
  rules:
    - if: "$CI_COMMIT_TAG =~ /^cleanup-catc/"
      when: always
```

## Summary

In this lab you:

- Cloned the Catalyst Center as Code repository and explored the four-file data model structure
- Understood how site hierarchy, IP pools, fabric configuration, and device inventory interrelate
- Saw how SGT references (`security_group_name`) link Catalyst Center fabric policy to ISE TrustSec
- Created a GitLab project, pushed the repository, and deployed through the CI/CD pipeline
- Verified the two-fabric SDA deployment in Catalyst Center

**Continue to [Lab 4 — ISE as Code](lab4_ise.md).**
