# RemasLogger Payload Manual - MTOS v1.6.18 and newer

Applies to MTOS versions: `v1.6.18` and newer, including current `master` / `v1.7.1.8`.
Use this manual when the vessel MTOS version is `v1.6.18` or newer.
Payload: `remas_struct`
Transport: `DP_STRUCT` UDP, or serial when `RemasLogger = 1`
Frequency: 1 Hz / 2 Hz / 10 Hz depending on `RemasLogger` value
Superseded by: not superseded in the audited release history.

This document describes the binary payload sent when `function.ini` enables the Remas logger as UDP output from an OS, for example:

```ini
RemasLogger         = 2    # 1=Serial, 2=UDP 1 Hz, 3=UDP 2 Hz, 4=UDP 10 Hz
RemasLoggerOSNumber = 1    # Only this OS outputs Remas data
```

The active UDP output is one packed C/C++ structure: `remas_struct`.

Source references:

- `OS\MTOSView\MTOSDoc.cpp`: `CaptureRemasSnapshot()` fills `remas_struct`.
- `OS\MTOSView\MTOSDoc.cpp`: RemasLogger sends `remas_struct` on the `DP_STRUCT` network when `RemasLogger >= 2`.
- `Common\Definition\MTStructDef.h`: `remas_struct`, `remas_struct2`, `playback_struct`, and nested structs.
- `Common\Definition\MTThrusterPackDefinition.h`: thruster enums and limits.
- `Common\Definition\MTSensorPackDefinitions.h`: sensor enums and limits.
- `Common\Definition\MTModeDefinitions.h`: mode constants.
- `Common\Definition\MTUnitsDef.h`: OS/CC/TCSCC/network device IDs.
- `Common\Definition\MTTCSDef.h`: TCSCC/DPCC status struct.

## Source History

This manual is the latest confirmed active RemasLogger UDP layout. The release breakpoint is `v1.6.18`, where the active `remas_struct` contains the current tail fields `dExtCtrlSpdRef`, `dExtCtrlAccRef`, `bAutonomyMode`, `trackingControllerData`, and `dRollStabilisationCommand` before timestamp/checksum markers.

Audited breakpoints:

| MTOS version | Source change | Manual impact |
|---|---|---|
| `v1.5.3.4` | UDP Remas output changed to `DP_STRUCT` (`12c3ebd61`). | First confirmed `DP_STRUCT` UDP manual range. |
| `v1.5.25` | Battery, remaining-time, and generator-fuel fields are present. | New payload tail fields before timestamp/checksum. |
| `v1.6.0.0` | Speed sensor and Voith pitch arrays are present. | New payload tail fields before timestamp/checksum. |
| `v1.6.18` | Current active layout with track-controller and roll-stabilisation fields is present. | Latest manual range starts here. |

Notes from Git history that did not create a separate released manual range:

- `RemasLogger = 3/4` frequency modes were found in branch history for 2 Hz / 10 Hz handling, but the audited public release tags already expose the current `RemasLogger` values in this latest range.
- `#2273 Checksum of Config files added to Remas struct` changes `dwCRC` semantics in branch history, but the commit audited in this repository is not reachable from current `master` or a public release tag. Treat `dwCRC` as a raw/checksum field unless validating a vessel-specific branch.
- 2024-2025 playback/logging commits touched `playback_struct` and logging paths; only changes to the active `remas_struct` send path were used as manual breakpoints.
- The local working tree currently contains unreleased `remas_struct` edits in `Common\Definition\MTStructDef.h` such as `sDPClass` and warning/alarm fields. Those fields are not in released `HEAD` / `v1.7.1.8` source and are not assigned to an MTOS manual range here.

## Transport

`RemasLogger = 2` sends the raw `remas_struct` payload over UDP at 1 Hz. `RemasLogger = 3` sends at 2 Hz, and `RemasLogger = 4` sends at 10 Hz. The selected source OS is controlled by `RemasLoggerOSNumber`.

The configured network section is `DP_STRUCT`. In the base DP configuration it is multicast `239.192.239.21`, local `PORT = 10000`, `REMOTE_PORT = 10001`.

The send call uses `NOTADD_SEND_IDENT`, so the UDP payload is the packed struct bytes only, without the normal `struct_ident` application header.

## Structs Available

| Struct | Availability | Notes |
|---|---|---|
| `remas_struct` | Active RemasLogger UDP payload | Sent by `RemasLogger >= 2` on `DP_STRUCT`. Packed with `#pragma pack(push,1)`. |
| `track_controller_struct` | Nested in `remas_struct` | Field `trackingControllerData`. |
| `remas_struct2` | Defined, legacy/alternate layout | Uses wider `long`/`double` fields and is not sent by the current RemasLogger UDP path. |
| `playback_struct` | Defined for binary playback/logging | Similar to `remas_struct`, with additional playback/logging fields. Not the active RemasLogger UDP payload. |

## Important Constants

| Name | Value | Meaning |
|---|---:|---|
| `MAX_NO_PLAYBACK_NAV` | 15 | Navigator slots in Remas/playback data. |
| `MAX_NO_GPS` | 4 | GPS compensation/distance slots. |
| `MAX_NO_GYRO` | 5 | Gyro slots. |
| `MAX_NO_WIND` | 5 | Wind sensor slots. |
| `MAX_NO_VRU` | 5 | VRU slots. |
| `MAX_NO_SPEED` | 3 | Speed sensor slots. |
| `MAX_NO_ROT` | 3 | ROT sensor slots. |
| `MAX_NO_EXT_FORCE` | 2 | External force sensor slots. |
| `MAX_NO_THRUSTER` | 10 | Thruster slots. |
| `MAX_NO_GENERATOR` | 10 | Generator slots. |
| `MAX_NO_MAIN_ENGINE` | 4 | Main engine slots. |
| `MAX_NO_BUSES` | 8 | Bus slots. |

Unused sensor/thruster array entries are filled with status/type `-1` and value sentinels such as `FLT_MAX` or `DBL_MAX`.

## Value Encoding Conventions

Fields with `float`, `double`, and count formats are continuous numeric values in the units shown in the field table. They do not have a finite list of possible values unless the table or source comment gives a range, for example `-1..1`, `-100..100 %`, or `0..1`.

All `bool` fields are encoded as C++ `bool`: `false = 0`, `true = 1`.

Unused entries use these sentinels where populated by `CaptureRemasSnapshot()`:

| Sentinel | Meaning |
|---|---|
| `-1` | Unused or unavailable id/status/type slot. |
| `FLT_MAX` | Unused `float` slot. |
| `DBL_MAX` | Unused `double` slot. |

`dwBytePattern1` and `dwBytePattern2` are fixed markers:

| Field | Hex value | ASCII |
|---|---:|---|
| `dwBytePattern1` | `0x4D544D54` | `MTMT` |
| `dwBytePattern2` | `0x48415348` | `HASH` |

## Enum and Status Values

### `ThrStatus`

Defined in `MTThrusterPackDefinition.h`:

| Name | Value |
|---|---:|
| `UNAVAIL` | 0 |
| `RUNNING` | 1 |
| `READY` | 2 |
| `IN_USE` | 3 |
| `GEARREADY` | 4 |

### `ThrusterType`

Defined in `MTThrusterPackDefinition.h`:

| Name | Value |
|---|---:|
| `TUNNEL` | 1 |
| `AZIMUTH` | 2 |
| `MAINPROP` | 3 |
| `RUDDER` | 4 |
| `COMBI` | 5 |
| `VOITH` | 6 |

### `SensorStatus`

Defined in `MTSensorPackDefinitions.h`:

| Name | Value |
|---|---:|
| `SENS_UNAVAIL` | 0 |
| `SENS_READY` | 1 |
| `SENS_CALIB` | 2 |
| `SENS_ENABLED` | 3 |
| `SENS_IN_USE` | 4 |

### Target / Transponder Types

Defined in `MTSensorPackDefinitions.h`. Used by `lTargetType`.

| Name | Value |
|---|---:|
| `ADMINISTRATOR_TP` | 0 |
| `SSBL_TP` | 1 |
| `LBL_TP` | 2 |
| `ROV_TP` | 3 |
| `VESSEL_TP` | 4 |
| `NAVIGATOR_TP` | 5 |
| `FOLLOW_TP` | 6 |
| `INS_TP` | 7 |

### Main Modes

Defined in `MTModeDefinitions.h`:

| Name | Value |
|---|---:|
| `MANUAL_LEVERS` | 1 |
| `STANDBY` | 2 |
| `JOYSTICK` | 4 |
| `DP` | 8 |
| `TRANSIT` | 16 |

### Sub Modes

Defined in `MTModeDefinitions.h`:

| Name | Value |
|---|---:|
| `NO_SUB_MODE` | 0 |
| `RELAXED` | 1 |
| `FOLLOW_TARGET` | 2 |
| `FOLLOW_LINE` | 4 |
| `SPEED_FROM_LEVERS` | 8 |
| `SPEED_FROM_JOYSTICK` | 16 |
| `AUTO_SPEED` | 32 |
| `DP_TRACK` | 64 |
| `SEISMIC_TRACK` | 128 |
| `CABLE_TRACK` | 256 |
| `AUTOCROSS` | 512 |
| `AUTOSAIL` | 1024 |

### DP Class

Defined in `MTModeDefinitions.h`. This mapping is included for branches that expose DP class in Remas/playback data; released `v1.6.18+` RemasLogger UDP layouts audited here do not include `sDPClass` in `remas_struct`.

| Name | Value |
|---|---:|
| `DP_CLASS_OFF` | 0 |
| `DP_CLASS_2` | 1 |
| `DP_CLASS_3` | 2 |

### Follow Target Mode

Defined in `MTModeDefinitions.h`. Used by `lFTMode`.

| Name | Value |
|---|---:|
| `FOLLOW_PLATFORM` | 0 |
| `FOLLOW_ROV` | 1 |

### Device IDs

Defined in `MTUnitsDef.h`. These are the values used by `lOSInCmd`, `lCCInUse`, `lTCSCCInUse`, and the TCSCC/DPCC in-use fields.

| Group | Values |
|---|---|
| OS units | `OS1=1` through `OS25=25`. |
| DP CC units | `CC1=26`, `CC2=27`, `CC3=28`. |
| Thruster cards | `TC1=37` through `TC29=65`. |
| DP struct network id | `DP_STRUCT=90`. |
| Remas network id | `REMAS=94`. |
| TCSCC units | `TCSCC1=100`, `TCSCC2=101`, `TCSCC3=102`. |

For example, `lCCInUse = 26` means `CC1` is in use.

### Vote Status

Defined in `MTSensorPackDefinitions.h`.

| Name | Value |
|---|---:|
| `NOT_CONFIGURED` | 0 |
| `VOTE_OK` | 1 |
| `VOTE_ERROR` | 2 |
| `VOTE_TIME_OUT` | 3 |

### Preferred CC Bitmask

Used by `sCCPrefCC` / `thr_voting_struct.sPrimID`. It is a preference bitmask, not a device id.

| Value | Meaning |
|---:|---|
| `0` | No explicit preference. |
| `1` | Prefer first CC in the voting group: `CC1` or `TCSCC1`, depending on context. |
| `2` | Prefer second CC in the voting group: `CC2` or `TCSCC2`, depending on context. |
| `4` | Prefer third CC in the voting group: `CC3` or `TCSCC3`, depending on context. Also forced for CC3 in fire mode in DPCC voting. |

### Vote Result Bitmask

Used by `sTCVoteStatus` / `thr_voting_reply_struct.sVoteRes`.

| Bit/value | Meaning |
|---:|---|
| `1` | First CC in group is in use (`CC1` for DPCC voting, `TCSCC1` for TCSCC voting). |
| `2` | Second CC in group is in use (`CC2` or `TCSCC2`). |
| `4` | Third CC in group is in use (`CC3` or `TCSCC3`). |
| `16` | First CC in group has vote error / voted out. |
| `32` | Second CC in group has vote error / voted out. |
| `64` | Third CC in group has vote error / voted out. |
| `256` | First CC in group timed out. |
| `512` | Second CC in group timed out. |
| `1024` | Third CC in group timed out. |

Values can be combined. For example, `273 = 1 + 16 + 256` means first CC is selected, has error, and has timed out.

### Vote Buffer Values

Used by `sCCVoteBuffer[3][5]` / `thr_voting_struct.sVoteData`.

| Index | Meaning |
|---:|---|
| `0` | Surge command vote value. |
| `1` | Sway command vote value. |
| `2` | Yaw command vote value. |
| `3` | Thruster/status vote value. |
| `4` | Spare / sensor-count vote value, depending on voting source. |

DPCC voting buffers are generated as signed byte-sized ramp values and clamped to `-63..63` before being stored as `short`. Some TCSCC paths pass thruster voting ramp buffers as `short`; treat these as raw signed vote/ramp values unless the sender-specific TCSCC implementation is also decoded.

### TCSCC/DPCC Status Values

Used by `sDPCC_TCSCCInUse` and `sTCSCC_DPCCInUse`. These fields copy `tcsCC_dpCC_status_struct.lStatus[0]`:

| Field | Source value | Expected values |
|---|---|---|
| `sDPCC_TCSCCInUse[i]` | DPCC value received from TCSCC status messages | DP CC device id: `CC1=26`, `CC2=27`, or `CC3=28`; `0` before data is received. |
| `sTCSCC_DPCCInUse[i]` | TCSCC value received from DPCC status messages | TCSCC device id: `TCSCC1=100`, `TCSCC2=101`, or `TCSCC3=102`; `0` before data is received. |

### Config-Dependent IDs

These fields are runtime/configuration IDs and do not have a fixed global enum:

| Field | Value source |
|---|---|
| `lTargetID[]` | Local navigator target/transponder id. |
| `lFTTarget1`, `lFTTarget2`, `lFTTransponderID` | Follow-target target/transponder ids. |
| `lFTAdmin1`, `lFTAdmin2`, `lFTHPRAdminID` | Navigator/admin/HPR ids. |
| `lOpPointNumber` | Operator point number configured in the DP system. |

## `remas_struct`

`remas_struct` is defined with `#pragma pack(push,1)`. Consumers must use the same packing and the same C type sizes as the sender.

| Field | C type | Format / unit | Notes |
|---|---|---|---|
| `lMainMode` | `short` | enum/bit value | See Main Modes. |
| `lSubMode` | `short` | enum/bit value | See Sub Modes. |
| `dPositionDeviation[2]` | `float[2]` | m | Surge and sway position deviation. In transit track mode, `[0]=DTW`, `[1]=XTE`. |
| `dHeadingDeviation` | `float` | deg | Heading deviation. In transit track mode, set from CTS. |
| `dVesselPos[2]` | `double[2]` | deg | `[0]=latitude`, `[1]=longitude`. |
| `dHeading` | `float` | deg | Vessel heading. |
| `dSpeed[2]` | `float[2]` | m/s | `[0]=surge/along`, `[1]=sway/athwart`. |
| `dRotSpeed` | `float` | deg/min | Vessel rate of turn. |
| `NavigatorStatus[MAX_NO_PLAYBACK_NAV]` | `short[15]` | `SensorStatus` | Navigator/reference-system status. See SensorStatus. Unused slots are `-1`. |
| `bNavigatorNotAccepted[MAX_NO_PLAYBACK_NAV]` | `bool[15]` | boolean | Navigator not-accepted flags. |
| `lTargetID[MAX_NO_PLAYBACK_NAV]` | `short[15]` | id | Target/transponder id for local navigators. See Config-Dependent IDs. Unused slots are `-1`. |
| `lTargetType[MAX_NO_PLAYBACK_NAV]` | `short[15]` | enum/id | Target/transponder type. See Target / Transponder Types. Unused slots are `-1`. |
| `dNavigatorPosition[2][MAX_NO_PLAYBACK_NAV]` | `double[2][15]` | deg | `[0][i]=latitude`, `[1][i]=longitude`. Unused slots are `DBL_MAX`. |
| `dTargetPosition[2][MAX_NO_PLAYBACK_NAV]` | `double[2][15]` | deg | `[0][i]=target latitude`, `[1][i]=target longitude`. |
| `dGpsVruComp[2][MAX_NO_GPS]` | `double[2][4]` | m | GPS VRU compensation offsets `[x,y]`. |
| `dGpsXYDist[2][MAX_NO_GPS]` | `double[2][4]` | m | GPS movement distance from roll/pitch `[x,y]`. |
| `dGyroHeading[MAX_NO_GYRO]` | `float[5]` | deg | Gyro headings. Unused slots are `FLT_MAX`. |
| `lGyroStatus[MAX_NO_GYRO]` | `short[5]` | `SensorStatus` | Gyro status. See SensorStatus. Unused slots are `-1`. |
| `bGyroNotAccepted[MAX_NO_GYRO]` | `bool[5]` | boolean | Gyro not-accepted flags. |
| `dVRU[3][MAX_NO_VRU]` | `float[3][5]` | deg, deg, m | `[0]=roll`, `[1]=pitch`, `[2]=heave`. Unused slots are `FLT_MAX`. |
| `lVRUStatus[MAX_NO_VRU]` | `short[5]` | `SensorStatus` | VRU status. See SensorStatus. Unused slots are `-1`. |
| `dWind[2][MAX_NO_WIND]` | `float[2][5]` | m/s, deg | `[0]=wind speed`, `[1]=wind direction`. Unused slots are `FLT_MAX`. |
| `lWindStatus[MAX_NO_WIND]` | `short[5]` | `SensorStatus` | Wind sensor status. See SensorStatus. Unused slots are `-1`. |
| `lExtForceSensor[2][MAX_NO_EXT_FORCE]` | `float[2][2]` | t, deg | `[0]=external force`, `[1]=external force direction`. Unused slots are `FLT_MAX`. |
| `lRotSensor[MAX_NO_ROT]` | `float[3]` | deg/min | ROT sensor values. Unused slots are `FLT_MAX`. |
| `dWindDirection` | `float` | deg | Vessel mean wind direction. |
| `dWindSpeed` | `float` | m/s | Vessel mean wind speed. |
| `dCurrentDirection` | `float` | deg | Estimated current/external-force direction except wind. |
| `dCurrentSpeed` | `float` | m/s | Estimated current/external-force speed except wind. |
| `dCarrotPosition[2]` | `double[2]` | deg | `[0]=latitude`, `[1]=longitude`. |
| `dCarrotHeading` | `float` | deg | Carrot heading. |
| `dPositionSetpoint[2]` | `double[2]` | deg | `[0]=latitude`, `[1]=longitude` of carrot end position. |
| `dHeadingSetpoint` | `float` | deg | Heading setpoint / carrot end heading. |
| `lThrusterType[MAX_NO_THRUSTER]` | `short[10]` | `ThrusterType` | Thruster type. See ThrusterType. Unused slots are `-1`. |
| `lThrusterStatus[MAX_NO_THRUSTER]` | `short[10]` | `ThrStatus` | Thruster status. See ThrStatus. Unused slots are `-1`. |
| `dRPMCommand[MAX_NO_THRUSTER]` | `float[10]` | -1..1 | RPM command. Unused slots are `FLT_MAX`. |
| `dRPMFeedback[MAX_NO_THRUSTER]` | `float[10]` | -1..1 | RPM feedback. Unused slots are `FLT_MAX`. |
| `dPitchCommand[MAX_NO_THRUSTER]` | `float[10]` | -1..1 | Pitch command. Unused slots are `FLT_MAX`. |
| `dPitchFeedback[MAX_NO_THRUSTER]` | `float[10]` | -1..1 | Pitch feedback. Unused slots are `FLT_MAX`. |
| `dRelativeThrustCommand[MAX_NO_THRUSTER]` | `float[10]` | -100..100 % | Relative thrust command. Unused slots are `FLT_MAX`. |
| `dRelativeThrustFeedback[MAX_NO_THRUSTER]` | `float[10]` | -100..100 % | Relative thrust feedback. Unused slots are `FLT_MAX`. |
| `dAzimuthCommand[MAX_NO_THRUSTER]` | `float[10]` | deg | Azimuth command. Unused slots are `FLT_MAX`. |
| `dAzimuthFeedback[MAX_NO_THRUSTER]` | `float[10]` | deg | Azimuth feedback. Unused slots are `FLT_MAX`. |
| `dThrPowerFeedback[MAX_NO_THRUSTER]` | `float[10]` | kW | Thruster power feedback. |
| `bAxisControl[3]` | `bool[3]` | boolean | `[0]=along`, `[1]=athwart`, `[2]=heading` control enabled. |
| `dForceCmd[3]` | `float[3]` | t | Commanded vessel force `[surge,sway,yaw]`. |
| `ForceFdb[3]` | `float[3]` | t | Feedback force `[surge,sway,yaw]`. |
| `dDebugData[10]` | `float[10]` | raw | Debug values. |
| `sTCVoteStatus[6]` | `short[6]` | bitmask | Vote result from TC1-TC6. See Vote Result Bitmask. |
| `sDPCC_TCSCCInUse[3]` | `short[3]` | device id | DPCC in-use values reported by TCSCC1-TCSCC3. See TCSCC/DPCC Status Values. |
| `sTCSCC_DPCCInUse[3]` | `short[3]` | device id | TCSCC in-use values reported by CC1-CC3. See TCSCC/DPCC Status Values. |
| `sCCVoteBuffer[3][5]` | `short[3][5]` | raw vote values | DPCC vote buffers from CC1-CC3. See Vote Buffer Values. |
| `dCarrotVelocity[3]` | `float[3]` | m/s, m/s, deg/min | Carrot velocity `[surge,sway,yaw]`. |
| `dBallisticSpeed[3]` | `float[3]` | m/s, m/s, deg/min | Ballistic speed `[surge,sway,yaw]`. |
| `dControllerP[3]` | `float[3]` | t | Controller proportional force/moment. |
| `dControllerD[3]` | `float[3]` | t | Controller derivative force/moment. |
| `dControllerI[3]` | `float[3]` | t | Controller integral force/moment. |
| `dControllerWindFF[3]` | `float[3]` | t | Wind feed-forward force/moment. |
| `dControllerAccFF[3]` | `float[3]` | t | Acceleration feed-forward force/moment. |
| `dGeneratorLoad[MAX_NO_GENERATOR]` | `float[10]` | kW | Generator load. |
| `dDieselLoad[MAX_NO_MAIN_ENGINE]` | `float[4]` | kW | Main engine/diesel load. |
| `bGenBreakerStatus[MAX_NO_GENERATOR]` | `bool[10]` | boolean | Generator breaker status. |
| `bThrBreakerStatus[MAX_NO_THRUSTER]` | `bool[10]` | boolean | Thruster breaker status. |
| `bBusBreakerStatus[MAX_NO_BUSES]` | `bool[8]` | boolean | Bus breaker status. |
| `lNoOfGenConnected[MAX_NO_BUSES]` | `short[8]` | count | Number of generators connected per bus. |
| `lNoOfThrConnected[MAX_NO_BUSES]` | `short[8]` | count | Number of thrusters connected per bus. |
| `sCCPrefCC[3]` | `short[3]` | bitmask | Preferred CC values from CC1-CC3. See Preferred CC Bitmask. |
| `lCCInUse` | `short` | device id | CC in use. Usually `CC1=26`, `CC2=27`, or `CC3=28`; `0` if not set. |
| `lTCSCCInUse` | `short` | device id | TCSCC in use. Usually `TCSCC1=100`, `TCSCC2=101`, or `TCSCC3=102`; `0` if not set. |
| `lOSInCmd` | `short` | device id | OS in command. Usually `OS1=1` through `OS25=25`; `0` if not set. |
| `lEstimatorThrForce[3]` | `float[3]` | t | Estimator thruster force. |
| `lEstimatorBiasForce[3]` | `float[3]` | t | Estimator bias force. |
| `lEstimatorDragForce[3]` | `float[3]` | t | Estimator drag force. |
| `lEstimatorDampForce[3]` | `float[3]` | t | Estimator damping force. |
| `lEstimatorWindForce[3]` | `float[3]` | t | Estimator wind force. |
| `lEstimatorExtForce[3]` | `float[3]` | t | Estimator external force. |
| `lJoystickCmd[3]` | `float[3]` | -100..100 % | Joystick command on all axes. |
| `lOpPointNumber` | `short` | id | Operating point number. See Config-Dependent IDs. |
| `lOPPointPos[2]` | `float[2]` | m | Operating point position. |
| `dPIDGainSettings[3]` | `float[3]` | % | Position, velocity, integral gain set in MTOS. |
| `dAlarmLimit` | `float` | m | Position alarm limit. |
| `lFTMode` | `short` | enum/id | Follow-target mode. See Follow Target Mode. |
| `lFTTarget1` | `short` | id | Follow-target target 1. See Config-Dependent IDs. |
| `lFTTarget2` | `short` | id | Follow-target target 2. See Config-Dependent IDs. |
| `lFTAdmin1` | `short` | id | Follow-target administrator 1. See Config-Dependent IDs. |
| `lFTAdmin2` | `short` | id | Follow-target administrator 2. See Config-Dependent IDs. |
| `lFTTransponderID` | `short` | id | Follow-target transponder id. See Config-Dependent IDs. |
| `lFTHPRAdminID` | `short` | id | Follow-target HPR administrator id. See Config-Dependent IDs. |
| `bRelHdgEnabled` | `bool` | boolean | Relative heading enabled. |
| `bFTCircleEnabled` | `bool` | boolean | Follow-target circle enabled. |
| `dFTCircleRadius` | `float` | m | Follow-target circle radius. |
| `dFTCirclePos[2]` | `float[2]` | m | Follow-target circle position. |
| `bAllocCombinatorMode` | `bool` | boolean | Allocation combinator mode. |
| `bEngineInVariableMode[MAX_NO_MAIN_ENGINE]` | `bool[4]` | boolean | Main engine variable mode flags. |
| `bBatteryReady[MAX_NO_GENERATOR]` | `bool[10]` | boolean | Battery/generator ready flags. |
| `dStateOfCharge[MAX_NO_GENERATOR]` | `float[10]` | 0..1 | Battery state of charge. |
| `dRemainingEnergy[MAX_NO_GENERATOR]` | `float[10]` | kWh | Remaining energy. |
| `dRemainingTimeMin[MAX_NO_GENERATOR]` | `float[10]` | min | Remaining time. |
| `dRemainingTimeMinWorstCase[MAX_NO_GENERATOR]` | `float[10]` | min | Worst-case remaining time. |
| `dGeneratorFuel[MAX_NO_GENERATOR]` | `float[10]` | liter | Generator fuel. |
| `lSpeedSensor[2][MAX_NO_SPEED]` | `float[2][3]` | m/s | `[0]=along`, `[1]=athwart`. Unused slots are `FLT_MAX`. |
| `dVoithLongitudalPitchCommand[MAX_NO_THRUSTER]` | `float[10]` | -1..1 | Voith longitudinal pitch command. |
| `dVoithTransversePitchCommand[MAX_NO_THRUSTER]` | `float[10]` | -1..1 | Voith transverse pitch command. |
| `dVoithLongitudalPitchFeedback[MAX_NO_THRUSTER]` | `float[10]` | -1..1 | Voith longitudinal pitch feedback. |
| `dVoithTransversePitchFeedback[MAX_NO_THRUSTER]` | `float[10]` | -1..1 | Voith transverse pitch feedback. |
| `dExtCtrlSpdRef[3]` | `float[3]` | m/s | External control speed reference. |
| `dExtCtrlAccRef[3]` | `float[3]` | m/s^2 | External control acceleration reference. |
| `bAutonomyMode` | `bool` | boolean | Autonomy mode active. |
| `trackingControllerData` | `track_controller_struct` | nested struct | Track controller data; see below. |
| `dRollStabilisationCommand[MAX_NO_THRUSTER]` | `float[10]` | command | Roll stabilisation command per thruster. |
| `i64TimeStamp` | `SYSTEMTIME` | Windows `SYSTEMTIME` | Timestamp. |
| `dwCRC` | `short` | checksum/raw | CRC field. |
| `dwBytePattern1` | `long` | raw | Byte-pattern marker. |
| `dwBytePattern2` | `long` | raw | Byte-pattern marker. |

## `track_controller_struct`

Nested field in `remas_struct` as `trackingControllerData`.

| Field | C type | Format / unit | Notes |
|---|---|---|---|
| `dHeadingPMoment` | `float` | moment/force | Heading proportional moment. |
| `dHeadingIMoment` | `float` | moment/force | Heading integral moment. |
| `dHeadingDMoment` | `float` | moment/force | Heading derivative moment. |
| `dXTDPMoment` | `float` | moment/force | Cross-track proportional moment. |
| `dXTDIMoment` | `float` | moment/force | Cross-track integral moment. |
| `dXTDDMoment` | `float` | moment/force | Cross-track derivative moment. |
| `dYawRateFF` | `float` | feed-forward | Yaw-rate feed-forward. |
| `dTotalMoment` | `float` | moment/force | Total moment. |
| `dCrabAngle` | `float` | deg | Crab angle. |
| `bInTurn` | `bool` | boolean | In-turn flag. |
| `dCrabInTurn` | `float` | deg | Crab angle in turn. |
| `dTurnRadius` | `float` | m | Turn radius. |
| `dXTE_used` | `float` | m | Cross-track error used by controller. |
| `dXTE_raw` | `float` | m | Raw cross-track error. |
| `dXTE_reference` | `float` | m | Cross-track reference. |
| `dHeadingReference` | `float` | deg | Heading reference. |
| `dYawRateReference` | `float` | deg/min | Yaw-rate reference. |
| `dWheelOverDistance` | `float` | m | Wheel-over distance. |

## Legacy/Related Structs

### `remas_struct2`

`remas_struct2` is another packed Remas layout defined in `MTStructDef.h`. It uses `long` and `double` for many fields and contains a smaller field set. The current `RemasLogger` UDP send path does not instantiate or send this struct.

### `playback_struct`

`playback_struct` is a packed binary playback/logging layout. It starts with `SYSTEMTIME i64TimeStamp`, then largely follows the same field order and units as `remas_struct`, with additional playback-specific fields such as:

- `lRotStatus[MAX_NO_ROT]`
- `dPosCarrotVelocitySetpointFrame[2]`
- system filter values (`dReferenceSystemFilter`, `dHeadingSystemFilter`, `dWindSystemFilter`, `dVRUSystemFilter`)
- simulated thruster feedback arrays
- nested `Seismic_Track_Data_struct`, `speed_controller_struct`, and `ilos_track_controller_struct`
- network node status arrays

This struct is used by playback/binary logging code, not by the active RemasLogger UDP output.
