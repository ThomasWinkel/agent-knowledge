---
title: Typed values in Visio ShapeSheet cells
description: Read when Visio code writes or reads text, numbers, booleans or dates in ShapeSheet cells (empty cells, types, errors, new rows, master instances, GUARD, read-only documents).
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
- Number: exponents work as `"R"` writes them (`1E+300`, `1.5E-05`, `double.MaxValue`; the getter shows `1e300`). Denormal doubles (`double.Epsilon` = `4.94065645841247E-324`) fail: COM exception `#VALUE!`. NaN/infinity have no formula: check before writing.
- Bool: comparisons give the same 1 / 0 (`1>0` -> 1). `ResultStr` gives `TRUE` / `FALSE`.
- Date: Visio counts days since 1899-12-30, as OLE dates. `DATETIME("2026-10-18 13:45:30")` also works. `Result(visDate)` keeps milliseconds in memory; after save the date is rounded to whole seconds (the formula keeps only that). `ResultStr` formats it by region (`18.10.2026 13:45:30` in de-DE). `DATETIME` takes one argument: `DATETIME(2026,10,1)` gives no error and means `DATETIME(2026)` = 1905-07-18; use `DATE(2026,10,1)+TIME(13,30,0)`.

## Empty cells, types and errors on read

`Result*` never says "empty" or "wrong type": check first.

| Check | How | Seen |
|---|---|---|
| Empty | `FormulaU.Length == 0` | empty formula: `Result` 0, `ResultStr` `0,0000`, `Units` 32 |
| Error | `Cell.Error != 0` | `1/0` -> 39, `ResultStr` `#DIV/0!`; `NA()` -> 74, `#N/A`. `Units` is not reliable then. |
| Type | `Cell.Units` | text 231 (`visUnitsString`); number and TRUE/FALSE 32 (`visNumber`, no unit for bool); `3 mm` 70; `3 in` 65; date 40 (`visDate`) |

- Without the type check: `Result(visNumber)` of the text `"5"` gives 5, of `"abc"` 0; of `3 mm` gives inches (0.118...).
- Row added by `AddNamedRow` (sections `User`, `Prop`): its value formula is `0`, not empty. Set it to `""` if a new row should read as "no value".
- `CellsU["User.X"]` of a missing row: COM exception `Unexpected end of file.` (does not name the cell). Check `CellExistsU(name, 0)` first.
- `CellExistsU("User.X.Prompt", 0)` is true when the row `User.X` exists.
- `PageSheet`: `Type` 1 (`visTypePage`), `NameU` `ThePage`, `ContainingPage` = the page. `DocumentSheet`: `Type` 6 (`visTypeDoc`), `NameU` `TheDoc`. Use the page or document name in messages.
- An invalid formula set by code (unknown function, `AlertResponse` not set, invisible instance) opened a modal Visio dialog instead of throwing: the automation call hangs. _Unverified:_ whether `AlertResponse = 1` prevents it.

## Master instances

- `FormulaU = string.Empty` (also `FormulaForceU`) on a cell of a master instance does not empty it: it removes the local formula, the cell inherits again (`IsInherited` true, later master changes arrive). On a shape without master it empties the cell. To clear a value in an instance, write the formula `""` (two quote characters = empty text; C# `"\"\""`; _unverified_ in an instance).
- `Shape.DeleteRow` on a row the instance inherits removes the row from the instance for good: it stays gone when the master changes later.
- Adding a row to a section of an instance (`AddNamedRow`), or deleting a row it inherits, copies the whole section into the instance: every row there turns local (`IsInherited` false), and later master changes of any cell in that section no longer arrive. Setting one cell makes only that cell local. Deleting the added row again does not bring inheritance back (Undo does); `FormulaU = ""` on each value cell does for these cells (other columns not checked).
  - Rows that instances fill in (e.g. an ID per instance): add them to the master, empty; the instance only sets the value.
- Measured with `User` and `Prop` rows, master in the document stencil, master changed by `Master.Open()`/`Close()` (section copy: 2026-10, Visio 2019 C2R x64).

## Read-only documents

A document opened read-only (`Documents.OpenEx(path, visOpenRO)`, `Document.ReadOnly` = -1; seen with a stencil in its own window) accepts cell writes:

- `AddNamedRow` + `FormulaU` on its `DocumentSheet` work, no exception, no dialog; Undo/Redo work.
- `Document.Saved` stays `True`: the change is not marked as unsaved and cannot be saved to that file.
- Check `Document.ReadOnly` yourself if a write there must fail or warn.

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
