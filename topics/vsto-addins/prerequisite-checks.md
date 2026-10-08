---
title: Installer prerequisite checks
description: Read when adding launch conditions to a VSTO add-in installer for the VSTO runtime or the Office application.
---

# Installer prerequisite checks

## VSTO runtime lives in the 32-bit registry view

Even with the x64 runtime ("Microsoft Visual Studio 2010 Tools for Office Runtime (x64)"):

```
HKLM\SOFTWARE\Microsoft\VSTO Runtime Setup\                  does not exist
HKLM\SOFTWARE\WOW6432Node\Microsoft\VSTO Runtime Setup\v4R   Version=10.0.60917
```

A `RegistrySearch` in an x64 package looks at the native view by default. It finds nothing, and a launch condition built on it blocks every machine. Set `Bitness="always32"`:

```xml
<Property Id="VSTO_RUNTIME_FOUND">
  <RegistrySearch Root="HKLM" Key="SOFTWARE\Microsoft\VSTO Runtime Setup\v4R" Name="Version" Type="raw" Bitness="always32" />
</Property>
```

## Prefer no hard launch condition

Registration is plain registry state. If Office is installed after the add-in, the add-in loads on first start. In companies, "add-in on the image, Office later" is common. A hard condition on Office or runtime presence breaks that. If a condition is wanted, block only the real error case:

```
NOT OFFICE_APP_INSTALLED OR VSTO_RUNTIME_FOUND
```

Office bitness on a machine (Click-to-Run): `Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Office\ClickToRun\Configuration" | Select-Object Platform`. _Unverified:_ MSI-based Office uses other keys.
