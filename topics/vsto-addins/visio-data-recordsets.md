---
title: Reading Visio data recordsets
description: Read when Visio code reads linked external data (DataRecordsets, GetRowData, DataColumns) or links an Excel sheet by code - value types, column names, empty fields, ACE connection, relative paths.
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

## Linking an Excel sheet by code

```powershell
$cn = "Provider=Microsoft.ACE.OLEDB.12.0;User ID=Admin;Data Source=$xlsx;Mode=Read;Extended Properties=`"Excel 12.0 Xml;HDR=YES;IMEX=1`""
$rs = $doc.DataRecordsets.Add($cn, "SELECT * FROM [My sheet`$]", 0, 'My sheet')   # last argument = recordset name
```

- `.xlsx` needs `Excel 12.0 Xml`. `Excel 12.0` alone (binary `.xlsb`) gives `Unable to connect to the data.` ACE `12.0` and `16.0` both work inside Visio C2R.
- Without the name argument Visio picks its own recordset name, not the sheet name.
- `Add` sets no primary key: refresh then matches rows by position. Set one: [visio-data-linking](visio-data-linking.md).
- Values from Excel cells: every number is `double` (also `1`, `400`), a cell with a currency format is `decimal`, dates `DateTime`, `TRUE`/`FALSE` `bool`, text `string` (`007` stays text if the cell is text).
- `IMEX=1`: a column with text and numbers becomes text; the numbers come formatted in the user's locale (`7,5` with German settings). In such a mixed column (German settings): `TRUE` -> `TRUE`, a date with number format `dd.mm.yyyy` -> `dd.10.yyyy` (only the month filled in). A column of only dates gives `DateTime`, only `TRUE`/`FALSE` gives `bool`.
- Column names: ACE changes some characters of the Excel header: `.` -> `#`, `!` -> `_`, `` ` `` -> `_`, `[` -> `(`, `]` -> `)`; a name that then collides gets `1`, `2`, ... appended (`a.b` and `a#b` -> `a#b`, `a#b1`). Names are cut to 64 characters. Kept as typed: spaces (also leading/trailing), `' " # $ % & ( ) * + , - / : ; < = > ? @ \ ^ _ { | } ~`, umlauts, a leading digit. An empty header is `Column1` (Excel names it so in a ListObject). A shape data label must equal the column name to be filled ([visio-data-linking](visio-data-linking.md)): labels with the changed characters or over 64 characters never match.
- A ListObject with only a header row (Excel keeps one empty body row, `ListRows.Count` 0) gives 0 rows.
- Rows without any value inside the sheet come as rows of `DBNull` when a value follows further down (in any column, also outside the ListObject). Empty rows at the end are left out, also empty rows at the end of a ListObject. ACE reads the sheet, not the ListObject: a note below the table is a row too.
- Relative path: `Data Source=data.xlsx` (no folder) is stored as given (`visio/data/connections.xml`: `<DataConnection FileName='' ConnectionString='...Data Source=data.xlsx;...'>`). `Add` resolves it against the document folder, `Refresh()` against Visio's current folder: [visio-data-linking](visio-data-linking.md).
- `recordsets.xml` stores the recordset `Name` and the query `Command` (`SELECT * FROM [My sheet$]`) separately: renaming the recordset (`Name = x`, then `Refresh()`) keeps the query and the data. Find the source sheet by `DataRecordset.CommandString`, not by the name.

Verified with Visio Pro 2019 C2R x64, German region settings, by COM automation from PowerShell (2026-10; copy test opened in the Visio UI). _Unverified:_ column order after the user reorders columns in the External Data window; recordsets linked to CSV, SQL or SharePoint.
