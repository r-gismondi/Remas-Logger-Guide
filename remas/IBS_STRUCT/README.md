# IBS_STRUCT Guides

This folder contains guides for MTIBS telegrams used to send IBS data to shore.

- Start with `mtibs-tag-pattern-guide.md` for MT in-house vessel GenSheets.
- Use `mtibs-tag-pattern-guide-HoglundProjects.md` for Polish HMA / Hoglund third-party projects.

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
- Each row normally starts with `MTIBS<n>` and carries up to 32 comma-separated data points.
- `T,B=<tag>` means the tag is sent as a boolean value, usually `0/1`.
- `T,R=<tag>` means the tag is sent as a real/analog value.
- Use the Hxxx detail workbooks (`Hxxx_COM`, `Hxxx_MAIN`, or `MT_Hxxx_Main`) to resolve `TagName`, `TaskName`, `Descr`, process section, alarm setup, and source references.


## Hoglund / Polish HMA Research Source

| Vessel folder | List source | MTIBS rows | Tags | Unique tags | Bool tags | Real tags |
|---|---|---:|---:|---:|---:|---:|
| `Polish HMA Projects` (Remontowa B851) | `Gmr100.xls` | 69 | 2167 | 2167 | 1751 | 416 |

Quick differences vs MT in-house tags:

- Prefer short segments such as `441_1_1_LS1` and valve IDs such as `512_1_02_03_ZSH`.
- Tank objects often use A-codes such as `512_1_A044_LT`.
- Detail workbooks are project-prefixed (`RemontowaB851_4_MAIN.xls`) with the same sheet families (`AIS`, `DIS`, `MOT`, `BVAL`, `TK`).
