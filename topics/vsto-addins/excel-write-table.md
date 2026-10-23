---
title: Writing a table to Excel by COM
description: Read when code in another process or add-in writes rows to Excel by COM - array write, text turning into numbers, dates, filtered or protected sheets, names, cleanup, PowerShell gotchas.
---

# Writing a table to Excel by COM

Each COM call into Excel crosses processes. Write all values with one assignment, then put the table over the range:

```csharp
var workbook = excel.Workbooks.Add(Excel.XlWBATemplate.xlWBATWorksheet);   // exactly one sheet
var sheet = (Excel.Worksheet)workbook.Worksheets[1];
var range = sheet.Range[sheet.Cells[1, 1], sheet.Cells[rows + 1, columns]];

// Text cells first: header row and every column that holds strings.
headerAndTextColumns.NumberFormat = "@";
range.Value = values;                      // object[rows + 1, columns], 0-based is fine

var list = sheet.ListObjects.Add(Excel.XlListObjectSourceType.xlSrcRange, range,
    XlListObjectHasHeaders: Excel.XlYesNoGuess.xlYes);
list.DisplayName = "MyTable";              // the name for formulas, see below
range.Columns.AutoFit();
```

Not `ListRows.AddEx()` + `ListRow.Range.Value` per row: two cross-process calls per row.

## Values

| .NET value in the array | Cell |
|---|---|
| `string` `"007"`, `"1E3"`, `"12e3"`, `"1/2"`, `"3-4"`, `"jan1"`, `"1jan"`, `"dec31"`, `"1:2"` | parsed like a typed entry: `7`, `1000`, `12000`, dates, a time. Only cells formatted `@` before the write keep the text. Stayed text: `e5`, `1d5`, `a12`, `0x1f`, `wahr`, `jan1234567`. |
| `DateTime` with `Range.Value` | date, each cell gets its own date format: `TT.MM.JJJJ hh:mm`, but `TT.MM.JJJJ` (time hidden) at midnight; a column can mix both |
| `DateTime` with `Range.Value2` | serial number, format stays `General`: looks like `46024,57` |
| `int`, `double`, `decimal` | number, format `General` (currency format is not applied) |
| `bool` | `TRUE`/`FALSE` |
| `null`, `DBNull` | empty cell |

## Filtered or protected sheet

- **With a filter active on the sheet, a write to a range of several cells goes only to its visible cells, and each gets the first value of the array** (`Range.Value2 = object[,]`; table filter and plain range AutoFilter alike). Hidden rows stay as they were; no error. `Range.NumberFormat` on such a range also skips hidden cells.
  - A single cell is written even when its row is hidden; so is a range of only hidden cells. Rows hidden by hand (`EntireRow.Hidden`, no filter) are written normally.
  - Test `Worksheet.FilterMode`: `True` also when the filter of *another* table on the sheet hides rows of yours; `ListObject.AutoFilter.FilterMode` sees only its own filter. Filtered: write cell by cell (one `NumberFormat` + one `Value2` call per cell; VSTO add-in, 10,000 cells: whole run 4.9 s, vs 0.5 s with one array write unfiltered).
- Protected sheet (`Worksheet.ProtectContents`; `Protect()` with defaults): setting `NumberFormat` throws `Unable to set the NumberFormat property of the Range class` even on unlocked cells. Check `ProtectContents` first and tell the user.

## Names

- **`ListObject.Name` takes any text and is not the table name.** The name used in formulas and the Name Manager is `DisplayName`. Setting `Name` to `Test data 1` succeeds silently.
- `DisplayName` is refused (COMException) unless it starts with a letter or `_`, holds only letters, digits, `_`, `.` (non-ASCII letters like `ü` are fine), and does not look like a cell or row/column reference (`A1`, `xfd1048576`, `R1C1`, `RC`, `R`, `C`, `C12`). Unique in the workbook ignoring case.
- Sheet names: at most 31 characters, none of `: \ / ? * [ ]`, not `History`, unique ignoring case. `'` inside is fine (`It's`); Excel refuses it at the start or end.
- `Workbooks.Add(xlWBATWorksheet)` gives one sheet regardless of the user's "sheets in new workbook" option: no placeholder sheet to delete.

## Which Excel, and cleanup

- `Marshal.GetActiveObject("Excel.Application")` gets a running Excel (COMException if none). An invisible running instance usually belongs to another program's automation: start your own (`new Excel.Application()`) instead.
- On an error: `workbook.Close(SaveChanges: false)`, and `Quit()` an instance you started, else a hidden `EXCEL.EXE` stays.
- Hand a started instance to the user with `Visible = true` and `UserControl = true`.
- An Excel the user closes stays as a process while .NET still holds RCWs (common knowledge, not tried). Remedy: do the COM work in a separate `[MethodImpl(MethodImplOptions.NoInlining)]` method, then `GC.Collect(); GC.WaitForPendingFinalizers();`. With it the process ended ~3 s after the window was closed (verified from a Visio VSTO add-in).
- A started Excel did not come to the front (add-in button clicked by UI Automation; a real click not tried).
- The new workbook is named `Sheet1` (title bar `Sheet1 - Excel`), not `Book1`/`Mappe1` (German UI, `xlWBATWorksheet`, from a VSTO add-in).

## Test scripts in Windows PowerShell 5.1

- Late-bound setters cache the argument type per property: after `$cell.Value2 = 'text'`, a later `$other.Value2 = 1` (or an `object[,]`) throws `Unable to cast object of type 'System.Int32' to type 'System.String'.` Write the whole block, header included, with one `Value2 = $array` (`New-Object 'object[,]' $rows, $cols`).
- `Range.Value = ...` fails the same way; use `Value2` (dates as `ToOADate()`).
- Late-bound `NumberFormat` reads the codes in the user's locale: with German settings use `TT.MM.JJJJ hh:mm`, `#.##0,00`; `dd.mm.yyyy` shows `dd.10.yyyy`.
- `Quit()` + `ReleaseComObject` leaves `EXCEL.EXE` running until the PowerShell process exits.

Verified with Excel 2019 C2R x64, German UI (2026-10): values, formats, names, `xlWBATWorksheet` and the PowerShell notes by COM automation from PowerShell; process end, foreground and workbook name from a VSTO add-in in Visio 2019; filtered and protected sheets from PowerShell and from a VSTO Excel add-in. _Unverified:_ cleanup after an error.
