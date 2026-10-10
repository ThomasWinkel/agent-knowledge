---
title: Publishing via the PublishComponent table
description: Read when publishing Visio stencils/templates from an MSI - PublishComponent GUIDs per Visio version, Qualifier and AppData format, WiX <Category>, ConfigChangeID bump.
---

# Publishing via the PublishComponent table

One `PublishComponent` row per file (and per Visio version GUID, per language). Visio enumerates the rows (qualified components) and lists the files under "New" (templates) and "More Shapes" (stencils).

Verified with Visio Professional 2019 C2R x64 (16.0 build 20430), WiX 7.0.0, x64 per-machine package (2026-10): install, display, uninstall.

## Columns

| Column | Content |
|---|---|
| `ComponentId` | fixed Visio category GUID by Visio version and content type (table below); not a component GUID |
| `Qualifier` | `<LCID>\<file name>`, e.g. `1\Flow_M.vstx`. `1` = all languages (verified). Add-ons: `<LCID>\<ordinal-1>\<file>`. `<LCID>\<file>` must be unique per Visio installation |
| `Component_` | component that installs the file |
| `AppData` | menu path and options, format below |
| `Feature_` | feature of that component |

## ComponentId GUIDs

Pattern: last digit = content type (`0` template, `1` stencil, `2` help, `3` add-on).

| Visio | Base | Template | Stencil |
|---|---|---|---|
| 2003, "version-neutral" | `{CF1F488D-8D6F-499C-A78D-026E1DF3810x}` | `...38100` | `...38101` |
| 2007 | `{6D9D8B6F-D0EF-4BC0-8DD4-09DD6CE2B00x}` | `...B000` | `...B001` |
| 2010 | `{6D9D8B6F-D0EF-4BC0-8DD4-09DD6CE2B10x}` | `...B100` | `...B101` |
| 2013 | `{6D9D8B6F-D0EF-4BC0-8DD4-09DD6CE2B20x}` | `...B200` | `...B201` |
| 2016+ | `{6D9D8B6F-D0EF-4BC0-8DD4-09DD6CE2B30x}` | `...B300` | `...B301` |

- **Visio 2016 and later (incl. 2019, Plan 2) read only `B30x`.** Verified on 2019: rows with the "version-neutral" `CF1F...` IDs (LCID 1033/1031) and 2013 `B20x` IDs were ignored, `B30x` rows showed up. bVisual reports the same for Visio Plan 2.
- Microsoft documents only the 2003 and 2007 IDs; 2010/2013 come from nbelyh/VisioWixSetup; `B30x` is not documented anywhere.
- To support several versions, add one row per GUID; other versions ignore foreign rows.

## AppData

| Visio | Stencil | Template |
|---|---|---|
| 2003 | `MenuPath\|AltNames` | `MenuPath\|AltNames` |
| 2007 | `MenuPath\|AltNames` | `MenuPath\|AltNames\|Featured` |
| 2010+ | `MenuPath\|AltNames\|QuickShapes\|Edition` | `MenuPath\|AltNames\|1\|Edition` |

- `MenuPath`: category path with `\`, last part = display name, e.g. `My Company\Electrical`. Templates appear under "New" → Categories → first part. Empty or a part starting with `_` = hidden.
- File name ending `_M` / `_U`: Visio appends " (Metric)" / " (US units)" to the display name (verified for `_M`). A `_M`/`_U` pair with the same name gives a unit choice. Lowercase `_m` gave no suffix (verified on 2019).
- `AltNames`: `;`-separated alternate names; Visio uses these, not the file's own `AlternateNames`.
- `QuickShapes`: number of masters shown as quick shapes (`0` = default).
- `Edition`: `-1` both, `32` or `64` Visio bitness only. Verified: `-1` works on 64-bit Visio.
- `Featured` (2007): `1` = featured template. nbelyh writes `1` in that position for 2010+ (used in the verified test).

## WiX v4+ (`<Category>`)

`<Category>` (child of `<Component>`) writes the row. Verified in WiX 7:

```xml
<Component Directory="StencilFolder">
  <File Source="Electrical_M.vssx" />
  <Category Id="{6D9D8B6F-D0EF-4BC0-8DD4-09DD6CE2B301}"
            Qualifier="1\Electrical_M.vssx"
            AppData="My Company\Electrical||0|-1" />
</Component>
<Component Directory="TemplateFolder">
  <File Source="Wiring_M.vstx" />
  <Category Id="{6D9D8B6F-D0EF-4BC0-8DD4-09DD6CE2B300}"
            Qualifier="1\Wiring_M.vstx"
            AppData="My Company\Wiring||1|-1" />
</Component>
```

- `Feature` attribute is optional; WiX fills `Feature_` with the component's feature.
- Check the built rows: `$db = (New-Object -ComObject WindowsInstaller.Installer).OpenDatabase($msi, 0)`, then `$db.OpenView("SELECT * FROM PublishComponent")`, `.Execute()`, `.Fetch()`, `.StringData(n)`.
- nbelyh/VisioWixSetup (WiX v3 extension, last change 2013) generated these rows; it does not load in v4+ ([wix-toolset/versions-and-project-format](../wix-toolset/versions-and-project-format.md)).

## ConfigChangeID

- **Required.** After install and after uninstall, change `HKLM\Software\Microsoft\Office\Visio\ConfigChangeID` (REG_DWORD); Visio then rebuilds its content cache on next start. Verified on 2019: after install without a change, Visio did not rewrite the cache; after changing the value it did, and the content appeared. Same for removal after uninstall.
- Visio 2019 C2R x64 reads the native (64-bit) view; the value existed there (`0`), nothing under `WOW6432Node` or the C2R virtual registry. _Unverified:_ which view 32-bit Visio reads.
- MSI cannot increment a value: needs a deferred, non-impersonated custom action (old tools: VBScript `VisSolPublish_BumpVisioChangeId`; Windows is phasing out VBScript, use a compiled custom action).
- Verified recipe: C# custom action ([wix-toolset/managed-custom-actions](../wix-toolset/managed-custom-actions.md)), last before `InstallFinalize`, on install and uninstall. Count up in both registry views; a view without the key means no Visio of that bitness. After such an install Visio 2019 x64 listed the stencil and the template.

  ```csharp
  // using Microsoft.Win32;
  const string KeyPath = @"Software\Microsoft\Office\Visio";
  const string ValueName = "ConfigChangeID";

  public static void BumpAllViews(Action<string> log)
  {
      foreach (var view in new[] { RegistryView.Registry64, RegistryView.Registry32 })
      {
          using var hklm = RegistryKey.OpenBaseKey(RegistryHive.LocalMachine, view);
          using var key = hklm.OpenSubKey(KeyPath, writable: true);
          if (key is null) { log($"{ValueName}: no Visio key in {view}, skipped."); continue; }

          var oldValue = key.GetValue(ValueName) as int? ?? 0;   // missing or wrong type counts as 0
          var newValue = unchecked(oldValue + 1);                // 0xFFFFFFFF wraps to 0
          key.SetValue(ValueName, newValue, RegistryValueKind.DWord);
          log($"{ValueName}: set to {newValue} in {view}.");
      }
  }
  ```

  Unit-testable: pass a throwaway `HKCU` key to the inner part instead of the Visio key.
- Visio start via COM (`Visio.Application`/`InvisibleApp`) rebuilds only add-on entries in the cache; stencils/templates are added when the UI shows them. Check publishing in the UI, not via a COM start.
- bVisual (2024): after the change, some stencils still showed the file name instead of the published display name (32-bit Visio, non-ASCII names); deleting `content16.dat` fixed it.

## Sources

- [Publishing Visio 2007 Solutions (Microsoft)](https://learn.microsoft.com/en-us/previous-versions/office/developer/office-2007/bb677166(v=office.12))
- [nbelyh/VisioWixSetup](https://github.com/nbelyh/VisioWixSetup): `VisioWixExtension/wixext/VisioCompilerExtension.cs` (row format), `ca/CustomAction.cpp` (ConfigChangeID)
- [Installing Visio Templates and Stencils (bVisual, 2026)](https://bvisual.net/2026/02/22/installing-visio-templates-and-stencils/)
- [Refreshing the cached installed files of Visio (bVisual, 2024)](https://bvisual.net/2024/08/06/refreshing-the-cached-installed-files-of-visio/)
