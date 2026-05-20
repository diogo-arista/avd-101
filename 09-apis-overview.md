# 09 — APIs Overview: REST, eAPI, gNMI, NETCONF

> **Goal of this chapter:** map the four most common ways tools talk to devices, know when each is the right pick, and recognize the strengths and weaknesses of each. This sets up [chapter 10](10-eapi-hands-on.md) (deep dive on eAPI) and prepares you for AVD's transport choices.

## 1. What "API" means here

An **API** (Application Programming Interface) is just a contract that says "send me requests shaped like *this*, and I'll respond with results shaped like *that*." For network devices, an API replaces SSH+screen-scraping with a programmatic, structured channel.

The four protocols you'll most likely encounter on Arista (and broader networking) gear:

| Protocol | Style | Encoding | Maintained by | Status on EOS |
|---|---|---|---|---|
| **eAPI** | JSON-RPC over HTTPS | JSON | Arista | Native, full coverage |
| **REST / RESTCONF** | HTTP+verbs over JSON | JSON | IETF / vendors | Partial — CloudVision yes, EOS via OpenConfig |
| **NETCONF** | RPC over SSH | XML | IETF (RFC 6241) | Supported, less common on EOS |
| **gNMI** | gRPC | Protocol Buffers | OpenConfig group | Supported, increasingly used for telemetry+config |

Plus, for completeness, **SNMP** (read-mostly, MIB-based) and **SSH CLI scraping** (Netmiko, Napalm) — both still in heavy use, both fragile compared to the above.

## 2. The mental model: who initiates, what's pushed/pulled

```
   ┌────────────┐                               ┌─────────────┐
   │  CLIENT    │  ── request (method, args) ─▶ │   DEVICE    │
   │  (you/AVD) │  ◀── response (data, error) ──│             │
   └────────────┘                               └─────────────┘
                                                       │
                       gNMI streaming:                 │
   ┌────────────┐  ◀── subscribe response 1 ──────────┤
   │  CLIENT    │  ◀── subscribe response 2 ──────────┤
   └────────────┘  ◀── subscribe response 3 ──────────┘  (continuous)
```

Most APIs are request/response. **gNMI Subscribe** is unique in that the device streams updates to the client — that's how modern telemetry works.

## 3. eAPI — Arista's JSON-RPC over HTTPS

### What it is

The HTTP-based interface to EOS. You POST a JSON payload describing CLI commands; the switch executes them and responds with structured JSON.

### Request shape

```json
{
  "jsonrpc": "2.0",
  "method":  "runCmds",
  "params": {
    "version": 1,
    "cmds":    ["show version", "show hostname"],
    "format":  "json"
  },
  "id": 1
}
```

- `method` is always `runCmds`.
- `cmds` is a list of EOS CLI strings.
- `format: "json"` returns structured data; `"text"` returns the raw `show` output as a string.
- `id` is echoed back; useful for matching requests in async clients.

### Response shape

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [
    { /* output of show version */ },
    { /* output of show hostname */ }
  ]
}
```

`result` is always a list, one entry per command in the request.

### Strengths

- **Easy to test** — `curl` works. No client SDK needed.
- **Universal coverage** — anything you can type at the CLI, you can run over eAPI.
- **JSON output for almost every show command** — no text parsing.

### Weaknesses

- Arista-only.
- Not a streaming protocol — you poll for updates, you don't subscribe.
- Configuration push is "send me a list of CLI commands" — not declarative in the model sense.

### Who uses it
AVD's `eos_config_deploy_eapi` role, Ansible's `arista.eos` collection (modules like `eos_command`, `eos_config`), and any custom Python that uses `pyeapi`. We'll do hands-on in chapter 10.

## 4. REST and RESTCONF

### Plain REST

A general design style: HTTP verbs (`GET`, `POST`, `PUT`, `DELETE`) against URL paths that represent resources, usually returning JSON. Each vendor's REST API is its own beast — there's no single "network REST standard." CloudVision's REST API, monitoring tools, and IPAM systems (NetBox, Infoblox) are all REST.

A CloudVision REST request looks like:

```bash
curl -k -H "Authorization: Bearer ${TOKEN}" \
     https://cvp.example.com/api/resources/inventory/v1/Device/all
```

…returning a JSON stream of device objects.

### RESTCONF

A standardized layer on top of REST ([RFC 8040](https://www.rfc-editor.org/rfc/rfc8040)) that uses YANG models as the resource definitions. URL paths map to YANG paths; payloads are JSON-or-XML serializations of the model.

Example URL for an interface config:

```
GET /restconf/data/openconfig-interfaces:interfaces/interface=Ethernet1/config
```

Strengths: model-aware, vendor-neutral when paired with OpenConfig.
Weaknesses: less widely deployed than NETCONF or gNMI on Arista.

## 5. NETCONF

### What it is

An RPC protocol over SSH (RFC 6241) using XML payloads. Built specifically for config management — has the notion of *candidate* and *running* configs, atomic commits, rollback.

### Request shape (XML, abbreviated)

```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target><running/></target>
    <config>
      <interfaces xmlns="http://openconfig.net/yang/interfaces">
        <interface>
          <name>Ethernet1</name>
          <config>
            <description>uplink to spine1</description>
          </config>
        </interface>
      </interfaces>
    </config>
  </edit-config>
</rpc>
```

Verbose, but every field is unambiguous and validatable against the YANG model.

### Strengths

- Standardized, vendor-neutral (with OpenConfig or native models).
- Transactional: candidate config, validate, commit, rollback.
- Strong tooling (`ncclient` in Python, `netconf-console`, vendor CLI tools).

### Weaknesses

- XML is verbose and unfamiliar to most network engineers.
- Slower than gNMI for high-frequency operations.
- Vendor model coverage varies.

### Who uses it
Heavily on Juniper, Cisco IOS-XR, Nokia. Available on EOS but less common than eAPI or gNMI in Arista shops.

## 6. gNMI

### What it is

**gRPC Network Management Interface.** A Google-originated, gRPC-based protocol that uses YANG models. The wire format is Protocol Buffers (binary, fast); for humans, tools display payloads as JSON.

Four RPC types:

| RPC | Purpose |
|---|---|
| `Get` | Read one or more paths once |
| `Set` | Update / replace / delete paths |
| `Subscribe` | Stream updates (the killer feature) |
| `Capabilities` | List supported models/encodings |

### Example with `gnmic`

```bash
# Read an OpenConfig BGP path
gnmic -a leaf1:6030 -u admin -p admin --insecure \
      --encoding json_ietf \
      get --path "/network-instances/network-instance/protocols/protocol/bgp/global/state"

# Stream interface counters every 10 seconds
gnmic -a leaf1:6030 -u admin -p admin --insecure \
      subscribe --path "/interfaces/interface/state/counters" \
                --sample-interval 10s
```

### Strengths

- **Streaming telemetry** — the device pushes updates as state changes, instead of you polling.
- **Fast** — binary protobuf, multiplexed gRPC connections.
- **Standardized + model-aware** — pairs naturally with OpenConfig.

### Weaknesses

- gRPC tooling is heavier than `curl`.
- Subscription mode requires more architectural thinking ("where does the data go?").
- Config push via gNMI is still maturing in some vendor stacks.

### Who uses it
Modern telemetry pipelines (gnmic → Kafka → time-series DB), CloudVision's telemetry ingestion, and increasingly tools like SuzieQ. Some shops use gNMI Subscribe for *observation* and eAPI/NETCONF for *configuration*.

## 7. Choosing the right tool

| Job to do | Best fit | Why |
|---|---|---|
| Push EOS CLI config from a script | eAPI | Native, simple |
| Read structured `show interfaces` output | eAPI | Built-in JSON shows |
| Stream BGP session-state changes at scale | gNMI Subscribe | Push-based, low overhead |
| Multi-vendor inventory data collection | RESTCONF + OpenConfig, or gNMI Get | Standardized models |
| Config rollback across a fabric atomically | NETCONF (with sessions) or eAPI sessions | Transactional |
| One-off `show version` from a script | eAPI | Lowest barrier |
| Programmatic CloudVision interaction | CVP REST | The CVP-specific API |

For AVD as currently shipped: the consumer-facing deploy role is `eos_config_deploy_eapi` (or `_cvp` via CloudVision REST). gNMI/NETCONF support is more about telemetry and validation tooling adjacent to AVD.

## 8. Authentication, briefly

| Protocol | Typical auth |
|---|---|
| eAPI | HTTP Basic (`admin:password`) over HTTPS, or per-user tokens |
| REST/RESTCONF | Bearer tokens, OAuth, or HTTP Basic |
| NETCONF | SSH key or password |
| gNMI | mTLS (client certs) is the standard; username/password supported too |

**For production:** never embed credentials in YAML or scripts. Use Ansible Vault, environment variables, or a secrets manager. AVD reads device credentials from inventory vars; you store those via `ansible-vault encrypt` or pull them from an env file.

For *labs*: hardcoding `admin/admin` is fine. Just don't push such inventory to public repos.

## 9. TLS / certificates and the `-k` curse

Lab devices serve eAPI/RESTCONF/gNMI over self-signed certificates. Every tool will refuse the connection by default. The escape hatches:

- `curl -k` — skip cert verification.
- `gnmic --insecure` — same.
- Ansible: `ansible_httpapi_validate_certs: false` in inventory.

⚠ In production, **don't** use `-k`. Either trust the device's cert via your CA, or upload proper TLS certs to the switch. Skipping verification opens you to MITM.

## 10. The CLI scraping era (and why we're moving past it)

Before APIs, the pattern was: SSH to the device, run a `show` command, parse the text. Tools like Netmiko and Napalm (Python libraries) industrialized this.

Pros: works on *anything* that has a CLI.
Cons:
- Text output changes between OS versions and breaks parsers.
- No type info — everything is a string.
- Slow — interactive shell vs single HTTP call.
- No structured error semantics.

You'll still encounter Netmiko/Napalm code in older codebases. Treat it as legacy: still works, but new development should target eAPI/gNMI/NETCONF.

## 11. Postman, Insomnia, Bruno

For *exploring* APIs by hand, a GUI client beats `curl`. The big three:

- **Postman** — proprietary, very popular, has free tier.
- **Insomnia** — open core.
- **Bruno** — fully open source, stores collections as files in your repo (great for version control).

For eAPI you can build a collection of common requests, save credentials per environment, and share with your team. Highly recommended for the early days of learning a new API.

## 12. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **JSON-RPC** | A simple RPC convention layered on JSON. eAPI uses 2.0. |
| **REST** | An architectural style using HTTP verbs against resource URLs. |
| **RESTCONF** | A standardized REST layer on top of YANG models. |
| **NETCONF** | XML-based config-management protocol over SSH. |
| **gNMI** | gRPC-based, model-driven protocol; great for streaming telemetry. |
| **gRPC** | Google's RPC framework on top of HTTP/2 with protobuf encoding. |
| **Protobuf** | Binary serialization format used by gRPC. |
| **Subscribe** | gNMI mode where the server pushes updates to the client. |
| **Idempotent** | A `PUT` or `Set` that produces the same end state regardless of how many times it's run. |
| **mTLS** | Mutual TLS — both client and server present certs. |
| **Bearer token** | An auth header `Authorization: Bearer <opaque-string>`. |

---

## 🧪 Exercise

1. **eAPI vs SSH speed test**. Time both:
   ```bash
   time ssh -o StrictHostKeyChecking=no admin@172.20.20.3 'show version' > /dev/null
   time curl -sk -u admin:admin https://172.20.20.3/command-api \
        -d '{"jsonrpc":"2.0","method":"runCmds","params":{"version":1,"cmds":["show version"],"format":"json"},"id":1}' \
        > /dev/null
   ```
   eAPI should win by a noticeable margin even on a lab.

2. **gNMI sanity check** (optional — requires installing `gnmic`):
   ```bash
   sudo bash -c "$(curl -sL https://get-gnmic.openconfig.net)"
   gnmic -a 172.20.20.3:6030 -u admin -p admin --insecure capabilities | head -40
   ```
   You'll see the YANG models the device supports. (cEOS enables gNMI by default in recent versions; you may need to add `management api gnmi / transport grpc default / no shutdown` in EOS config first.)

3. **Read up on three APIs.** Spend 10 minutes browsing each docs site:
   - eAPI: <https://eos.arista.com/eos-4-32-0f/eos-api-eapi/>
   - gNMI: <https://github.com/openconfig/reference/blob/master/rpc/gnmi/gnmi-specification.md>
   - NETCONF on EOS: <https://www.arista.com/en/um-eos/eos-section-44-2-netconf>

4. **Postman/Bruno experiment**: install Bruno (`brew install --cask bruno` on Mac), create a collection "EOS Lab", add a single request that does `show version` via eAPI, parameterize credentials. Commit the collection folder to your repo (Bruno files are plain JSON/YAML).

5. **Write down**: when would you reach for each protocol? Sketch a decision tree in your notes for "I need to do X — which API?".

---

**Next:** [10 — Arista eAPI Hands-On](10-eapi-hands-on.md)
