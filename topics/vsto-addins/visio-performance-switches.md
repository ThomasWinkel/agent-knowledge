---
title: Visio performance switches for bulk changes
description: Read when Visio code changes many shapes or cells and you weigh ScreenUpdating, ShowChanges, DeferRecalc, EventsEnabled, UndoEnabled or batch calls - effects, side effects, one measurement.
---

# Visio performance switches for bulk changes

`Visio.Application` has switches that speed up a series of changes. Most have side effects: choose per action, do not turn on all of them everywhere. All are settings of the whole application: keep the old value and put it back in `finally`.

| Property | Type (PIA) | Effect while switched | Side effect | Use |
|---|---|---|---|---|
| [`ScreenUpdating`](https://learn.microsoft.com/en-us/office/vba/api/visio.application.screenupdating) = 0 | `short` | no redraw; Visio still redraws now and then | none on the selection | safe default |
| [`ShowChanges`](https://learn.microsoft.com/en-us/office/vba/api/visio.application.showchanges) = false | `bool` | no redraw at all, views do not react to changes; faster than `ScreenUpdating` alone | **deselects all shapes** in drawing and stencil windows; **code cannot change the selection** | actions that do not depend on the selection |
| [`DeferRecalc`](https://learn.microsoft.com/en-us/office/vba/api/visio.application.deferrecalc) = 1 | `short` | formulas are recalculated later | queue is processed early when objects are released | write-only phases |
| [`InhibitSelectChange`](https://learn.microsoft.com/en-us/office/vba/api/visio.application.inhibitselectchange) = true | `bool` | dropped shapes are not selected, the selection is kept where possible; code can still select | - | many drops |
| [`DeferRelationshipRecalc`](https://learn.microsoft.com/en-us/office/vba/api/visio.application.deferrelationshiprecalc) = true | `bool` | containers do not resize while members move or resize; `false` processes all deferred actions at once | - | diagrams with containers |
| [`EventsEnabled`](https://learn.microsoft.com/en-us/office/vba/api/visio.application.eventsenabled) = 0 | `short` | Visio fires no events, runs no add-ons, does not execute `RUNADDON` code in cells | hits every add-in and VBA project, not only yours | avoid |
| [`UndoEnabled`](https://learn.microsoft.com/en-us/office/vba/api/visio.application.undoenabled) = false | `bool` | no undo information is collected | **clears the existing undo history** (measured: a step before it no longer undoes) | avoid when the user must be able to undo; data outside Undo: [visio-undo-scopes](visio-undo-scopes.md#data-outside-undo) |
| [`LiveDynamics`](https://learn.microsoft.com/en-us/office/vba/api/visio.application.livedynamics) | `bool` | recalculation on every mouse move of a drag, or only on release | - | mouse drags only, no gain for code |

```csharp
bool showChanges = application.ShowChanges;
short deferRecalc = application.DeferRecalc;
application.ShowChanges = false;
application.DeferRecalc = 1;
try
{
    // Change shapes and cells.
}
finally
{
    application.DeferRecalc = deferRecalc;
    application.ShowChanges = showChanges;
}
// Select the result only now: ShowChanges = false blocks it.
```

## Gotchas

- `ShowChanges`: get `Window.Selection` before switching it off, select results after switching it on again. An action that fails has still lost the user's selection.
- Setting `ShowChanges` sets `ScreenUpdating` too; setting `ScreenUpdating` does not set `ShowChanges`.
- With both off, Visio updates the views only when `ShowChanges` comes back: a noticeable delay after a long sequence. Switching `ScreenUpdating` on again right after `ShowChanges = false` shortens that delay and lengthens the sequence.
- Visio turns `ShowChanges`, `ScreenUpdating` and `InhibitSelectChange` back by itself when the user does something. Do not rely on it; for `DeferRecalc` the docs demand the reset.
- `DeferRecalc`: Visio works off the pending recalculations when objects are released or many commands arrive. Keep the `Cell` objects in variables until the end and only set formulas or results in the deferred part.
- Show a message box only after the switches are back: with `ShowChanges = false` Visio does not repaint its windows, also not when another window uncovered them.

## Measured: the switches gave nothing, the batch call did

One case: a VSTO add-in (in-process), ribbon click handler, one undo scope, moves n plain rectangles on a new blank page by two cell writes each (`PinX`, `PinY` via `Cell.ResultIU`). Time of the handler:

| Shapes | Single writes | `Page.SetResults` |
|---|---|---|
| 100 | 0.07 s | 0.05 s |
| 500 | 0.3 s | 0.13 s |
| 2000 | 4.1 s | 1.2-2.2 s |

- **No measurable gain**, alone or combined: `ScreenUpdating = 0`, `ShowChanges = false`, `DeferRecalc = 1`, `DeferRelationshipRecalc = true`, `Page.AutoSize = false`. Measure before adding switches to a handler.
- [`Page.SetResults`](https://learn.microsoft.com/en-us/office/vba/api/visio.page.setresults) (all cells in one call) made the write part 3x (500 shapes) to 5x (2000) faster.
- The time per cell write grows with the number of shapes written: 0.2 ms at 500, 0.8 ms at 2000; also inside the batch call. Cause not found.
- Not in the write loop: with 2000 shapes `EndUndoScope` took about 0.9 s, and Visio reacted again only about 2 s after the handler returned, in every variant.

```csharp
// Stream: 4 shorts per cell = sheet ID, section, row, cell.
short[] stream = { id, (short)Visio.VisSectionIndices.visSectionObject,
                       (short)Visio.VisRowIndices.visRowXFormOut,
                       (short)Visio.VisCellIndices.visXFormPinX /* , next cell ... */ };
Array streamArray = stream;
Array units = new object[] { (short)Visio.VisUnitCodes.visInches };   // one entry: used for all
Array results = new object[] { 1.25 /* , ... */ };
short written = page.SetResults(ref streamArray, ref units, ref results, 0);
```

Measured with Visio Pro 2019 C2R x64, Debug build, stopwatch in the add-in (2026-10). _Unverified:_ shapes with many formulas or sub-shapes, pages with connectors, what `SetResults` does with guarded cells.

Other statements - source: the Microsoft reference pages linked above (2026-10); not tested. _Unverified_ (concluded, not stated there): marker events raised while `EventsEnabled = 0` are lost, not delivered later; in .NET the garbage collector decides when a `Cell` object is released, so `DeferRecalc` defers less reliably than in VBA; results read while `DeferRecalc` is on may be stale.
