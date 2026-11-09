---
title: Visio shape UniqueID on copy, cut and paste
description: Read when Visio code needs a stable shape identity, e.g. to tell a copied shape from the original or a moved one - which edits and Undo keep, drop or never set Shape.UniqueID.
---

# Visio shape UniqueID on copy, cut and paste

`Shape.UniqueID(visGetOrMakeGUID)` (1) gives the shape a GUID and returns it; `UniqueID(visGetGUID)` (0) returns it, or nothing: `""` in PowerShell/VBA, **`null` in C#** (interop). A shape has none until code asks for one with 1 (e.g. drawn or dropped shapes).

| Edit | Original | New shape |
|---|---|---|
| Copy + paste (same or other document) | keeps its GUID | none |
| Duplicate (Ctrl+D) | keeps its GUID | none |
| Duplicate page (`Page.Duplicate`) | keeps its GUID | none |
| Cut + paste (same or other document, other page) | - | **same GUID** |
| Cut, any `Page.Drop`, paste | - | none |
| Delete + Undo | same GUID | - |
| Save As, save + reopen | same GUID | - |
| Drop from a master whose shape had a GUID; `Page.Drop(shape)` | - | none |
| Undo of the scope that made the GUID | **keeps it** (also after save + reopen) | - |

Use: store the GUID next to your own key in the shape (e.g. a User cell). Later, GUID empty or different from the stored one = the shape is a copy (the copy carries the stored value too) -> give it a new key; GUID equal = original, also after cut + paste.

```csharp
string? guid = shape.UniqueID[(short)VisUniqueIDArgs.visGetGUID];   // null if none
bool isCopy = guid != storedGuid;
bool hasGuid = !string.IsNullOrEmpty(guid);   // not guid.Length: NullReferenceException
```

## Gotchas

- A cut + paste is a move: the pasted shape is the original (same GUID), also in another document. Copy, delete the original, paste: the pasted shape is a copy.
- **A drop between cut and paste makes the paste a copy.** Any `Page.Drop` (master or shape; on the same page, another page or in another document) after the cut: the pasted shape has no GUID. Drawing a shape, writing cells or `Pages.Add` in between keep it. So code that drops shapes on its own (e.g. a page border on every new page) turns the user's cut, new page, paste into a copy; so does a stencil drop by the user (not measured by mouse). Plan for it: a GUID kept proves a move, a missing GUID does not prove a copy.
- A second paste of the same clipboard: no GUID (a copy).
- Making a GUID adds no Undo step, inside or outside an undo scope: Undo of a scope that made a GUID and wrote a User cell takes back the cell only, the GUID stays. A scope that only makes a GUID is empty (Undo skips it). So a GUID kept outside the document (e.g. in `Document.SolutionXMLElement`) still finds the shape after the user undoes the step that stored its key in a cell.

Verified with Visio Pro 2019 C2R x64 by COM automation from PowerShell (visible Visio, out of process), 2026-10: copy, cut, paste and duplicate by `Application.DoCmd` (`visCmdUFEditCopy` 1021, `visCmdUFEditCut` 1020, `visCmdUFEditPaste` 1022, `visCmdUFEditDuplicate` 1024) on a drawn rectangle; Undo by `Application.Undo()` after `BeginUndoScope`/`EndUndoScope`. Drops by `Page.Drop` (master from the document stencil, and a shape). Drop between cut and paste, other page, second paste and the C# `null`: same Visio, 2026-11 (COM from PowerShell, events off and on; C# in a VSTO add-in event). _Unverified:_ Ctrl+drag copy with the mouse, Paste Special, drop by mouse from a stencil, shapes inside groups.
