---
name: hal
description: >
  Drive instruments through the B101 HAL from NI TestStand .NET steps: power
  supplies, DMMs, switches, muxes, matrices, oscilloscopes, programmers and
  serial ports. Use when a sequence must call HAL instead of talking to
  hardware directly. Assumes the ts-authoring skill (ts-cli) is already known;
  this skill only adds what is HAL-specific.
---

# Using HAL from TestStand

This skill assumes you already know how to author sequences with `ts-cli` (see
the `ts-authoring` skill): adapters, `TS.SData`, the call list, parameters.
Here you only learn what is specific to HAL.

## The only entry point

Use the facade assembly **`B101.Hal.dll`** (`$HAL` in the examples), and nothing
else. The ATE Framework installer leaves it at the framework root:

```text
C:\Program Files\B101\AteFramework\B101.Hal.dll        (64-bit station)
C:\Program Files (x86)\B101\AteFramework\B101.Hal.dll  (32-bit station)
```

Point `TS.SData.AssemblyPath` at that file. A bare `B101.Hal.dll` also resolves
when TestStand finds it on its search path. The native `B101.Hal.Runtime.dll`
sits next to the facade in the same folder; the facade cannot start without it,
so never point at a copy that is missing it.

- Do **not** reference `B101.Hal.Core.dll` as a step assembly: it only holds
  interfaces and is an implementation detail.
- Do **not** reference, name or import the `B101.Hal.Wrappers` types.
- The only usable API is the **`Hal` class and its instance**: create it with
  `new Hal()` and use what that instance exposes (its interface
  implementations and their methods).

Pick the facade that matches the installed TestStand: the `net8` one for
TestStand 2023 or newer (2025, 2026), and the `net48/` one for TestStand 2022
and older.

## Sequence shape

Create the `Hal` instance **once** and reuse it for the whole sequence.

- First step: `Calls[0]` is the constructor `B101.Hal.Hal`; store the result in
  an object reference variable (`Locals.Hal` here).
- Every later step: `Calls[0]` is `Use Existing Object` bound to `Locals.Hal`,
  then `Calls[1]` is the method of the interface implementation you need.

```bash
# The instance variable is created once.
ts-cli add-prop --file "$SEQ" --sequence MainSequence \
  --path Hal --type reference

# First step: create the instance and keep it.
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path TS.SData.AssemblyPath --text "$HAL"
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path TS.SData.ClassName --text "B101.Hal.Hal"
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path TS.SData.Calls[0].MemberName --text "Hal"
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path TS.SData.Calls[0].MemberType --number 4
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path "TS.SData.Calls[0].Params[0].Name" --text "Return Value"
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path "TS.SData.Calls[0].Params[0].ArgVal" --text "Locals.Hal"
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path "TS.SData.Calls[0].Params[0].TypeName" --text "B101.Hal.Hal"

# Later step: use the existing instance.
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path TS.SData.AssemblyPath --text "$HAL"
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path TS.SData.Calls[0].MemberName --text "Use Existing Object"
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path TS.SData.Calls[0].MemberType --number 6
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path "TS.SData.Calls[0].Params[0].Name" --text "Existing Object"
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path "TS.SData.Calls[0].Params[0].ArgVal" --text "Locals.Hal"
ts-cli set-prop --file "$SEQ" --step-id "$ID" \
  --path "TS.SData.Calls[0].Params[0].TypeName" --text "B101.Hal.Hal"
```

Never store an instrument in a variable: keep only the `Hal` instance and reach
the instrument from it inside each operation step.

## Reaching an instrument operation

An instrument cannot be instantiated or stored on its own. The only shape is
one step per operation that, from the `Hal` instance, calls the instrument
method and immediately the action on what it returns:

- `Calls[0]` = `Use Existing Object` on `Locals.Hal`.
- `Calls[1]` = the instrument method that returns the implementation
  (`PowerSupply("PSU1")`).
- `Calls[2]` = the action on that implementation (`SetVoltage(...)`).

So `SetVoltage` is never called on a stored power supply: it always follows its
`PowerSupply(...)` call inside the same step.

The **module class** (`TS.SData.ClassName`) is always **`B101.Hal.Hal`**, the
root of everything. Every call also carries its own `ClassName`: `B101.Hal.Hal`
on the calls that go through the HAL instance, and the **instrument interface**
(`B101.Hal.Core.Interfaces.IPowerSupply`, ...) on the call that runs the
action. The object a call receives or returns carries that type in `TypeName`.

## Interfaces exposed by the instance

- `IInstrument` (SCPI base): `Connect`, `Disconnect`, `Write`, `Query`.
- `IPowerSupply`: `SetVoltage`, `SetCurrentLimit`, `SetOVP`, `SetOCP`, `Enable`,
  `Disable`, `IsOutputEnabled`, `MeasureVoltage`, `MeasureCurrent`, `Reset`.
- `IDMM`: `ConfigureFunction`, `Measure`, `AutoZero`, `SetNplc`.
- `ISwitch`: `Close`, `Open`, `OpenAll`, `IsClosed`.
- `IMux` (extends `ISwitch`): `ConfigureChannel`, `AutoZero`.
- `IMatrix`: `Connect`, `Disconnect`, `ConnectMultiple`, `DisconnectAll`,
  `IsConnected`.
- `IOscilloscope`: `ConfigureChannel`, `SetTimebase`, `Run`, `Single`,
  `FetchWaveform`, `Measure`.
- `IProgrammer`: `Connect`, `Flash`, `Verify`, `Erase`, `Reset`.
- `ISerialComm`: `Configure`, `Open`, `Read`, `Write`, `SendReceive`.

Method arguments become step parameters (see `ts-authoring`). Enums such as
`DmmFunction` are passed as their numeric or string value.

## Rules

- Only `B101.Hal.dll` and the `Hal` instance; never `B101.Hal.Core`, never
  wrappers.
- The module class is always `B101.Hal.Hal`; instrument calls use the instrument
  interface as their call class.
- Create `Hal` once and reuse the instance; do not construct it per step.
- Never instantiate or store an instrument: each operation step calls the
  instrument method and then its action, in that order.
- Names are the unspaced .NET identifiers, not display names.
- Authoring only: this skill never runs sequences, deploys or touches hardware.
