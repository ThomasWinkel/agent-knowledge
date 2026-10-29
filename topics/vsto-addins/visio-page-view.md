---
title: Visio page turns and zoom by code
description: Read when Visio code turns pages (Window.Page), zooms (ViewFit) or goes to a shape (CenterViewOnShape) - Undo step per page turn, zoom per page and in the file, flags, hidden pages/layers.
---

# Visio page turns and zoom by code

A drawing window keeps zoom and scroll position per page. Code can set them for every page by turning to each page. Reference: [Window.Page](https://learn.microsoft.com/en-us/office/vba/api/visio.window.page), [Window.ViewFit](https://learn.microsoft.com/en-us/office/vba/api/visio.window.viewfit).

```csharp
// Whole page in the window, for every page; one Undo step.
int scope = app.BeginUndoScope("Fit Pages");
try
{
    foreach (Visio.Page page in window.Document.Pages)
    {
        window.Page = page;
        window.ViewFit = (short)Visio.VisWindowFit.visFitPage;
    }
    window.Page = window.Document.Pages[1];
    app.EndUndoScope(scope, true);
}
catch
{
    app.EndUndoScope(scope, false);
    throw;
}
```

## Gotchas

- **Every `Window.Page = ...` by code is an Undo step** (`Go To Page`). A loop over N pages adds N steps and pushes the user's real changes out of the limited Undo list. Wrap the loop in one undo scope: one step under its name.
  - Undo of that step: the window goes back to the page it showed before, with that page's old zoom. The other pages keep the new zoom.
  - Without a scope, each Undo goes back one page and restores the view as it was before that turn.
- Zoom set by code stays with the page: turning away and back shows it again (`Zoom` 2 on page 1, 3 on page 2, back to page 1: 2).
- **The zoom is saved in the file:** VSDX `visio/pages/pages.xml`, attributes of `<Page>`: `ViewScale='-1'` = fit page (`ViewFit` 1), plus `ViewCenterX`/`ViewCenterY`. After reopening each page shows with its saved zoom.
- Turning pages and zooming do not make the document dirty (`Document.Saved` stays True, with and without undo scope): Visio does not ask to save, the view is lost unless the document is saved.
- Turning to another page clears the selection (`Window.Selection.Count` 0 afterwards).
- `Document.Pages` includes background pages; `ViewFit` works on them too. Seen with one background page: it came after the foreground pages in `Pages`, so `Pages[1]` was the first foreground page.
- `Window.Page` exists only in a page window (`Type` 1 `visDrawing`, `SubType` 128 `visPageWin`); check first, see [visio-active-window](visio-active-window.md).
- Time: 51 pages with one rectangle each, fit by an add-in ribbon handler: about 0.5 s until Visio answered COM calls again (no undo scope).

## Go to a shape on any page

`window.CenterViewOnShape(shape, Visio.VisCenterViewFlags.visCenterViewSelectShape)` turns the window to the shape's page, centers it and selects it (selection = only that shape). Reference: [Window.CenterViewOnShape](https://learn.microsoft.com/en-us/office/vba/api/visio.window.centerviewonshape).

- Flag values (interop enum): `visCenterViewDefault` 0, `visCenterViewIfOffScreen` 1, `visCenterViewSelectShape` 2. Late-bound callers that pass 1 get no selection.
- Works for shapes on background pages, hidden pages (`UIVisibility` 1, [visio-hidden-pages](visio-hidden-pages.md)) and hidden background pages: the window shows that page.
- **Also selects shapes the user cannot see or select:** a shape whose only layer is hidden and/or locked is selected all the same (`Selection.Count` 1). Filter such targets yourself if the user should not land on them.
- Undo: a call that turns to another page is one Undo step, like `Window.Page` (Undo: window back on the old page, the cell change made before stays). Inside an undo scope: still one step. A call on the shown page adds no Undo step.

Verified with Visio Pro 2019 C2R x64 (2026-10): page turns and zoom by a VSTO add-in ribbon handler clicked by UI Automation, visible instance; `CenterViewOnShape` by COM automation, own visible instance. _Unverified:_ group and master edit windows, `ShowChanges = false` during the loop (effect on flicker and on `ViewFit`), pages with many shapes; Undo step names.
