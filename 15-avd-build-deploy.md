# 15 — AVD Deep Dive 2: Build, Deploy, Validate Workflow

> **Goal of this chapter:** run the full AVD pipeline end-to-end against your lab fabric — generate configs, push them, validate state. By the end you should have a working EVPN/VXLAN fabric brought up purely from the YAML you wrote in [chapter 14](14-avd-inputs.md).

This is the chapter where everything pays off.

## 1. The pipeline (recap)

```
   group_vars/         ┌─────────────────┐  structured_configs/      ┌────────────────────┐
   inventory.yml  ───▶ │  eos_designs    │ ───▶  *.yml per device ──▶│ eos_cli_config_gen │
                       └─────────────────┘                           └─────────┬──────────┘
                                                                               │
                                                                               ▼
                                                                       intended/configs/
                                                                        *.cfg per device
                                                                               │
                                                                               ▼
                                                                  ┌────────────────────────┐
                                                                  │ eos_config_deploy_eapi │
                                                                  │      (or _cvp)         │
                                                                  └─────────┬──────────────┘
                                                                            │
                                                                            ▼
                                                                       running switches
                                                                            │
                                                                            ▼
                                                                  ┌────────────────────────┐
                                                                  │  eos_validate_state    │
                                                                  └────────────────────────┘
                                                                            │
                                                                            ▼
                                                                      reports/*.md|csv
```

Three plays, one pipeline. Let's build the playbooks and run them.

## 2. `build.yml` — render configs (no device contact)

This play runs *locally* — no device connectivity required. It reads your inputs and writes files.

```yaml
---
- name: Build AVD configurations
  hosts: FABRIC
  gather_facts: false
  connection: local
  collections:
    - arista.avd
  tasks:
    - name: Generate per-device structured configs
      ansible.builtin.import_role:
        name: arista.avd.eos_designs

    - name: Render structured configs to EOS CLI + documentation
      ansible.builtin.import_role:
        name: arista.avd.eos_cli_config_gen
```

Run it:

```bash
cd labs/14-avd-min
ansible-playbook -i inventory.yml build.yml
```

What you'll see:

- Per-host tasks fan out across all four devices in parallel.
- `intended/structured_configs/*.yml` appears.
- `intended/configs/*.cfg` appears.
- `documentation/devices/*.md` and `documentation/fabric/<fabric>.md` appear.

The first run takes 20-60 seconds. Subsequent runs are faster because Ansible's fact cache helps.

## 3. Reading the generated structured config

Open `intended/structured_configs/DC1-LEAF1A.yml`:

```yaml
hostname: DC1-LEAF1A
ip_routing: true
service_routing_protocols_model: multi-agent
vlans:
  - id: 100
    name: USERS
    tenant: TENANT_A
vrfs:
  - name: VRF_PROD
    ip_routing: true
vlan_interfaces:
  - name: Vlan100
    description: USERS
    vrf: VRF_PROD
    ip_address_virtual: 10.1.100.1/24
    tenant: TENANT_A
router_bgp:
  as: 65101
  router_id: 10.255.0.3
  vlans:
    - id: 100
      tenant: TENANT_A
      rd: 10.255.0.3:10100
      route_targets:
        both: ["10100:10100"]
…
```

This is the *intent* dict for that device. Every RD, RT, VNI, neighbor entry that you didn't write — AVD computed.

## 4. Reading the generated EOS CLI config

`intended/configs/DC1-LEAF1A.cfg`:

```
hostname DC1-LEAF1A
!
vlan 100
   name USERS
!
vrf instance VRF_PROD
!
interface Vlan100
   description USERS
   vrf VRF_PROD
   ip address virtual 10.1.100.1/24
!
router bgp 65101
   router-id 10.255.0.3
   vlan 100
      rd 10.255.0.3:10100
      route-target both 10100:10100
      redistribute learned
   !
…
```

This is what AVD will push. **Read these files.** Diff them across PRs. They're the closest thing to a "what's about to happen on the network" report you'll get.

## 5. Validating before deploy: lint and check mode

Before pushing anything to devices, sanity-check:

```bash
# YAML/Ansible syntax + style
yamllint group_vars/ inventory.yml
ansible-lint .

# Dry-run the playbook
ansible-playbook -i inventory.yml build.yml --check
```

If `build.yml` succeeds in check mode, the *render* is sound. That doesn't validate semantics — your underlay numbering plan can still be wrong — but it catches typos.

## 6. `deploy.yml` — push to devices

```yaml
---
- name: Deploy AVD configurations to devices
  hosts: FABRIC
  gather_facts: false
  connection: ansible.netcommon.httpapi      # eAPI
  collections:
    - arista.avd
  tasks:
    - name: Configure devices via eAPI
      ansible.builtin.import_role:
        name: arista.avd.eos_config_deploy_eapi
```

Run:

```bash
ansible-playbook -i inventory.yml deploy.yml --check --diff
```

In check mode, AVD will:

- Connect to each switch via eAPI.
- Pull the current running config.
- Diff against `intended/configs/<host>.cfg`.
- Print the diff but make no changes.

The diff output is gold for code review. Lines starting with `-` are removals; `+` are additions. Read every diff before approving an apply.

When you're ready:

```bash
ansible-playbook -i inventory.yml deploy.yml
```

AVD uses **configure sessions** (chapter 10 §4) so each device's change is applied atomically.

## 7. What happens during deploy, in detail

For each target:

1. AVD opens an eAPI session.
2. Starts a configure session (named something like `avd-session-<timestamp>`).
3. Computes the *minimum* set of CLI commands needed to transform running config → intended config (it doesn't re-push the whole config every time).
4. Sends those commands.
5. Pre-checks: does the session show any rejected commands? If so, abort.
6. Commit the session — atomic apply.
7. `write` the new running config to startup.

If anything fails, the session is aborted — running config is unchanged. This is *much* safer than line-by-line CLI push.

## 8. `eos_validate_state` — confirm reality matches intent

After deploy, run validation:

```yaml
---
- name: Validate post-deploy state
  hosts: FABRIC
  gather_facts: false
  connection: ansible.netcommon.httpapi
  collections:
    - arista.avd
  tasks:
    - name: Run validation
      ansible.builtin.import_role:
        name: arista.avd.eos_validate_state
```

Save as `validate.yml`, run:

```bash
ansible-playbook -i inventory.yml validate.yml
```

This role runs ~30 categories of tests per device — BGP sessions in `Established`, expected VLAN/VRF presence, MLAG health, VXLAN tunnels formed, interface oper-up, software version match, etc. The list is computed from your *intent*: "you said you'd have 5 BGP peers, here's whether they're all up."

Results land in `reports/`:

- `reports/<fabric>-state.csv` — one row per test
- `reports/<fabric>-state.md` — human-readable, per-device summary

Open the markdown in VS Code preview and you have your "post-change verification" doc.

## 9. The day-2 loop

Once the initial deploy is good, daily work is:

```
edit YAML → build.yml → review diff → PR → deploy.yml → validate.yml
```

Repeat for every change. Every commit is a snapshot; every PR has a config diff; every deploy is validated. This is the *whole pitch* of AVD: not "automate the initial fabric build" but "make every subsequent change as safe as a code change."

## 10. Common workflows

### Add a VLAN

1. Edit `group_vars/NETWORK_SERVICES.yml` to add the VLAN.
2. `ansible-playbook build.yml` → regenerates affected `.cfg`.
3. `git diff intended/configs/` → review the impact.
4. Commit, PR, merge.
5. `ansible-playbook deploy.yml --limit DC1_L3_LEAVES --check --diff`.
6. `ansible-playbook deploy.yml --limit DC1_L3_LEAVES`.
7. `ansible-playbook validate.yml --limit DC1_L3_LEAVES`.

### Add a new leaf pair

1. Edit `inventory.yml` to add the new hosts.
2. Edit `group_vars/DC1_L3_LEAVES.yml` to add the new node_group.
3. `build.yml` → new `.cfg` files appear; existing files may also change (e.g., spines get new uplink config).
4. Review *every* changed file in the diff.
5. Deploy.

### Decom a device

1. Edit `inventory.yml` to remove the hosts.
2. Remove from `group_vars/DC1_L3_LEAVES.yml`.
3. `build.yml` → adjacent devices' configs lose references to the removed device. Note: orphaned `.cfg` files for the removed device remain on disk — clean them up manually (`git clean -f intended/`) or delete them by hand.
4. Deploy adjacent devices first (clean up first), then `wr erase`/`reload` the removed device.

## 11. Tags and `--limit`: subset operations

You can scope an AVD run to a subset:

```bash
# Just one DC
ansible-playbook deploy.yml --limit DC1

# Just one node group
ansible-playbook deploy.yml --limit "LEAF1*"

# Skip deploy for one device temporarily
ansible-playbook deploy.yml --limit '!DC1-LEAF1B'
```

Useful when doing surgical changes or when one device is in maintenance.

## 12. Connection options

AVD supports two main push backends:

| Backend | Role | When to use |
|---|---|---|
| **eAPI direct** | `arista.avd.eos_config_deploy_eapi` | Lab, small fleets, fast feedback. Direct HTTPS to each device. |
| **CloudVision** | `arista.avd.eos_config_deploy_cvp` | Anywhere CloudVision is the management plane. Pushes via CVP's task workflow — get audit trail, rollback, and concurrent change control. |

CVP path is more enterprise-friendly but requires CloudVision running and configured.

## 13. Troubleshooting cycle

| Symptom | Likely cause | Diagnosis |
|---|---|---|
| `build.yml` fails on a key | YAML typo or wrong type | The error includes line numbers; look at the structured_config it was generating |
| `build.yml` succeeds but `.cfg` looks wrong | Logic bug in your group_vars (wrong tag, wrong group) | Compare `intended/structured_configs/<host>.yml` against your inputs |
| `deploy.yml --check` shows a giant diff on every run | Drift — someone touched the device manually | Either accept and reconcile, or fix the device manually first |
| `deploy.yml` fails with eAPI auth error | Wrong creds in `group_vars/FABRIC.yml` or `host_vars` | Test with `curl -u admin:admin -k https://<ip>/command-api` |
| `validate.yml` reports a BGP peer down | The peer never came up; underlay issue | SSH to device, `show ip bgp summary`, check L1 and L3 |

Every AVD run has a **`changed_when`** somewhere. If you keep getting `changed=1` on deploy with no diff visible — that's usually a `write` task being non-idempotent. Worth investigating but rarely critical.

## 14. The two playbooks you'll write per project

Most AVD projects ship two playbooks:

```yaml
# build.yml — local only
- hosts: FABRIC
  connection: local
  tasks:
    - import_role: { name: arista.avd.eos_designs }
    - import_role: { name: arista.avd.eos_cli_config_gen }
```

```yaml
# deploy.yml — talks to devices
- hosts: FABRIC
  connection: ansible.netcommon.httpapi
  tasks:
    - import_role: { name: arista.avd.eos_config_deploy_eapi }
```

Some teams add a `snapshot.yml` (pre-change config backup) and a `validate.yml` (post-deploy state check). The exact naming is a team choice.

## 15. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **Build** | Run `eos_designs` + `eos_cli_config_gen`. No device contact. |
| **Deploy** | Push intended config to devices via eAPI or CVP. |
| **Validate** | Run `eos_validate_state` against deployed devices; produce a report. |
| **Check mode** | `--check` — dry-run that diffs without applying. |
| **Configure session** | EOS feature that lets a candidate config be applied atomically. |
| **Drift** | When running config no longer matches intended config. |
| **Snapshot** | A point-in-time copy of running config (manual safety habit). |

---

## 🧪 Exercise

Pick up where [chapter 14](14-avd-inputs.md) left off — your `labs/14-avd-min/` project.

1. **Build it**:
   ```bash
   cd labs/14-avd-min
   ansible-playbook -i inventory.yml build.yml
   ls intended/configs/
   ls intended/structured_configs/
   ls documentation/devices/
   ```
   Open one `.cfg` and one structured_config YAML side by side. Confirm the YAML drives the CLI.

2. **Read every doc**: open `documentation/fabric/DC1.md` in VS Code preview. Read it cover to cover. Notice it shows your fabric topology, ASNs, links, tenants, services — all auto-generated.

3. **Dry-run deploy**:
   ```bash
   ansible-playbook -i inventory.yml deploy.yml --check --diff
   ```
   Read every diff. It should show massive additions (the fabric isn't configured yet).

4. **Actually deploy** (assuming your containerlab fabric is up and eAPI is enabled):
   ```bash
   ansible-playbook -i inventory.yml deploy.yml
   ```

5. **Verify the fabric formed**: SSH into a leaf:
   ```
   show ip bgp summary
   show vxlan vni
   show bgp evpn summary
   ```
   Then ping between SVIs on different leaves.

6. **Validate**:
   ```bash
   ansible-playbook -i inventory.yml validate.yml
   cat reports/DC1-state.md
   ```

7. **Add a VLAN**: edit `NETWORK_SERVICES.yml` to add VLAN 200. Run build → diff → deploy → validate. The end-to-end change for adding a fabric-wide VLAN should be: 1 YAML edit + 3 playbook runs.

---

**Next:** [16 — AVD Deep Dive 3: Reading the Generated Artifacts](16-avd-artifacts.md)
