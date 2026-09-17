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
else:

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

Do not keep one variable per instrument: keep the `Hal` instance and take the
instrument from it where the step needs it.

## Reaching an instrument operation

There are two normal shapes. Use the first when several steps work on the same
instrument; use the second (the common one) when a step needs a single
operation.

**Keep the instrument.** One step stores the instance and later steps reuse it:

- Store step: `Calls[0]` = `Use Existing Object` on `Locals.Hal`, then
  `Calls[1]` = `PowerSupply("PSU1")` with its `Return Value` written to a
  variable (`Locals.Psu`).
- Operation step: `Calls[0]` = `Use Existing Object` on `Locals.Psu`, then
  `Calls[1]` = the method (`SetVoltage(...)`).

**Two-in-one.** From the HAL instance, call the factory and then the method on
the implementation it returns, in the same step's call list:

- `Calls[0]` = `Use Existing Object` on `Locals.Hal`.
- `Calls[1]` = the factory (`PowerSupply("PSU1")`).
- `Calls[2]` = the method on that implementation (`SetVoltage(...)`).

In both shapes each call has its own `ClassName`: the HAL facade class on the
calls that go through `Hal`, and the instrument interface/implementation on the
calls that go through the instrument. Pick them from TestStand's `.NET` browser
rather than inventing names.

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
- Create `Hal` once and reuse the instance; do not construct it per step.
- Names are the unspaced .NET identifiers, not display names.
- Authoring only: this skill never runs sequences, deploys or touches hardware.
