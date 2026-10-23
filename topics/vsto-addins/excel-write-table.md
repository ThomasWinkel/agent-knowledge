---
title: Writing a table to Excel by COM
description: Read when code writes rows to Excel by COM - array write, text turning into numbers, dates, appending to a table, filtered or protected sheets, names, open workbook, cleanup, PowerShell.
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
| `string` `"007"`, `"1E3"`, `"12e3"`, `"1/2"`, `"3-4"`, `"jan1"`, `"1jan"`, `"dec31"`, `"1:2"`, `"TRUE"` | parsed like a typed entry: `7`, `1000`, `12000`, dates, a time, a bool. Only cells formatted `@` before the write keep the text (`=1+1` too). Stayed text: `e5`, `1d5`, `a12`, `0x1f`, `wahr`, `jan1234567`. |
| `string` with a leading `'` (`"'007"`, `"'TRUE"`, `"'=1+1"`) | text without the `'` (`PrefixCharacter` = `'`), in `General` and `@` cells alike. **Also in `@` cells a leading `'` is eaten**: `"'x"` gives `x`, so write `"''x"` for `'x`. Text guard for cells of unknown format: prefix every string with `'`. |
| string over 32,767 chars | cut to 32,766 chars, no error. A row holding such a cell makes `Range.Formula` throw `0x800A03EC`; `WorksheetFunction.CountA` works. |
| `DateTime` with `Range.Value` | date, each cell gets its own date format: `TT.MM.JJJJ hh:mm`, but `TT.MM.JJJJ` (time hidden) at midnight; a column can mix both. A cell already formatted `m/d/yyyy` keeps it. In an `@` cell: US text `10/23/2026`. **Before 1900: `COMException` `E_OUTOFMEMORY` (0x8007000E)**: write the serial number instead. |
| `DateTime` with `Range.Value2` | serial number, format stays `General`: looks like `46024,57` |
| `int`, `double`, `decimal` | number, format `General` (currency format is not applied) |
| `bool` | `TRUE`/`FALSE` |
| `null`, `DBNull` | empty cell |

## Number formats and empty tables

- **Early-bound interop (C# with the PIA) reads and writes `NumberFormat` in English codes, whatever the thread culture** (`General`, `@`, `m/d/yyyy`; same with `de-DE` and `en-US`). `NumberFormatLocal` gives the local code (`Standard`, `TT.MM.JJJJ`). Late-bound PowerShell differs (below).
- `m/d/yyyy` is Excel's built-in short date: cells show the date as the region settings say (`TT.MM.JJJJ` with German settings).
- A table with no data yet: `ListObjects.Add` over the header row plus one empty row gives a body of one row. Set formats and validation on `ListColumns[i].DataBodyRange`: rows added by `ListRows.Add()` take them on.
- `ListObjects.Add` over the header row alone: `ListRows.Count` 0, `DataBodyRange` null, but `Range` includes an empty `InsertRowRange`. The first `ListRows.Add()` then gives **two** rows (count 2): loop `while (ListRows.Count < n) ListRows.Add();`.
- Mixed formats in a range: `Range.NumberFormat` returns no string (null/`DBNull`). `Range.EntireRow.Hidden`: `true` all hidden, `false` none, else null.
- `ListColumn.DataBodyRange.HasFormula` is `true` when every cell has a formula: a usable test for a calculated column. Writing a formula into one table cell by COM makes Excel fill the whole column (seen once, not isolated).

## Appending rows to an existing table

Fast and like `ListRows.Add` (only cells below the table in its columns move down; formats, validation and calculated-column formulas carried on):

```csharp
var range = list.Range;                                   // header + body (+ total row)
int below = range.Row + range.Rows.Count, left = range.Column, right = left + range.Columns.Count - 1;
sheet.Range[sheet.Cells[below, left], sheet.Cells[below + n - 1, right]]
    .Insert(Excel.XlInsertShiftDirection.xlShiftDown, Excel.XlInsertFormatOrigin.xlFormatFromLeftOrAbove);
list.Resize(sheet.Range[sheet.Cells[range.Row, left], sheet.Cells[below + n - 1, right]]);
// then one Range.Value write per column; skip a column with no values, else null clears a calculated formula
```

| 300 rows | Time (out of process) |
|---|---|
| `Insert` + `Resize` | 9 ms |
| `ListRows.Add()` per row | 1,032 ms |
| values, 6 columns, one write per column | 39 ms (cell by cell: 3,462 ms) |

- `Resize` alone also carries formats, validation and formulas, but swallows content below the table into it.
- A write into the row right below a table extends the table (auto-expand works for COM writes).
- **With a filter active on the sheet, `Range.Insert` (shift down) fails** (`Insert method of Range class failed`), even below the table. `ListRows.Add()` and `Resize` work; inserting whole sheet rows works but moves cells beside the table too. Use `ListRows.Add()` then; also when the table has no rows (Insert would copy the header's formats) or a total row (`ShowTotals`).
- New rows appended while filtered stay visible; an array write into all-visible rows is fine with `FilterMode` on.
- Insert into a range crossing a merged cell: with `DisplayAlerts = true` Excel shows a message box and **the COM call waits until the user closes it**; with `false` it inserts and silently unmerges the cell.
- `Workbook.Save()` onto a file whose read-only attribute was set after it was opened: Excel shows "Save As" even with `DisplayAlerts = false`; the call waits.
- Undo an append in a user's open workbook: `ListRows[i].Delete()` from the end, `ClearContents` on cells you filled in reused rows (only your columns: a calculated formula stays), then `Workbook.Saved = true`.

## Filtered or protected sheet

- **With a filter active on the sheet, a write to a range of several cells goes only to its visible cells, and each gets the first value of the array** (`Range.Value2 = object[,]`; table filter and plain range AutoFilter alike). Hidden rows stay as they were; no error. `Range.NumberFormat` on such a range also skips hidden cells.
  - A single cell is written even when its row is hidden; so is a range of only hidden cells. Rows hidden by hand (`EntireRow.Hidden`, no filter) are written normally.
  - Test `Worksheet.FilterMode`: `True` also when the filter of *another* table on the sheet hides rows of yours; `ListObject.AutoFilter.FilterMode` sees only its own filter. Filtered: write cell by cell (one `NumberFormat` + one `Value2` call per cell; VSTO add-in, 10,000 cells: whole run 4.9 s, vs 0.5 s with one array write unfiltered).
- Protected sheet (`Worksheet.ProtectContents`; `Protect()` with defaults): setting `NumberFormat` throws `Unable to set the NumberFormat property of the Range class` even on unlocked cells. Check `ProtectContents` first and tell the user.

## Names

- **`ListObject.Name` takes any text and is not the table name.** The name used in formulas and the Name Manager is `DisplayName`. Setting `Name` to `Test data 1` succeeds silently.
- `DisplayName` is refused (COMException `0x800A03EC`, no text) when a workbook-level defined name (`Workbook.Names`, not `Sheet!Name`) has the name, and unless it starts with a letter or `_`, holds only letters, digits, `_`, `.` (non-ASCII letters like `ü` are fine), and does not look like a cell or row/column reference (`A1`, `xfd1048576`, `R1C1`, `RC`, `R`, `C`, `C12`). Unique in the workbook ignoring case.
- Sheet names: at most 31 characters, none of `: \ / ? * [ ]`, not `History`, unique ignoring case (taken: `That name is already taken. Try a different one.`). `'` inside is fine (`It's`); Excel refuses it at the start or end.
- Workbook structure protected (`Workbook.ProtectStructure`): `Worksheets.Add` throws `0x800A03EC` without text. Check it first.
- `Workbooks.Add(xlWBATWorksheet)` gives one sheet regardless of the user's "sheets in new workbook" option: no placeholder sheet to delete.

## Which Excel, and cleanup

- `Marshal.GetActiveObject("Excel.Application")` gets a running Excel (COMException if none). Right after an instance quit, its stale entry can still be returned: the cast to `Excel.Application` throws `InvalidCastException` (`RPC_E_DISCONNECTED` 0x80010108), not `COMException`; treat it as "none running". An invisible running instance usually belongs to another program's automation: start your own (`new Excel.Application()`) instead. An invisible leftover (e.g. of a crashed script) that registered first hides every visible instance from `GetActiveObject`.
- `GetActiveObject` returns only the first instance (the Running Object Table holds one `Excel.Application` entry). Every open workbook of every instance is in the ROT under its full path, also one opened read-only in a second instance (`excel.exe /x /r`). Find a workbook by path: enumerate the ROT (`GetRunningObjectTable` -> `EnumRunning`, compare `IMoniker.GetDisplayName` ignoring case, then `GetObject`); the 8.3 short path does not match, so compare full long paths. Then `Workbook.ReadOnly`, `Saved` (`false` after an unsaved change) and `Application` (that instance).
- Not `Marshal.BindToMoniker(path)` / VB `GetObject(path)` to look for it: for a file that is not open it opens the workbook in the first running instance with a hidden window (no new process, 161 ms). For an open one it returns it (path case does not matter).
- A workbook open in Excel locks the file (other processes get a sharing violation on write); opened read-only it does not.
- `Workbooks.Open` with `DisplayAlerts = false`: a file with the read-only attribute, or held by another process that allows reading, opens with `ReadOnly = true`, no question. A file another process holds exclusively throws `0x800A03EC` `Excel cannot open the file ... because the file format or file extension is not valid`: open it with `File.Open(path, Open, Read, ReadWrite | Delete)` first for a true error.
- An instance can hold one workbook per file name: with `data.xlsx` of another folder open, `Workbooks.Open` of a second `data.xlsx` returns `null` silently (`DisplayAlerts = false`), `SaveAs` to that name throws `0x800A03EC`. Check `Workbook.Name` of the open workbooks and use another instance.
- Take back a sheet added to the user's saved, open workbook: `DisplayAlerts = false`, `sheet.Delete()`, then `Workbook.Saved = true` (Excel asks nothing on close, content is as saved).
- Starting Excel (`new Excel.Application()`) took 2.0-2.1 s, opening a small workbook then 0.1-0.3 s (3 runs, add-ins installed).
- On an error: `workbook.Close(SaveChanges: false)`, and `Quit()` an instance you started, else a hidden `EXCEL.EXE` stays.
- Hand a started instance to the user with `Visible = true` and `UserControl = true`.
- An Excel the user closes stays as a process while .NET still holds RCWs (common knowledge, not tried). Remedy: do the COM work in a separate `[MethodImpl(MethodImplOptions.NoInlining)]` method, then `GC.Collect(); GC.WaitForPendingFinalizers();`. With it the process ended ~3 s after the window was closed (verified from a Visio VSTO add-in).
- A started Excel did not come to the front (add-in button clicked by UI Automation; a real click not tried).
- The new workbook is named `Sheet1` (title bar `Sheet1 - Excel`), not `Book1`/`Mappe1` (English UI, `xlWBATWorksheet`, from a VSTO add-in).

## Test scripts in Windows PowerShell 5.1

- Late-bound setters cache the argument type per property: after `$cell.Value2 = 'text'`, a later `$other.Value2 = 1` (or an `object[,]`) throws `Unable to cast object of type 'System.Int32' to type 'System.String'.` Write the whole block, header included, with one `Value2 = $array` (`New-Object 'object[,]' $rows, $cols`).
- `Range.Value = ...` fails the same way; use `Value2` (dates as `ToOADate()`).
- Late-bound `NumberFormat` reads the codes in the user's locale: with German settings use `TT.MM.JJJJ hh:mm`, `#.##0,00`; `dd.mm.yyyy` shows `dd.10.yyyy`.
- `Quit()` + `ReleaseComObject` leaves `EXCEL.EXE` running until the PowerShell process exits.

Verified with Excel 2019 C2R x64, English UI (`LanguageSettings.LanguageID(msoLanguageIDUI)` 1033), German region settings (2026-10): values, formats, names, `xlWBATWorksheet` and the PowerShell notes by COM automation from PowerShell; process end, foreground and workbook name from a VSTO add-in in Visio 2019; filtered and protected sheets from PowerShell and from a VSTO Excel add-in. ROT lookup and start times by PowerShell with workbooks opened by `excel.exe` (2026-10). Number formats, empty tables, opening, same file name, structure protection, sheet rollback: C# console (net48, embedded PIA), thread culture `de-DE` and `en-US` (2026-10). Cleanup after an error (`Close(false)`, `Quit()`, then `GC.Collect()`): no `EXCEL.EXE` left 3 s later (C# console). Appending to tables, leading `'`, long text, dates before 1900, header-only tables, merged cell, Save As, stale `GetActiveObject`: C# console (net48), German region (2026-10). _Unverified:_ network paths (mapped drive vs. UNC) in the ROT.
