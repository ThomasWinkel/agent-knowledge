---
title: WinForms dialogs in add-ins - DataGridView gotchas
description: "Read when an add-in dialog edits values in a DataGridView: Esc closes the whole form (CancelButton), first focus on OK, AltGr characters lost, error icons that do not show, late check boxes."
---

# WinForms dialogs in add-ins - DataGridView gotchas

A WinForms dialog with OK/Cancel (`AcceptButton`, `CancelButton`) and an editable `DataGridView`.

## Esc closes the form while a cell is edited

With `CancelButton` set, Esc in a cell being typed in closes the whole dialog (`DialogResult.Cancel`) instead of only taking back the cell. If Cancel undoes all the dialog's changes, the user loses them. Catch Esc in the form while a grid has an editing control:

```csharp
protected override bool ProcessCmdKey(ref Message msg, Keys keyData)
{
    if (keyData == Keys.Escape && myGrid.EditingControl is not null)
    {
        myGrid.CancelEdit();   // cell back to its value
        myGrid.EndEdit();
        return true;
    }
    return base.ProcessCmdKey(ref msg, keyData);
}
```

## First focus on OK instead of the grid

A `UserControl` holding the grid, added in code into a designer panel that already holds the OK/Cancel panel (`TabIndex` 1), gets the next free `TabIndex` (2): the dialog opens with the focus on OK, and keys meant for the grid (Ins, Del, F2) do nothing until the user clicks into it. Give it the first place:

```csharp
myControl = new MyGridControl { Dock = DockStyle.Fill, TabIndex = 0 };
layoutPanel.Controls.Add(myControl, 0, 0);
```

Verified: before, focus on OK (`GetGUIThreadInfo`); after, focus in the grid and a posted Ins right after open reached its `KeyDown`.

## AltGr characters do not start editing

With `EditMode = EditOnKeystroke` or `EditOnKeystrokeOrF2`, a character typed into a cell that is not in edit mode starts editing, except one typed with AltGr (Ctrl+Alt): `\ [ ] { } | @ ~` on a German keyboard. That character is lost; the next one starts editing. A regex typed as `\d+` arrives as `d+`. The character still reaches the grid's `KeyPress`: start editing there.

```csharp
private void MyGrid_KeyPress(object sender, KeyPressEventArgs e)
{
    if (myGrid.IsCurrentCellInEditMode || char.IsControl(e.KeyChar) || myGrid.CurrentCell is null)
        return;
    if (!myGrid.BeginEdit(false) || myGrid.EditingControl is not TextBox box)
        return;
    box.Text = e.KeyChar.ToString();   // replaces the text, as a plain key does
    box.SelectionStart = box.Text.Length;
    e.Handled = true;
}
```

Verified for `\` (`SendKeys` with German layout into a text cell not in edit mode, `\d{+}`: before `d+`, after `\d+`). _Unverified:_ the other AltGr characters, a real keyboard.

## Errors and commits

- `DataGridViewRow.ErrorText` shows only in the row header: invisible with `RowHeadersVisible = false`. A cell's `ErrorText` icon is covered while the cell is edited. For a value rejected in `CellValidating` (`e.Cancel = true`, the cell stays in edit mode) show the reason elsewhere, e.g. `ToolTip.Show(text, grid, cell.Left, cell.Bottom, 3000)` with `cell = grid.GetCellDisplayRectangle(col, row, false)`.
- A check box cell changes its value (`CellValueChanged`) only when the cell is left. To act on the click: `CurrentCellDirtyStateChanged` -> `if (grid.IsCurrentCellDirty && grid.CurrentCell is DataGridViewCheckBoxCell) grid.CommitEdit(DataGridViewDataErrorContexts.Commit);`.
- Filling the grid again (`Rows.Clear()`, `Rows.Add`) inside its own cell events can throw (reentrant call to `SetCurrentCellAddressCore`). Refill after the event: `BeginInvoke(new Action(Refill))`.
- `EditMode = EditOnKeystroke` keeps F2 free for an own command (e.g. Rename); then start editing on double click with `grid.BeginEdit(true)` in `CellDoubleClick`.

Verified with a VSTO add-in dialog in Visio Pro 2019 C2R x64, .NET Framework 4.8, keys and clicks posted by Win32 messages ([ribbon-ui-test](ribbon-ui-test.md)), 2026-10; AltGr by `SendKeys` (German keyboard layout), 2026-11. _Unverified:_ Esc from a real keyboard (posted `WM_KEYDOWN` takes the same path); the reentrancy exception (avoided, not provoked); the tooltip in an active dialog (not visible in the test, see ribbon-ui-test).
