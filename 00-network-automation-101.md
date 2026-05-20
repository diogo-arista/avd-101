# 00 — Network Automation 101

> **Goal of this chapter:** give you a *mental map* of the entire network automation stack — not deep knowledge of any one piece. By the end you should be able to look at any tool (Ansible, Nornir, Terraform, AVD, gNMI, Batfish, etc.) and slot it into the right layer.

## 1. Why automation, in network-engineer terms

You already know the pain:
- You SSH into 48 leaves to make the same VLAN change.
- A typo on one device causes a 2 AM page.
- Six months later, nobody remembers *why* that prefix-list exists.
- A new fabric takes 3 weeks to stand up and 2 of them are config copy-paste.

Automation isn't really about saving keystrokes. It's about three things:

1. **Consistency** — every device gets the *same* config built the *same* way from the *same* inputs. No drift.
2. **Source of truth** — the network's intended state lives in version control, not in someone's head or on the running config.
3. **Repeatability** — building fabric #2 takes hours instead of weeks because the work is templated.

AVD is essentially Arista's opinionated, production-grade implementation of all three.

## 2. The mental model: intent → render → push → validate

Almost every modern network automation tool — including AVD — follows this loop:

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  1. INTENT   │ → │  2. RENDER   │ → │   3. PUSH    │ → │ 4. VALIDATE  │
│  (YAML/JSON) │   │  (templates) │   │  (eAPI/SSH/  │   │  (state ==   │
│              │   │              │   │  NETCONF/    │   │   intent?)   │
│              │   │              │   │   gNMI)      │   │              │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
       ▲                                                          │
       └──────────────── feedback / drift detection ──────────────┘
```

| Step | What it means | Example tool |
|---|---|---|
| **1. Intent** | Describe what you *want*, in structured data (not CLI). | YAML files in a Git repo |
| **2. Render** | Turn intent into device-specific config. | Jinja2 templates |
| **3. Push** | Deliver the config to the device. | Ansible + eAPI / SSH |
| **4. Validate** | Confirm the device's actual state matches intent. | ANTA, `eos_validate_state` |

> **Why a network engineer should care:** the loop is the unit of value. Tools that only do "push" (Netmiko, Paramiko) are useful but limited. Tools that do all four (AVD) are what teams build platforms on.

## 3. The layered stack

Think of network automation as a layered stack, similar to OSI but for tooling:

```
┌─────────────────────────────────────────────────────┐
│  L7: Frameworks / Validated Designs                 │  ← AVD, Cisco NSO services
├─────────────────────────────────────────────────────┤
│  L6: Orchestration                                  │  ← Ansible, Nornir, Salt, Terraform
├─────────────────────────────────────────────────────┤
│  L5: Templating                                     │  ← Jinja2, Go templates
├─────────────────────────────────────────────────────┤
│  L4: Data Models                                    │  ← OpenConfig, YANG, vendor models
├─────────────────────────────────────────────────────┤
│  L3: Data Formats                                   │  ← YAML, JSON, XML
├─────────────────────────────────────────────────────┤
│  L2: Transport / APIs                               │  ← eAPI (REST), gNMI, NETCONF, SSH
├─────────────────────────────────────────────────────┤
│  L1: Programming & Environment                      │  ← Python, Bash, Linux, Git, Docker
└─────────────────────────────────────────────────────┘
```

You don't need mastery of every layer to be useful — but you need *literacy* at each. The curriculum is structured to give you that.

### L1 — Programming & environment
The foundation. **Python** dominates network automation (Ansible is Python, Nornir is Python, most vendor SDKs are Python). You don't need to be a software engineer — you need to read code, write small scripts, and understand data structures (dicts, lists). **Git** stores your intent. **Linux/Bash** is where everything runs. **Docker/containerlab** is how you build labs.

### L2 — Transport / APIs
How you actually *talk* to a device.
- **SSH/CLI scraping** — what Netmiko, Napalm, and `expect` do. Universal but fragile.
- **eAPI** — Arista's HTTP/JSON-RPC interface. Send CLI commands, get structured JSON back. The Arista workhorse.
- **NETCONF** — XML over SSH. Standardized (RFC 6241), used heavily by Cisco IOS-XR, Juniper.
- **gNMI** — Google's gRPC-based protocol. Streaming telemetry + config. Modern, fast, vendor-neutral.
- **REST APIs** — CloudVision, controllers, IPAM systems.

### L3 — Data formats
The *syntax* you write/parse data in.
- **YAML** — human-friendly, indentation-sensitive, the dominant input format for Ansible, AVD, Kubernetes, CI configs.
- **JSON** — machine-friendly, the dominant API response format. eAPI returns JSON.
- **XML** — older, NETCONF uses it. You'll read it occasionally.

A common confusion: YAML and JSON are interchangeable in many tools (YAML is a superset of JSON). Use whichever is more readable for the context.

### L4 — Data models
The *semantics* — what fields exist, what type they are, what's required.
- **OpenConfig** — vendor-neutral models for routing, interfaces, BGP, etc. Maintained by an industry group.
- **YANG** — the modeling *language* itself (RFC 7950). OpenConfig models are written in YANG.
- **Vendor-native models** — Arista's, Cisco's, Juniper's own YANG models. Often richer than OpenConfig but not portable.
- **AVD's data model** — Arista's structured input format for fabrics. Documented per-key in the AVD docs site.

> **Why this matters:** without a model, "interface description" might be a string for one tool and an object for another. Models give automation tools something stable to read/write against.

### L5 — Templating
Turning structured intent into device-specific text config.
- **Jinja2** — a templating language originally built for Python web frameworks, but now everywhere. Used by Ansible, AVD, SaltStack, Helm-adjacent tools. You'll write/read a *lot* of this.

**What Jinja actually is, in plain terms:**

Think of Jinja as "config files with holes in them that get filled in from data." You write a text file (the *template*) that looks almost like a final config, but with placeholders for the variable parts. When the template is *rendered*, Jinja walks through it, replaces the placeholders using values from a data structure (a dict / YAML file), and spits out plain text.

Jinja has three syntax markers:

| Marker | Meaning | Example |
|---|---|---|
| `{{ expr }}` | **Output** — evaluate the expression and insert it into the output. | `interface {{ name }}` |
| `{% statement %}` | **Logic** — `if`, `for`, `set`, `include`, etc. Controls flow, produces no output itself. | `{% for v in vlans %}...{% endfor %}` |
| `{# comment #}` | **Comment** — stripped from output. | `{# this VLAN is for users #}` |

Concrete example. Given this data (think of it as a YAML file loaded into a Python dict):

```yaml
hostname: leaf1
vlans:
  - id: 100
    name: USERS
  - id: 200
    name: VOICE
  - id: 300
    name: GUEST
```

…and this Jinja template:

```jinja
hostname {{ hostname }}
!
{% for v in vlans %}
vlan {{ v.id }}
   name {{ v.name }}
!
{% endfor %}
```

…the rendered output is:

```
hostname leaf1
!
vlan 100
   name USERS
!
vlan 200
   name VOICE
!
vlan 300
   name GUEST
!
```

The template has *no idea* how many VLANs you'll throw at it. Add a fourth VLAN to the YAML, re-render, and a fourth stanza appears. That's the magic: **one template can describe a pattern; the data dictates how many times that pattern repeats.**

Jinja also supports:

- **Conditionals** — `{% if interface.lacp %}lacp port-priority 100{% endif %}`
- **Filters** — pipe-style transforms: `{{ name | upper }}` → uppercases; `{{ ip | ipaddr('network') }}` → networking-aware helpers.
- **Includes / macros** — break large templates into reusable pieces. AVD does this heavily.
- **Whitespace control** — `{%- ... -%}` strips surrounding whitespace, useful when generating tightly-formatted CLI.

**Why a network engineer should care:** every line of EOS config that AVD produces was generated by a Jinja template that lives in the `eos_cli_config_gen` role. When you eventually need to understand *why* AVD generated some specific line of config, you'll read a `.j2` template file and trace its data inputs. We do a full chapter on Jinja later — for now, just know that the `{{ }}` and `{% %}` markers you see in AVD templates are Jinja, and they're not scary.

### L6 — Orchestration
The engine that runs the loop across many devices.
- **Ansible** — agentless, YAML playbooks, huge ecosystem. AVD is built on Ansible. The default for network teams.
- **Nornir** — Python-native alternative to Ansible. More flexible, less out-of-the-box.
- **Terraform** — declarative state management. Common for cloud, growing in networking.
- **Salt / Puppet / Chef** — broader IT config management; less common in pure network shops.

### L7 — Frameworks / Validated Designs
Opinionated stacks that wrap all the lower layers into a turnkey solution.
- **AVD (Arista Validated Designs)** — Ansible collection + data model + templates + validation, productized as "describe your fabric in YAML, get a working L3LS EVPN/VXLAN fabric."
- **Cisco NSO**, **Juniper Apstra** — vendor equivalents.

## 4. Where AVD fits

AVD is a vertical slice through layers 1–7:

```
L7: AVD (the framework itself)
L6: Ansible (collections: arista.avd, arista.eos, arista.cvp)
L5: Jinja2 (templates inside the eos_designs and eos_cli_config_gen roles)
L4: AVD data model (documented per-key, YANG-adjacent in spirit)
L3: YAML (inputs) + JSON (intermediate "structured_configs")
L2: eAPI / SSH / CloudVision API (for pushing config)
L1: Python (AVD plugins, custom filters), Linux, Docker, Git
```

What AVD gives you out of the box:
- A **data model** for L3LS spine/leaf fabrics with EVPN/VXLAN.
- A **role pipeline**: `eos_designs` → `eos_cli_config_gen` → `eos_config_deploy_eapi` (or `_cvp`).
- **Generated artifacts**: per-device `intended/configs/*.cfg`, per-device YAML documentation, fabric documentation in Markdown, snapshots for change review.
- **Validation**: `eos_validate_state` role + ANTA tests to confirm the deployed fabric matches intent.

What AVD does *not* give you:
- Magic. Bad inputs → bad configs.
- Day-2 operational tooling (use ANTA, CloudVision).
- Non-Arista support (it's purpose-built for EOS).

## 5. Key terminology cheat sheet

| Term | What it means |
|---|---|
| **Source of truth (SoT)** | The system that holds the *intended* state of the network. For us: a Git repo of YAML. |
| **Idempotent** | Running the same operation twice produces the same result. Ansible aims for this. |
| **Declarative** | You say *what* you want; the tool figures out *how*. (vs imperative.) |
| **Inventory** | The list of devices you're managing + their metadata. |
| **Playbook** | An Ansible script — a YAML list of tasks to run against an inventory. |
| **Role** | A reusable bundle of Ansible tasks, templates, and defaults. AVD ships several. |
| **Collection** | A bundle of roles/modules/plugins. `arista.avd` is a collection. |
| **Structured config** | Device intent as a structured data tree (dict) before it's rendered to CLI text. |
| **Intended config** | The final CLI config AVD generates and plans to push to the device. |
| **Drift** | When the running config no longer matches the intended config. |

## 6. How to read this curriculum

You picked **top-down, AVD-driven**, which means:
1. **Chapter 01** will walk through a real AVD repo *without setting anything up* — just to anchor what we're building toward.
2. **Chapter 02** gets your local lab running so you can actually execute things.
3. **Chapters 03–13** teach the underlying layers (Git, Linux, YAML, Jinja, Python, Ansible, APIs) — but always referencing back to "here's where you'll see this in AVD".
4. **Chapters 14–18** are AVD deep-dives, where the earlier chapters pay off.

Trust the sequence. The top-down jump in Chapter 01 will leave you with questions — that's intentional. Those questions are the hooks the later chapters hang on.

---

## 🧪 Exercise

No coding for this chapter. Instead:

1. **Draw the loop** (intent → render → push → validate) on paper, and annotate it with the *current* manual steps you do at work. Which step is most painful today?
2. **Map your current tools** to the 7-layer stack. If you've used `show run`, `expect` scripts, or Postman against an API — slot them in.
3. **Write down 3 questions** you have about AVD or automation in general. Keep these in a `notes/` file. We'll come back to them.

---

**Next:** [01 — A Guided Tour of AVD](01-avd-guided-tour.md) *(coming next)*
