---
title: Versions and project format
description: Read when choosing between WiX v3 and v4+, migrating a v3 project, picking the Visual Studio extension, or replacing heat.exe.
---

# Versions and project format

WiX v3 and v4+ are different toolsets. v4, v5, v6 and v7 share the same project format and `.wxs` schema.

| | WiX v3 | WiX v4+ (v7) |
|---|---|---|
| Project | `ToolsVersion="4.0"`, imports `$(WixTargetsPath)` (`Microsoft\WiX\v3.x\Wix.targets`) | SDK-style: `<Project Sdk="WixToolset.Sdk/7.0.0">` |
| `.wxs` namespace | `http://schemas.microsoft.com/wix/2006/wi` | `http://wixtoolset.org/schemas/v4/wxs` (unchanged in v5–v7) |
| Extensions | DLL via `<WixExtension><HintPath>` | NuGet `PackageReference` (`WixToolset.Util.wixext`, `WixToolset.UI.wixext`, `WixToolset.Netfx.wixext`, ...), same version as the SDK |
| CLI | `candle`, `light`, `heat` | `wix.exe` (.NET tool `wix`: `dotnet tool install --global wix`) |
| VS extension | "WiX Toolset Visual Studio 2022 Extension", ID `WixToolset.VisualStudioExtension.Dev17`, v3 only | HeatWave (FireGiant, free), Marketplace ID `FireGiant.FireGiantHeatWaveDev17`, one extension for VS2022 and VS2026 |

- `.slnx` solutions do not build a `.wixproj` by default (`The project "X" is not selected for building in solution configuration "Debug|Any CPU"`, visible with `-v:n`). Add `<Build />` inside its `<Project Path="...wixproj">` element (verified with MSBuild 18).

## Migration from v3

- `wix convert <file.wxs>` rewrites v3 source to the v4+ schema (`wix format` normalizes formatting). The `.wixproj` must be rewritten to SDK style by hand or with HeatWave.
- _Unverified:_ HeatWave's v3 project conversion; not tried.
- v3 and v4+ projects can coexist in one solution; both VS extensions can be installed side by side (different IDs and template folders).
- v3 custom compiler extensions (`.wixext` DLLs built against `wix.dll` 3.x) do not load in v4+. Example: `nbelyh/VisioWixSetup` (`<visio:PublishAddin>`) is v3 only; replace it with plain registry entries ([vsto-addins/registration](../vsto-addins/registration.md)).

## Harvesting: heat.exe is gone in v7

- Heat (`WixToolset.Heat` NuGet) was deprecated in v6 (warning) and removed in v7; its last version is 6.0.2.
- Use the `<Files>` element (since v5) instead: [harvesting-project-output](harvesting-project-output.md).

## HeatWave

- Project templates: `WixMsiPackage`, `WixBundle`, `WixLibrary`, `WixMergeModule`, `WixCppCustomAction`, `WixCSharpCustomAction`; item templates for `.wxs`, `.wxi`, `.wxl`.
- Prerequisite: workload `Microsoft.VisualStudio.Workload.ManagedDesktop`. Check it with `vswhere -requires`: [vs-extension-diagnostics](vs-extension-diagnostics.md).
- Templates missing from "New Project" although HeatWave is installed: [heatwave-templates-missing](heatwave-templates-missing.md).
- Template default configuration is `Debug|x86` → [package-bitness](package-bitness.md).
