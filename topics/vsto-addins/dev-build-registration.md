---
title: Dev build registration and the installed add-in - which one loads
description: Read when a dev machine has both a VSTO dev build and the MSI installed - Office loads the wrong one, how to tell which one runs, what to remove before which test.
---

# Dev build registration and the installed add-in

Every build of a VSTO project (plain `msbuild /t:Build` suffices, no F5) runs target `RegisterOfficeAddin` and writes the HKCU registration, e.g.:

```
HKCU\Software\Microsoft\Visio\Addins\<KeyName>
  LoadBehavior = 3
  Manifest     = file:///C:/.../bin/Debug/<AssemblyName>.vsto|vstolocal
```

Same key name as the MSI's registration (HKLM). With both present only one loads (`COMAddIns` lists one entry), and which one is not fixed:

- Visio 2019 C2R x64 build 20527 (2026-11): the **installed** add-in loaded (HKLM won). The dev build in `bin\Debug` never ran after the MSI install, although HKCU pointed to it and every build rewrote it; uninstalling the MSI made Visio load the dev build at the next start.
- Earlier case (app and version not recorded): the **dev** build loaded (HKCU won), see the symptoms below.

So: one of the two on a dev machine. Uninstall the MSI to test a dev build; delete the HKCU keys (below) to test the MSI.

## Which one runs

- `COMAddIns` `Description`: taken from the winning registry key; the MSI usually writes the product name, the build writes the assembly name.
- VSTO shadow-copies the add-in DLL to `%LOCALAPPDATA%ssembly\dl3\...\<AssemblyName>.DLL`; the `__AssemblyInfo__.ini` next to it (UTF-16) holds version and origin URL (`file:///C:/Program Files/...` or `.../bin/Debug/...`), the DLL's time = last load. The DLL in the install or build folder is not locked by a running Office.
- A Debug-only behavior (e.g. a debug log) that never shows is the quickest hint.

## Symptoms when the dev build wins

- Trust error instead of loading: "The certificate used to sign the deployment manifest is unknown, and the customization itself (...) is not on the inclusion list". Outside `Program Files` the location-based FullTrust does not apply, so the test certificate is checked. Looks like an expired certificate ([signing](signing.md)), but the load location is wrong.
- Event log (provider "VSTO 4.0", event 4096): `Customization URI` points to the build output.
- `COMAddIns` `Description` shows the dev entry's values, not the MSI's.

## Before an installer test on a dev machine

```powershell
reg delete "HKCU\SOFTWARE\Microsoft\Visio\Addins\<KeyName>" /f
reg delete "HKCU\SOFTWARE\Microsoft\Visio\AddinsData\<KeyName>" /f
```

Other apps: `HKCU\SOFTWARE\Microsoft\Office\<App>\Addins\<KeyName>`. Rebuilding the add-in recreates the key. End-user machines are not affected.

_Unverified:_ HKLM and HKCU entries with different key names both load (both appeared in `COMAddIns`); only the same-name case is tested.
