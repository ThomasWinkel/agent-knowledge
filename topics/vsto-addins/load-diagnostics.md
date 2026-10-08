---
title: Load diagnostics
description: Read when a VSTO add-in is installed but does not appear or does not load - COMAddIns check via COM, LoadBehavior, VSTO event log, VSTO_LOGALERTS, diagnosis order.
---

# Load diagnostics

## Diagnosis order

1. Is it in `Application.COMAddIns` (script below)?
   - **Not listed:** wrong registry key, wrong KeyName, or WOW6432Node mismatch ([registration](registration.md)).
   - **Listed, `Connect=False`:** found but loading failed; continue.
2. `LoadBehavior` after starting the app: 3 → 2 means a load attempt failed.
3. Event log, provider "VSTO 4.0" (event 4096): message and `Customization URI`. If the URI points to the build output, it is a dev registration ([dev-build-registration](dev-build-registration.md)).
4. Reset to 2 but no VSTO event: the runtime was never reached. Check the `Manifest` value format (`file:///`, [registration](registration.md)).
5. Event present: missing files ([manifest-dependencies](manifest-dependencies.md)), hash/size mismatch or certificate ([signing](signing.md)).

```powershell
Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=(Get-Date).AddMinutes(-10)} |
  Where-Object ProviderName -match 'VSTO' | Select-Object TimeCreated, Message
```

## Check load state via COM

More reliable than the Options dialog:

```powershell
$app = New-Object -ComObject Visio.Application   # Word.Application, Excel.Application, ...
foreach ($a in $app.COMAddIns) { if ($a.ProgId -eq "<KeyName>") { "$($a.ProgId): Connect=$($a.Connect)" } }
$app.Quit()
```

- `New-Object -ComObject` attaches to a running instance if one is registered. Check the process list and start time first, so you test a fresh instance and do not disturb the user's open documents.
- After a failed load, reset `LoadBehavior` to 3 before the next attempt.

## Make errors visible

- Environment variables (documented): `VSTO_SUPPRESSDISPLAYALERTS=0` shows error dialogs. `VSTO_LOGALERTS=1` writes `<Name>.vsto.log` next to the `.vsto`; it includes "Loaded Assemblies".
- `HKCU\Software\Microsoft\VSTO\Security`, DWORD `Debug` = 1: showed the "Publisher cannot be verified" dialog in a real case. _Unverified:_ not found in Microsoft docs. Remove it afterwards.
- Remove the log files before building an installer from that folder; wildcard harvesting picks them up.
