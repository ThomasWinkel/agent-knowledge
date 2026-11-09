---
title: Visio master versions - stencil edits and documents with old copies
description: Read when shipping a changed master in a stencil, or when code must bring a new master version into drawings that already use the old one - BaseID, UniqueID, drop, MatchByName, edit in place, Undo.
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

## Undo

- Wrap every `Master.Open()`/`Close()` in its own `BeginUndoScope`/`EndUndoScope`. Then each Undo takes back one edit, master and instances together (deleted rows and sub-shapes come back with their local values), and Redo works, also with `FormulaU = ""` resets of instance cells in the same scope (measured up to 500 instances).
- Without own scopes, several master edits in a row: the first Undo took back only the last edit; further Undo calls changed nothing.
- _Unverified:_ a Redo failed with `0x86DB089A` after a longer sequence (six scoped edits incl. sub-shape add/delete, five Undo, one more scope, Undo, Redo); a shorter sequence redid fine. Cause unknown.

## Gotchas

- `BaseID` is the only ID that survives edits; it identifies "the same master in another version". The delete-and-redrag way of editing loses it; then only the name is left to match.
- A master made from a shape (`Masters.Drop(shape, 0, 0)`) is named `Master.<n>` until renamed.

Verified with Visio Pro 2019 C2R x64 by COM automation from PowerShell, 2026-10 (stencil opened read-write, drops by `Page.Drop`); edit in place, ReplaceShape, Undo, events: 2026-11, build 20527, master in the document stencil, top shape a group with sub-shapes. _Unverified:_ drop by mouse from the Shapes window (may differ from `Page.Drop`), `MatchByName` only on the stencil master, the document master's own `Edit` UI.
