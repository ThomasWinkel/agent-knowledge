---
title: Visio side panel with a WinForms control (anchor window)
description: Read when a Visio add-in shows its own docked panel (tree, list) - anchor window + WinForms control, keys Visio steals, close events, tabs by merge ID, DPI, drag.
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
- Visio is DPI aware: the control gets physical pixels (at 125 %: anchor 580x400 inside, a DPI-unaware script sees 464x320). Test scripts that click need `SetProcessDPIAware()`.
- Same merge ID for two anchor windows of one drawing window: one pane with tabs (as Shape Data / Size & Pos). `visWSAnchorMerged` with a unique ID: own pane under the others on the right.
- Opening does not take the focus from the drawing. `Windows.Add` + control: 10-40 ms.
- Dark Office theme: the WinForms control stays light (system colors).
- Drag inside a `TreeView` (`ItemDrag` -> `DoDragDrop`) works. Dropped onto the drawing page Visio refuses both a custom format and `DataFormats.Text` (`DoDragDrop` returns `None`, no shape made).
- `VisioIsIdle` comes after Undo and Redo too; it may fire hundreds of times per second while the UI is busy: keep the handler a flag check.
- `TreeView` cost (8,689 nodes): `Nodes.Clear` + refill 196-274 ms with collapsed nodes, 1,032-1,876 ms with `ExpandAll`; changing 1-2,000 node texts in `BeginUpdate`/`EndUpdate` 69-133 ms. Update changed nodes instead of rebuilding.
