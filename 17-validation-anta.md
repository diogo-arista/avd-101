# 17 — Validation with ANTA and `eos_validate_state`

> **Goal of this chapter:** know how to prove that your network matches intent — both as a one-shot post-deploy check and as a continuous health monitor. By the end you should be using `eos_validate_state` (AVD's wrapper) and ANTA (the underlying tool) confidently.

A network change isn't done when the config is pushed. It's done when you've *proven* the state matches what you intended.

## 1. The two tools

| Tool | What it is | When to use |
|---|---|---|
| **ANTA** (Arista Network Test Automation) | A standalone Python framework + CLI for running test catalogs against EOS devices via eAPI. | Continuous health checks, ad-hoc verification, custom test suites. |
| **`eos_validate_state`** | An AVD role that *uses* ANTA under the hood, with tests auto-generated from your AVD intent. | Post-deploy validation in AVD workflows. |

`eos_validate_state` is the easy on-ramp. ANTA standalone is more flexible but you write the test catalog yourself.

## 2. ANTA in 5 minutes

ANTA is a Python framework where each **test** is a class that:
1. Asks the device for some state (via eAPI).
2. Asserts something about the response.
3. Returns `success`, `failure`, or `error`.

Install in your venv:

```bash
pip install anta
anta --help
```

A **test catalog** is a YAML file that lists tests to run on each device:

```yaml
# anta-catalog.yml
anta.tests.software:
  - VerifyEOSVersion:
      versions:
        - 4.32.5.1M

anta.tests.system:
  - VerifyUptime:
      minimum: 60         # uptime ≥ 60 seconds

anta.tests.routing.bgp:
  - VerifyBGPSpecificPeers:
      address_families:
        - afi: ipv4
          safi: unicast
          peers:
            - peer_address: 10.255.255.1
              vrf: default
              state: Established
```

An **inventory file** lists devices:

```yaml
# anta-inventory.yml
anta_inventory:
  hosts:
    - host: 172.20.20.4
      name: leaf1
    - host: 172.20.20.5
      name: leaf2
```

Run:

```bash
anta nrfu --username admin --password admin --enable \
          --catalog anta-catalog.yml \
          --inventory anta-inventory.yml \
          text
```

Output (text mode):

```
leaf1 :: VerifyEOSVersion :: SUCCESS
leaf1 :: VerifyUptime :: SUCCESS
leaf1 :: VerifyBGPSpecificPeers :: SUCCESS (1 peer, state=Established)
leaf2 :: VerifyEOSVersion :: SUCCESS
leaf2 :: VerifyUptime :: SUCCESS
leaf2 :: VerifyBGPSpecificPeers :: FAILURE
   peer 10.255.255.1 vrf default state=Idle (expected Established)
```

Other output modes: `table`, `json`, `md-report`, `csv` — useful for piping into reports or CI logs.

## 3. The built-in test catalog

ANTA ships with hundreds of tests across categories:

| Category | Examples |
|---|---|
| `anta.tests.software` | EOS version, agents healthy |
| `anta.tests.system` | Uptime, reload reason, CPU, memory |
| `anta.tests.hardware` | Power supplies, fans, temperature, transceiver alarms |
| `anta.tests.interfaces` | Status, MTU, errors, counters, port speed |
| `anta.tests.routing.bgp` | Peer state, prefix counts, communities |
| `anta.tests.routing.ospf` | Neighbor state, interface metrics |
| `anta.tests.evpn` | VTEPs, VNIs, EVPN routes |
| `anta.tests.mlag` | MLAG state, peer-link health |
| `anta.tests.snmp` | Reachability, configured destinations |
| `anta.tests.security` | AAA config, SSH ciphers |
| `anta.tests.connectivity` | Reachability tests (ping) |

Browse: <https://anta.arista.com/stable/api/tests/>.

## 4. Writing a custom test

When the built-in catalog doesn't cover what you need:

```python
# anta_custom/my_tests.py
from anta.models import AntaTest, AntaCommand

class VerifyNoCRCErrors(AntaTest):
    """Check no interface has CRC errors."""
    name = "VerifyNoCRCErrors"
    description = "All interfaces must have 0 CRC error count."
    categories = ["interfaces"]
    commands = [AntaCommand(command="show interfaces counters errors")]

    @AntaTest.anta_test
    def test(self) -> None:
        data = self.instance_commands[0].json_output
        bad = [(i, c["frameTooLong"]) for i, c in data["interfaceErrorCounters"].items()
               if c.get("frameTooLong", 0) > 0]
        if bad:
            self.result.is_failure(f"interfaces with CRC errors: {bad}")
        else:
            self.result.is_success()
```

Reference it in the catalog:

```yaml
anta_custom.my_tests:
  - VerifyNoCRCErrors:
```

This is more involved than YAML test catalogs but unlocks arbitrary checks.

## 5. `eos_validate_state` — the AVD shortcut

For AVD users, the killer feature is that **AVD already knows the intent** — so the test catalog can be auto-generated. The `eos_validate_state` role does exactly that.

In your AVD project, add to a `validate.yml`:

```yaml
---
- name: Validate post-deploy state
  hosts: FABRIC
  gather_facts: false
  connection: ansible.netcommon.httpapi
  collections:
    - arista.avd
  tasks:
    - import_role:
        name: arista.avd.eos_validate_state
```

Run:

```bash
ansible-playbook -i inventory.yml validate.yml
```

What it does:

1. Reads your `intended/structured_configs/<host>.yml`.
2. From that intent, generates an ANTA catalog per device:
   - Expected BGP peers → `VerifyBGPSpecificPeers`
   - Expected VLANs/VRFs → `VerifyVlans`, `VerifyVrf`
   - Expected MLAG → `VerifyMlagStatus`
   - Expected VXLAN tunnels → `VerifyVxlanTunnels`
   - Expected interfaces up → `VerifyInterfacesStatus`
   - Software version, reload cause, uptime, hardware health
3. Runs the catalog via ANTA.
4. Writes `reports/<fabric>-state.csv` (machine-readable) and `reports/<fabric>-state.md` (human-readable).

The auto-generated tests adapt as your intent changes. Add a VLAN to your YAML → next deploy → validate → the new VLAN is now part of the expected state and gets checked.

## 6. Reading the validation report

`reports/<fabric>-state.md`:

```markdown
# DC1 — Validate State Report

| Metric           | Total | Pass | Fail | Skip |
|------------------|-------|------|------|------|
| Tests            | 124   | 122  |   2  |   0  |
| Devices          |   4   |   3  |   1  |   0  |

## Failed tests

| Device | Test | Description |
|---|---|---|
| DC1-LEAF1B | VerifyBGPSpecificPeers | peer 10.255.255.2 vrf default state=Idle (expected Established) |
| DC1-LEAF1B | VerifyVxlanTunnels | expected 1 VTEP (10.255.1.1), found 0 |

## Per-device summary
…
```

A network engineer's tool: open this in VS Code preview, scan for ❌, fix.

`reports/<fabric>-state.csv` is the same data but flat, one row per test — easier to grep, easier to load into Excel or a dashboard.

## 7. Continuous health checks

`eos_validate_state` is great for *post-deploy*. For *continuous* checks (e.g., "every 5 minutes, confirm the fabric is still up"), you have two options:

### Option A: Run AVD validate on a schedule

A cron/CI job that runs `ansible-playbook validate.yml` periodically and alerts on failure. Simple but slow at scale (full ANTA run per cycle).

### Option B: Use ANTA standalone with a curated catalog

Hand-pick the most useful 20–30 tests, run them every 1–5 minutes, push results to your monitoring system. Faster, but no longer driven by intent — you maintain the catalog by hand.

Most teams do both: AVD validate after every deploy, ANTA standalone for continuous checks.

## 8. ANTA output formats and CI integration

ANTA can emit:

- `text` (default, human)
- `table` (terminal table)
- `json` (machine-readable; pipe into `jq`)
- `md-report` (Markdown; great for PR comments and reports)
- `csv` (spreadsheet-friendly)

In CI (chapter 18) you'll typically run `anta nrfu json` and feed the result into:
- A *quality gate* — fail the build if any test failed.
- A *report uploader* — attach the markdown to the PR or Slack channel.

## 9. Where ANTA fits in the bigger picture

```
intent (AVD inputs)
       │
       ▼
   eos_designs → structured config
       │
       ▼
   eos_cli_config_gen → .cfg files
       │
       ▼
   eos_config_deploy_eapi → devices
       │
       ▼
   eos_validate_state ────► ANTA ──► reports
       │
       ▼
   (humans review reports, fix issues)
```

ANTA closes the loop. Without it, you're trusting "the config pushed successfully" as proof of correctness — which it isn't.

## 10. When validation fails: triage

Treat a failed validation like a P2 incident:

1. **Read the test** — what was expected vs what was found.
2. **Hit the device** — `show ip bgp summary`, `show vxlan vni`, whatever the test was checking.
3. **Decide** — is the *device* broken, or was the *intent* wrong?
4. **If device** — fix it on the device, then re-run validate. If it's drift (someone touched the device manually), re-deploy AVD's intent.
5. **If intent** — fix the AVD YAML inputs, build, deploy, validate.

Most failures fall into a small handful of patterns: BGP peer down (cabling/state), MLAG split-brain (peer-link issue), unexpected interface state (port admin down), VXLAN tunnel missing (VNI mismatch). After a few incidents you'll recognize them on sight.

## 11. Edge cases and gotchas

⚠ **Tests run only on devices AVD knows about.** If you have non-AVD devices in your fabric, ANTA can still check them, but `eos_validate_state` won't auto-generate tests for them.

⚠ **Some tests require enable mode.** Pass `--enable` to the ANTA CLI; AVD does this automatically.

⚠ **TLS verification** — same `-k` curse as eAPI. AVD's role uses inventory vars for TLS settings; ANTA standalone needs `--insecure` or proper certs.

⚠ **Test timing** — right after `deploy.yml`, BGP peers and VXLAN tunnels may take seconds-to-minutes to come up. AVD's role has retry logic for some checks; for ANTA standalone, build retries into your CI step or add a `sleep`.

## 12. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **ANTA** | Arista Network Test Automation — the standalone test framework. |
| **Catalog** | YAML file listing which tests to run. |
| **NRFU** | Network Ready For Use — ANTA's CLI subcommand for "run the catalog." |
| **Test** | A Python class deriving from `AntaTest`. |
| **`eos_validate_state`** | AVD role that auto-generates an ANTA catalog from intent. |
| **State report** | The Markdown/CSV output produced after validation. |
| **Drift** | Running state diverges from intended state. |
| **Pre-flight** | Tests run *before* deploy to ensure preconditions. |
| **Post-flight** | Tests run *after* deploy to confirm intent. |

---

## 🧪 Exercise

1. **Standalone ANTA**: install ANTA, write a minimal catalog with `VerifyEOSVersion` + `VerifyUptime`. Run `anta nrfu --catalog ... --inventory ...` against the lab. Output in `table` mode.

2. **`eos_validate_state`**: run the `validate.yml` from chapter 15. Open `reports/<fabric>-state.md` in VS Code preview. Identify any failures and follow the §10 triage.

3. **Cause a failure on purpose**: log into one leaf, manually `configure` and `shutdown` an uplink interface. Re-run validate. Confirm the report shows the failure. Re-deploy AVD to fix it, re-run validate, see it pass.

4. **Custom ANTA test**: write a simple `VerifyHostname` test class that checks the device's hostname matches inventory_hostname. Add it to a catalog file. Run.

5. **Markdown output for CI**: `anta nrfu md-report --md-output /tmp/report.md` and open it. This is what we'll feed into PR comments in chapter 18.

6. **Browse the ANTA docs**: <https://anta.arista.com>. Read one test category's API page (e.g., `anta.tests.routing.bgp`) and pick three tests you'd add to a continuous-monitoring catalog.

---

**Next:** [18 — CI/CD Basics with GitHub Actions](18-cicd-github-actions.md)
