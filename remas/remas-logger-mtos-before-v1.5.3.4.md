# RemasLogger Payload Manual - MTOS before v1.5.3.4

Applies to MTOS versions: older than `v1.5.3.4`
Use this manual when the vessel MTOS version predates the confirmed `DP_STRUCT` UDP release.
Payload: `remas_struct`
Transport: serial Remas output when `RemasLogger = 1`; older branch/source history may also contain legacy `REMAS` network paths.
Frequency: typically 1 Hz in the audited serial path.
Superseded by: `remas-logger-mtos-v1.5.3.4-to-v1.5.24.md`

## Important

This is a historical range. The exact payload layout can vary by archived vessel/project branch. For dashboard work on a vessel older than `v1.5.3.4`, verify the vessel source branch before assuming the later `DP_STRUCT` UDP layout.

The shared enum and value mappings are the same families documented in `remas-logger-mtos-v1.6.18-and-newer.md`: thruster status/type, sensor status, mode values, device IDs, voting values, boolean encoding, sentinels, and byte patterns.

## Transport

The first audited RemasLogger output support was serial Remas output (`f048636b3`, 2008). The confirmed `DP_STRUCT` UDP send path starts at `v1.5.3.4`; do not configure this historical range as UDP from OS1 unless the vessel branch is verified to contain that support.

## Structs Available

| Struct | Availability | Notes |
|---|---|---|
| `remas_struct` | Active historical RemasLogger payload | Packed structure. Field order must be read from the exact vessel branch for pre-`v1.5.3.4` vessels. |
| `remas_struct2` | Related/legacy definition | Not the primary active RemasLogger payload in the audited send path. |
| `playback_struct` | Playback/logging definition | Not the active RemasLogger output payload. |

## Source History

| Commit/date | Finding |
|---|---|
| `f048636b3` / 2008-05-13 | Adds support for sending Remas struct over serial. |
| `d2ed358e6` / 2008-11-19 | Changes analog sensor, binary logger, and `remas_struct`. |
| `12c3ebd61` / 2014-10-30 | Later change that moves confirmed UDP output to `DP_STRUCT`; this starts the next manual range. |

## Dashboard Mapping Notes

- Use this manual only as a warning/lookup entry. For any actual mapping, inspect `Common/Definition/MTStructDef.h` and `OS/MTOSView/MTOSDoc.cpp` at the vessel release branch.
- Do not assume double-precision latitude/longitude fields; the later `v1.5.3.4` layout still uses `float` for vessel, navigator, target, carrot, and setpoint positions.
- Do not assume `RemasLogger = 2/3/4` UDP frequency behavior unless the vessel branch contains that implementation.
