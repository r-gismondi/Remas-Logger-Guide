# RemasLogger Payload Manual - MTOS v1.5.3.4 to v1.5.24.x

Applies to MTOS versions: `v1.5.3.4` through `v1.5.24.x`
Use this manual when the vessel MTOS version is in this range.
Payload: `remas_struct`
Transport: `DP_STRUCT` UDP when `RemasLogger = 2`; serial when `RemasLogger = 1`
Frequency: 1 Hz in the audited tagged `v1.5.3.4` send path
Superseded by: `remas-logger-mtos-v1.5.25-to-v1.5.29.md`

## Transport

This is the first confirmed MTOS release range where RemasLogger UDP sends raw packed `remas_struct` bytes on the `DP_STRUCT` channel using `NOTADD_SEND_IDENT`.

For the usual OS1 UDP setup:

```ini
RemasLogger         = 2
RemasLoggerOSNumber = 1
```

The sender calls `Send(DP_STRUCT, NOTADD_SEND_IDENT, (long*)&sendData, sizeof(remas_struct), 1)`. Consumers must not expect the normal `struct_ident` application header.

## Layout Differences From Current

- Navigator arrays are fixed at `10` slots in the audited `v1.5.3.4` source, not `MAX_NO_PLAYBACK_NAV = 15`.
- Vessel, navigator, target, carrot, and setpoint positions are `float`, not `double`.
- VRU data is `dVRU[2][MAX_NO_VRU]` for roll and pitch only; current layouts use three rows including heave.
- This range does not contain the later DP/TCS in-use arrays, estimator arrays, follow-target fields, battery fields, speed-sensor fields, Voith fields, autonomy, track-controller data, or roll-stabilisation command.

## Field Order

| Field | C type | Format / unit |
|---|---|---|
| `lMainMode` | `short` | Main mode value |
| `lSubMode` | `short` | Sub mode value |
| `dPositionDeviation[2]` | `float[2]` | m |
| `dHeadingDeviation` | `float` | deg |
| `dVesselPos[2]` | `float[2]` | deg |
| `dHeading` | `float` | deg |
| `dSpeed[2]` | `float[2]` | m/s |
| `dRotSpeed` | `float` | deg/min |
| `NavigatorStatus[10]` | `short[10]` | `SensorStatus` |
| `bNavigatorNotAccepted[10]` | `bool[10]` | boolean |
| `lTargetID[10]` | `short[10]` | config-dependent id |
| `lTargetType[10]` | `short[10]` | target/transponder type |
| `dNavigatorPosition[2][10]` | `float[2][10]` | deg |
| `dTargetPosition[2][10]` | `float[2][10]` | deg |
| `dGyroHeading[MAX_NO_GYRO]` | `float[]` | deg |
| `lGyroStatus[MAX_NO_GYRO]` | `short[]` | `SensorStatus` |
| `bGyroNotAccepted[MAX_NO_GYRO]` | `bool[]` | boolean |
| `dVRU[2][MAX_NO_VRU]` | `float[2][]` | roll/pitch deg |
| `lVRUStatus[MAX_NO_VRU]` | `short[]` | `SensorStatus` |
| `dWind[2][MAX_NO_WIND]` | `float[2][]` | m/s, deg |
| `lWindStatus[MAX_NO_WIND]` | `short[]` | `SensorStatus` |
| `dWindDirection` | `float` | deg |
| `dWindSpeed` | `float` | m/s |
| `dCurrentDirection` | `float` | deg |
| `dCurrentSpeed` | `float` | m/s |
| `dCarrotPosition[2]` | `float[2]` | deg |
| `dCarrotHeading` | `float` | deg |
| `dPositionSetpoint[2]` | `float[2]` | deg |
| `dHeadingSetpoint` | `float` | deg |
| `lThrusterType[MAX_NO_THRUSTER]` | `short[]` | `ThrusterType` |
| `lThrusterStatus[MAX_NO_THRUSTER]` | `short[]` | `ThrStatus` |
| `dRPMCommand[MAX_NO_THRUSTER]` | `float[]` | -1..1 |
| `dRPMFeedback[MAX_NO_THRUSTER]` | `float[]` | -1..1 |
| `dPitchCommand[MAX_NO_THRUSTER]` | `float[]` | -1..1 |
| `dPitchFeedback[MAX_NO_THRUSTER]` | `float[]` | -1..1 |
| `dRelativeThrustCommand[MAX_NO_THRUSTER]` | `float[]` | -100..100 % |
| `dRelativeThrustFeedback[MAX_NO_THRUSTER]` | `float[]` | -100..100 % |
| `dAzimuthCommand[MAX_NO_THRUSTER]` | `float[]` | deg |
| `dAzimuthFeedback[MAX_NO_THRUSTER]` | `float[]` | deg |
| `dThrPowerFeedback[MAX_NO_THRUSTER]` | `float[]` | kW |
| `bAxisControl[3]` | `bool[3]` | boolean |
| `dForceCmd[3]` | `float[3]` | t |
| `ForceFdb[3]` | `float[3]` | t |
| `dDebugData[10]` | `float[10]` | raw |
| `sTCVoteStatus[6]` | `short[6]` | vote bitmask |
| `sCCPrefCC[3]` | `short[3]` | preferred CC bitmask |
| `sCCVoteBuffer[3][5]` | `short[3][5]` | raw vote values |
| `dCarrotVelocity[3]` | `float[3]` | m/s, m/s, deg/min |
| `dBallisticSpeed[3]` | `float[3]` | m/s, m/s, deg/min |
| `dControllerP[3]` | `float[3]` | t |
| `dControllerD[3]` | `float[3]` | t |
| `dControllerI[3]` | `float[3]` | t |
| `dControllerWindFF[3]` | `float[3]` | t |
| `dControllerAccFF[3]` | `float[3]` | t |
| `dGeneratorLoad[MAX_NO_GENERATOR]` | `float[]` | kW |
| `dDieselLoad[MAX_NO_MAIN_ENGINE]` | `float[]` | kW |
| `bGenBreakerStatus[MAX_NO_GENERATOR]` | `bool[]` | boolean |
| `bThrBreakerStatus[MAX_NO_THRUSTER]` | `bool[]` | boolean |
| `bBusBreakerStatus[MAX_NO_BUSES]` | `bool[]` | boolean |
| `lNoOfGenConnected[MAX_NO_BUSES]` | `short[]` | count |
| `lNoOfThrConnected[MAX_NO_BUSES]` | `short[]` | count |
| `lCCInUse` | `short` | device id, usually `CC1=26`..`CC3=28` |
| `lOSInCmd` | `short` | device id, usually `OS1=1`..`OS25=25` |
| `i64TimeStamp` | `SYSTEMTIME` | Windows timestamp |
| `dwCRC` | `short` | checksum/raw |
| `dwBytePattern1` | `long` | marker |
| `dwBytePattern2` | `long` | marker |

## Shared Value Tables

Use `remas-logger-mtos-v1.6.18-and-newer.md` for enum/value mappings. For this range, ignore mappings for fields that do not exist in the table above.
