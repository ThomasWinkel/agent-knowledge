---
title: Publishing via the PublishComponent table
description: Read when publishing Visio stencils/templates from an MSI - PublishComponent GUIDs per Visio version, Qualifier and AppData format, WiX <Category>, ConfigChangeID bump.
---

# Publishing via the PublishComponent table

_Unverified as a whole:_ compiled from the sources below, not yet tested.

One `PublishComponent` row per file (and per Visio version GUID, per language). Visio enumerates the rows (qualified components) and lists the files in its UI.

## Columns

| Column | Content |
|---|---|
| `ComponentId` | fixed Visio category GUID by Visio version and content type (table below); not a component GUID |
| `Qualifier` | `<LCID>\<file name>`, e.g. `1033\Flow_M.vstx`. `1` = all languages. Add-ons: `<LCID>\<ordinal-1>\<file>`. `<LCID>\<file>` must be unique per Visio installation |
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

- Microsoft documents only 2003 and 2007 IDs. 2010/2013 come from nbelyh/VisioWixSetup; 2016+ (`B30x`) was extrapolated by bVisual (2026) and reported working with Visio Plan 2.
- bVisual: the "version-neutral" `CF1F...` IDs showed nothing in Visio Plan 2. To cover several versions, add one row per GUID.
- Same file name for 2003 and 2007+ rows: Visio 2007 uses only the 2007 row.

## AppData

| Visio | Stencil | Template |
|---|---|---|
| 2003 | `MenuPath\|AltNames` | `MenuPath\|AltNames` |
| 2007 | `MenuPath\|AltNames` | `MenuPath\|AltNames\|Featured` |
| 2010+ | `MenuPath\|AltNames\|QuickShapes\|Edition` | `MenuPath\|AltNames\|1\|Edition` |

- `MenuPath`: category path with `\`, last part = display name, e.g. `My Company\Electrical`. Empty or a part starting with `_` = hidden.
- `AltNames`: `;`-separated alternate names; Visio uses these, not the file's own `AlternateNames`.
- `QuickShapes`: number of masters shown as quick shapes (`0` = default).
- `Edition`: `-1` both, `32` or `64` Visio bitness only.
- `Featured` (2007): `1` = featured template. nbelyh writes `1` in that position for 2010+.
- Template units: file names ending `_M` (metric) / `_U` (US); a `_M`/`_U` pair gives a unit choice in the UI.

## WiX v4+ (`<Category>`)

`<Category>` (child of `<Component>`, attributes `Id`, `Qualifier`, `AppData`, `Feature`; still in v7) writes the row:

```xml
<Component Directory="StencilFolder">
  <File Source="Electrical_M.vssx" />
  <Category Id="{6D9D8B6F-D0EF-4BC0-8DD4-09DD6CE2B301}"
            Qualifier="1\Electrical_M.vssx"
            AppData="My Company\Electrical||0|-1" />
</Component>
```

nbelyh/VisioWixSetup (WiX v3 extension, last change 2013) generated these rows; it does not load in v4+ ([wix-toolset/versions-and-project-format](../wix-toolset/versions-and-project-format.md)).

## ConfigChangeID

- After install and uninstall, increment `HKLM\Software\Microsoft\Office\Visio\ConfigChangeID` (REG_DWORD) so Visio rebuilds its content cache.
- MSI cannot increment a value: needs a deferred, non-impersonated custom action. Old tools used VBScript (`VisSolPublish_BumpVisioChangeId`); Windows is phasing out VBScript, so use a compiled custom action.
- Present in the native (64-bit) view on Visio 2019 C2R x64 (value `0`); nothing under `WOW6432Node` or the C2R virtual registry. _Unverified:_ which view 32-bit Visio reads.
- bVisual (2024): after the bump, some stencils still showed the file name instead of the published display name (32-bit Visio, non-ASCII names); deleting `content16.dat` fixed it.

## Sources

- [Publishing Visio 2007 Solutions (Microsoft)](https://learn.microsoft.com/en-us/previous-versions/office/developer/office-2007/bb677166(v=office.12))
- [nbelyh/VisioWixSetup](https://github.com/nbelyh/VisioWixSetup): `VisioWixExtension/wixext/VisioCompilerExtension.cs` (row format), `ca/CustomAction.cpp` (ConfigChangeID)
- [Creating an installer for Visio with WiX (Unmanaged Visio)](https://unmanagedvisio.com/creating-an-installer-for-visio-with-wix/)
- [Installing Visio Templates and Stencils (bVisual, 2026)](https://bvisual.net/2026/02/22/installing-visio-templates-and-stencils/)
- [Refreshing the cached installed files of Visio (bVisual, 2024)](https://bvisual.net/2024/08/06/refreshing-the-cached-installed-files-of-visio/)
