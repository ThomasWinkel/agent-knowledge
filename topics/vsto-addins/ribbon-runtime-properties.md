---
title: Changing ribbon properties at runtime
description: Read when code sets properties of a VSTO designer ribbon after it is created (e.g. translated Label/ScreenTip/SuperTip in Load) - which can be set, "Component is read only", how to check.
---

# Changing ribbon properties at runtime

A VSTO designer ribbon (`RibbonBase`) is loaded when its `Load` event runs. After that, a property can be set only if its setter stores the value as dynamic.

## Text properties

- `Label`, `ScreenTip`, `SuperTip`, `KeyTip`, `Title` of tab, group, dialog launcher, button, toggle button, check box, split button, menu, gallery, drop-down (and its items), combo box, edit box, label: settable at any time (`RibbonPropertyStorage.SetDynamicValue`). Office updates the UI at once.
- Drop-down items: `Items.Clear()` / `Items.Add(Factory.CreateRibbonDropDownItem())` and `SelectedItem` work at any time (verified: rebuilt many times after load); item `Label`, `Tag`, `OfficeImageId` dynamic.
- **`RibbonDropDown.SizeString`: only before load** (`SetStaticValue`): set it in the designer. Value = sample text, not a character count: the control gets the width of this text (`20` -> two digits wide); e.g. `xxxxxxxxxxxxxxxxxxxx` for about 20 characters.
- **`RibbonSeparator.Title`: only before load** (`TitleSetStaticValue`). Setting it in `Load` throws `InvalidOperationException: Component is read only. Cannot change value of this property.` Even setting it to its current value (`null`) throws.

## Gotcha: exception in Load hides the whole tab

An exception in the ribbon's `Load` handler gives no dialog and no event log entry. The add-in stays loaded (`COMAddIns` shows `Connect=True`), but the custom tab is missing. To see the exception, catch it temporarily and write it to a file:

```csharp
private void MyRibbon_Load(object sender, RibbonUIEventArgs e)
{
    try { DoWork(); }
    catch (Exception ex) { File.WriteAllText(Path.Combine(Path.GetTempPath(), "ribbon-load.txt"), ex.ToString()); throw; }
}
```

## Check any property

The implementation classes (`Ribbon*Impl`) are in `Microsoft.Office.Tools.Common.Implementation` (GAC). Scan the setter IL for the methods it calls (PowerShell 5.1):

```powershell
$asm = [Reflection.Assembly]::LoadWithPartialName('Microsoft.Office.Tools.Common.Implementation')
$t = $asm.GetTypes() | Where-Object Name -eq 'RibbonSeparatorImpl'
$set = $t.GetProperty('Title', [Reflection.BindingFlags]'Public,NonPublic,Instance').SetMethod
$il = $set.GetMethodBody().GetILAsByteArray()
for ($i = 0; $i -lt $il.Length - 4; $i++) {
    if ($il[$i] -eq 0x28 -or $il[$i] -eq 0x6F) {   # call, callvirt
        try { $t.Module.ResolveMethod([BitConverter]::ToInt32($il, $i + 1)).Name } catch { }
    }
}
```

- Most `*Impl` setters call `RibbonPropertyStorage.set_<Name>`; check that setter the same way (`RibbonPropertyStorage` type). `SetDynamicValue` = runtime OK, `SetStaticValue` / `<Name>SetStaticValue` = before load only.

Verified 2026-10: Visio 2019 C2R x64, VSTO runtime 10.0 (Microsoft.Office.Tools.Common.Implementation 10.0.0.0). Other Office apps share the same assembly; not tested there.
