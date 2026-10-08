---
title: Harvesting project output
description: Read when adding another project's build output to a WiX v4+ package - ProjectReference, bindpath, <Files> syntax, Exclude, WIX8600, and checking the built MSI's contents.
---

# Harvesting project output

Verified with WiX 7.0.0 and a classic (non-SDK) .NET Framework VSTO project.

## ProjectReference

```xml
<Project Sdk="WixToolset.Sdk/7.0.0">
  <ItemGroup>
    <ProjectReference Include="..\MyAddin\MyAddin.csproj" />
  </ItemGroup>
</Project>
```

- Creates bindpath `!(bindpath.MyAddin)` (project name) → the project's normal output folder (`bin\<Configuration>\`).
- No `Publish="true"`: it runs `dotnet publish`, which does not work for VSTO/.NET Framework projects.
- x64 package referencing an AnyCPU-only project needs `SetPlatform`: [package-bitness](package-bitness.md).

## `<Files>`

`<Files>` is a child of a `ComponentGroup`/`Directory`; it has no `ComponentGroup` or `Exclude` attribute (both give `WIX0004 ... unexpected attribute`). Exclusions are child elements:

```xml
<Fragment>
  <ComponentGroup Id="AddinFiles" Directory="INSTALLFOLDER">
    <Files Include="!(bindpath.MyAddin)\**">
      <Exclude Files="!(bindpath.MyAddin)\**\*.pdb" />
      <Exclude Files="!(bindpath.MyAddin)\**\*.xml" />
      <Exclude Files="!(bindpath.MyAddin)\**\*.log" />
    </Files>
  </ComponentGroup>
</Fragment>
```

- One component per harvested file. Reference with `<ComponentGroupRef Id="AddinFiles" />` in a `<Feature>`.
- `**` is recursive and keeps the subfolder structure (`runtimes\win-x64\native\...` verified).
- Wildcards take everything in the folder, including leftovers (diagnostic logs, stale files). Exclude `*.log`: VSTO diagnostics write `<name>.vsto.log` next to the `.vsto`.
- v7: relative `Include` paths resolve against the source (`.wxs`) file (wix issue 9097). Bindpath-based paths are not affected.

## Gotchas

- Zero matches is only a warning: `warning WIX8600: Inclusions and exclusions resulted in zero files harvested.` Treat it as an error.
- Output file names come from `AssemblyName`, not the project file name (`MyAddin.csproj` may produce `CompanyProduct.dll`). List `bin\<Configuration>\` before writing explicit `Include` paths.

## Check the built MSI's contents

A clean build does not prove the right files are inside. Administrative install extracts without installing:

```powershell
$p = Start-Process msiexec.exe -ArgumentList '/a "Setup.msi" /qn TARGETDIR="C:\temp\extract" /L*v "C:\temp\admin.log"' -PassThru -Wait
$p.ExitCode   # 0 = ok
Get-ChildItem C:\temp\extract -Recurse -File | ForEach-Object FullName
```

Run it via PowerShell `Start-Process -Wait`; started from Bash in the background `msiexec /a` hung.
