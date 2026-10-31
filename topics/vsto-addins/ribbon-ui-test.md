---
title: Clicking a ribbon button from a test script
description: "Read when a test must click a control of an add-in's custom ribbon tab (VSTO designer or XML) without a person, or drive the dialog it opens - UI Automation and Win32 in PowerShell."
---

# Clicking a ribbon button from a test script

COM automation cannot run the click handler of a custom ribbon button. UI Automation can: the test drives the Office app by COM and clicks the button by its label.

```powershell
Add-Type -AssemblyName UIAutomationClient, UIAutomationTypes
$uia = [System.Windows.Automation.AutomationElement]

$app = New-Object -ComObject Visio.Application          # visible instance
$doc = $app.Documents.Add('')
Start-Sleep -Seconds 2                                   # let the ribbon load

$root = $uia::FromHandle([IntPtr]$app.WindowHandle32)
function Find($name, $type) {
    $cond = New-Object System.Windows.Automation.AndCondition(
        (New-Object System.Windows.Automation.PropertyCondition($uia::NameProperty, $name)),
        (New-Object System.Windows.Automation.PropertyCondition($uia::ControlTypeProperty, $type)))
    $root.FindFirst([System.Windows.Automation.TreeScope]::Descendants, $cond)
}

$tab = Find 'My Tab' ([System.Windows.Automation.ControlType]::TabItem)
$tab.GetCurrentPattern([System.Windows.Automation.SelectionItemPattern]::Pattern).Select()
Start-Sleep -Seconds 1
$button = Find 'My Button' ([System.Windows.Automation.ControlType]::Button)
$button.GetCurrentPattern([System.Windows.Automation.InvokePattern]::Pattern).Invoke()
Start-Sleep -Milliseconds 1500                           # handler runs in the app
```

- Names are the labels of tab and button. A tab or button that is not found is `$null`: the add-in or its ribbon did not load.
- Split button (`RibbonSplitButton`): two elements with its label, `SplitButton` (drop-down) and inside it a `Button` (main part). Invoke the `Button`; `Find` with type `Button` finds it.
- Item in the drop-down of a split button: `Expand()` the `SplitButton` (`ExpandCollapsePattern`), wait, then search the item from `AutomationElement.RootElement` (verified; search from the app window not tried); it is a `MenuItem` with `InvokePattern`.
  ```powershell
  $split.GetCurrentPattern([System.Windows.Automation.ExpandCollapsePattern]::Pattern).Expand(); Start-Sleep -Milliseconds 800
  $cond = New-Object System.Windows.Automation.PropertyCondition($uia::NameProperty, 'My Item')
  $item = $uia::RootElement.FindFirst([System.Windows.Automation.TreeScope]::Descendants, $cond)
  $item.GetCurrentPattern([System.Windows.Automation.InvokePattern]::Pattern).Invoke()
  ```
- The `$button` element stays valid: invoke it again for the next case.
- Check box (`RibbonCheckBox`): type `CheckBox`. `TogglePattern.Toggle()` runs the `Click` handler; read `TogglePattern.Current.ToggleState` (`On`/`Off`) and `Current.IsEnabled`.
- Drop-down (`RibbonDropDown`): type `ComboBox`, named by its label; `ValuePattern.Current.Value` = label of the selected item; `Current.IsEnabled`. A search for all `ListItem`s from `RootElement` also finds list items of other apps (browser tabs): only the first entries are the drop-down's. `Expand()` it (`ExpandCollapsePattern`), wait, find the item as `ListItem` from `AutomationElement.RootElement`, `SelectionItemPattern.Select()`: runs `SelectionChanged`. Items added in code (`Factory.CreateRibbonDropDownItem()` in `Load`) show the same way.
- After a window change (seen in Visio: a stencil opened in its own window, and after closing it again) the custom tab may no longer be selected and its controls are not found: select the tab again, then search.
- While a message box or a modal dialog of the add-in is open, the tab is not found from `FromHandle(WindowHandle32)`: `Application.WindowHandle32` then returns the handle of the dialog, not of the main window (Visio 2019, WinForms dialog). Find the main window by Win32: `EnumWindows`, visible window of the app's process without owner (`GetWindow(h, 4 /*GW_OWNER*/) == 0`), class `VISIOA` for Visio. `EnumWindows` lists topmost first: a message box over a dialog of the same title comes first.
- Click with a modifier key (e.g. Ctrl+click): `Invoke()` carries no keys. Bring the app window to the front, then `keybd_event` key down, real click by `SetCursorPos` + `mouse_event` in the middle of `Current.BoundingRectangle`, key up (user32, via `Add-Type`). Moves the real mouse. While a drop-down is open, do not call `SetForegroundWindow`: it closes the drop-down and the click goes nowhere.
- Prepare the state by COM before the click (`Window.Select`, `Window.DeselectAll`), read the result by COM after it.
- Undo name of the last action (e.g. the name given to `BeginUndoScope`): COM has no getter, but the Undo button of the Quick Access Toolbar is named after it. Search all descendants of the app window for `Name` matching `^Undo ` (English Office UI): `Undo My Action` (verified Visio 2019; two elements with that name). `HelpText` is empty.
  - Seen in another session (Visio 2019, dark theme): the button stayed `Can't Undo` (gray, its list `Undo 0 Action`) after the handler and also after a native nudge by `SendKeys`, while Ctrl+Z worked. Then check Undo itself: app window in front, `SendKeys` `^z`, read the state by COM.
- Read the tooltip (ScreenTip + SuperTip) of a button: move the real mouse to the middle of it (`SetCursorPos`), wait ~2.5 s, then find the element of type `ToolTip` among the descendants of the app window; its `Name` = title + SuperTip. For an item in an open drop-down search from `AutomationElement.RootElement`. `HelpText` of the button is empty.
- A message box of the handler: close it from a second process, see "Test without the UI" in [visio-shapesheet-calls](visio-shapesheet-calls.md). In a visible instance filter the dialogs by window class (`#32770` message box, `NUIDialog` Office dialog), or the main window is closed too.
  - Found by UI Automation, its parts show as `Pane` and the OK button has no `InvokePattern`: send `BM_CLICK` (0x00F5) to the button's `Current.NativeWindowHandle` (`SendMessage`).
- Visio UI command that opens a modal Visio dialog (e.g. `DoCmd(2165)` `visCmdInsertNewBackgroundPage` -> `Page Setup`, `#32770`): the COM call blocks until it is closed. Call it in a second runspace with its own `GetActiveObject('Visio.Application')` (`[PowerShell]::Create().AddScript(...).BeginInvoke()`), then `PostMessage` `BM_CLICK` to its `OK` (found by UI Automation name).
- Quit without a save prompt: `$doc.Saved = $true; $app.Quit()`. Seen once (Visio, after many Undo/Redo by COM): the save prompt came anyway (`NUIDialog`, text `Do you want to save <name>?`) and `Quit` hung; `Invoke()` its `Don't Save` button by UI Automation.

## Modal WinForms dialog of the handler

- `Invoke()` on the button returns while the handler's `Form.ShowDialog()` is still open. Start the driver of the dialog first, as a second process (`Start-Process powershell -File driver.ps1`), then invoke.
- The dialog opens behind the app (the click comes from another process); `SetForegroundWindow` from the driver does not help, also not after `AttachThreadInput` to the foreground thread + `BringWindowToTop`. Drive it without the foreground. _Unverified:_ in front after a real click.
  - Never `SendKeys` (or `keybd_event`) to the dialog without checking `GetForegroundWindow() == dialog` right before: real keys go to whatever window is in front.
- The test's COM calls keep working while the dialog is open. A step that leaves the dialog open (e.g. OK refused by a check) lets later COM steps (`Undo`, writes) run behind it: check the dialog is gone before going on, by `IsWindow(handle)`: UI Automation `Current` keeps answering for a closed window.
- UI Automation shows its controls only as `Pane` without patterns (Name = text, AutomationId = window handle) (VSTO add-in dialog in Visio, .NET Framework 4.8). Use Win32 instead: `FindWindow`, `EnumChildWindows`, `GetClassName` (`WindowsForms10.BUTTON...`, `...LISTBOX...`, `...STATIC...`), `GetWindowText`.
  - PowerShell passes `$null` to a P/Invoke `string` parameter as `""`: `FindWindow($null, 'My Dialog')` finds nothing. Use `[NullString]::Value`.
- Button: `SendMessage(h, 0x00F5 /*BM_CLICK*/, 0, 0)`, also for a check box. `BM_GETCHECK` on a WinForms `CheckBox` returned 0 although it was checked: read states from a screenshot.
  - A button whose handler opens another modal dialog: `SendMessage` returns only when that dialog is closed and the driver hangs. Use `PostMessage(h, 0x00F5, 0, 0)`.
- `GetWindowText` (`WM_GETTEXT`) on a WinForms text box from another process may return the old text after `WM_SETTEXT` or typing: check the result by screenshot or by the app's state.
- `DataGridView` without coordinates: MSAA. `AccessibleObjectFromWindow(grid, 0xFFFFFFFC /*OBJID_CLIENT*/, IID_IAccessible)`; children: header row (role 28 `ROW`, name `Top Row`, column headers role 25), then one role-28 child per data row (`Row 0`, value `a;b`), cells role 29 (name `<header> Row 0`, value = cell text, `(null)` for null).
  - `get_accValue` reads; `set_accValue` sets the cell value without editing (no `CellValidating`; the app's own checks on OK still run); `accSelect(3 /*TAKEFOCUS|TAKESELECTION*/)` makes it the current cell and gives the grid the focus. `accSelect` throws `InvalidOperationException` when the current cell cannot change: the edited cell's `CellValidating` set `e.Cancel` (= value refused).
  - Typing into a cell as a user: while a cell is edited, the focus (`GetGUIThreadInfo(thread).hwndFocus`, no foreground needed) is the editing `WindowsForms10.EDIT...`; `WM_SETTEXT` (SendMessage), then `accSelect` another cell = the user leaves the cell, `CellValidating` runs.
  - Write the MSAA part in C# (`Add-Type -ReferencedAssemblies Accessibility`, `AccessibleChildren` from oleacc.dll): PowerShell cannot pass the returned child objects back as `IAccessible`.
  - Posted `WM_KEYDOWN` Back, Enter, Tab to the editing control are unreliable: in one run Tab and Enter committed, in another the text only grew (`S1` + `X1` + ` neu`). Posted `WM_CHAR` text arrives.
- `DataGridView` by mouse (no child window per cell): post mouse messages in grid client coordinates, `lParam = (y << 16) | x`; take the cell positions from a screenshot.
  - `WM_LBUTTONDOWN` (0x0201, `wParam` 1) + `WM_LBUTTONUP` (0x0202) selects the cell; on a check box cell it also toggles it. A posted Space key did not toggle a check box cell.
  - `WM_LBUTTONDBLCLK` (0x0203) + up on an editable cell starts editing (here the app called `BeginEdit` on double click). The editing control is the visible `WindowsForms10.EDIT...` child of the grid: `WM_SETTEXT`, then post Enter (`WM_KEYDOWN` 0x0D) to commit.
  - Keys for the grid itself (Ins, F2, Del): posted `WM_KEYDOWN` + `WM_KEYUP` to the grid reach its `KeyDown` event.
- `ToolTip.Show(text, control, x, y, ms)` in a dialog that is not the active window (the test case): the tooltip window is made but stays invisible (0x0), also with `ShowAlways = true`.
- WinForms `ListBox` (`...LISTBOX...`): `LB_GETCOUNT` (0x018B), `LB_GETCURSEL` (0x0188) and `LB_GETTEXT` (0x0189, `StringBuilder` parameter) work from another process. Select like a click: `LB_SETCURSEL` (0x0186), then `SendMessage(GetParent(list), 0x0111 /*WM_COMMAND*/, (1 /*LBN_SELCHANGE*/ << 16) | GetDlgCtrlID(list), list)`: raises `SelectedIndexChanged`. Posted `WM_LBUTTONDOWN`, `WM_LBUTTONUP`, `WM_LBUTTONDBLCLK`, `WM_LBUTTONUP` at `x=5, y=5` raised `DoubleClick` on item 0.
- Common file dialog (`OpenFileDialog`, `#32770`, caption = its `Title`): put the path into the `Edit` child whose `GetDlgCtrlID` is 1148 (`WM_SETTEXT`), then `PostMessage(dialog, 0x0111, 1 /*IDOK*/, 0)`; cancel with 2. `WM_GETTEXT` on that edit gave `""` although the dialog showed the preset `FileName` (screenshot).
- Common color dialog (`ColorDialog`, class `#32770`): its title is localized by Windows (`Colour` on en-GB). Set the Red/Green/Blue edits `GetDlgItem(h, 706/707/708)` by `WM_SETTEXT`, then click OK `GetDlgItem(h, 1)`.
- Gotchas of the dialog itself (Esc in a `DataGridView` closes the form): [winforms-dialogs](winforms-dialogs.md).
- `CheckedListBox` item: `SendMessage(list, 0x0186 /*LB_SETCURSEL*/, index, 0)`, then `PostMessage(list, 0x0100 /*WM_KEYDOWN*/, 0x20, 1)` and `WM_KEYUP` (0x0101): toggles it once, with `ItemCheck`. Do not post `WM_CHAR` as well: the app's message loop makes one from the posted `WM_KEYDOWN`, so the item toggles twice. Posted mouse clicks (`WM_LBUTTONDOWN`/`UP` inside `LB_GETITEMRECT`) are unreliable: a click on one item toggled the selected item.
  - Seen in another dialog: Space never toggled item 0 (twice, also when it was the selected item), but did toggle items 2 and 4 of the same list. A posted `WM_LBUTTONDOWN` (`wParam` 1) + `WM_LBUTTONUP` on the check box of item 0 (client `x=10, y=8`, standard item height) toggled it (`CheckOnClick` on, one item in the list). Check every toggle by screenshot.
- `ListView` with check boxes (`WindowsForms10.SysListView32...`): a posted `WM_LBUTTONDOWN` (`wParam` 1) + `WM_LBUTTONUP` on the check box of an item (list client coordinates, e.g. `x=13`, item centre `y`; header ~28 px, item height 20 px at 150 %, take them from a screenshot) toggled it every time, also for an item not selected; on the text it selects the item. Posted `WM_KEYDOWN`/`WM_KEYUP` Space (0x20) toggles the selected item. One toggle per click, with `ItemCheck`.
- Screenshot of a window behind others: `PrintWindow(h, hdc, 2)` (`PW_RENDERFULLCONTENT`) into a bitmap of `GetWindowRect` size. Call `SetProcessDPIAware()` first, else the rect is scaled and the picture cut off.

Verified with Visio Pro 2019 C2R x64, VSTO ribbon designer tab, Windows PowerShell 5.1 (2026-10). _Unverified:_ invisible instance (`Visio.InvisibleApp`), other Office apps, button without selecting the tab first.
