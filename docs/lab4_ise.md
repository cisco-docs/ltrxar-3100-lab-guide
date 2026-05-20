# Lab 4 — ISE as Code

In this lab you will use the **ISE as Code** Terraform module to configure Cisco Identity Services Engine. You will define TrustSec Security Groups (SGTs), Security Group ACLs (SGACLs), the policy enforcement matrix, and register network devices as TrustSec enforcement points — all deployed through a GitLab CI/CD pipeline.

## Lab Objectives

After completing this lab, you will be able to:

- Describe the ISE as Code data model for TrustSec, network access, and network resources
- Understand the three-plane TrustSec model: classification, propagation, and enforcement
- Push the repository to GitLab and trigger the CI/CD pipeline
- Verify TrustSec policy in the ISE GUI

## TrustSec Background

TrustSec provides scalable policy enforcement using **Security Group Tags (SGTs)** rather than IP addresses. The end-to-end TrustSec flow has three planes:

| Plane | What it does | Configured in |
|---|---|---|
| **Classification** | Assigns an SGT to an endpoint or traffic flow | ISE (static IP-SGT mappings or 802.1X authz) |
| **Propagation** | Carries the SGT tag across the network | SD-WAN (`propagate_sgt: true`), SDA fabric |
| **Enforcement** | Applies SGACL policy at the egress point | ISE policy matrix + Catalyst Center fabric edge |

This lab configures all three planes in ISE. The SD-WAN propagation was already enabled in Lab 2 (`propagate_sgt: true`), and Catalyst Center fabric enforcement was enabled in Lab 3 (`group_based_policy_enforcement_enabled: true`).

## Repository Structure

```
ltrxar-3100-ise/
├── main.tf                       # Terraform entry point
├── data/
│   ├── trust_sec.nac.yaml        # SGTs, SGACLs, IP-SGT mappings, policy matrix
│   ├── network_access.nac.yaml   # Wired access policy sets and rules
│   └── network_resources.nac.yaml# Network devices (TrustSec enforcement points)
├── schemas/                      # JSON Schema for YAML validation
├── validation/                   # pytest-based semantic tests
└── .gitlab-ci.yml                # CI/CD pipeline definition
```

## Step 1: Connect to the Windows Workstation

If you have closed the RDP session, reconnect:

- **IP:** `198.18.133.10`
- **Username:** `admin`
- **Password:** `C1sco12345`

Open **Visual Studio Code** and open a new terminal: **Terminal → New Terminal**.

## Step 2: Clone the Repository

Clone the ISE as Code repository from GitHub:

```bash
git clone https://github.com/cisco-docs/ltrxar-3100-ise.git
cd ltrxar-3100-ise
```

Open the folder in VS Code: **File → Open Folder** → select `ltrxar-3100-ise`. Trust the workspace when prompted.

## Step 3: Explore the Terraform Entry Point

Open `main.tf`:

```hcl
terraform {
  required_providers {
    ise = {
      source  = "CiscoDevNet/ise"
      version = "0.2.14"
    }
  }
  backend "http" {}
}

module "ise" {
  source  = "netascode/nac-ise/ise"
  version = "0.2.2"

  yaml_directories = ["data/"]
}
```

The ISE module is the most minimal entry point of the four labs — there are no feature flags. All configuration scope is driven by which data files are present in `data/`. The `backend "http" {}` block is configured by the pipeline to store Terraform state in GitLab.

## Step 4: Explore the Data Model

### TrustSec Policy (`trust_sec.nac.yaml`)

Open `data/trust_sec.nac.yaml`. This is the heart of the lab's security policy.

#### Security Groups (SGTs)

```yaml
ise:
  trust_sec:
    security_groups:
      - name: Employees
        description: Internal user endpoints (laptops, workstations)
        value: 10
      - name: Servers
        description: Production servers
        value: 20
      - name: Cameras
        description: IoT cameras and surveillance devices
        value: 30
```

SGT values (10, 20, 30) are 16-bit tags carried in the Ethernet frame (CMD header) or across the SD-WAN overlay. These values must be consistent across all enforcement domains — ISE, Catalyst Center, and SD-WAN all reference the same numeric values.

#### Security Group ACLs (SGACLs)

```yaml
    security_group_acls:
      - name: PERMIT-ANY
        description: Permit all IP traffic between SGTs
        ip_version: IP_AGNOSTIC
        acl_content: |
          permit ip

      - name: PERMIT-CAMERA-STREAM
        description: Allow Employees to view camera streams (RTSP + HTTP)
        ip_version: IPV4
        acl_content: |
          permit tcp dst eq 554
          permit tcp dst eq 8080
          deny ip

      - name: DENY-IP-LOG
        description: Deny everything and log
        ip_version: IP_AGNOSTIC
        acl_content: |
          deny ip log
```

SGACLs are written in Cisco ACL syntax and applied between SGT pairs at egress. They can reference protocols and ports for fine-grained control.

#### Static IP-SGT Mappings (Classification)

```yaml
    ip_sgt_mappings:
      - name: laptop
        host_ip: 192.168.100.10
        sgt: Employees
        deploy_type: ALL
      - name: server
        host_ip: 192.168.110.10
        sgt: Servers
        deploy_type: ALL
      - name: camera
        host_ip: 192.168.200.10
        sgt: Cameras
        deploy_type: ALL
```

Static mappings assign an SGT to a specific IP address. In production, SGTs are typically assigned dynamically via 802.1X. In this lab, static mappings are used because the CML endpoint stubs do not have a real 802.1X supplicant.

`deploy_type: ALL` means ISE will push this mapping to all registered TrustSec-capable network devices.

#### Policy Matrix (Enforcement)

```yaml
    matrix_entries:
      - source_sgt: Employees
        destination_sgt: Servers
        sgacl_name: PERMIT-ANY
        rule_status: ENABLED

      - source_sgt: Employees
        destination_sgt: Cameras
        sgacl_name: PERMIT-CAMERA-STREAM
        rule_status: ENABLED

      - source_sgt: Cameras
        destination_sgt: Servers
        sgacl_name: DENY-IP-LOG
        rule_status: ENABLED

      - source_sgt: Cameras
        destination_sgt: Employees
        sgacl_name: DENY-IP-LOG
        rule_status: ENABLED
```

The policy matrix specifies which SGACL applies for each source SGT → destination SGT pair:

|  | → Employees | → Servers | → Cameras |
|---|---|---|---|
| **Employees →** | (default) | `PERMIT-ANY` | `PERMIT-CAMERA-STREAM` |
| **Servers →** | (default) | (default) | (default) |
| **Cameras →** | `DENY-IP-LOG` | `DENY-IP-LOG` | (default) |

> **Security intent:** Cameras can stream to Employees (RTSP/HTTP only) but cannot initiate other connections. Cameras cannot reach Servers at all. This models a common IoT isolation pattern.

### Network Resources (`network_resources.nac.yaml`)

Open `data/network_resources.nac.yaml`. This file registers the SDA fabric devices as TrustSec enforcement points in ISE.

```yaml
ise:
  network_resources:
    network_devices:
      - name: BORDER
        description: SDA Fabric A border + control plane node
        ips:
          - ip: 198.18.130.10
        radius:
          shared_secret: __PLACEHOLDER_SHARED_SECRET__
        trust_sec:
          device_id: BORDER
          device_password: __PLACEHOLDER_DEVICE_PASSWORD__
          send_configuration_to_device: true
          send_configuration_to_device_using: ENABLE_USING_COA
          download_sgacl_lists_every_x_seconds: 60
```

**Key points:**
- `send_configuration_to_device_using: ENABLE_USING_COA` means ISE pushes policy updates via RADIUS Change of Authorization — without waiting for the device's next scheduled refresh
- The `__PLACEHOLDER__` values are secrets replaced at pipeline run time via GitLab CI/CD variables

> **Note:** In a production deployment, these secrets are injected from a secrets manager (HashiCorp Vault, CyberArk) as masked CI/CD variables — never committed in plaintext.

## Step 5: Create a GitLab Project

Open a browser and navigate to the GitLab instance: `http://198.18.128.50`

Log in with `labuser` / `C1sco12345`.

Create a new project:

1. Click **New project → Create blank project**
2. Set the **Project name** to `ltrxar-3100-ise`
3. Set the **Namespace** to `md-as-code`
4. Set **Visibility level** to **Private**
5. **Uncheck** "Initialize repository with a README"
6. Click **Create project**

> **Note:** ISE credentials (`ISE_URL`, `ISE_USERNAME`, `ISE_PASSWORD`) and the TrustSec device secrets are pre-configured at the `md-as-code` group level.

## Step 6: Push to GitLab

In the VS Code terminal:

```bash
git remote add gitlab http://198.18.128.50/md-as-code/ltrxar-3100-ise.git
git push gitlab main
```

When prompted, enter `labuser` / `C1sco12345`.

> **Note:** The ISE repository uses `main` as the default branch name.

## Step 7: Monitor the Pipeline

In GitLab, navigate to **Build → Pipelines** in your `ltrxar-3100-ise` project.

The ISE pipeline has a simplified set of stages compared to the ACI, SD-WAN, and Catalyst Center pipelines:

| Stage | Job | What it does |
|---|---|---|
| **validate** | `validate` | Checks Terraform HCL formatting |
| **plan** | `plan` | `terraform plan` — shows all ISE resources to be created |
| **deploy** | `deploy` | **Manual trigger** — `terraform apply` pushes configuration to ISE |
| **cleanup** | `cleanup` | **Manual trigger** — `terraform destroy` (for lab reset) |

> **Note:** The ISE pipeline uses a pinned Docker image (`danischm/nac:0.1.5`) for stability with the ISE provider version. It also includes a DNS workaround in `before_script` and disables IPv6 via `GODEBUG: net.ipv6=off` to address known ISE provider compatibility issues.

Wait for **validate** and **plan** to complete. Review the plan output.

## Step 8: Trigger the Deploy and Verify

Click the **play button (▶)** next to the `deploy` job.

ISE API calls take several seconds per resource; expect 3–5 minutes total.

**Verify in ISE:**

Open a browser and navigate to ISE: `https://198.18.133.30`

Log in with `admin` / `C1sco12345`.

1. Navigate to **Work Centers → TrustSec → Components → Security Groups**. Confirm `Employees` (SGT 10), `Servers` (SGT 20), and `Cameras` (SGT 30) exist.
2. Navigate to **Work Centers → TrustSec → Components → Security Group ACLs**. Click on `PERMIT-CAMERA-STREAM` and confirm the ACL content.
3. Navigate to **Work Centers → TrustSec → TrustSec Policy → Egress Policy → Matrix**. Confirm the four matrix entries are in place.
4. Navigate to **Administration → Network Resources → Network Devices**. Confirm `BORDER`, `FIAB`, and the cEdge devices are registered.

![ISE TrustSec Matrix](./assets/ise_matrix.png)

## Understanding the CI/CD Pipeline

Open `.gitlab-ci.yml`. The ISE pipeline has a few notable characteristics compared to the other labs:

```yaml
image:
  name: danischm/nac:0.1.5   # Pinned version for ISE provider compatibility
  pull_policy: if-not-present

stages:
  - validate
  - plan
  - deploy
  - cleanup

variables:
  GODEBUG: net.ipv6=off        # Disables IPv6 (ISE provider limitation)

before_script:
  - cp /etc/resolv.conf /tmp/resolv.conf
  - sed -i 's/8.8.8.8/1.1.1.1/g' /tmp/resolv.conf
  - cp /tmp/resolv.conf /etc/resolv.conf
```

The `before_script` DNS patch replaces the Google DNS resolver with Cloudflare — a workaround for DNS resolution issues in some lab environments where `8.8.8.8` is unreachable.

The `cleanup` stage is a **manually triggered** `terraform destroy` — this allows lab environments to be reset without requiring direct access to the Terraform state or ISE:

```yaml
cleanup:
  stage: cleanup
  script:
    - terraform init -input=false
    - terraform destroy -input=false -auto-approve
  when: manual
  only:
    - main
```

## Day-2 Operation: Modify the Policy Matrix

Simulate a policy change by adding a new restriction: prevent servers from initiating connections to employee endpoints.

Open `data/trust_sec.nac.yaml` and add a new matrix entry:

```yaml
    matrix_entries:
      # ... existing entries ...
      - source_sgt: Servers
        destination_sgt: Employees
        sgacl_name: DENY-IP-LOG
        rule_status: ENABLED
```

Commit and push:

```bash
git add data/trust_sec.nac.yaml
git commit -m "Restrict Servers from initiating connections to Employees"
git push gitlab main
```

A new pipeline run starts automatically. The **plan** stage shows exactly one new resource. Trigger the **deploy** job and verify the new matrix cell in the ISE GUI under **TrustSec Policy → Egress Policy → Matrix**.

## Summary

In this lab you:

- Defined TrustSec SGTs, SGACLs, static IP-SGT mappings, and the policy matrix in YAML
- Registered fabric devices as TrustSec enforcement points in ISE
- Created a GitLab project and deployed through the CI/CD pipeline
- Verified the complete TrustSec policy in ISE
- Made a day-2 policy change through the pipeline

**Continue to [Lab 5 — Multi-Domain Integration](lab5_multidomain.md).**
