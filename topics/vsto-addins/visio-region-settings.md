---
title: Visio ShapeSheet values and Windows region settings
description: Read when Visio code builds Shape Data lists (Prop.Format), uses INDEX/LOOKUP/LISTSEP or list formulas, or reads numbers with ResultStr - list separator, decimal symbol, INDEX edge cases.
---

# Visio ShapeSheet values and Windows region settings

The Windows region settings of the user (`HKCU\Control Panel\International`: `sList` list separator, `sDecimal` decimal symbol) change some ShapeSheet results, not all. Measured with Visio Pro 2019 C2R x64, English UI, region de-DE (`;` / `,`) and the same with `,` / `.` (en-US style).

## Lists: always `;`, never `LISTSEP()`

| Formula / cell | `sList = ;` | `sList = ,` |
|---|---|---|
| `Prop.x.Format = "A;B;C"` (Type 1, fixed list) | 3 items A, B, C | 3 items A, B, C |
| `Prop.x.Format = "A,B,C"` | 1 item `A,B,C` | 1 item `A,B,C` |
| `INDEX(1,"A;B;C")` | `B` | `B` |
| `INDEX(1,"A;B;C",LISTSEP())` | `B` | empty |
| `INDEX(1,"A,B,C",LISTSEP())` | empty | `B` |
| `LOOKUP("B","A;B;C")` | 1 | 1 |
| `LISTSEP()` | `;` | `,` |

- The Shape Data window splits `Prop.Format` at `;` in every region.
- `INDEX`/`LOOKUP` without delimiter use `;` in every region.
- `LISTSEP()` returns the region's separator; with a fixed `;` list it **breaks** the formula in regions with another separator. Do not use it for lists stored in the ShapeSheet.
- The selected value (`Prop.x.Value`) is the item text, without separator.
- _Unverified:_ other Visio versions and UI languages.

## Lists in formulas: edge cases

Measured with Visio 2019 C2R x64 (region de-DE):

| Formula | Result |
|---|---|
| `INDEX(0,"S1;X1;A2",";")` / `INDEX(2,...)` | `S1` / `A2` (0-based) |
| `INDEX(3,"S1;X1;A2",";")`, `INDEX(-1,...)` | empty, no error |
| `INDEX(3,"S1;X1;A2",";","none")`, `INDEX(-1,...,"none")` | `none` (4th argument = value when out of range) |
| `INDEX(0,"",";","none")` | empty, not `none`: an empty list holds one empty item |
| `INDEX(1,"S1;;A2",";")` | empty: empty items count |
| `LOOKUP("X1","S1;X1;A2",";")` / `LOOKUP("Q",...)` | `1` / `-1` |
| `LEN(L)-LEN(SUBSTITUTE(L,";",""))+1` | item count, but `1` for `L = ""`: wrap in `IF(LEN(L)=0,0,...)` |
| `"+"&SUBSTITUTE("S1;X1",";","+")` | `+S1+X1` (all occurrences without a 4th argument) |
| `STRSAME("s1","S1")` / `STRSAME("s1","S1",TRUE)` | `FALSE` / `TRUE` (case-sensitive unless the 3rd argument is `TRUE`) |

- Cut a known head `H` off a list `L`: `IF(STRSAME(L,H),"",IF(STRSAME(LEFT(L,LEN(H)+1),H&";"),MID(L,LEN(H)+2,999),L))`. Compare with `H&";"`, not `H` alone (`S1` must not match `S10;X1`), and handle `L = H` on its own.

## Numbers: `ResultStr` follows the region

`User.x = 12.5` gives `ResultStr(visNoCast)` = `12,5000` (de-DE) and `12.5000` (en-US style). Read numbers with `Result(visNumber)` / `Result("mm")` / `ResultIU`, write them with `FormulaU` and `ToString(CultureInfo.InvariantCulture)`.

## Test recipe

- Change the separators with `SetLocaleInfo(LOCALE_USER_DEFAULT, LOCALE_SLIST / LOCALE_SDECIMAL, ...)` plus a `WM_SETTINGCHANGE "intl"` broadcast (not a raw registry write), back them up first, restart Visio, restore afterwards.
- No API lists the items of a Shape Data list. UI route: show the Shape Data window (`ActiveWindow.Windows.ItemFromID(1658).Visible = true`), get its owner-drawn `ListBox` (no UIA children) by UI Automation, click the right part of a row (`LB_GETITEMRECT`), then a new `REComboBox20W` window appears; read it with `CB_GETCOUNT` / `CB_GETLBTEXT`. Escape ends the edit.
