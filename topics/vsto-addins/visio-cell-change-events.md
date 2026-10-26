---
title: Visio cell change events - marker cell vs filtered CellChanged, Undo
description: "Read when a Visio add-in must react to changed Shape Data or other cells (name typed, drop, paste, delete, Undo), writes cells in the handler, or batches work per user action."
---

# Visio cell change events

Two ways to learn that a cell changed: a marker cell in every shape (`DEPENDSON` + `QUEUEMARKEREVENT`), or the `CellChanged` event with a Visio-side filter. Measured 2026-10-26, Visio 2019 C2R x64, out of process (C#, events on `Application`, own `Visio.InvisibleApp`), actions by COM: counts and Undo behavior are valid, times are not in-process times. UI actions checked by hand with a visible Visio (see UI below).

## What fires

| Action | marker cell `RUNADDONWARGS("QueueMarkerEvent","/x")+DEPENDSON(Prop.Name,Prop.Unit)` | `CellChanged`, filter on the Prop section | `ShapeAdded` / `BeforeShapeDelete` |
|---|---|---|---|
| Name set to a new value | 1 | 1 | - |
| Name set to the same value | 0 | 0 (no event at all) | - |
| other Prop cell changed | 0 | 1 | - |
| Undo / Redo of a name change | 1-2 / 1 | 1 / 1 | - |
| drop 1, `DropMany` 1,000, paste 1/100, duplicate | 0 | 0 | 1 per shape |
| delete 100 / its Undo | 0 | 0 | 100 / 100 |
| move 1,000 shapes (`Selection.Move`) | 0 | 0 (unfiltered: 1,000) | - |
| Redo of `DropMany` 1,000 + 50 name changes | 50 | 50 (unfiltered: 2,050) | 1,000 |
| open the document | 0 | 0 | 0 |
| page rename | 0 | 0 | `PageChanged` 1 |

- The marker cell fires only on a real change of a cell it depends on; a new instance (drop, paste, duplicate) or opening the file does not make it fire.
- The marker arrives after `ExitScope` of the action. `IsUndoingOrRedoing` is `false` in the marker handler, also on Undo: it cannot tell Undo from a user change.
- In the `CellChanged` handler `IsUndoingOrRedoing` is `true` on Undo/Redo. It needs no formula in the shapes, so it also covers shapes without master or built by the user.
- A handler that writes the watched cell (e.g. `Prop.Name`) triggers itself once more: guard it.

## UI actions

- Shape Data window with 50 shapes selected, one field changed: one Undo step `Shape Data`; 50 filtered `CellChanged`, 50 markers. One Ctrl+Z / Ctrl+Y takes back / brings back all 50 changes together with the handler's writes; `IsUndoingOrRedoing` true in every `CellChanged` call.
- Ctrl+C/Ctrl+V of 50: `ShapeAdded` 50, nothing else. Delete 50: `BeforeShapeDelete` 50. Dragging 100 shapes: unfiltered `CellChanged` 208, filtered 0.
- Multiple selection shows only Shape Data rows with a `Label` ("No common data fields" otherwise).
- Editing the master (3 Prop labels): filtered `CellChanged` for every instance (303 with 100 instances).

## Filter CellChanged in Visio

`SetFilterSRC` takes **7 values per range**: section, row, cell of the start, section, row, cell of the end, then `-1` (True = listen) or `0` (exclude). Six values throw `An exception occurred.` ([reference](https://learn.microsoft.com/en-us/office/vba/api/visio.event.setfiltersrc)).

```csharp
const short visEvtCellChanged = 0x800 | 0x2000;               // visEvtCell + visEvtMod
var ev = app.EventList.AddAdvise(visEvtCellChanged, new Sink(), "", "");
Array src = new short[] { 243, 0, 0, 243, 32767, 32767, -1 };   // whole Prop section (visSectionProp)
ev.SetFilterSRC(ref src);

[ComVisible(true)]
public class Sink : Visio.IVisEventProc   // public, also its outer class; else AddAdvise throws
{
    public object VisEventProc(short code, object source, int id, int seq, object subject, object more)
    {
        var cell = (Visio.Cell)subject;    // cell.Name e.g. "Prop.Name", cell.Shape
        return null;
    }
}
```

## Writes in the handler and Undo

- Written directly in the marker handler or the `CellChanged` handler (guarded by `IsUndoingOrRedoing`): **same Undo step** as the change that fired it, also when the handler opens its own undo scope. One Undo takes back both, Redo brings both.
- Written later, in `VisioIsIdle` or in the next `ExitScope`: an Undo step of its own; one Undo takes back only the handler's write. So work cannot be collected and done once per user action without an extra Undo step.
- A write during Undo that changes a value clears the Redo stack. `CellChanged`: skip when `IsUndoingOrRedoing`. Marker: no guard possible; write only when the value differs (Undo already restored a value derived from the restored cells, so the write is a no-op and Redo survives).

## Gotchas

- An out-of-process probe killed while its events were advised left that Visio session without any `Application` events (`MarkerEvent`, `CellChanged`, `ExitScope` ...) for later probes; only a sink added by `EventList.AddAdvise` still got events. Restart Visio between probe runs that may crash. Cause _unverified_.
- `Visio.InvisibleApp`: `VisioIsIdle` fires all the time (a few per second); do not use it to detect "quiet". `Quit()` hung with a VSTO add-in loaded; end the process found by comparing `Process.GetProcessesByName("VISIO")` before and after start.

Related: glue and container events [visio-glue-events](visio-glue-events.md); marker cells and context strings [visio-shapesheet-calls](visio-shapesheet-calls.md); paste/delete give no glue event either.
