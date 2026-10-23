---
title: Excel data validation by COM
description: Read when code adds Excel data validation (drop-down lists, custom formulas) by COM - list separator by region, 255-character limit that breaks the file, booleans, case.
---

# Excel data validation by COM

```csharp
var sep = (string)excel.International[Excel.XlApplicationInternational.xlListSeparator];
var items = string.Join(sep, new[] { "In", "Out", "Bus" });
if (items.Length <= 255)
{
    range.Validation.Delete();
    range.Validation.Add(Excel.XlDVType.xlValidateList, Excel.XlDVAlertStyle.xlValidAlertStop,
        Excel.XlFormatConditionOperator.xlBetween, items);
    // Suggestion only (other values allowed, no message): range.Validation.ShowError = false;
}
```

## Gotchas

- **`Formula1` is read in the user's local format.** A typed list needs Excel's list separator (`International[xlListSeparator]`, `;` with German region settings): `a,b,c` there is *one* item `a,b,c`. Custom formulas too: `=AND(ISLOGICAL(A1);TRUE)` worked, the same with `,` threw `0x800A03EC`; function names follow the UI language (English UI: `ISLOGICAL` ok, `ISTLOG` refused). Locale-proof custom formula: write it with `Range.Formula` into a spare cell and read `FormulaLocal` _(unverified)_.
- **A typed list over 255 characters (items plus separators) is accepted by `Validation.Add` and kept in memory, but the saved file no longer opens**: `Workbooks.Open` fails with `Unable to get the Open property of the Workbooks class`, also with `CorruptLoad`. 255 reopened fine. Check the length before `Add`; longer lists need a cell range as source (`=Lists!$A$1:$A$300`).
- An item cannot contain the list separator. Kept literally: spaces inside, `"` (item `"q"` accepts `"q"`, not `q`), `,` when the separator is `;`. An item with leading/trailing spaces (` a `) accepted neither `a` nor ` a `: leave such lists without validation.
- Text items match case-sensitive: list `Abc` rejects `abc` and `ABC`.
- The cell value is parsed before the check, like a typed entry: list `1;2;3` in a General cell holds `2` as a number, in a `@` cell as text; both pass.
- Booleans: list `TRUE;FALSE` in a General cell: typed `TRUE`/`true` -> `bool`, passes; in a `@` cell `TRUE` stays text and passes, `true` fails. `WAHR` is text in an English-UI Excel and fails. List from a range of two `bool` cells (`=Lists!$A$1:$A$2`): typed `TRUE` passes as `bool`; the drop-down shows the UI language's words. Custom `=ISLOGICAL(<cell>)` passes only `bool` (no drop-down). Same in every UI language, with drop-down: `@` cells + list `TRUE;FALSE` (text; Visio turns it into a real Boolean on link: [visio-data-linking](visio-data-linking.md)).
- `Validation.Value` (read-only, `bool`) tells whether the cell's current value passes; handy in tests.
- Alert style `xlValidAlertWarning`/`Information` still show a message on a wrong value; `ShowError = false` shows none.
- Validation does not stop paste; a value set by code is not checked.

Verified with Excel 2019 C2R x64 (version 16.0, UI language 1033, German region settings), by COM from PowerShell (2026-10), values set with `Range.FormulaLocal` like a typed entry. _Unverified:_ a pick from the drop-down stores the same as typing; a German-UI Excel (`WAHR`, local function names).
