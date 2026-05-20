# Network Automation Study — AVD Edition

A self-study curriculum for a network engineer ramping up on network automation, using **Arista AVD** (Arista Validated Designs) as the practical anchor.

## How this is organized

The approach is **top-down and AVD-driven**: we look at AVD first to understand the *shape* of what we're building, then go back and learn each layer of the stack (Git, YAML, Jinja, Ansible, Python, Linux, APIs) as it becomes relevant.

Every chapter is sized for ~30-60 minutes of reading plus a hands-on exercise on a local **containerlab + cEOS-lab** environment.

## Learning path

### Part 1 — Orientation
- **[00 — Network Automation 101](00-network-automation-101.md)** — the mental map: layers of the automation stack and where AVD fits.
- **01 — A Guided Tour of AVD** — look at a real AVD repo and trace YAML → playbook → device config, without setting anything up yet.

### Part 2 — Build your workbench
- **02 — Lab Setup: containerlab + cEOS** — get a 4-node fabric running locally.
- **03 — Linux & Bash Essentials for Automation** — just enough shell to be dangerous.
- **04 — Git & GitHub for Network Engineers** — version control as a network engineer's safety net.
- **05 — Devcontainers & VS Code** — reproducible dev environments so "works on my machine" goes away.

### Part 3 — The data layer
- **06 — YAML Deep Dive** — the language of AVD inputs; structure, anchors, gotchas.
- **07 — JSON, and How It Differs From YAML** — the language of APIs.
- **08 — Data Models: OpenConfig, YANG, and Vendor Models** — why structured data beats CLI scraping.

### Part 4 — Talking to devices
- **09 — APIs Overview: REST, eAPI, gNMI, NETCONF** — the protocols you'll actually use.
- **10 — Arista eAPI Hands-On** — call eAPI from `curl`, Python, and Postman.

### Part 5 — The orchestration layer
- **11 — Python Essentials for Network Engineers** — variables, data structures, functions, modules, virtualenvs.
- **12 — Jinja2 Templating** — how structured data becomes device config.
- **13 — Ansible Fundamentals** — playbooks, inventory, roles, collections.

### Part 6 — AVD in depth (consumer focus)
- **14 — AVD Deep Dive 1: Inputs & Fabric Variables** — fabric, group_vars, host_vars, structured config.
- **15 — AVD Deep Dive 2: Build, Deploy, Validate workflow** — the roles, the order, the outputs.
- **16 — AVD Deep Dive 3: Reading the Generated Artifacts** — intended config, documentation, snapshots.

### Part 7 — Closing the loop
- **17 — Validation with ANTA and `eos_validate_state`** — proving the network matches intent.
- **18 — CI/CD Basics with GitHub Actions** — running AVD on every PR.

## Prerequisites

You should already be comfortable with:
- Networking fundamentals (BGP, OSPF, VLANs, EVPN/VXLAN, EOS CLI).
- Reading config files and understanding what they do.
- Basic command line use.

You do **not** need to know Python, Ansible, Git, or Linux beyond surface level — those are covered in the curriculum.

## How to use this repo

1. Read chapters in order the first time through — they reference each other.
2. Do the hands-on exercise at the end of each chapter before moving on. Reading without typing is the #1 way to "learn" automation and then immediately forget it.
3. Keep a `notes/` folder of your own — write down what surprised you, what broke, and what you'd want to revisit. (Don't commit it if it has secrets.)
4. When something doesn't click, jump back to the prerequisite chapter rather than pushing through.

## Conventions used in chapters

- **Code blocks** are runnable — copy them.
- **`>` callouts** are "why a network engineer should care" notes.
- **⚠ Gotcha** boxes flag things that have bitten people in production.
- **🧪 Exercise** sections at the end of each chapter are mandatory if you want the chapter to stick.
