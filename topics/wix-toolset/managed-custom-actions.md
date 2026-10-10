---
title: Managed custom actions (DTF)
description: Read when writing a C# custom action for a WiX v4+ MSI - WixToolset.Dtf.CustomAction project, .CA.dll, OSMF EULA, shim bitness, deferred action as SYSTEM, logging.
---

# Managed custom actions (DTF)

Verified with WiX 7.0.0, `WixToolset.Dtf.CustomAction` 7.0.0, net48, x64 per-machine package, Windows 11 (2026-10): one deferred action, run as SYSTEM on install and uninstall.

## Project

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
    <AcceptEula>wix7</AcceptEula>   <!-- only after the user accepted: osmf-eula.md -->
  </PropertyGroup>
  <ItemGroup>
    <Content Include="CustomAction.config" CopyToOutputDirectory="PreserveNewest" />
    <PackageReference Include="WixToolset.Dtf.CustomAction" Version="7.0.0" />
  </ItemGroup>
</Project>
```

- The build writes `<AssemblyName>.CA.dll` next to the assembly: native shim (SfxCA) + the assembly + its CopyLocal references + all `Content` items. The MSI embeds the `.CA.dll`, never the plain DLL.
- The package enforces the OSMF EULA like the SDK: `error WIX7015` in the `.csproj` build until `AcceptEula` is set there too ([osmf-eula](osmf-eula.md)).
- Shim bitness follows `$(Platform)`: `x64` and `ARM64` get their own shim, everything else (AnyCPU) the x86 one. The x86 shim works in an x64 package, but the action then runs as a 32-bit process: open the registry with an explicit `RegistryView`.
- `CustomAction.config` selects the runtime. With the file below the log shows `SFXCA: Binding to CLR version v4.0.30319`. _Unverified:_ behavior without the file.

```xml
<configuration>
  <startup useLegacyV2RuntimeActivationPolicy="true">
    <supportedRuntime version="v4.0" />
  </startup>
</configuration>
```

- Plain SDK-style project: builds with `dotnet build`. Keep the logic in a class that does not use `Session` and unit-test it there.
- _Unverified:_ target frameworks other than .NET Framework.

## Code

```csharp
using WixToolset.Dtf.WindowsInstaller;

public static class CustomActions
{
    [CustomAction]
    public static ActionResult DoWork(Session session)
    {
        try
        {
            Worker.Run(session.Log);          // Action<string>
            return ActionResult.Success;
        }
        catch (Exception ex)
        {
            session.Log($"DoWork failed: {ex}");
            using var record = new Record(0);
            record[0] = $"Could not ... Reason: {ex.Message}";
            session.Message(InstallMessage.Error, record);
            return ActionResult.Failure;
        }
    }
}
```

- `session.Log` works in deferred actions; the lines appear only in a verbose log (`msiexec /l*v`).
- _Unverified:_ the `session.Message` error dialog; it was never triggered in the test.

## Authoring

```xml
<Binary Id="CustomActionsDll" SourceFile="!(bindpath.My.Setup.CustomActions)\My.Setup.CustomActions.CA.dll" />

<CustomAction Id="DoWorkOnInstall"   BinaryRef="CustomActionsDll" DllEntry="DoWork"
              Execute="deferred" Impersonate="no" Return="check" />
<CustomAction Id="DoWorkOnUninstall" BinaryRef="CustomActionsDll" DllEntry="DoWork"
              Execute="deferred" Impersonate="no" Return="ignore" />

<InstallExecuteSequence>
  <Custom Action="DoWorkOnInstall"   Before="InstallFinalize" Condition="NOT (REMOVE~=&quot;ALL&quot;)" />
  <Custom Action="DoWorkOnUninstall" Before="InstallFinalize" Condition="REMOVE~=&quot;ALL&quot;" />
</InstallExecuteSequence>
```

- `ProjectReference` to the custom action project gives the bind path; dots in the project name work in `!(bindpath.<name>)`.
- `Execute="deferred" Impersonate="no"` runs as SYSTEM and may write HKLM. Scheduled last before `InstallFinalize`, little can fail after it, so no rollback action was needed.
- Two entries for one `DllEntry`: a failure aborts an install but never blocks an uninstall. If the custom action sits in a `Fragment`, pull it in with `<CustomActionRef>`.
- Built MSI, `CustomAction.Type`: 3073 = DLL from Binary (1) + deferred (1024) + no impersonation (2048); 3137 = the same + ignore return code (64).

## Verify in the log

```
MSI_LUA : Custom Action 'DoWorkOnInstall' is running with sufficient privileges.
SFXCA: Extracting custom action to temporary directory: C:\WINDOWS\Installer\SFXCA...\
SFXCA: Binding to CLR version v4.0.30319
Calling custom action My.Setup.CustomActions!My.Setup.CustomActions.CustomActions.DoWork
```

Example use: [visio-content-deployment/publish-component](../visio-content-deployment/publish-component.md#configchangeid).
