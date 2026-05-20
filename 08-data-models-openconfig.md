# 08 — Data Models: OpenConfig, YANG, and Vendor Models

> **Goal of this chapter:** understand *why* data models exist, what YANG is, how OpenConfig fits in, and how Arista's native EOS YANG models compare. By the end you should be able to read a YANG snippet, find a specific config knob in OpenConfig, and recognize when AVD's data model maps to YANG-ish concepts vs when it doesn't.

This chapter is more conceptual than hands-on — but it's the foundation that makes APIs and AVD intelligible.

## 1. The problem data models solve

Before models, every vendor described a "BGP neighbor" their own way:

- Cisco IOS CLI: `neighbor 10.0.0.1 remote-as 65001`
- Juniper Junos: nested config blocks
- Arista EOS CLI: similar to IOS but not identical
- Each SNMP MIB: arbitrary OID trees
- Each show output: free-form text

To write a "configure BGP" function that worked across vendors, you needed a translator per vendor — fragile, hand-maintained, full of corner cases.

A **data model** is a vendor-agnostic, formal description of *what fields exist*, *what types they are*, *what relationships hold*. With a shared model, a tool can read or write configuration intent in one canonical form, and each device implements the same model.

> **Network-engineer analogy:** a data model is to network config what a database schema is to a database. It defines the columns, the types, the constraints — separately from any specific row of data.

## 2. YANG: the modeling language

**YANG** (Yet Another Next Generation) is the standardized *language* used to *write* data models. Defined in [RFC 7950](https://www.rfc-editor.org/rfc/rfc7950). It looks like a programming-language type declaration:

```yang
module example-bgp {
  namespace "http://example.com/bgp";
  prefix "ex-bgp";

  container bgp {
    leaf local-as {
      type uint32;
      mandatory true;
      description "Local autonomous system number";
    }
    list neighbor {
      key "neighbor-address";
      leaf neighbor-address {
        type inet:ipv4-address;
      }
      leaf peer-as {
        type uint32;
      }
      leaf description {
        type string;
      }
    }
  }
}
```

What's declared here:

- A **module** `example-bgp` (the unit of distribution).
- A **container** `bgp` (a YAML mapping / JSON object).
- A scalar **leaf** `local-as` of type `uint32`, mandatory.
- A **list** of neighbors keyed by `neighbor-address`, each with `peer-as` and `description`.

YANG is *itself* not data — it's the *schema*. You don't deploy YANG; you deploy YAML/JSON/XML data that *conforms to* the YANG model.

### Where YANG runs

| Protocol | Uses YANG models | Encoding on the wire |
|---|---|---|
| **NETCONF** | Yes | XML |
| **RESTCONF** | Yes | JSON or XML |
| **gNMI** | Yes | Protocol Buffers (JSON-ish for humans) |

So when you push BGP config to an EOS box via gNMI using the OpenConfig BGP model, the path is:
1. Your tool serializes a payload conforming to `openconfig-bgp.yang` into protobuf.
2. The switch validates the payload against its YANG implementation.
3. The switch applies the change.

## 3. OpenConfig: the vendor-neutral models

**OpenConfig** is an industry group (Google, Facebook, AT&T, Verizon, and most network vendors including Arista) that publishes a *family* of YANG models that are vendor-neutral.

The promise: "configure BGP the same way on any vendor that implements `openconfig-bgp.yang`."

Reality: implementations vary in coverage, and some features only exist in vendor-native models. But for the core operational knobs — interfaces, BGP, OSPF, LLDP, telemetry — OpenConfig is increasingly usable.

The models are organized by area:

- `openconfig-interfaces` — ports, sub-interfaces, statistics
- `openconfig-vlan` — VLANs
- `openconfig-bgp`, `openconfig-network-instance` — routing
- `openconfig-acl` — ACLs
- `openconfig-system` — hostname, NTP, DNS, AAA
- `openconfig-telemetry` — streaming telemetry config

Browse them at <https://github.com/openconfig/public/tree/master/release/models>.

## 4. Native vendor models: Arista EOS YANG

Vendors publish their *own* YANG models that cover everything OpenConfig misses (and many features that exist only in their OS). Arista's are at <https://github.com/aristanetworks/yang>.

When to use native vs OpenConfig:

| Use OpenConfig when… | Use vendor-native when… |
|---|---|
| You need multi-vendor portability | You're 100% on one vendor |
| You're only touching standard features | You need a vendor-specific feature (e.g., Arista TerminAttr, Arista-specific PIM knobs) |
| You're building a tool meant to be vendor-agnostic | You're managing a CloudVision or Junos-only fleet |

In practice many networks use **both**: OpenConfig for "platform" knobs, native for the long tail.

## 5. How models, encoding, and protocols stack

```
┌─────────────────────────────────────────────┐
│  Data model (YANG)                          │  ← openconfig-bgp.yang
├─────────────────────────────────────────────┤
│  Encoding (XML / JSON / Protobuf)           │  ← JSON
├─────────────────────────────────────────────┤
│  Protocol (NETCONF / RESTCONF / gNMI)       │  ← gNMI
├─────────────────────────────────────────────┤
│  Transport (TLS over TCP)                   │
└─────────────────────────────────────────────┘
```

You pick a YANG model (intent), pick an encoding (how it goes on the wire), pick a protocol (how the conversation is structured). [Chapter 09](09-apis-overview.md) covers the protocols in detail.

## 6. AVD's data model: where does it sit?

AVD's input data model is *its own* thing — not a YANG model, not OpenConfig, not gnmi. It's a documented set of YAML keys that AVD's `eos_designs` role understands.

So why does AVD bother having its own model?

- **Higher-level abstractions.** AVD's `tenants → vrfs → svis → tags` model lets you say "VLAN 100 on all L3 leaves" in one place. The equivalent OpenConfig payload would require explicit per-device config — there's no concept of "tag" or "tenant" in OpenConfig.
- **Opinions baked in.** AVD makes design choices (route-distinguisher format, RT scheme, BFD timers, MTU defaults). YANG describes *possible* fields; AVD's model describes *recommended* combinations.
- **Operator-friendly key names.** AVD's keys read like fabric design vocabulary, not protocol-spec vocabulary.

AVD's pipeline conceptually *renders* high-level intent down to low-level config — and the final step pushes via eAPI (today) or potentially gNMI/OpenConfig (future). In that sense AVD sits *above* OpenConfig in the abstraction stack.

## 7. Schema-driven editing in VS Code

The `redhat.vscode-yaml` extension can validate any YAML file against a JSON Schema. AVD ships such a schema (auto-generated from its data model). When you have it wired up in a VS Code workspace, you get:

- Autocomplete on keys.
- Type-aware validation on values.
- Hover tooltips with field descriptions.
- Red squiggles for unknown keys or wrong types.

In a real AVD repo, the workspace `settings.json` typically points at the AVD schema:

```json
{
  "yaml.schemas": {
    "https://www.avd.sh/en/latest/schemas/eos_designs.jsonschema.json": [
      "**/group_vars/**/*.yml",
      "**/host_vars/**/*.yml"
    ]
  }
}
```

We won't wire this up until [chapter 14](14-avd-inputs.md), but file it away: *schema = autocomplete + early error detection*.

## 8. Reading a real OpenConfig YANG snippet

From `openconfig-interfaces.yang` (abbreviated):

```yang
list interface {
  key "name";
  leaf name {
    type leafref { path "../config/name"; }
  }
  container config {
    leaf name { type string; }
    leaf type { type identityref { base ift:INTERFACE_TYPE; } }
    leaf enabled { type boolean; default true; }
    leaf mtu { type uint16; }
    leaf description { type string; }
  }
  container state {
    config false;        // operational state, read-only
    // …
  }
}
```

What this says, in English:

- "There is a list of interfaces, keyed by `name`."
- "Each interface has a `config` container (writable) and a `state` container (read-only operational data)."
- "`enabled` defaults to true; `mtu` is a 16-bit unsigned int; `type` references an identity from another module."

You can fetch the equivalent JSON from a switch that supports OpenConfig via gNMI; we cover that in chapter 09.

## 9. A common confusion, cleared up

**"Is YANG the same as YAML?"**
No. **YANG** is a schema language (describes structure). **YAML** is a data format (carries values). YANG models can be *expressed* in JSON or XML for transport, but rarely in YAML. The names are confusingly similar.

**"Do I need to write YANG modules?"**
Almost never as a consumer. You use existing models (OpenConfig or vendor-native) to *talk to* devices. Writing new models is for vendor SDK teams.

**"Can I derive a YANG model from AVD's data model?"**
There are community efforts to do exactly this, but AVD does not ship YANG modules of its inputs. The "schema" AVD ships is JSON Schema, which is enough for validation and editor support but not for NETCONF/gNMI transport.

## 10. Useful tools

| Tool | What it does |
|---|---|
| `pyang` | YANG validator and pretty-printer. `pip install pyang`, then `pyang -f tree my-model.yang`. |
| `gnmic` | Command-line gNMI client. Great for poking real switches. <https://gnmic.openconfig.net> |
| `ncclient` | Python NETCONF library. |
| `pygnmi` | Python gNMI library. |
| `yanglint` | C-based YANG validator (faster than `pyang`). |
| OpenConfig YANG Explorer | Browser-based YANG tree explorer for OpenConfig models. |

## 11. Terminology cheat sheet

| Term | Meaning |
|---|---|
| **Data model** | A formal description of structure — fields, types, relationships. |
| **YANG** | The modeling language used to express data models in network management. |
| **Module** | A unit of YANG distribution — like a file or package. |
| **Container** | A YANG grouping that becomes an object/dict at runtime. |
| **List** | A keyed collection. |
| **Leaf** | A scalar field. |
| **OpenConfig** | A vendor-neutral family of YANG models. |
| **Native model** | A vendor-specific YANG model (e.g., `arista-bgp-augments`). |
| **JSON Schema** | A different schema language, used to validate JSON/YAML data. Used by AVD. |
| **gNMI** | A gRPC-based protocol for streaming telemetry and config that uses YANG models. |
| **NETCONF** | XML-based config protocol that uses YANG models. |
| **Identity / identityref** | YANG's way of describing enumerated, extensible "kinds of thing" (interface types, address families, etc.). |

---

## 🧪 Exercise

1. **Install `pyang`** in your venv and download the OpenConfig interfaces model:
   ```bash
   pip install pyang
   mkdir ~/yang && cd ~/yang
   curl -O https://raw.githubusercontent.com/openconfig/public/master/release/models/interfaces/openconfig-interfaces.yang
   curl -O https://raw.githubusercontent.com/openconfig/public/master/release/models/types/openconfig-yang-types.yang
   pyang -f tree openconfig-interfaces.yang 2>/dev/null | head -60
   ```
   Read the tree output. Note which leaves are under `config` (writable) vs `state` (read-only).

2. **Map a familiar feature to a model.** Pick "BGP neighbor description" and find:
   - The OpenConfig path: `network-instances/network-instance/protocols/protocol/bgp/neighbors/neighbor/config/description`
   - The EOS CLI equivalent: `router bgp X / neighbor 1.1.1.1 description ...`
   - What it would look like in AVD's data model (hint: `router_bgp.neighbors[].description` in structured_config, or higher-level in `bgp_peer_groups`).

3. **Schema-driven editing**: in VS Code, open any YAML file in this repo. Now add this to your local `.vscode/settings.json` (you may need to create the file):
   ```json
   {
     "yaml.schemas": {
       "https://json-schema.org/draft/2020-12/schema": ["*.yml"]
     }
   }
   ```
   Reload VS Code. Hover keys, watch for inline hints. The schema is overly generic but you'll see the mechanism work.

4. **(Conceptual) Write the YANG.** Sketch (in pseudo-YANG) a model for a "fabric" with a list of "switches", each with a name, role (spine|leaf), and ASN. Compare your structure to how AVD models the same thing in `group_vars/FABRIC.yml`. What's different about AVD's approach (hint: AVD adds *inheritance* across groups, which YANG doesn't natively).

---

**Next:** [09 — APIs Overview: REST, eAPI, gNMI, NETCONF](09-apis-overview.md)
