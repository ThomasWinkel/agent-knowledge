---
title: Referencing VSTO assemblies from an SDK-style project
description: Read when a non-VSTO SDK-style library (net48) must use VSTO types such as RibbonBase or RibbonButton, e.g. shared ribbon code - search path, dotnet build limit.
---

# Referencing VSTO assemblies from an SDK-style project

Classic VSTO projects reference `Microsoft.Office.Tools.Common` by name only. A plain SDK-style project does not find it that way (`MSB3245 Could not resolve this reference`). The SDK leaves out the Visual Studio search paths.

## Where the types live

- `Microsoft.Office.Tools.Common.dll`: ribbon interfaces (`RibbonButton`, `RibbonTab`, `RibbonGroup`, ...), namespace `Microsoft.Office.Tools.Ribbon`. Also in the GAC.
- `Microsoft.Office.Tools.Common.v4.0.Utilities.dll`: `RibbonBase` (base class of designer ribbons). Not in the GAC.
- Classic VSTO projects get both from `<VS>\Common7\IDE\ReferenceAssemblies\v4.0\`. The search path for that folder comes from `<VS>\MSBuild\Current\Bin\AssemblyFolders.config`.

## Recipe

```xml
<PropertyGroup>
  <AssemblySearchPaths>$(AssemblySearchPaths);{AssemblyFoldersFromConfig:$(MSBuildToolsPath32)\AssemblyFolders.config,v4.8}</AssemblySearchPaths>
</PropertyGroup>
<ItemGroup>
  <Reference Include="Microsoft.Office.Tools.Common, Version=10.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a" Private="False" />
  <Reference Include="Microsoft.Office.Tools.Common.v4.0.Utilities, Version=10.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a" Private="False" />
</ItemGroup>
```

- `MSBuildToolsPath32`, not `MSBuildToolsPath`: `AssemblyFolders.config` is only in `<VS>\MSBuild\Current\Bin\`. The VS IDE builds with the 64-bit MSBuild (`Bin\amd64\`), where `MSBuildToolsPath` gives `MSB4018 ... AssemblyFolders config file specified does not exist: ...\Bin\amd64\AssemblyFolders.config`. The `MSBuild.exe` that `vswhere -find 'MSBuild\**\Bin\MSBuild.exe'` returns first is the one in `Bin\` and hides the problem; test with both.
- Put it in the project body. The SDK fills `AssemblySearchPaths` in its `.props`, so appending here works. Setting `AssemblySearchPath_UseGAC` or `..._UseAssemblyFoldersConfigFileSearchPath` to `true` has no effect. The SDK's list (`Microsoft.NET.Sdk.props`) has only `{CandidateAssemblyFiles}`, `{HintPathFromItem}`, `{TargetFrameworkDirectory}` and `{RawFileName}`.
- `Private="False"`: the consuming VSTO project ships the DLLs itself.

## Gotchas

- Works with Visual Studio MSBuild only. `dotnet build` fails with `MSB4018 ... AssemblyFolders config file specified does not exist: <dotnet sdk>\AssemblyFolders.config`. That is fine: VSTO projects need VS MSBuild anyway.
- `{GAC}` instead of the line above (VS MSBuild) finds `Common` but not `...Utilities`. The dotnet CLI does not search the GAC.

Verified 2026-10: VS 2026 Community (MSBuild 18, `Bin\` and `Bin\amd64\`, VS IDE), .NET SDK 10.0.402, net48.
