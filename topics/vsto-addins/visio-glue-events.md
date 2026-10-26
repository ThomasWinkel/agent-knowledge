---
title: Visio glue and container events - what fires, Undo, paste, delete
description: "Read when an add-in reacts to glue (ConnectionsAdded/Deleted) or container membership events, writes cells in the handler, or must handle Undo, paste and delete."
---

# Visio glue and container events

Measured 2026-10-26, Visio 2019 C2R x64, out-of-process C# (events on `Application`, STA, `DoEvents` pump), glue by `Cell.GlueTo`, unglue by setting `BeginX` to a constant.

## ConnectionsAdded / ConnectionsDeleted

- Fire once per glue change with the `Connect`s: `FromSheet`/`FromCell` (`BeginX`, `EndX`) and `ToSheet`/`ToCell` (`Connections.S2.X`; `ToCell.RowNameU` = `S2`, `ToCell.Row` = row index).
- Connection row names: adding one named row names all rows of the section (`Row_1`, `Row_2`, ...). In a section without names `ToCell` is `Connections.X3` (`RowNameU` empty, `Row` 2; other cells `Connections.Y3`, `Connections.DirX[3]`). The row type only renames columns 2-6 (`DirX, DirY, Type, AutoGen, Prompt` vs. `A, B, C, D, E`), and both names resolve in either type (`Connections.S2.D` on a normal row is its `AutoGen` cell; unnamed: `Connections.D3`). Measured 2026-10, Visio 2019 C2R x64.
- Do **not** fire when the glued-to shape moves (glue stays).
- Undo/Redo fire them again with `Application.IsUndoingOrRedoing = true`. Skip those in the handler: Undo already restores your cell changes.
- A cell written inside the handler (not undoing) lands in the **same Undo step** as the glue: one Undo takes back glue and write; Redo brings both.
- **No event on paste**: pasting a copied 1D shape together with its target keeps the glue (between the copies), but fires no `ConnectionsAdded`. A 1D shape copied alone is pasted unglued, no event.
- **No event on delete**: deleting the glued-to shape fires only `BeforeShapeDelete`; the 1D shape is unglued (`BeginX` becomes a constant) without `ConnectionsDeleted`.
- -> Handle `ShapeAdded`/paste and `BeforeShapeDelete` too, or offer a document-wide refresh that rebuilds everything from `Shape.Connects`.

```csharp
app.ConnectionsAdded += connects =>
{
    if (app.IsUndoingOrRedoing) return;
    foreach (Visio.Connect c in connects)
    {
        string row = c.ToCell.RowNameU;   // "S2"; empty for unnamed rows (then c.ToCell.Row)
        // write into c.FromSheet ...
    }
};
```

## ContainerRelationshipAdded / Deleted

- `RelatedShapePairEvent`: `FromShapeID` = container, `ToShapeID` = member (IDs on `ContainingPage`).
- Fire on `ContainerProperties.AddMember` and when a member is moved out by code. Moving it back in by code adds no membership and no event (UI drag: _unverified_).

Formulas that read the partner and what Visio does with them on copy/delete: [visio-cross-shape-references](visio-cross-shape-references.md).
