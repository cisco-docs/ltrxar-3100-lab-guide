# Lab 5 — Multi-Domain Integration

In this lab you'll bring together the work from Labs 1 through 4 and deploy the cross-domain integration that connects the ACI fabric, SD-WAN overlay, and SDA campus into a unified policy domain. A single GitLab repository orchestrates child CI/CD pipelines that configure the L3 handoffs between fabric borders and SD-WAN edges in the correct dependency order.

!!! warning "Prerequisite"
    Complete Labs 1 through 4 before starting this lab.

## Lab Objectives

After completing this lab, you'll be able to:

- Understand the multi-domain integration architecture (ACI ↔ SD-WAN ↔ Campus)
- Deploy L3 handoffs between SDA border devices and SD-WAN edges from YAML
- Deploy SD-WAN device templates that compose feature templates into a complete device configuration
- Explain how the orchestrator pipeline coordinates across Catalyst Center and SD-WAN domains
- Verify cross-domain reachability and SGT propagation

## End-to-End Architecture

```
  ┌─────────────────────┐          ┌──────────────────────┐
  │   ACI Data Center   │          │  Campus SDA (CATC)   │
  │                     │          │                      │
  │  PROD Tenant        │          │  Fabric A (Bld A)    │
  │  BD-Servers         │          │  Campus VN           │
  │  BD-Web             │          │  SGT IT_Admin (40)   │
  │  L3OUT-SDWAN-PROD   │          │  SGT Servers (20)    │
  │  LEAF102 Gi1/1      │          │                      │
  │  VLAN 3010          │          │  Fabric B (Bld B)    │
  │  10.100.10.1/30     │          │  IoT VN              │
  │  BGP AS 65100       │          │  SGT Cameras (30)    │
  └────────┬────────────┘          └──────────┬───────────┘
           │ eBGP (APIC L3Out)                │ eBGP (CATC border handoff)
           │                                  │ BORDER AS 65001 / FIAB AS 65002
           ▼                                  ▼
  ┌────────────────────────────────────────────────────────┐
  │                  SD-WAN Overlay (VPN 30)               │
  │              Transit AS 65200                          │
  │                                                        │
  │  C-EDGE-01 (Site 1)        C-EDGE-02 (Site 2)          │
  │  VLAN 3050 ↔ FIAB          VLAN 3010 ↔ BORDER          │
  │  10.100.50.2/30            10.100.30.2/30              │
  └────────────────────────────────────────────────────────┘
```

**SGT propagation path:**
- ISE assigns SGT (40/20/30) via static IP-SGT mapping
- Catalyst Center programs SGT-to-VLAN bindings on fabric edge switches
- SGT is carried inline (CMD header) through the SDA fabric
- SD-WAN cEdge preserves SGT across the overlay (`propagate_sgt: true`)
- ACI L3Out receives BGP routes from SD-WAN (peer Idle in simulator — expected)

## Repository Structure

The multi-domain repository acts as an orchestrator. It doesn't contain the data files themselves. Instead, it triggers the pipelines of the parent repositories where the multidomain-specific data files reside.

```
ltrxar-3100-multidomain/
└── .gitlab-ci.yml    # Orchestrator pipeline (triggers parent project pipelines)

Multidomain data files in the parent repositories:
  ltrxar-3100-sdwan/data/
  ├── multidomain_edge_feature_templates.nac.yaml  # Handoff sub-interface + BGP templates
  ├── multidomain_edge_device_templates.nac.yaml   # Device template composing all feature templates
  └── multidomain_sites.nac.yaml                   # cEdge site definitions + device variables

  ltrxar-3100-catalystcenter/backup_data/
  ├── multidomain_fabric.nac.yaml    # Fabric sites, L3 VNs, anycast gateways, L3 handoffs
  └── multidomain_devices.nac.yaml   # Device inventory and fabric role assignments
```

## Step 1: Connect to the Windows Workstation

If you've closed the RDP session, reconnect:

- **IP:** `198.18.133.10`
- **Username:** `admin`
- **Password:** `C1sco12345`

Open **Visual Studio Code** and open a new terminal: **Terminal → New Terminal**.

## Step 2: Clone the Repository

Clone the multi-domain repository from GitHub:

```bash
git clone https://github.com/cisco-docs/ltrxar-3100-multidomain.git
cd ltrxar-3100-multidomain
```

Open the folder in VS Code: **File → Open Folder** → select `ltrxar-3100-multidomain`. Trust the workspace when prompted.

## Step 3: Explore the Orchestrator Pipeline

Open `.gitlab-ci.yml` at the root of the repository. This is the **orchestrator**. It doesn't run Terraform itself. Instead, it uses **multi-project triggers** to start the pipelines in the `sdwan` and `catc` projects in the correct sequence:

```yaml
stages:
  - deploy-sdwan
  - deploy-catc

deploy_sdwan:
  stage: deploy-sdwan
  trigger:
    project: md-as-code/sdwan
    branch: master
    strategy: depend
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: always

deploy_catc:
  stage: deploy-catc
  trigger:
    project: md-as-code/catc
    branch: main
    strategy: depend
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: always
```

**Key design decisions:**

- **`strategy: depend`** — the parent pipeline waits for each child pipeline to finish and inherits its exit status.
- **SD-WAN runs first** — cEdge handoff sub-interfaces and BGP come up first. BGP stays Idle/Active toward the SDA side until CATC runs. This is safe and ensures no blast radius before the campus is ready.
- **Multi-project trigger** — this pipeline triggers the existing `md-as-code/sdwan` and `md-as-code/catc` projects directly. The multidomain data files must be present in those projects' directories before the trigger fires.

## Step 4: Activate the Multidomain Data Files

Before pushing the multidomain repository, you need to ensure the multidomain-specific YAML files are in the active `data/` directory in both the `sdwan` and `catc` repositories.

**In the SD-WAN repository (`ltrxar-3100-sdwan`):**

Open the `ltrxar-3100-sdwan` folder in VS Code. The multidomain files were placed in the `data/` directory during setup. Verify they are present:

```bash
ls data/multidomain_*.nac.yaml
```

Expected output:
```
data/multidomain_edge_device_templates.nac.yaml
data/multidomain_edge_feature_templates.nac.yaml
data/multidomain_sites.nac.yaml
```

**In the Catalyst Center repository (`ltrxar-3100-catalystcenter`):**

Open the `ltrxar-3100-catalystcenter` folder in VS Code. Move the multidomain files from `backup_data/` to `data/`:

```bash
cp backup_data/multidomain_fabric.nac.yaml data/
cp backup_data/multidomain_devices.nac.yaml data/
```

Commit and push the changes to the catc GitLab project:

```bash
git add data/multidomain_fabric.nac.yaml data/multidomain_devices.nac.yaml
git commit -m "Activate multidomain fabric and device configuration"
git push gitlab main
```

!!! important
    Wait for the Catalyst Center and SDWAN pipelines to complete its **validate** and **plan** stages before proceeding. Don't trigger the deploy yet. The multidomain orchestrator will handle the deployment sequence.

## Step 5: Explore the Catalyst Center Multi-Domain Data

Open `data/multidomain_fabric.nac.yaml` in the `ltrxar-3100-catalystcenter` repository.

```yaml
catalyst_center:
  fabric:
    l3_virtual_networks:
      - name: Campus
      - name: IoT
    transits:
      - name: BGP-SDWAN
        type: IP_BASED_TRANSIT
        routing_protocol_name: BGP
        autonomous_system_number: 65200
    # ... anycast gateways with security_group_name: IT_Admin, Servers, Cameras ...
    border_devices:
      - name: BORDER.cisco.eu
        local_autonomous_system_number: 65001
        l3_handoffs:
          - name: BGP-SDWAN
            interfaces:
              - name: GigabitEthernet1/0/24
                virtual_networks:
                  - name: Campus
                    vlan: 3010
                    local_ip_address: 10.100.30.1/30
                    peer_ip_address: 10.100.30.2/30
      - name: FIAB.cisco.eu
        local_autonomous_system_number: 65002
        l3_handoffs:
          - name: BGP-SDWAN
            interfaces:
              - name: GigabitEthernet1/0/24
                virtual_networks:
                  - name: IoT
                    vlan: 3050
                    local_ip_address: 10.100.50.1/30
                    peer_ip_address: 10.100.50.2/30
```

**Key points:**
- Device names use FQDN: `BORDER.cisco.eu` and `FIAB.cisco.eu`.
- The anycast gateways use SGT names like `IT_Admin` (value 40).
- The L3 handoff uses BGP AS 65200 for the SD-WAN transit.

The `data/multidomain_devices.nac.yaml` file lists the four devices: `BORDER.cisco.eu`, `EDGE01.cisco.eu`, `EDGE02.cisco.eu`, and `FIAB.cisco.eu`, all with `state: PROVISION`.

## Step 6: Explore the SD-WAN Multi-Domain Data

Open the multidomain files in the `ltrxar-3100-sdwan` repository.

The device template `DT-HUB-C8000V-MULTIDOMAIN` in `multidomain_edge_device_templates.nac.yaml` composes the base feature templates with new handoff templates:
- `FT-ETH3-SUBIF-VPN30-HANDOFF` — dot1q sub-interface on Gi3 with CTS inline tagging (`sgt_propagation: true`, `sgt_trusted: true`).
- `FT-BGP-VPN30-HANDOFF` — EBGP template for VPN 30 handoff (AS 65200).

In `multidomain_sites.nac.yaml`, you'll find the site definitions:

- **Site 1** — C-EDGE-01: faces Fabric B (`FIAB.cisco.eu`), sub-interface Gi3.3050, IP 10.100.50.2/30, BGP peer 10.100.50.1 AS 65002.
- **Site 2** — C-EDGE-02: faces Fabric A (`BORDER.cisco.eu`), sub-interface Gi3.3010, IP 10.100.30.2/30, BGP peer 10.100.30.1 AS 65001.

## Step 7: Create a GitLab Project

Open a browser and navigate to the GitLab instance: `http://198.18.128.50`

Log in with `labuser` / `C1sco12345`.

Create a new project:

1. Click **New project → Create blank project**
2. Set the **Project name** to `ltrxar-3100-multidomain`
3. Set the **Namespace** to `md-as-code`
4. Set **Visibility level** to **Private**
5. **Uncheck** "Initialize repository with a README"
6. Click **Create project**

## Step 8: Push to GitLab

In the VS Code terminal within the `ltrxar-3100-multidomain` folder:

```bash
git remote add gitlab http://198.18.128.50/md-as-code/ltrxar-3100-multidomain.git
git push gitlab master
```

When prompted, enter `labuser` / `C1sco12345`.

## Step 9: Monitor and Trigger the Orchestrator Pipeline

In GitLab, navigate to **Build → Pipelines** in your `ltrxar-3100-multidomain` project.

The orchestrator pipeline shows two stages: `deploy-sdwan` followed by `deploy-catc`. Each stage triggers the corresponding parent project pipeline.

1. The orchestrator pipeline starts automatically when you push to `master`.
2. The `deploy-sdwan` stage triggers the `md-as-code/sdwan` pipeline. This runs validate → plan → **manual deploy**. You must click the play button inside the sdwan child pipeline to proceed.
3. After the sdwan pipeline completes, `deploy-catc` triggers the `md-as-code/catc` pipeline. Again, you must click the manual deploy inside the catc child pipeline.

To navigate between pipelines, click on the stage badge in the orchestrator view to drill into the triggered child pipeline.

## Step 10: Verify the Multi-Domain Integration

### Catalyst Center — Border L3 Handoffs

Navigate to **Provision → Fabric Sites → Bld A → Border Devices**. Confirm `BORDER.cisco.eu` shows the L3 handoff for `BGP-SDWAN` on VLAN 3010.

Navigate to **Provision → Fabric Sites → Bld B → Border Devices**. Confirm `FIAB.cisco.eu` shows the L3 handoff on VLAN 3050.

### SD-WAN Manager — BGP Status

Open SD-WAN Manager: `https://198.18.185.11` (admin / C1sco12345).

Navigate to **Monitor → Devices → C-EDGE-01 → Events**. Verify the BGP session toward `10.100.50.1` (`FIAB.cisco.eu`) is `Established`.

Navigate to **Monitor → Devices → C-EDGE-02 → Events**. Verify the BGP session toward `10.100.30.1` (`BORDER.cisco.eu`) is `Established`.

### Catalyst Center Phase 4: Push CTS Bootstrap Template

In your `ltrxar-3100-catc` local clone, open `data/multidomain_devices.nac.yaml` and uncomment the `dayn_templates:` block for `BORDER.cisco.eu` and `FIAB.cisco.eu`.

```yaml
        # Phase 4: uncomment to push CTS bootstrap template
        # dayn_templates:
        #   regular:
        #     - name: cts-bootstrap
        #       variables:
        #         - name: device_name
        #           value: BORDER.cisco.eu
        #         - name: cts_password
        #           value: C1sco12345
```

Once the changes are made, commit these changes and push to GitLab:
```bash
git add data/multidomain_devices.nac.yaml
git commit -m "Push CTS bootstrap template for SDA Edge devices"
git push gitlab main
```

A new pipeline run starts automatically. The **plan** stage shows exactly one new resource. Trigger the **deploy** job in Catalyst Center repository and verify the CTS bootstrap template is pushed to the fabric devices.

### ISE — TrustSec Propagation

1. In your `ltrxar-3100-ise` local clone, open `data/trust_sec.nac.yaml` and add the following:

```yaml
security_groups:
  - name: PCI
    description: PCI-scoped payment processing servers
    value: 50

security_group_acls:
  - name: DENY_PCI_INBOUND
    ip_version: IP_AGNOSTIC
    acl_content: |
      deny ip log

matrix_entries:
  - source_sgt: IT_Admin
    destination_sgt: PCI
    sgacl_name: DENY_PCI_INBOUND
    rule_status: ENABLED
  - source_sgt: Cameras
    destination_sgt: PCI
    sgacl_name: DENY_PCI_INBOUND
    rule_status: ENABLED
```

2. Commit and push to GitLab:

```bash
git add data/trust_sec.nac.yaml
git commit -m "Add PCI zone isolation (SGT 50)"
git push gitlab main
```

3. The ISE pipeline runs automatically. Complete the validate and plan stages, then trigger the manual deploy.

4. Verify the `PCI` SGT and matrix entries appear in ISE after the pipeline completes.

This demonstrates the full Net-as-Code operational model: a declarative change in Git leads to automated validation and controlled deployment.

## Summary

In this lab you:

- Explored the multi-domain integration architecture (ACI + SD-WAN + Campus SDA)
- Understood the orchestrator pipeline pattern using multi-project triggers
- Deployed cross-domain L3 handoffs using Catalyst Center as Code
- Attached a composed SD-WAN device template to cEdge devices
- Verified BGP peering and TrustSec propagation
- Demonstrated an end-to-end policy change through the pipeline

**Congratulations — you have completed all five labs!**

**Continue to the [Conclusion](conclusion.md).**
