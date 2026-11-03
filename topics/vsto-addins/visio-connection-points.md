---
title: Visio connection point rows - row types, A/B/C/D, DirX/DirY/Type, row names
description: Read when adding connection points by code, storing data in A/B/C/D cells, naming rows, or relying on connector direction or 2D point-to-point glue (chain limit 40, F4 after a glued copy).
---

# Visio connection point rows

Measured 2026-10 and 2026-11, Visio 2019 C2R x64, COM, `Cell.GlueTo`. Section `visSectionConnectionPts` (7).

## Row types

- Tags (interop `VisRowTags`): `visTagCnnctPt` 153, `visTagCnnctNamed` 185, `visTagCnnctPtABCD` 186, `visTagCnnctNamedABCD` 187. Older lists giving 154 are wrong (`AddRow` fails: `Operation not allowed for this row type.`).
- The type only renames columns 2-6: `DirX, DirY, Type, AutoGen, Prompt` vs. `A, B, C, D, Prompt` (`visCnnctA` = `visCnnctDirX` = 2, `visCnnctD` = `visCnnctAutoGen` = 5). Both names resolve in either type (`Connections.S2.D` on a normal row is its `AutoGen`; unnamed: `Connections.D3`, `Connections.DirX[3]`).
- Types may be mixed in one section; switch per row: `shape.RowType(7, row) = 186` (PowerShell: same syntax).
- **ABCD row = no glue meaning**: A/B/C hold any text; connectors ignore them (no leaving direction); the point cannot be the source of 2D glue. Use ABCD rows for data on a point, normal rows for direction or 2D glue.
- Normal row: text written into `AutoGen` (D) becomes `TRUE`. `Prompt` (column 6) holds text in both types.

## DirX/DirY/Type (normal rows)

- A dynamic connector glued to the point leaves **opposite** to (DirX, DirY): `-1,0` -> right, `1,0` -> left (detour around the shape), `0,1` -> down, `0,-1` -> up, `0,0` -> free.
- Type `1` (outward) or `2` (inward + outward): point can be the source of 2D glue: `src.CellsSRC(7, r, 0).GlueTo(tgt.CellsSRC(7, r2, 0))` writes a `PinX`/`PinY` formula (`PNTX(LOCTOPAR(PNT(Sheet.5!Connections.X1,...),Sheet.5!EventXFMod,EventXFMod))+5 mm`). Type `0` source or any ABCD row: `Inappropriate source object for this action.`; `PinX.GlueTo` too.
- COM `GlueTo` of a 1D end to a point accepted every Type (0, 1, 2, text). UI glue rules per Type: unverified.

## 2D glue: direction decides the angle, docking a 1D shape

Measured 2026-11-03, Visio 2019 build 20527, COM `GlueTo`; target shape moved, turned 90 deg, flipped (`FlipX`).

- 2D glue of a point (Type 1/2) to a target point: if either point has `DirX/DirY 0,0`, only `PinX`/`PinY` are glued; the shape follows moves but never turns. With directions on both, `Angle = ANGLETOPAR(<a>,<target>!EventXFMod,EventXFMod)` is glued too (2 `Connect`s): **opposite** directions (`0,-1` on the shape, `0,1` on the target; or `-1,0` / `1,0`) give angle 0, the **same** direction gives 180 deg (shape upside down).
- Target flipped: the 2D-glued shape is turned 180 deg, not mirrored; with an off-center glue point it lands at the mirrored place but upside down, its glue point no longer on the target point.
- Turning the glued shape itself drops the angle glue, the position glue stays.
- 1D shape glued at both ends (`PAR(PNT(..))`): follows move, turn and flip if `Width`/`Angle` are the 1D defaults; ends on points that do not fit the length stretch the shape. With `Angle = GUARD(0DEG)` the body stays horizontal while the glued ends turn with the target: the ends lie outside the body. A self-repair `IF(<length>=Width,"",SETF("EndX","BeginX+Width*COS(Angle)")+SETF(..))` unglues an End that does not fit.
- UI drag of the 1D body (real mouse, Begin dropped 0.4 mm off a point): ends on points that fit -> both glued; End 0.5 mm off a point while `EndX` is tied by formula (`BeginX+Width*COS(Angle)`, also `GUARD(..)`) -> only Begin glued, End formula kept, shape not deformed; plain 1D shape -> both glued, stretched. 2D shape with a directed point -> `PinX` + `Angle` glued as by `GlueTo`. Glue to geometry on (`Document.GlueSettings` + 32), target without points: Visio adds a connection row (`X1`) to the target at the drop point and glues Begin to it.

## 2D glue chains stop after 40 shapes

Measured 2026-11-03, Visio 2019 build 20527, COM: shapes glued side by side, each to the one before (`B.Connections.L.X.GlueTo(A.Connections.R.X)`).

- Chain of 30 follows the first shape (move by `PinX` formula or `Selection.Move`, `Rotate90` with direction set). Chain of 60/100: shapes 1-40 follow, shape 41 on stays put. Same for groups and plain rectangles, both direction variants; reopening the file does not catch up. Writing shape 41's `PinX` formula again pulls 41-60 in (rest of a 60 chain); the next move leaves 41 behind again.
- Cause: the transform dependency of `LOCTOPAR`, not the `Connect`. The glue formula written by hand without glue stops at 40 too, also with `prev!EventXFMod` as both refs. Plain cell formulas have no such limit: `PinX = prev!PinX-prev!LocPinX+prev!Connections.R.X+<half width>` (and `PinY` alike) followed a chain of 60, a plain `PinX` reference chain of 100.
- Source: Mark Nelson (Microsoft), comment of 2007-06-17 on [When 2D Glue Breaks](https://www.visguy.com/2007/06/14/when-2d-glue-breaks/): 2D glue relies on a circular reference; Visio's hard-coded limit of recalc iterations for circular references is 40. Visio 2003/2007 broke after 10 shapes.
- Copy of a glued 2D shape (`Selection.Duplicate`, `DoCmd(visCmdUFEditDuplicate)` 1024): the copy is not glued, `PinX`/`PinY` constant, default duplicate offset. UI drag of the first shape (real mouse): same limit 40. Area-selected chain dragged by one shape: 10 all move, glue kept; 50 (dragged by the first, a middle or the last): shapes 41-50 stay, so selecting the chain does not help. One middle shape dragged alone: its glue to the one before is gone, the rest follow it; on drop a normal row also glues to an ABCD row (`visTagCnnctPtABCD`) of a neighbor.
- Ctrl+drag copy + F4 (Repeat) by the real mouse (measured 2026-11-03, Visio 2019, new metric drawing): without glue F4 repeats the drop offset exactly. If the dropped copy glued to a point of A (copy beside A, `PinX = PNTX(LOCTOPAR(PNT(A!Connections.R.X,..),..))+<half width>`), every F4 copy is glued to the **same** point of A with `+0 mm`: all land on top of each other, pin on A's point, not in a row. With directions set, their `Angle` is glued to the page (`ANGLETOPAR(0 deg,ThePage!EventXFMod,EventXFMod)`), not to A.
- Glue only as a positioning aid (snap + highlight while dragging, no chain), measured 2026-11-03 in a VSTO add-in: a `ConnectionsAdded` handler (`EventList.AddAdvise`, skipped while `Application.IsUndoingOrRedoing`) moves the shape so its own point sits on the target point, then writes `PinX`/`PinY`/`Angle` as constants (`Cell.ResultIU`): the glue is gone. Same Undo step as the drop (one Ctrl+Z removes a Ctrl+drag copy, Ctrl+Y brings it back placed). With a check "target point taken by another shape -> next free point along the row", F4 after the Ctrl+drag copy adds each copy at the end of the row.

## Row names

- Allowed: `a-z A-Z 0-9 _` only; refused (`1.1`, `L+`, `-`, `Ä1`, `A 1`, `+24V`, `X1:3`): `Cell names may consist of the characters a-z, A-Z, 0-9, and underscore symbol (_).` Names starting with a digit (`13`, `0V`) work. Unique ignoring case.
- Naming one row names all others `Row_<n>`. `AddRow(7, 0, tag)` inserts at index 0 in an unnamed section, but appends in a named one.
- Named ABCD rows: `AddNamedRow(7, name, 186)` fails (`Operation not allowed for this row type.`). Add the first row with `AddRow(7, -2, 186)`, then set `CellsSRC(7, row, 0).RowNameU`; once the section is named, `AddRow` with 186 (or 153) fails the same way: use 187 (`visTagCnnctNamedABCD`; 185 for normal rows), then rename `Row_<n>`.
- Glue formula `PAR(PNT(Sheet.1!Connections.A2.X,Sheet.1!Connections.A2.Y))` and formulas like `Connections.A2.D` follow a rename, rows inserted or deleted before them, and the first naming (`Connections.X2` -> `Connections.Row_3.X`).
- Glued row deleted: the glued end and formulas referencing the row become constants (`70 mm`, `"pin2"`).
- `Connect.ToCell`: `RowNameU` = row name (empty in an unnamed section), `Row` = index. Glue events: [visio-glue-events.md](visio-glue-events.md).
