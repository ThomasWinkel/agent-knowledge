---
title: Ctrl+click on a ribbon button opens help
description: Read when a VSTO ribbon should open help on Ctrl+click on all buttons without code per handler - wrap the click handlers, open a local help page at an anchor.
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
- Other containers hold buttons too: `RibbonBox.Items`, `RibbonButtonGroup.Items`, `RibbonGallery.Buttons`, `RibbonDropDown.Buttons` (walking them builds and runs; Ctrl+click there _unverified_).
- Toggle buttons and check boxes: leave them out. Office probably flips the state before the handler runs; skipping the action would leave the state wrong _(unverified)_.

## Run it again (language switch, new entries)

Code that sets ribbon texts at runtime (e.g. a language switch) also resets `SuperTip`, so a help hint there must be added again. Make the hook safe to call any number of times:

- Never wrap a component twice: remember the wrapped ones in a `ConditionalWeakTable<object, object>`.
- Let the wrapper look up the current help table at click time, not the one from the first call: new entries work, removed ones fall back to the action.
- Hint in the SuperTip (e.g. `Ctrl+click: Help`): before adding it, remove the hint added last if the SuperTip still ends with it (the text code may or may not have run in between). Controls without an entry get no hint.
- Order: set texts first, then hook.

## Open a local help page at an anchor

`Process.Start("file:///C:/.../page.html#anchor")` (ShellExecute) opens the page **without** the anchor. Write a redirect page to the temp folder and open that:

```csharp
var target = new Uri(pagePath).AbsoluteUri + "#" + anchor;
var redirect = Path.Combine(Path.GetTempPath(), "MyProduct-help.html");
File.WriteAllText(redirect, $"<!doctype html><meta charset=utf-8><script>location.replace({JsonString(target)})</script>");
Process.Start(redirect);
```

_Open:_ the browser sometimes opens in the background (Firefox as default browser, cold start and running instance). `AllowSetForegroundWindow(ASFW_ANY)` before `Process.Start`: effect unclear (maybe helps when the browser already runs; not with a cold start). Reliable but not simple: wait for the new browser window and call `SetForegroundWindow` on it while the Office app is still in front _(not tried)_.

`JsonString`: any JSON string encoder (quotes and escapes the URL). Append the anchor after `AbsoluteUri` as above: on .NET Framework, `new Uri(fileBaseUri, "page.html#anchor")` escapes `#` to `%23` (`page.html%23anchor`), so split a relative target at `#` before combining. `https://` URLs (e.g. a video with `?t=42`) keep query and anchor with `Process.Start(url)` _(unverified)_.

Verified with Visio Pro 2019 C2R x64, VSTO ribbon designer, real mouse clicks with Ctrl/Shift held (SendInput), 2026-10; anchor test with Firefox as default browser. Also verified in a full implementation: button, split button main part, items in its drop-down; hint shown once after repeated language switches. Finding a help file beside the add-in: [addin-folder](addin-folder.md). _Unverified:_ other Office apps, gallery/check box/toggle button, other browsers (the redirect does not depend on the browser).
