---
title: Ctrl+click on a ribbon button opens help
description: Read when a VSTO ribbon should react to a modifier key (e.g. Ctrl+click = help) on all buttons without code per handler - wrap the click handlers, read the key, open a local help page at an anchor.
---

# Ctrl+click on a ribbon button opens help

A modifier key on a ribbon click can run something else (help page, video) instead of the button's action. The public VSTO API cannot do it generically: `Click` handlers cannot be read or cancelled, and a handler added later runs after the designer's. Wrapping the private handler field works.

## Where VSTO keeps the handlers

`Microsoft.Office.Tools.Common.Implementation` (VSTO runtime 10.0, GAC):

- `RibbonButtonImpl.Click` adds to `properties.ControlAction`; field `properties` (`RibbonPropertyStorage`) is declared in base `RibbonComponentImpl`.
- `RibbonPropertyStorage.controlAction` (`RibbonControlEventHandler`) holds all handlers; the `onAction` callback of Office calls it. Same storage for toggle button, check box, gallery etc.
- `RibbonSplitButtonImpl.Click` forwards to its inner button: private field `button` (`RibbonButtonImpl`). The split button's own storage has no handler.
- Handler `sender`: the control (split button: the split button); `e.Control.Id`: `<name>` (split button main part: `<name>__btn`).

## Recipe

Call in the ribbon's `Load`, after the designer has added the handlers.

```csharp
const BindingFlags Any = BindingFlags.Instance | BindingFlags.NonPublic | BindingFlags.Public;

static void HookAll(RibbonBase ribbon, Func<string, bool> tryShowHelp)
{
    foreach (var tab in ribbon.Tabs)
        foreach (var group in tab.Groups)
            foreach (var item in group.Items)
                Hook(item, tryShowHelp);
}

static void Hook(RibbonControl control, Func<string, bool> tryShowHelp)
{
    Wrap(control, control.Name, tryShowHelp);
    if (control is RibbonSplitButton split)
    {
        Wrap(split.GetType().GetField("button", Any)!.GetValue(split), control.Name, tryShowHelp);
        foreach (var item in split.Items) Hook(item, tryShowHelp);
    }
    else if (control is RibbonMenu menu)
        foreach (var item in menu.Items) Hook(item, tryShowHelp);
}

static void Wrap(object component, string name, Func<string, bool> tryShowHelp)
{
    FieldInfo? propertiesField = null;
    for (var t = component.GetType(); t != null && propertiesField == null; t = t.BaseType)
        propertiesField = t.GetField("properties", Any | BindingFlags.DeclaredOnly);
    var properties = propertiesField?.GetValue(component);
    var actionField = properties?.GetType().GetField("controlAction", Any);
    if (actionField?.GetValue(properties) is not RibbonControlEventHandler original)
        return;                                    // no handler (separator, label, split button itself)

    actionField.SetValue(properties, (RibbonControlEventHandler)((sender, e) =>
    {
        if ((Control.ModifierKeys & Keys.Control) != 0 && tryShowHelp(name))
            return;                                // help shown: skip the action
        original(sender, e);
    }));
}
```

- `System.Windows.Forms.Control.ModifierKeys` in the handler gives the keys held during the click: `Control` seen for a button, the main part of a split button and an item in its drop-down; `Shift` for a drop-down item (others not tried).
- No help entry for the control: `tryShowHelp` returns false, the action runs as usual.
- Avoid Alt: Office shows its KeyTips.
- Private fields: wrap in try/catch and log; if VSTO ever changes them, buttons keep working without help.

## Open a local help page at an anchor

`Process.Start("file:///C:/.../page.html#anchor")` (ShellExecute) opens the page **without** the anchor. Write a redirect page to the temp folder and open that:

```csharp
var target = new Uri(pagePath).AbsoluteUri + "#" + anchor;
var redirect = Path.Combine(Path.GetTempPath(), "MyProduct-help.html");
File.WriteAllText(redirect, $"<!doctype html><meta charset=utf-8><script>location.replace({JsonString(target)})</script>");
Process.Start(redirect);
```

`JsonString`: any JSON string encoder (quotes and escapes the URL). `https://` URLs (e.g. a video with `?t=42`) keep query and anchor with `Process.Start(url)` _(unverified)_.

Verified with Visio Pro 2019 C2R x64, VSTO ribbon designer, real mouse clicks with Ctrl/Shift held (SendInput), 2026-10; anchor test with Firefox as default browser. _Unverified:_ other Office apps, gallery/check box/toggle button, other browsers (the redirect does not depend on the browser).
