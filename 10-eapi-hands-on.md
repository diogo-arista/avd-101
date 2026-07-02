# 10 — Arista eAPI Hands-On

> **Goal of this chapter:** call eAPI from `curl`, then from Python via `pyeapi`, run config commands, batch multiple commands, and handle errors. By the end you should be comfortable scripting any `show` or `config` you can do interactively on EOS.

This chapter uses your containerlab fabric from chapter 02. Make sure all four nodes are up: `containerlab inspect -t labs/02-first-fabric/topology.clab.yml`.

## 1. Enable eAPI on the lab (recap)

From chapter 02 §9, on each device:

```
configure
management api http-commands
   no shutdown
end
write
```

Confirm reachability from inside the orb VM:

```bash
curl -sk -u admin:admin https://172.20.20.3/command-api \
     -d '{"jsonrpc":"2.0","method":"runCmds","params":{"version":1,"cmds":["show version"],"format":"json"},"id":1}' \
     | jq -r '.result[0].version'
```

If you get an EOS version string back, you're set.

## 2. Anatomy of an eAPI request

Build a request file `req-show-version.json`:

```json
{
  "jsonrpc": "2.0",
  "method":  "runCmds",
  "params": {
    "version": 1,
    "cmds":    ["show version"],
    "format":  "json"
  },
  "id": 1
}
```

Send it:

```bash
curl -sk -u admin:admin https://172.20.20.3/command-api -d @req-show-version.json | jq .
```

The `-d @file.json` syntax tells curl to read the body from a file — much cleaner than embedding JSON in the shell.

### `format: json` vs `format: text`

Switch the format:

```json
"format": "text"
```

You'll get the *raw text output* of `show version`, exactly as a human would see it, wrapped in a JSON envelope. Useful for commands EOS doesn't (yet) have JSON output for, or for diff-friendly snapshots.

For most use, stick with `json` — that's the whole point.

## 3. Multiple commands in one request

```json
{
  "jsonrpc": "2.0",
  "method":  "runCmds",
  "params": {
    "version": 1,
    "cmds":    ["show hostname", "show version", "show ip interface brief"],
    "format":  "json"
  },
  "id": 1
}
```

The response's `result` is a list with three entries, one per command, in the same order.

```bash
curl -sk -u admin:admin https://172.20.20.3/command-api -d @req-multi.json \
  | jq '.result[0].hostname, .result[1].version, (.result[2].interfaces | keys)'
```

## 4. Sending configuration via eAPI

`cmds` can contain config-mode commands. EOS treats the list as a `configure` session:

```json
{
  "jsonrpc": "2.0",
  "method":  "runCmds",
  "params": {
    "version": 1,
    "cmds": [
      "configure",
      "interface Ethernet1",
      "description provisioned-via-eapi",
      "exit",
      "end",
      "write"
    ],
    "format": "json"
  },
  "id": 1
}
```

eAPI commands already run in privileged mode — no `enable` needed in the `cmds` list.

Response (on success): an empty `result` list of dicts, no error.

Verify:

```bash
curl -sk -u admin:admin https://172.20.20.3/command-api -d '{
  "jsonrpc":"2.0","method":"runCmds",
  "params":{"version":1,"cmds":["show interfaces Ethernet1"],"format":"json"},
  "id":1}' \
  | jq -r '.result[0].interfaces.Ethernet1.description'
```

⚠ **No transaction guarantees** for plain `runCmds` config push. If one command fails halfway, prior commands have already been applied. For atomic config changes, use **configure sessions**:

```json
"cmds": [
  "configure session avd-001",
  "interface Ethernet1",
  "description provisioned-via-eapi",
  "commit"
]
```

A session lets you build up a candidate config and `commit` (atomic) or `abort`. This is how AVD's deploy role applies safe changes.

## 5. Error handling

When something goes wrong, eAPI returns an `error` field instead of `result`:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": 1003,
    "message": "CLI command 4 of 4 'show frobnicate' failed: invalid command",
    "data": [
      {},
      {},
      {},
      {
        "errors": ["Invalid input"]
      }
    ]
  }
}
```

Notice `data` is a parallel array to your `cmds`: empty dicts for commands that ran successfully (commands 1–3), and an `errors` array for the failed one. So in failure mode, you can still see which commands ran.

Always check for `error` before assuming `result` exists:

```bash
curl … | jq 'if .error then .error.message else .result end'
```

## 6. Talking to eAPI from Python with `pyeapi`

`pyeapi` is Arista's official Python client. It wraps the JSON-RPC details and gives you a friendly API.

```bash
source ~/.venvs/avd/bin/activate
pip install pyeapi
```

### Hello world

```python
# scripts/hello_eapi.py
import pyeapi

conn = pyeapi.connect(
    transport="https",
    host="172.20.20.3",
    username="admin",
    password="admin",
    port=443,
)
node = pyeapi.client.Node(conn)

result = node.enable("show version")
print(result[0]["result"]["version"])
```

Run it:

```bash
python3 scripts/hello_eapi.py
# 4.32.5.1M
```

Three things `pyeapi` did for you:

- Built the JSON-RPC envelope.
- Sent the HTTPS POST.
- Verified the response and raised an exception on error.

### Multiple commands, structured access

```python
cmds = ["show hostname", "show ip bgp summary"]
responses = node.enable(cmds)

print("hostname:", responses[0]["result"]["hostname"])
print("BGP peers:")
for peer, info in responses[1]["result"]["vrfs"]["default"]["peers"].items():
    print(f"  {peer:>20}  {info['peerState']}")
```

### Configuration

```python
node.config([
    "interface Ethernet1",
    "description provisioned-via-pyeapi"
])
```

### Using a connection config file

Hardcoding hosts and credentials gets old. `pyeapi` reads `~/.eapi.conf`:

```ini
[connection:leaf1]
host=172.20.20.3
username=admin
password=admin
transport=https

[connection:leaf2]
host=172.20.20.4
username=admin
password=admin
transport=https
```

Now:

```python
import pyeapi
node = pyeapi.connect_to("leaf1")
print(node.enable("show version")[0]["result"]["version"])
```

⚠ Don't commit `~/.eapi.conf` to a public repo — it has passwords.

## 7. A useful script: fabric inventory

Pull state from every device and print a summary table:

```python
# scripts/fabric_inventory.py
import pyeapi
from rich.table import Table
from rich.console import Console

DEVICES = ["172.20.20.2", "172.20.20.3", "172.20.20.4", "172.20.20.5"]

rows = []
for ip in DEVICES:
    conn = pyeapi.connect(transport="https", host=ip,
                          username="admin", password="admin")
    node = pyeapi.client.Node(conn)
    ver = node.enable("show version")[0]["result"]
    host = node.enable("show hostname")[0]["result"]["hostname"]
    rows.append((host, ip, ver["version"], ver["modelName"]))

table = Table(title="Lab fabric")
for col in ("Host", "IP", "Version", "Model"):
    table.add_column(col)
for r in rows:
    table.add_row(*r)
Console().print(table)
```

Install `rich` first (`pip install rich`). Run and you get a pretty table — the kind of output your colleagues will actually want to read.

## 8. Streaming many commands across many devices

For large fabrics, do work in parallel:

```python
from concurrent.futures import ThreadPoolExecutor
import pyeapi

DEVICES = [f"172.20.20.{i}" for i in (2, 3, 4, 5)]

def get_version(ip):
    conn = pyeapi.connect(transport="https", host=ip,
                          username="admin", password="admin")
    node = pyeapi.client.Node(conn)
    return ip, node.enable("show version")[0]["result"]["version"]

with ThreadPoolExecutor(max_workers=20) as ex:
    for ip, ver in ex.map(get_version, DEVICES):
        print(f"{ip}: {ver}")
```

20 threads is plenty for hundreds of devices — eAPI responses are typically milliseconds. (For thousands of devices, switch to `asyncio` + `httpx` for a non-blocking client; pyeapi is sync only.)

## 9. CLI commands you'll always want via eAPI

Useful shows (all return rich JSON):

| Command | What's in the JSON |
|---|---|
| `show version` | Version, model, uptime, MAC, serial |
| `show inventory` | Hardware modules |
| `show interfaces` | Per-port counters, status, descriptions |
| `show interfaces status` | Brief link/oper/admin per port |
| `show ip route` | Routing table |
| `show ip bgp summary` | BGP peer state |
| `show vlan` | VLAN table |
| `show mac address-table` | MAC table |
| `show lldp neighbors` | LLDP topology |
| `show running-config` | Full config as text (in `output` field even with json format) |

⚠ `show running-config` is one of the few that doesn't have a true JSON form — you get the text back as a single string. For structured config data, use individual `show` commands with `"format": "json"` or use the structured config from AVD instead.

> **Tip:** Every EOS device with eAPI enabled has a built-in **Command Explorer** at `https://<switch-ip>/explorer`. It lets you type CLI commands and instantly see the JSON-RPC request/response — the fastest way to learn eAPI interactively.

## 10. Security and ops considerations

- **HTTPS, always.** Plain HTTP eAPI exists but never enable it.
- **Use a dedicated automation user**, not `admin`. EOS supports role-based AAA.
- **Rate limit / source-restrict** eAPI in production via ACLs under `management api http-commands`.
- **Audit**: every eAPI request is logged. You can review who pushed what.
- **Cert pinning** for the truly serious: stop using `-k` / `verify=False` and trust the device's cert via your internal CA.

## 11. eAPI vs Ansible's `eos_command` — which to use when

| Situation | Use |
|---|---|
| One-off script, prototyping | `curl` or `pyeapi` |
| Repeatable infra workflow against many devices | Ansible playbook with `arista.eos.eos_command` / `eos_config` |
| Full AVD-driven fabric deploy | AVD's `eos_config_deploy_eapi` role (which uses Ansible under the hood) |
| Real-time monitoring / dashboard | gNMI streaming or eAPI polling, depending on data volume |

The deeper the workflow, the more value Ansible provides (idempotency, inventory, change reporting). For quick scripts, raw eAPI is fine.

## 12. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **JSON-RPC 2.0** | The wire-protocol envelope eAPI uses. |
| **runCmds** | The (almost always only) eAPI method. |
| **Configure session** | A named candidate-config that you can commit atomically. |
| **`pyeapi`** | Arista's Python client. |
| **`.eapi.conf`** | The connection-config file pyeapi reads. |
| **`enable` vs `config`** | Two convenience methods on a pyeapi node. |

---

## 🧪 Exercise

In `labs/10-eapi/`:

1. **`curl` warm-up**: write a request that runs `show lldp neighbors` against `leaf1` and extracts just the neighbor hostnames using `jq`.

2. **Multi-command request**: combine `show hostname`, `show ip route summary`, `show ip bgp summary` in one request. Parse out the `summaryStats.uniqueRoutes` from the route summary.

3. **`pyeapi` exploration**:
   ```python
   import pyeapi
   conn = pyeapi.connect(host="172.20.20.3", username="admin",
                         password="admin", transport="https")
   node = pyeapi.client.Node(conn)
   # interactively introspect:
   r = node.enable("show interfaces status")
   print(r[0]["result"]["interfaceStatuses"]["Ethernet1"])
   ```

4. **Config push**: use pyeapi to set `description set-via-script` on `Ethernet2` of `leaf1`. Verify with another `show`. Then revert it (`no description`).

5. **Configure session safety**: write a python script that uses a session named `test-session`, adds a deliberately invalid command (`vlan 99999`), then `abort`s the session. Confirm via `show running-config | grep vlan` that nothing changed.

6. **Parallel fabric scan**: adapt §8 to also fetch `show lldp neighbors count` per device. Print a "fabric LLDP map" of who sees whom.

---

**Next:** [11 — Python Essentials for Network Engineers](11-python-essentials.md)
