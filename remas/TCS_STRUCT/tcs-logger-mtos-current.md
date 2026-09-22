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

`lIndicators[i]` is the live on/off state of the **NOTIFICATIONS** lamps on the TCS page for thruster slot `i`. Those lamps are configured in `TCSView.ini` as `IND#Setup` (`IND1Setup` ... `IND32Setup`). The payload carries only the lamp bits, not the text.

`sButtonInd[i][g]` is the live LED state of the custom **button groups** on that same TCS page (Start / Stop / Reset and other project buttons). Those buttons are configured in `TCSView.ini` as `BTN#Setup` (`BTN1Setup` ... `BTN120Setup`). `sButtonDisable[i][g]` uses the same packing: bit set means that button is disabled / not available. Standard command widgets such as Bridge / DP/JS / Autopilot / Manual are not this datapoint.

| Datapoint | What it is | What the value means |
|---|---|---|
| `lIndicators[i]` | NOTIFICATIONS lamps for thruster `i` | 32-bit bitfield of `IND#Setup` lamps. `IND1Setup` = bit 0, `IND2Setup` = bit 1, ... `IND32Setup` = bit 31. Bit set = lamp on. Combined value is the sum of those bit weights. Lamp text comes from that `IND#Setup` entry. |
| `sButtonInd[i][g]` | Custom button LED state for thruster `i`, group `g` | 5-bit field. Group `g` (`0..23`) holds five `BTN#Setup` buttons. Bit `0` = first button in the group, bit `4` = fifth. Bit set = LED on. Combined value is `0..31`. |
| `sButtonDisable[i][g]` | Custom button unavailable state | Same packing as `sButtonInd[i][g]`. Bit set = button disabled / not available. |
| `dAnalogIndValue[i][a]` | Custom analog indication | Thruster `i`, analog slot `a` (`0..9`). Meaning/unit come from vessel GUI config. |

For `lIndicators[i]`, each value is one 32-bit number for thruster slot `i`. `TCSView.ini` keys are 1-based (`IND1Setup` is channel 0 / bit 0):

| Channel | `TCSView.ini` key | Bit | Bit weight |
|---:|---|---:|---:|
| 0 | `IND1Setup` | 0 | 1 |
| 1 | `IND2Setup` | 1 | 2 |
| 2 | `IND3Setup` | 2 | 4 |
| 3 | `IND4Setup` | 3 | 8 |
| 4 | `IND5Setup` | 4 | 16 |
| 5 | `IND6Setup` | 5 | 32 |
| 6 | `IND7Setup` | 6 | 64 |
| 7 | `IND8Setup` | 7 | 128 |
| 8 | `IND9Setup` | 8 | 256 |
| 9 | `IND10Setup` | 9 | 512 |
| 10 | `IND11Setup` | 10 | 1024 |
| 11 | `IND12Setup` | 11 | 2048 |
| 12 | `IND13Setup` | 12 | 4096 |
| 13 | `IND14Setup` | 13 | 8192 |
| 14 | `IND15Setup` | 14 | 16384 |
| 15 | `IND16Setup` | 15 | 32768 |
| 16 | `IND17Setup` | 16 | 65536 |
| 17 | `IND18Setup` | 17 | 131072 |
| 18 | `IND19Setup` | 18 | 262144 |
| 19 | `IND20Setup` | 19 | 524288 |
| 20 | `IND21Setup` | 20 | 1048576 |
| 21 | `IND22Setup` | 21 | 2097152 |
| 22 | `IND23Setup` | 22 | 4194304 |
| 23 | `IND24Setup` | 23 | 8388608 |
| 24 | `IND25Setup` | 24 | 16777216 |
| 25 | `IND26Setup` | 25 | 33554432 |
| 26 | `IND27Setup` | 26 | 67108864 |
| 27 | `IND28Setup` | 27 | 134217728 |
| 28 | `IND29Setup` | 28 | 268435456 |
| 29 | `IND30Setup` | 29 | 536870912 |
| 30 | `IND31Setup` | 30 | 1073741824 |
| 31 | `IND32Setup` | 31 | 2147483648 |

The payload stores this as a signed 32-bit `long`. If channel 31 is on, a signed display of the raw number is negative (`-2147483648` when only bit 31 is set). Decode with unsigned 32-bit masking when possible: channel `n` is on when `(value & bit_weight) != 0`.

To calculate an expected value, add the bit weights of the NOTIFICATIONS lamps that are currently on for that thruster. Unused or unconfigured `IND#Setup` channels stay `0`. Read the lamp text from that thruster's `IND#Setup` entries in `TCSView.ini`.

This is not the same datapoint as `sThrusterRunning`. A TCS page can show a "THRUSTER RUNNING" notification lamp via `IND#Setup` while `sThrusterRunning` is the standard per-thruster running bitfield.

Examples:

- No lamps on -> `0`
- Only `IND1Setup` on -> `1`
- `IND1Setup` and `IND3Setup` on -> `1 + 4 = 5`
- `IND1Setup`, `IND2Setup`, and `IND5Setup` on -> `1 + 2 + 16 = 19`
- Only `IND32Setup` on -> unsigned `2147483648`, signed `-2147483648`
- Example TCS page with `IND1Setup` = THRUSTER RUNNING (on) and `IND2Setup` = BACKUP IN COMMAND (off) -> `1`

For `sButtonInd[i][g]`, each value is one 5-bit number for button group `g` on thruster slot `i`. There are 24 groups (`g = 0..23`) and five buttons per group. `TCSView.ini` keys are 1-based and run in group order: group `0` is `BTN1Setup`..`BTN5Setup`, group `1` is `BTN6Setup`..`BTN10Setup`, and so on through group `23` = `BTN116Setup`..`BTN120Setup`.

Bit weights inside one group:

| Button in group | Bit | Bit weight | `TCSView.ini` key |
|---|---:|---:|---|
| 1st | 0 | 1 | `BTN{g*5+1}Setup` |
| 2nd | 1 | 2 | `BTN{g*5+2}Setup` |
| 3rd | 2 | 4 | `BTN{g*5+3}Setup` |
| 4th | 3 | 8 | `BTN{g*5+4}Setup` |
| 5th | 4 | 16 | `BTN{g*5+5}Setup` |

The payload stores each group as a `short`. Only bits `0..4` are used, so each `sButtonInd[i][g]` value is `0..31`. Channel `n` in that group is on when `(value & bit_weight) != 0`. Unused buttons in a group stay `0`. Button labels come from that `BTN#Setup` entry. Group titles on the TCS page (for example RESET DRIVE, THRUSTER) come from the vessel GUI layout, not from this bitfield.

Group-to-key map:

| Group `g` | `sButtonInd[i][g]` | `TCSView.ini` keys |
|---:|---|---|
| 0 | group 0 | `BTN1Setup` .. `BTN5Setup` |
| 1 | group 1 | `BTN6Setup` .. `BTN10Setup` |
| 2 | group 2 | `BTN11Setup` .. `BTN15Setup` |
| 3 | group 3 | `BTN16Setup` .. `BTN20Setup` |
| 4 | group 4 | `BTN21Setup` .. `BTN25Setup` |
| 5 | group 5 | `BTN26Setup` .. `BTN30Setup` |
| 6 | group 6 | `BTN31Setup` .. `BTN35Setup` |
| 7 | group 7 | `BTN36Setup` .. `BTN40Setup` |
| 8 | group 8 | `BTN41Setup` .. `BTN45Setup` |
| 9 | group 9 | `BTN46Setup` .. `BTN50Setup` |
| 10 | group 10 | `BTN51Setup` .. `BTN55Setup` |
| 11 | group 11 | `BTN56Setup` .. `BTN60Setup` |
| 12 | group 12 | `BTN61Setup` .. `BTN65Setup` |
| 13 | group 13 | `BTN66Setup` .. `BTN70Setup` |
| 14 | group 14 | `BTN71Setup` .. `BTN75Setup` |
| 15 | group 15 | `BTN76Setup` .. `BTN80Setup` |
| 16 | group 16 | `BTN81Setup` .. `BTN85Setup` |
| 17 | group 17 | `BTN86Setup` .. `BTN90Setup` |
| 18 | group 18 | `BTN91Setup` .. `BTN95Setup` |
| 19 | group 19 | `BTN96Setup` .. `BTN100Setup` |
| 20 | group 20 | `BTN101Setup` .. `BTN105Setup` |
| 21 | group 21 | `BTN106Setup` .. `BTN110Setup` |
| 22 | group 22 | `BTN111Setup` .. `BTN115Setup` |
| 23 | group 23 | `BTN116Setup` .. `BTN120Setup` |

`sButtonDisable[i][g]` uses this same table. A set bit means that `BTN#Setup` button is disabled / not available, even if `sButtonInd` is also set.

Examples:

- No button LEDs on in group `g` -> `sButtonInd[i][g] = 0`
- Only the first button LED on -> `1`
- First and second on -> `1 + 2 = 3`
- First and third on -> `1 + 4 = 5`
- All five on -> `1 + 2 + 4 + 8 + 16 = 31`
- Example TCS page: RESET DRIVE is group `0` with RESET as `BTN1Setup` (off), THRUSTER is group `1` with START as `BTN6Setup` (on) and STOP as `BTN7Setup` (off) -> `sButtonInd[i][0] = 0`, `sButtonInd[i][1] = 1`

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

NOTIFICATIONS lamps from `TCSView.ini` `IND#Setup`.

| Value | Meaning |
|---:|---|
| 1 | `IND1Setup` on |
| 2 | `IND2Setup` on |
| 4 | `IND3Setup` on |
| 8 | `IND4Setup` on |
| 16 | `IND5Setup` on |
| 32 | `IND6Setup` on |
| 64 | `IND7Setup` on |
| 128 | `IND8Setup` on |
| 256 | `IND9Setup` on |
| 512 | `IND10Setup` on |
| 1024 | `IND11Setup` on |
| 2048 | `IND12Setup` on |
| 4096 | `IND13Setup` on |
| 8192 | `IND14Setup` on |
| 16384 | `IND15Setup` on |
| 32768 | `IND16Setup` on |
| 65536 | `IND17Setup` on |
| 131072 | `IND18Setup` on |
| 262144 | `IND19Setup` on |
| 524288 | `IND20Setup` on |
| 1048576 | `IND21Setup` on |
| 2097152 | `IND22Setup` on |
| 4194304 | `IND23Setup` on |
| 8388608 | `IND24Setup` on |
| 16777216 | `IND25Setup` on |
| 33554432 | `IND26Setup` on |
| 67108864 | `IND27Setup` on |
| 134217728 | `IND28Setup` on |
| 268435456 | `IND29Setup` on |
| 536870912 | `IND30Setup` on |
| 1073741824 | `IND31Setup` on |
| 2147483648 | `IND32Setup` on (signed display `-2147483648`) |

Values can combine. Lamp labels for each `IND#Setup` come from vessel `TCSView.ini`.

### Button LED bits (`sButtonInd[i][g]`)

Custom TCS page button LEDs from `TCSView.ini` `BTN#Setup`. One value per group `g` (`0..23`).

| Value | Meaning in that group |
|---:|---|
| 1 | 1st button on (`BTN{g*5+1}Setup`) |
| 2 | 2nd button on (`BTN{g*5+2}Setup`) |
| 4 | 3rd button on (`BTN{g*5+3}Setup`) |
| 8 | 4th button on (`BTN{g*5+4}Setup`) |
| 16 | 5th button on (`BTN{g*5+5}Setup`) |

Values can combine (`0..31`). `sButtonDisable[i][g]` uses the same bits for disabled / not available. Button labels come from vessel `TCSView.ini`.
