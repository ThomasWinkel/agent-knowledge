---
title: Dev build registration overrides the installed add-in
description: Read when testing an installed VSTO add-in on a development machine, or when Office loads the add-in from bin\Debug instead of the install folder.
---

# Dev build registration overrides the installed add-in

Every build of a VSTO project (plain `msbuild /t:Build` suffices, no F5) runs target `RegisterOfficeAddin` and writes the HKCU registration, e.g.:

```
HKCU\Software\Microsoft\Visio\Addins\<KeyName>
  LoadBehavior = 3
  Manifest     = file:///C:/.../bin/Debug/<AssemblyName>.vsto|vstolocal
```

Same key name as the MSI's registration, and HKCU wins over HKLM. Office then loads the dev build, not the installed one.

## Symptoms

- Trust error instead of loading: "The certificate used to sign the deployment manifest is unknown, and the customization itself (...) is not on the inclusion list". Outside `Program Files` the location-based FullTrust does not apply, so the test certificate is checked. Looks like an expired certificate ([signing](signing.md)), but the load location is wrong.
- Event log (provider "VSTO 4.0", event 4096): `Customization URI` points to the build output.
- `COMAddIns` `Description` shows the dev entry's values, not the MSI's.

## Before every installer test on a dev machine

```powershell
reg delete "HKCU\SOFTWARE\Microsoft\Visio\Addins\<KeyName>" /f
reg delete "HKCU\SOFTWARE\Microsoft\Visio\AddinsData\<KeyName>" /f
```

Other apps: `HKCU\SOFTWARE\Microsoft\Office\<App>\Addins\<KeyName>`. Rebuilding the add-in recreates the key. End-user machines are not affected.

_Unverified:_ HKLM and HKCU entries with different key names both load (both appeared in `COMAddIns`); only the same-name case is tested.
