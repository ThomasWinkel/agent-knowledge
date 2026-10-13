---
title: Calling a Visio add-in from the ShapeSheet
description: Read when a ShapeSheet cell (EventDrop, EventDblClick, Actions) must run code in a Visio VSTO/COM add-in - marker events, context string format, QUEUEMARKEREVENT vs RUNADDONWARGS, redo.
---

# Calling a Visio add-in from the ShapeSheet

A cell cannot call a COM/VSTO add-in: `RUNADDON` needs a VSL/EXE add-on, `CALLTHIS` is VBA only. Instead the cell queues a marker event; the add-in handles `Application.MarkerEvent` and reads the context string. Reference: [MarkerEvent](https://learn.microsoft.com/en-us/office/vba/api/visio.application.markerevent), [QUEUEMARKEREVENT](https://learn.microsoft.com/en-us/office/client-developer/visio/queuemarkerevent-function).

## Two formulas, two context strings

| Formula in the cell | `ContextString` in the handler |
|---|---|
| `QUEUEMARKEREVENT("/myapp /cmd=Run")` | `/myapp /cmd=Run` |
| `RUNADDONWARGS("QueueMarkerEvent","/myapp /cmd=Run")` | `/doc=1 /page=1 /shape=Sheet.5 /shapeu=Sheet.5 /myapp /cmd=Run` |

- **Only the built-in add-on `QueueMarkerEvent` tells where the cell is.** The function sends the text unchanged; the handler then does not know the shape. The reference does not mention this difference; the prefix is described only for the add-on ([Visio 2002 article](https://learn.microsoft.com/en-us/previous-versions/office/developer/office-xp/aa140366(v=office.10))).
- `/doc` = index in `Application.Documents`, `/page` = index in `Document.Pages` (indexes, not IDs).
- Cell in a master: `/doc=1 /master=2 /shape=Sheet.5 /shapeu=Sheet.5 ...`; `/master` = index in `Document.Masters`, no `/page`. Happens e.g. on a double-click in the master edit window.
- `/shape` and `/shapeu` are always `Sheet.<ID>`, also for a shape with a name of its own. For a sub-shape of a group it is the sub-shape. Resolve: `page.Shapes.ItemFromID[id]` (finds nested shapes).
- With the function the formula can add the shape itself: `"/id="&ID()` arrived as `/id=5`. _Unverified:_ no ShapeSheet function gives the document.

## Handler

```csharp
// ThisAddIn_Startup
Application.MarkerEvent += OnMarkerEvent;

void OnMarkerEvent(Visio.Application app, int sequenceNum, string contextString)
{
    // Every add-in gets the marker events of all solutions: check for the own switch first.
    // Split at the own switch: text before it is the place written by Visio, text after it is yours.
}
```

## Gotchas

- Event cells (`EventDblClick`, `EventDrop`, `Actions.X.Action`) fire only when triggered, not when the formula is set. A User cell fires as soon as the formula is set.
- `EventDrop` fires on dropping a master instance and on duplicating a shape. It **fires again on redo** of the drop (not on undo). `Application.IsUndoingOrRedoing` is `False` in the marker handler, so it is no guard.
- Only the dropped shape fires: `EventDrop` of the sub-shapes of a group does not fire when the group is duplicated or dropped as a master instance, also not when the handler then ungroups it.
- Changes the handler makes for `EventDrop` join the Undo step of the drop: one Undo removes the shape. After Redo **the handler's change is there twice** (redo fires `EventDrop` again): a handler that counts `K1` up to `K2` on drop leaves `K3` after Undo + Redo. Seen with the handler's change inside `BeginUndoScope`/`EndUndoScope`.
- A handler that deletes or ungroups the dropped shape: Redo brings the handler's result back, then fires `EventDrop` for a `/shape` that no longer exists. `ItemFromID` throws (`An exception occurred.`); handle it.
- A handler change for `EventDblClick` inside one undo scope is one Undo step of its own. A scope without changes adds no Undo step, so the handler can open a scope for every command, also read-only ones.
- Cell values joined into the formula arrive as text in the UI locale, with unit: `"/w="&Width` gave `/w=25,4MM`, `"/n="&(1/3)` gave `/n=0,33333333333333` (German Windows). Pass the cell name and read the cell in the add-in.
- `""` in the formula gives `"` in the context string: `"/label=""two words"""` arrives as `/label="two words"`.
- No listener (add-in not loaded): nothing happens, no error.

## Test without the UI

- COM automation is enough: `Visio.InvisibleApp`, set the formula, then `cell.Trigger()` for event cells or `page.Drop(master, x, y)` for `EventDrop`. A VSTO add-in registered in HKCU loads in the invisible instance too.
- To log marker events from outside: `app.EventList.AddAdvise(0x1100, sink, "", "")` (`visEvtApp + visEvtMarker`) with a `[ComVisible(true)]` class that has `object VisEventProc(short code, object source, int id, int seqNum, object subject, object moreInfo)`. Read the text with `app.EventInfo[0]`; `EventInfo[seqNum]` failed for some events.
- A modal dialog opened by the handler (message box) or by Visio blocks the test: the automation call does not return until it is closed. Close it from a second process: enumerate the visible top-level windows of the Visio process (an invisible instance has no others), then `PostMessage(dialog, WM_COMMAND, GetDlgCtrlID(button), button)` with the first child of class `Button`.
- `Application.ProcessID` is not the Windows process ID (seen: 747 for process 14740). Find the process of a new instance by comparing `Get-Process VISIO` before and after it starts.
- Run it on an STA thread that pumps messages (`Application.DoEvents()` in a wait loop). C# compiled with `Add-Type` on its own STA thread works; a version in plain PowerShell hung.

Verified with Visio Pro 2019 C2R x64 on German Windows (2026-10).
