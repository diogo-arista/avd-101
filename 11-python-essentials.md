# 11 — Python Essentials for Network Engineers

> **Goal of this chapter:** get you fluent enough in Python to *read* AVD's filters, *write* small scripts against eAPI, and *modify* Ansible's filter/plugin Python when needed. Not a CS course — a network-engineer-grade Python toolkit.

You don't need to become a software engineer. You need to read structures, write 50–100 line scripts, and not be afraid when you crack open a `.py` file inside an Ansible role.

## 1. Why Python

- Ansible is Python. AVD's plugins, filters, and modules are Python.
- Most network SDKs (pyeapi, ncclient, pygnmi, paramiko, netmiko, napalm) are Python.
- It's a clean, readable language — closer to pseudocode than to Bash.
- The standard library covers 80% of what you need (HTTP, JSON, YAML, regex, file IO, subprocess).

## 2. Python on your machine

```bash
python3 --version       # should be 3.10+ in your venv
which python3
```

Inside the orb VM, the chapter-02 venv at `~/.venvs/avd` is your "active" Python:

```bash
source ~/.venvs/avd/bin/activate    # activates
deactivate                          # leaves
```

The shell prompt shows `(avd)` when the venv is active.

> **Why venvs again?** Each project's dependencies are isolated. `pip install` only affects the active venv, not system Python. We covered this in [chapter 02 §10](02-lab-setup.md).

## 3. The basics in one screen

```python
# Variables and types
hostname = "leaf1"           # str
asn = 65101                   # int
holdtime = 90.5               # float
enabled = True                # bool
nothing = None                # NoneType (like null)

# Lists (ordered, mutable)
spines = ["spine1", "spine2"]
spines.append("spine3")
first = spines[0]
last = spines[-1]
slice_ = spines[1:3]          # ["spine2", "spine3"]

# Dicts (key-value, like JSON objects)
device = {
    "hostname": "leaf1",
    "asn": 65101,
    "interfaces": ["Ethernet1", "Ethernet2"],
}
device["hostname"]            # "leaf1"
device.get("missing", "default")   # safe lookup

# Sets (unique, unordered)
uniq = {"a", "b", "a"}        # {"a", "b"}

# Tuples (immutable lists)
pair = ("leaf1", "10.0.0.1")

# f-strings: formatted strings
print(f"Device {hostname} has ASN {asn}")

# Conditionals
if asn > 65000 and asn < 70000:
    print("private ASN range")
elif asn == 0:
    print("unset")
else:
    print("public")

# Loops
for spine in spines:
    print(spine)

for i, spine in enumerate(spines, start=1):
    print(f"{i}: {spine}")

for key, value in device.items():
    print(f"{key} = {value}")

# Comprehensions (concise loops with output)
upper_spines = [s.upper() for s in spines]                       # list comp
asns = {f"leaf{i}": 65100 + i for i in range(1, 5)}              # dict comp
active = [d for d in devices if d["enabled"]]                    # filtering
```

## 4. Functions

```python
def is_private_asn(asn: int) -> bool:
    """Return True if asn is in the private 4-byte range."""
    return 64512 <= asn <= 65534 or 4_200_000_000 <= asn <= 4_294_967_294

# Default args, keyword args
def connect(host, username="admin", password="admin", port=443):
    print(f"connecting to {username}@{host}:{port}")

connect("leaf1")
connect("leaf2", port=8443)
connect(host="leaf3", username="netops")

# *args / **kwargs (variable arguments)
def log(level, *messages, **context):
    for m in messages:
        print(f"[{level}] {m} {context}")

log("INFO", "BGP up", device="leaf1", peer="10.0.0.1")
```

Type hints (`asn: int`, `-> bool`) are optional but recommended — they're the difference between "what does this take?" requiring a guess vs being self-documenting.

## 5. Modules and imports

```python
import json                          # standard library
import yaml                          # third-party (pip install pyyaml)
import pyeapi
from pathlib import Path             # cherry-pick from a module
from collections import defaultdict

# Read a YAML file
data = yaml.safe_load(Path("group_vars/FABRIC.yml").read_text())

# Write JSON
Path("out.json").write_text(json.dumps(data, indent=2))
```

## 6. File I/O

```python
from pathlib import Path

# Read entire file as text
text = Path("config.cfg").read_text()

# Write text
Path("out.cfg").write_text("hostname leaf1\n")

# Iterate line by line (for big files)
with open("big.log") as f:
    for line in f:
        if "ERROR" in line:
            print(line.rstrip())

# JSON
import json
with open("data.json") as f:
    obj = json.load(f)
with open("out.json", "w") as f:
    json.dump(obj, f, indent=2)

# YAML
import yaml
with open("inputs.yml") as f:
    data = yaml.safe_load(f)
```

Always use `yaml.safe_load`, never plain `yaml.load` — the latter can execute arbitrary code from a hostile YAML file. AVD inputs aren't hostile, but the habit matters.

## 7. Error handling

```python
try:
    result = node.enable("show frobnicate")
except pyeapi.eapilib.CommandError as exc:
    print(f"eAPI rejected the command: {exc}")
except Exception as exc:
    print(f"Unexpected: {exc}")
    raise
```

Rules of thumb:
- Catch specific exceptions, not bare `Exception`.
- Don't swallow errors silently — at least log them.
- Re-raise (`raise`) if you can't actually handle the error.

## 8. The standard library you'll actually use

| Module | What you'll do with it |
|---|---|
| `json`, `yaml` | Parse/emit data formats. |
| `pathlib` | File paths without string concatenation. |
| `subprocess` | Run shell commands from Python. |
| `os`, `os.path` | Env vars (`os.environ`), filesystem. |
| `argparse` | Parse CLI args. |
| `logging` | Structured logs instead of `print`. |
| `re` | Regular expressions. |
| `datetime` | Dates and times. |
| `csv` | Read/write CSVs (validation reports, inventory dumps). |
| `ipaddress` | Parse/manipulate IP addresses and networks. |
| `collections` | `defaultdict`, `Counter`, `OrderedDict`. |
| `concurrent.futures` | Easy thread/process pools for parallel work. |

Example using `ipaddress`:

```python
import ipaddress
net = ipaddress.ip_network("10.255.0.0/24")
print(net.network_address, net.broadcast_address, net.num_addresses)
for i, host in enumerate(net.hosts()):
    if i < 5: print(host)
```

## 9. Third-party libraries common in network code

```bash
pip install requests pyyaml pyeapi rich httpx jinja2 ansible-core
```

- `requests` — HTTP client. The most-used third-party library, period.
- `httpx` — async HTTP client. Use when you need parallelism.
- `rich` — pretty terminal output (tables, trees, syntax highlighting).
- `jinja2` — templating engine (chapter 12).
- `ansible-core` — Ansible itself, importable from Python in plugin-writing scenarios.
- `pydantic` — data validation with type hints; AVD uses this internally.

## 10. A complete example: AVD-adjacent script

Pull every device's BGP peer state and write a markdown report:

```python
#!/usr/bin/env python3
"""Generate a quick BGP status report for the lab fabric."""
import pyeapi
from pathlib import Path
from concurrent.futures import ThreadPoolExecutor

DEVICES = {
    "spine1": "172.20.20.2",
    "spine2": "172.20.20.3",
    "leaf1":  "172.20.20.4",
    "leaf2":  "172.20.20.5",
}

def get_peers(name, ip):
    conn = pyeapi.connect(host=ip, username="admin", password="admin",
                          transport="https")
    node = pyeapi.client.Node(conn)
    r = node.enable("show ip bgp summary")[0]["result"]
    peers = r["vrfs"]["default"].get("peers", {})
    return name, [(p, info["peerState"]) for p, info in peers.items()]

with ThreadPoolExecutor() as ex:
    results = dict(ex.map(lambda kv: get_peers(*kv), DEVICES.items()))

lines = ["# BGP report\n"]
for dev, peers in results.items():
    lines.append(f"\n## {dev}\n")
    if not peers:
        lines.append("_no peers_\n")
    else:
        lines.append("| Peer | State |\n|---|---|\n")
        for p, s in sorted(peers):
            lines.append(f"| {p} | {s} |\n")

Path("bgp-report.md").write_text("".join(lines))
print("wrote bgp-report.md")
```

Save as `scripts/bgp_report.py`, `chmod +x`, run, and view the output. This is the shape of about 60% of all "ad-hoc network automation" scripts.

## 11. Reading Ansible / AVD Python

Sooner or later you'll open a file like `roles/eos_designs/python_modules/some_filter.py`. Here's what to expect:

```python
from ansible.errors import AnsibleFilterError

def my_filter(value, suffix=""):
    """A simple Ansible Jinja2 filter."""
    if not isinstance(value, str):
        raise AnsibleFilterError("expected a string")
    return f"{value}{suffix}"

class FilterModule:
    def filters(self):
        return {"my_filter": my_filter}
```

- A `FilterModule` class with a `.filters()` method that returns a dict of name → function. That's how Ansible exposes Jinja filters.
- Modules that talk to devices follow a similar plugin pattern (`ActionModule`, `Cliconf`, `Httpapi`, `Inventory`, etc.).
- AVD adds layers (Pydantic models for inputs, code generators, internal "schema → docs" pipeline). You don't need to write these to be a consumer.

You don't have to *write* this style of plugin code to use AVD. But being able to *read* it unlocks debugging in real projects.

## 12. Tooling we recommend

- **`ipython`** — a better REPL. `pip install ipython`; type `ipython` instead of `python`. Has tab completion, `?` for docs, `%timeit` for benchmarks.
- **`ruff`** — fast Python linter/formatter. `pip install ruff && ruff check . && ruff format .`.
- **`black`** — older formatter; ruff supersedes it for new projects.
- **`mypy` or `pyright`** — static type checkers.
- VS Code's **Python extension** (`ms-python.python`) — already in the devcontainer from chapter 05.

## 13. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **REPL** | Read-Eval-Print Loop — the interactive Python prompt. |
| **Module** | A `.py` file you can `import`. |
| **Package** | A directory of modules with an `__init__.py`. |
| **Virtual environment** | Isolated set of installed packages per project. |
| **pip** | The Python package installer. |
| **PyPI** | The Python Package Index — where pip downloads packages from. |
| **dict / list / tuple / set** | The core container types. |
| **Comprehension** | Concise `[x for x in iter]` syntax. |
| **f-string** | `f"hello {name}"` — formatted string literal. |
| **Lambda** | Anonymous one-line function: `lambda x: x.upper()`. |
| **Decorator** | A function that wraps another function: `@retry`. |
| **Context manager** | The `with open(...) as f:` pattern — auto-cleanup. |

---

## 🧪 Exercise

1. **Hello, fabric**: write `scripts/list_devices.py` that prints "name → IP" for every device in the lab, sourced from a YAML file you create at `inventory/devices.yml`. Use `yaml.safe_load` and an f-string loop.

2. **Refactor a bash script**: take the `check_lab.sh` from chapter 03 §exercise 4 and rewrite it in Python using `subprocess.run` to call `ping`. Print a table with `rich`.

3. **Read the AVD source**: clone the AVD repo and grep for a filter:
   ```bash
   cd ~ && git clone https://github.com/aristanetworks/avd.git
   grep -rn "def natural_sort" avd/python-avd/
   ```
   Read the function. Try calling it from a small script of your own (after `pip install pyavd`).

4. **JSON to YAML script**: write a 10-line script that takes a path argument and prints the file converted (autodetect by extension).

5. **Parallel scan with `concurrent.futures`**: adapt the §10 example to also fetch `show version` and add a `version` column. Time the script with and without the thread pool to see the speedup.

6. **REPL exploration**: `ipython`, then:
   ```python
   import pyeapi
   conn = pyeapi.connect(host="172.20.20.3", username="admin",
                         password="admin", transport="https")
   node = pyeapi.client.Node(conn)
   r = node.enable("show interfaces")
   r?              # IPython: open docs
   type(r)
   len(r[0]["result"]["interfaces"])
   ```
   The REPL is the fastest way to learn an unfamiliar API.

---

**Next:** [12 — Jinja2 Templating](12-jinja2-templating.md)
