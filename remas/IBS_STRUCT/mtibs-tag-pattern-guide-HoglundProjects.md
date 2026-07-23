# MTIBS Tag Pattern Guide ? H?glund / Polish HMA Projects

This guide describes the MTIBS tag patterns found in Polish HMA projects implemented by a third party (H?glund). The spreadsheet layout is the same GenSheet / List / detail-workbook model as other IBS vessels, but the tag naming style is different from the MT in-house pattern documented in `mtibs-tag-pattern-guide.md`.

Research sample: `Polish HMA Projects` GenSheet set for Remontowa B851 (files such as `Gmr100.xls`, `RemontowaB851_4_MAIN.xls`, `RemontowaB851_4_COM.xls`, `RemontowaB851_4_PMS.xls`). Source Excel files stay local and are not copied to GitHub.

| List source | MTIBS rows | Tags | Unique tags | Bool tags | Real tags |
|---|---:|---:|---:|---:|---:|
| `Gmr100.xls` | 69 | 2167 | 2167 | 1751 | 416 |

## What MTIBS Sends

MTIBS is an MT proprietary telegram used to send IBS data to shore. The `List` tab in `Gmr100.xls` defines how many MTIBS telegrams are sent and which tags are included in each telegram.

A typical row looks like:

```text
MTIBS1, 1, T,B=441_1_1_LS1, T,B=441_1_1_LS2, T,B=441_1_1_PS1, T,B=441_1_1_PS2, T,R=441_1_1_PS3, T,R=441_1_1_PS4, T,R=441_1_1_PS5, T,R=441_1_1_PS6, T,B=441_1_1_PS7, T,R=441_1_1_PT1, ...
```

The first column is usually the telegram name, the second is the telegram number, and the remaining cells are comma-separated tag definitions.

## List Token Format

| Token part | Meaning |
|---|---|
| `T` | Data point/tag entry. |
| `B` | Boolean value. Interpret as `0/1`, commonly closed/open, off/on, normal/alarm, or false/true depending on the tag description. |
| `R` | Real value. Interpret as a numeric analog/derived value. Units must be resolved from the detail workbook or project context. |
| `<tag>` | The tag name to cross-reference in the Remontowa / MAIN / PMS sheets. |

Examples:

| Token | Interpretation |
|---|---|
| `T,B=441_1_1_LS1` | Boolean low-level switch for DG1 LT tank water. |
| `T,R=441_1_1_PS3` | Real starting-air pressure for DG1. |
| `T,B=512_1_02_03_ZSH` | Boolean opened status for FO valve `512.1-02.03`. |
| `T,R=512_1_A044_LT` | Real level transmitter for FO EDG service tank `A044`. |

## Tag Name Families

H?glund List tabs are dominated by short underscore-separated numeric tags. Compared with the MT in-house style (`326_00051_ZS1`), these tags use smaller middle segments and often encode P&ID valve or tank IDs directly.

| Family | Shape | Examples | Notes |
|---|---|---|---|
| Equipment / point tags | `<system>_<group>_<unit>_<suffix>` | `441_1_1_LS1`, `511_2_2_ZS1`, `512_1_1_PT1` | Most common for DG sensors, pumps, fans, and alarms. |
| Valve / line tags | `<system>_<group>_<line>_<valve>[_suffix]` | `512_1_02_03`, `512_1_02_03_ZSH`, `522_5_01_01_ZZL` | Common on FO, dry-bulk, and methanol valves. Base object lives in `BVAL`. |
| Tank / A-code tags | `<system>_<group>_A<nnn>[_suffix]` | `512_1_A044`, `512_1_A044_LT`, `512_1_A026_LT` | Tank objects in `TK`; level usually `_LT` rather than `_LT1`. |
| Base object tags | `<system>_<group>_<unit>` | `511_2_2`, `513_1_1`, `512_1_1` | Motor/object rows in `MOT` / `BVAL` / `TK` without a signal suffix. |
| Variant letter tags | `<system>_<group>_<unit><letter>` | `511_2_2E`, `511_4_2E` | Extra motor/object variant; resolve description in `MOT`. |
| Occasional MT-style tags | `<system>_<5digit>_<suffix>` | `404_10001_XI`, `871_10099_ZZL` | A small minority in this sample; treat like MT in-house tags when present. |
| Named tags | Word names | `PRINTERCTRL` | Rare; resolve in detail sheets by exact `TagName`. |

## Structured Numeric Tags

Typical H?glund equipment/point tag:

```text
441_1_1_LS1
|   | | |
|   | | suffix / signal role
|   | equipment or unit number
|   group / instance within the system
system or process family code
```

Typical valve tag:

```text
512_1_02_03_ZSH
|   | |  |  |
|   | |  |  suffix / limit role
|   | |  valve number
|   | line or valve group
|   group / instance
system or process family code
```

Typical tank tag:

```text
512_1_A044_LT
|   | |    |
|   | |    suffix / signal role
|   | tank A-code from project drawings / GenSheet
|   group / instance
system or process family code
```

### System / Process Code

The leading numeric code groups tags by vessel system. Always verify in the detail workbook because meanings are project-specific.

Observed examples from Remontowa B851:

| Code | Observed descriptions / context |
|---|---|
| `441` | Diesel generators (DG1?DG4) pressures, temperatures, levels, alarms, resets. |
| `442` | Heat-exchanger / DG boost and related cooling values. |
| `511` | Central cooling SW/FW pumps and related status/commands. |
| `512` | Fuel oil transfer pumps, FO valves, FO tanks. |
| `513` | Lub oil transfer / filter pumps. |
| `514` | Starting-air compressor alarms. |
| `521` | Bilge pumps. |
| `522` | Dry-bulk and methanol valves/pumps. |
| `531` | Fresh-water tank overflow / level alarms. |
| `551` | Fire pumps. |
| `562` | Engine-room fans. |
| `572` | Sewage treatment plant alarms. |
| `580` | Provision / HVAC compressor alarms. |
| `601` | Main HV transformer alarms (also a few MT-style `601_xxxxx` tags). |
| `610` | Generator electrical values such as active power. |
| `611` | Azimuth thruster winding temperatures / machinery. |
| `613` | Fi-Fi system common alarm. |
| `614` | Azimuth thruster AQM oil level alarms. |
| `633` | Navigation lights common alarm. |
| `641` | GMDSS power-supply fail. |
| `663`, `670`, `672`, `673` | UPS / AMS supply and insulation alarms. |
| `675` | Draught measurement. |
| `404` | Bridge/manual lever mode indicators (MT-style IDs in this sample). |

Treat this as a lookup hint, not a contract. The authoritative text is the matching `Descr` in the detail workbook.

### Group / Unit / Line Segments

Unlike MT in-house five-digit equipment numbers (`00051`, `10001`), H?glund middle segments are short and often mirror drawing numbers:

| Segment pattern | Typical meaning |
|---|---|
| second number (`1`, `2`, `5`, `6`, ?) | Subsystem instance, side group, or package number. |
| third number (`1`, `2`, `15`, ?) | Equipment / unit number within that group. |
| third + fourth (`02_03`, `01_01`, `22_04`) | Valve line and valve number, often shown in descriptions as `512.1-02.03`. |
| `A026`, `A044`, `A049` | Tank A-codes used as the object ID in `TK`. |
| trailing letter on unit (`2E`) | Object variant; confirm in `MOT`. |

Examples:

| Base tag | Description evidence |
|---|---|
| `441_1_1` | DG1 points (`DG1 STARTING AIR PRESSURE`, `DG1 LOW LT TANK WATER LEVEL`). |
| `511_2_2` | Central cooling SW pump No.2. |
| `512_1_02_03` | FO valve No. `512.1-02.03`. |
| `512_1_A044` | FO EDG service tank. |
| `522_5_01_01` | Methanol valve No. `522.5-01.01`. |

### Suffix / Signal Role

Suffixes are largely the same ISA-style letters seen on MT projects, with a few H?glund preferences.

| Suffix | Observed meaning / use | Typical type in MTIBS |
|---|---|---|
| `_PT`, `_PT1`, `_PT2` | Pressure transmitter / pressure analog. | `T,R` |
| `_TT`, `_TT1`?`_TT8` | Temperature transmitter / temperature analog. | `T,R` |
| `_LT` | Tank level transmitter. Prefer `_LT` here; `_LT1`/`_LT2` are less common than on MT in-house sheets. | `T,R` |
| `_PS`, `_PS1`? | Pressure switch or pressure alarm/status. Type follows List token (`B` or `R`). | `T,B` or `T,R` |
| `_TS`, `_TS1`? | Temperature switch / temperature status. | Usually `T,B` or `T,R` as listed |
| `_LS`, `_LS1`, `_LS2` | Level switch / low-level alarm. | `T,B` |
| `_RH` | Running-hours / hours accumulator linked to running status. | `T,R` |
| `_XA`, `_XA1`, `_XA2`? | Alarm / fail / common-alarm indication. | `T,B` |
| `_XC`, `_XC1`, `_XC2` | Command/control/reset outputs. Example: `441_1_1_XC1` = DG1 reset. | `T,B` |
| `_XI`, `_XI1`? | Indication / measured status or electrical indication. | `T,B` or `T,R` |
| `_XY0` | Stop / close command. | `T,B` |
| `_XY1` | Start / open command. | `T,B` |
| `_ZS1` | Running / position status. | `T,B` |
| `_HS1` | Local / auto or hand status. | `T,B` |
| `_ZSH`, `_ZSL` | Valve opened / closed status. | `T,B` |
| `_ZZH`, `_ZZL` | Valve open / close command (or related end positions depending on object wiring). On `BVAL`, Open/Close often reference `ZZH`/`ZZL`. | `T,B` |

The List token type (`B` or `R`) is authoritative for the value format sent in MTIBS.

## How To Resolve a Tag

1. Open `Gmr100.xls` and find the `List` tab.
2. Collect all `T,B=<tag>` and `T,R=<tag>` entries from MTIBS rows.
3. Search the Remontowa detail workbooks for exact `TagName = <tag>`. Start with `RemontowaB851_4_MAIN.xls`, then `RemontowaB851_4_PMS.xls` / `RemontowaB851_4_COM.xls` as needed.
4. If exact search fails, strip the final suffix and search the base object, for example search `511_2_2` for `511_2_2_ZS1`, or `512_1_A044` for `512_1_A044_LT`.
5. For valve tags, also search `BVAL` using the four-segment base (`512_1_02_03`) and read Open/Close/Opened/Closed reference columns.
6. Use the row's `TaskName`, `Descr`, and communication/reference columns to interpret the point.
7. Use the List token type for the transmitted value format: `B` boolean or `R` real.

## Useful Detail Sheets

Workbook names in this sample use the project prefix `RemontowaB851_4_*` instead of `Hxxx_*`, but the sheet families are the same.

| Sheet family | Workbook examples | What it usually contains | Useful columns |
|---|---|---|---|
| `AIS`, `AIS02` | `*_MAIN.xls`, `*_PMS.xls` | Analog inputs / measured values. | `TagName`, `Descr`, ranges, alarm limits. |
| `DIS`, `DIS02` | `*_MAIN.xls`, `*_PMS.xls` | Digital inputs / status / alarms. | `TagName`, `Descr`, `NormPos`, CommProvider/address/bit. |
| `DOS` | `*_MAIN.xls` | Digital outputs / commands. | `TagName`, `Descr`, pulse/command source. |
| `AOS` | `*_MAIN.xls` | Analog outputs. | `TagName`, `Descr`. |
| `BVAL` | `*_MAIN.xls` | Valve objects. | Base `TagName`, `Descr`, `Open`, `Close`, `Opened`, `Closed`, `Local`. |
| `TK` | `*_MAIN.xls` | Tank objects. | Base `TagName`, `Descr`, `LT1`/`LT2` refs (often `gv..._LT`). |
| `MOT` | `*_MAIN.xls` | Motor/pump/fan objects. | Base `TagName`, `Descr`, `Start`, `Stop`, `Running`, `Fail`. |
| `RunningHours` | `*_MAIN.xls` | Hours accumulators. | `TagName`, `Descr`, input references. |
| `MBTCP*`, `MBRead*`, `MBSlave` | `*_COM.xls` | Communication mapping. | Register/coil mapping columns. |
| `GEN`, `GenAdmin` | `*_PMS.xls` | Generator / PMS objects. | Generator status and command tags. |

## Evidence Examples

These examples were cross-referenced from List tags into Remontowa B851 detail sheets.

| List tag | Detail evidence | Meaning inferred |
|---|---|---|
| `441_1_1_LS1` | `DIS`: `Descr=DG1 LOW LT TANK WATER LEVEL`. | Boolean low-level switch for DG1 LT tank. |
| `441_1_1_PS3` | `AIS`: `Descr=DG1 STARTING AIR PRESSURE`. | Real DG1 starting-air pressure. |
| `441_1_2_TT7` | `AIS`: `Descr=DG2 FRESH WATER TEMP OUTLET`. | Real DG2 FW outlet temperature. |
| `441_1_1_XC1` | `DOS`: `Descr=DG1 RESET`. | Boolean reset command. |
| `511_2_2_ZS1` | `DIS02`: running status; `MOT` base `511_2_2` = Central cooling SW pump No.2. | Boolean pump running. |
| `511_2_2_HS1` | `DIS02`: `CC SW PUMP2 LOCAL/AUTO`. | Boolean local/auto status. |
| `511_2_2_XA` | `DIS`: `CENTRAL COOLING SW PUMP 2 FAILURE`. | Boolean fail alarm. |
| `511_2_2_XY0` / `_XY1` | `DOS`: stop / start commands. | Boolean stop and start outputs. |
| `511_2_2_RH` | `RunningHours`: linked to pump running. | Real running-hours value. |
| `512_1_02_03_ZSH` / `_ZSL` | `DIS02`: FO valve opened/closed; `BVAL` base `512_1_02_03`. | Boolean valve position status. |
| `512_1_02_03` Open/Close refs | `BVAL` Open=`gv512_1_02_03_ZZH`, Close=`gv512_1_02_03_ZZL`, Opened/Closed use `ZSH`/`ZSL`. | `ZZH`/`ZZL` command side; `ZSH`/`ZSL` feedback side. |
| `512_1_A044_LT` | `AIS02`: FO EDG service tank level; `TK` base `512_1_A044`. | Real tank level. |
| `513_1_1_ZS1` | `DIS02` + `MOT` clean LO transfer pump. | Boolean pump running. |
| `404_10001_XI` | `DIS02`: `BRIDGE IN MANUAL LEVER MODE`. | Boolean bridge mode indication using MT-style ID. |

## Important Caveats

- The H?glund tag shape is consistent enough for discovery, but final meaning still requires the detail workbook `Descr` row.
- Do not assume MT in-house five-digit equipment IDs. Most tags here use short group/unit/line segments or tank A-codes.
- `_XY0` is stop/close and `_XY1` is start/open in this sample. Confirm before mapping dashboards.
- Valve command vs feedback can look similar (`ZZH`/`ZZL` vs `ZSH`/`ZSL`). Prefer `BVAL` reference columns over guessing from the suffix alone.
- A few MT-style tags coexist in the same List. Resolve them the same way: exact `TagName` search, then base-tag fallback.
- Workbook names are project-prefixed (`RemontowaB851_4_MAIN.xls`), but sheet families (`AIS`, `DIS`, `MOT`, `BVAL`, `TK`) match the usual IBS GenSheet model.