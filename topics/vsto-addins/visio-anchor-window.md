---
title: Visio side panel with a WinForms control (anchor window)
description: Read when a Visio add-in shows its own docked panel (tree, list) - anchor window + WinForms control, keys Visio steals, close events, tabs by merge ID, DPI, drag, TreeView speed, UI tests.
---

# Visio side panel with a WinForms control (anchor window)

An add-in panel is an anchor window of a drawing window with a WinForms control put into it by `SetParent`. Measured with Visio 2019 C2R x64 build 20527, VSTO add-in in-process, .NET Framework 4.8 (2026-11).

```csharp
const string MergeId = "{<own GUID>}";   // same ID for all panels of the add-in -> one pane with tabs
int flags = (int)(Visio.VisWindowStates.visWSDockedRight | Visio.VisWindowStates.visWSVisible
                | Visio.VisWindowStates.visWSAnchorMerged);
Visio.Window anchor = drawingWindow.Windows.Add("My Panel", flags,
    (int)Visio.VisWinTypes.visAnchorBarAddon, 0, 0, 300, 400, MergeId, "", 0);

var panel = new MyPanel();                // UserControl
panel.CreateControl();
SetParent(panel.Handle, (IntPtr)anchor.WindowHandle32);
// Follow the size: subclass the anchor hwnd (NativeWindow.AssignHandle) and on WM_SIZE (0x0005)
// set panel bounds = GetClientRect(anchor hwnd).

anchor.BeforeWindowClosed += w => Release();          // user clicked X, or anchor.Close()
drawingWindow.BeforeWindowClosed += w => Release();   // window or document closed, see Gotchas
app.OnKeystrokeMessageForAddon += OnKey;              // once per add-in, see Keys
```

## Keys

Without a handler Visio translates its accelerators first: `Delete` in the panel deletes the selected shapes of the drawing, `Ctrl+C/V/Z/A`, `F2`, `Tab`, `Esc`, `Enter` go to Visio. Letters, `Backspace` and arrows reach the control.
`Application.OnKeystrokeMessageForAddon` gets every key message of an add-on anchor window before that. Return `true` = Visio does nothing with it; then dispatch it yourself:

```csharp
bool OnKey(Visio.MSGWrap m)
{
    var hwnd = (IntPtr)m.hwnd;
    var control = Control.FromChildHandle(hwnd);
    if (control is null || !IsInMyPanel(control))
        return false;                                   // other add-ins' windows
    var msg = Message.Create(hwnd, m.message, (IntPtr)m.wParam, (IntPtr)m.lParam);
    if (control.PreProcessControlMessage(ref msg) == PreProcessControlState.MessageProcessed)
        return true;                                    // Tab, dialog keys
    var native = new MSG { hwnd = hwnd, message = m.message, wParam = (IntPtr)m.wParam,
                           lParam = (IntPtr)m.lParam, time = m.posttime, x = m.ptx, y = m.pty };
    TranslateMessage(ref native);                       // user32
    DispatchMessage(ref native);
    return true;
}
```

- Measured with this handler: all keys above reach the control, none reaches the drawing; `Tab` moves the focus between controls (only with `PreProcessControlMessage`; plain dispatch gives the control a `Tab` KeyPress); `Ctrl+Z` undoes in a `TextBox`.
- Returning `true` for everything also blocks Visio's own shortcuts (`Ctrl+S`) while the panel has the focus; return `false` for keys the panel does not use if they should reach Visio (not measured).
- `TreeView.LabelEdit`: `F2` did not start editing in either mode (not looked into).

## Gotchas

- Closing the drawing window or the document fires **no** `BeforeWindowClosed` on the anchor window; only the control's `HandleDestroyed`. Listen to the parent window's `BeforeWindowClosed`. Quit with an open panel: no hang.
- Anchor windows of add-ons are not saved: after save + reopen the drawing window has no panel. Reopen it from your own setting.
- `Window.NewWindow()` (View > New Window) gives a second drawing window without the panel: one panel per drawing window.
- Keep panels open in every drawing window from a setting: on `VisioIsIdle`, only when `Application.Windows.Count` changed (or a drawing window closed), add the panel to each window with `Type == visDrawing` and `SubType == visPageWin` (128; skips master and group edit windows) that has none. Measured: `NewWindow`, a reopened drawing, a new drawing and a Visio restart all get it.
- Visio's own panes (Shape Data, Size & Position, External Data, Shapes) in `drawingWindow.Windows` report `Type == visAnchorBarAddon` too; hidden ones have `Caption` `""` and `WindowHandle32` 0. Find your own panels by `MergeID` or keep the reference.
- `anchor.Caption = "..."` renames the pane at runtime (e.g. after a UI language switch).
- Visio is DPI aware: the control gets physical pixels (at 125 %: anchor 580x400 inside, a DPI-unaware script sees 464x320). Test scripts that click need `SetProcessDPIAware()`.
- Same merge ID for two anchor windows of one drawing window: one pane with tabs (as Shape Data / Size & Pos). `visWSAnchorMerged` with a unique ID: own pane under the others on the right.
  - Tabs in the order of `Windows.Add`; the first added is the shown tab when the pane opens. `anchorWindow.Activate()` brings its tab to the front. Both anchors report `Visible == True`; only the shown tab's control window is visible (`IsWindowVisible`). The pane caption is the shown tab's `Caption`.
  - `Close()` of one tab's anchor fires that anchor's `BeforeWindowClosed`, as the pane's X does. Whether Visio closes the other tabs too: unverified (tested only with an add-in that closes all its tabs on this event). If one switch should rule the whole pane, close the others yourself.
  - Each tab is its own reader of shared add-in state: a change list that one reader empties when it reads it loses changes for the other tabs (and for second windows of the drawing). Give every reader its own list.
- Opening does not take the focus from the drawing. `Windows.Add` + control: 10-40 ms.
- Dark Office theme: the WinForms control stays light (system colors).
- Drag inside a `TreeView` (`ItemDrag` -> `DoDragDrop`) works. Dropped onto the drawing page Visio refuses both a custom format and `DataFormats.Text` (`DoDragDrop` returns `None`, no shape made).
- Drag a master instead (Visio 2019, 2026-11): a `Master` RCW is an OLE `IDataObject` (cast works; formats `Visio 15.0 Masters`, `Visio 11.0 Masters`, `Embed Source`, `Object Descriptor`, `NoteshNote`, `EnhancedMetafile`, `MetaFilePict`, `DeviceIndependentBitmap`); `new DataObject(master)` offers the same. The page's drop target is the drawing window itself (`Window.WindowHandle32`, class `VISIOG`, `GetProp(h, "OleDropTargetInterface")`). Its `IDropTarget` called directly (no `DoDragDrop`, no mouse) while another app covered Visio: refused (effect 0, `Drop` S_OK, no shape, no event); `DragEnter` reads only the format `Visio11DynTrackInfo` (not enumerated, still served).
  - Real drag works (WinForms control in Visio's thread: a top-most form on `MouseDown`, or the panel's `TreeView` on `ItemDrag`, both `DoDragDrop(master, DragDropEffects.Copy)`, real mouse): returns `Copy`, the new shape is on the page and is the only selection when it returns; pin at the release point, snapped to the grid; a stencil master is copied into the document stencil; released inside a container it becomes a member. Esc or release over the Shapes window: `None`, no shape. The tree's own `DragOver`/`DragDrop` handlers get the master data while the mouse is over the tree: keep the dragged node in a field, not in the data object.
  - All events of the drop (`EnterScope` "Drop On Page", `ShapeAdded`, `SelectionChanged`, `ExitScope`, the master's `EventDrop` marker) arrive only **after** `DoDragDrop` returned, at the add-in's next Visio call: nothing tells you the drop during the call; the selection does afterwards. (Only the `EnterScope` of an own undo scope opened before the call came during the drag.)
  - Any drawing window takes the drop: a second window of the same drawing (`Window.NewWindow`) or a window of another drawing. The target window becomes active and holds the selection; another drawing gets a copy of the master in its document stencil (also when the master came from the first drawing's document stencil). To refuse a target, check `ActiveWindow.Selection[1].Document` after the return and end your undo scope with `EndUndoScope(id, false)`: shape and master copy are gone, nothing is left on Undo; `ShapeAdded` still arrives afterwards, with `IsUndoingOrRedoing` true, and so does the master's `EventDrop` marker (for a shape that is gone).
  - The drop glues nothing: a 2D master released with its connection point exactly on the free end of a connector leaves the end unglued (no `ConnectionsAdded`). It splits no connector either ([visio-connector-splitting](visio-connector-splitting.md)).
  - One Undo step for drop + own writes: open an undo scope before `DoDragDrop`, write into `ActiveWindow.Selection[1]` after it returns, close the scope (Undo name = the scope's; `Drop On Page` nests inside). Or keep a "pending" flag until `ShapeAdded` comes and write there (Undo name `Drop On Page`). Writes after the return without a scope: two Undo steps.
- `VisioIsIdle` comes after Undo and Redo too; it may fire hundreds of times per second while the UI is busy: keep the handler a flag check.
- UI tests: UI Automation sees the `TreeView` in the panel as a `Pane` without descendants (class `WindowsForms10.SysTreeView32...`; control, raw and content walker, `FindAll`: 0 items). Two ways to read the nodes:
  - Out of process, no add-in code (verified 2026-11, Visio 2019 x64, PowerShell 5.1 x64 with `Add-Type` C#): the WinForms tree is a native tree view. `TVM_GETNEXTITEM` (0x110A; `TVGN_ROOT` 0, `TVGN_NEXT` 1, `TVGN_CHILD` 4) walks all nodes, also children of collapsed nodes. Text and rect need a buffer in the target process: `OpenProcess` (`VM_OPERATION | VM_READ | VM_WRITE | QUERY_INFORMATION`), `VirtualAllocEx` 4 KB. Text: write a `TVITEMW` (x64: `mask` at 0 = `TVIF_TEXT` 1, `hItem` at 8, `pszText` at 24 = buffer + 512, `cchTextMax` at 32; 56 bytes), send `TVM_GETITEMW` (0x113E), `ReadProcessMemory` the UTF-16 text. Rect: the `HTREEITEM` in the first 8 bytes of the buffer, `TVM_GETITEMRECT` (0x1104, `wParam` 1 = text only): 0 = node out of sight (collapsed parent), else client rect -> `ClientToScreen`. Not readable this way: `ForeColor` (use a screenshot), `ToolTipText` (see next point).
  - Node tooltip (`ShowNodeToolTips`): real mouse rest on the node text, `SetCursorPos` in ~6 small steps 150 ms apart (2 steps 200 ms apart showed none); a visible top-level `tooltips_class32` window (no `WindowsForms10.` prefix) of the process appears within ~0.5 s: UI Automation `Name` = the text, lines joined by a newline; `GetWindowText` = empty. The first rest showed it 1 of 5 times; moved out of the tree, rect read again, a second rest showed it 3 of 3 (cause unknown). A node without `ToolTipText` shows none.
  - In process: a debug-only marker command that gets the tree by `Control.FromHandle(<tree hwnd>)` and writes text, level, `ForeColor`, `ToolTipText` and the screen center of `node.Bounds` (`tree.PointToScreen`) to a file; click and drag at those points with the real mouse.
- `TreeView` cost (WinForms, all inside `BeginUpdate`/`EndUpdate`; ~9,000 nodes = 5 parents x 600 children x 2 leaves; in the panel and in a standalone net48 form, 2026-11):
  - Expand a node **before** it is in the tree: `Expand()` on a node without handle is only remembered and done while the TreeView adds it. Refill with all nodes expanded: 194-219 ms vs 1,876-1,903 ms for `Expand()` after adding (standalone); 501-838 ms vs 2,551-3,074 ms in the panel. Collapsed refill: 87-88 ms standalone, ~300 ms in the panel.
  - Add a parent to the tree only **after** its children: a node added without children stays collapsed, the remembered `Expand()` is lost (seen in the panel: all parents collapsed).
  - Never loop over the siblings per inserted node (e.g. "expand as remembered" over the whole parent): 2,910 inserts into parents of 600 took 12.7-13.0 s.
  - Per changed node: text only 0.03 ms; `Remove` + `Insert` 0.42-0.56 ms standalone, in the panel ~1.8 ms when expanded and ~0.13 ms when collapsed (why: unverified). So update only changed nodes for small changes, but refill above about 1 changed node per 30 nodes in the tree (break-even measured all expanded; collapsed, updating stayed faster up to ~2,900 changes).
