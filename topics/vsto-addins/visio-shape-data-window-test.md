---
title: Editing a value in Visio's Shape Data window from a test script
description: "Read when a test must type into Visio's anchored Shape Data window like a user (not by COM), e.g. to check what a user edit does to SETATREF or GUARD cells - window messages, no focus needed."
---

# Editing a value in Visio's Shape Data window from a test script

A COM write (`Cell.FormulaU`) is not the same code path as a user edit. To test the user path, drive the anchored Shape Data window. Measured with Visio 2019 C2R x64 (2026-10).

- Show it by COM: `$app.ActiveWindow.Windows.ItemFromID(1658).Visible = $true` (`visWinIDCustProp`); select the shape first. With nothing selected (`DeselectAll`) it shows the page's Shape Data: pane name `Shape Data - ThePage` (2026-10-31).
- UI Automation sees only a pane named `Shape Data - <shape name>` with a `ListBox` child (owner-drawn, no item elements). Take its `NativeWindowHandle`.
- One list item per visible row, in section order. `LB_GETCOUNT` (0x018B) gives the count, `LB_GETITEMRECT` (0x0198) the item rectangle.
- A click on the value part (right half of the item) opens an in-place editor: a new visible `RichEdit20W` child of the Visio main window (`VISIOA`).
- Set its text with `WM_SETTEXT` and post `WM_KEYDOWN`/`WM_KEYUP` with `VK_RETURN`: Visio commits the value as a user edit, one Undo step.

Use window messages, not `SetForegroundWindow` + `SendKeys` or `mouse_event`: from a background process Windows usually refuses to bring Visio to the front, and the keystrokes go to the window that has the focus.

```powershell
$lp = [IntPtr](($y -shl 16) -bor $x)                         # client coords inside the item
[Win]::PostMessage($listBox, 0x0201, [IntPtr]1, $lp)         # WM_LBUTTONDOWN
[Win]::PostMessage($listBox, 0x0202, [IntPtr]0, $lp)         # WM_LBUTTONUP
Start-Sleep -Milliseconds 600                                # editor appears
[Win]::SendText($editor, 0x000C, [IntPtr]::Zero, "10A")      # WM_SETTEXT (SendMessageW, string lParam)
[Win]::PostMessage($editor, 0x0100, [IntPtr]0x0D, [IntPtr]0x001C0001)   # WM_KEYDOWN Enter
[Win]::PostMessage($editor, 0x0101, [IntPtr]0x0D, [IntPtr]0xC01C0001)   # WM_KEYUP Enter
```

- The editor stays open after the click until Enter; a second click script finds no *new* editor, so look for the visible `RichEdit20W` too.
- Fixed list row (`Type = 1`): the click opens a `REComboBox20W` (`CBS_DROPDOWNLIST`) as child of the Shape Data list box, not a `RichEdit20W`. Choose by `CB_SETCURSEL` (0x014E, wParam = item index), then post Enter to the combo; `WM_SETTEXT` does nothing. Items: `CB_GETCOUNT` (0x0146), `CB_GETLBTEXT` (0x0148). What the choice writes: [visio-cell-values](visio-cell-values.md#fixed-list-chosen-by-the-user).
- Verified: a text row and a number row (`7` -> `7`), each edit one Undo step; fixed list rows on a shape and on the page.
