# RemasLogger Breakpoint Audit

This audit maps source-history breakpoints to engineer-facing MTOS manual ranges. Commit hashes are included for traceability only; engineers should choose manuals by MTOS version using `README.md`.

## Confirmed Manual Breakpoints

| Breakpoint | First confirmed MTOS tag | Source evidence | Impact |
|---|---|---|---|
| Legacy Remas output before `DP_STRUCT` UDP | Before `v1.5.3.4` | `f048636b3` introduced serial Remas support in 2008; older send paths used serial / legacy Remas network concepts. | Historical manual only. Do not assume `DP_STRUCT` UDP for these versions. |
| UDP moved to `DP_STRUCT` | `v1.5.3.4` | `12c3ebd61` changes the Remas UDP send call to `Send(DP_STRUCT, NOTADD_SEND_IDENT, ...)`. | First confirmed `DP_STRUCT` UDP manual range. Payload is raw packed `remas_struct` bytes without `struct_ident`. |
| Extended power/battery/follow-target layout | `v1.5.25` | Tagged `v1.5.25` struct contains `bBatteryReady`, `dStateOfCharge`, `dRemainingEnergy`, `dRemainingTimeMin`, `dRemainingTimeMinWorstCase`, and `dGeneratorFuel`. | New payload tail fields; split from `v1.5.3.4` range. |
| Speed sensor and Voith layout | `v1.6.0.0/MTOS` | Tagged source contains `lSpeedSensor[2][MAX_NO_SPEED]` and Voith pitch command/feedback arrays before timestamp/checksum. | New payload tail fields; split from `v1.5.25` range. |
| Current active layout | `v1.6.18` | Tagged source contains current active tail fields including `dExtCtrlSpdRef`, `dExtCtrlAccRef`, `bAutonomyMode`, `trackingControllerData`, and `dRollStabilisationCommand`. | Latest manual range starts here. |

## Audited Non-Breakpoints

| Source history | Finding | Manual decision |
|---|---|---|
| `f3678ef05` / `6699d8ead` 10 Hz Remas branch history | The commits are visible on archived/feature branches but were not found as first-class MTOS release breakpoints in the current `master` release path. | Document frequency behavior where present, but do not split a release manual solely on this branch-only history. |
| `2b0f1b6a3` checksum semantic change | The commit was not reachable from current `master` or public release tags during this audit. | Keep `dwCRC` documented as checksum/raw in manuals; verify branch-specific vessels separately. |
| 2024-2025 playback/logging changes (`#4907`, `#5435`) | These touched `playback_struct`, logging setup, or binary logging paths. The active RemasLogger UDP send path still sends `remas_struct`. | Do not create separate RemasLogger UDP manuals unless the active `remas_struct` layout or send path changed. |
| `remas_struct2` and `playback_struct` definitions | Present in `MTStructDef.h`, but not sent by the active RemasLogger UDP path. | Include as related structs, not as primary manual selection rows. |

## Verification Pointers

Use these files for future audits:

- `Common/Definition/MTStructDef.h`: packed struct definitions and field order.
- `OS/MTOSView/MTOSDoc.cpp`: Remas snapshot population and send path.
- `OS/MTOSView/MTFunction.cpp` and `function.ini`: RemasLogger setting semantics.
- `Configuration.base.DP/Network/communication.ini`: `DP_STRUCT` network channel configuration.

Only add a new manual range when the active RemasLogger payload layout, transport/channel, frequency behavior, or field semantics change in a released MTOS version.
