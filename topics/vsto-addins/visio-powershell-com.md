---
title: Visio COM from PowerShell - gotchas
description: Read when a PowerShell 5.1 script drives Visio by COM - pipeline-wrapped arguments hang Visio, COM collections unroll into items, typed arrays for DropMany/DrawPolyline, alias clashes.
---

# Visio COM from PowerShell

Windows PowerShell 5.1, Visio 2019 C2R x64 build 20527 (2026-11). Each of these failed silently or with a misleading error.

## Wrapped objects as COM arguments

An object that came through a pipeline (`Where-Object`, `ForEach-Object`, `Select-Object`) or from some cmdlets (`Join-Path`, `New-Object`) is wrapped in a `PSObject`. Passed as an **argument** to a Visio method it can hang the call: no dialog, no error, the script blocks until killed.

- `Window.Select($s, 2)` with `$s = @($page.Shapes | Where-Object { ... })[0]` or `... | Select-Object -First 1`: blocked > 30 s on a blank drawing with 2 rectangles (killed; Visio's main thread stayed busy). The same shape from a `foreach` statement or as `$s.psobject.BaseObject`: 1-20 ms.
- `Documents.OpenEx((Join-Path $dir 'My.vssx'), 2)`: hang; `[string](Join-Path ...)` works.
- `New-Object 'double[]' n` passed `[ref]` to `DrawPolyline`/`DropMany`: `Cannot convert ... value of type "psobject" to type "Object"`; `[double[]]::new(n)` or `[double[]]$xy = @(...)` works.
- Calls **on** a wrapped object work (`$s.Delete()`, `$s.CellsU('PinX')`, `$addin.Connect = $false`).

Check: `$x -is [psobject]` is `True` only for a wrapped object (raw COM object: `False`). Fix: `$x.psobject.BaseObject`, a `[string]` cast for text, or loop with the `foreach` statement (yields raw objects):

```powershell
$hit = $null; foreach ($s in $page.Shapes) { if ($s.NameU -eq 'Box') { $hit = $s } }   # raw: safe as argument
$win.Select($hit, 2)
```

## Collections unroll into their items

A function or script block that outputs a COM collection (`$win.Selection`, `$page.Shapes`) writes its items, not the collection: the caller gets `object[]` of shapes, or a single shape. Keep the collection with the comma operator:

```powershell
function All($page) { $w = $app.ActiveWindow; $w.DeselectAll(); foreach ($s in $page.Shapes) { $w.Select($s, 2) }; , $w.Selection }
$sel = All $page          # __ComObject, $sel.Count = 100
$sel.Duplicate()          # one Selection.Duplicate
```

- A method called on the unrolled array runs on every item (member enumeration), without any error: `$sel.Duplicate()` then called `Shape.Duplicate` 100 times = 100 Undo units (one COM `Undo` took back one copy), 2.9 s instead of 1.1 s for one `Selection.Duplicate` (100 shapes, with an add-in's handlers; measured 2026-11-14).

## Arrays for DropMany

`Page.DropMany`: masters as `[object[]]`, coordinates `[double[]]::new(2 * n)`, the out array initialized (`$ids = [int16[]]@()`), all three passed `[ref]`. A `$null` out array gives `Object reference not set`.

```powershell
$m = [object[]](@($master) * $n); $xy = [double[]]::new(2 * $n); $ids = [int16[]]@()
# fill $xy: x0, y0, x1, y1, ... (inches)
[void]$page.DropMany([ref]$m, [ref]$xy, [ref]$ids)
```

## Names of helper functions

Built-in aliases beat functions of the same name: `r` (`Invoke-History`), `group` (`Group-Object`), `lp` (`Out-Printer`), `move` (`Move-Item`). Name helpers e.g. `ReadCell`, `ToLParam`, `MoveTo`.
