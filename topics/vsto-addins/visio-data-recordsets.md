---
title: Reading Visio data recordsets
description: Read when Visio code reads linked external data (Document.DataRecordsets, GetRowData, DataColumns) - value types, column names and order, empty fields, column properties.
---

# Reading Visio data recordsets

Linked data (Data > Link Data) lives in `Document.DataRecordsets`. Read a whole recordset:

```csharp
// Row 0 = column names, in the order of the values of every row.
var columns = ((object[])recordset.GetRowData(0)).Cast<string>().ToList();
var rowIds = (int[])recordset.GetDataRowIDs("");          // "" = all rows
foreach (var id in rowIds)
{
    var values = (object[])recordset.GetRowData(id);      // typed, see below
}
```

| Column type (source) | `GetRowData` value | `DataColumn.GetProperty(visDataColumnPropertyType)` |
|---|---|---|
| text | `string` | 0 (`visPropTypeString`) |
| integer | `int` | 2 (`visPropTypeNumber`) |
| float | `double` | 2 |
| date/time | `DateTime` | 5 (`visPropTypeDate`) |
| boolean | `bool` | 3 (`visPropTypeBool`) |
| currency | `decimal` | 7 (`visPropTypeCurrency`) |
| empty field | `DBNull` | - |

- Values are **not** strings: no locale formatting, no parsing needed. Convert `DBNull` before handing values on.
- The interop signatures return `System.Array`; the runtime types are 0-based `object[]` and `int[]`.
- Column names from `GetRowData(0)` are the source names (`DataColumn.Name`). What the External Data window shows is `DataColumn.DisplayName` (may be renamed by the user).
- `DataColumn.GetProperty` throws `Invalid parameter.` for a property that does not apply to the type: `visDataColumnPropertyUnits` (4) only for numbers, `LangID` (2) for text/date, `Calendar` (3) for dates, `Currency` (5) for currency (e.g. `EUR`). `DisplayName` (6), `Visible` (7), `Hyperlink` (8) always work.
- Test data without a file: `Document.DataRecordsets.AddFromXML(adoXml, 0, "<name>")` takes an ADO rowset XML (`xmlns:s/dt/rs/z`, `s:datatype dt:type='string|int|float|dateTime|boolean'`, currency: `dt:type='number' rs:dbtype='currency'`). A name with `: /` is accepted.

Verified with Visio Pro 2019 C2R x64, German region settings, by COM automation from PowerShell (2026-10). _Unverified:_ column order after the user reorders columns in the External Data window; recordsets linked to Excel, SQL or SharePoint (only `AddFromXML` tested).
