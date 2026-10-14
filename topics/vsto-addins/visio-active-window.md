---
title: Visio ActiveWindow and Selection by window type
description: Read when a Visio add-in command uses ActiveWindow or its Selection, or selects shapes by code - ShapeSheet, master and group edit window, no document, group selection, select all.
---

# Visio ActiveWindow and Selection by window type

A command that works on `Application.ActiveWindow.Selection` only fails loudly when no document is open. In the other windows it does nothing or changes a master. Check `Window.Type` and `SubType` first. Reference: [Window.Type](https://learn.microsoft.com/en-us/office/vba/api/visio.window.type), [Window.SubType](https://learn.microsoft.com/en-us/office/vba/api/visio.window.subtype).

| Active window | `ActiveWindow` | `Type` / `SubType` | `Selection` | `Window.Page` / `ActivePage` |
|---|---|---|---|---|
| No document | `null` | - | - | - |
| Drawing page | window | 1 `visDrawing` / 128 `visPageWin` | selected shapes | the page |
| Master edit window | window | 1 `visDrawing` / 64 `visMasterWin` | shapes of the master | `null` |
| Group edit window (Open Group, `Shape.OpenDrawWindow` of a group on a page) | window | 1 `visDrawing` / 160 `visPageGroupWin` | shapes inside the group | not checked |
| ShapeSheet window | window | 3 `visSheet` / 3 | empty (`Count` 0), no exception | the page |

## Gotchas

- **ShapeSheet window: `Selection` does not throw.** It returns an empty selection, also when a shape is selected in the drawing window behind it. A loop over it silently does nothing.
- The shape of a ShapeSheet window: `ActiveWindow.SelectedCell.Shape` (page sheet: shape `ThePage`).
- **Master edit window has the same `Type` as a page window.** `Selection` returns shapes of the master (`Shape.ContainingMaster` set, `Shape.ContainingPage` is `null`); a command changes the master. Tell them apart by `SubType`, or by `Window.Page == null`.
- `Application.ActivePage` is `null` in a master edit window; it does not throw.
- **A master edit window works on a copy of the master.** Shapes taken from the `Master` object (verified: the return of `Masters.Add`) are not the shapes in the window: `Window.Select` with them throws `Inappropriate target object for this action`. Take them from the window: `window.Selection.ContainingMaster.Shapes`, or `Window.Master` (the copy; draw on it, select its shapes).
- Closing a changed master edit window by code (`Window.Close`) asks `Update '<name>'?` (`NUIDialog`, modal): the COM call waits until someone answers.
- **Shapes selected inside a group do not count.** `Window.Selection` has [IterationMode](https://learn.microsoft.com/en-us/office/vba/api/visio.selection.iterationmode) 1280 (`visSelModeSkipSuper + visSelModeSkipSub`). Only shapes of one group selected (click into the group): `Count` is 0. Together with top-level shapes: only those count. `selection.IterationMode = 0` gives all, the group included (3 sub-shapes: `Count` 4).
- Select a shape inside a group by code: `Window.Select(shape, visSubSelect)` (3). `visSelect` (2) throws `Inappropriate target object for this action`.
- **Group edit window:** a shape inside the group is selected with `Window.Select(shape, visSelect)` (2). Selecting top-level shapes of the page there failed with `Inappropriate target object for this action` (seen through an add-in; exact call not isolated). A command that selects shapes of the page: exclude it by `SubType` (`visPageGroupWin` 160, `visMasterGroupWin` 96).

## Set the selection by code

Build a selection, then set it in one call:

```csharp
Visio.Selection result = page.CreateSelection(Visio.VisSelectionTypes.visSelTypeEmpty);   // master: master.CreateSelection
result.Select(first, (short)Visio.VisSelectArgs.visSelect);   // first selected = PrimaryItem
result.Select(other, (short)Visio.VisSelectArgs.visSelect);
window.Selection = result;
```

- `Selection.ContainingPage` is `null` for a selection in a master edit window; use `ContainingMaster` there.
- **Select all: not `CreateSelection(visSelTypeAll)`.** It also takes shapes on a locked layer and shapes whose layers are all hidden; set as window selection they stay selected (Delete removes hidden shapes). `Window.SelectAll()` skips them, but works only on a window. Same rule in code:
  ```csharp
  // Rule of Window.SelectAll: no layer locked; no layers, or at least one visible.
  static bool CanSelect(Visio.Shape shape)
  {
      var layers = Enumerable.Range(1, shape.LayerCount).Select(i => shape.Layer[(short)i]).ToList();
      return layers.All(l => l.CellsC[(short)Visio.VisCellIndices.visLayerLock].ResultIU == 0)
          && (layers.Count == 0 || layers.Any(l => l.CellsC[(short)Visio.VisCellIndices.visLayerVisible].ResultIU != 0));
  }
  ```
  Loop over `ContainingShape.Shapes` of a selected shape (page or master) and select those that pass. `SelectAll` also takes guides and shapes with `LockSelect` = 1 (document not protected).

## Guard

```csharp
Visio.Window window = application.ActiveWindow;
if (window == null
    || window.Type != (short)Visio.VisWinTypes.visDrawing
    || window.SubType != (short)Visio.VisWinTypes.visPageWin)
    return;
Visio.Selection selection = window.Selection;
```

Verified with Visio 2019 C2R x64 by COM automation (`Visio.InvisibleApp`; group selection: visible instance) (2026-10). Set the selection by code: page and master edit window; group edit window on a page (shape inside the group). Select-all rule: compared with `Window.SelectAll` for shapes on locked, hidden, locked + normal, hidden + normal layers, guide (page window). _Unverified:_ `LockSelect` in a protected document; group edit window of a master (`visMasterGroupWin` 96), stencil windows, and whether a custom ribbon tab stays clickable while a ShapeSheet window is active.
