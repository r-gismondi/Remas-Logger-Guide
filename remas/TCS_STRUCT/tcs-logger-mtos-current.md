# TCS Logger Datapoint Guide - Current MTOS

Audience: project engineers reading TCS datapoint values to build remote-monitoring dashboards.

Goal of this guide: for each datapoint, understand **what it is** and **what the value means**.

Applies to current MTOS (`master` / `v1.7.1.8`).
Data source: `tcs_diagnostic_struct` sent once per second when TCSLogger is enabled.

Example enablement:

```ini
# function.ini
TCSLogger         = 2    # 1=Serial, 2=UDP
TCSLoggerOSNumber = 1    # Only this OS outputs TCS data

# communication.ini
[TCS_STRUCT]
IP_A        = 239.192.239.21
PORT        = 11000
REMOTE_PORT = 11001
```

## How to read thruster datapoints

Many datapoints are per thruster. There are up to **10 thruster slots** (`0` to `9`).

| What you see | Meaning |
|---|---|
| ` somehow[0]`, `something[1]`, ... | Value for thruster slot 0, 1, ... |
| Unused slots | Usually `0` when the vessel has fewer than 10 thrusters |
| Vessel thruster name | Not included in the data. Map slot index to the vessel thruster name from project config (example: slot 0 = Bow Tunnel) |

Some status datapoints are packed into one number:

| What you see | Meaning |
|---|---|
| Bitfield value | One number holding true/false for thrusters 0..9 |
| Bit `i` set | That condition is true for thruster slot `i` |
| Bit `i` clear | That condition is false for thruster slot `i` |

Example: if `sThrusterRunning = 5`, bits 0 and 2 are set -> thrusters 0 and 2 are running.

## Suggested dashboard use

| Dashboard area | Start with these datapoints |
|---|---|
| Thruster overview | `sThrusterType`, `sThrusterRunning`, `sThrusterReady`, `sThrusterFault`, `sActiveCmdOwner` |
| Who is in command | `sActiveCmdOwner`, `sActiveCommandStand`, `sAcceptDP`, `sAcceptAutopilot`, `sBackupModeActive` |
| Demand vs feedback | speed / pitch / angle / thrust reference + feedback pairs |
| Power | `dThrusterLoad`, `sPowerReduced`, `sClutchEngaged` |
| Auxiliaries | hydraulic pumps, lift-cylinder locks |
| Control mode | `sDriveProgram`, `sThrusterControlMode` |
| Redundancy | `sTCVoteStatus`, `sCCPrefCC` |
| Network health | `lOSstatus`, `sCCstatus`, `lTCstatus`, `lLCstatus`, `lTHRDEVstatus` |
| Software versions | `szOSFilesVersion`, `szCCFilesVersion` |
| Vessel-specific extras | `lIndicators`, `sButtonInd`, `sButtonDisable`, `dAnalogIndValue` (need vessel `TCSView.ini`) |

---

## 1. Time and packet markers

| Datapoint | What it is | What the value means |
|---|---|---|
| `i64TimeStamp` | Time when this TCS snapshot was taken on the OS | Local OS date/time. Use as "last update" / stale-data check. Data is normally 1 Hz. |
| `dwCRC` | Reserved checksum | Currently unused (`0`). Ignore for dashboard logic. |
| `dwBytePattern1` | Fixed packet marker | Always `MTMT`. Confirms framing is intact. |
| `dwBytePattern2` | Fixed packet marker | Always `_TCS`. Confirms this is TCS diagnostic data (not DP Remas). |

---

## 2. Thruster identity and command ownership

Per thruster slot `i` (`0..9`).

| Datapoint | What it is | What the value means |
|---|---|---|
| `sThrusterType[i]` | Physical type of thruster | `1` Tunnel, `2` Azimuth, `3` Main prop, `4` Rudder, `5` Combi, `6` Voith |
| `sActiveCmdOwner[i]` | Who currently commands this thruster | `0` None, `1` DP, `2` Lever, `3` Autopilot, `4` Service mode, `5` GUI / local, `6` External |
| `sActiveCommandStand[i]` | Which command stand owns it | Command stand number: `1` = CS1, `2` = CS2, and so on. Useful when owner is Lever. |

---

## 3. Thruster status (true/false per thruster)

Each datapoint below is one number. **Bit `i` = thruster slot `i`.**

| Datapoint | What it is | What the value means |
|---|---|---|
| `sAcceptDP` | Thruster available to DP | Bit set = DP can use this thruster |
| `sAcceptAutopilot` | Thruster available to autopilot | Bit set = autopilot can use this thruster |
| `sBackupModeActive` | TCS backup mode | Bit set = thruster is in backup mode |
| `sThrusterRunning` | Running feedback | Bit set = thruster is running |
| `sThrusterReady` | Ready feedback | Bit set = thruster is ready |
| `sThrusterFault` | Fault feedback | Bit set = thruster has a fault |
| `sPowerReduced` | Power management reduction | Bit set = PMS has reduced thruster power |
| `sClutchEngaged` | Clutch engaged | Bit set = clutch 1 is engaged |
| `sHydrPump1Running` | Hydraulic pump 1 | Bit set = pump 1 running |
| `sHydrPump2Running` | Hydraulic pump 2 | Bit set = pump 2 running |
| `sLiftCylinderUpperLocked` | Retractable thruster upper lock | Bit set = locked in upper position. Ignore if vessel has no retractables. |
| `sLiftCylinderLowerLocked` | Retractable thruster lower lock | Bit set = locked in lower position. Ignore if vessel has no retractables. |

For `sLiftCylinderUpperLocked` and `sLiftCylinderLowerLocked`, bit weight depends on thruster number (`#1` = bit 0, `#2` = bit 1, ...):

| Thruster number | Bit weight |
|---|---:|
| #1 | 1 |
| #2 | 2 |
| #3 | 4 |
| #4 | 8 |
| #5 | 16 |
| #6 | 32 |
| #7 | 64 |
| #8 | 128 |
| #9 | 256 |
| #10 | 512 |

To calculate an expected value, add the bit weights of the retractable thrusters that currently have that lock active. Clear bits for thrusters that do not have the lock (or are not retractable).

Examples:

- Retractables `#3` and `#4`, both upper-locked -> Upper = `4 + 8 = 12`, Lower = `0`
- Retractables `#2` and `#4`, only `#2` lower-locked -> Lower = `2`, Upper = `0`
- Retractables `#2` and `#4`, both lower-locked -> Lower = `2 + 8 = 10`


Common thruster state logic for slot `i`:

1. Fault bit set -> Fault
2. Else running bit set -> Running
3. Else ready bit set -> Ready
4. Else -> Stopped / unavailable

---

## 4. Thruster command and feedback values

Per thruster slot `i` (`0..9`). These are the main live monitoring values.

| Datapoint | What it is | What the value means |
|---|---|---|
| `dThrusterSpeedReference[i]` | Commanded speed / RPM | Usually normalized about `-1` to `+1` (`±1` = full scale). Not raw RPM unless the vessel project converts it. |
| `dThrusterSpeedFeedback[i]` | Measured speed / RPM | Same scale as reference in normal configs. Compare to reference to see follow-up. |
| `dThrusterPitchReference[i]` | Commanded pitch | Usually normalized about `-1` to `+1`. Relevant for CPP / pitch-controlled thrusters. |
| `dThrusterPitchFeedback[i]` | Measured pitch | Same scale as pitch reference. |
| `dThrusterAngleReference[i]` | Commanded azimuth / rudder angle | Degrees. |
| `dThrusterAngleFeedback[i]` | Measured azimuth / rudder angle | Degrees. |
| `dThrusterLoad[i]` | Thruster load | Kilowatts (kW). A very large negative number usually means "no power feedback available". |
| `dThrustCommand[i]` | Calculated thrust demand | Usually relative to max thrust, about `-1` to `+1`. |
| `dThrustFeedback[i]` | Calculated thrust achieved | Usually relative to max thrust, about `-1` to `+1`. |

Reading tips:

- Pair each reference with its feedback on the same widget.
- Tunnel thrusters usually do not need angle displays.
- Rudders often do not need speed/pitch displays.
- If a vessel needs "% RPM" labels, confirm the project conversion. Do not assume `0.5` always means `50%` without project confirmation.

---

## 5. Drive program and control mode

Per thruster slot `i`.

| Datapoint | What it is | What the value means |
|---|---|---|
| `sDriveProgram[i]` | Active drive program | `0` Free running / transit, `1` Manoeuvre |
| `sThrusterControlMode[i]` | How speed and pitch are controlled | `0` Combinator mode, `1` Constant speed mode |

---

## 6. Voting and preferred TCS controller

Use these for redundancy / controller-health views.

| Datapoint | What it is | What the value means |
|---|---|---|
| `sTCVoteStatus[j]` | Vote result from thruster-controller reply `j` (`0..5`) | Bitmask. Common bits: `1`/`2`/`4` = TCSCC1/2/3 in use; `16`/`32`/`64` = vote error; `256`/`512`/`1024` = timeout. Values can combine. |
| `sCCPrefCC[k]` | Preferred TCSCC for controller group `k` (`0..2`) | Preference bitmask, not a device id: `0` none, `1` prefer TCSCC1, `2` prefer TCSCC2, `4` prefer TCSCC3 |
| `sCCVoteBuffer[k][n]` | Raw vote channel from TCSCC `k` | `n=0` surge, `1` sway, `2` yaw, `3` thruster/status, `4` spare. Advanced diagnostics; usually not first-page values. |

---

## 7. Software versions

Each entry is a short version text string.

| Datapoint | What it is | What the value means |
|---|---|---|
| `szOSFilesVersion[0]` | OS application version | Version text of `MTOS.exe` |
| `szOSFilesVersion[1]` | OS I/O library version | Version text of `MTOSIO.dll` |
| `szOSFilesVersion[2]` | Operator panel library version | Version text of `MTOPPanel.dll` |
| `szOSFilesVersion[3]` / `[4]` | Unused | Ignore |
| `szCCFilesVersion[0]` | DP controller version | Version text of `DPCC.exe` |
| `szCCFilesVersion[1]` | Controller I/O library version | Version text of `MTIO.dll` |
| `szCCFilesVersion[2]` | TCS controller version | Version text of `TCSCC.exe` |
| `szCCFilesVersion[3]` | Gateway version | Version text of `MTGateway.exe` |
| `szCCFilesVersion[4]` | Gateway I/O version | Version text of `MTGatewayIO.dll` |

Best placed on a system-info page, not the live thruster overview.

---

## 8. Network node health

Each datapoint is a bitmask. A set bit means that node currently reports network OK.

| Datapoint | What it is | What the value means |
|---|---|---|
| `lOSstatus` | Operating stations online | Bit for each OS device index that is OK |
| `sCCstatus` | Controllers online | Bits `0..2` = DPCC1-3; bits `3..5` = TCSCC/LTC1-3 |
| `lTCstatus` | Thruster cards / gateways online | Bit for each TC device index that is OK |
| `lLCstatus` | Lever cards online | Bit for each lever-card device index that is OK |
| `lTHRDEVstatus` | Thruster devices online | Bit for each thruster-device index that is OK |

Map bit positions to vessel device names using the vessel network configuration. Do not assume every vessel uses the same indexes.

---

## 9. Vessel-specific GUI extras

These are project-specific. Labels come from vessel `TCSView.ini` / GUI config, not from a global standard.

| Datapoint | What it is | What the value means |
|---|---|---|
| `lIndicators[i]` | Custom digital lamps for thruster `i` | 32-bit bitfield. Bit `n` = digital indicator channel `n` (`0..31`). Bit set = lamp on. Combined value is the sum of those bit weights. Lamp text is vessel-defined in `TCSView.ini`. |
| `sButtonInd[i][g]` | Custom button LED state | Thruster `i`, button group `g` (`0..23`). Bits `0..4` are the five buttons in that group. Bit set = indication on. |
| `sButtonDisable[i][g]` | Custom button unavailable state | Same packing as above. Bit set = button disabled / not available. |
| `dAnalogIndValue[i][a]` | Custom analog indication | Thruster `i`, analog slot `a` (`0..9`). Meaning/unit come from vessel GUI config. |

For `lIndicators[i]`, each value is one 32-bit number for thruster slot `i`. Channel `0` = bit 0, channel `1` = bit 1, ... channel `31` = bit 31:

| Channel | Bit | Bit weight |
|---:|---:|---:|
| 0 | 0 | 1 |
| 1 | 1 | 2 |
| 2 | 2 | 4 |
| 3 | 3 | 8 |
| 4 | 4 | 16 |
| 5 | 5 | 32 |
| 6 | 6 | 64 |
| 7 | 7 | 128 |
| 8 | 8 | 256 |
| 9 | 9 | 512 |
| 10 | 10 | 1024 |
| 11 | 11 | 2048 |
| 12 | 12 | 4096 |
| 13 | 13 | 8192 |
| 14 | 14 | 16384 |
| 15 | 15 | 32768 |
| 16 | 16 | 65536 |
| 17 | 17 | 131072 |
| 18 | 18 | 262144 |
| 19 | 19 | 524288 |
| 20 | 20 | 1048576 |
| 21 | 21 | 2097152 |
| 22 | 22 | 4194304 |
| 23 | 23 | 8388608 |
| 24 | 24 | 16777216 |
| 25 | 25 | 33554432 |
| 26 | 26 | 67108864 |
| 27 | 27 | 134217728 |
| 28 | 28 | 268435456 |
| 29 | 29 | 536870912 |
| 30 | 30 | 1073741824 |
| 31 | 31 | 2147483648 |

The payload stores this as a signed 32-bit `long`. If channel 31 is on, a signed display of the raw number is negative (`-2147483648` when only bit 31 is set). Decode with unsigned 32-bit masking when possible: channel `n` is on when `(value & bit_weight) != 0`.

To calculate an expected value, add the bit weights of the lamps that are currently on for that thruster. Unused or unconfigured channels stay `0`. Lamp names are not in the payload; map channel `n` for thruster `i` from that vessel's `TCSView.ini` / TCS GUI config.

Examples:

- No lamps on -> `0`
- Only channel 0 on -> `1`
- Channels 0 and 2 on -> `1 + 4 = 5`
- Channels 0, 1, and 4 on -> `1 + 2 + 16 = 19`
- Only channel 31 on -> unsigned `2147483648`, signed `-2147483648`

If the dashboard only needs standard thruster monitoring, you can skip this whole group.

---

## Quick value reference

### Thruster type (`sThrusterType`)

| Value | Meaning |
|---:|---|
| 1 | Tunnel |
| 2 | Azimuth |
| 3 | Main prop |
| 4 | Rudder |
| 5 | Combi |
| 6 | Voith |

### Command owner (`sActiveCmdOwner`)

| Value | Meaning |
|---:|---|
| 0 | None |
| 1 | DP |
| 2 | Lever |
| 3 | Autopilot |
| 4 | Service mode |
| 5 | GUI / local |
| 6 | External |

### Drive program (`sDriveProgram`)

| Value | Meaning |
|---:|---|
| 0 | Free running / transit |
| 1 | Manoeuvre |

### Control mode (`sThrusterControlMode`)

| Value | Meaning |
|---:|---|
| 0 | Combinator |
| 1 | Constant speed |

### Preferred controller (`sCCPrefCC`)

| Value | Meaning |
|---:|---|
| 0 | No preference |
| 1 | Prefer TCSCC1 |
| 2 | Prefer TCSCC2 |
| 4 | Prefer TCSCC3 |

### Vote result bits (`sTCVoteStatus`)

| Value | Meaning |
|---:|---|
| 1 | TCSCC1 in use |
| 2 | TCSCC2 in use |
| 4 | TCSCC3 in use |
| 16 | TCSCC1 vote error |
| 32 | TCSCC2 vote error |
| 64 | TCSCC3 vote error |
| 256 | TCSCC1 timeout |
| 512 | TCSCC2 timeout |
| 1024 | TCSCC3 timeout |

### Digital indicator bits (`lIndicators[i]`)

| Value | Meaning |
|---:|---|
| 1 | Channel 0 on |
| 2 | Channel 1 on |
| 4 | Channel 2 on |
| 8 | Channel 3 on |
| 16 | Channel 4 on |
| 32 | Channel 5 on |
| 64 | Channel 6 on |
| 128 | Channel 7 on |
| 256 | Channel 8 on |
| 512 | Channel 9 on |
| 1024 | Channel 10 on |
| 2048 | Channel 11 on |
| 4096 | Channel 12 on |
| 8192 | Channel 13 on |
| 16384 | Channel 14 on |
| 32768 | Channel 15 on |
| 65536 | Channel 16 on |
| 131072 | Channel 17 on |
| 262144 | Channel 18 on |
| 524288 | Channel 19 on |
| 1048576 | Channel 20 on |
| 2097152 | Channel 21 on |
| 4194304 | Channel 22 on |
| 8388608 | Channel 23 on |
| 16777216 | Channel 24 on |
| 33554432 | Channel 25 on |
| 67108864 | Channel 26 on |
| 134217728 | Channel 27 on |
| 268435456 | Channel 28 on |
| 536870912 | Channel 29 on |
| 1073741824 | Channel 30 on |
| 2147483648 | Channel 31 on (signed display `-2147483648`) |

Values can combine. Lamp labels for each channel come from vessel `TCSView.ini`.
