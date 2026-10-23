---
title: Linking Visio shapes to data rows and refreshing
description: Read when Visio code links shapes to data recordset rows or refreshes linked data - primary key, row matching, refresh conflicts, shape data matching, edited values, column changes.
---

# Linking Visio shapes to data rows and refreshing

Measured with Visio Pro 2019 C2R x64, German region settings, COM from PowerShell, recordsets from `.xlsx` via ACE (connection: [visio-data-recordsets](visio-data-recordsets.md)). Calls with `out` arrays (`GetLinkedDataRecordsetIDs`, `GetShapesLinkedToData*`, `GetPrimaryKey`) fail in PowerShell late binding; call them from C# (`Add-Type -ReferencedAssemblies` the interop DLL from `C:\Windows\assembly\GAC_MSIL\Microsoft.Office.Interop.Visio`). Signature: `GetPrimaryKey(out VisPrimaryKeySettings, out System.Array)` (`out string[]` does not compile).

## Primary key: set it

`DataRecordsets.Add` gives no primary key: rows match **by position** on refresh (saved as `RowOrder='1'`). Sort or insert a row in the source and every linked shape silently shows another row's data. Set a key column right after `Add`:

```csharp
recordset.SetPrimaryKey(Visio.VisPrimaryKeySettings.visKeySingle, new[] { "ID" });  // enum: RowOrder 1, Single 2, Composite 3
```

Saved as `<PrimaryKey ColumnNameID='ID'><RowKeyValue RowID='1' Value='a1'/>...`. With a key, reordered source rows keep their Visio row IDs and links.

Key matching on `Refresh()`:

| Source change | Result |
|---|---|
| Rows reordered | links kept |
| Key changes case only (`a1` -> `A1`) | same row, link kept: matching ignores case |
| Two keys differ only in case (`abc`, `ABC`) | both shapes lose their link, already on an unchanged refresh |
| Key duplicated (second row `a1`) | shape linked to `a1` loses its link; not restored when the duplicate is removed again |
| One row with empty key | treated as key `""`, link kept |
| Two rows with empty key | the linked shape loses its link |
| Two rows with no value at all (blank rows, key `DBNull`), no shape linked to them | no conflict, links of the other rows kept |
| Row removed | shape loses its link, keeps its values |
| Key `0007`, `1E3`, `-5`, `12.3` in text cells | kept as text, matched |

- Refresh time with a key grows about with the square of the row count (8 text columns, unchanged data, no linked shapes, `Refresh()` by COM): 1,000 rows 394 ms, 2,500 rows 1,935 ms, 5,000 rows 7,221 ms, 10,000 rows 29,683 ms. Same 10,000 rows without key: 711 ms (`Add` 747 ms). `SetPrimaryKey` itself: a few ms.
- Visio row IDs are never reused (a removed or failed row burns its ID).
- A shape that lost its link stays in `DataRecordset.GetAllRefreshConflicts()`; `GetMatchingRowsForRefreshConflict(shape)` gives the competing rows (duplicate key) or nothing (row removed). `Refresh()` by code shows no conflict UI. Relink: `shape.LinkToData(recordsetId, rowId, false)`.

## Linking

- `Shape.LinkToData(recordsetId, rowId, applyDataGraphic)` and `Page.LinkShapesToDataRows(recordsetId, rowIds, shapeIds, applyDataGraphic)` link directly, no matching field needed.
- `Selection.AutomaticLink(recordsetId, columnNames, fieldTypes, fieldNames, flags, out shapeIds)` matches by value, e.g. column `Name` = shape data label `Name` (`visAutoLinkCustPropsLabel` = 2, flags `visAutoLinkSelectedShapesOnly` 1 + `visAutoLinkNoApplyDataGraphic` 4). Returns the number of linked shapes.
- A column fills the existing shape data row whose **label** equals the column name. The row name does not count: column `Sig3` and a row `Prop.Sig3` with label `Signal 3` give a new row `Prop._VisDM_Sig3`. Columns without a matching label become new rows `Prop._VisDM_<column>`. A row with an empty label is never matched: column `NoLabel` next to `Prop.NoLabel` (label empty) gives `Prop._VisDM_NoLabel`.
- Linking writes values over formulas (`Prop.F = Prop.A&"!"` becomes `"99"`).
- Into typed rows: fixed list (type 1) gets `INDEX(n,Prop.X.Format)`; a value not in the list gives `INDEX(0,...)` = first entry, without warning. Number (type 2): `12` from `5 mm` -> unit lost; text `7,5` is parsed with the region (`7.5`). Date (type 5): text `2026-10-22` and `22.10.2026` -> `DATETIME(46317)`. Boolean (type 3): text `TRUE`/`FALSE` (column of text cells, ACE gives `string`) -> formula `TRUE`/`FALSE`, the same as from a `bool` column.
- Visio types each data column itself: `DataColumn.GetProperty(1)` (`visDataColumnPropertyType`) = `visPropType*` value, e.g. text 0, number 2, date 5. Text and number values into a **Duration** row (type 6) give `0` (tried number `5`, texts `5 min`, `5 Min.`, `5 ed.`, `0:05`, `1,5`). A **fixed list whose `Format` is a formula** (`IF(STRSAME(Prop.T,"In"),"x;y","z")`) gets `0`, also when the value is in the list the formula gives.
- `DataColumn.SetProperty(1, 6)` (column type Duration) is accepted, `SetProperty(1, 1)` (fixed list) fails with `Invalid parameter`. After setting Duration, `Refresh()` + `LinkToData` gave the Duration row still `0`, and the shape's Number, Duration and Currency rows (`Prop.Channel`, `Prop.Wait`, `Prop.Price`) were replaced by `Prop._VisDM_*` rows: formulas referring to them break. Cause not isolated; do not change column types on a recordset with linked shapes.
- A shape data row with `Invisible = TRUE` and a matching label is filled like any other and stays hidden; no dialog appeared, neither by code (`AlertResponse` 0 and 1) nor when the user dragged the row from the External Data window onto the shape.
- One shape linked to two recordsets with the same column name (e.g. both have `ID`) shares one shape data row: the second link overwrites the first one's value.
- Find multi-links: `Shape.GetLinkedDataRecordsetIDs(out ids)` (recordsets of a shape), `Page.GetShapesLinkedToDataRow(recordsetId, rowId, out shapeIds)` (shapes of a row), `Page.GetShapesLinkedToData(recordsetId, out shapeIds)`.

## Refresh and linked shapes

- Changed values reach linked shapes; the row is matched by key (above).
- A linked value the user edits in the shape (shape data window or `FormulaU`) is **kept** on refresh; the shape's other linked values still update. `recordset.RefreshSettings = visRefreshOverwriteAll` (1) overwrites edited values too. Default `RefreshSettings` = 0. Repair one shape: `LinkToData` to the same row again restores the row's values, and later refreshes update them again (`BreakLinkToData` first is not needed).
- Column added in the source: appears in the recordset and as a new shape data row in every linked shape. Column removed: its shape data row is removed. Column renamed: = removed + added (`_VisDM_X` gone, `_VisDM_Y` new).
- Hidden column (`DataColumn.SetProperty(visDataColumnPropertyVisible, false)` before linking): no shape data row is created. Made visible later: the row appears at once with the current value, but refresh does not update it until the shape is linked again (`LinkToData`).
- Text of 400 characters came through ACE complete (2 rows; longer text below 8 short rows not tested, ACE guesses types from the first rows).

## Relative Data Source

- `DataRecordsets.Add` with `Data Source=data.xlsx` (also `.\data\x.xlsm`, `..\other\x.xlsx`) resolves against the **document folder** (saved document).
- `DataRecordset.Refresh()` and Data > Refresh All in the UI resolve it against the **current folder of the Visio process**. Wrong folder: `Unable to connect to the data.` (UI: `Error 80004005: ... could not find the object '...\AppData\Local\Temp\VSO<hex>.xlsx'`), or silently another folder's `data.xlsx`.
- The current folder depends on how documents were opened: the File Open dialog sets it to the document's folder; Visio started by COM, and a running Visio that got a document by double-click in Explorer, stayed at `C:\WINDOWS\system32`. `VISIO.EXE "<file>"` started by a process keeps that process's current folder (`Start-Process -WorkingDirectory`), it does not switch to the file's folder. With documents from two folders open, the last File Open decides. Once a `Refresh()` succeeded although the folder was `system32` (not reproduced, cause unknown).
- Workaround in a VSTO add-in (measured Visio 2019 C2R x64): keep the current folder on the folder of the active drawing. The add-in runs in Visio's process, so `Directory.SetCurrentDirectory` sets the folder ACE uses. Document untouched (not dirty). Same test with the add-in off: every refresh silently read the other folder's file. Measured with `DataRecordset.Refresh()` by COM; the UI's Refresh All should follow (same folder rule), _unverified_. Call it also right before an own `Refresh()`.
  ```csharp
  Application.VisioIsIdle += app =>
  {
      var doc = app.ActiveWindow?.Document;   // may throw COMException: catch
      if (doc?.Type != Visio.VisDocumentTypes.visTypeDrawing || doc.Path.Length == 0)
          return;                              // stencil, unsaved drawing: leave it
      if (!string.Equals(doc.Path.TrimEnd('\'), Directory.GetCurrentDirectory().TrimEnd('\'), StringComparison.OrdinalIgnoreCase))
          Directory.SetCurrentDirectory(doc.Path);   // Path ends with '\'; may throw (URL, deleted folder)
  };
  ```
  Without an add-in: set `DataConnection.ConnectionString` to the absolute path, `Refresh()`, set the relative string back (changes the document). Each `Add` gets its own `DataConnection` (setting one does not change the other).
- The error looks random (works after save, close, reopen; works in a copied folder; the Link Data wizard works only after browsing to the file): all match the current folder rule. `DataConnection.FileName` (read-only) is empty for `Add` by code and set by the wizard; setting it in `connections.xml` did not change the error (VisGuy forum thread "DatarecordSet with relative path to Excel worksheet", 2022). _Unverified:_ that the wizard's browse dialog sets the current folder; behavior of Visio before 2019.
- Tests: read another process's current folder from its PEB (x64: `NtQueryInformationProcess` -> `PEB+0x20` ProcessParameters -> `+0x38` CurrentDirectory); set it inside Visio with `doc.ExecuteLine("ChDir ""<path>""")`. A whole `Refresh` in a wrong folder in one call (no idle between): `doc.ExecuteLine("ChDir ""<path>"": ThisDocument.DataRecordsets(1).Refresh")`. Start Visio in a given folder: `Start-Process VISIO.EXE '"<file>"' -WorkingDirectory <folder>` (add-ins load), then `GetActiveObject`. `Documents.Open`/`OpenEx` by COM from PowerShell hung on one machine; `doc.ExecuteLine("Documents.Open ""<file>""")` opened the second document.

## Undo

`Add` (with `SetPrimaryKey`), `Refresh()` and `Delete()` inside one undo scope ([visio-undo-scopes](visio-undo-scopes.md)) are one Undo step: `Undo()` removes the added recordsets, puts the old values back into the recordset and the linked shapes, and brings a deleted recordset back with its key; `Redo()` does it all again (measured by COM, VSTO add-in held the scope).

## Source open in Excel

- ACE reads a workbook open in Excel: it gets the **saved** state; unsaved edits are not seen.
- While Excel has it open, another process cannot open the file for writing (sharing violation); reading works.
- Another process holding the file open, even read-only with `FileShare.ReadWrite | FileShare.Delete`: `DataRecordsets.Add` failed with `Unable to connect to the data.` (measured once). Close own readers of the file before Visio reads it.
