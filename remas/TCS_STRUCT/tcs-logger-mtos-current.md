# TCSLogger Payload Manual - Current MTOS

Applies to MTOS versions: audited against current `master` / `v1.7.1.8`.
Payload: `tcs_diagnostic_struct`
Transport: `TCS_STRUCT` UDP, or serial when `TCSLogger = 1`
Frequency: 1 Hz
Superseded by: not superseded in the audited source.

This document describes the binary payload sent when `function.ini` enables the TCS logger as UDP output from an OS, for example:

```ini
TCSLogger         = 2    # 1=Serial, 2=UDP
TCSLoggerOSNumber = 1    # Only this OS outputs TCS diagnostic data
```

And when `communication.ini` defines the network endpoint, for example:

```ini
[TCS_STRUCT]
IP_A        = 239.192.239.21      # Multicast (or unicast target)
PORT        = 11000
REMOTE_PORT = 11001
```

The active UDP output is one packed C/C++ structure: `tcs_diagnostic_struct`.

Source references:

- `OS\MTOSView\MTOSDoc.cpp`: `CaptureTCSSnapshot()` fills `tcs_diagnostic_struct`.
- `OS\MTOSView\MTOSDoc.cpp`: TCSLogger sends `tcs_diagnostic_struct` on the `TCS_STRUCT` network when `TCSLogger == 2`.
- `OS\MTOSView\MTFunction.cpp` / `MTFunction.h`: reads `TCSLogger` and `TCSLoggerOSNumber` from `function.ini`.
- `OS\MTOSIO\MTOSIOConn.cpp`: opens/uses the `TCS_STRUCT` network connection.
- `Common\Definition\MTTCSDef.h`: `tcs_diagnostic_struct` and GUI array limits.
- `Common\Definition\MTThrusterPackDefinition.h`: `MAX_NO_THRUSTER`, thruster type enums.
- `Common\Definition\MTUnitsDef.h`: `TCS_STRUCT = 89`.
- `Common\Library\MTDeviceId.h`: `eDeviceTypeTCS_STRUCT`.

## Transport

`TCSLogger = 2` sends the raw `tcs_diagnostic_struct` payload over UDP at 1 Hz when the local OS command id matches `TCSLoggerOSNumber`.
`TCSLogger = 1` sends the same packed struct bytes on the Remas serial path.
`TCSLogger > 0` also binary-logs the same snapshot when the OS binary logger is enabled.

Unlike RemasLogger, TCSLogger has no 2 Hz / 10 Hz mode values. The send path is gated by the 1 Hz tick (`bIsOneHz`).

The configured network section is `TCS_STRUCT` (`TCS_STRUCT = 89` in `MTUnitsDef.h`). Demo/base configs commonly use multicast `239.192.239.21`, local `PORT = 11000`, `REMOTE_PORT = 11001`.

The send call uses `NOTADD_SEND_IDENT`, so the UDP payload is the packed struct bytes only, without the normal `struct_ident` application header.

## Structs Available

| Struct | Availability | Notes |
|---|---|---|
| `tcs_diagnostic_struct` | Active TCSLogger UDP/serial payload | Sent by `TCSLogger == 2` on `TCS_STRUCT`, or serial when `TCSLogger == 1`. Packed with `#pragma pack(push,1)`. |

## Important Constants

| Name | Value | Meaning |
|---|---:|---|
| `MAX_NO_THRUSTER` | 10 | Thruster slots in the diagnostic payload. |
| `GUI_MAX_GROUPS` | 24 | Button-group slots per thruster. |
| `GUI_MAX_GROUPS_PER_STRUCT` | 8 | Groups packed into each live GUI indicator struct before sequencing. |
| `GUI_MAX_BUTTONS` | 5 | Buttons per group; packed into each `short` bitfield. |
| `GUI_MAX_ANALOG_IND` | 10 | Analog indication slots per thruster. |
| `MAX_DI_CH` | 32 | Digital indicator bits packed into each thruster `lIndicators` value. |
| `TCS_STRUCT` | 89 | Network unit id for the TCS diagnostic UDP channel. |

Packed size with the constants above: **2080 bytes**.
The trailing comment `//1440 bytes` in `MTTCSDef.h` is stale relative to the current field layout.

`CaptureTCSSnapshot()` zero-initializes the struct, then fills configured thrusters in iteration order. Unused thruster slots remain zero unless later overwritten.

## Value Encoding Conventions

Fields with `float` formats are continuous numeric values in the units shown in the field table.

Thruster boolean status fields such as `sAcceptDP`, `sThrusterRunning`, and `sClutchEngaged` are packed bitfields:

- Bit `N` corresponds to thruster index `N` (`0..9`).
- A set bit means the boolean was true for that thruster when the snapshot was captured.

Button indication/disable values are also bit-packed:

- `sButtonInd[thruster][group]` bit `B` is button `B` (`0..4`) within that group.
- `sButtonDisable[thruster][group]` uses the same packing.

`lIndicators[thruster]` packs digital indicators from `TCSView.ini` into the low 32 bits (`MAX_DI_CH`).

`dwCRC` is present in the struct but is not assigned by `CaptureTCSSnapshot()` in the audited source, so receivers should treat it as `0` unless a vessel-specific branch fills it.

`dwBytePattern1` and `dwBytePattern2` are fixed markers written by `CaptureTCSSnapshot()`:

| Field | Source constant | Little-endian ASCII |
|---|---:|---|
| `dwBytePattern1` | `0x544D544D` | `MTMT` |
| `dwBytePattern2` | `0x5343545F` | `_TCS` |

## Enum and Status Values

### `ThrusterType`

Defined in `MTThrusterPackDefinition.h`. Used by `sThrusterType[]`.
Source comments in `tcs_diagnostic_struct` list the same values:

| Name | Value |
|---|---:|
| `TUNNEL` | 1 |
| `AZIMUTH` | 2 |
| `MAINPROP` | 3 |
| `RUDDER` | 4 |
| `COMBI` | 5 |
| `VOITH` | 6 |

### Active Command Owner

Used by `sActiveCmdOwner[]`:

| Value | Meaning |
|---:|---|
| `0` | None |
| `1` | DP |
| `2` | Lever |
| `3` | Autopilot |

### Drive Program

Used by `sDriveProgram[]`:

| Value | Meaning |
|---:|---|
| `0` | Free running (transit) |
| `1` | Manoeuvre |

### Thruster Control Mode

Used by `sThrusterControlMode[]`:

| Value | Meaning |
|---:|---|
| `0` | Combinator mode |
| `1` | Constant speed mode |

### Preferred CC Bitmask

Used by `sCCPrefCC[3]`. Prefer bitmask, not a device id:

| Value | Meaning |
|---:|---|
| `0` | No explicit preference. |
| `1` | Prefer first CC in the TCS voting group (`TCSCC1`). |
| `2` | Prefer second CC (`TCSCC2`). |
| `4` | Prefer third CC (`TCSCC3`). |

### Vote Result Bitmask

Used by `sTCVoteStatus[6]` from the first six thruster-controller vote replies:

| Bit/value | Meaning |
|---:|---|
| `1` | First CC in group is in use (`TCSCC1`). |
| `2` | Second CC in group is in use (`TCSCC2`). |
| `4` | Third CC in group is in use (`TCSCC3`). |
| `16` | First CC in group has vote error / voted out. |
| `32` | Second CC in group has vote error / voted out. |
| `64` | Third CC in group has vote error / voted out. |
| `256` | First CC in group timed out. |
| `512` | Second CC in group timed out. |
| `1024` | Third CC in group timed out. |

Values can be combined.

### Vote Buffer Values

Used by `sCCVoteBuffer[3][5]` from `thr_voting_struct.sVoteData` for TCSCC1-3:

| Index | Meaning |
|---:|---|
| `0` | Surge command vote value. |
| `1` | Sway command vote value. |
| `2` | Yaw command vote value. |
| `3` | Thruster/status vote value. |
| `4` | Spare / sensor-count vote value, depending on voting source. |

Treat these as raw signed vote/ramp values unless decoding a vessel-specific TCSCC implementation.

### Network Node Status Bitmasks

Filled from live network-node OK status:

| Field | Packing |
|---|---|
| `lOSstatus` | Bit `DeviceIndex` for each OS node that is OK. |
| `sCCstatus` | Bits `0..2` = DPCC1-3; bits `3..5` = TCSCC/LTC1-3. |
| `lTCstatus` | Bit `DeviceIndex` for thruster cards / gateways reported as TC. |
| `lLCstatus` | Bit `DeviceIndex` for lever cards. |
| `lTHRDEVstatus` | Bit `DeviceIndex` for thruster devices. |

A set bit means `IsNetworkNodeStatusOK()` was true for that device index.

### File Version Strings

ASCII version strings, 10 bytes each slot (not necessarily null-terminated if the source string fills the slot):

| Field | Index | Source |
|---|---:|---|
| `szOSFilesVersion` | 0 | `MTOS.exe` |
| `szOSFilesVersion` | 1 | `MTOSIO.dll` |
| `szOSFilesVersion` | 2 | `MTOPPanel.dll` |
| `szOSFilesVersion` | 3..4 | Reserved / unused by current capture path |
| `szCCFilesVersion` | 0 | `DPCC.exe` |
| `szCCFilesVersion` | 1 | `MTIO.dll` |
| `szCCFilesVersion` | 2 | `TCSCC.exe` |
| `szCCFilesVersion` | 3 | `MTGateway.exe` |
| `szCCFilesVersion` | 4 | `MTGatewayIO.dll` |

## Field Layout for Dashboard Mapping

This section is written for project engineers building remote-monitoring dashboards.
Use it to decide **which datapoint belongs on which widget**, not only how to decode bytes.

Offsets assume `#pragma pack(1)` and the constants in this manual.
Total size: **2080 bytes**.
Windows `long` is 4 bytes. `SYSTEMTIME` is 16 bytes (`8 x WORD`).

### How thruster indexes work

Most thruster values are arrays of 10 slots (`MAX_NO_THRUSTER`).

| Concept | Meaning for dashboards |
|---|---|
| Slot index `i` | `0` = first thruster filled by the OS vessel thruster list, then `1`, `2`, ... |
| Unused slots | Remain `0` after zero-init when the vessel has fewer than 10 thrusters. |
| Vessel thruster name | Not in this payload. Map slot `i` to the vessel thruster name/number from project config (for example Thruster 1 = bow tunnel). |
| Bitfield bit `i` | Same thruster as array slot `i`. Example: bit 0 of `sThrusterRunning` is thruster slot 0. |

Practical rule: build one thruster card/widget per configured thruster, bind all `*[i]` and bit `i` fields to that card, and hide unused slots.

### Suggested dashboard groups

| Dashboard area | Use these fields |
|---|---|
| Message health / freshness | `i64TimeStamp`, `dwBytePattern1`, `dwBytePattern2`, payload length |
| Thruster overview tiles | `sThrusterType`, `sThrusterRunning`, `sThrusterReady`, `sThrusterFault`, `sActiveCmdOwner` |
| Command / control ownership | `sActiveCmdOwner`, `sActiveCommandStand`, `sAcceptDP`, `sAcceptAutopilot`, `sBackupModeActive` |
| Thruster demand vs feedback | speed/pitch/angle/thrust reference + feedback pairs |
| Power / load | `dThrusterLoad`, `sPowerReduced`, `sClutchEngaged` |
| Machinery auxiliaries | hydraulic pumps, lift-cylinder locks |
| Control mode labels | `sDriveProgram`, `sThrusterControlMode` |
| Redundancy / voting | `sTCVoteStatus`, `sCCPrefCC`, `sCCVoteBuffer` |
| Network topology health | `lOSstatus`, `sCCstatus`, `lTCstatus`, `lLCstatus`, `lTHRDEVstatus` |
| Software inventory | `szOSFilesVersion`, `szCCFilesVersion` |
| Vessel-specific extras | `lIndicators`, `sButtonInd`, `sButtonDisable`, `dAnalogIndValue` (need `TCSView.ini`) |

---

### 1. Packet time and integrity

| Offset | Field | Type | What it means | Dashboard use |
|---:|---|---|---|---|
| 0 | `i64TimeStamp` | `SYSTEMTIME` | Local OS clock when the snapshot was captured. | Show "last update" / stale-data alarm if older than expected (~1 s cadence). |
| 2070 | `dwCRC` | `short` | Reserved checksum field. | Do **not** rely on this in current MTOS; capture path leaves it `0`. |
| 2072 | `dwBytePattern1` | `long` | Fixed marker `MTMT` (`0x544D544D`). | Validate packet framing before decoding. |
| 2076 | `dwBytePattern2` | `long` | Fixed marker `_TCS` (`0x5343545F`). | Confirm this is TCS diagnostic data, not DP Remas. |

---

### 2. Thruster identity and who is commanding it

All arrays below are `[10]` unless noted.

| Offset | Field | Type | What it means | Dashboard use |
|---:|---|---|---|---|
| 16 | `sThrusterType[i]` | `short` | Physical thruster kind: tunnel, azimuth, main prop, rudder, combi, Voith. | Choose icon/label per thruster tile. See ThrusterType enum. |
| 36 | `sActiveCmdOwner[i]` | `short` | Who currently owns thruster command: None / DP / Lever / Autopilot. | Primary "In command" badge on each thruster. |
| 56 | `sActiveCommandStand[i]` | `short` | Active command stand id for that thruster (`CS1=1`, `CS2=2`, ...). | Show which bridge/wing station holds command when owner is lever/manual. |

---

### 3. Thruster status bits (one short, bits 0..9)

These are **not** per-thruster arrays. Each field is one `short` where **bit `i` = thruster slot `i`**.
Decode with `(value & (1 << i)) != 0`.

| Offset | Field | What bit `i` means | Dashboard use |
|---:|---|---|---|
| 76 | `sAcceptDP` | Thruster is enabled/available for DP control. | Show whether DP can take this thruster. |
| 78 | `sAcceptAutopilot` | Thruster is enabled/available for autopilot. | Show AP availability. |
| 80 | `sBackupModeActive` | Thruster is in TCS backup mode. | Warning/state chip; backup is a special local operating mode. |
| 82 | `sThrusterRunning` | Running feedback is true. | Green running indication. |
| 84 | `sThrusterReady` | Ready feedback is true. | Ready/standby indication. |
| 86 | `sThrusterFault` | Fault feedback is true. | Alarm/red state. |
| 88 | `sPowerReduced` | PMS has reduced thruster power. | Power-limit warning. |
| 90 | `sClutchEngaged` | Engine/clutch 1 engaged. | Clutch status lamp. |
| 92 | `sHydrPump1Running` | Hydraulic pump 1 running. | Aux machinery status. |
| 94 | `sHydrPump2Running` | Hydraulic pump 2 running. | Aux machinery status. |
| 96 | `sLiftCylinderUpperLocked` | Retractable thruster locked upper. | Useful on retractable units; ignore if vessel has none. |
| 98 | `sLiftCylinderLowerLocked` | Retractable thruster locked lower. | Same as above. |

Typical overview logic for thruster `i`:

1. Fault bit set → show fault.
2. Else running bit set → show running.
3. Else ready bit set → show ready.
4. Else show stopped/unavailable.

---

### 4. Thruster command and feedback analogs

These are the main live values for remote thruster monitoring.
All are `float[10]`.

| Offset | Field | Typical engineering meaning | Dashboard use |
|---:|---|---|---|
| 100 | `dThrusterSpeedReference[i]` | Commanded RPM/speed signal from TCS (normally normalized about `-1..1`, where `±1` is full scale). | Command needle / setpoint. |
| 140 | `dThrusterSpeedFeedback[i]` | Measured RPM/speed feedback signal (same normalized scale in normal RPM configs; some vessels use shaft-speed feedback instead). | Feedback needle; compare to reference for follow-up error. |
| 180 | `dThrusterPitchReference[i]` | Commanded pitch signal (normally normalized about `-1..1`). | Pitch setpoint. Relevant for CPP / pitch-controlled units. |
| 220 | `dThrusterPitchFeedback[i]` | Measured pitch feedback (same normalized scale). | Pitch feedback. |
| 260 | `dThrusterAngleReference[i]` | Commanded azimuth or rudder angle in **degrees**. | Direction setpoint for azimuth/rudder widgets. |
| 300 | `dThrusterAngleFeedback[i]` | Measured azimuth/rudder angle in **degrees**. | Direction feedback. |
| 340 | `dThrusterLoad[i]` | Thruster electrical/mechanical load in **kW**. Source may leave a very large negative sentinel when no power feedback is configured. | Load bar / kW readout. Treat extreme negatives as "no data". |
| 380 | `dThrustCommand[i]` | Calculated relative thrust command, typically normalized to max thrust (about `-1..1`). | Overall thrust demand gauge. |
| 420 | `dThrustFeedback[i]` | Calculated relative thrust feedback, typically normalized to max thrust (about `-1..1`). | Overall thrust achieved gauge. |

Notes for engineers:

- Prefer **reference + feedback pairs** on the same widget so operators can see demand vs response.
- Speed/pitch are usually **normalized signals**, not raw RPM/% text unless the vessel project documents a conversion.
- Angle fields are already in degrees in this payload.
- For tunnel thrusters, angle widgets are usually not meaningful; for rudders, speed/pitch may be unused.

---

### 5. Thruster control settings

| Offset | Field | Type | What it means | Dashboard use |
|---:|---|---|---|---|
| 460 | `sDriveProgram[i]` | `short[10]` | `0` = free running / transit program, `1` = manoeuvre program. | Label "Transit" vs "Manoeuvre" on thruster or vessel status. |
| 480 | `sThrusterControlMode[i]` | `short[10]` | `0` = combinator mode, `1` = constant-speed mode. | Show how RPM/pitch are being coordinated. |

---

### 6. TCS voting and preferred controller

These support redundancy views (which TCSCC is preferred / in use, and vote health).

| Offset | Field | Type | What it means | Dashboard use |
|---:|---|---|---|---|
| 500 | `sTCVoteStatus[j]` | `short[6]` | Vote-result bitmask from the first six thruster-controller vote replies. | Redundancy health for TC voting. Decode with Vote Result Bitmask. |
| 512 | `sCCPrefCC[k]` | `short[3]` | Preferred TCSCC bitmask for TCSCC1-3 (`k=0..2`). | Show preferred controller preference, not a device-id number. |
| 518 | `sCCVoteBuffer[k][n]` | `short[3][5]` | Raw vote channels from TCSCC1-3. `n`: surge, sway, yaw, thruster/status, spare. | Advanced diagnostics; usually not first-page dashboard values. |

---

### 7. Software versions on OS and controllers

Each string slot is 10 ASCII bytes (may be unterminated if fully filled).

| Offset | Field | Slot | Source file | Dashboard use |
|---:|---|---:|---|---|
| 548 | `szOSFilesVersion` | 0 | `MTOS.exe` | OS application version. |
| 548 | `szOSFilesVersion` | 1 | `MTOSIO.dll` | OS I/O library version. |
| 548 | `szOSFilesVersion` | 2 | `MTOPPanel.dll` | Operator panel library version. |
| 548 | `szOSFilesVersion` | 3..4 | unused by current capture | Ignore. |
| 598 | `szCCFilesVersion` | 0 | `DPCC.exe` | DP controller version. |
| 598 | `szCCFilesVersion` | 1 | `MTIO.dll` | Controller I/O library version. |
| 598 | `szCCFilesVersion` | 2 | `TCSCC.exe` | TCS controller version. |
| 598 | `szCCFilesVersion` | 3 | `MTGateway.exe` | Gateway version. |
| 598 | `szCCFilesVersion` | 4 | `MTGatewayIO.dll` | Gateway I/O version. |

Use these on a "System info / versions" page, not on the live thruster overview.

---

### 8. Network node health bitmasks

A set bit means that node currently reports network status OK.

| Offset | Field | Type | Bit meaning | Dashboard use |
|---:|---|---|---|---|
| 648 | `lOSstatus` | `long` | Bit `DeviceIndex` of each OS. | OS online matrix. |
| 652 | `sCCstatus` | `short` | Bits `0..2` = DPCC1-3; bits `3..5` = TCSCC/LTC1-3. | Controller online lamps. |
| 654 | `lTCstatus` | `long` | Bit `DeviceIndex` of thruster cards/gateways typed as TC. | Thruster-card communication health. |
| 658 | `lLCstatus` | `__int64` | Bit `DeviceIndex` of lever cards. | Lever-card health. |
| 666 | `lTHRDEVstatus` | `long` | Bit `DeviceIndex` of thruster devices. | Drive/device health. |

Map bit positions to vessel device names from `communication.ini` / network config. Do not assume every vessel uses the same device indexes.

---

### 9. Vessel-specific GUI indicators

These fields mirror TCS GUI configuration and are **project-specific**.
Do not hard-code global meanings; resolve labels from vessel `TCSView.ini` / GUI config.

| Offset | Field | Type | What it means | Dashboard use |
|---:|---|---|---|---|
| 670 | `lIndicators[i]` | `long[10]` | Up to 32 digital indicator bits for thruster `i` (`bit 0..31`). | Custom lamps defined for that vessel thruster page. |
| 710 | `sButtonInd[i][g]` | `short[10][24]` | Button LED/indication bits for thruster `i`, group `g` (`0..23`). Bits `0..4` = buttons in that group. | Recreate custom button feedback states if required ashore. |
| 1190 | `sButtonDisable[i][g]` | `short[10][24]` | Same packing; bit set means button is disabled / not available. | Grey-out / unavailable indication for custom controls. |
| 1670 | `dAnalogIndValue[i][a]` | `float[10][10]` | Analog indication `a` (`0..9`) for thruster `i`. | Custom gauges (pressure, current, etc.) only when the vessel defines them. |

If the remote dashboard only needs standard thruster monitoring, you can ignore this whole group and still cover running/fault/command/feedback/load.

---

### Compact binary offset map

Use this when implementing parsers. For meaning, use the sections above.

| Offset | Type | Field | Shape |
|---:|---|---|---|
| 0 | `SYSTEMTIME` | `i64TimeStamp` | 1 |
| 16 | `short` | `sThrusterType` | `[10]` |
| 36 | `short` | `sActiveCmdOwner` | `[10]` |
| 56 | `short` | `sActiveCommandStand` | `[10]` |
| 76 | `short` | `sAcceptDP` | 1 |
| 78 | `short` | `sAcceptAutopilot` | 1 |
| 80 | `short` | `sBackupModeActive` | 1 |
| 82 | `short` | `sThrusterRunning` | 1 |
| 84 | `short` | `sThrusterReady` | 1 |
| 86 | `short` | `sThrusterFault` | 1 |
| 88 | `short` | `sPowerReduced` | 1 |
| 90 | `short` | `sClutchEngaged` | 1 |
| 92 | `short` | `sHydrPump1Running` | 1 |
| 94 | `short` | `sHydrPump2Running` | 1 |
| 96 | `short` | `sLiftCylinderUpperLocked` | 1 |
| 98 | `short` | `sLiftCylinderLowerLocked` | 1 |
| 100 | `float` | `dThrusterSpeedReference` | `[10]` |
| 140 | `float` | `dThrusterSpeedFeedback` | `[10]` |
| 180 | `float` | `dThrusterPitchReference` | `[10]` |
| 220 | `float` | `dThrusterPitchFeedback` | `[10]` |
| 260 | `float` | `dThrusterAngleReference` | `[10]` |
| 300 | `float` | `dThrusterAngleFeedback` | `[10]` |
| 340 | `float` | `dThrusterLoad` | `[10]` |
| 380 | `float` | `dThrustCommand` | `[10]` |
| 420 | `float` | `dThrustFeedback` | `[10]` |
| 460 | `short` | `sDriveProgram` | `[10]` |
| 480 | `short` | `sThrusterControlMode` | `[10]` |
| 500 | `short` | `sTCVoteStatus` | `[6]` |
| 512 | `short` | `sCCPrefCC` | `[3]` |
| 518 | `short` | `sCCVoteBuffer` | `[3][5]` |
| 548 | `char` | `szOSFilesVersion` | `[5][10]` |
| 598 | `char` | `szCCFilesVersion` | `[5][10]` |
| 648 | `long` | `lOSstatus` | 1 |
| 652 | `short` | `sCCstatus` | 1 |
| 654 | `long` | `lTCstatus` | 1 |
| 658 | `__int64` | `lLCstatus` | 1 |
| 666 | `long` | `lTHRDEVstatus` | 1 |
| 670 | `long` | `lIndicators` | `[10]` |
| 710 | `short` | `sButtonInd` | `[10][24]` |
| 1190 | `short` | `sButtonDisable` | `[10][24]` |
| 1670 | `float` | `dAnalogIndValue` | `[10][10]` |
| 2070 | `short` | `dwCRC` | 1 |
| 2072 | `long` | `dwBytePattern1` | 1 |
| 2076 | `long` | `dwBytePattern2` | 1 |

## Receiver Checklist

1. Match on UDP destination from `[TCS_STRUCT]` (`IP_A` / `PORT` / `REMOTE_PORT`).
2. Expect raw packed bytes with no `struct_ident` header.
3. Validate trailing markers `MTMT` / `_TCS`.
4. Confirm payload length is **2080** bytes for the current layout.
5. Decode thruster bitfields with thruster index as bit position.
6. Map thruster slot indexes to vessel thruster names from project config.
7. Resolve custom button/analog meanings from vessel `TCSView.ini` / GUI config only when those widgets are required.
