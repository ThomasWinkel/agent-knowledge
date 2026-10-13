---
title: Visio ActiveWindow and Selection by window type
description: Read when a Visio add-in command (ribbon button, event handler) uses Application.ActiveWindow, its Selection or ActivePage - ShapeSheet window, master edit window, no document.
---

# Visio ActiveWindow and Selection by window type

A command that works on `Application.ActiveWindow.Selection` only fails loudly when no document is open. In the other windows it does nothing or changes a master. Check `Window.Type` and `SubType` first. Reference: [Window.Type](https://learn.microsoft.com/en-us/office/vba/api/visio.window.type), [Window.SubType](https://learn.microsoft.com/en-us/office/vba/api/visio.window.subtype).

| Active window | `ActiveWindow` | `Type` / `SubType` | `Selection` | `Window.Page` / `ActivePage` |
|---|---|---|---|---|
| No document | `null` | - | - | - |
| Drawing page | window | 1 `visDrawing` / 128 `visPageWin` | selected shapes | the page |
| Master edit window | window | 1 `visDrawing` / 64 `visMasterWin` | shapes of the master | `null` |
| ShapeSheet window | window | 3 `visSheet` / 3 | empty (`Count` 0), no exception | the page |

## Gotchas

- **ShapeSheet window: `Selection` does not throw.** It returns an empty selection, also when a shape is selected in the drawing window behind it. A loop over it silently does nothing.
- The shape of a ShapeSheet window: `ActiveWindow.SelectedCell.Shape` (page sheet: shape `ThePage`).
- **Master edit window has the same `Type` as a page window.** `Selection` returns shapes of the master (`Shape.ContainingMaster` set, `Shape.ContainingPage` is `null`); a command changes the master. Tell them apart by `SubType`, or by `Window.Page == null`.
- `Application.ActivePage` is `null` in a master edit window; it does not throw.

## Guard

```csharp
Visio.Window window = application.ActiveWindow;
if (window == null
    || window.Type != (short)Visio.VisWinTypes.visDrawing
    || window.SubType != (short)Visio.VisWinTypes.visPageWin)
    return;
Visio.Selection selection = window.Selection;
```

Verified with Visio 2019 C2R x64 by COM automation (`Visio.InvisibleApp`) (2026-10). _Unverified:_ group edit windows (`visPageGroupWin` 160, `visMasterGroupWin` 96), stencil windows, and whether a custom ribbon tab stays clickable while a ShapeSheet window is active.
