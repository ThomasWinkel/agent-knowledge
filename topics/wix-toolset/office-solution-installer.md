---
title: Installer for a Visio/Excel solution
description: Read first when building an MSI for an Office solution with WiX v7 - VSTO add-ins (Visio, Excel), Visio stencils and templates - steps in order, complete example, test procedure.
---

# Installer for a Visio/Excel solution

One MSI that installs VSTO add-ins for Visio and Excel, Visio stencils and templates, and other files. This file gives the order, one complete example and the test procedure. Reasons, gotchas and variants live in the linked files: read each one before you write its part.

Verified with WiX 7.0.0, Visio and Excel 2019 C2R x64, Windows 11 (2026-10): install, content in the Visio UI, add-ins loaded, uninstall. _Unverified:_ 32-bit Office, upgrade from an older version, the license dialog (built, never shown: the test used `/qb`).

## Choices that keep it simple

- **Per-machine, fixed folder under Program Files.** VSTO trusts Program Files by location ([vsto-addins/signing](../vsto-addins/signing.md)); Visio publishing is machine-wide ([visio-content-deployment](../visio-content-deployment/index.md)). Do not offer a folder choice: use `WixUI_Minimal`.
- **x64 package, registry values in both views**, so 32- and 64-bit Office find the add-ins ([package-bitness](package-bitness.md), [vsto-addins/registration](../vsto-addins/registration.md)).
- **No launch conditions** for Office or the VSTO runtime ([vsto-addins/prerequisite-checks](../vsto-addins/prerequisite-checks.md)).
- **One folder per add-in**, each with the add-in's whole build output. Two add-ins share file names (the shared library, `Microsoft.Office.Tools.Common.v4.0.Utilities.dll`); in one folder the harvests collide.

## Steps

1. **EULA.** WiX v7 does not build until the user accepted the OSMF EULA. Ask; never accept it yourself: [osmf-eula](osmf-eula.md).
2. **Custom action project** (C#, DTF) that counts up Visio's `ConfigChangeID`. Without it Visio does not list the stencils and templates: project in [managed-custom-actions](managed-custom-actions.md), code in [visio-content-deployment/publish-component](../visio-content-deployment/publish-component.md#configchangeid).
3. **Setup project** (`.wixproj`) with a `ProjectReference` to each add-in and to the custom action project: [versions-and-project-format](versions-and-project-format.md), [harvesting-project-output](harvesting-project-output.md).
4. **Add-in files:** harvest the whole output, minus `*.pdb`, `*.xml`, `*.log`: [vsto-addins/manifest-dependencies](../vsto-addins/manifest-dependencies.md).
5. **Add-in registry keys.** Read the key name from `keyName` in the built `<AssemblyName>.dll.manifest`. Visio's key path has no `Office`: [vsto-addins/registration](../vsto-addins/registration.md).
6. **Stencils and templates:** one explicit component per file with `<Category>` rows: [visio-content-deployment/publish-component](../visio-content-deployment/publish-component.md).
7. **Build** with Visual Studio's MSBuild. `dotnet build` fails on VSTO projects (`MSB4019 ... Microsoft.VisualStudio.Tools.Office.targets`):
   ```sh
   MSB=$("/c/Program Files (x86)/Microsoft Visual Studio/Installer/vswhere.exe" -latest -requires Microsoft.Component.MSBuild -find 'MSBuild\**\Bin\MSBuild.exe' | head -1)
   "$MSB" My.Setup/My.Setup.wixproj -restore -v:m -nologo
   ```
8. **Check the MSI tables, then test** (below).
9. One version number for assemblies, VSTO manifests and package: [single-version-source](single-version-source.md).
10. Release builds: sign add-in assemblies, custom action DLL and MSI: [vsto-addins/signing](../vsto-addins/signing.md).

## Example

Condensed from the verified project; names replaced. Projects `My.VisioAddIn`, `My.ExcelAddIn` (same pattern, omitted below), `My.Setup.CustomActions`; ship files in `content\`.

```xml
<!-- My.Setup.wixproj -->
<Project Sdk="WixToolset.Sdk/7.0.0">
  <PropertyGroup>
    <AcceptEula>wix7</AcceptEula>                    <!-- only after the user accepted -->
    <InstallerPlatform>x64</InstallerPlatform>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="WixToolset.UI.wixext" Version="7.0.0" />
    <ProjectReference Include="..\My.VisioAddIn\My.VisioAddIn.csproj" />
    <ProjectReference Include="..\My.Setup.CustomActions\My.Setup.CustomActions.csproj" />
    <BindPath Include="..\..\content" BindName="Content" />
  </ItemGroup>
</Project>
```

```xml
<!-- Package.wxs -->
<Wix xmlns="http://wixtoolset.org/schemas/v4/wxs" xmlns:ui="http://wixtoolset.org/schemas/v4/wxs/ui">
  <Package Name="My Product" Manufacturer="My Company" Version="1.0.0"
           UpgradeCode="PUT-A-NEW-GUID-HERE" Scope="perMachine">
    <MajorUpgrade DowngradeErrorMessage="A newer version of [ProductName] is already installed." />
    <MediaTemplate EmbedCab="yes" />
    <ui:WixUI Id="WixUI_Minimal" />
    <WixVariable Id="WixUILicenseRtf" Value="License.rtf" />

    <Feature Id="Main">
      <ComponentGroupRef Id="Content" />
      <ComponentGroupRef Id="VisioAddIn" />
    </Feature>

    <!-- Count up Visio's ConfigChangeID as SYSTEM, last step of install and uninstall. -->
    <Binary Id="CustomActionsDll" SourceFile="!(bindpath.My.Setup.CustomActions)\My.Setup.CustomActions.CA.dll" />
    <CustomAction Id="BumpVisioConfigChangeIdOnInstall" BinaryRef="CustomActionsDll" DllEntry="BumpVisioConfigChangeId"
                  Execute="deferred" Impersonate="no" Return="check" />
    <CustomAction Id="BumpVisioConfigChangeIdOnUninstall" BinaryRef="CustomActionsDll" DllEntry="BumpVisioConfigChangeId"
                  Execute="deferred" Impersonate="no" Return="ignore" />
    <InstallExecuteSequence>
      <Custom Action="BumpVisioConfigChangeIdOnInstall" Before="InstallFinalize" Condition="NOT (REMOVE~=&quot;ALL&quot;)" />
      <Custom Action="BumpVisioConfigChangeIdOnUninstall" Before="InstallFinalize" Condition="REMOVE~=&quot;ALL&quot;" />
    </InstallExecuteSequence>
  </Package>

  <Fragment>
    <StandardDirectory Id="ProgramFiles6432Folder">
      <Directory Id="INSTALLFOLDER" Name="My Product">
        <Directory Id="StencilsFolder" Name="Stencils" />
        <Directory Id="TemplatesFolder" Name="Templates" />
        <Directory Id="VisioAddInFolder" Name="VisioAddIn" />
      </Directory>
    </StandardDirectory>
  </Fragment>

  <Fragment>
    <ComponentGroup Id="Content">
      <Component Directory="StencilsFolder">
        <File Source="!(bindpath.Content)\Stencils\Basic.vssx" />
        <Category Id="{6D9D8B6F-D0EF-4BC0-8DD4-09DD6CE2B301}" Qualifier="1\Basic.vssx" AppData="My Company\Basic||0|-1" />
      </Component>
      <Component Directory="TemplatesFolder">
        <File Source="!(bindpath.Content)\Templates\Wiring.vstx" />
        <Category Id="{6D9D8B6F-D0EF-4BC0-8DD4-09DD6CE2B300}" Qualifier="1\Wiring.vstx" AppData="My Company\Wiring||1|-1" />
      </Component>
      <!-- Everything else in content\, same layout. Published files must be excluded (WIX0369). -->
      <Files Directory="INSTALLFOLDER" Include="!(bindpath.Content)\**">
        <Exclude Files="!(bindpath.Content)\Stencils\Basic.vssx" />
        <Exclude Files="!(bindpath.Content)\Templates\Wiring.vstx" />
      </Files>
    </ComponentGroup>
  </Fragment>

  <Fragment>
    <ComponentGroup Id="VisioAddIn" Directory="VisioAddInFolder">
      <Files Include="!(bindpath.My.VisioAddIn)\**">
        <Exclude Files="!(bindpath.My.VisioAddIn)\**\*.pdb" />
        <Exclude Files="!(bindpath.My.VisioAddIn)\**\*.xml" />
        <Exclude Files="!(bindpath.My.VisioAddIn)\**\*.log" />
      </Files>
      <Component Id="VisioAddInRegistry64" Bitness="always64">
        <RegistryKey Root="HKLM" Key="Software\Microsoft\Visio\Addins\My.VisioAddIn">
          <RegistryValue Name="Description" Value="My Product add-in for Visio" Type="string" />
          <RegistryValue Name="FriendlyName" Value="My Product" Type="string" />
          <RegistryValue Name="LoadBehavior" Value="3" Type="integer" KeyPath="yes" />
          <RegistryValue Name="Manifest" Value="file:///[VisioAddInFolder]My.VisioAddIn.vsto|vstolocal" Type="string" />
        </RegistryKey>
      </Component>
      <Component Id="VisioAddInRegistry32" Bitness="always32" Directory="TARGETDIR">
        <!-- the same RegistryKey again -->
      </Component>
    </ComponentGroup>
  </Fragment>
</Wix>
```

- Excel add-in: same two fragments with key `Software\Microsoft\Office\Excel\Addins\My.ExcelAddIn`.
- A drawing created from the published template opened its stencil (stencil published too, installed next to it in `Stencils\`).
- A stencil or template added to `content\` later is installed by `<Files>` but not published: it needs its own component and `<Category>`.

## Check the built MSI

Read the tables before installing (how: [visio-content-deployment/publish-component](../visio-content-deployment/publish-component.md), "Check the built rows"). In MSI SQL quote the `Registry` column `` `Key` `` with backticks.

| Check | Expected |
|---|---|
| Summary property 7 | `x64;1033` |
| `Property` `ALLUSERS` | `1` |
| `PublishComponent` | one row per `<Category>` |
| `Registry` | each add-in key twice (64- and 32-bit component) |
| `CustomAction.Type` | 3073 (install), 3137 (uninstall) |
| `InstallExecuteSequence` | both actions directly before `InstallFinalize` |

## Test on a dev machine

Installing changes Program Files and HKLM and needs elevation: ask the user first.

1. Delete the HKCU dev registrations of the add-ins, or Office loads the build output instead: [vsto-addins/dev-build-registration](../vsto-addins/dev-build-registration.md). Close Visio and Excel. Note the value `HKLM\Software\Microsoft\Office\Visio\ConfigChangeID`.
2. Install with a log. From a non-elevated shell this shows one UAC prompt:
   ```powershell
   $p = Start-Process msiexec.exe -ArgumentList "/i `"$msi`" /qb /l*v `"$log`"" -Verb RunAs -PassThru -Wait
   $p.ExitCode   # 0 = ok
   ```
3. Check: the custom action lines in the log ([managed-custom-actions](managed-custom-actions.md)); `ConfigChangeID` is one higher; files in the install folder; add-in keys under `HKLM\Software\...` and `HKLM\Software\WOW6432Node\...`.
4. Add-ins: `Connect=True` in `COMAddIns`, script in [vsto-addins/load-diagnostics](../vsto-addins/load-diagnostics.md).
5. Visio content: a person must look in the Visio UI (a COM start does not fill that part of the cache): File → New → Categories, and More Shapes. Create a drawing from the template.
6. Uninstall the same way with `/x`. Check: install folder and all add-in keys gone, `ConfigChangeID` one higher again.
7. Rebuild the add-ins to get the dev registrations back.
