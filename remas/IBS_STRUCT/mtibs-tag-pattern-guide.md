# MTIBS Tag Pattern Guide

This guide describes the patterns found in MTIBS telegram definitions for IBS shore data. It is based on the eight vessel GenSheet folders listed in `README.md`.

## What MTIBS Sends

MTIBS is an MT proprietary telegram used to send IBS data to shore. The `List` tab in `Gmr100.xls` or `Gmr100PrilogSetup.xls` defines how many MTIBS telegrams are sent and which tags are included in each telegram.

In the GenSheet `List` tab, rows are labeled `MTIBS1`, `MTIBS2`, and so on. That trailing digit is the ListDef row/sequence identifier used to define and order telegrams. It is not part of the MTIBS telegram structure itself.

The telegram structure identifier is `$MTIBS` without the ListDef digit. A correct structure sample looks like:

```text
$MTIBS ,404_10004_TT ,404_10005_TT ,404_10005_XA ,404_10006_TT ,404_10006_XA ,404_10006_XC ,404_10007_IT_ ,404_10007_IT_A ,404_10007_IT_B
```

On the wire, received sentences may still appear as `$MTIBS1`, `$MTIBS2`, and so on so the receiver can tell telegram instances apart. Map those sentences to the corresponding List row by sequence number, then decode the payload fields against the `$MTIBS` tag order for that row.

A typical List-tab definition row looks like:

```text
MTIBS1, 1, T,B=900_00001_XA, T,R=801_00001, T,R=801_00001_LT1, ...
```

The first column is the ListDef telegram label (`MTIBS` + sequence digit), the second is the telegram number, and the remaining cells are comma-separated tag definitions for the `$MTIBS` payload layout.

## List Token Format

| Token part | Meaning |
|---|---|
| `T` | Data point/tag entry. |
| `B` | Boolean value. Interpret as `0/1`, commonly closed/open, off/on, normal/alarm, or false/true depending on the tag description. |
| `R` | Real value. Interpret as a numeric analog/derived value. Units must be resolved from the Hxxx workbook or project context. |
| `<tag>` | The tag name to cross-reference in Hxxx sheets. |

Examples:

| Token | Interpretation |
|---|---|
| `T,B=FUEL` | Boolean tag named `FUEL`. The description sheet must define whether `1` means open, running, alarm, etc. |
| `T,R=601_10001_LOAD` | Real value for the `601_10001` equipment/object with suffix `LOAD`. |
| `T,B=326_00051_ZS1` | Boolean status/switch point for equipment/object `326_00051`. |

## Tag Name Families

The List tabs contain three broad tag families.

| Family | Shape | Examples | Notes |
|---|---|---|---|
| Structured numeric tags | `<system>_<equipment>_<suffix>` or `<system>_<equipment>` | `326_00051_ZS1`, `351_10001_LT1`, `601_10001_FORATE` | Most common for IO, tanks, motors, engines, pumps, valves, alarms, and analog values. |
| Named project tags | Words plus numbers | `Ballast1`, `DryBulk10`, `DENS_FO_N1` | Usually resolved in object/helper sheets such as `BVAL`, `DATREAL`, `DATBOOL`, or project-specific sheets. |
| Object/property tags | Dot-separated object fields | `EMSDG1.Pow`, `EMSDG1.Status.CalcRunning`, `EMS_Main.Speed` | Usually derived EMS/energy-monitoring values rather than direct IO points. |

## Structured Numeric Tags

Structured tags usually split like this:

```text
326_00051_ZS1
|   |     |
|   |     suffix / signal role
|   equipment or object number
system or process family code
```

### System / Process Code

The first three digits group tags by vessel system or application area. The code is not enough by itself; always verify in the Hxxx workbook because meanings can vary by vessel and project.

Observed examples:

| Code | Observed descriptions / context |
|---|---|
| `326` | Dry bulk compressors, dry bulk tanks, dry bulk cooling pumps. |
| `351` | Fuel oil tanks, fuel oil transfer, fuel oil day tanks. |
| `352` | Dirty oil / oil transfer examples. |
| `357`, `358`, `359` | Tank/level and utility values in several vessels. |
| `404`, `626` | Engine / machinery values in 280/312 class examples. |
| `601` | Main engine/thruster/engine fuel and status values in several examples. |
| `610` | Generators. |
| `803` | Bilge/cargo/bilge alarm examples. |
| `867`, `871`, `873` | Electrical/UPS/power-related alarms and status examples. |
| `900` | Common/system alarms or generic system points. |

Treat this as a lookup hint, not a contract. The authoritative text is the matching `Descr` in the Hxxx workbook.

### Equipment / Object Number

The middle segment is usually a five-character equipment/object number, for example `10001`, `20001`, `00051`, or `30002`.

Common patterns:

| Pattern | Typical meaning |
|---|---|
| `0xxxx` | Common, centerline, shared, or non-side-specific object. |
| `1xxxx` | Port / PS / No. 1 side or first equipment group. |
| `2xxxx` | Starboard / SB / No. 2 side or second equipment group. |
| `3xxxx` and higher | Additional tanks, trains, switchboards, cargo/dry-bulk groups, or project-specific groupings. |
| last two or three digits | Equipment sequence within the system/group. |

Examples from Hxxx sheets:

| Base tag | Description evidence |
|---|---|
| `326_00051` | Dry bulk air compressor 1 / DB air comp. 1. |
| `351_10001` | Ship fuel / fuel oil 1-P tank level. |
| `601_10001` | Main engine / PME / thruster-related values depending on vessel. |

### Suffix / Signal Role

The suffix describes the specific point for a base object. It often follows ISA-style instrumentation letters, but project-specific suffixes are common.

| Suffix | Observed meaning / use | Typical type in MTIBS |
|---|---|---|
| `_PT` | Pressure transmitter / pressure analog value. | `T,R` |
| `_TT` | Temperature transmitter / temperature analog value. | `T,R` |
| `_LT`, `_LT1`, `_LT2` | Level transmitter / tank level. `TK` rows often reference `LT1`/`LT2` columns. | `T,R` |
| `_JT` | Power/load/current-derived machinery value in engine/electrical contexts. | `T,R` |
| `_IT`, `_ST`, `_SI` | Current/speed/status analog values depending on sheet and description. | `T,R` |
| `_FORATE` | Fuel or flow consumption rate. | `T,R` |
| `_LOAD`, `_PCT_LOAD` | Load or percent load. | `T,R` |
| `_HR`, `_HOUR`, `_RH` | Hourmeter or running-hours value. | `T,R` |
| `_XA` | Alarm/fail indication. Examples include overload, VFD fault, high tank alarm. | `T,B` |
| `_XAX` | Alarm variant / extended alarm point. Verify description. | `T,B` |
| `_XC`, `_XC_`, `_XCB`, `_XC2`, `_XC3` | Command/control/status variant. Verify source column and description. | Usually `T,B` |
| `_XI`, `_XI_`, `_XI1` | Indication/status value. Type can vary by source. | Usually `T,B` or `T,R` depending on List token. |
| `_XY1`, `_XY2` | Output command points, often start/stop/open/close/reset commands. | `T,B` |
| `_ZS1` | Status/switch point; often running, closed, or position status. | `T,B` |
| `_ZSH`, `_ZSL`, `_ZZH` | High/low/limit switch status. | `T,B` |
| `_HS1` | Hand/local switch or local status. | `T,B` |
| `_COMFAIL`, `_NO_CONN` | Communication failure / no connection indicator. | `T,B` or `T,R` as listed |
| `_SD1`, `_SD2`, `_SD3` | Shutdown/status bits, project-specific. | `T,B` |

The List token type (`B` or `R`) is authoritative for the value format sent in MTIBS, even when a suffix is reused in different contexts.

## How To Resolve a Tag

Use this workflow for dashboard mapping or decoding.

1. Open `Gmr100.xls` or `Gmr100PrilogSetup.xls` and find the `List` tab.
2. Collect all `T,B=<tag>` and `T,R=<tag>` entries from MTIBS rows.
3. Search the Hxxx detail workbooks for exact `TagName = <tag>`.
4. If exact search fails, strip the final suffix and search the base tag, for example search `326_00051` for `326_00051_ZS1`.
5. Use the row's `TaskName`, `Descr`, and source/reference columns to interpret the point.
6. Use the List token type for the transmitted value format: `B` boolean or `R` real.

## Useful Hxxx Sheets

| Sheet family | What it usually contains | Useful columns |
|---|---|---|
| `AIS`, `AIS02` | Analog input or analog source values. | `TaskName`, `TagName`, `Descr`, `PhysRange`, alarm limits. |
| `DIS`, `DIS02` | Digital input/status/alarm points. | `TagName`, `Descr`, `NormPos`, communication address/bit. |
| `DOS` | Digital output/command points. | `TagName`, `Descr`, command source. |
| `AOS` | Analog outputs. | `TagName`, `Descr`, range/source columns. |
| `BVAL` | Valve objects. | Base `TagName`, `Descr`, `Open`, `Close`, `Opened`, `Closed`, `Local`. |
| `TK` | Tank objects. | Base `TagName`, `Descr`, `LT1`, `LT2`, density and alarm columns. |
| `MOT` | Motor/pump/compressor objects. | Base `TagName`, `Descr`, `Start`, `Stop`, `Running`, `Fail`, `Local`. |
| `RunningHours` | Derived running-hour values. | `TagName`, `Descr`, `Input1`, `Value`. |
| `DATREAL`, `DATBOOL`, `DATINT` | Constants or derived/shared data values. | `TagName`, `Descr`, `Value`. |
| `EMSMain`, `EMSDG`, `EMSIntegrator` | Energy monitoring derived objects. | Object `TagName`, `Descr`, input/output property columns. |
| `MBTCP*`, `MBRead*`, `BoolProvider32` | Communication mapping/source providers. | Modbus/register/bit mapping columns. |

## Evidence Examples

These examples were cross-referenced from the List tags into Hxxx detail sheets.

| List tag | Hxxx evidence | Meaning inferred |
|---|---|---|
| `326_00051_HS1` | `DIS02` row: `Descr=DB PORT AIR COMP. 1 LOCAL STAT`; `MOT` base row references `Local=gv326_00051_HS1.IO`. | Boolean local/hand status for dry bulk air compressor 1. |
| `326_00051_ZS1` | `DIS02` row: `Descr=DB PORT AIR COMP. 1 RUN STAT`; `MOT` base row references `Running=gv326_00051_ZS1.IO`. | Boolean running/status switch for the motor object. |
| `326_00051_XA` | `DIS` row: overload/fail alarm; `MOT` base row references `Fail=gv326_00051_XA.IO`. | Boolean alarm/fail point. |
| `326_00051_XY1` | `DOS` row: start output; `MOT` base row references `Start=gv326_00051_XY1`. | Boolean start/output command. |
| `326_00051_HOUR` / `326_00051_HR` | `DATREAL` / `RunningHours` rows describe compressor hours and use `ZS1` running status as input. | Real running-hours value. |
| `351_10001_LT1` | `AIS02` row describes ship fuel 1-P tank level; `TK` base row `351_10001` references `LT1=gv351_10001_LT1`. | Real tank level transmitter value. |
| `601_10001_PT` | `AIS` rows describe fuel oil pressure or emergency air pressure depending on vessel. | Real pressure value; description is vessel-specific. |
| `601_10001_FORATE` | `AIS02` row: fuel consumption rate. | Real flow/fuel consumption rate. |
| `601_10001_XA` | `DIS` row: VFD fault alarm in one vessel. | Boolean alarm/fault. |

## Important Caveats

- The tag structure is consistent enough for discovery, but not enough for final meaning without the Hxxx workbook row.
- The same system code can describe different equipment on different vessel classes.
- `TaskName` values such as `Normal`, `Slow`, `HighCom`, `Pumps`, `BVAL`, and `Modbus` are useful hints about update rate/source, not a full semantic definition.
- Some tags in the List are derived application values, not direct IO points. Examples include `EMSDG1.Pow`, `EMS_Main.Speed`, `DENS_FO_N1`, and `Ballast1`.
- The Hxxx source row may be a base object row rather than the exact List tag row. In that case use the object reference columns to map suffixes.