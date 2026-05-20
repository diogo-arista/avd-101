# AVD triage — symptom → cause → diagnosis

A first-aid kit for AVD pipeline failures. When the user reports a problem, find the matching symptom and walk the diagnosis steps. Most issues fit one of these patterns.

---

## Build (`ansible-playbook build.yml`) failures

### `ERROR! Invalid type for variable X: expected str, got NoneType`

**Cause**: A YAML key was written but left empty (`mgmt_ip:` with no value → `None`).
**Diagnose**:
1. Read the traceback's "duration" line for the exact file and key.
2. `grep -rn "<the-key>:" group_vars/` to find every place it's set.
3. Confirm each non-empty.
**Fix**: provide a value, comment out the line, or wrap with `{% if x is defined %}` in templates.

### `Unknown attribute X for variable Y`

**Cause**: Typo in a key name, or a key that exists in a newer/older AVD version than installed.
**Diagnose**:
1. `ansible-galaxy collection list arista.avd` — check installed version.
2. Browse <https://avd.arista.com> for the matching version; check whether the key exists.
3. If correct in docs but rejected locally — version mismatch.
**Fix**: correct the typo, or bump/pin the AVD collection version.

### `Required field X is missing`

**Cause**: A mandatory field for the selected design is absent.
**Diagnose**:
1. Look up the role's argument spec: `cat ~/.ansible/collections/ansible_collections/arista/avd/roles/eos_designs/meta/argument_specs.yml | less`.
2. Search for the field name; note which container it belongs to.
**Fix**: add the field at the appropriate group level.

### `Loop variable already defined`

**Cause**: Two YAML files at different levels both define a `node_groups:` list (or similar) without `<<` merge handling.
**Diagnose**: `ansible-inventory -i inventory.yml --host <hostname> -y` and look for the field — Ansible will show only the precedence-winning value, hiding the override.
**Fix**: consolidate the definition at one level. Lower-level wins.

### `KeyError: 'tags'` (or similar) when expanding tenants

**Cause**: An SVI's `tags:` list references a group that doesn't exist in inventory.
**Diagnose**: cross-check every `tags:` value in `NETWORK_SERVICES.yml` against group names in `inventory.yml`.
**Fix**: match the tag name to a real inventory group.

### `RecursionError: maximum recursion depth exceeded`

**Cause**: Circular reference in `structured_config:` overrides or Jinja2 vars cross-referencing.
**Diagnose**: find recent changes (`git log --oneline group_vars/`) and bisect.
**Fix**: break the cycle; usually a `{{ var }}` that resolves to itself.

---

## Deploy (`ansible-playbook deploy.yml`) failures

### `Authentication failed` / `403 Forbidden`

**Cause**: Wrong eAPI credentials in `group_vars/FABRIC.yml` (or `host_vars` overrides).
**Diagnose**:
```bash
curl -k -u <user>:<pass> https://<device-ip>/command-api -X POST \
     -d '{"jsonrpc":"2.0","method":"runCmds","params":{"version":1,"cmds":["show version"],"format":"json"},"id":1}'
```
**Fix**: correct the creds (or the vault entry); confirm `management api http-commands` is enabled with the right protocol.

### `Connection refused` / `No route to host`

**Cause**: Mgmt interface down, wrong IP, or `management api http-commands` is `shutdown`.
**Diagnose**:
1. Console / OOB into the device.
2. `show management api http-commands` — confirm `Enabled` and port.
3. `ping <controller-IP>` from the device.
**Fix**: enable eAPI; correct mgmt IP; fix mgmt network reachability.

### Diff in `deploy.yml --check` shows changes you didn't expect

**Cause**: Configuration drift — someone touched the device manually since last deploy.
**Diagnose**: read the diff carefully — is it the kind of change a human would make (descriptions, ACL tweaks, banner changes)?
**Fix**:
- If the drift is desired, capture it back into AVD inputs (so future deploys preserve it).
- If undesired, deploy AVD's intent to flush the drift. Run `validate.yml` afterward.

### `arista.eos.eos_config` reports `failed: True` mid-deploy

**Cause**: AVD pushed CLI that EOS rejected — usually a key that AVD generates but the EOS version doesn't support (or vice versa).
**Diagnose**:
1. Find the rejected command in the traceback.
2. Try the command interactively on the device — read the EOS error.
**Fix**: bump EOS version to one that supports the command, or pin the AVD collection version to one that doesn't emit it.

### Deploy hangs on one host

**Cause**: eAPI request is processing a slow command (e.g., `write` on a busy box) or device CPU pegged.
**Diagnose**: console into the device, check `show processes`.
**Fix**: wait it out; if chronic, raise `forks=` in `ansible.cfg` so others don't block.

---

## Validate (`ansible-playbook validate.yml`) failures

### `VerifyBGPSpecificPeers` failure: peer state `Idle` or `Active`

**Cause**: BGP session not established. Could be L1, L3 reachability, mismatched ASN, ACL, or password.
**Diagnose** on the device:
```
show ip bgp summary
show ip bgp neighbors <peer-ip>
show ip bgp neighbors <peer-ip> | grep -i auth
```
**Fix**: depends on the root cause. Cross-check `bgp_as` in `group_vars/DC1_FABRIC.yml` (or per-group) against what the device thinks.

### `VerifyVxlanTunnels` failure

**Cause**: VTEP not discovering peers via EVPN — usually a BGP EVPN session is down, or VNI/VRF mapping mismatch.
**Diagnose**:
```
show bgp evpn summary
show vxlan vni
show vxlan address-table
```
**Fix**: fix the EVPN underlay first; tunnels follow.

### `VerifyMlagStatus` failure

**Cause**: MLAG peer-link down, or peer-config mismatch (different VLANs allowed on peer-link).
**Diagnose**:
```
show mlag
show mlag config-sanity
```
**Fix**: address the sanity-check violations; usually a config mismatch between peers — re-deploy AVD to fix.

### `VerifyVlans` reports missing VLAN

**Cause**: SVI's `tags:` doesn't include this device's group, so AVD didn't render the VLAN.
**Diagnose**: open `intended/structured_configs/<device>.yml` — confirm `vlans:` does NOT include the missing one. Then check `NETWORK_SERVICES.yml` — does the SVI's `tags:` include this leaf's group?
**Fix**: correct the `tags:` list (most common); re-run build → deploy → validate.

### `VerifyInterfacesStatus` shows admin-down on an uplink

**Cause**: Either AVD generated `shutdown` (because the link isn't in `uplink_switch_interfaces`), or the cable isn't connected, or the peer is down.
**Diagnose**:
1. Confirm topology matches `uplink_switches` / `uplink_interfaces` for this node.
2. `show lldp neighbors` on the device.
3. Confirm the peer interface is up.
**Fix**: update YAML to match physical topology, re-deploy.

### Validation reports lots of failures all at once

**Cause**: Either the deploy didn't actually apply (eAPI auth failure earlier), or the fabric is genuinely broken.
**Diagnose**:
1. `ansible-playbook deploy.yml --check --diff` — should show NO changes if the previous deploy succeeded.
2. If changes are pending, the prior deploy didn't apply — re-run it.
3. If clean, the failures are real — triage device by device.

---

## "Why did AVD generate that?" questions

When the user asks "why does the config have X?":

1. Open `intended/structured_configs/<device>.yml`.
2. Find the key/section that produced X.
3. Trace it up: is it in `group_vars/<this-device's-groups>/*.yml`? Is it from a tenant in `NETWORK_SERVICES.yml`?
4. If not found in user inputs, AVD computed it from a default. Search the role's `defaults/main.yml` or argument specs.

The structured_config YAML is the *Rosetta stone* between inputs and CLI output.

---

## "How do I figure out what's actually on the device vs intent?"

```bash
# What AVD wants
cat intended/configs/<host>.cfg

# What's running
ssh admin@<host>
> show running-config

# Diff
diff -u <(ssh admin@<host> 'show running-config') intended/configs/<host>.cfg | less
```

A clean diff = no drift. Persistent diff = something pushed manually after AVD's last deploy.

---

## When you can't reproduce

If the user says "it broke yesterday" and you can't reproduce now:

1. `git log --oneline --since="2 days ago"` — what changed?
2. `git checkout <commit-before-issue>` on a temp branch and `build.yml`.
3. Bisect with `git bisect` if needed.

AVD's outputs are deterministic from inputs + collection version, so if you can't reproduce, suspect:
- Different `arista.avd` version installed.
- Different `arista.eos` collection version.
- Manual edits to `intended/configs/*` that weren't committed.
