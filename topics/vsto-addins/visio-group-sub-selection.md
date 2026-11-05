---
title: Visio group members that cannot be sub-selected
description: Read when users must not select some members of a Visio group (SelectMode 1) while others stay movable - LockSelect, document protection, SelectionChanged redirect.
---

# Visio group members that cannot be sub-selected

A group with `SelectMode = 1` (group first, then members): first click selects the group, a second click on a member sub-selects it. No cell keeps one member from being sub-selected without document protection; an add-in can redirect the selection.

Measured 2026-11-05, Visio 2019 C2R x64 build 20527, own visible Visio, real mouse (`mouse_event`) + COM. Group of a locked drawing member (`LockMoveX/Y`, `LockWidth/Height`, `LockRotate`, `LockTextEdit`, `GlueType 8`), a locked control member and an unlocked text member; group `LockGroup = 1`.

## Facts

- `LockSelect = 1` on a member, document **not** protected: ignored. Second click and `Window.Select(member, visSubSelect)` (3) sub-select it.
- Document protected (`Document.Protection = 2`, visProtectShapes): `Window.Select(member, 3)` throws `Shape is locked against selection.`; a click on that member selects **nothing, not even the group**; a drag that starts on it (nothing selected) still moves the group. Other members sub-select as before.
- A sub-selected locked member cannot be dragged (nothing moves): users who click a selected group again on its drawing and drag get no move. A drag that starts on a member while the group is selected (no click before) moves the group, locked member or not.
- `Selection.IterationMode = 0` lists the group too; `Selection.ItemStatus(i)`: `1` selected top-level shape, `3` sub-selected member, `4` its group (superselected).
- `Application.SelectionChanged` fires for mouse and COM selection changes. A `Window.Select` inside the handler fires it once more after the handler returned (the guard flag below is already reset; that run finds nothing to redirect).

## Recipe: redirect to the group

In `Application.SelectionChanged`, replace a sub-selected member that must not be selected by its top-level shape. Mark such members with `LockSelect = 1`: Visio enforces it itself in a protected document, the add-in does it otherwise.

```csharp
private bool busy;
private void OnSelectionChanged(Visio.Window window)
{
    if (busy) return;
    var selection = window.Selection;
    selection.IterationMode = 0;
    for (var i = 1; i <= selection.Count; i++)
    {
        if (selection.ItemStatus[i] != 3) continue;          // sub-selected member
        var shape = selection[i];
        if (shape.CellsU["LockSelect"].ResultIU == 0) continue;
        var top = shape;
        while (top.ContainingShape is { } p && p.Type != (short)Visio.VisShapeTypes.visTypePage)
            top = p;
        busy = true;
        try { window.Select(top, (short)Visio.VisSelectArgs.visDeselectAll + (short)Visio.VisSelectArgs.visSelect); }
        finally { busy = false; }
        return;
    }
}
```

Measured result (same setup): second click on the marked member -> group stays selected (selection is the group 5-26 ms after the event, add-in log); a drag right after it moves the group; COM `Window.Select(member, 3)` -> group; unmarked members and the text member sub-select and drag as before. Calling `Select` inside the event and deferring it to the next `VisioIsIdle` gave the same results.

## Gotchas

- Events never arrive while `Application.EventsEnabled = 0`, which persists into later sessions: [visio-performance-switches](visio-performance-switches.md).
- _Unverified:_ visible flicker of the short sub-selection (Shape Data window, ribbon), Tab key selection, group edit window, protected document together with the redirect.
