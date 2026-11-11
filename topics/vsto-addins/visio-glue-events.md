---
title: Visio glue and container events - what fires, Undo, paste, delete, keeping glue out
description: "Read when an add-in reacts to glue or container events, writes cells in the handler, handles Undo/paste/delete, or protects a group shape against unwanted glue."
---

# Visio glue and container events

Measured 2026-10-26, Visio 2019 C2R x64, out-of-process C# (events on `Application`, STA, `DoEvents` pump), glue by `Cell.GlueTo`, unglue by setting `BeginX` to a constant.

## ConnectionsAdded / ConnectionsDeleted

- Fire once per glue change with the `Connect`s: `FromSheet`/`FromCell` (`BeginX`, `EndX`) and `ToSheet`/`ToCell` (`Connections.S2.X`; `ToCell.RowNameU` = `S2`, `ToCell.Row` = row index).
- Row names, row types, A/B/C/D: [visio-connection-points.md](visio-connection-points.md).
- Do **not** fire when the glued-to shape moves (glue stays).
- The connector is not rerouted yet when `ConnectionsAdded` fires: its geometry is still the old route; `Width`, `PinX`, `PinY`, `Height`, `LocPinX/Y` `CellChanged` events of the reroute follow in the same action (in-process add-in, `Cell.GlueTo`, Visio 2019 build 20527, 2026-11). Geometry checks (e.g. [touching](visio-spatial-touching.md)) belong after those.
- Undo/Redo fire them again with `Application.IsUndoingOrRedoing = true`. Skip those in the handler: Undo already restores your cell changes.
- A cell written inside the handler (not undoing) lands in the **same Undo step** as the glue: one Undo takes back glue and write; Redo brings both.
- **No event on paste**: pasting a copied 1D shape together with its target keeps the glue (between the copies), but fires no `ConnectionsAdded`. A 1D shape copied alone is pasted unglued, no event.
- **Copy by Ctrl+drag, then F4** (measured 2026-11-10, Visio 2019 build 20527, real mouse, connector glued to points of two shapes, copy dragged onto an equal pair below): the Ctrl+drag copy is glued by Visio. Each F4 (repeat) adds a copy at the same offset that is **not glued**, though its ends lie on the points (deviation 0 to 2.3E-14 mm); `ShapeAdded` fires for it. To heal: in `ShapeAdded`, glue each open end to a connection point within a tiny tolerance (`Page.SpatialSearch[x, y, visSpatialContainedIn | visSpatialTouching, tol, 0]` at the end, then compare `XYToPage` of each connection row, `Cell.GlueTo`); the glue lands in the user's Undo step. In C# `SpatialSearch` is an indexed property (`[...]`, not `(...)`: CS1955).
- Drag modifiers (Visio user's report, not measured by script): while the button is held, any mix of Alt (no snap), Shift (only horizontal or vertical) and Ctrl (copy) may be pressed; the keys down at release count. Script gotchas: Ctrl pressed by `keybd_event` before the mouse-down copies; Ctrl+Shift pressed before the mouse-down did not copy, pressed mid-drag it hung the script with the button down (cause unknown, likely the script; release keys and button in `finally`). Start the drag between the handles of a connector segment, else it drags a vertex.
- **Z-order**: `Shape.Index` of a top-level shape = its place in the page's z-order (1 = back); `BringToFront` makes it the last, one Undo restores the order. Called in a glue or `ShapeAdded` handler it joins the user's Undo step.
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

## Keeping glue out of a built group shape

Goal: connectors glue only to the shape's connection points, nothing glues to the lines inside, inner parts cannot be deleted. Measured 2026-11-02, Visio 2019 C2R x64 build 20527, COM out of process, glue by `GlueTo`/`GlueToPos`.

- `GlueType = 8` (`visGlueTypeNoWalkingTo`): "Do not allow this 2-D shape to be connected to with dynamic glue" ([GlueType cell](https://learn.microsoft.com/en-us/office/client-developer/visio/gluetype-cell-glue-info-section)). Static glue to points/geometry stays possible; programs can establish dynamic glue anyway (same page). Set it on the group and on every sub-shape.
- UI effect (measured 2026-11-05, Visio 2019 build 20527, real mouse, `Document.GlueSettings = 8` = connection points only): end of a dynamic connector dragged onto a group without connection points (on its outline, onto a sub-shape's text, onto a sub-shape's line). Group `GlueType 0`: every drag glued dynamically (`EndX` = `_WALKGLUE(...)`, `Connect` to the group's `PinX`) and the end walked to the side's middle; so the connection-points-only setting does not stop dynamic glue. `GlueType 8` on the group and every sub-shape: none glued, end stayed where dropped. COM `EndX.GlueTo(<group or sub-shape>!PinX)` glued in both cases: test this by mouse, not by COM.
- Inner 1D sub-shape glued to an outside shape (static, `EndX.GlueTo(<connection point>)`): `EndX`/`EndY` become `PAR(PNT(Sheet.n!Connections.X1,...))`; moving the group then stretches the inner line (10 mm -> 32.02 mm), the shape is deformed.
- Same with `GUARD(...)` on the sub-shape's `BeginX/Y`, `EndX/Y` (and `Width`, `Height`, `PinX/Y`, `LocPinX/Y`, `Angle`): `GlueTo` throws nothing but makes no `Connect`; formulas and geometry unchanged after moving.
- Outside line end `GlueToPos(<inner sub-shape>, 0.25, 0)` with `GlueType 8` everywhere: succeeds and Visio adds a connection row (`Connections.X1`) to the inner sub-shape. UI glue to shape geometry: _unverified_.
- `LockGroup = 1` on the group: `Shape.Delete` of a sub-shape and `Selection.Delete` of a sub-selected one (`Window.Select(sub, 3)`, `visSubSelect`) throw `Shape protection, container, and/or layer properties prevent complete execution of this command.`; with `LockGroup = 0` it is deleted. Ungroup: [visio-ungroup](visio-ungroup.md).

## ContainerRelationshipAdded / Deleted

- `RelatedShapePairEvent`: `FromShapeID` = container, `ToShapeID` = member (IDs on `ContainingPage`).
- Fire on `ContainerProperties.AddMember` and when a member is moved out by code. Moving it back in by code adds no membership and no event (UI drag: _unverified_).

Formulas that read the partner and what Visio does with them on copy/delete: [visio-cross-shape-references](visio-cross-shape-references.md).
