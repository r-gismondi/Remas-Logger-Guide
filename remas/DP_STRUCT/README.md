# RemasLogger Manual Lookup

Use this index when the only known input is the MTOS software version installed on a vessel. Pick the row whose MTOS version range contains the vessel version, then use the linked manual for the RemasLogger payload layout, units, and value mappings.

| MTOS version range | Manual file | Payload struct | Transport/channel | Frequency behavior | Notes |
|---|---|---|---|---|---|
| Before `v1.5.3.4` | `remas-logger-mtos-before-v1.5.3.4.md` | `remas_struct` | Serial / legacy `REMAS` paths | 1 Hz serial when `RemasLogger = 1` | Historical range before the confirmed `DP_STRUCT` UDP change. Verify vessel branch before building new integrations. |
| `v1.5.3.4` through `v1.5.24.x` | `remas-logger-mtos-v1.5.3.4-to-v1.5.24.md` | `remas_struct` | `DP_STRUCT` UDP | 1 Hz UDP when `RemasLogger = 2` in audited tagged source | First confirmed `DP_STRUCT` UDP payload range. Uses 10 navigator slots and float latitude/longitude fields. |
| `v1.5.25` through `v1.5.29.x` | `remas-logger-mtos-v1.5.25-to-v1.5.29.md` | `remas_struct` | `DP_STRUCT` UDP | 1 Hz / 2 Hz / 10 Hz depending on source branch/config support | Adds DP/TCS voting, estimator, joystick, follow-target, battery, remaining-energy, and generator-fuel fields. |
| `v1.6.0.0` through `v1.6.17.x` | `remas-logger-mtos-v1.6.0.0-to-v1.6.17.md` | `remas_struct` | `DP_STRUCT` UDP | 1 Hz / 2 Hz / 10 Hz depending on `RemasLogger` value | Adds speed-sensor and Voith pitch arrays before timestamp/checksum. |
| `v1.6.18` and newer | `remas-logger-mtos-v1.6.18-and-newer.md` | `remas_struct` | `DP_STRUCT` UDP | 1 Hz / 2 Hz / 10 Hz depending on `RemasLogger` value | Current confirmed active layout. Adds GPS VRU compensation/distance, external control references, autonomy, track-controller data, and roll-stabilisation command. |

## How To Choose

1. Normalize the vessel MTOS version to the release number, for example `v1.6.29.3` belongs to the `v1.6.18 and newer` range.
2. If the vessel uses a project tag such as `projects/<vessel>/v1.6.24.9901`, use the version number inside that tag and then check the manual notes for project-specific caveats.
3. If the vessel version is older than `v1.5.3.4` or comes from an archived branch, verify the exact source branch before assuming UDP `DP_STRUCT`.

## Shared Mappings

The latest manual, `remas-logger-mtos-v1.6.18-and-newer.md`, contains the shared enum/value tables for thruster status/type, sensor status, main and sub modes, DP class, follow-target mode, device IDs, voting values, sentinels, and byte patterns. Older manuals list the range-specific field order and call out fields that differ from the latest layout.

## Audit

The breakpoint audit is captured in `breakpoint-audit.md`. Manual ranges are split only where Git history and release tags showed a change to the active RemasLogger payload layout, transport/channel, frequency behavior, or field meaning.
