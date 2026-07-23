# RemasLogger Payload Manual - MTOS v1.6.0.0 to v1.6.17.x

Applies to MTOS versions: `v1.6.0.0` through `v1.6.17.x`
Use this manual when the vessel MTOS version is in this range.
Payload: `remas_struct`
Transport: `DP_STRUCT` UDP, or serial when `RemasLogger = 1`
Frequency: 1 Hz / 2 Hz / 10 Hz depending on `RemasLogger` value in the vessel release
Superseded by: `remas-logger-mtos-v1.6.18-and-newer.md`

## Layout Summary

This range adds speed-sensor and Voith pitch arrays to the `v1.5.25` layout. It is still older than the current `v1.6.18+` layout.

Compared with `v1.6.18 and newer`, this range does not contain:

- `dGpsVruComp[2][MAX_NO_GPS]`
- `dGpsXYDist[2][MAX_NO_GPS]`
- `dExtCtrlSpdRef`, `dExtCtrlAccRef`, `bAutonomyMode`
- `trackingControllerData`
- `dRollStabilisationCommand`

## Fields Added In This Range

These fields are present after `dGeneratorFuel[MAX_NO_GENERATOR]` and before `i64TimeStamp`:

| Field | C type | Format / unit |
|---|---|---|
| `lSpeedSensor[2][MAX_NO_SPEED]` | `float[2][3]` | m/s, `[0]=along`, `[1]=athwart` |
| `dVoithLongitudalPitchCommand[MAX_NO_THRUSTER]` | `float[10]` | -1..1 |
| `dVoithTransversePitchCommand[MAX_NO_THRUSTER]` | `float[10]` | -1..1 |
| `dVoithLongitudalPitchFeedback[MAX_NO_THRUSTER]` | `float[10]` | -1..1 |
| `dVoithTransversePitchFeedback[MAX_NO_THRUSTER]` | `float[10]` | -1..1 |

## Field Order

The first fields match `remas-logger-mtos-v1.5.25-to-v1.5.29.md` through `dGeneratorFuel[MAX_NO_GENERATOR]`. The complete tail for this range is:

```text
bool bAllocCombinatorMode
bool bEngineInVariableMode[MAX_NO_MAIN_ENGINE]
bool bBatteryReady[MAX_NO_GENERATOR]
float dStateOfCharge[MAX_NO_GENERATOR]
float dRemainingEnergy[MAX_NO_GENERATOR]
float dRemainingTimeMin[MAX_NO_GENERATOR]
float dRemainingTimeMinWorstCase[MAX_NO_GENERATOR]
float dGeneratorFuel[MAX_NO_GENERATOR]
float lSpeedSensor[2][MAX_NO_SPEED]
float dVoithLongitudalPitchCommand[MAX_NO_THRUSTER]
float dVoithTransversePitchCommand[MAX_NO_THRUSTER]
float dVoithLongitudalPitchFeedback[MAX_NO_THRUSTER]
float dVoithTransversePitchFeedback[MAX_NO_THRUSTER]
SYSTEMTIME i64TimeStamp
short dwCRC
long dwBytePattern1
long dwBytePattern2
```

## Transport And Value Mappings

Use the transport rules and shared enum/value mappings in `remas-logger-mtos-v1.6.18-and-newer.md`, but apply only the field order documented here for binary decoding.
