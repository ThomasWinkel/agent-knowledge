---
title: Visio layers by code
description: "Read when Visio code creates, names, hides, locks or deletes layers, or shapes are on several layers - which layer wins for visible/print/lock, names, copy/paste and master drop."
---

# Visio layers by code

A shape can be on any number of layers of its page (`Shape.LayerCount`, `Shape.Layer[i]`; `Layer.Add(shape, 0)`, `Layer.Remove(shape, 0)`). Membership is stored as layer indexes in the shape's `LayerMember` cell (`"0;1"`), not names. Layer cells: `Layer.CellsC[(short)VisCellIndices.visLayerVisible]` etc.

## Shape on several layers

| Layer cell | Shape is ... | Rule |
|---|---|---|
| `visLayerVisible` | visible | while **any** of its layers is visible |
| `visLayerPrint` | printed, in PDF export | while **any** of its layers has `Print` = 1, **independent of `Visible`**: a hidden layer with `Print` = 1 prints its shapes |
| `visLayerLock` | locked | as soon as **any** of its layers is locked, even if it is visible through another layer |

- To hide shapes by a layer, also set `Print` = 0, and keep the shapes on no other visible or printing layer.
- `Layer.Delete(1)` deletes only shapes on no other layer, of any kind ([Layer.Delete](https://learn.microsoft.com/en-us/office/vba/api/visio.layer.delete), Remarks). To delete by your own rule, delete the shapes yourself, then `Layer.Delete(0)`.
- Layers come in without code: a connector made by `Shape.AutoConnect` goes on a layer `Connector` (_unverified:_ same for the Connector tool); masters bring their layers along (below). "On no other layer" is rarely true for shapes of a real drawing.

## Layer color

`visLayerColor` = `255` means none (default): the shapes keep their own colors. Any other value (`RGB(0,255,0)`) draws the lines of the layer's shapes in that color; a white fill stays white. `visLayerColorTrans` (`30%`) sets its transparency.

- A shape on **two layers**, one colored and the other uncolored (`255`) or of another color (either order): **no layer color**, the shape shows its own colors. So a layer color marks only shapes on no other layer (or, _unverified_, on layers of the same color).

## Locked layer: what fails

A shape is locked when any of its layers has `Lock` = 1:

- `Window.SelectAll()` skips it ([visio-active-window](visio-active-window.md)); `Window.Select(shape, visSelect)` by code still selects it.
- Hiding or locking a layer deselects its shapes (`Window.Selection` drops them; a selected shape on another layer stays selected).
- Setting `Shape.Text` throws `Requested operation is presently disabled.`; setting a cell (`PinX`) works; `Layer.Add`/`Remove` (also on the locked layer) work.
- **Paste, `Shape.Duplicate` and `Page.Drop(master)` fail** when the target page has a locked layer with the name of one of the shape's layers: `Attempting to change one or more locked layers.` (paste, duplicate), `Inappropriate source object for this action.` (drop). `Shape.Copy` works.
- So locking hidden layers adds no protection (hidden shapes cannot be clicked, `SelectAll` skips them) but blocks pasting and dropping shapes that use the layer.

## Copy, paste, drop

Layers travel with the shape **by name**:

- Target page has no layer of that name: Visio creates it with the **source layer's cells** (visible, print, lock, color). A shape copied from a page where its layer is hidden arrives on a new hidden layer: invisible after paste.
- Target page has a layer of that name, **ignoring case** (`mylayer:a` for `MyLayer:A`): the shape joins it; the layer keeps the target's cells and name.
- Matched by local `Name`, not `NameU`: a target layer `Name`=`A`, `NameU`=`X` takes a shape from layer `A`; a target layer `Name`=`X`, `NameU`=`A` does not (Visio adds a new layer `A`).
- Same for: paste into another drawing, `Shape.Duplicate`, sub-shapes of a pasted group, `Page.Duplicate` (copies all layers). A master made from a shape (`Masters.Drop(shape, ...)`) stores its layers with their cells; dropping it creates them the same way.

## Names

- `Layers.Add(name)` with an existing name, ignoring case, returns the existing layer (no new one). `Layers.Item(name)` searches the local `Name` ignoring case; `Layers.ItemU(name)` searches `NameU`.
- Case rule = .NET `StringComparer.OrdinalIgnoreCase` (one character to one): `ä`=`Ä`, `é`=`É`; `ß` is not `ss`, `ẞ` or `SS` (`straße` and `STRASSE` are two layers). Differs only for rare letters: Visio also matches `ı`=`I` and `ǅ`=`ǆ`, `OrdinalIgnoreCase` does not; `i`/`İ` differ in both.
- `;` is not allowed: `Layers.Add("a;b")` creates two layers `a` and `b`, then throws `Cannot create object.` Setting `Layer.Name` to a name with `;` throws; writing the formula `"a;b"` into the `visLayerName` cell works (avoid).
- Allowed and kept after save + reopen: `"`, `,`, `'`, `!`, `[]`, `\`, `/`, `|`, `<>`, `*?`, `:`, `#`, `=`, `&`, `%`, `+`, `.`, `{}`, `()`, `~`, `@`, `$`, `^`, umlauts, `Ω€`, leading/trailing spaces, tab, LF, CR, a single space.
- Max 255 characters: longer names are cut **silently**, so two long names can end as the same layer. Empty name: `Add("")` throws `Invalid parameter.`; `Layer.Name = ""` does nothing.
- `Layer.Name = x` changes only the local name, `NameU` stays; `Layer.NameU = x` sets the universal name. Set both when your code finds layers by name.
- **Undo of `Layer.Name = x` is broken**: afterwards the layer still reports the new `Name`, `NameU` is empty, its cells throw `An exception occurred.` (also after a dropped undo scope); only save + reopen shows the old name. Undo of `Layer.NameU = x` and of a formula in the `visLayerName` cell is clean (also inside nested and dropped scopes, case-only renames too). Rename so Undo works:
  ```csharp
  layer.NameU = newName;   // first: the Layer object is stale after the next line
  page.Layers[index].CellsC[(short)VisCellIndices.visLayerName].FormulaU =
      "\"" + newName.Replace("\"", "\"\"") + "\"";
  ```
  The `visLayerNameUniv` cell cannot be set by formula (`An exception occurred.`). Another clean way: add a layer with the new name, copy its cells, move the shapes, `Delete(0)` the old one. Undo scopes: [visio-undo-scopes](visio-undo-scopes.md).
- `Layer.Name = x` accepts a name another layer already has (any case): two layers with the same name. Check uniqueness yourself.

## Gotchas

- A `Layer` object got by `Layers.Item("dUp")` or `Layers.Add("DUP")` reports the string you passed as `Name` (`dUp`), not the stored name (`Dup`). Read names from `Layers.Item(index)`.
- Removing shapes from all other layers makes hiding by layer reliable, but loses the user's layer assignments for good.

Verified with Visio Pro 2019 C2R x64 by COM automation (`Visio.InvisibleApp`); print by `Document.PrintOut` to `Microsoft Print to PDF` and by `ExportAsFixedFormat`, PDF text checked (2026-10). Visible rule also by hand in Visio. Layer color by `Page.Export` to PNG, pixels checked. _Unverified:_ clicking a locked shape and pasting in the UI (same messages assumed); layer color on text, on colored fills and in print; whether the Layers dialog renames `Name` only or also `NameU`.
