# 01 — A Guided Tour of AVD

> **Goal of this chapter:** look at a real AVD project end-to-end and trace data from a YAML file all the way to a deployed device config — *without setting anything up*. Expect to finish with more questions than answers. That's the point.

We won't run anything in this chapter. You're going to read structures the way you'd read a topology diagram on a whiteboard before you cable anything.

## 1. What AVD actually is, concretely

When someone says "we use AVD," they mean:

1. An **Ansible collection** called `arista.avd` that you install with `ansible-galaxy collection install arista.avd`.
2. A **data model** — a documented set of YAML keys that describe your fabric (spines, leaves, links, VLANs, tenants, BGP, EVPN, etc.).
3. A **role pipeline** — three or four Ansible roles that, in order:
   - Take your YAML → produce a fully-structured per-device config tree.
   - Render that tree → produce EOS CLI text (`.cfg` files).
   - Push those configs to the switches (via eAPI or CloudVision).
   - Validate that the switches actually reflect the intent.
4. A pile of **example repositories** that show "here's a working L3LS EVPN/VXLAN fabric — fill in your details."

You consume AVD by maintaining your own *project repository* that imports the collection and provides the inputs. AVD itself is upstream — you don't fork it.

## 2. The canonical example: a single-DC L3LS fabric

The standard onboarding example is the **"Single DC L3LS"** project. It builds a 2-spine, 4-leaf EVPN/VXLAN fabric. You can generate it locally with `ansible-playbook arista.avd.init` (we'll do this in chapter 02), but here's what the resulting repo looks like:

```
single-dc-l3ls/
├── ansible.cfg                    # Ansible behavior settings
├── inventory.yml                  # device list + group hierarchy
├── group_vars/
│   ├── FABRIC.yml                 # fabric-wide common settings
│   ├── DC1.yml                    # DC1-specific design choice (L3LS-EVPN)
│   ├── DC1_FABRIC.yml             # ASNs, loopbacks, P2P link ranges
│   ├── DC1_SPINES.yml             # spine role defaults
│   ├── DC1_L3_LEAVES.yml          # leaf role defaults
│   ├── DC1_L2_LEAVES.yml          # L2-only leaf defaults (optional)
│   ├── CONNECTED_ENDPOINTS.yml    # server/firewall/router connections
│   └── NETWORK_SERVICES.yml       # tenants, VRFs, SVIs, VLANs
├── build.yml                      # playbook: YAML → configs (no device contact)
├── deploy.yml                     # playbook: push configs to devices
└── README.md
```

Two big ideas hide in this layout:

- **Group hierarchy mirrors fabric hierarchy.** `FABRIC` contains `DC1`, `DC1` contains `DC1_FABRIC`, which contains `DC1_SPINES` and `DC1_L3_LEAVES`. Variables defined higher up apply to everything beneath, unless overridden. This is exactly how Ansible inventory inheritance works — and AVD leans on it heavily.
- **There's no per-device config file.** You describe *roles* (spine, leaf) and *services* (tenants, VLANs), and AVD computes what each individual device needs. You only drop to per-device YAML when you need an exception.

## 3. Following the data flow

Let's trace one specific thing — **a VLAN** — from intent to deployed config.

### Step 1: You write the intent (YAML)

In `group_vars/NETWORK_SERVICES.yml`:

```yaml
tenants:
  - name: TENANT_A
    vrfs:
      - name: VRF_PROD
        vrf_vni: 10
        svis:
          - id: 100
            name: USERS
            ip_address_virtual: 10.1.100.1/24
            tags: [DC1_L3_LEAVES]
```

That's it. You said: "Tenant A has a VRF called PROD, and inside that VRF there's a VLAN 100 named USERS, with an anycast gateway 10.1.100.1/24, deployed on all DC1 L3 leaves."

You did **not**:
- Pick which VNI to use for the L2 segment (AVD computes it).
- Decide which BGP EVPN route-targets to assign (AVD computes them).
- Write the per-device VLAN/SVI/VRF/route-map/EVPN config (AVD generates it).
- Decide which leaves get it (AVD reads the `tags` field and figures it out).

### Step 2: `eos_designs` role expands intent into structured config

You run `ansible-playbook build.yml`, which invokes the `arista.avd.eos_designs` role. For each device, the role walks the YAML inputs, applies the design (L3LS-EVPN), and produces an *intermediate* structured representation — a big dict of every config knob the device needs.

For leaf `DC1-LEAF1A`, the structured output (a YAML file written to `intended/structured_configs/DC1-LEAF1A.yml`) contains things like:

```yaml
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
  vlans:
    - id: 100
      tenant: TENANT_A
      rd: "192.168.255.3:10100"
      route_targets:
        both: ["10100:10100"]
      redistribute_routes:
        - learned
```

Notice how *all the fiddly stuff* — RDs, RTs, VNIs, neighbor entries — has been computed. This is the value of a validated design: opinionated, consistent choices made by the framework.

### Step 3: `eos_cli_config_gen` role renders structured config → CLI text

The second role in the pipeline reads the structured config dict and renders it through **Jinja2 templates** to produce the actual EOS CLI configuration.

> **Jinja2 in 30 seconds** (see chapter 00 §L5 for the longer version): Jinja is a templating language — text files with `{{ placeholders }}` and `{% for/if %}` logic that get filled in from a data structure. A snippet from inside AVD's templates looks roughly like this:
>
> ```jinja
> {% for vlan in vlans | arista.avd.natural_sort('id') %}
> vlan {{ vlan.id }}
> {%     if vlan.name is arista.avd.defined %}
>    name {{ vlan.name }}
> {%     endif %}
> !
> {% endfor %}
> ```
>
> Fed the structured config below, that renders the EOS `vlan 100 / name USERS` stanza you'll see in the next code block. One template, any number of VLANs. AVD ships hundreds of templates like this — together they cover every EOS feature it supports.

The role reads the structured config dict and produces:

```
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
   vlan 100
      rd 192.168.255.3:10100
      route-target both 10100:10100
      redistribute learned
   !
```

This lands in `intended/configs/DC1-LEAF1A.cfg`. **At this point, no switch has been touched.** You've just generated text files.

### Step 4: `eos_config_deploy_eapi` (or `_cvp`) pushes to the device

`ansible-playbook deploy.yml` invokes the deploy role, which:
- Connects to each switch over eAPI (HTTP/JSON-RPC) — or to CloudVision if you use the CVP variant.
- Diffs the running config against the intended config.
- Applies the difference (typically as `configure session` → `commit`).

### Step 5: `eos_validate_state` confirms reality matches intent

After deployment, the validation role runs a battery of checks per device — BGP sessions up, MLAG healthy, expected VLANs present, VXLAN tunnels formed, interface counters not erroring — and produces a CSV/Markdown report.

## 4. The artifacts AVD produces (and why each matters)

After a build+deploy cycle, your repo (or a CI artifact bucket) contains:

```
intended/
├── configs/                       # full per-device CLI (.cfg)
│   ├── DC1-SPINE1.cfg
│   ├── DC1-LEAF1A.cfg
│   └── ...
└── structured_configs/            # per-device YAML (the dict from step 2)
    ├── DC1-SPINE1.yml
    └── ...
documentation/
├── fabric/
│   └── DC1.md                     # full fabric docs as Markdown
└── devices/
    ├── DC1-SPINE1.md              # per-device docs (interfaces, BGP, VLANs)
    └── ...
reports/
└── DC1-state-report.md            # validation results
```

| Artifact | What it's for |
|---|---|
| `intended/configs/*.cfg` | The contract. This is what the switch should look like. Review these in PR diffs. |
| `intended/structured_configs/*.yml` | Useful for debugging "why did AVD generate that?" — shows the *intent* before templating. |
| `documentation/devices/*.md` | Self-updating per-device docs. No more stale Confluence. |
| `documentation/fabric/DC1.md` | Fabric-level overview: topology, P2P links, ASNs, VLANs by tenant. |
| `reports/*-state-report.md` | Did the deploy actually result in the intended state? Gate your PR merges on this. |

> **Why a network engineer should care:** the `.cfg` files in `intended/configs/` are your single best review tool. When someone opens a PR changing a YAML file, the *diff in the generated `.cfg` files* is what tells you what will actually happen on the box.

## 5. The Git workflow around it

> **What's CI/CD in one paragraph?** **CI** = *Continuous Integration*: an automated service (GitHub Actions, GitLab CI, Jenkins) watches your repo and, on every commit or pull request, runs jobs like "lint the YAML", "regenerate the AVD configs and confirm they match what was committed", "run tests against a lab". **CD** = *Continuous Deployment*: when CI passes on the main branch, automatically deliver the change to production — for us, that's running `deploy.yml` against the fabric. In short: CI = "did we break anything?", CD = "ship it." For network teams, CI/CD turns a config change into a code-reviewable, automatically-tested, auditable event instead of an SSH session. We'll build a working CI pipeline in chapter 18.

In a real shop, the project repo lives in GitHub/GitLab and the workflow looks like:

1. Engineer opens a feature branch, edits the YAML inputs (e.g., adds a VLAN).
2. They run `ansible-playbook build.yml` locally — this regenerates the `.cfg` and docs files.
3. They commit both the YAML change *and* the regenerated artifacts.
4. They open a PR. **CI** re-runs `build.yml` to confirm reproducibility, runs linters, may run `eos_validate_state` against a lab.
5. Reviewers look at the **diff in the generated `.cfg` files** to see the real impact.
6. PR is merged. **CI/CD** (or a human) runs `deploy.yml` against production.
7. Post-deploy, `eos_validate_state` runs and the report is attached to the PR.

This is why Git + GitHub literacy is essential: AVD only delivers its value when wrapped in a code-review workflow.

## 6. Common confusions, cleared up early

**"Is AVD the same as Ansible?"**
No. AVD is built *on* Ansible. Ansible is the engine; AVD is the design + data model + roles that ride on it.

**"Do I need to learn all of Ansible?"**
No. As a consumer you mostly run two or three playbooks and edit YAML. You need to understand inventory hierarchy, group_vars, and basic playbook structure. Modules and writing your own roles are extender territory.

**"Where does CloudVision fit?"**
CloudVision is Arista's management platform. AVD can push configs *via* CloudVision (using the `_cvp` deploy role) instead of direct eAPI. The configs and workflow are identical; only the transport changes.

**"What about EVPN/VXLAN — do I need to know it?"**
Yes — AVD's defaults assume you know what you want. AVD will happily build a broken EVPN fabric if you give it nonsensical inputs. The whole point is to remove *config typing*, not *design thinking*.

**"What if my fabric doesn't fit AVD's L3LS model?"**
AVD supports a few designs (L3LS-EVPN, MPLS, L2LS), and you can also provide `structured_config` directly to skip the design layer for specific cases. But if you're 100% bespoke, AVD is overkill.

## 7. Terminology cheat sheet (AVD-specific)

| Term | Meaning |
|---|---|
| **Project repo** | Your own Git repo containing inventory + group_vars + playbooks. You own this. |
| **Collection** | `arista.avd` — the upstream code you depend on. |
| **Role** | `eos_designs`, `eos_cli_config_gen`, `eos_config_deploy_eapi`, `eos_validate_state` — the four main pipeline stages. |
| **Design** | A high-level fabric pattern: `l3ls-evpn`, `mpls`, `l2ls`. Selected via `design.type` in YAML. |
| **Node type** | What role a device plays: `spine`, `l3leaf`, `l2leaf`, `super-spine`. |
| **Node group** | A pair (or more) of nodes that share state — typically MLAG leaves. |
| **Structured config** | The intermediate dict the design role produces per device. |
| **Intended config** | The rendered `.cfg` file per device. |
| **Custom structured config** | YAML you inject directly into the structured config to override or extend AVD's output. The main extension mechanism for consumers. |

## 8. What you should be left wondering

By design, this tour skips a *lot*. You probably have questions like:
- "What does the inventory file actually look like, and how does the group hierarchy work?"
- "Where does Jinja2 come in — can I see a template?"
- "How does `ansible-playbook` actually run? What's a 'task'?"
- "Where do credentials for eAPI live? How is auth handled?"
- "What if I want to change something AVD doesn't expose as a YAML key?"
- "How do I diff two fabric versions to see what's changing?"

Write these down in your `notes/` folder. They're the seeds of the next 15 chapters.

---

## 🧪 Exercise

You're not running anything yet, but you *can* read the official examples in your browser:

1. Open the AVD documentation site: **<https://avd.arista.com>**. Find the "Getting Started" section.
2. Browse the **examples** directory on GitHub: **<https://github.com/aristanetworks/avd/tree/devel/ansible_collections/arista/avd/examples>**. Open the `single-dc-l3ls` example.
3. **Read one full `group_vars/` file** end to end — e.g., `DC1_FABRIC.yml`. Don't try to understand every key. Just notice the *shape*: nested dicts, lists of dicts, repeated patterns.
4. **Read one generated `.cfg` file** if you can find one in the example outputs (or search the docs for sample output). Notice how much config came from how little YAML.
5. Write a one-paragraph answer in your notes: **"In my own words, what does AVD do?"** Revisit and rewrite this after chapter 16.

---

**Next:** [02 — Lab Setup: containerlab + cEOS](02-lab-setup.md)
