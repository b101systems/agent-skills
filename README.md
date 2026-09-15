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

## Install

```bash
gh skill install b101systems/agent-skills ts-authoring
```

Update later with:

```bash
gh skill update ts-authoring
```
