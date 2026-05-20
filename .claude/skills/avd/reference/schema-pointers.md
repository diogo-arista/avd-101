# AVD schema pointers — "I want to configure X, where in AVD?"

Quick lookup table for "what AVD key controls X". Organized by what the user typically asks for. Targets AVD 5.x — for 4.x cross-check the user's installed version.

When the table doesn't have what you need, the authoritative reference is <https://avd.arista.com> → **Roles → eos_designs → Inputs**. The schema is also browsable in JSON Schema form at the URL shown in chapter 08 §7.

---

## Underlay / fabric base

| User wants to set… | Where | Example key |
|---|---|---|
| ASN per role (spine/leaf) | `group_vars/<DC>_FABRIC.yml` | `spine.bgp_as`, `l3leaf.defaults.bgp_as` or per node_group `bgp_as` |
| Per-MLAG-pair ASN | `group_vars/<DC>_L3_LEAVES.yml` | `l3leaf.node_groups[].bgp_as` |
| Loopback subnet | `group_vars/<DC>_FABRIC.yml` | `loopback_ipv4_pool` |
| VTEP loopback subnet | `group_vars/<DC>_FABRIC.yml` | `vtep_loopback_ipv4_pool` |
| Spine-leaf P2P subnet | `group_vars/<DC>_FABRIC.yml` | `fabric_ip_addressing.p2p_uplinks_ip_pool` |
| MLAG peer-link addressing | `group_vars/<DC>_FABRIC.yml` | `mlag_ips.peer_link.ipv4_pool` |
| Underlay protocol (ebgp vs ospf) | `group_vars/<DC>_FABRIC.yml` | `underlay_routing_protocol: ebgp` |
| Overlay protocol | (same) | `overlay_routing_protocol: ebgp` |
| MTU fabric-wide | `group_vars/<DC>_FABRIC.yml` | `p2p_uplinks_mtu`, or per-role `l3leaf.defaults.mtu` |
| BFD timers | role defaults | `bgp_peer_groups.*.bfd_timers` |
| Maximum-paths / ECMP | per role | `l3leaf.defaults.maximum_paths` |

---

## Tenants / VRFs / SVIs

| User wants… | Where | Example key |
|---|---|---|
| A new VLAN | `NETWORK_SERVICES.yml` | `tenants[].vrfs[].svis[]` |
| A new VRF | (same) | `tenants[].vrfs[]` |
| A new tenant | (same) | `tenants[]` |
| Anycast gateway IP | per SVI | `tenants[].vrfs[].svis[].ip_address_virtual` |
| VLAN scope (which leaves) | per SVI | `tenants[].vrfs[].svis[].tags: []` (matched against inventory groups) |
| Specific L2 VNI override | per SVI | `tenants[].vrfs[].svis[].vni_override` |
| L3 VNI for VRF | per VRF | `tenants[].vrfs[].vrf_vni` |
| Custom RD per VLAN | per SVI | `tenants[].vrfs[].svis[].rd_override` |
| Custom RT per VLAN | per SVI | `tenants[].vrfs[].svis[].rt_override` |
| Static routes per VRF | per VRF | `tenants[].vrfs[].static_routes[]` |
| BGP peers in a VRF | per VRF | `tenants[].vrfs[].bgp_peers[]` |
| L3 interface in a VRF (e.g., to external router) | per VRF | `tenants[].vrfs[].l3_interfaces[]` |
| DHCP relay on SVI | per SVI | `tenants[].vrfs[].svis[].ip_helpers[]` |

---

## Connected endpoints (servers, firewalls, external)

| User wants… | Where | Example key |
|---|---|---|
| Server (single switch) | `CONNECTED_ENDPOINTS.yml` | `servers[].adapters[].switches: [LEAF1]` |
| Server (MLAG bonded) | (same) | `adapters[].switches: [LEAF1A, LEAF1B]`, `adapters[].port_channel.short_esi: auto` |
| Trunk/access mode | per adapter or per profile | `adapters[].mode` or `port_profiles[].mode` |
| Allowed VLANs | per adapter | `adapters[].vlans: "100,200"` (string of comma+range syntax) |
| Spanning tree edge | per adapter | `adapters[].spanning_tree_portfast: edge` |
| LACP mode | per port-channel | `adapters[].port_channel.mode: active` |
| Reusable port templates | top of file | `port_profiles[]` then `adapters[].profile: <name>` |
| Generic generic devices (non-server) | (same) | `generic_devices[]` similar shape to `servers[]` |

---

## Management plane

| User wants… | Where | Example key |
|---|---|---|
| Management interface | role defaults | `mgmt_interface: Management1` |
| Management VRF | role defaults | `mgmt_interface_vrf: MGMT` |
| Management gateway | per device | `mgmt_default_gateway: 172.20.20.1` (or fabric default) |
| Static management IP | per device | `mgmt_ip: 172.20.20.4/24` |
| OOB DNS / NTP servers | `group_vars/FABRIC.yml` | `name_servers[]`, `ntp.servers[]` |
| AAA TACACS | `group_vars/FABRIC.yml` | `aaa_authentication`, `tacacs_servers[]` |
| Local users | `group_vars/FABRIC.yml` | `local_users[]` |
| SSH keys per user | (same) | `local_users[].sshkey` |

---

## Routing / BGP details

| User wants… | Where | Example key |
|---|---|---|
| Custom BGP peer group | role defaults | `bgp_peer_groups[]` |
| EVPN address-family settings | (same) | `evpn_address_family.*` |
| Route maps | per device or via `structured_config` | `route_maps[]` |
| Prefix lists | (same) | `ip_prefix_lists[]` |
| Filter on EVPN routes | per VRF or role | `bgp_peer_groups[].route_map_*` |
| AS-path access lists | (same) | `as_path[].access_lists[]` |
| BGP timers per peer-group | (same) | `bgp_peer_groups[].timers` |

---

## Security / ACLs

| User wants… | Where | Example key |
|---|---|---|
| Standard ACL | `structured_config:` | `standard_access_lists[]` |
| Extended ACL | `structured_config:` | `access_lists[]` |
| ACL on interface | per interface in `structured_config:` | `ethernet_interfaces[].access_group_in/out` |
| MAC ACL | `structured_config:` | `mac_access_lists[]` |
| Control-plane policing | `structured_config:` | `policy_maps.control_plane[]` |

(ACLs are partially modeled; complex ones often land in `structured_config:`.)

---

## EOS features that don't have first-class AVD keys

For these, drop to `structured_config:` (or `raw_eos_cli:` as a last resort):

- Detailed QoS policies
- Storm-control
- IGMP snooping fine-tuning
- IPv6 multicast PIM tweaks
- VRRP (use `ip_address_virtual` for anycast GW instead — usually the better choice)
- Custom event-handlers
- TerminAttr (CloudVision streaming) details

**Always** check the eos_cli_config_gen schema before assuming a key is missing — many "missing" knobs exist under `structured_config:` even if not surfaced in `eos_designs`.

---

## How to find a key fast

When the user asks "where's X?":

1. **Grep AVD's own examples** for the keyword:
   ```bash
   grep -rn "<keyword>" ~/.ansible/collections/ansible_collections/arista/avd/examples/
   ```
2. **Grep the role's argument specs**:
   ```bash
   grep -rn "<keyword>" ~/.ansible/collections/ansible_collections/arista/avd/roles/eos_designs/meta/argument_specs.yml
   grep -rn "<keyword>" ~/.ansible/collections/ansible_collections/arista/avd/roles/eos_cli_config_gen/meta/argument_specs.yml
   ```
3. **Search the JSON Schema** at <https://avd.arista.com> (use the Search box).

If none of those hit, the feature is *probably* not modeled — escape hatch territory.

---

## Version-sensitive keys (4.x vs 5.x)

A few keys moved between major versions. When in doubt, run `ansible-galaxy collection list arista.avd` and check the docs for that version. Common renames:

| 4.x key | 5.x key |
|---|---|
| `evpn_vlan_aware_bundle` | `evpn_l2_multi_domain` (semantics changed too) |
| `network_services_keys[].name` | now part of `tenants[]` structure |
| `evpn_short_esi_prefix` | `default_short_esi_prefix` |

Don't pretend to be authoritative on every rename — when uncertain, recommend the user check release notes between their version and the version of any docs page they're following.
