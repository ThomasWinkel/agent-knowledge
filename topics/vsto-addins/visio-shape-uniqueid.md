---
title: Visio shape UniqueID on copy, cut and paste
description: Read when Visio code needs a stable shape identity, e.g. to tell a copied shape from the original or a moved one - which edits keep, drop or never set Shape.UniqueID.
---

# Visio shape UniqueID on copy, cut and paste

`Shape.UniqueID(visGetOrMakeGUID)` (1) gives the shape a GUID and returns it; `UniqueID(visGetGUID)` (0) returns it or `""`. A shape has none until code asks for one with 1 (e.g. drawn or dropped shapes).

| Edit | Original | New shape |
|---|---|---|
| Copy + paste (same or other document) | keeps its GUID | none |
| Duplicate (Ctrl+D) | keeps its GUID | none |
| Duplicate page (`Page.Duplicate`) | keeps its GUID | none |
| Cut + paste (same or other document) | - | **same GUID** |
| Delete + Undo | same GUID | - |
| Save As, save + reopen | same GUID | - |

Use: store the GUID next to your own key in the shape (e.g. a User cell). Later, GUID empty or different from the stored one = the shape is a copy (the copy carries the stored value too) -> give it a new key; GUID equal = original, also after cut + paste.

```csharp
string guid = shape.UniqueID[(short)VisUniqueIDArgs.visGetGUID];
bool isCopy = guid != storedGuid;   // copies have "" until made
```

## Gotchas

- A cut + paste is a move: the pasted shape is the original (same GUID), also in another document. Copy, delete the original, paste: the pasted shape is a copy.

Verified with Visio Pro 2019 C2R x64 by COM automation from PowerShell (visible Visio, out of process), 2026-10: copy, cut, paste and duplicate by `Application.DoCmd` (`visCmdUFEditCopy` 1021, `visCmdUFEditCut` 1020, `visCmdUFEditPaste` 1022, `visCmdUFEditDuplicate` 1024) on a drawn rectangle. _Unverified:_ Ctrl+drag copy with the mouse, Paste Special, instances dropped from a stencil, shapes inside groups.
