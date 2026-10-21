---
title: Clicking a ribbon button from a test script
description: Read when a test must click a button, check box or drop-down of an add-in's custom ribbon tab (VSTO designer or XML) without a person - UI Automation recipe in PowerShell, limits.
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
- While a message box of the add-in is open, the tab is not found (`FindFirst` gives `$null`). Close the box first, then search again from `FromHandle(WindowHandle32)`.
- Click with a modifier key (e.g. Ctrl+click): `Invoke()` carries no keys. Bring the app window to the front, then `keybd_event` key down, real click by `SetCursorPos` + `mouse_event` in the middle of `Current.BoundingRectangle`, key up (user32, via `Add-Type`). Moves the real mouse. While a drop-down is open, do not call `SetForegroundWindow`: it closes the drop-down and the click goes nowhere.
- Prepare the state by COM before the click (`Window.Select`, `Window.DeselectAll`), read the result by COM after it.
- Undo name of the last action (e.g. the name given to `BeginUndoScope`): COM has no getter, but the Undo button of the Quick Access Toolbar is named after it. Search all descendants of the app window for `Name` matching `^Undo ` (English Office UI): `Undo My Action` (verified Visio 2019; two elements with that name). `HelpText` is empty.
- Read the tooltip (ScreenTip + SuperTip) of a button: move the real mouse to the middle of it (`SetCursorPos`), wait ~2.5 s, then find the element of type `ToolTip` among the descendants of the app window; its `Name` = title + SuperTip. For an item in an open drop-down search from `AutomationElement.RootElement`. `HelpText` of the button is empty.
- A message box of the handler: close it from a second process, see "Test without the UI" in [visio-shapesheet-calls](visio-shapesheet-calls.md). In a visible instance filter the dialogs by window class (`#32770` message box, `NUIDialog` Office dialog), or the main window is closed too.
  - Found by UI Automation, its parts show as `Pane` and the OK button has no `InvokePattern`: send `BM_CLICK` (0x00F5) to the button's `Current.NativeWindowHandle` (`SendMessage`).
- Quit without a save prompt: `$doc.Saved = $true; $app.Quit()`.

Verified with Visio Pro 2019 C2R x64, VSTO ribbon designer tab, Windows PowerShell 5.1 (2026-10). _Unverified:_ invisible instance (`Visio.InvisibleApp`), other Office apps, button without selecting the tab first.
