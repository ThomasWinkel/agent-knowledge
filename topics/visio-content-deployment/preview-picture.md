---
title: Preview picture of a template or stencil
description: Read when a Visio template/stencil/drawing shows a blurry or empty picture in File > New or Explorer - where the picture lives, what Visio writes, replace it, keep it in drawings.
---

# Preview picture of a template or stencil

Visio File > New and the Windows Explorer (Visio's thumbnail handler) show the package part `docProps/thumbnail.emf`.

## Key facts

Measured on Visio 2019 C2R (Office16):

- On save, Visio writes a snapshot of page 1 there: a 96 x 68 px bitmap wrapped in an EMF (one `EMR_BITBLT` record). In a gallery tile it is scaled up and blurry. For an empty template page it shows almost nothing.
- Built-in templates (e.g. `Visio Content\1033\BASICD_M.VSTX`) carry a drawn picture instead: EMF+ dual, square, 915 x 915, flat shapes in one or two colors, no page snapshot.
- `Page.Export("<file>.emf")` writes vector EMF+ dual (bounds = page incl. line widths). So a picture drawn on a square page in Visio is a fitting replacement.
- Replacing the part in the saved file works: Windows shell thumbnail shows the new picture, the template opens unchanged. `[Content_Types].xml` (`emf` default) and `_rels/.rels` already point to it; keep the part name.
- Visio overwrites the part on its next save of that file. Patch it after the last `SaveAs` in a build script.
- Stencils (`.vssx`): Visio's UI shows no picture of a stencil (More Shapes menu and the stencil list of the Shapes window are text only; Visio 2019). Only the Explorer and file dialogs show `docProps/thumbnail.emf`. A stencil saved by Visio gets an empty EMF there (no drawing records), shown as a blank white tile. Built-in stencils (all 725 `*.VSSX` in `Visio ContentC3`) have no such part, so the Explorer shows the Visio file icon. Fix: replace it as above (dropping the part and its relationship: _Unverified_).
- File > New reads it via the content cache (`thumbs.dat`, see [index](index.md)): after a change, bump `ConfigChangeID` or delete the cache with Visio closed. _Unverified_ that the gallery shows the replaced picture.

## Preview of drawings made from a template

Measured 2026-11-13, Visio 2019 C2R, invisible Visio by COM:

- Document ShapeSheet cells (`visio/document.xml`): `PreviewQuality` (0 draft = the 96 x 68 px bitmap, 1 detailed = vector EMF+ of page 1, sharp), `PreviewScope`, `LockPreview`. COM: `Document.SavePreviewMode` 1 = draft (default), 2 = detailed, 0 = no preview part at all; 3 = `Invalid parameter.`
- A new drawing takes `PreviewQuality` from its template (template saved with mode 2 -> drawing mode 2). So a template can make all its drawings save a sharp page preview.
- `LockPreview` is not taken over (template `1` -> new drawing `0`). Set by code in a new drawing without a picture: no preview part is saved, now or later.
- `Document.CopyPreviewPicture(<other document>)` copies the preview of another open document (e.g. the template, `OpenEx` with `visOpenRO + visOpenHidden`). With `LockPreview` 1 the drawing keeps that picture on every later save. Also `Document.PreviewPicture` (IPictureDisp, get/set).
- Visio keeps a drawing's file locked while it is open: the package cannot be patched from outside then.

## Recipe (PowerShell, Visio COM)

```powershell
# 1. Draw on a square page and export.
$doc = $app.Documents.AddEx('', 1, 0)            # visMSMetric
$page = $doc.Pages.Item(1)
$page.PageSheet.CellsU('PageWidth').FormulaU  = '100 mm'
$page.PageSheet.CellsU('PageHeight').FormulaU = '100 mm'
# ... DrawRectangle / DrawPolyline / DrawOval (internal units: inch), thick lines (>= 5 pt): 1 mm ~ 1 px at gallery size
$page.Export('C:\temp\preview.emf')
$doc.Saved = $true; $doc.Close()

# 2. After the template is saved and closed: replace the part.
Add-Type -AssemblyName System.IO.Compression, System.IO.Compression.FileSystem
$zip = [IO.Compression.ZipFile]::Open('C:\out\MyTemplate.vstx', 'Update')
try {
    $zip.GetEntry('docProps/thumbnail.emf').Delete()
    $s = $zip.CreateEntry('docProps/thumbnail.emf').Open()
    $b = [IO.File]::ReadAllBytes('C:\temp\preview.emf'); $s.Write($b, 0, $b.Length); $s.Close()
} finally { $zip.Dispose() }
```

## Gotchas

- `DrawPolyline` from PowerShell: `[double[]]$xy = @(...)` passed directly works; `$page.DrawPolyline([ref]$xy, 0)` with a `New-Object 'double[]'` array failed ("Cannot convert ... value of type \"psobject\" to type \"Object\"").
- Design hint (not measured): tiles are small (~64 px); bold, flat shapes read better than thin real symbols.
