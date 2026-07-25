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

## Field Layout

Offsets below assume `#pragma pack(1)` and the constants in this manual.
Total size: **2080 bytes**.

| Offset | Type | Field | Count / shape | Unit / notes |
|---:|---|---|---|---|
| 0 | `SYSTEMTIME` | `i64TimeStamp` | 1 | Local OS time when the snapshot was captured. |
| 16 | `short` | `sThrusterType` | `[10]` | Thruster type enum. |
| 36 | `short` | `sActiveCmdOwner` | `[10]` | 0=None, 1=DP, 2=Lever, 3=Autopilot. |
| 56 | `short` | `sActiveCommandStand` | `[10]` | Active command stand for each thruster. |
| 76 | `short` | `sAcceptDP` | 1 | Bitfield: DP enabled per thruster. |
| 78 | `short` | `sAcceptAutopilot` | 1 | Bitfield: Autopilot enabled per thruster. |
| 80 | `short` | `sBackupModeActive` | 1 | Bitfield: backup mode active. |
| 82 | `short` | `sThrusterRunning` | 1 | Bitfield: thruster running. |
| 84 | `short` | `sThrusterReady` | 1 | Bitfield: thruster ready. |
| 86 | `short` | `sThrusterFault` | 1 | Bitfield: thruster fault. |
| 88 | `short` | `sPowerReduced` | 1 | Bitfield: PMS power reduced. |
| 90 | `short` | `sClutchEngaged` | 1 | Bitfield: clutch engaged. |
| 92 | `short` | `sHydrPump1Running` | 1 | Bitfield: hydraulic pump 1 running. |
| 94 | `short` | `sHydrPump2Running` | 1 | Bitfield: hydraulic pump 2 running. |
| 96 | `short` | `sLiftCylinderUpperLocked` | 1 | Bitfield: lift cylinder upper locked. |
| 98 | `short` | `sLiftCylinderLowerLocked` | 1 | Bitfield: lift cylinder lower locked. |
| 100 | `float` | `dThrusterSpeedReference` | `[10]` | Thruster speed reference. |
| 140 | `float` | `dThrusterSpeedFeedback` | `[10]` | Thruster speed feedback. |
| 180 | `float` | `dThrusterPitchReference` | `[10]` | Pitch reference. |
| 220 | `float` | `dThrusterPitchFeedback` | `[10]` | Pitch feedback. |
| 260 | `float` | `dThrusterAngleReference` | `[10]` | Azimuth/rudder angle reference. |
| 300 | `float` | `dThrusterAngleFeedback` | `[10]` | Azimuth/rudder angle feedback. |
| 340 | `float` | `dThrusterLoad` | `[10]` | Thruster load (kW). |
| 380 | `float` | `dThrustCommand` | `[10]` | Calculated thrust command. |
| 420 | `float` | `dThrustFeedback` | `[10]` | Calculated thrust feedback. |
| 460 | `short` | `sDriveProgram` | `[10]` | 0=transit / free running, 1=manoeuvre. |
| 480 | `short` | `sThrusterControlMode` | `[10]` | 0=combinator, 1=constant speed. |
| 500 | `short` | `sTCVoteStatus` | `[6]` | Vote reply bitmask from first 6 TCs. |
| 512 | `short` | `sCCPrefCC` | `[3]` | Preferred TCSCC bitmask for TCSCC1-3. |
| 518 | `short` | `sCCVoteBuffer` | `[3][5]` | TCSCC voting values from TCSCC1-3. |
| 548 | `char` | `szOSFilesVersion` | `[5][10]` | OS file version strings. |
| 598 | `char` | `szCCFilesVersion` | `[5][10]` | CC/gateway file version strings. |
| 648 | `long` | `lOSstatus` | 1 | OS network OK bitmask. |
| 652 | `short` | `sCCstatus` | 1 | DPCC/TCSCC network OK bitmask. |
| 654 | `long` | `lTCstatus` | 1 | TC/gateway network OK bitmask. |
| 658 | `__int64` | `lLCstatus` | 1 | Lever-card network OK bitmask. |
| 666 | `long` | `lTHRDEVstatus` | 1 | Thruster-device network OK bitmask. |
| 670 | `long` | `lIndicators` | `[10]` | Digital indicators from `TCSView.ini`. |
| 710 | `short` | `sButtonInd` | `[10][24]` | Button LED/indication bits per group. |
| 1190 | `short` | `sButtonDisable` | `[10][24]` | Button disable / not-available bits per group. |
| 1670 | `float` | `dAnalogIndValue` | `[10][10]` | Analog indication values from GUI config. |
| 2070 | `short` | `dwCRC` | 1 | Present in struct; not filled by current capture path. |
| 2072 | `long` | `dwBytePattern1` | 1 | Marker `MTMT` (`0x544D544D`). |
| 2076 | `long` | `dwBytePattern2` | 1 | Marker `_TCS` (`0x5343545F`). |

Windows `long` is 4 bytes in this codebase. `SYSTEMTIME` is 16 bytes (`8 x WORD`).

## Receiver Checklist

1. Match on UDP destination from `[TCS_STRUCT]` (`IP_A` / `PORT` / `REMOTE_PORT`).
2. Expect raw packed bytes with no `struct_ident` header.
3. Validate trailing markers `MTMT` / `_TCS`.
4. Confirm payload length is **2080** bytes for the current layout.
5. Decode thruster bitfields with thruster index as bit position.
6. Resolve button/analog meanings from vessel `TCSView.ini` / GUI config, not from fixed global enums.
