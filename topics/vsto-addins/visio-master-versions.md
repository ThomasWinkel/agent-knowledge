---
title: Visio master versions - stencil edits and documents with old copies
description: Read when shipping a changed master in a stencil, or when code must bring a new master version into drawings that already use the old one - BaseID, UniqueID, drop, MatchByName.
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
- `MatchByName = true` on the **document** master: the drop reuses the old document master by name, the new content never arrives.

Bring a new version into a drawing: edit the **document** master (`docMaster.Open()`, change cells, `Close()`): every instance that inherits the cell gets the new value at once; instances with a local value keep it.

```csharp
Master copy = doc.Masters.ItemU["Board"].Open();
copy.Shapes[1].CellsU["User.MyData"].FormulaU = "\"v2\"";
copy.Close();   // commits; instances inherit "v2", the document master gets a new UniqueID
```

## Gotchas

- `BaseID` is the only ID that survives edits; it identifies "the same master in another version". The delete-and-redrag way of editing loses it; then only the name is left to match.
- A master made from a shape (`Masters.Drop(shape, 0, 0)`) is named `Master.<n>` until renamed.

Verified with Visio Pro 2019 C2R x64 by COM automation from PowerShell, 2026-10 (stencil opened read-write, drops by `Page.Drop`). _Unverified:_ drop by mouse from the Shapes window (may differ from `Page.Drop`), `MatchByName` only on the stencil master, the document master's own `Edit` UI.
