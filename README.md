# B101 Agent Skills

Public agent skills for B101 Systems products. Each skill is a `SKILL.md`
following the [Agent Skills](https://agentskills.io) layout, so tools such as
`gh skill` can discover, install and update it.

> This repository is generated. Do not edit files here by hand: the source of
> truth lives in the private `b101systems/ate-framework-core` repository and is
> synced by CI.

## Skills

### `ts-authoring`

Create and edit NI TestStand `.seq` files with `ts-cli`: sequences, steps,
limits, locals/globals, arrays and batches. It does not run sequences, deploy
or touch hardware.

It requires a licensed
[B101 ATE Framework](https://github.com/b101systems/ate-framework-releases/releases/latest)
install and
[NI TestStand](https://www.ni.com/es/support/downloads/software-products/download.teststand.html)
2022 or newer with a development license; the skill drives the `ts-cli`
command that ships with the framework.

### `hal`

Drive instruments through the B101 HAL from NI TestStand `.NET` steps: power
supplies, DMMs, switches, muxes, matrices, oscilloscopes, programmers and
serial ports. It assumes `ts-authoring` is already known and adds only what is
HAL-specific: which facade assembly to reference and how to build the steps.

### `test-spec-authoring`

Turn a schematic or a bill of materials into a test spec using the ATE MCP:
component test methods, required instruments, DUT connections, measurable
parameters and station pinout. It is read-only — it does not create TestStand
sequences, edit limits or run tests — and hands the spec off to `ts-authoring`
and `hal`.

## Install

```bash
gh skill install b101systems/agent-skills ts-authoring
gh skill install b101systems/agent-skills hal
gh skill install b101systems/agent-skills test-spec-authoring
```

Update later with:

```bash
gh skill update ts-authoring
gh skill update hal
gh skill update test-spec-authoring
```
