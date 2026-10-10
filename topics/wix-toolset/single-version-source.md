---
title: One version number for assemblies, VSTO manifests and MSI
description: Read when a solution with SDK-style projects, classic VSTO add-in projects and a WiX v4+ setup should take its version from one MSBuild property.
---

# One version number for assemblies, VSTO manifests and MSI

`<Version>` in `Directory.Build.props` at the solution root is the single source. Every project type imports that file, but only SDK-style C# projects use `Version` by themselves.

| Consumer | Default without this recipe | What to add |
|---|---|---|
| SDK-style C# project | assembly and file version from `Version` | nothing |
| Classic VSTO project: assembly | `AssemblyVersion`/`AssemblyFileVersion` in `Properties\AssemblyInfo.cs` | generate both attributes (target below); delete them from `AssemblyInfo.cs` |
| Classic VSTO project: `.vsto`, `.dll.manifest` | `1.0.0.0`: `Microsoft.VisualStudio.Tools.Office.targets` sets `PublishVersion` from `ApplicationVersion` | set `ApplicationVersion` (four parts) |
| WiX package | literal in `<Package Version>` | pass the property as a preprocessor variable |

```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <Version>1.2.3</Version>
  </PropertyGroup>
</Project>
```

```xml
<!-- Directory.Build.targets -->
<Project>
  <PropertyGroup Condition="'$(UsingMicrosoftNETSdk)' != 'true' and '$(MSBuildProjectExtension)' == '.csproj'">
    <ApplicationVersion>$(Version).0</ApplicationVersion>
    <VersionAttributesFile>$(IntermediateOutputPath)VersionAttributes.g.cs</VersionAttributesFile>
  </PropertyGroup>

  <Target Name="WriteVersionAttributes" BeforeTargets="CoreCompile" Condition="'$(VersionAttributesFile)' != ''"
          Inputs="$(MSBuildThisFileDirectory)Directory.Build.props;$(MSBuildThisFileFullPath)" Outputs="$(VersionAttributesFile)">
    <ItemGroup>
      <VersionAttribute Include="System.Reflection.AssemblyVersion;System.Reflection.AssemblyFileVersion">
        <_Parameter1>$(ApplicationVersion)</_Parameter1>
      </VersionAttribute>
    </ItemGroup>
    <WriteCodeFragment Language="C#" OutputFile="$(VersionAttributesFile)" AssemblyAttributes="@(VersionAttribute)" />
    <ItemGroup>
      <Compile Include="$(VersionAttributesFile)" />
      <FileWrites Include="$(VersionAttributesFile)" />
    </ItemGroup>
  </Target>
</Project>
```

```xml
<!-- .wixproj -->
<PropertyGroup>
  <DefineConstants>$(DefineConstants);ProductVersion=$(Version)</DefineConstants>
</PropertyGroup>

<!-- Package.wxs -->
<Package Name="My Product" Version="$(var.ProductVersion)" ...>
```

## Gotchas

- `_Parameter1` must be a child element. As an attribute (`<VersionAttribute Include="..." _Parameter1="..." />`) the build fails with MSB4066 "attribute `_Parameter1` ... is unrecognized".
- The condition must exclude the `.wixproj`: it also has a `CoreCompile` target and imports `Directory.Build.targets`.
- `Inputs`/`Outputs` skip the target when the version files did not change, so the generated file does not trigger a recompile. Price: `-p:Version=...` on the command line does not reach the classic projects' attributes; change the file.
- Leaving the two attributes in `AssemblyInfo.cs` gives CS0579 (duplicate attribute).
- MSI takes no prerelease suffix (`1.2.3-alpha.5`); keep `Version` to three numeric parts if the same property feeds the package.

Verified with VS 2026 (MSBuild 18), WiX 7.0.0, two classic VSTO projects (Visio, Excel) (2026-10): after changing only `<Version>`, the new number was in all assemblies, in the identities of `.vsto` and `.dll.manifest`, and in the MSI `ProductVersion`; both add-ins still loaded from the build output. _Unverified:_ whether the VSTO runtime needs a changed manifest version when an MSI replaces an installed add-in.
