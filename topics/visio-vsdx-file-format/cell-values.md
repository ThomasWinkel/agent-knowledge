---
title: VSDX cell values vs COM results
description: Read when code turns a .vsdx <Cell V F U> into the text or value COM gives (ResultStr, FormulaU), or must tell empty, constant and inherited cells apart in the XML.
---

# VSDX cell values vs COM results

`<Cell N="Value" V="..." F="..." U="..."/>`: `V` = cached result at save, `F` = formula, `U` = unit. Measured 2026-11 with Visio Pro 2019 C2R x64, German region (de-DE): XML vs `Cell.ResultStr(visNoCast)` and `Cell.FormulaU` of the same saved file, 270,062 User Value/Prompt cells.

| XML | COM |
|---|---|
| `U="STR"`, no `F` | constant string: `ResultStr` = `V` exactly |
| User `Prompt`, no `U` (`V="1"`, `V="(none)"`) | string column: `ResultStr` = `V` exactly |
| `U="STR"`, `F="<formula>"` | string result: `ResultStr` = `V` exactly |
| `V=""`, no `F` | formula `""` |
| `F="No Formula"` | empty formula (`FormulaU` = `""`) |
| `F="Inh"` | formula inherited from the master; `V` is the local result |
| `U="BOOL"`, `V="0"`/`"1"` | `ResultStr` `FALSE`/`TRUE` |
| no `U`, `V="1"`, no `F` | constant `1`: `ResultStr` `1,0000` |
| no `U`, `V="1"`, `F="IF(...)"` | `ResultStr` `1,0000` |
| no `U`, `V="0"`, `F="SETF(...)"` | `ResultStr` `0` |
| `U="MM"`, `V="0.787..."` | `V` in inches; `ResultStr` `20,0000 mm` |

- Strings and Booleans: take `V` (BOOL -> `TRUE`/`FALSE`).
- Text-only columns carry no `U="STR"`: decide by the column, not by `U`. User `Prompt`: all 135,031 cells of the measured file equal `ResultStr` as `V`, also digits (`V="1"`). Other text columns (Shape Data `Label`/`Prompt`/`Format`): unverified.
- Numbers: the `ResultStr` text cannot be made from `V`: its format depends on the formula, which `V`/`U` do not show. Two formula cells without `U` read `1,0000` (`IF(...)`, `V="1"`) and `0` (`SETF(...)`, `V="0"`, not `0,0000`). Compare numbers as numbers (`V` is invariant culture, internal units = inches), or read their text by COM.
- No `F` attribute = the cell's formula is the constant in `V`. The XML does not keep the formula text of a constant with units (`20 mm` is `V="0.787..." U="MM"`); `FormulaU` would show `20 mm`.
- A cell missing from a row = inherited from the master row, or the default if no master row has it.
- `V` is the result at save time: a formula that depends on other shapes, the page or the document was not recalculated after that.
- Unverified: error results (`E` attribute: none in the measured file), Shape Data (`Property`) types, dates.
