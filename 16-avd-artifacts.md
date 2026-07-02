# 16 — AVD Deep Dive 3: Reading the Generated Artifacts

> **Goal of this chapter:** become fluent at reading every file AVD produces during a build — structured configs, intended configs, documentation, snapshots, validation reports. By the end you should be able to walk into a working AVD repo cold and understand what's been built and deployed without running a thing.

A network engineer's most underrated skill: reading what's there. AVD produces a lot. Knowing where to look saves hours.

## 1. The directory tree after a build

After `ansible-playbook build.yml`:

```
your-avd-project/
├── intended/
│   ├── configs/
│   │   ├── DC1-SPINE1.cfg
│   │   ├── DC1-SPINE2.cfg
│   │   ├── DC1-LEAF1A.cfg
│   │   └── DC1-LEAF1B.cfg
│   └── structured_configs/
│       ├── DC1-SPINE1.yml
│       ├── DC1-SPINE2.yml
│       ├── DC1-LEAF1A.yml
│       └── DC1-LEAF1B.yml
├── documentation/
│   ├── devices/
│   │   ├── DC1-SPINE1.md
│   │   ├── DC1-SPINE2.md
│   │   ├── DC1-LEAF1A.md
│   │   └── DC1-LEAF1B.md
│   └── fabric/
│       └── DC1.md
└── reports/          # after validate.yml
    ├── DC1-state.csv
    └── DC1-state.md
```

Each artifact has a specific job. Let's go through them.

## 2. `intended/configs/<host>.cfg` — the deployable contract

What it is: the **exact** EOS CLI config AVD wants on that device.

How to read it:

- Top-down — it's organized by feature roughly in the order EOS expects.
- Look for the four landmarks: hostname, mgmt interface, BGP block, EVPN block. Anything between is detail.

How to *use* it:

- **In a PR diff**: this is what the reviewer reads. A clean diff (only the lines you intended to change) means low blast radius.
- **As deploy preview**: if `deploy.yml --check` shows changes outside this file, something has drifted on the device.
- **As a network engineer's `show running-config`**: it's exactly what `show running` should be after deploy. Diff against the device to find drift.

> **PR-review trick:** when reviewing an AVD PR, *first* read the `intended/configs/*.cfg` diff. If the YAML-input change is small but the `.cfg` diff is huge, something's wrong.

## 3. `intended/structured_configs/<host>.yml` — the why behind the what

What it is: the per-device dict that `eos_designs` produced *before* templating to CLI.

Why it exists:
- It's the boundary between "high-level AVD inputs" and "low-level rendered CLI."
- When the CLI looks wrong, this is where you start debugging — the bug is either above (in your inputs) or below (in a template).
- It's a stable interface — third-party tooling can consume it without parsing AVD's inputs or EOS CLI.

How to read it:

```yaml
vlans:
  - id: 100
    name: USERS
    tenant: TENANT_A
vlan_interfaces:
  - name: Vlan100
    description: USERS
    vrf: VRF_PROD
    ip_address_virtual: 10.1.100.1/24
router_bgp:
  as: 65101
  vlans:
    - id: 100
      rd: 10.255.0.3:10100
      route_targets:
        both: ["10100:10100"]
```

You're looking at a tree that *closely* mirrors the eventual EOS CLI structure. Top-level keys (`vlans`, `vlan_interfaces`, `router_bgp`) map to top-level CLI blocks. Subkeys map to nested config.

How to *use* it:

- **Diff between commits** to see exactly what AVD recomputed. Often more meaningful than the CLI diff because RDs/RTs/VNIs that AVD auto-allocates appear here, not "above" in your inputs.
- **As an integration point** — pass to a downstream tool, or write your own filter against it.

## 4. `documentation/devices/<host>.md` — the device datasheet

What it is: a per-device markdown doc auto-generated from the structured config.

What's in it (typical sections):

- Management interfaces and IPs
- Loopbacks
- Routing protocols (BGP overview, peers)
- Interfaces (with descriptions, IPs, VLANs)
- VLANs and VLAN-interfaces
- VRFs
- MLAG status
- VXLAN tunnels
- Spanning tree, ACLs, AAA, etc.

How to *use* it:

- **Onboarding** — hand to a new engineer instead of pointing them at the device.
- **Change reviews** — see the markdown diff alongside the CLI diff in PRs.
- **Out-of-band reference** — keep it open in another tab while debugging.

> **Why this matters:** documentation that's *generated from intent* is always current. The painful old habit of "update the wiki when you change the device" disappears. Forever.

## 5. `documentation/fabric/<fabric>.md` — the fabric overview

What it is: a single document describing the entire fabric — topology, ASNs, P2P link map, MLAG pairs, tenants, services.

What's in it (varies by AVD version):

- Topology diagram (mermaid or generated SVG)
- Per-device summary table
- Fabric-wide ASN/loopback allocation
- Tenant + VRF + VLAN summary
- Per-link interface map ("DC1-SPINE1 Eth1 ↔ DC1-LEAF1A Eth1")
- Connected endpoints list

How to *use* it:

- **The single source of truth for "what's the fabric look like?"** — replaces the dusty Visio diagram.
- **Search it** — `grep` for an ASN or an IP to find what owns it.
- **PR sanity check** — if you added a leaf, the fabric doc's tables should reflect it.

## 6. `reports/<fabric>-state.md` — the post-deploy proof

After `validate.yml` runs, you get:

- `reports/<fabric>-state.csv` — machine-readable test results.
- `reports/<fabric>-state.md` — human-readable summary.

The markdown has sections like:

```markdown
# DC1 state report

## Summary

| Test category | Pass | Fail |
|---|---|---|
| Software version | 4 | 0 |
| BGP peers established | 8 | 0 |
| MLAG peer | 2 | 0 |
| ...

## Per-device results

### DC1-LEAF1A
- ✅ Software version matches: 4.32.5.1M
- ✅ BGP peer 10.255.255.1 (DC1-SPINE1) Established
- ✅ MLAG with DC1-LEAF1B: Active
- ❌ VLAN 200 expected but not present
```

How to *use* it:

- **PR gate** — your CI should fail the merge if validation fails.
- **Post-change confirmation** — attach the report to the change ticket.
- **Drift detection** — schedule validate.yml to run nightly; failures = someone touched a device.

## 7. Snapshots and config archives

AVD doesn't ship a snapshot role by default, but many teams add one. A common pattern:

```yaml
- name: Snapshot running-config before deploy
  hosts: FABRIC
  connection: ansible.netcommon.httpapi
  vars:
    snapshot_dir: "snapshots/{{ '%Y%m%dT%H%M%S' | strftime }}"
  tasks:
    - file: { path: "{{ snapshot_dir }}", state: directory }
      delegate_to: localhost
      run_once: true
    - arista.eos.eos_command:
        commands: ["show running-config"]
      register: rc
    - copy:
        content: "{{ rc.stdout[0] }}"
        dest: "{{ snapshot_dir }}/{{ inventory_hostname }}.cfg"
      delegate_to: localhost
```

Save as `snapshot.yml`. Run before any production deploy. Commit the snapshot directory or store in object storage.

## 8. Diff strategies

### `git diff` on `intended/`

The bread-and-butter. Before deploy:

```bash
git status
git diff intended/configs/         # what AVD will push
git diff intended/structured_configs/  # what AVD recomputed under the hood
```

### `diff` between a snapshot and current intended config

```bash
diff -u snapshots/20260520T140000/DC1-LEAF1A.cfg intended/configs/DC1-LEAF1A.cfg | less
```

### EOS-side: `show session-config diffs`

After AVD's deploy opens a session but before commit, EOS can show the diff between running and the candidate. AVD logs this but you can also peek manually:

```
configure session <name>
show session-config diffs
```

## 9. Where each input maps to which artifact

| You changed | AVD recomputes | Visible in |
|---|---|---|
| `NETWORK_SERVICES.yml` (a VLAN) | VLAN, VLAN-interface, EVPN, BGP VLAN block | `intended/configs/*` for every leaf with the matching tag |
| `DC1_FABRIC.yml` (ASN range) | All BGP `router bgp` blocks | All spines and leaves |
| `CONNECTED_ENDPOINTS.yml` (a server) | port-channel, member port, ESI | Just the leaves the server connects to |
| `DC1_SPINES.yml` (a new spine) | New spine + every leaf's uplinks | All devices |
| inventory (new device) | Net-new `.cfg` plus changes to adjacent devices | The new device + adjacent existing devices |

If you change *one* input and the diff explodes across the fabric, take that as a signal to *read* the diff, not panic about it. AVD is honest: a change with wide impact has wide consequences.

## 10. Reading a strange AVD repo for the first time

When you inherit an AVD repo (or fork one), read in this order:

1. `README.md` — what's this project?
2. `ansible.cfg` — where do collections/inventory live?
3. `inventory.yml` — what fabrics and devices exist?
4. `group_vars/*.yml` — start at `FABRIC.yml`, then drill down.
5. `documentation/fabric/<fabric>.md` — the picture.
6. `intended/configs/<one-device>.cfg` — pick a leaf, read the whole config.
7. `build.yml` / `deploy.yml` — confirm the standard pipeline.
8. Any custom playbooks (`snapshot.yml`, `pre-deploy.yml`) — non-standard glue.
9. `.github/workflows/` — what does CI do? (chapter 18)

By step 6 you'll usually know "is this a healthy AVD project or a mess?".

## 11. AVD docs site

The official, exhaustive reference is <https://avd.arista.com>:

- "Roles" → each role's inputs and behavior.
- "Getting Started" → official examples.
- "Schema reference" → every YAML key, type, default.
- "Plugins" → custom filters, lookups.

Bookmark the schema reference. It's where you'll go when "AVD doesn't have a knob for X" — usually it does, you just need to find the key.

## 12. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **Artifact** | Anything AVD writes to disk. |
| **Intended config** | The `.cfg` file in `intended/configs/`. |
| **Structured config** | The YAML in `intended/structured_configs/` — intent before templating. |
| **Fabric doc** | The single Markdown overview at `documentation/fabric/`. |
| **Device doc** | Per-device Markdown at `documentation/devices/`. |
| **State report** | Output of `eos_validate_state` in `reports/`. |
| **Snapshot** | Manual point-in-time backup of running config. |
| **Drift** | Running config diverges from intended. |

---

## 🧪 Exercise

Using the project from chapter 14/15:

1. **Read all four `intended/configs/*.cfg`** end to end. Identify the four landmarks (hostname, mgmt, BGP, EVPN) in each.

2. **Open `documentation/fabric/DC1.md` in VS Code preview** (`Cmd+Shift+V` while the .md file is open). Read every section.

3. **Open `documentation/devices/DC1-LEAF1A.md`**. Compare to its `.cfg` — every CLI block has a doc section.

4. **Compute what changes** when you increase the VRF VNI by 1 in `NETWORK_SERVICES.yml`:
   - Before: note the RD/RT values in `intended/structured_configs/DC1-LEAF1A.yml`.
   - Edit: change `vrf_vni` from 100 to 101.
   - Re-run `build.yml`.
   - `git diff intended/` — which fields changed? Which devices? Which sections of `.cfg`?
   - Revert the change.

5. **Hunt for an ASN**:
   ```bash
   grep -r "router bgp 65101" intended/configs/
   ```
   Confirm only the leaves you expect own that ASN.

6. **Add a `snapshot.yml`** per §7. Run it before next chapter's CI exercises.

7. **Browse the AVD docs site**: read the page for one role you've been using (e.g., `eos_designs` → "Inputs"). Skim the schema reference for one section (e.g., `tenants`). Notice how every key from this chapter is documented.

---

**Next:** [17 — Validation with ANTA and `eos_validate_state`](17-validation-anta.md)
