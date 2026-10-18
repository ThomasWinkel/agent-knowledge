---
title: Typed values in Visio ShapeSheet cells
description: Read when Visio code writes or reads text, numbers, booleans or dates in ShapeSheet cells, or writes cells protected with GUARD - formulas, results, what survives save.
---

# Typed values in Visio ShapeSheet cells

Write a literal with `FormulaU`, read the evaluated value with `Result*`. Measured with Visio Pro 2019 C2R x64, COM from PowerShell, save as `.vsdx` and reopen. Region effects (`ResultStr`, lists): [visio-region-settings](visio-region-settings.md). Long text: [visio-cell-string-length](visio-cell-string-length.md).

| Type | Write `FormulaU` | Read | After save + reopen |
|---|---|---|---|
| Text | `"` + text with `"` doubled + `"` | `ResultStr(visNoCast)` | same, except CR LF -> LF |
| Number | `v.ToString("R", CultureInfo.InvariantCulture)` | `Result(visNumber)` | same double |
| Bool | `TRUE` / `FALSE` | `Result(visNumber)`: 1 / 0 | same |
| Date | `DATETIME(<OADate>)` | `DateTime.FromOADate(Result(visDate))` | whole seconds |

- Text: tab, LF, `"`, umlauts, euro sign, emoji (surrogate pair) come back unchanged. CR LF comes back as LF after save; normalize to `\n` before writing.
- Text without quotes is a formula: `007` gives the number 7. Always quote text.
- `""` gives an empty string (not the same as a missing row).
- Number: the `FormulaU` getter shows ~14 significant digits (`0.33333333333333`), but the value keeps the full double, also after save.
- Bool: comparisons give the same 1 / 0 (`1>0` -> 1). `ResultStr` gives `TRUE` / `FALSE`.
- Date: Visio counts days since 1899-12-30, as OLE dates. `DATETIME("2026-10-18 13:45:30")` also works. `Result(visDate)` keeps milliseconds in memory; after save the date is rounded to whole seconds (the formula keeps only that). `ResultStr` formats it by region (`18.10.2026 13:45:30` in de-DE).

## GUARD

[Microsoft](https://learn.microsoft.com/en-us/office/vba/api/visio.cell.formulau) only says to use `FormulaForceU` for guarded cells. Measured:

| Call on a cell with `GUARD(...)` | Effect |
|---|---|
| `FormulaU = ...` | COM exception `Cell is guarded.`, formula unchanged |
| `ResultIU = ...` | COM exception `Cell is guarded.` |
| `FormulaForceU = ...` | written, `GUARD` is gone |
| `Shape.SetFormulas(..., visSetUniversalSyntax)` | COM exception `Cell is guarded.` |
| `Shape.SetFormulas(..., visSetUniversalSyntax + visSetBlastGuards)` | written |

- No silent failure: catch the exception (or test `GUARD` in `FormulaU` first) to skip guarded cells.
- `FormulaForceU` replaces the whole formula, `GUARD` included; write `GUARD(...)` again if the cell should stay protected.
