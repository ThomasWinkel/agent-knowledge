---
title: WinForms dialogs in add-ins - DataGridView gotchas
description: "Read when an add-in dialog edits values in a DataGridView: Esc closes the whole form (CancelButton), error icons that do not show, check boxes that apply late."
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

## Errors and commits

- `DataGridViewRow.ErrorText` shows only in the row header: invisible with `RowHeadersVisible = false`. A cell's `ErrorText` icon is covered while the cell is edited. For a value rejected in `CellValidating` (`e.Cancel = true`, the cell stays in edit mode) show the reason elsewhere, e.g. `ToolTip.Show(text, grid, cell.Left, cell.Bottom, 3000)` with `cell = grid.GetCellDisplayRectangle(col, row, false)`.
- A check box cell changes its value (`CellValueChanged`) only when the cell is left. To act on the click: `CurrentCellDirtyStateChanged` -> `if (grid.IsCurrentCellDirty && grid.CurrentCell is DataGridViewCheckBoxCell) grid.CommitEdit(DataGridViewDataErrorContexts.Commit);`.
- Filling the grid again (`Rows.Clear()`, `Rows.Add`) inside its own cell events can throw (reentrant call to `SetCurrentCellAddressCore`). Refill after the event: `BeginInvoke(new Action(Refill))`.
- `EditMode = EditOnKeystroke` keeps F2 free for an own command (e.g. Rename); then start editing on double click with `grid.BeginEdit(true)` in `CellDoubleClick`.

Verified with a VSTO add-in dialog in Visio Pro 2019 C2R x64, .NET Framework 4.8, keys and clicks posted by Win32 messages ([ribbon-ui-test](ribbon-ui-test.md)), 2026-10. _Unverified:_ Esc from a real keyboard (posted `WM_KEYDOWN` takes the same path); the reentrancy exception (avoided, not provoked); the tooltip in an active dialog (not visible in the test, see ribbon-ui-test).
