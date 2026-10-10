---
title: Package bitness and registry redirection
description: Read when choosing x86 vs x64 for a WiX package, when per-machine registry values land in WOW6432Node, or when Platform=x64 breaks a ProjectReference build.
---

# Package bitness and registry redirection

## x86 is the default

A WiX v4+ package builds as x86 unless set otherwise (HeatWave template: `Debug|x86`). On 64-bit Windows an x86 per-machine package writes `HKLM\Software\...` (also via `Root="HKMU"`) to `HKLM\Software\WOW6432Node\...` and installs to `Program Files (x86)`, regardless of the contained assemblies being AnyCPU. 64-bit processes never read those keys.

```xml
<Project Sdk="WixToolset.Sdk/7.0.0">
  <PropertyGroup>
    <Platform>x64</Platform>
  </PropertyGroup>
</Project>
```

With x64, `ProgramFiles6432Folder` and `HKLM`/`HKMU` resolve to the native 64-bit locations; no other change needed.

- Alternative: `<InstallerPlatform>x64</InstallerPlatform>` instead of `<Platform>`. The package is x64 (`Template` = `x64;1033`), but `Platform` stays AnyCPU: ProjectReferences need no `SetPlatform` (see below) and the output stays in `bin\<Configuration>\`.
- An x64 package does not cover 32-bit consumers (e.g. 32-bit Office). For keys both bitnesses must read, write two registry components with `Bitness="always64"` and `Bitness="always32"`: recipe and ICE80 fix in [vsto-addins/registration](../vsto-addins/registration.md).
- Searching a 32-bit key from an x64 package needs `Bitness="always32"` on the `RegistrySearch`.
- `HKCU\Software` is not redirected; per-user packages need no second component.

## `Platform` propagates to ProjectReferences

```
error : The BaseOutputPath/OutputPath property is not set for project 'MyAddin.csproj'. ... Configuration='Debug' Platform='x64'.
```

The referenced project has no x64 configuration (e.g. AnyCPU-only VSTO project). Pin it:

```xml
<ProjectReference Include="..\MyAddin\MyAddin.csproj">
  <SetPlatform>Platform=AnyCPU</SetPlatform>
</ProjectReference>
```

## Verify

- MSI summary information property 7 (Template): `x64;1033` = x64, `Intel;1033` = x86. Read with Orca or the `WindowsInstaller.Installer` COM object.
- Administrative install ([harvesting-project-output](harvesting-project-output.md#check-the-built-msis-contents)): x64 extracts to `PFiles64\`, x86 to `PFiles\`.
