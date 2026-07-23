# RemasLogger Payload Manual - MTOS v1.5.25 to v1.5.29.x

Applies to MTOS versions: `v1.5.25` through `v1.5.29.x`
Use this manual when the vessel MTOS version is in this range.
Payload: `remas_struct`
Transport: `DP_STRUCT` UDP, or serial when `RemasLogger = 1`
Frequency: 1 Hz / 2 Hz / 10 Hz depending on the exact release branch and `RemasLogger` support
Superseded by: `remas-logger-mtos-v1.6.0.0-to-v1.6.17.md`

## Layout Summary

This range keeps the same active `remas_struct` send path on `DP_STRUCT` UDP. Compared with `v1.5.3.4` through `v1.5.24.x`, the tagged `v1.5.25` layout adds DP/TCS in-use values, estimator values, joystick and operating-point values, follow-target values, allocation/combinator values, and battery/fuel values.

Compared with `v1.6.18 and newer`, this range does not contain:

- `dGpsVruComp[2][MAX_NO_GPS]`
- `dGpsXYDist[2][MAX_NO_GPS]`
- `lSpeedSensor[2][MAX_NO_SPEED]`
- Voith pitch command/feedback arrays
- `dExtCtrlSpdRef`, `dExtCtrlAccRef`, `bAutonomyMode`
- `trackingControllerData`
- `dRollStabilisationCommand`

## Field Order

Fields are packed in this order. Use the latest manual for units and enum/value mappings for the same field names.

```text
short lMainMode
short lSubMode
float dPositionDeviation[2]
float dHeadingDeviation
double dVesselPos[2]
float dHeading
float dSpeed[2]
float dRotSpeed
short NavigatorStatus[MAX_NO_PLAYBACK_NAV]
bool bNavigatorNotAccepted[MAX_NO_PLAYBACK_NAV]
short lTargetID[MAX_NO_PLAYBACK_NAV]
short lTargetType[MAX_NO_PLAYBACK_NAV]
double dNavigatorPosition[2][MAX_NO_PLAYBACK_NAV]
double dTargetPosition[2][MAX_NO_PLAYBACK_NAV]
float dGyroHeading[MAX_NO_GYRO]
short lGyroStatus[MAX_NO_GYRO]
bool bGyroNotAccepted[MAX_NO_GYRO]
float dVRU[3][MAX_NO_VRU]
short lVRUStatus[MAX_NO_VRU]
float dWind[2][MAX_NO_WIND]
short lWindStatus[MAX_NO_WIND]
float lExtForceSensor[2][MAX_NO_EXT_FORCE]
float lRotSensor[MAX_NO_ROT]
float dWindDirection
float dWindSpeed
float dCurrentDirection
float dCurrentSpeed
double dCarrotPosition[2]
float dCarrotHeading
double dPositionSetpoint[2]
float dHeadingSetpoint
short lThrusterType[MAX_NO_THRUSTER]
short lThrusterStatus[MAX_NO_THRUSTER]
float dRPMCommand[MAX_NO_THRUSTER]
float dRPMFeedback[MAX_NO_THRUSTER]
float dPitchCommand[MAX_NO_THRUSTER]
float dPitchFeedback[MAX_NO_THRUSTER]
float dRelativeThrustCommand[MAX_NO_THRUSTER]
float dRelativeThrustFeedback[MAX_NO_THRUSTER]
float dAzimuthCommand[MAX_NO_THRUSTER]
float dAzimuthFeedback[MAX_NO_THRUSTER]
float dThrPowerFeedback[MAX_NO_THRUSTER]
bool bAxisControl[3]
float dForceCmd[3]
float ForceFdb[3]
float dDebugData[10]
short sTCVoteStatus[6]
short sDPCC_TCSCCInUse[3]
short sTCSCC_DPCCInUse[3]
short sCCVoteBuffer[3][5]
float dCarrotVelocity[3]
float dBallisticSpeed[3]
float dControllerP[3]
float dControllerD[3]
float dControllerI[3]
float dControllerWindFF[3]
float dControllerAccFF[3]
float dGeneratorLoad[MAX_NO_GENERATOR]
float dDieselLoad[MAX_NO_MAIN_ENGINE]
bool bGenBreakerStatus[MAX_NO_GENERATOR]
bool bThrBreakerStatus[MAX_NO_THRUSTER]
bool bBusBreakerStatus[MAX_NO_BUSES]
short lNoOfGenConnected[MAX_NO_BUSES]
short lNoOfThrConnected[MAX_NO_BUSES]
short sCCPrefCC[3]
short lCCInUse
short lTCSCCInUse
short lOSInCmd
float lEstimatorThrForce[3]
float lEstimatorBiasForce[3]
float lEstimatorDragForce[3]
float lEstimatorDampForce[3]
float lEstimatorWindForce[3]
float lEstimatorExtForce[3]
float lJoystickCmd[3]
short lOpPointNumber
float lOPPointPos[2]
float dPIDGainSettings[3]
float dAlarmLimit
short lFTMode
short lFTTarget1
short lFTTarget2
short lFTAdmin1
short lFTAdmin2
short lFTTransponderID
short lFTHPRAdminID
bool bRelHdgEnabled
bool bFTCircleEnabled
float dFTCircleRadius
float dFTCirclePos[2]
bool bAllocCombinatorMode
bool bEngineInVariableMode[MAX_NO_MAIN_ENGINE]
bool bBatteryReady[MAX_NO_GENERATOR]
float dStateOfCharge[MAX_NO_GENERATOR]
float dRemainingEnergy[MAX_NO_GENERATOR]
float dRemainingTimeMin[MAX_NO_GENERATOR]
float dRemainingTimeMinWorstCase[MAX_NO_GENERATOR]
float dGeneratorFuel[MAX_NO_GENERATOR]
SYSTEMTIME i64TimeStamp
short dwCRC
long dwBytePattern1
long dwBytePattern2
```

## Shared Value Tables

Use `remas-logger-mtos-v1.6.18-and-newer.md` for shared enum/value mappings. Ignore mappings for fields absent from this range.
