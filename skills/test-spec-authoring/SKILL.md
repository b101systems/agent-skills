---
name: test-spec-authoring
description: >
  Turn a schematic or a bill of materials into a test spec using the ATE MCP:
  component test methods, required instruments, DUT connections, measurable
  parameters and station pinout. Use when given a schematic or asked to create
  a test spec for a component or an electronic assembly. Read-only: it does not
  create TestStand sequences, edit limits or run tests. Pairs with the
  ts-authoring and hal skills, which take the spec into a sequence.
---

# Authoring test specs with the ATE MCP

This skill produces the **test spec**. It does not build the sequence: that is
`ts-authoring` (`ts-cli`), and the instrument calls inside the steps are the
`hal` skill. Here you only learn how to turn a schematic into a spec an
engineer can review.

## Scope

You can **read** the ATE knowledge base and write a **test spec** from a
schematic. You cannot change anything, cannot run tests and cannot create
TestStand sequences.

A good result: the DUT, the methods that apply, the instruments required, the
connections and the measures — with no invented data and no limits.

## Prerequisites

1. The B101 ATE Framework services are running with a valid license.
2. The `ate-mcp` server is configured in your client and lists its tools.
3. The Bridge exposes a read-only surface; every call below is a query.

If the tools are not listed, stop and report; do not fall back to guessing.

## The MCP surface

Resources (read directly):

| Resource | What it is |
|---|---|
| `ate://station/pinout` | Physical pinout of the station: signals, connectors, pins |
| `ate://parts/{id}` | A component or assembly: family, MPN, pins |
| `ate://test-methods/{id}` | A full method: roles, connections, steps, parameters |
| `ate://products/{id}` | A product and the methods planned for it |

Tools (call these):

| Tool | What it returns |
|---|---|
| `find_part` | Parts matching `mpn`, `manufacturer` or `family` |
| `get_part` | One part plus its pins |
| `resolve_methods` | Methods that apply to a part or a family |
| `get_method` | Roles, connections, steps and parameters of a method |
| `get_step_rule` | How an action maps to a TestStand step type |
| `get_station_pinout` | Station signals with connector and pin |
| `get_limits` | Limits that already exist for a product/variant/parameter |

Tools never mutate. Names and ids you did not get from a tool response do not
exist; never invent them.

## Workflow: schematic -> test spec

1. **Read the schematic.** List the parts with their reference designators,
   values and packages. Identify the DUT: the assembly being tested or the
   component under review.
2. **Resolve the DUT in the catalog.** `find_part` by MPN, then `get_part`.
   If it is not there, report the MPN and refdes and stop; do not invent a part.
3. **Find the methods.** `resolve_methods` for the part, then for the family.
   Prefer `approved` methods; if only a `draft` exists, say so and ask.
4. **Read the method.** `get_method` gives the instrument roles, the DUT
   connections, the steps and the parameter names with units.
5. **Check the station.** `get_station_pinout` to confirm each connection can be
   made on this station. Report any role or pin the station cannot provide.
6. **Write the spec** in the format below, using only the methods, pins and
   parameters the tools returned. Leave limits out.
7. **Report**: DUT and revision, methods used (with ids), anything missing or
   mismatched, and what an engineer still has to decide.

## Test spec format

    # Test spec — <DUT> rev <rev>

    ## DUT
    - Part: <manufacturer> <mpn> (id <partId>)
    - Family: <family>
    - Methods: <method names and ids>

    ## Instruments
    | Alias | Role | Requirements |
    |---|---|---|
    | VIN | psu | <from the method> |
    | VOUT | dmm | <from the method> |

    ## Connections
    | DUT pin | Alias | Kind |
    |---|---|---|
    | VIN | VIN | source |
    | GND | GND | ground |
    | OUT | VOUT | measure |

    ## Measures
    | # | Action | Target | Parameter | Condition | Unit |
    |---|---|---|---|---|---|
    | 1 | set | VIN | — | 5 V | V |
    | 2 | delay | — | — | 50 ms | — |
    | 3 | measure | VOUT | Vout | Vin=5 V, Iload=100 mA | V |

    ## Limits
    Resolved from the variant store at runtime. Do not fill values here.

## Rules

- Never write a limit, a nominal value or a pass/fail threshold. Limits live in
  the variant store; the spec only names the parameter.
- Never invent a measurement, a pin, an instrument or a method. If the catalog
  or the station does not have it, report the gap.
- Use the DUT pin names exactly as the catalog and the schematic use them.
- One measurable per row; keep the method's step order.
- When the schematic and the catalog disagree, the schematic defines the DUT and
  the catalog defines the method; report the mismatch instead of choosing
  silently.
- Read-only: never claim to have changed anything in the framework.

## Hand-off

The spec is the input for the next two skills:

- `ts-authoring` — creates the `.seq` with `ts-cli`, one measure per step, and
  the limits linked to the variant store.
- `hal` — builds the instrument calls inside the steps, from the method's roles
  and connections.

`get_step_rule` names the TestStand step type for each action; use it in the
hand-off, but do not build the sequence yourself.

## Troubleshooting

- No MCP tools listed — the `ate-mcp` server is not configured; stop.
- `part not found` — report MPN and refdes; do not guess a match.
- No method for the family — report it; the library has a gap.
- Only a draft method — ask before using it.
- A role the station cannot map — report it; the binding is an engineer decision.

## Known gaps (v0)

- Read-only: no limits, no variant edits, no catalog edits.
- No TestStand sequence creation; that is `ts-authoring`.
- No internal netlist of an assembly; only the catalog pins.
- No multi-DUT or panel test flows.
