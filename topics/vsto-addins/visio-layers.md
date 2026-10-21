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
- Layers come in without code: _unverified_ connectors drawn with the Connector tool go on a layer `Connector`; masters bring their layers along (below). "On no other layer" is rarely true for shapes of a real drawing.

## Locked layer: what fails

A shape is locked when any of its layers has `Lock` = 1:

- `Window.SelectAll()` skips it ([visio-active-window](visio-active-window.md)); `Window.Select(shape, visSelect)` by code still selects it.
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
- `;` is not allowed: `Layers.Add("a;b")` creates two layers `a` and `b`, then throws `Cannot create object.` Setting `Layer.Name` to a name with `;` throws; writing the formula `"a;b"` into the `visLayerName` cell works (avoid).
- Allowed and kept after save + reopen: `"`, `,`, `'`, `!`, `[]`, `\`, `/`, `|`, `<>`, `*?`, `:`, `#`, `=`, `&`, `%`, `+`, `.`, `{}`, `()`, `~`, `@`, `$`, `^`, umlauts, `Ω€`, leading/trailing spaces, tab, LF, CR, a single space.
- Max 255 characters: longer names are cut **silently**, so two long names can end as the same layer. Empty name: `Add("")` throws `Invalid parameter.`; `Layer.Name = ""` does nothing.
- `Layer.Name = x` changes only the local name, `NameU` stays; `Layer.NameU = x` sets the universal name. Set both when your code finds layers by name.
- `Layer.Name = x` accepts a name another layer already has (any case): two layers with the same name. Check uniqueness yourself.

## Gotchas

- A `Layer` object got by `Layers.Item("dUp")` or `Layers.Add("DUP")` reports the string you passed as `Name` (`dUp`), not the stored name (`Dup`). Read names from `Layers.Item(index)`.
- Removing shapes from all other layers makes hiding by layer reliable, but loses the user's layer assignments for good.

Verified with Visio Pro 2019 C2R x64 by COM automation (`Visio.InvisibleApp`); print by `Document.PrintOut` to `Microsoft Print to PDF` and by `ExportAsFixedFormat`, PDF text checked (2026-10). Visible rule also by hand in Visio. _Unverified:_ clicking a locked shape and pasting in the UI (same messages assumed); which layer color wins (`visLayerColor`) on two colored layers; whether the Layers dialog renames `Name` only or also `NameU`.
