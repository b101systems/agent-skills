---
name: ts-authoring
description: >
  Create or edit NI TestStand .seq files with ts-cli: sequences, steps, limits,
  locals/globals, arrays and batches. Use when asked to scaffold or edit a
  sequence from a spec, limits table or pinout. Does not run sequences, deploy
  or touch hardware.
---

# Authoring TestStand sequences with ts-cli

## Scope

You can **create and edit** `.seq` files. You cannot run them, deploy them,
or touch hardware. A good result is a file an engineer can open in TestStand
and finish by hand: correct sequences, steps, limits and values.

## Prerequisites

1. The B101 ATE Framework services are running with a valid license.
2. The pc has NI TestStand installed with a development license.

The tool is `ts-cli`. Confirm it is live with `ts-cli status` (`ready: true`); if
it is not ready, stop and report.

## The CLI

`ts-cli --help` and `ts-cli <command> --help` are the authority for flags and
defaults. Do not probe mutating commands with guessed arguments on real files.

Commands print JSON; errors are JSON on stderr with exit 1. Read ids such as
`unique_step_id` and `step_ids` from responses instead of inventing them.

## Reconnaissance

Before editing an existing `.seq`, narrow step by step; each call returns only
what you ask for.

```bash
ts-cli inspect --file "$SEQ"                        # sequences + globals
ts-cli inspect --file "$SEQ" --sequence PowerRails  # locals + steps (ids)
ts-cli inspect --file "$SEQ" --step-id <id>         # the step's properties
```

`--sequences` and `--globals` narrow the first level. `unique_step_id`s come
from the second. To verify a change, filter with
`inspect --values --match <text>`.

## The API

| Operation | What it does |
|---|---|
| `status` | Engine/pipe readiness |
| `create` | New `.seq` containing a default `MainSequence` |
| `add-sequence` | Add another named sequence to the file |
| `add-prop` | Create a container, array or scalar property |
| `set-prop` | Change one property of a step, locals or FileGlobals |
| `delete-prop` | Delete a local or file global property |
| `insert-step` | Add one step, optionally with properties |
| `clone-step` | Copy an existing step, optionally renaming it |
| `move-step` | Reorder a step, optionally into another group |
| `delete-step` | Remove a step by its unique step id |
| `apply-batch` | Apply many operations in one write |
| `inspect` | Read the file: sequences/globals, one sequence or one step |

Step types use the unspaced TestStand names: `NumericLimitTest`,
`StringValueTest`, `PassFailTest`, `Action`, `SequenceCall`, `Statement`,
`Goto`, `MessagePopup`, `NI_Wait`, `NI_MultipleNumericLimitTest`.

Step groups: `setup`, `main` (default), `cleanup`. `--index` is 0-based;
`-1` appends.

Property paths inside a step are **relative to the step**: `Limits.Low`,
`Limits.High`, `Comp`, `TS.PassAction` is not valid. Start with
`inspect --step-id <id>` on a step of the target type to discover the exact
paths; the fresh `NumericLimitTest` exposes `TS, Result, Limits, Comp,
CompExpr, UseCompExpr, InBuf, DataSource`.

## Arrays and containers

Give exactly one target: `--step-id` (a step; path relative to the step),
`--sequence` (a sequence's locals; path relative to `Locals`) or `--globals`
(the file's `FileGlobals`).

```bash
ts-cli add-prop --file "$SEQ" --sequence MainSequence \
  --path RailNames --type string --array --length 4
ts-cli set-prop --file "$SEQ" --sequence MainSequence \
  --path "RailNames[0]" --text VCC_5V0
ts-cli add-prop --file "$SEQ" --globals --path Timeout --type number
ts-cli set-prop --file "$SEQ" --globals --path Timeout --number 30
```

`--type` is `number`, `string`, `boolean` or `container`. Array elements are
addressed as `Name[i]`; writing an index beyond the current length grows the
array automatically. Existing step arrays (e.g. `NumericArray` on
`NI_MultipleNumericLimitTest`) are written the same way.

`delete-prop` removes a local or file global (not a step property, and not a
single array element). Use the same `--sequence`/`--globals` targets; deleting
a container removes its children too. Anything a step expression references is
not rechecked, so review the file after deleting.

## Reordering steps

`move-step` puts a step at a final 0-based position (`--index`, `-1` appends).
By default the step stays in its own sequence and group; pass `--sequence` or
`--group` to move it elsewhere. Use `inspect --sequence` to see each group's
steps in order and pick the index. The `unique_step_id` does not change when a
step moves, so you can keep using it afterwards.

## Batch

For anything bigger than a couple of changes, build a batch and apply it in a
single write: it is atomic (if any operation fails, nothing is saved). For a
single change, use the matching command.

Operations (JSON `op`): `add_sequence`, `add_property`, `set_property`,
`delete_property`, `insert_step`, `clone_step`, `move_step`, `delete_step`.
Give each inserted/cloned step a `key`; later operations reference that step by
`key` (or by an existing unique step id). `add_property` takes `path`, `type`,
optional `array`/`length`, and targets a step `key`, a `sequence` (its locals)
or `globals: true`. `delete_property` takes `path` and the same
`sequence`/`globals` targets.

```json
{
  "operations": [
    {"op": "add_sequence", "name": "PowerRails"},
    {"op": "insert_step", "key": "vcc", "sequence": "PowerRails",
     "step_type": "NumericLimitTest", "name": "VCC_5V0",
     "set": {"Limits.Low": 4.75, "Limits.High": 5.25}},
    {"op": "clone_step", "key": "vcc3", "source": "vcc",
     "sequence": "PowerRails", "name": "VCC_3V3"},
    {"op": "set_property", "step": "vcc3", "path": "Limits.Low",
     "value": 3.135}
  ]
}
```

```bash
ts-cli apply-batch --file "$SEQ" --batch batch.json
```

The response maps each `key` to its new `unique_step_id`, so you can keep
editing the same steps afterwards.

## Workflow: test spec -> sequence

1. **Read the spec.** Extract each measurable: name, type, low/high or
   expected string, units, test condition. Map each measurable to a step.
2. **Create the file.**
   ```bash
   ts-cli create --file C:/Seq/UUT_RevA.seq --overwrite
   ```
3. **Add steps in order**, one measurement per step, limits inline:
   ```bash
   ts-cli insert-step --file C:/Seq/UUT_RevA.seq \
     --step-type NumericLimitTest --name "VCC_5V0" \
     --set Limits.Low=4.75 --set Limits.High=5.25
   ```
   Name steps after the signal in the spec so the sequence reads like the
   limits table.
4. **Fix individual properties** when a value comes later:
   ```bash
   ts-cli set-prop --file C:/Seq/UUT_RevA.seq \
     --step-id <id> --path Limits.High --number 5.25
   ```
5. **Verify** with `inspect --values --match <signal>` and check every limit.
   `insert-step` returns the `unique_step_id`; use it for `set-prop`.
6. **Report** the produced file path and the list of steps created. Do not
   claim the sequence is production-ready.

## Recipes

Limit sweep — one step per row of a limits table:

```bash
for row in "VCC_5V0 4.75 5.25" "VIO_3V3 3.135 3.465"; do
  set -- $row
  ts-cli insert-step --file "$SEQ" \
    --step-type NumericLimitTest --name "$1" \
    --set "Limits.Low=$2" --set "Limits.High=$3"
done
```

Group steps under their own sequence (one per test phase) by adding a
sequence first, then targeting it with `--sequence`:

```bash
ts-cli add-sequence --file "$SEQ" --name PowerRails
ts-cli insert-step --file "$SEQ" \
  --sequence PowerRails --step-type NumericLimitTest --name VCC_5V0 \
  --set Limits.Low=4.75 --set Limits.High=5.25
```

Clone a configured step to repeat it with different limits. `clone-step`
copies the whole step (all properties); set the ones that change on the
returned `unique_step_id`:

```bash
ts-cli clone-step --file "$SEQ" --step-id "$SRC" --name VCC_3V3
ts-cli set-prop --file "$SEQ" \
  --step-id "$NEW" --path Limits.Low --number 3.135
```

## Rules

- One measurement per step; never invent signals not present in the spec.
- If a property path is rejected, `inspect --step-id` an existing step of that
  type and use the real path. Do not guess twice.
- `inspect` formats numbers with a decimal comma (`1,5`). Normalize when
  comparing to the spec.
- Do not add `Setup`/`Cleanup` steps, sequence calls or custom data types
  unless the spec asks for them.
- Every mutating command saves immediately; there is no undo. Back up an
  existing sequence before editing it.
- If the file is open in TestStand, never save from the editor; reload from
  disk so the agent's changes are not overwritten.

## Troubleshooting

- `TestStand authoring is disabled.` — this deployment has authoring turned
  off; sequences cannot be edited here.
- Missing or expired license on a fresh install: `ts-cli license-ensure`.

## Known gaps (v0)

- Only normal sequences; no callback or model-override sequences.
- No custom data types (NamedType) and no user-defined enums; standard steps
  expose enum-like values (`Comp`, `Limits.ThresholdType`) as strings.
- Property deletion covers locals and file globals, not step properties.
- No sequence reordering and no moves between files.
- No execution.
