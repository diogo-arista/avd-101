# 14 — AVD Deep Dive 1: Inputs & Fabric Variables

> **Goal of this chapter:** understand the structure of an AVD project's input YAML — fabric-wide settings, group-vars hierarchy, host-vars, tenant/service definitions, and how AVD's data model is organized. By the end you should be able to read any AVD project's `group_vars/` and explain what each file controls.

This is where everything before clicks. Have your VS Code Remote-SSH window open into `avdlab@orb`, with the curriculum repo loaded.

## 1. The AVD project repo layout (recap)

From [chapter 01 §2](01-avd-guided-tour.md):

```
single-dc-l3ls/
├── ansible.cfg
├── inventory.yml
├── group_vars/
│   ├── FABRIC.yml
│   ├── DC1.yml
│   ├── DC1_FABRIC.yml
│   ├── DC1_SPINES.yml
│   ├── DC1_L3_LEAVES.yml
│   ├── DC1_L2_LEAVES.yml
│   ├── CONNECTED_ENDPOINTS.yml
│   └── NETWORK_SERVICES.yml
├── host_vars/                  # rare; only for per-device exceptions
├── build.yml                   # → eos_designs + eos_cli_config_gen
└── deploy.yml                  # → eos_config_deploy_eapi
```

The inventory and group_vars are where you live. Almost no editing happens elsewhere.

## 2. The inventory hierarchy IS the fabric hierarchy

AVD uses Ansible's group inheritance to model the fabric. A typical `inventory.yml`:

```yaml
all:
  children:
    FABRIC:                              # top-level "all my fabrics"
      vars:
        ansible_connection: ansible.netcommon.httpapi
        ansible_network_os: arista.eos.eos
        ansible_user: admin
        ansible_password: admin
        ansible_httpapi_use_ssl: true
        ansible_httpapi_validate_certs: false
      children:
        DC1:                             # site / data center
          children:
            DC1_FABRIC:                  # the routed fabric
              children:
                DC1_SPINES:
                  hosts:
                    DC1-SPINE1: {ansible_host: 172.20.20.2}
                    DC1-SPINE2: {ansible_host: 172.20.20.3}
                DC1_L3_LEAVES:
                  hosts:
                    DC1-LEAF1A: {ansible_host: 172.20.20.4}
                    DC1-LEAF1B: {ansible_host: 172.20.20.5}
            CONNECTED_ENDPOINTS:         # servers / firewalls / external routers
              hosts: {}
            NETWORK_SERVICES:            # tenant / VRF / SVI definitions
              hosts: {}
```

Two things to internalize:

1. **The group names match the `group_vars/<name>.yml` files.** When you write `group_vars/DC1_SPINES.yml`, every host in the `DC1_SPINES` group inherits those vars.
2. **Inheritance flows down.** A var defined in `group_vars/FABRIC.yml` applies to *everything* below it — DC1, DC1_FABRIC, all leaves, all spines — unless overridden lower down.

That's why AVD's groups are named for *fabric concepts* (FABRIC, DC1, DC1_SPINES) and not for *Ansible patterns* (`all`, `webservers`). The group hierarchy IS the data model.

## 3. The five logical groupings of input data

AVD's input model splits into five conceptual areas. Each tends to live in its own `group_vars/<group>.yml` file (or files):

| Area | Typical file | What goes here |
|---|---|---|
| **Design choice** | `group_vars/DC1.yml` | Which design (l3ls-evpn, mpls, l2ls), high-level toggles. |
| **Fabric topology** | `group_vars/DC1_FABRIC.yml` | ASN range, loopback subnets, P2P link subnets, MTU, BFD, BGP timers. |
| **Per-role node config** | `group_vars/DC1_SPINES.yml`, `DC1_L3_LEAVES.yml` | Platform, uplink type, uplink port count, MLAG settings, route-server role. |
| **Connected endpoints** | `group_vars/CONNECTED_ENDPOINTS.yml` | Servers/firewalls and their port mappings. |
| **Tenants / services** | `group_vars/NETWORK_SERVICES.yml` | Tenants → VRFs → SVIs → VLANs, route-targets. |

## 4. Design choice: pick a fabric pattern

`group_vars/DC1.yml`:

```yaml
---
design:
  type: l3ls-evpn        # or 'mpls', 'l2ls'

fabric_name: DC1
```

The `design.type` value tells AVD which set of opinions to apply. `l3ls-evpn` is the most common — Layer-3 leaf-spine with EVPN/VXLAN overlay.

## 5. Fabric topology: the numbering plan

`group_vars/DC1_FABRIC.yml` is where you encode "how IP/ASN/MTU is allocated":

```yaml
---
# ASN allocation
spine:
  bgp_as: 65100
l3leaf:
  defaults:
    bgp_as: 65101
  node_groups:
    - group: LEAF1
      bgp_as: 65101
      nodes:
        - name: DC1-LEAF1A
        - name: DC1-LEAF1B
    - group: LEAF2
      bgp_as: 65102

# Loopback ranges
loopback_ipv4_pool: 10.255.0.0/24       # Loopback0 per device, IPs auto-allocated
vtep_loopback_ipv4_pool: 10.255.1.0/24  # Loopback1 per VTEP

# P2P spine-leaf interconnect
fabric_ip_address_family: ipv4
underlay_routing_protocol: ebgp         # or ospf
fabric_ip_addressing:
  p2p_uplinks_ip_pool: 10.255.255.0/24

# MLAG (peer-link) addressing
mlag_ips:
  leaf_peer_l3:
    ipv4_pool: 10.255.252.0/24
  peer_link:
    ipv4_pool: 10.255.253.0/24

# Underlay defaults
underlay_filter_redistribute_connected: true
default_mgmt_method: oob
mgmt_interface: Management1
mgmt_destination_networks:
  - 0.0.0.0/0
mgmt_interface_vrf: MGMT
```

AVD will *compute* every individual address from these pools — you say "use 10.255.0.0/24 for loopbacks" and AVD assigns `10.255.0.1` to DC1-SPINE1, `10.255.0.2` to DC1-SPINE2, etc. Deterministic and consistent.

## 6. Per-role defaults: SPINES, LEAVES, BORDERS

`group_vars/DC1_SPINES.yml`:

```yaml
---
spine:
  defaults:
    platform: vEOS-LAB
    bgp_as: 65100
    loopback_ipv4_offset: 0          # spine1 gets pool.first + offset
  nodes:
    - name: DC1-SPINE1
      id: 1
      mgmt_ip: 172.20.20.2/24
    - name: DC1-SPINE2
      id: 2
      mgmt_ip: 172.20.20.3/24
```

`group_vars/DC1_L3_LEAVES.yml`:

```yaml
---
l3leaf:
  defaults:
    platform: vEOS-LAB
    spanning_tree_mode: mstp
    mlag_peer_l3_vlan: 4093
    mlag_peer_vlan: 4094
    virtual_router_mac_address: 00:1c:73:00:00:99
    uplink_type: p2p
    uplink_switches: [DC1-SPINE1, DC1-SPINE2]
    uplink_interfaces: [Ethernet1, Ethernet2]
    uplink_switch_interfaces:
      DC1-LEAF1A: [Ethernet1, Ethernet1]
      DC1-LEAF1B: [Ethernet2, Ethernet2]
    structured_config:
      router_bgp:
        bgp_cluster_id: 10.255.0.1
  node_groups:
    - group: LEAF1
      bgp_as: 65101
      nodes:
        - name: DC1-LEAF1A
          id: 1
          mgmt_ip: 172.20.20.4/24
        - name: DC1-LEAF1B
          id: 2
          mgmt_ip: 172.20.20.5/24
```

The `node_groups` pattern bundles MLAG pairs — `LEAF1A` and `LEAF1B` are an MLAG pair sharing ASN 65101.

## 7. Tenants → VRFs → SVIs → VLANs (the service model)

This is where most day-to-day editing happens. `group_vars/NETWORK_SERVICES.yml`:

```yaml
---
tenants:
  - name: TENANT_A
    mac_vrf_vni_base: 10000
    vrfs:
      - name: VRF_PROD
        vrf_vni: 100
        svis:
          - id: 100
            name: USERS
            ip_address_virtual: 10.1.100.1/24
            tags: [DC1_L3_LEAVES]
          - id: 200
            name: VOICE
            ip_address_virtual: 10.1.200.1/24
            tags: [DC1_L3_LEAVES]

  - name: TENANT_B
    vrfs:
      - name: VRF_DEV
        vrf_vni: 200
        svis:
          - id: 300
            name: DEV_LAB
            ip_address_virtual: 10.2.100.1/24
            tags: [DC1_L3_LEAVES]
```

What AVD does with this:

- For each SVI, computes the L2 VNI (typically `mac_vrf_vni_base + svi_id`).
- Sets RD = `<router_id>:<vni>`.
- Sets RT both = `<asn>:<vni>`.
- Adds the SVI + VLAN + L2 VNI + EVPN config on every leaf whose `node.tags` matches the SVI's `tags`.
- Adds the VRF + RD + RT on every leaf where any of its SVIs are deployed.

Adding a new VLAN that spans the whole DC = appending a 4-line YAML block. That's the value.

## 8. Connected endpoints: servers and external links

`group_vars/CONNECTED_ENDPOINTS.yml`:

```yaml
---
port_profiles:
  - profile: SERVER_TRUNK
    mode: trunk
    vlans: "100,200,300"
    spanning_tree_portfast: edge
  - profile: SERVER_MLAG
    mode: trunk
    vlans: "100,200,300"
    port_channel:
      mode: active

servers:
  - name: server01
    rack: rack1
    adapters:
      - endpoint_ports: [eth0, eth1]
        switch_ports: [Ethernet5, Ethernet5]
        switches: [DC1-LEAF1A, DC1-LEAF1B]
        profile: SERVER_MLAG
        port_channel:
          mode: active
          short_esi: auto
```

This produces (for each switch in `switches`): port-channel + member port + EVPN ESI multi-homing config. Again — you say "server01 connects via ESI MLAG to ports 5 on both LEAF1 nodes"; AVD generates the ~30 lines of config that implement it.

## 9. Custom structured config: the escape hatch

Sometimes AVD's data model doesn't have a key for the thing you want to configure. Two escape hatches:

### Inline `structured_config:` per node

```yaml
l3leaf:
  defaults:
    structured_config:
      banner_motd: |
        Lab device — chapter 14 demo
      management_console:
        idle_timeout: 30
```

Anything under `structured_config:` is merged directly into the intermediate dict that `eos_cli_config_gen` consumes. So you can configure *any EOS feature* this way — the trade-off is you lose AVD's higher-level abstractions for that piece.

### Custom raw EOS config

```yaml
structured_config:
  raw_eos_cli: |
    !
    ! Anything in raw_eos_cli goes verbatim into the .cfg
    !
    snmp-server community myCommunity ro
```

Use sparingly — code review can't catch errors here.

## 10. Variable validation: the AVD schema

AVD ships a JSON Schema for every key it understands. With `redhat.vscode-yaml` extension + the schema mapping configured in `.vscode/settings.json`, you get autocompletion, type hints, and error squiggles for every AVD input file.

A workspace `settings.json`:

```json
{
  "yaml.schemas": {
    "https://www.avd.sh/en/stable/schemas/eos_designs.jsonschema.json": [
      "**/group_vars/**/*.yml",
      "**/host_vars/**/*.yml"
    ]
  }
}
```

Once enabled, typo'ing `node_groupz` instead of `node_groups` gets a red squiggle immediately. This is the single most productive AVD configuration tweak.

## 11. A minimal AVD project, copy-paste-ready

For learning, let's set up the smallest AVD project that can deploy something real. We'll put this in our curriculum repo under `labs/14-avd-min/`. Files needed:

- `ansible.cfg`
- `inventory.yml`
- `group_vars/FABRIC.yml`
- `group_vars/DC1.yml`
- `group_vars/DC1_FABRIC.yml`
- `group_vars/DC1_SPINES.yml`
- `group_vars/DC1_L3_LEAVES.yml`
- `group_vars/NETWORK_SERVICES.yml`
- `build.yml`
- `deploy.yml`

We'll work through these in the exercises below and then again in [chapter 15](15-avd-build-deploy.md).

## 12. Reading a real AVD example

The canonical learning example is `examples/single-dc-l3ls` inside the `arista.avd` collection. Find it locally:

```bash
ls ~/.ansible/collections/ansible_collections/arista/avd/examples/single-dc-l3ls
```

Open the project in VS Code (Remote-SSH window). Browse the files. You'll recognize every section in this chapter.

## 13. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **Design** | A named fabric pattern (`l3ls-evpn`, `mpls`, `l2ls`). |
| **Node type** | `spine`, `l3leaf`, `l2leaf`, `super-spine`. |
| **Node group** | A pair of devices sharing state, typically MLAG. |
| **Tenant** | A high-level grouping of VRFs/services — your customer or org unit. |
| **VRF** | A routing instance within a tenant. |
| **SVI** | Switched virtual interface — a Layer-3 interface attached to a VLAN. |
| **VNI** | VXLAN Network Identifier. |
| **RD / RT** | Route Distinguisher / Route Target — BGP EVPN extended community attributes. |
| **Structured config** | The intermediate per-device dict AVD's design role produces. |
| **`raw_eos_cli`** | Verbatim EOS commands injected into intent. |
| **Tag** | A free-form label used to map services to leaves. |

---

## 🧪 Exercise

In `labs/14-avd-min/`, build a 2-spine / 2-leaf AVD project. Start small.

1. **Create the skeleton**:
   ```bash
   mkdir -p labs/14-avd-min/group_vars
   cd labs/14-avd-min
   ```

2. **Inventory** (`inventory.yml`): 2 spines + 2 leaves matching your containerlab IPs (`172.20.20.2-5`). Group them as `DC1_SPINES` and `DC1_L3_LEAVES`, both children of `DC1_FABRIC`, child of `DC1`, child of `FABRIC`.

3. **`group_vars/FABRIC.yml`**: connection vars (httpapi, eAPI creds, no cert validation).

4. **`group_vars/DC1.yml`**: `design.type: l3ls-evpn`, `fabric_name: DC1`.

5. **`group_vars/DC1_FABRIC.yml`**: ASN ranges (spine 65100, leaves 65101-65102), loopback/VTEP/P2P pools, `underlay_routing_protocol: ebgp`.

6. **`group_vars/DC1_SPINES.yml`** and **`group_vars/DC1_L3_LEAVES.yml`**: node definitions with mgmt IPs and uplink mappings.

7. **`group_vars/NETWORK_SERVICES.yml`**: one tenant, one VRF, one SVI 100 spanning both leaves.

8. **`ansible.cfg`** pointing at `inventory.yml` and `collections_path = ../../collections`.

We'll *run* this in chapter 15.

9. **Enable VS Code schema validation**: add a `.vscode/settings.json` in the project (or repo root) pointing AVD's schema at `**/group_vars/**/*.yml`. Verify autocomplete works on a key like `design.type`.

10. **Read the canonical example** (§12): identify each of the seven file types from this chapter in the real example.

---

**Next:** [15 — AVD Deep Dive 2: Build, Deploy, Validate](15-avd-build-deploy.md)
