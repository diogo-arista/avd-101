# AVD common patterns — copy-pasteable YAML

Reference YAML for the most common AVD edits. Tweak names/IDs/IPs to match the user's project. All snippets assume `design.type: l3ls-evpn` unless noted.

> **Important:** every key shown here is from AVD 5.x's `eos_designs` model. For older 4.x projects, some keys differ (notably `evpn_vlan_aware_bundle`-related keys and a few rename). Always confirm against the user's installed version via `ansible-galaxy collection list arista.avd`.

---

## Add a fabric-wide VLAN with anycast gateway

Edit `group_vars/NETWORK_SERVICES.yml`:

```yaml
tenants:
  - name: TENANT_A
    vrfs:
      - name: VRF_PROD
        vrf_vni: 100
        svis:
          - id: 100
            name: USERS
            ip_address_virtual: 10.1.100.1/24
            tags: [DC1_L3_LEAVES]
          # ↓ ADD:
          - id: 110
            name: ENG
            ip_address_virtual: 10.1.110.1/24
            tags: [DC1_L3_LEAVES]
```

That's the *entire* change. AVD will compute VLAN, SVI, L2VNI, EVPN VLAN config, BGP VLAN block, RD/RT on every leaf whose tags include `DC1_L3_LEAVES`.

---

## Add a new tenant

```yaml
tenants:
  - name: TENANT_A
    # … existing …
  # ↓ ADD ENTIRE TENANT:
  - name: TENANT_B
    mac_vrf_vni_base: 20000        # offset so VNIs don't collide with TENANT_A
    vrfs:
      - name: VRF_DEV
        vrf_vni: 200
        svis:
          - id: 300
            name: DEV_LAB
            ip_address_virtual: 10.2.100.1/24
            tags: [DC1_L3_LEAVES]
```

Notes:
- `mac_vrf_vni_base` is the start of the auto-assigned L2 VNI range. Default is 10000 + `svi.id`; for a second tenant pick something non-overlapping.
- `vrf_vni` is the L3 VNI for the VRF itself. Pick a value outside the L2 ranges.

---

## Add a new VRF inside an existing tenant

```yaml
tenants:
  - name: TENANT_A
    vrfs:
      - name: VRF_PROD
        # … existing …
      # ↓ ADD:
      - name: VRF_INFRA
        vrf_vni: 105
        svis:
          - id: 105
            name: INFRA_MGMT
            ip_address_virtual: 10.1.105.1/24
            tags: [DC1_L3_LEAVES]
```

---

## Add a new MLAG-paired L3 leaf

Two parts: inventory + group_vars.

### 1. `inventory.yml`

```yaml
DC1_L3_LEAVES:
  hosts:
    # … existing …
    DC1-LEAF2A: { ansible_host: 172.20.20.6 }
    DC1-LEAF2B: { ansible_host: 172.20.20.7 }
```

### 2. `group_vars/DC1_L3_LEAVES.yml`

Inside the existing `l3leaf:` block, append to `node_groups:`:

```yaml
l3leaf:
  defaults: { … unchanged … }
  node_groups:
    - group: LEAF1
      # … unchanged …
    # ↓ ADD:
    - group: LEAF2
      bgp_as: 65103
      nodes:
        - name: DC1-LEAF2A
          id: 3
          mgmt_ip: 172.20.20.6/24
          uplink_switch_interfaces: [Ethernet3, Ethernet3]
        - name: DC1-LEAF2B
          id: 4
          mgmt_ip: 172.20.20.7/24
          uplink_switch_interfaces: [Ethernet4, Ethernet4]
```

`uplink_switch_interfaces` must match unused ports on the spines (in this case, spine Eth3/Eth4). Cross-check `DC1_SPINES.yml` defaults.

---

## Add a server with ESI multi-homing (MLAG-bonded server)

`group_vars/CONNECTED_ENDPOINTS.yml`:

```yaml
port_profiles:
  - profile: SERVER_MLAG
    mode: trunk
    vlans: "100,110,200"
    spanning_tree_portfast: edge
    port_channel:
      mode: active

servers:
  - name: server02
    rack: rack1
    adapters:
      - endpoint_ports: [eth0, eth1]
        switch_ports: [Ethernet5, Ethernet5]
        switches: [DC1-LEAF1A, DC1-LEAF1B]
        profile: SERVER_MLAG
        port_channel:
          mode: active
          short_esi: auto         # AVD computes ESI from name+port
```

`short_esi: auto` is the safer default — explicit ESIs need careful planning to stay unique across the fabric.

---

## Add a singly-attached server (no MLAG)

```yaml
servers:
  - name: server03-singlehome
    rack: rack2
    adapters:
      - endpoint_ports: [eth0]
        switch_ports: [Ethernet6]
        switches: [DC1-LEAF1A]
        profile: SERVER_TRUNK
```

No `port_channel`, single switch in `switches`. The port profile decides trunk vs access.

---

## Add an L3 connection to an external router (BGP peering)

For a CE/PE-style external L3 connection on a leaf, use `network_services`'s `l3_interfaces` + add a BGP neighbor. Sketch:

```yaml
tenants:
  - name: TENANT_EDGE
    vrfs:
      - name: VRF_EDGE
        vrf_vni: 300
        l3_interfaces:
          - interfaces: [Ethernet7]
            nodes: [DC1-LEAF1A]
            ip_addresses: ["172.31.0.1/30"]
            description: "Edge router uplink"
        bgp_peers:
          - ip_address: 172.31.0.2
            remote_as: 65999
            description: "External BGP peer"
```

(Variants exist depending on AVD version. If the user is on 4.x, check `network_services` schema in their docs.)

---

## Override a single value on a single device

`group_vars/` for groups, `host_vars/<device>.yml` for per-device exceptions:

```yaml
# host_vars/DC1-LEAF1A.yml
structured_config:
  banner_motd: |
    DC1-LEAF1A — under maintenance window 2026-05-20
```

Or for an AVD-modeled key:

```yaml
# host_vars/DC1-LEAF1A.yml
l3leaf:
  defaults:
    mtu: 1500   # this device only, override fabric default
```

Note: per-host overrides of `l3leaf.defaults` only work because of how AVD merges; complex deep merges may need `structured_config` instead.

---

## Add a custom banner / SNMP / NTP

Most of these are AVD-modeled keys, not `raw_eos_cli`:

```yaml
# group_vars/FABRIC.yml
banner_motd: |
  This system is property of …

snmp_server:
  community:
    - name: ro_community
      ro: true
  hosts:
    - host: 10.0.0.100
      community: ro_community
      version: "2c"

ntp:
  authenticate: true
  local_interface:
    name: Management1
  servers:
    - name: time.example.com
      iburst: true
      preferred: true
      vrf: MGMT
```

---

## Configure BGP timers / cluster ID / route maps fabric-wide

```yaml
# group_vars/DC1_FABRIC.yml
bgp_default_ipv4_unicast: false
bgp_distance:
  external_routes: 20
  internal_routes: 200
  local_routes: 200

l3leaf:
  defaults:
    bgp_defaults:
      - "no bgp default ipv4-unicast"
      - "timers bgp 5 15"
      - "maximum-paths 4 ecmp 4"
```

`bgp_defaults:` is a list of literal CLI lines pushed into every BGP block. Powerful but it bypasses AVD's modeling — only use for things AVD doesn't model.

---

## Add a routed P2P L3 link (e.g., DCI)

```yaml
# group_vars/NETWORK_SERVICES.yml (or a dedicated DCI file)
tenants:
  - name: TENANT_TRANSIT
    vrfs:
      - name: VRF_TRANSIT
        l3_interfaces:
          - interfaces: [Ethernet48]
            nodes: [DC1-BORDER1]
            ip_addresses: ["10.99.0.1/31"]
            description: "DCI to DC2-BORDER1"
        bgp_peers:
          - ip_address: 10.99.0.0
            remote_as: 65200
            description: "DC2 transit peer"
```

---

## Use the `structured_config:` escape hatch

When the AVD model doesn't expose a knob you need:

```yaml
# group_vars/DC1_L3_LEAVES.yml
l3leaf:
  defaults:
    structured_config:
      management_console:
        idle_timeout: 30
      ip_radius_source_interfaces:
        - name: Loopback0
          vrf: MGMT
```

`structured_config:` accepts any key from `eos_cli_config_gen`'s data model. Inspect available keys here:

```bash
ls ~/.ansible/collections/ansible_collections/arista/eos_cli_config_gen/molecule/eos_cli_config_gen/inventory/group_vars/
```

…or in the docs: <https://avd.arista.com> → eos_cli_config_gen → roles/eos_cli_config_gen/docs/inputs/.

---

## Use the `raw_eos_cli:` escape hatch (last resort)

```yaml
# host_vars/DC1-LEAF1A.yml
structured_config:
  raw_eos_cli: |
    !
    ! Workaround for vendor-specific feature not in AVD model
    !
    queue-monitor length
    !
```

Use only when:
1. The structured_config model also doesn't have it.
2. You've confirmed by reading the eos_cli_config_gen schema.
3. You've left a comment explaining why.

Reviews can't validate `raw_eos_cli` — treat it like raw assembly in a high-level language.

---

## Scaffold a new minimal AVD project

For a fresh repo, create the skeleton with these files. Adjust names + IPs + ASNs to match the user's plan.

```
my-fabric/
├── ansible.cfg
├── inventory.yml
├── group_vars/
│   ├── FABRIC.yml             # connection creds
│   ├── DC1.yml                # design.type
│   ├── DC1_FABRIC.yml         # ASNs, pools, routing
│   ├── DC1_SPINES.yml         # spine nodes
│   ├── DC1_L3_LEAVES.yml      # leaf nodes
│   └── NETWORK_SERVICES.yml   # tenants/VRFs/SVIs
├── build.yml
└── deploy.yml
```

Use the templates from the curriculum chapter 14 §11 (`labs/14-avd-min/`) as the starting point — they're tested and minimal.
