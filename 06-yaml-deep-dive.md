# 06 — YAML Deep Dive

> **Goal of this chapter:** read and write YAML fluently. By the end you should be able to spot indentation bugs at a glance, know when to use a list vs a dict vs a string, understand anchors and merge keys, and not be surprised by YAML's "Norway problem" gotchas.

YAML is the language of AVD inputs, Ansible playbooks, containerlab topologies, CI configs, Kubernetes — basically every modern infrastructure tool. It's deceptively simple. The traps are subtle.

## 1. What YAML actually is

YAML stands for "YAML Ain't Markup Language" (yes, recursive). It's a **data serialization format** — a way to write structured data (maps, lists, strings, numbers, booleans) in a human-readable text form.

Two key properties:

- **Indentation defines structure.** No braces, no brackets. Indentation level = nesting level. Spaces only — *never* tabs.
- **Superset of JSON.** Anything valid JSON is valid YAML. YAML adds comments, anchors, multi-line strings, and a much friendlier syntax.

A common, accurate mental model: YAML is JSON with humans in mind.

## 2. The four building blocks

### Scalars (single values)

```yaml
name: leaf1                  # string
asn: 65101                   # integer
mtu: 9214                    # integer
holdtime: 90.5               # float
shutdown: false              # boolean
description: null            # null (also: ~ or empty)
```

Strings *usually* don't need quotes. Use quotes when:
- The value could be misread as something else: `version: "4.32"` (without quotes it's a float `4.32`, losing the trailing zero).
- The value contains special characters: `: # > | & * ! @ %`.
- You want to force string type: `vlan_id: "100"` (string) vs `vlan_id: 100` (int).

⚠ **The "Norway problem":** `country: NO` becomes the boolean `false` in YAML 1.1 because `NO` is interpreted as "no". Use `country: "NO"`. Other values that get type-coerced: `yes/no/on/off/true/false`, and sometimes `null` / `~`. YAML 1.2 fixed most of these; many tools still default to 1.1.

### Sequences (lists)

Block style (most common):

```yaml
spines:
  - spine1
  - spine2
  - spine3
```

Flow style (JSON-like, useful for short lists):

```yaml
spines: [spine1, spine2, spine3]
```

Both produce the same data. Use block for anything you want diff-friendly; flow for compact, one-line cases.

### Mappings (dicts / objects)

```yaml
fabric:
  asn_range: 65000-65500
  loopback_subnet: 10.255.0.0/16
  mlag_subnet: 10.255.252.0/22
```

Flow form:

```yaml
fabric: { asn_range: 65000-65500, loopback_subnet: 10.255.0.0/16 }
```

### Combined: lists of mappings, the AVD workhorse

This is what 90% of AVD inputs look like:

```yaml
svis:
  - id: 100
    name: USERS
    ip_address_virtual: 10.1.100.1/24
  - id: 200
    name: VOICE
    ip_address_virtual: 10.1.200.1/24
  - id: 300
    name: GUEST
    ip_address_virtual: 10.1.30.1/24
```

Read it as: "`svis` is a list of three dicts; each dict has `id`, `name`, `ip_address_virtual`."

Note the indentation: the `- id` starts at the same column as the parent's first letter would; subsequent keys (`name`, `ip_address_virtual`) align with `id`, not with `-`.

## 3. Indentation rules

- Use **spaces only**. Convention: 2 spaces per level. Some projects use 4; pick one and stick to it.
- A child must be indented *strictly more* than its parent.
- Sibling keys must be at the *exact same* indentation.

This is valid:

```yaml
device:
  hostname: leaf1
  interfaces:
    - name: Ethernet1
      description: to-spine1
    - name: Ethernet2
      description: to-spine2
```

This is invalid (mismatched sibling indentation):

```yaml
device:
  hostname: leaf1
   interfaces:           # 3 spaces, not 2 — error!
    - name: Ethernet1
```

⚠ **Tabs vs spaces**: YAML *explicitly forbids tabs* for indentation. Configure your editor: VS Code → bottom-right "Spaces: 2" / `"editor.insertSpaces": true`. The `redhat.vscode-yaml` extension catches tab-indent errors immediately.

## 4. Strings: the four styles

YAML has *four* ways to write a string. You will encounter all of them.

### Plain (unquoted)

```yaml
description: connects to spine1
```

Simplest. Subject to type coercion (the Norway problem). Cannot contain `:` or `#` followed by space.

### Single-quoted

```yaml
description: 'connects to spine1: trunk'
```

No escape sequences. To embed a single quote, double it: `'It''s the description'`.

### Double-quoted

```yaml
description: "connects to spine1\nuses 100G optics"
```

Supports `\n`, `\t`, `\"`, etc. Use when you need escape sequences.

### Block scalars (multi-line)

For long strings — e.g., a banner or a Jinja template embedded in YAML.

**Literal (`|`)** preserves newlines exactly:

```yaml
banner_motd: |
  Authorized access only.
  This system is monitored.
  Violators will be prosecuted.
```

**Folded (`>`)** collapses single newlines into spaces (paragraphs separated by blank lines):

```yaml
description: >
  This is one long
  logical line that
  will be joined into
  a single string.
```

Add a `-` (e.g., `|-` or `>-`) to *strip* the trailing newline. `|+` keeps trailing blank lines.

## 5. Comments

```yaml
# This is a comment — anything after # to end of line
asn: 65101  # inline comments work too
```

There is no block comment syntax. Use `#` on every line.

Use comments to explain *why* a value is set, not what it is — the YAML key already says what.

## 6. Anchors, aliases, and merge keys (DRY YAML)

When several entries share most of their fields, you can define them once with an **anchor** (`&name`), reference them with an **alias** (`*name`), and selectively override with the **merge key** (`<<`).

```yaml
defaults: &leaf_defaults
  uplink_type: port-channel
  uplink_switches: [SPINE1, SPINE2]
  mtu: 9214

l3_leaves:
  - hostname: LEAF1
    <<: *leaf_defaults                # inherit all fields from defaults
    id: 1
    mgmt_ip: 192.168.0.11/24
  - hostname: LEAF2
    <<: *leaf_defaults
    id: 2
    mgmt_ip: 192.168.0.12/24
    mtu: 1500                         # override just this one field
```

> **Heads-up:** AVD itself rarely uses YAML anchors because the data model has its own inheritance via group_vars / host_vars (chapter 14). Anchors are a YAML-level feature; AVD's inheritance is an Ansible-level feature. They can coexist but anchors complicate parsing and PR diffs, so most AVD repos avoid them.

## 7. Multi-document YAML

A single `.yml` file can contain *multiple* documents separated by `---`:

```yaml
---
device: leaf1
asn: 65101
---
device: leaf2
asn: 65102
```

Common in Ansible playbooks (one play per document) and Kubernetes manifests. Most YAML parsers expose this as a list of dicts.

A leading `---` at the top of a file is optional but conventional — many linters require it.

## 8. The most common mistakes (and how to spot them)

### Forgotten `-` in a list

```yaml
interfaces:
  name: Ethernet1            # ❌ this isn't a list item, it's a key under interfaces
  name: Ethernet2            # and now there are TWO `name` keys — undefined behaviour
```

Should be:

```yaml
interfaces:
  - name: Ethernet1
  - name: Ethernet2
```

### Mixed indentation

```yaml
node_groups:
  - name: MLAG1
    nodes:
      - leaf1
      - leaf2
  mlag: true                # ❌ aligned with the `-`, not with `name` — this is a sibling of the list item, not a key inside it
```

Should be:

```yaml
node_groups:
  - name: MLAG1
    nodes:
      - leaf1
      - leaf2
    mlag: true              # ✅ same indent as `name` and `nodes` — inside the list item
```

### Stringy version numbers becoming floats

```yaml
eos_version: 4.32           # ❌ becomes float 4.32 — loses trailing zeros
```

Use:

```yaml
eos_version: "4.32"         # ✅
```

### Trailing whitespace and tabs

Hard to spot visually. Run `yamllint` and configure VS Code to show whitespace:

```json
// VS Code settings.json
"editor.renderWhitespace": "all"
```

### `:` inside an unquoted string

```yaml
description: spine1: uplink     # ❌ YAML parses as { description: { "spine1": "uplink" } }
```

Use quotes:

```yaml
description: "spine1: uplink"
```

## 9. Validating YAML

### `yamllint` — style + correctness linter

```bash
pip install yamllint
yamllint -d "{rules: {line-length: {max: 120}}}" my-file.yml
yamllint .                                  # whole directory
```

Catches: indentation issues, duplicate keys, trailing whitespace, line length, missing document start markers, etc.

### Just parse it

If you only want to know "is this file syntactically valid?":

```bash
python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" my-file.yml
```

No output = OK. Exception = parse error with line number.

### `redhat.vscode-yaml` in your editor

Best feedback loop. With schema mapping configured (which AVD provides for its data model), you get red squiggles for unknown keys, type mismatches, missing required fields — all live.

## 10. YAML vs JSON: when to use which

| Use YAML when… | Use JSON when… |
|---|---|
| Humans will read/write it | A machine produces or consumes it directly |
| Comments matter | Strict performance or interop |
| You need anchors/aliases | A REST API requires JSON body |
| The file lives in Git for review | The data is short-lived (in-memory, on the wire) |

AVD's inputs are YAML for humans. AVD's *internal* structured config (the intermediate dict in chapter 01 §step 2) is often serialized to YAML or JSON for inspection. eAPI returns JSON. Same data, different format.

We cover the JSON side in [chapter 07](07-json-vs-yaml.md).

## 11. Real AVD YAML, annotated

A small chunk from a typical AVD project's `group_vars/CONNECTED_ENDPOINTS.yml`:

```yaml
servers:                              # list of server definitions
  - name: server01                    # logical server name
    rack: rack1                       # informational
    adapters:                         # NIC/port mappings
      - endpoint_ports: [eth0, eth1]  # server-side port names (used in docs)
        switch_ports: [Ethernet5, Ethernet5]
        switches: [LEAF1A, LEAF1B]    # MLAG pair; same port number on both
        profile: SERVER_MLAG          # named profile defined elsewhere
        port_channel:
          mode: active                # LACP
```

Three list levels deep, with `port_channel` being a nested dict. The `[a, b]` flow lists are used inline for the short paired values; the outer `adapters` and `servers` lists use block form for diff-friendliness.

## 12. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **Scalar** | A single value (string, number, bool, null). |
| **Sequence** | A list. Indicated by `-` or `[...]`. |
| **Mapping** | A dict / object. Indicated by `key: value`. |
| **Document** | A single top-level data structure in a YAML file. Multiple docs separated by `---`. |
| **Anchor (`&name`)** | Names a node for later reuse. |
| **Alias (`*name`)** | References an anchor. |
| **Merge key (`<<`)** | Merges another mapping into the current one. |
| **Block style** | The indented, multi-line form. |
| **Flow style** | The compact `{}`/`[]` form. |
| **Block scalar** | A multi-line string introduced with `|` or `>`. |

---

## 🧪 Exercise

In your repo, create `labs/06-yaml/`:

1. **Fix the bug** — copy the broken snippet below into `labs/06-yaml/broken.yml`. Make it valid YAML representing the obvious intent. Run `yamllint broken.yml` until it passes.

   ```yaml
    devices:
    -name: leaf1
       mgmt: 192.168.0.11
       interfaces:
        - Ethernet1
          - Ethernet2
   ```

2. **Norway problem hands-on**: write a `regions.yml` containing two countries:
   ```yaml
   countries:
     - code: NO
       name: Norway
     - code: SE
       name: Sweden
   ```
   Parse it with `python3 -c 'import yaml,json,sys; print(json.dumps(yaml.safe_load(open("regions.yml")), indent=2))'`. What is the actual type of `code` for Norway? Fix it with quoting and re-parse.

3. **Anchors**: rewrite the following without anchors first, then add anchors to eliminate duplication. Confirm both parse to the same dict structure.
   ```yaml
   l3_leaves:
     - hostname: LEAF1
       mtu: 9214
       uplinks: [SPINE1, SPINE2]
     - hostname: LEAF2
       mtu: 9214
       uplinks: [SPINE1, SPINE2]
     - hostname: LEAF3
       mtu: 9214
       uplinks: [SPINE1, SPINE2]
   ```

4. **Block scalar**: write a YAML file with a `motd:` key containing a 3-line banner. First use `|`, then `>`. Print the resulting string each time and observe the difference.

5. **Lint the repo's existing YAML**: run `yamllint labs/02-first-fabric/topology.clab.yml`. Fix any complaints (likely none — but if there are, that's chapter 02's bug to fix).

---

**Next:** [07 — JSON, and How It Differs From YAML](07-json-vs-yaml.md)
