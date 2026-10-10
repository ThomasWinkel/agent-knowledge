---
title: Registry registration via MSI
description: Read when registering a VSTO add-in from an MSI/WiX installer - registry path (Visio differs), required values, file:/// and |vstolocal, key name, HKMU, 32- and 64-bit Office.
---

# Registry registration via MSI

Office finds a VSTO add-in only through these registry values. Reference: [Registry entries for VSTO Add-ins](https://learn.microsoft.com/en-us/visualstudio/vsto/registry-entries-for-vsto-add-ins).

| Application | Key (`Root` = HKCU or HKLM) |
|---|---|
| Visio | `Root\Software\Microsoft\Visio\Addins\<KeyName>` (no `Office`) |
| All others | `Root\Software\Microsoft\Office\<App>\Addins\<KeyName>` (`Word`, `Excel`, `Outlook`, ...) |

Visio does not read `Software\Microsoft\Office\Visio\Addins` at all: the add-in is then missing from `COMAddIns` (verified with Visio 2019 C2R x64).

| Value | Type | Content |
|---|---|---|
| `Description` | REG_SZ | shown in the Add-ins options |
| `FriendlyName` | REG_SZ | shown in the COM Add-ins dialog |
| `LoadBehavior` | REG_DWORD | `3` = load at startup |
| `Manifest` | REG_SZ | `file:///<path>\<Name>.vsto|vstolocal` |

## `Manifest` value

- MSI deployments need the `file:///` prefix (documented for all apps). Without it Visio silently resets `LoadBehavior` 3 → 2 and does not load; forcing `COMAddIn.Connect` gives `HRESULT 0x8004062A`, no event log entry.
- Backslashes after `file:///` work (A/B-tested in Visio), so `file:///[INSTALLFOLDER]Name.vsto|vstolocal` is enough. No custom action is needed to convert slashes.
- `|vstolocal` loads from the install folder instead of the ClickOnce cache.

## KeyName: read it from the manifest

Not `<RootNamespace>.ThisAddIn`. Take `keyName` from `<AssemblyName>.dll.manifest` in the add-in's build output (by default the assembly name):

```xml
<vstov4:appAddIn application="Visio" loadBehavior="3" keyName="CompanyProduct">
```

A wrong KeyName installs without error, and the add-in never appears.

## WiX: per-machine, 32- and 64-bit Office

Microsoft recommends both registry views for all-users installs, because Office may be 32- or 64-bit independently of the package. Files need no second copy: an AnyCPU add-in loads in both, and `Program Files` is not redirected.

```xml
<ComponentGroup Id="VstoRegistration" Directory="INSTALLFOLDER">
  <Component Bitness="always64">
    <RegistryKey Root="HKMU" Key="Software\Microsoft\Visio\Addins\CompanyProduct">
      <RegistryValue Name="Description"  Value="Product"   Type="string" />
      <RegistryValue Name="FriendlyName" Value="Product"   Type="string" />
      <RegistryValue Name="LoadBehavior" Value="3"         Type="integer" KeyPath="yes" />
      <RegistryValue Name="Manifest"     Value="file:///[INSTALLFOLDER]CompanyProduct.vsto|vstolocal" Type="string" />
    </RegistryKey>
  </Component>
  <Component Bitness="always32" Directory="TARGETDIR">
    <!-- same RegistryKey -->
  </Component>
</ComponentGroup>
```

- `Root="HKMU"` resolves to HKLM per-machine and HKCU per-user (MSI `Registry.Root = -1`).
- `Directory="TARGETDIR"` on the 32-bit component avoids `error WIX0204: ICE80: This 32BitComponent ... uses 64BitDirectory INSTALLFOLDER`. A registry-only component's directory is irrelevant.
- Built MSI: `Component.Attributes` 260 (256 = 64-bit + 4 = registry key path) and 4.
- Per-user package: drop the 32-bit component. HKCU is not redirected; two components with the same key path break MSI component rules.
- AnyCPU check: PE `Machine = 0x14C` and COR20 flag `32BITREQUIRED` not set. `0x8664` = 64-bit only; then skip the 32-bit registration.
- HKCU `LoadBehavior` (e.g. user disabled the add-in) overrides the HKLM value per user.
- _Unverified:_ the 32-bit branch was only checked in the MSI tables, never loaded in a 32-bit Office.
- The `Office\<App>` path is verified for Excel 2019 C2R x64 (per-machine MSI, `Connect=True`). _Unverified:_ other apps.

## Related

- Package bitness basics: [wix-toolset/package-bitness](../wix-toolset/package-bitness.md)
- Visual Studio's own HKCU registration overrides this one on dev machines: [dev-build-registration](dev-build-registration.md)
- Launch conditions for VSTO runtime / Office: [prerequisite-checks](prerequisite-checks.md)
