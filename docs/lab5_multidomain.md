# Lab 5 — Multi-Domain Integration

In this lab you will bring together the work from Labs 1–4 and deploy the cross-domain integration that connects the ACI fabric, SD-WAN overlay, and SDA campus into a unified policy domain. A single GitLab repository orchestrates child CI/CD pipelines that configure the L3 handoffs between fabric borders and SD-WAN edges in the correct dependency order.

> **Prerequisite:** Complete Labs 1 through 4 before starting this lab.

## Lab Objectives

After completing this lab, you will be able to:

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
  │  BD-Web             │          │  VLAN_EMPLOYEES      │
  │  L3OUT-SDWAN-PROD   │          │  VLAN_SERVERS        │
  │  LEAF102 Gi1/1      │          │                      │
  │  VLAN 3010          │          │  Fabric B (Bld B)    │
  │  10.100.10.1/30     │          │  IoT VN              │
  │  BGP AS 65100       │          │  VLAN_CAMERAS        │
  └────────┬────────────┘          └──────────┬───────────┘
           │ eBGP (APIC L3Out)                │ eBGP (CATC border handoff)
           │                                  │ BORDER AS 65001 / FIAB AS 65002
           ▼                                  ▼
  ┌────────────────────────────────────────────────────────┐
  │                  SD-WAN Overlay (VPN 30)               │
  │              Transit AS 65200                          │
  │                                                        │
  │  C-EDGE-01 (Site 2)        C-EDGE-02 (Site 3)         │
  │  VLAN 3050 ↔ FIAB          VLAN 3010 ↔ BORDER         │
  │  10.100.50.2/30            10.100.30.2/30              │
  └────────────────────────────────────────────────────────┘
```

**SGT propagation path:**
- ISE assigns SGT (10/20/30) via static IP-SGT mapping
- Catalyst Center programs SGT-to-VLAN bindings on fabric edge switches
- SGT is carried inline (CMD header) through the SDA fabric
- SD-WAN cEdge preserves SGT across the overlay (`propagate_sgt: true`)
- ACI L3Out receives BGP routes from SD-WAN (peer Idle in simulator — expected)

## Repository Structure

The multi-domain repository is organized into two sub-directories, one per controller. The root `.gitlab-ci.yml` orchestrates both as sequential child pipelines:

```
ltrxar-3100-multidomain/
├── .gitlab-ci.yml                    # Orchestrator pipeline (triggers child pipelines)
├── catc/
│   ├── main.tf
│   ├── .gitlab-ci.yml                # Child pipeline: Catalyst Center
│   └── data/
│       ├── transit.nac.yaml          # BGP transit toward SD-WAN AS 65200
│       ├── border_BORDER.nac.yaml    # Fabric A border: L3 handoff (BORDER)
│       └── border_FIAB.nac.yaml      # Fabric B border: L3 handoff (FIAB)
└── sdwan/
    ├── main.tf
    ├── .gitlab-ci.yml                # Child pipeline: SD-WAN Manager
    └── data/
        ├── sites.nac.yaml            # cEdge site definitions + device variables
        ├── edge_feature_templates.nac.yaml  # Handoff-specific feature templates
        └── edge_device_templates.nac.yaml   # Device template composing all feature templates
```

## Step 1: Connect to the Windows Workstation

If you have closed the RDP session, reconnect:

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

Open `.gitlab-ci.yml` at the root of the repository. This is the **orchestrator** — it does not run Terraform itself. Instead, it triggers the child pipelines in the `catc/` and `sdwan/` subdirectories in the correct sequence:

```yaml
stages:
  - deploy-catc
  - deploy-sdwan

deploy_catc:
  stage: deploy-catc
  trigger:
    include: catc/.gitlab-ci.yml
    strategy: depend
  rules:
    - if: '$CI_COMMIT_BRANCH == "master"'
      when: always

deploy_sdwan:
  stage: deploy-sdwan
  trigger:
    include: sdwan/.gitlab-ci.yml
    strategy: depend
  rules:
    - if: '$CI_COMMIT_BRANCH == "master"'
      when: always
```

**Key design decisions:**

- **`strategy: depend`** — the parent pipeline waits for each child pipeline to finish and inherits its exit status. A failure in `deploy-catc` prevents `deploy-sdwan` from running, since the SDA border handoff must be up before SD-WAN device templates are attached
- **Sequential stages** — CATC runs first so fabric border configurations are in place; SD-WAN runs second to attach device templates that reference the handoff VLANs
- **ACI and ISE are not in this pipeline** — they are managed by their own independent repositories (Labs 1 and 4). The multidomain repo only handles the handoffs between CATC and SD-WAN

## Step 4: Explore the Catalyst Center Multi-Domain Data

### Transit (`catc/data/transit.nac.yaml`)

```yaml
catalyst_center:
  fabric:
    transits:
      - name: BGP-SDWAN
        type: IP_BASED_TRANSIT
        routing_protocol_name: BGP
        autonomous_system_number: 65200
```

The `BGP-SDWAN` transit tells Catalyst Center that BGP AS 65200 is the external routing domain. Both Fabric A (BORDER) and Fabric B (FIAB) reference this transit by name in their L3 handoff configurations. A single transit object can serve multiple fabric borders.

### Fabric A Border Handoff (`catc/data/border_BORDER.nac.yaml`)

```yaml
catalyst_center:
  inventory:
    devices:
      - name: BORDER
        device_ip: 198.18.130.10
        site: Global/USA/Nevada/Las Vegas/Bld A
        fabric_site: Global/USA/Nevada/Las Vegas/Bld A
        fabric_roles:
          - BORDER_NODE
          - CONTROL_PLANE_NODE

  fabric:
    border_devices:
      - name: BORDER
        border_types:
          - LAYER_3
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
                    tcp_mss_adjustment: 1350
```

**Key points:**
- `local_autonomous_system_number: 65001` is Fabric A's BGP AS — it peers with C-EDGE-02 (AS 65200) on `GigabitEthernet1/0/24`
- The L3 handoff is named `BGP-SDWAN` — matching the transit name in `transit.nac.yaml`
- `tcp_mss_adjustment: 1350` is required because the SD-WAN IPsec tunnel reduces the effective MTU

### Fabric B Border Handoff (`catc/data/border_FIAB.nac.yaml`)

```yaml
catalyst_center:
  fabric:
    border_devices:
      - name: FIAB
        border_types:
          - LAYER_3
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
                    tcp_mss_adjustment: 1350
```

Fabric B uses AS 65002 and hands off the `IoT` VN to C-EDGE-01 over VLAN 3050.

## Step 5: Explore the SD-WAN Multi-Domain Data

### Device Template (`sdwan/data/edge_device_templates.nac.yaml`)

The device template `DT-HUB-C8000V-MULTIDOMAIN` composes feature templates into a complete device configuration for both C-EDGE-01 and C-EDGE-02:

```yaml
sdwan:
  edge_device_templates:
    - name: DT-HUB-C8000V-MULTIDOMAIN
      description: "Multidomain HUB C8000V (SDA fabric border handoff over VPN 30)"
      device_model: C8000V
      system_template: FT-REMOTE-EDGE-SYSTEM-01
      vpn_0_template:
        name: FT-REMOTE-VPN0-OVERLAY
        ethernet_interface_templates:
          - name: FT-TLOC1-PUBLIC-REMOTE-VPN0    # from Lab 2 repo
          - name: FT-TLOC2-PRIVATE-REMOTE-VPN0   # from Lab 2 repo
      vpn_service_templates:
        - name: FT-REMOTE-VPN30-CORP             # from Lab 2 repo
          bgp_template: FT-BGP-VPN30-HANDOFF
          ethernet_interface_templates:
            - name: FT-ETH3-SUBIF-VPN30-HANDOFF  # handoff sub-interface
      vpn_512_template:
        name: FT-REMOTE-VPN512-MGMT
```

> **Key insight:** The base feature templates (`FT-REMOTE-VPN30-CORP`, `FT-TLOC1-*`, `FT-TLOC2-*`) come from the Lab 2 SD-WAN repository — they were deployed in Lab 2. The multidomain repository adds only the handoff-specific templates on top. This composability is a core benefit of the feature template pattern.

### Site Definitions (`sdwan/data/sites.nac.yaml`)

```yaml
sdwan:
  sites:
    # C-EDGE-01 (Site 2) — faces Fabric B (FIAB)
    - id: 2
      routers:
        - chassis_id: C8K-D4CE7174-5261-7E6F-91EA-4926BCF4C2DD
          model: C8000V
          device_template: DT-HUB-C8000V-MULTIDOMAIN
          device_variables:
            site_id: 2
            system_ip: 10.0.0.2
            system_hostname: C-EDGE-01
            vpn30_service_gi3_subif_vlan: 3050
            vpn30_service_gi3_subif_ip: 10.100.50.2/30
            vpn30_bgp_neighbor_ip: 10.100.50.1
            vpn30_bgp_neighbor_as: 65002

    # C-EDGE-02 (Site 3) — faces Fabric A (BORDER)
    - id: 3
      routers:
        - chassis_id: C8K-A1BC2345-6789-ABCD-EF01-234567890ABC
          model: C8000V
          device_template: DT-HUB-C8000V-MULTIDOMAIN
          device_variables:
            site_id: 3
            system_ip: 10.0.0.3
            system_hostname: C-EDGE-02
            vpn30_service_gi3_subif_vlan: 3010
            vpn30_service_gi3_subif_ip: 10.100.30.2/30
            vpn30_bgp_neighbor_ip: 10.100.30.1
            vpn30_bgp_neighbor_as: 65001
```

The same device template is attached to both cEdges — only the variable values differ. One template definition, two device instantiations.

## Step 6: Create a GitLab Project

Open a browser and navigate to the GitLab instance: `http://198.18.128.50`

Log in with `labuser` / `C1sco12345`.

Create a new project:

1. Click **New project → Create blank project**
2. Set the **Project name** to `ltrxar-3100-multidomain`
3. Set the **Namespace** to `md-as-code`
4. Set **Visibility level** to **Private**
5. **Uncheck** "Initialize repository with a README"
6. Click **Create project**

> **Note:** Catalyst Center and SD-WAN credentials are pre-configured at the `md-as-code` group level and inherited by this project automatically.

## Step 7: Push to GitLab

In the VS Code terminal:

```bash
git remote add gitlab http://198.18.128.50/md-as-code/ltrxar-3100-multidomain.git
git push gitlab master
```

When prompted, enter `labuser` / `C1sco12345`.

## Step 8: Monitor and Trigger the Orchestrator Pipeline

In GitLab, navigate to **Build → Pipelines** in your `ltrxar-3100-multidomain` project.

The orchestrator pipeline shows two stages:

```
deploy-catc    →    deploy-sdwan
```

Each stage contains a **child pipeline** that is triggered automatically. Click on either stage to drill into the child pipeline view showing the full validate → plan → deploy → destroy stages running inside the subdirectory context.

The orchestrator pipeline runs fully automatically on `master` push — both child pipelines run to completion (or stop on failure) without requiring manual intervention at the orchestrator level. The `deploy` jobs **within each child pipeline** remain manual gates.

**To trigger the CATC deploy:**
1. Click **deploy-catc** in the orchestrator view
2. Drill into the child pipeline
3. Click the **play button (▶)** next to the `deploy` job

Wait for the CATC child pipeline to complete, then:

**To trigger the SD-WAN deploy:**
1. Click **deploy-sdwan** in the orchestrator view
2. Drill into the child pipeline
3. Click the **play button (▶)** next to the `deploy` job

![Orchestrator Pipeline](./assets/multidomain_pipeline.png)

## Step 9: Verify the Multi-Domain Integration

### Catalyst Center — Border L3 Handoffs

Navigate to **Provision → Fabric Sites → Bld A → Border Devices**.

Confirm `BORDER` shows:
- Role: `BORDER_NODE, CONTROL_PLANE_NODE`
- L3 handoff: `BGP-SDWAN`, Campus VN, VLAN 3010, `10.100.30.1/30`

Navigate to **Provision → Fabric Sites → Bld B → Border Devices**.

Confirm `FIAB` shows:
- L3 handoff: `BGP-SDWAN`, IoT VN, VLAN 3050, `10.100.50.1/30`

### SD-WAN Manager — Device Template Attachment

Open a browser and navigate to SD-WAN Manager: `https://198.18.185.11`

Log in with `admin` / `C1sco12345`.

Navigate to **Configuration → Templates → Device Templates**.

Confirm `DT-HUB-C8000V-MULTIDOMAIN` is attached to `C-EDGE-01` and `C-EDGE-02`.

Navigate to **Monitor → Devices** and select `C-EDGE-01`. Under **Real-Time → BGP**, verify the BGP session toward `10.100.50.1` (FIAB) shows as `Established`.

> **Note:** In the simulator environment, BGP sessions may take 2–3 minutes to establish after device template attachment.

### ISE — TrustSec Propagation

Navigate to **Work Centers → TrustSec → SXP** and verify that the static IP-SGT mappings are being propagated to the network devices.

### ACI — L3Out BGP Peer Status

In the APIC GUI (`https://198.18.133.200`), navigate to **Tenants → PROD → Networking → L3Outs → L3OUT-SDWAN-PROD → Nodes and Interfaces Protocols → BGP**.

The BGP peer `10.100.10.2` will show as `Idle` — this is expected in the simulator because the physical link between LEAF102 and C-EDGE-02 is not simulated. In a real deployment, this peer would establish and exchange routes between the ACI fabric and SD-WAN VPN30.

![Multi-Domain Topology](./assets/multidomain_topology.png)

## Step 10: Simulate an End-to-End Policy Change

To demonstrate the full pipeline operational model, add a new network segment — `PCI` zone (SGT 40) — isolated from all other zones.

1. In your `ltrxar-3100-ise` local clone, open `data/trust_sec.nac.yaml` and add:

   ```yaml
   security_groups:
     - name: PCI
       description: PCI-scoped payment processing servers
       value: 40

   security_group_acls:
     - name: DENY-PCI-INBOUND
       ip_version: IP_AGNOSTIC
       acl_content: |
         deny ip log

   matrix_entries:
     - source_sgt: Employees
       destination_sgt: PCI
       sgacl_name: DENY-PCI-INBOUND
       rule_status: ENABLED
     - source_sgt: Cameras
       destination_sgt: PCI
       sgacl_name: DENY-PCI-INBOUND
       rule_status: ENABLED
   ```

2. Commit and push to GitLab:

   ```bash
   git add data/trust_sec.nac.yaml
   git commit -m "Add PCI zone isolation (SGT 40)"
   git push gitlab main
   ```

3. The ISE pipeline runs automatically — validate → plan → manual deploy.

4. Verify the `PCI` SGT and matrix entries appear in ISE within minutes of the pipeline completing.

This demonstrates the full Net-as-Code operational model: a declarative change in Git → automated validation → controlled deployment → verified state.

## Summary

In this lab you:

- Explored the multi-domain integration architecture (ACI + SD-WAN + Campus SDA)
- Understood the orchestrator pipeline pattern (root pipeline triggers child pipelines in sequence)
- Deployed cross-domain L3 handoffs using Catalyst Center as Code (transit, BORDER, FIAB)
- Attached a composed SD-WAN device template to cEdge devices with per-site variable values
- Verified BGP peering, device template attachment, and TrustSec propagation
- Demonstrated an end-to-end policy change through the pipeline

**Congratulations — you have completed all five labs!**

**Continue to the [Conclusion](conclusion.md).**
