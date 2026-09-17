---
name: hal
description: >
  Drive instruments through the B101 HAL from NI TestStand .NET steps:
  power supplies, DMMs, switches, muxes, matrices, oscilloscopes, programmers
  and serial ports. Use when a sequence must call HAL instead of talking to
  hardware directly. Assumes the ts-authoring skill (ts-cli) is already known;
  this skill only adds what is HAL-specific.
---

# Using HAL from TestStand

This skill assumes you already know how to author sequences with `ts-cli` (see
the `ts-authoring` skill): adapters, `TS.SData`, the call list, parameters.
Here you only learn what is specific to HAL.

## Which assembly

Reference the facade **`B101.Hal.dll`** (`$HAL` in the examples).
`B101.Hal.Core.dll` holds only interfaces and is **not** referenced as a step
assembly; `B101.Hal.Runtime` is native and is loaded by the facade.

Pick the facade that matches the installed TestStand: the `net8` one for
TestStand 2023 or newer (2025, 2026), and the `net48/` one for TestStand 2022
and older.

## Instance model

`Hal` is the root class and the only usage facade. Create **one** instance and
reuse it for the whole sequence instead of constructing it in every step:

- The first step constructs `Hal` and stores it in an object reference variable
  (`Locals.Hal` here).
- Every later step takes that instance and calls it.

Instruments come from the instance: `PowerSupply("PSU1")` returns an
`IPowerSupply`, `DMM("DMM_MAIN")` an `IDMM`, and so on. Storing the returned
instrument in its own reference variable (`Locals.Psu`) and reusing it is also
valid; the point is not to build a new `Hal` per step.

## Building a HAL step with ts-cli

Use the `.NET` module rules from `ts-authoring`. HAL specifics:

```bash
# 1. Create the HAL instance once (store it in Locals.Hal)
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
```

- Instrument step: `Calls[0]` = `Use Existing Object` from `Locals.Hal`, then
  `Calls[1]` = `PowerSupply` / `DMM` / ... Store its `Return Value` in a
  variable (`Locals.Psu`) when you want to reuse the instrument.
- Operation step (for example `SetVoltage`): `Calls[0]` = `Use Existing Object`
  from `Locals.Psu`, then `Calls[1]` = the method with its arguments.
- In an object parameter, `TypeName` is the interface
  (`B101.Hal.Core.Interfaces.IPowerSupply`), while the step `ClassName` is the
  type that declares the method.

## Interfaces and methods

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

- Never reference `B101.Hal.Core.dll` as a step assembly; use `B101.Hal.dll`.
- Create `Hal` once and reuse it; do not construct it per step.
- Module and method names are the unspaced .NET identifiers, not display names.
- Authoring only: this skill never runs sequences, deploys or touches hardware.
