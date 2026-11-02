---
title: Visio connection point rows - row types, A/B/C/D, DirX/DirY/Type, row names
description: Read when adding connection points by code, storing data in A/B/C/D cells, naming rows, or relying on connector direction or 2D point-to-point glue.
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

## Row names

- Allowed: `a-z A-Z 0-9 _` only; refused (`1.1`, `L+`, `-`, `Ä1`, `A 1`, `+24V`, `X1:3`): `Cell names may consist of the characters a-z, A-Z, 0-9, and underscore symbol (_).` Names starting with a digit (`13`, `0V`) work. Unique ignoring case.
- Naming one row names all others `Row_<n>`. `AddRow(7, 0, tag)` inserts at index 0 in an unnamed section, but appends in a named one.
- Glue formula `PAR(PNT(Sheet.1!Connections.A2.X,Sheet.1!Connections.A2.Y))` and formulas like `Connections.A2.D` follow a rename, rows inserted or deleted before them, and the first naming (`Connections.X2` -> `Connections.Row_3.X`).
- Glued row deleted: the glued end and formulas referencing the row become constants (`70 mm`, `"pin2"`).
- `Connect.ToCell`: `RowNameU` = row name (empty in an unnamed section), `Row` = index. Glue events: [visio-glue-events.md](visio-glue-events.md).
