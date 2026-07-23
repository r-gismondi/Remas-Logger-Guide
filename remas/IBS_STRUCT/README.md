# IBS_STRUCT Guides

This folder contains the guide for MTIBS telegrams used to send IBS data to shore.

Start with `mtibs-tag-pattern-guide.md`.

## Research Sources

The guide was derived from eight local vessel GenSheet folders provided for research only. The source Excel files were not copied to GitHub.

| Vessel folder | List source | MTIBS rows | Tags | Unique tags | Bool tags | Real tags |
|---|---|---:|---:|---:|---:|---:|
| `280 Class - H270 Guyana Hero` | `Gmr100.xls` | 53 | 1613 | 1585 | 933 | 680 |
| `280 Class - H281 C-Installer` | `Gmr100.xls` | 64 | 1958 | 1908 | 1339 | 619 |
| `312 Class - H292 Sanibel Island` | `Gmr100.xls` | 88 | 2691 | 2619 | 1886 | 805 |
| `312 Class - H299 Wine Island` | `Gmr100.xls` | 81 | 2499 | 2447 | 1776 | 723 |
| `Anchor - H138 Bram Force` | `Gmr100PrilogSetup.xls` | 105 | 3312 | 3255 | 2237 | 1075 |
| `Anchor - H145 Bram Rio` | `Gmr100PrilogSetup.xls` | 89 | 2799 | 2736 | 1861 | 938 |
| `Windfarm - H338 ECO Edison` | `Gmr100.xls` | 85 | 2650 | 2574 | 1671 | 979 |
| `Windfarm - H341 ECO Liberty` | `Gmr100.xls` | 98 | 3043 | 2916 | 1934 | 1109 |

## Quick Rules

- The `List` tab defines the MTIBS telegram rows.
- ListDef labels look like `MTIBS1`, `MTIBS2`, and so on. The trailing digit is only the List sequence identifier.
- The telegram structure identifier is `$MTIBS` without that digit. Decode payload fields against the ordered tag list for the matching List row.
- Each List row normally carries up to 32 comma-separated data points.
- `T,B=<tag>` means the tag is sent as a boolean value, usually `0/1`.
- `T,R=<tag>` means the tag is sent as a real/analog value.
- Use the Hxxx detail workbooks (`Hxxx_COM`, `Hxxx_MAIN`, or `MT_Hxxx_Main`) to resolve `TagName`, `TaskName`, `Descr`, process section, alarm setup, and source references.