---
title: Folder of the add-in DLL at runtime
description: Read when a VSTO add-in must find files next to its DLL or in its install folder (help, data, config) - Location is the shadow copy, CodeBase the real folder.
---

# Folder of the add-in DLL at runtime

VSTO shadow-copies the add-in (`AppDomain.ShadowCopyFiles` = true, see [satellite-assemblies](satellite-assemblies.md)). `Assembly.Location` is the copy under `%LOCALAPPDATA%\assembly\dl3\...`, where the files next to the DLL are missing. `Assembly.CodeBase` is the folder the DLL was copied from:

```csharp
var folder = Path.GetDirectoryName(new Uri(typeof(MyRibbon).Assembly.CodeBase).LocalPath);
```

- Dev build (HKCU registration from the build): the build folder (`bin\Debug`). Verified.
- MSI install (HKLM registration, `Manifest` = `file:///C:\Program Files\MyProduct\MyAddIn\MyAddIn.vsto|vstolocal`): the install folder `C:\Program Files\MyProduct\MyAddIn`. Verified.
- `CodeBase` is obsolete only on .NET 5+; VSTO runs on .NET Framework.

## Installed and dev build without #if DEBUG

To ship a file beside the add-in folder and still find it in a dev build, look for the installed place first, then walk up to the solution file:

```csharp
var installed = Path.Combine(folder, "..", "help.ini");      // <Install>\help.ini, add-in in <Install>\MyAddIn\
if (File.Exists(installed)) return installed;
for (var dir = new DirectoryInfo(folder); dir != null; dir = dir.Parent)
    if (File.Exists(Path.Combine(dir.FullName, "MyProduct.slnx")))
        return Path.Combine(dir.FullName, "content", "help.ini");   // the repo's source of the shipped file
return null;
```

Any build configuration in the repo finds the repo's file; a test copy at `bin\help.ini` (above `bin\Debug`) simulates the installed place.

Verified with Visio Pro 2019 C2R x64, dev build and MSI install (2026-10).
