---
title: Building Visio shapes and stencils by code
description: Read when code builds Visio masters, groups or a stencil by COM (geometry rows, text fields, Shape Data, groups from shapes, master from a shape) - row tags, write order, stable IDs, icons, errors.
---

# Building Visio shapes and stencils by code

Recipe for a script that generates a stencil (`.vssx`) from nothing, so the script is the source and the stencil can be rebuilt. Measured 2026-11-02, Visio 2019 C2R x64 build 20527, `Visio.InvisibleApp` from Windows PowerShell 5.1. Connection rows: [visio-connection-points](visio-connection-points.md).

## Recipe

```powershell
$app = New-Object -ComObject Visio.InvisibleApp
$app.EventsEnabled = 0    # add-ins load here too; set back to -1 at the end (Visio keeps it)
$app.AlertResponse = 1    # IDOK: a dialog in an invisible Visio hangs the script without output
try {
    $doc = $app.Documents.AddEx('', 1, 0)                  # visMSMetric
    # Document/page cells that master formulas read must exist first (else #NAME?)
    $m = $doc.Masters.Add(); $m.NameU = 'MyBlock'; $m.Name = 'MyBlock'; $m.Prompt = 'Tooltip in the stencil'
    $copy = $m.Open()
    $a = $copy.DrawRectangle(0, 0, 1, 0.5); $b = $copy.DrawRectangle(0, 0, 0.5, 0.25)
    $sel = $copy.CreateSelection(0); $sel.Select($a, 2); $sel.Select($b, 2)
    $g = $sel.Group()                                      # sub-shape formulas: "$($g.NameID)!Width"
    $copy.Close()                                          # writes the master and its icon
    $doc.SaveAsEx('C:\path\MyStencil.vssx', 0)             # the extension makes it a stencil
} finally {
    foreach ($d in @($app.Documents)) { $d.Saved = $true } # Quit must not ask to save after an error
    $app.EventsEnabled = -1; $app.Quit()
}
```

- Masters reference their own sheets by `Sheet.<n>`; Visio renumbers them consistently on drop.
- The saved `.vssx` keeps the drawing's `Page-1` (harmless: a stencil window shows no pages).

## Stable master IDs

Every build gives each master a new `BaseID` and `UniqueID`; both are read-only in COM. Visio matches master versions by them ([visio-master-versions](visio-master-versions.md)), so a rebuilt stencil looks like new masters. Fix: patch `visio/masters/masters.xml` in the saved `.vssx` (a zip):

- `BaseID`: a fixed GUID per master (`NameU`), kept in the script.
- `UniqueID`: a hash of the master's content (its `<Master>` element without the two IDs + its `masterN.xml`, found via `visio/masters/_rels/masters.xml.rels`). Write it in Visio's form: Visio keeps only the first 8 bytes and replaces the rest with `8E40-00608CF305B2` when it reads the file (`{83A034C7-8A0A-90AE-F071-...}` reads back as `{83A034C7-8A0A-90AE-8E40-00608CF305B2}`).

```powershell
Add-Type -AssemblyName System.IO.Compression, System.IO.Compression.FileSystem
$zip = [IO.Compression.ZipFile]::Open($path, 'Update')
# per <Master ...>...</Master> in masters.xml:
$hash = (New-Object Guid (,$md5.ComputeHash($contentBytes))).ToString().ToUpper()
$unique = '{' + $hash.Substring(0, 18) + '-8E40-00608CF305B2}'
# replace UniqueID='...' and BaseID='...', delete the entry, write it again, $zip.Dispose()
```

Copies keep the IDs: `Document.Masters.Drop(master)` and `Page.Drop(master)` into another document give the copy the source's `BaseID` (measured 2026-11-12 over two copies). A master made from a shape (`Masters.Drop(shape)`) gets new ones, so a stencil assembled from such masters needs the patch too; it works the same on copied masters.

Measured: Visio opens the patched stencil and reports the patched IDs; a drop of an unchanged rebuild reuses the drawing's document master (no `Name.<n>`); a master edited in place (new `UniqueID`, same `BaseID`) still adds `Name.<n>`. Two builds then give the same XML; the zip still differs in timestamps.

## Rows

- Geometry section from nothing (e.g. a line on a group): `AddSection(10)`, then `AddRow(10, -2, 137)` (component row, holds `NoFill`/`NoLine`), then vertex rows `AddRow(10, -2, 138)` = `visTagMoveTo`, `AddRow(10, -2, 139)` = `visTagLineTo`. The first `AddRow` after `AddSection` always becomes the component row, whatever tag it gets. Cells: `Geometry1.X1` = first vertex.
- A group can hold geometry of its own (one sub-shape fewer for a line).
- Text transform cells (`TxtPinX`, `TxtWidth`, ...) exist only after `AddRow(1, 12, 0)` (visSectionObject, visRowTextXForm); check with `RowExists(1, 12, 0)`.
- Shape Data row added without a value: its value is the number `0`, so `LEN(Prop.X)` = 1 and `IF(LEN(Prop.X)>0, ...)` takes the wrong branch. Write `""` for text rows.

## Order of writes

- A formula referring to a row that does not exist yet fails with `#NAME?` (COM exception). Add rows in dependency order (settings before the cells that read them).
- `SETF(GETREF(cell),"="&other)` runs at once: if `other` is empty when written, it writes an empty formula into `cell` and the default formula is gone. Provide the source first (e.g. the document cell `other` reads).
- `LockTextEdit = 1` also blocks code: `Shape.Text` and `Characters.AddCustomFieldU` throw `Requested operation is presently disabled.` Write the text first or unlock around the write.
- `Characters.AddCustomFieldU(formula, 0)` with a reference to a missing cell throws `#NAME?`.
- `Characters.AddCustomFieldU` also throws when the formula is valid but its result is an error right now (`Sheet.1!User.L*2` with `User.L = "?"` -> `#VALUE!`), e.g. a field that reads a value filled in later. Fix: add a placeholder field `""`, then write the field cell: `CellsSRC(visSectionTextField (8), 0, visFieldCell (0)).FormulaForceU = formula`. The cell keeps the formula (`Error` 47), the text is empty until the result is valid, then shows it (`5 A`). Unknown cells (`#NAME?`) and syntax errors are still refused there. `IFERROR(x,"?")` works in a field to show a placeholder instead. Measured 2026-11-03, Visio 2019 C2R x64 build 20527, invisible, COM.
- A field that shows a fallback text while its inputs are placeholders: `IF(OR(STRSAME(User.a,"?"),STRSAME(User.b,"?")),"-",User.a*2&" A")`. `IF` ignores an error in the branch not taken, so `AddCustomFieldU` takes it directly (text `-`, no error); `STRSAME` works on a number too (`User.a = 2.5` -> `5 A`); `OR` with one argument works. Measured 2026-11-07, Visio 2019 C2R x64 build 20527, invisible, COM.

## Group from shapes, master from a shape

Recipe: copy the source shapes, move the copies into one group, freeze/delete rows, make a document master, put an instance in place. Measured 2026-11-02, Visio 2019 C2R x64 build 20527, COM.

```powershell
$sel.Duplicate()                       # window selection: all copies, same offset; the copies are selected now
$group.CellsU('LockCalcWH').FormulaForceU = '1'   # BEFORE adding members
foreach ($s in $members) { $s.Parent = $group }   # keeps the page position
$m = $doc.Masters.Drop($group, 0, 0)   # document master; all formulas copied as local
$m.NameU = 'MyShape'; $m.Name = 'MyShape'
$x = $group.CellsU('PinX').ResultIU; $y = $group.CellsU('PinY').ResultIU
$group.Delete()
$inst = $page.Drop($m, $x, $y)         # pin at x, y; every cell inherited
```

- **`Shape.Parent = group` grows the group** to include the new member, unless the group has `LockCalcWH = 1`. Members get formulas relative to the group size (`Sheet.5!Width*0.246`), so members added earlier stretch with each growth.
- **`Shape.Parent = group` with `LockGroup = 1` on the group does nothing, without error**: the shape stays on the page (`ContainingShape` = page). Set `LockGroup = 0` before adding members, back to 1 after (measured 2026-11-05, same build, visible Visio).
- `Masters.Drop(instance, 0, 0)` of a master instance: the new master's shape has no master link; formulas, deleted rows and sub-shapes are copied.
- `Shape.ReplaceShape(master)` keeps the old shape's local values (`Width` not inherited): use `Page.Drop` + delete instead.
- Instance of a master, inherited row deleted (`DeleteRow`): the row is gone (`CellExistsU` 0), but inherited formulas that read it keep the master formula, still pointing at it. Replace those references by values (write the cells locally) before deleting the row.
- **`DeleteSection` on an instance whose master has that section** (also after deleting all its rows): `Masters.Drop` of the instance, or of a group holding it, brings row 0 of the master's section back. Delete rows only; delete empty sections later in the new master (`Master.Open()`, `DeleteSection`, `Close()`: its shapes are no instances).

## Connector master from the built-in connector

Visio's Dynamic connector is built in (no stencil). Copy it into an own master, then change its cells. Measured 2026-11-08, Visio 2019 C2R x64 build 20527, invisible Visio.

```powershell
$s = $page.Drop($app.ConnectorToolDataObject, 2, 2)   # adds document master 'Dynamic connector'
$s.CellsU('ShapeRouteStyle').FormulaU = '16'           # straight: a 2-point line for icon and drop
$s.CellsU('BeginX').FormulaU = '0 mm';  $s.CellsU('BeginY').FormulaU = '10 mm'
$s.CellsU('EndX').FormulaU   = '20 mm'; $s.CellsU('EndY').FormulaU   = '10 mm'
$s.CellsU('ConFixedCode').FormulaU = '2'               # never reroute: keeps the straight line
$m = $doc.Masters.Drop($s, 0, 0); $s.Delete()
$doc.Masters.ItemU('Dynamic connector').Delete()
$m.NameU = 'MyConnector'; $m.Name = 'MyConnector'      # then Master.Open(), set cells (ShapeRouteStyle 1, ...), Close()
```

- Without route style 16 + `ConFixedCode 2` before `Masters.Drop`, the master reroutes when its route style changes: the stencil line becomes a U or an L.
- `Copyright` (Microsoft's text) is read-only; `HelpTopic` can be emptied. `LayerMember` = `""` + deleting the master's layers removes the `Connector` layer.
- Ends moved by code (`BeginX`/`EndX` cells) on a `ConFixedCode 2` connector keep the old route: a stale segment stays.

## Master icons (Shapes window)

Measured 2026-11-13, Visio 2019 C2R, masters with `IconUpdate` automatic (default):

- The Shapes window draws each icon live from the master's content, scaled to fit, in color. The `<Icon>` stored in `masters.xml` (32 x 32, 4 bit ICO, 766 bytes; built-in `BASIC_M.VSSX` the same) is not what it shows.
- Line weight and line pattern scale down with the master: a 100 x 60 mm rectangle, no fill, 0.5 pt dash-dot (`LinePattern` 4) shows only one faint edge; 40 x 24 mm the same. Visible: 20 x 12 mm, or 2 pt dash-dot, or a solid line (0.5 pt or 2 pt), or a light fill. Small masters with absolute weights (5 mm symbols) look bold.
- So a big, thin, dashed, unfilled master (e.g. a container) needs a smaller default size, a bolder line or a fill to be recognizable.
- `Master.ImportIcon(<hand-made 32 x 32 ICO, 32 bit or 4 bit>)` on a master of an open stencil failed: `Inappropriate source object for this action.` A manual icon would be 32 x 32 anyway.

## Gotchas

- Reading a missing cell (`CellsU('User.X')`) throws `Unexpected end of file.`; check `CellExistsU(name, 0)`.
- A killed script leaves `VISIO.EXE /Automation /Invisible -Embedding`; find it by command line (`Win32_Process`) and stop that PID only, never a user's Visio.
- **PowerShell: a path from `Join-Path` passed to `Documents.OpenEx` hangs Visio without a message** (no dialog; Join-Path wraps its result in a PSObject). Cast it: `[string](Join-Path $dir 'My.vssx')`. Measured 2026-11-08, build 20527.
- PowerShell: a helper function named `R` loses to the built-in alias `r` (`Invoke-History`); pick another name.
