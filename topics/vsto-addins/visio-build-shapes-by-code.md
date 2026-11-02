---
title: Building Visio shapes and stencils by code
description: Read when code builds Visio masters, groups or a stencil from scratch by COM (geometry rows, text fields, Shape Data, invisible Visio) - row tags, order of writes, errors.
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

## Gotchas

- Reading a missing cell (`CellsU('User.X')`) throws `Unexpected end of file.`; check `CellExistsU(name, 0)`.
- A killed script leaves `VISIO.EXE /Automation /Invisible -Embedding`; find it by command line (`Win32_Process`) and stop that PID only, never a user's Visio.
- PowerShell: a helper function named `R` loses to the built-in alias `r` (`Invoke-History`); pick another name.
