---
title: User settings location and Office updates
description: Read when a VSTO add-in uses Properties.Settings - where user.config lives, why settings vanish after Office updates, and a safe Settings.Upgrade() pattern.
---

# User settings location and Office updates

`Properties.Settings.Default` of a VSTO add-in is stored in:

```
%LOCALAPPDATA%\Microsoft_Corporation\<AssemblyName>._Path_<hash>\<Office version>\user.config
```

- `Microsoft_Corporation` and `<hash>` come from the host process (VISIO.EXE, WINWORD.EXE, ...), not from the add-in. Two different add-ins showed the same hash.
- The add-in's install location and version do not affect the path.
- Add-in updates and `bin\Debug` vs `Program Files` share the same file. Settings survive updates. A "fresh" installer test therefore is not fresh: rename the version folder first.
- **Every Office update creates a new, empty folder** (Microsoft 365 C2R: about monthly). .NET does not migrate automatically. Users lose settings and tokens on every update. A real case had 16 version folders in 2 years.

## Fix: guarded `Upgrade()`

User-scoped bool setting `UpgradeRequired`, default `True`:

```csharp
if (Properties.Settings.Default.UpgradeRequired)
{
    if (string.IsNullOrEmpty(Properties.Settings.Default.SomeKeySetting))
        Properties.Settings.Default.Upgrade();   // copies from the previous version folder
    Properties.Settings.Default.UpgradeRequired = false;
    Properties.Settings.Default.Save();
}
```

- **The empty-value guard matters.** When the flag first ships, every existing config lacks it and reads `True`. Without the guard, `Upgrade()` overwrites newer current values with older ones.
- **Run it early enough.** `ThisAddIn_Startup` runs after `ThisAddIn` field initializers, which run in the constructor. Put the migration in the constructor of the single class that accesses `Properties.Settings.Default`.

## Verify

```powershell
$x = [xml](Get-Content "<path>\user.config" -Raw)
$x.SelectSingleNode("//setting[@name='SomeKeySetting']/value").InnerText
```

Use this XPath. `$_.value.InnerText` via PowerShell's XML adapter returns empty strings here and suggests lost settings. After an Office update: value unchanged and `UpgradeRequired` = `False`.

_Unverified:_ whether `<hash>` changes when Office moves (MSI vs C2R install path); whether `Upgrade()` finds the last used version across several skipped Office versions.
