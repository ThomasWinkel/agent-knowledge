---
title: Visio Shape.Ungroup from code
description: Read when a Visio add-in or script calls Shape.Ungroup - confirmation dialog for master instances, AlertResponse, errors for non-groups and locked groups, what is left of the group.
---

# Visio Shape.Ungroup from code

`Shape.Ungroup()` on an instance of a master **shows a confirmation dialog, also when called by code** and in `Visio.InvisibleApp`: the instance loses the link to its master. The call waits for the answer. Set `Application.AlertResponse` for the call. Reference: [Shape.Ungroup](https://learn.microsoft.com/en-us/office/vba/api/visio.shape.ungroup), [AlertResponse](https://learn.microsoft.com/en-us/office/vba/api/visio.application.alertresponse).

```csharp
Visio.Application application = shape.Application;
short alertResponse = application.AlertResponse;
application.AlertResponse = 1; // IDOK: answer the dialog, do not show it
try
{
    shape.Ungroup();
}
finally
{
    application.AlertResponse = alertResponse; // 0 = show dialogs again
}
```

## Behavior

| Shape | `Ungroup()` |
|---|---|
| Group drawn on the page | no dialog; group gone, sub-shapes stay |
| Instance of a master | dialog (Office dialog, window class `NUIDialog`, title `Microsoft Visio`); with `AlertResponse = 1` no dialog, sub-shapes have no master |
| Dialog cancelled | COM error `Cancel.`, nothing changed |
| Not a group (`Shape.Type != visTypeGroup`) | COM error `Requested operation is presently disabled.` |
| Group with `LockGroup = 1` | COM error `Shape protection, container, and/or layer properties prevent complete execution of this command.`; stays a group |
| Group inside a group | sub-shapes move into the outer group |

## Gotchas

- `AlertResponse` is a setting of the whole application and stays until it is reset: restore it in `finally`, or every later dialog of Visio is answered silently.
- A group with text of its own does not vanish: it stays as a plain shape (same ID, `Type` 3) with that text, beside the former sub-shapes. A group without text is deleted.
- The sub-shapes keep their IDs; the `Shape` object of the group is invalid afterwards (unless it stayed because of its text).
- Group and ungroup is not neutral: after `Selection.Group()` and `Ungroup()` the shapes lie in front of shapes that were between or above them (z-order). Undo restores the order.
- Inside `BeginUndoScope`/`EndUndoScope` the ungroup is one Undo step; Undo brings the group and its master link back.
- Called from a marker event of `EventDrop` ([visio-shapesheet-calls](visio-shapesheet-calls.md)): drop and ungroup are one Undo step. Redo fires `EventDrop` again for the group, which no longer exists.

Verified with Visio Pro 2019 C2R x64 by COM automation (`Visio.InvisibleApp`) (2026-10). _Unverified:_ the dialog text, groups with geometry of their own, shapes on locked layers.
