# 13 — Ansible Fundamentals

> **Goal of this chapter:** understand what Ansible is, how a playbook executes, how inventory + group_vars/host_vars work, what a role and a collection are, and how AVD's roles slot into all this. By the end you'll be able to read AVD's playbooks and write your own simple ones.

We've named-dropped Ansible repeatedly. Time to learn it properly.

## 1. What Ansible is

Ansible is a **task-based automation engine** that runs tasks *in order* against a list of target machines (your **inventory**), driven by YAML files called **playbooks**, with logic factored into reusable **roles** distributed as **collections**.

Two design choices that make it dominant in network shops:

- **Agentless.** No software needs to run on the target device — Ansible connects via SSH or HTTP-based APIs (eAPI, NETCONF, gNMI) to do its work.
- **YAML, not code.** A playbook reads like a checklist. Non-programmers can follow it.

Trade-off: you give up some flexibility vs writing a custom Python tool. For 95% of network ops, that trade is worth it.

## 2. The pieces

```
project/
├── ansible.cfg                 # behavior knobs (paths, defaults)
├── inventory.yml               # list of devices + groupings
├── group_vars/
│   ├── all.yml                 # vars for all hosts
│   ├── SPINES.yml              # vars for hosts in group SPINES
│   └── LEAVES.yml              # vars for hosts in group LEAVES
├── host_vars/
│   └── leaf1.yml               # vars for one specific host
├── roles/                      # local roles (rare in AVD projects)
├── collections/                # downloaded collections (or ~/.ansible/)
└── playbooks/
    ├── build.yml
    └── deploy.yml
```

In an AVD project, most of `roles/` is empty because AVD's roles come from the `arista.avd` collection. You only add local roles for project-specific glue.

## 3. Inventory: who you're talking to

Inventory is the list of devices Ansible will target, organized into groups. YAML format (Ansible also supports INI):

```yaml
all:
  children:
    FABRIC:
      children:
        DC1:
          children:
            DC1_FABRIC:
              children:
                DC1_SPINES:
                  hosts:
                    spine1:
                      ansible_host: 172.20.20.2
                    spine2:
                      ansible_host: 172.20.20.3
                DC1_LEAVES:
                  hosts:
                    leaf1:
                      ansible_host: 172.20.20.4
                    leaf2:
                      ansible_host: 172.20.20.5
```

Notice the hierarchy: `FABRIC > DC1 > DC1_FABRIC > DC1_SPINES + DC1_LEAVES`. This nesting is what AVD's group hierarchy plugs into.

Verify what Ansible sees:

```bash
ansible-inventory -i inventory.yml --list
ansible-inventory -i inventory.yml --graph
```

## 4. Variable precedence (the one chart you must memorize)

Variables can be defined in many places. When the same key exists in multiple places, Ansible picks one according to a fixed precedence (low → high; high wins):

1. role defaults
2. inventory file vars
3. inventory `group_vars/all`
4. inventory `group_vars/<group>`
5. inventory `host_vars/<host>`
6. play vars
7. task vars
8. command-line `-e` extra vars (always win)

For AVD specifically, you'll set most things in `group_vars/` so they cascade by group, and override per-host only when needed.

Quickly inspect the final merged vars for a host:

```bash
ansible -i inventory.yml leaf1 -m debug -a "var=hostvars[inventory_hostname]"
```

## 5. Playbooks

A playbook is a YAML file of *plays*. A play is "run these *tasks* against this set of *hosts*."

Minimal example:

```yaml
---
- name: Show version on all leaves
  hosts: DC1_LEAVES
  gather_facts: false
  tasks:
    - name: Run show version
      arista.eos.eos_command:
        commands: ["show version"]
      register: ver

    - name: Print version
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} → {{ ver.stdout[0].version }}"
```

Run:

```bash
ansible-playbook -i inventory.yml playbooks/show-version.yml
```

Key concepts:

- **`hosts:`** can be a group name (`DC1_LEAVES`), a hostname (`leaf1`), a wildcard (`leaf*`), or an intersection (`DC1:&LEAVES`).
- **`gather_facts: false`** — turn off automatic fact collection (network devices don't support most of it, and it's slow).
- **`register:`** — capture a task's return value into a variable.
- **`debug`** — print stuff.
- Modules are namespaced: `arista.eos.eos_command`, `ansible.builtin.debug`. Older syntax `eos_command` still works but is discouraged.

## 6. Tasks: the unit of work

Each task invokes one *module* (eos_command, file, copy, template, command…). Modules are idempotent when written properly — running twice doesn't change state on the second run if nothing's different.

A task is a dict with `name`, the module name and its args, and optional control keys:

```yaml
- name: Configure interface description
  arista.eos.eos_config:
    lines:
      - description provisioned-via-ansible
    parents: interface Ethernet1
  when: "'leaf' in inventory_hostname"      # only run on leaves
  tags: [interfaces]                        # selective execution
  delegate_to: localhost                    # run on the controller, not the target
  changed_when: false                       # tell Ansible this never "changes" things
```

## 7. Roles: reusable bundles

A role packages tasks, templates, defaults, and metadata. Layout:

```
roles/my_role/
├── tasks/main.yml
├── templates/foo.j2
├── defaults/main.yml
├── vars/main.yml
├── handlers/main.yml
├── meta/main.yml
└── README.md
```

Invoke from a playbook:

```yaml
- hosts: leaves
  roles:
    - my_role
    - role: arista.avd.eos_designs
    - role: arista.avd.eos_cli_config_gen
```

AVD ships these roles (the ones you actually consume):

| Role | What it does |
|---|---|
| `arista.avd.eos_designs` | Reads your high-level YAML inputs and produces per-device structured config dicts. |
| `arista.avd.eos_cli_config_gen` | Renders structured config dicts through Jinja templates into EOS CLI text. |
| `arista.avd.eos_config_deploy_eapi` | Pushes intended config to devices via eAPI. |
| `arista.avd.eos_config_deploy_cvp` | Same, via CloudVision. |
| `arista.avd.eos_validate_state` | After deployment, validates the running state matches intent. |
| `arista.avd.dhcp_provisioner` | Optional, for ZTP. |

You'll invoke these directly in your AVD project's `build.yml` and `deploy.yml`.

## 8. Collections

A **collection** bundles roles, modules, plugins, and filters under a namespace like `arista.avd`. Hosted on Ansible Galaxy or private hubs.

```bash
# Install latest
ansible-galaxy collection install arista.avd

# Install a specific version
ansible-galaxy collection install arista.avd:==5.2.0

# Install from a requirements file (recommended for reproducibility)
cat > collections/requirements.yml <<EOF
collections:
  - name: arista.avd
    version: "5.2.0"
  - name: arista.eos
    version: "10.1.0"
  - name: arista.cvp
    version: "3.12.0"
EOF
ansible-galaxy collection install -r collections/requirements.yml
```

In `ansible.cfg`, point at the install location:

```ini
[defaults]
collections_path = ./collections:~/.ansible/collections
```

## 9. `ansible.cfg`: behavior knobs

Minimal `ansible.cfg` for an AVD-style project:

```ini
[defaults]
inventory = ./inventory.yml
roles_path = ./roles:~/.ansible/roles
collections_path = ./collections:~/.ansible/collections
host_key_checking = False
forks = 20
gathering = explicit
stdout_callback = yaml

[ssh_connection]
pipelining = True
```

Settings of note:

- **`host_key_checking = False`** — necessary when targets are short-lived (lab containers).
- **`forks = 20`** — how many devices to operate on in parallel.
- **`gathering = explicit`** — don't auto-gather facts.
- **`stdout_callback = yaml`** — readable output format (alternative: `default`, `json`, `oneline`).

## 10. Running playbooks

Common invocations:

```bash
ansible-playbook -i inventory.yml playbooks/build.yml

# Limit to a subset
ansible-playbook -i inventory.yml playbooks/build.yml --limit leaf1
ansible-playbook -i inventory.yml playbooks/build.yml --limit DC1_LEAVES

# Tags
ansible-playbook playbooks/build.yml --tags interfaces
ansible-playbook playbooks/build.yml --skip-tags slow

# Dry run + diff
ansible-playbook playbooks/deploy.yml --check --diff

# Override a variable
ansible-playbook playbooks/build.yml -e "fabric_name=DC2"

# Verbose
ansible-playbook playbooks/build.yml -vvv
```

`--check --diff` is the network-engineer's safety net: "what *would* this do?".

## 11. Modules you'll use a lot

| Module | Use case |
|---|---|
| `ansible.builtin.debug` | Print variables, sanity checks |
| `ansible.builtin.set_fact` | Compute a variable to use in subsequent tasks |
| `ansible.builtin.template` | Render a Jinja template to a file |
| `ansible.builtin.copy` | Push a static file to a target |
| `ansible.builtin.file` | Create/remove dirs, set permissions |
| `ansible.builtin.lineinfile` | Edit a line in a file |
| `ansible.builtin.assert` | Halt with an error if a condition isn't met |
| `arista.eos.eos_command` | Run any EOS CLI command |
| `arista.eos.eos_config` | Push EOS config lines (idempotent) |
| `arista.eos.eos_facts` | Gather facts about an EOS device |

## 12. A realistic mini-playbook

Run a pre-change snapshot:

```yaml
---
- name: Snapshot all device configs before change
  hosts: FABRIC
  gather_facts: false
  vars:
    snapshot_dir: snapshots/{{ '%Y-%m-%dT%H%M%S' | strftime }}
  tasks:
    - name: Ensure snapshot dir exists
      ansible.builtin.file:
        path: "{{ snapshot_dir }}"
        state: directory
      delegate_to: localhost
      run_once: true

    - name: Fetch running-config
      arista.eos.eos_command:
        commands: ["show running-config"]
      register: rc

    - name: Save per-device
      ansible.builtin.copy:
        content: "{{ rc.stdout[0] }}"
        dest: "{{ snapshot_dir }}/{{ inventory_hostname }}.cfg"
      delegate_to: localhost
```

Notice:

- `delegate_to: localhost` — file ops happen on the control node, not the device.
- `run_once: true` — only one host needs to create the directory.

## 13. AVD playbook patterns (preview)

A typical AVD project's `build.yml`:

```yaml
---
- name: Build EOS configs
  hosts: FABRIC
  gather_facts: false
  connection: local
  collections:
    - arista.avd
  tasks:
    - name: Generate structured configs
      ansible.builtin.import_role:
        name: arista.avd.eos_designs

    - name: Render EOS CLI configs + docs
      ansible.builtin.import_role:
        name: arista.avd.eos_cli_config_gen
```

And `deploy.yml`:

```yaml
---
- name: Deploy EOS configs
  hosts: FABRIC
  gather_facts: false
  connection: local
  collections:
    - arista.avd
  tasks:
    - name: Push intended config via eAPI
      ansible.builtin.import_role:
        name: arista.avd.eos_config_deploy_eapi
```

We'll dissect both in [chapter 15](15-avd-build-deploy.md).

## 14. Troubleshooting Ansible runs

- **Use `-vvv`** for verbose output. `-vvvv` shows the raw SSH/HTTP traffic.
- **`--check` + `--diff`** is your friend. Always run before `deploy`.
- **Read tracebacks bottom-up.** The error message at the very bottom is the cause; everything above is plumbing.
- **`ansible-playbook --start-at-task "Render configs"`** to resume mid-playbook after a failure.
- **Inspect facts**: `ansible -i inventory.yml leaf1 -m setup` (slow on network devices).

⚠ When something fails on one host but not others, suspect either inventory differences or per-host vars. Run `ansible-inventory --host leaf1 -i inventory.yml` to see the exact merged vars for that host.

## 15. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **Inventory** | The list of target hosts + their grouping. |
| **Host** | A target device (or local control node). |
| **Group** | A named set of hosts. |
| **Play** | A YAML doc that maps tasks to hosts. |
| **Task** | One module invocation. |
| **Module** | A unit of executable functionality (`debug`, `eos_config`, etc.). |
| **Role** | A reusable bundle of tasks + templates + defaults. |
| **Collection** | A bundle of roles/modules/plugins under a namespace. |
| **Handler** | A task that runs only if notified (typical: restart service). |
| **Vars** | Variables — see precedence table in §4. |
| **Facts** | Auto-gathered info about a host. |
| **`delegate_to`** | Run this task on a different host. |
| **`run_once`** | Run this task only on the first host in the play. |
| **Idempotent** | Re-running produces the same end state. |
| **Connection plugin** | How Ansible reaches the host (`ssh`, `httpapi`, `network_cli`, `local`). |

---

## 🧪 Exercise

In `labs/13-ansible/`:

1. **Inventory + ping**: write an inventory of your four lab nodes (`spine1/2`, `leaf1/2`) with IPs. Use `ansible -i inventory.yml all -m ping` to confirm — note that for network devices you often need `ansible_connection: ansible.netcommon.httpapi` etc.; for now, install python on the lab nodes via `apt` is not an option (cEOS), so target the *Mac*/orb VM as a sanity check.

2. **eos_command playbook**: write `show-version.yml` that runs `show version` on all four lab nodes via the `arista.eos.eos_command` module. You'll need inventory vars `ansible_connection: ansible.netcommon.httpapi`, `ansible_network_os: arista.eos.eos`, `ansible_user: admin`, `ansible_password: admin`, `ansible_httpapi_use_ssl: true`, `ansible_httpapi_validate_certs: false`.

3. **`group_vars` cascade**: split the common vars (`ansible_user`, `ansible_connection`, etc.) into `group_vars/all.yml`. Confirm with `ansible-inventory --host leaf1 -i inventory.yml` that the merged vars include them.

4. **Config push**: write a playbook that uses `arista.eos.eos_config` to set `description set-by-ansible` on `Ethernet2` of each device. Run with `--check --diff` first. Then apply.

5. **Idempotency check**: re-run the previous playbook. The output should report `changed=0` — that's idempotency.

6. **Read an AVD role**:
   ```bash
   ls ~/.ansible/collections/ansible_collections/arista/avd/roles/eos_cli_config_gen/
   ```
   Notice the lack of a `tasks/main.yml` — AVD uses a different role pattern (plugins + filters drive most behavior). Read `meta/argument_specs.yml` to see what inputs the role expects.

---

**Next:** [14 — AVD Deep Dive 1: Inputs & Fabric Variables](14-avd-inputs.md)
