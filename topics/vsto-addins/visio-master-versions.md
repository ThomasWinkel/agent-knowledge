---
title: Visio master versions - stencil edits and documents with old copies
description: Read when shipping a changed stencil master, or when code must bring a new master version into drawings using the old one - BaseID, UniqueID, MatchByName, edit in place, sync, Undo.
---

# Visio master versions

A drawing holds its own copy of every master it uses (document stencil). A changed stencil master never reaches that copy by itself.

| Edit of the stencil master | `BaseID` | `UniqueID` |
|---|---|---|
| Edit in place (`Master.Open()`, change, `Close()`; UI: Edit Master Shape) | kept | new |
| Rename (`Name`/`NameU`) | kept | new |
| Delete master, drag the shape back into the stencil (new master) | **new** | new |
| Set `MatchByName` + save | kept | new |

Drop of the changed master into a drawing that has the old one (`Page.Drop(stencilMaster, x, y)`):

- Default: a second document master `Board.4` (next free number) with the new content; old instances keep the old master. Every further version adds one more (`Board.5`, ...). Same for both ways of editing.
- `MatchByName = true` on the **document** master: the drop reuses the old document master by name, the new content never arrives. The match uses the document master's local `Name` (it worked with `Name` = stencil name and a different `NameU`; measured 2026-11-09). Setting `MatchByName` gives the document master a new `UniqueID`.

Bring a new version into a drawing: edit the **document** master (`docMaster.Open()`, change cells, `Close()`): every instance that inherits the cell gets the new value at once; instances with a local value keep it.

```csharp
Master copy = doc.Masters.ItemU["Board"].Open();
copy.Shapes[1].CellsU["User.MyData"].FormulaU = "\"v2\"";
copy.Close();   // commits; instances inherit "v2", the document master gets a new UniqueID
```

## What instances keep when code edits the document master

Instances bind to the master's shapes by shape ID (`Shape.MasterShape`), and to its rows by the row itself (not its name, not its index). So edit the shapes of the master copy in place; never replace them.

| Change in `Master.Open()` copy, then `Close()` | Instances |
|---|---|
| Delete the top shape, `copy.Drop(newShape, 0, 0)` | broken: new ID, every instance loses its master, rows and sub-shapes (plain empty group); glue turns into constant coordinates |
| Cell formula changed (top shape or sub-shape, text) | inherited cells follow; local values (also in sub-shapes, local text) kept |
| Row deleted before another row (index shift) | local value and glue stay on their row |
| Row renamed (`Cell.RowNameU`, User/Prop/Connections) | local value stays; glue formula of a connector is rewritten to the new name |
| Row deleted | gone in every instance with its local value; a connector glued to a deleted connection row is unglued and keeps its position |
| Row added (`AddNamedRow`) | added to every instance |
| Sub-shape deleted | gone in every instance with its local values; glue to it lost as above |
| Sub-shape added (`groupInCopy.DrawRectangle(...)`) | added to every instance |
| Sub-shape z-order (`BringToFront`) | follows |

- `Shape.AddRow(section, 0, tag)` in `Prop` and `Connections` sections appends at the end (returns the last index), it does not insert at row 0. New rows can only go to the end of these sections.
- Swapping two row names through a temporary name (`A`->`T`, `B`->`A`, `T`->`B`) left two rows named `T` in between and swapped formulas at the end. Delete and add rows instead of swapping names.
- `Shape.ReplaceShape(newMaster, 0)` is no substitute: new shape ID and name (`UniqueID` gone); glue and references from other shapes follow, but cells the old instance inherited become local with the old master's value (the new value never arrives), and local values in sub-shapes are lost.
- Events: an edit of a master with 500 instances fired 0-10 `CellChanged`, one `BeforeShapeDelete` for a deleted sub-shape and no `ShapeAdded` for added sub-shapes: event handlers do not see the instances change.
- Time with 500 instances (add-in loaded): 120-430 ms per `Open`/`Close` (one value, row add/delete, sub-shape add/delete, sub-shape geometry); Undo 30-310 ms.

## Sync a document master to a newer version

Brings a new version of a master (same `BaseID`, other `UniqueID`, e.g. a changed stencil master) into a drawing: make the document master equal to it in place, the instances follow and keep local values, glue and shape IDs.

1. `copy = docMaster.Open()`. Pair shapes: top shape with top shape, sub-shapes by `NameU` without Visio's `.<n>` suffix (recursive). Keep a map new-master shape ID -> copy shape ID.
2. Per shape pair, sections: 1 (object; fixed row indices, probe `RowExists` 0..120), 3-9, geometry (`visSectionFirstComponent + i`), 240-255; also the page sheets (`Master.PageSheet`).
   - User, Prop, Actions, Controls, Connections, Hyperlink: pair rows by `CellsSRC(sec, row, 0).RowNameU`; `DeleteRow` what the new master lacks, `AddNamedRow(sec, name, newShape.RowType[sec, row])` what is missing (lands at the end).
   - Other sections: pair by index. A different row count, geometry section count or sub-shape set cannot be fixed without losing instance data: report it.
3. Per cell: differs in `FormulaU` or `IsInherited` -> `FormulaForceU = formula` (passes `GUARD`), or `""` if inherited in the new master. Rewrite `Sheet.<id>!` from new-master IDs to copy IDs (not inside quoted text).
4. Check the structure in a first pass and change only without problems, so no master stays half synced; then `Close()`. Copy `Master.Prompt` too.

```csharp
var copy = docMaster.Open();
var cell = copy.Shapes[1].CellsSRC[sec, row, i];
var from = newMaster.Shapes[1].CellsSRC[sec, row, i];
if (cell.FormulaU != from.FormulaU || cell.IsInherited != from.IsInherited)
    cell.FormulaForceU = from.IsInherited != 0 ? "" : from.FormulaU;
copy.Close();
```

- Result (6 masters, 23 instances): file export of each document master equal to the new one except master `ID`, `UniqueID` and the place of added rows; every local value of the instances kept; 0 error cells.
- After `Close` the document master has a Visio-made `UniqueID` (e.g. `{0A74B211-0009-0000-8E40-00608CF305B2}`), not the new master's: a "current?" check by `UniqueID` still says stale. Patch it in the saved file (`visio/masters/masters.xml`) if it matters.
- Time: in-process (add-in) 0.4-0.8 s per master (group + 1 sub-shape, ~40 User/Prop rows). Out of process from PowerShell ~10 ms per COM call (also from C# compiled by `Add-Type`), 30-60 s per master.
- `Master.PageSheet.Characters` throws "Cannot create object." `Characters.Text` of two such masters with text fields differed although both printed the same: text is not compared this way (cause not found).

## Undo

- Wrap every `Master.Open()`/`Close()` in its own `BeginUndoScope`/`EndUndoScope`. Then each Undo takes back one edit, master and instances together (deleted rows and sub-shapes come back with their local values), and Redo works, also with `FormulaU = ""` resets of instance cells in the same scope (measured up to 500 instances).
- Without own scopes, several master edits in a row: the first Undo took back only the last edit; further Undo calls changed nothing.
- _Unverified:_ a Redo failed with `0x86DB089A` after a longer sequence (six scoped edits incl. sub-shape add/delete, five Undo, one more scope, Undo, Redo); a shorter sequence redid fine. Cause unknown.

## Gotchas

- `BaseID` is the only ID that survives edits; it identifies "the same master in another version". The delete-and-redrag way of editing loses it; then only the name is left to match.
- A master made from a shape (`Masters.Drop(shape, 0, 0)`) is named `Master.<n>` until renamed.

Verified with Visio Pro 2019 C2R x64 by COM automation from PowerShell, 2026-10 (stencil opened read-write, drops by `Page.Drop`); edit in place, ReplaceShape, Undo, events, sync: 2026-11, build 20527, master in the document stencil, top shape a group with sub-shapes. _Unverified:_ drop by mouse from the Shapes window (may differ from `Page.Drop`), `MatchByName` only on the stencil master, the document master's own `Edit` UI.
