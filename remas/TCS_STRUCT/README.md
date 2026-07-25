# TCSLogger Manual Lookup

Use this index when integrating against the TCS diagnostic logger payload sent on the `TCS_STRUCT` network.

These guides are aimed at project engineers mapping datapoints into remote-monitoring dashboards.

| Manual file | Payload struct | Transport/channel | Frequency behavior | Notes |
|---|---|---|---|---|
| `tcs-logger-mtos-current.md` | `tcs_diagnostic_struct` | `TCS_STRUCT` UDP, or serial when `TCSLogger = 1` | 1 Hz | Current layout audited from DP `master` / `v1.7.1.8`. Includes dashboard-oriented field explanations. |

## How To Choose

1. Use `tcs-logger-mtos-current.md` for vessels running current MTOS with `TCSLogger` enabled.
2. Confirm `function.ini` (`TCSLogger`, `TCSLoggerOSNumber`) and `communication.ini` (`[TCS_STRUCT]`) match the vessel configuration.
3. If a vessel is on a very old TCS/MTOS branch, verify `Common\Definition\MTTCSDef.h` before assuming this 2080-byte layout.

## Audit

This folder currently documents one active layout. Unlike `DP_STRUCT`, manuals are not yet split by historical MTOS version ranges.
