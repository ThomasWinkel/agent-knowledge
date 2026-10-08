---
title: Manifest dependencies must be installed
description: Read when an installed VSTO add-in silently does not load although registration is correct, or when deciding which DLLs the installer must ship.
---

# Manifest dependencies must be installed

With `|vstolocal` the VSTO runtime checks every `dependentAssembly dependencyType="install"` in `<AssemblyName>.dll.manifest` against the install folder before loading. One missing file → the add-in silently does not load.

## Ship the Microsoft.Office.Tools DLLs too

Hand-maintained file lists typically miss the VSTO assemblies that Visual Studio adds. A real case missed 5 of 10:

```
Microsoft.Office.Tools.dll
Microsoft.Office.Tools.Common.dll
Microsoft.Office.Tools.v4.0.Framework.dll
Microsoft.VisualStudio.Tools.Applications.Runtime.dll
Microsoft.Web.WebView2.Wpf.dll
```

- At runtime the `Microsoft.Office.Tools*` assemblies load from the GAC, which wins for strong-named assemblies (visible in the `VSTO_LOGALERTS` log, "Loaded Assemblies"). The manifest check still requires them in the install folder.
- Exception: `Microsoft.Office.Tools.Common.v4.0.Utilities.dll` is not in the GAC and really loads locally. Visual Studio marks only this reference `<Private>True</Private>`.
- `<Private>False</Private>` on a reference removes it from output and manifest. It saves about 230 KB, and VS designers may reset it. Not worth it.

## Fix: harvest the whole output

Use the recursive `<Files Include="!(bindpath.X)\**">` with excludes from [wix-toolset/harvesting-project-output](../wix-toolset/harvesting-project-output.md). It also catches files missing from the manifest, e.g. WebView2's native loader `runtimes\win-<arch>\native\WebView2Loader.dll`. Without it the add-in loads, but the WebView2 control fails at runtime.

## Check manifest against a folder

Works on the install folder or the build output:

```powershell
$dir = "C:\Program Files\<Vendor> <Product>"
Select-Xml -Path "$dir\<AssemblyName>.dll.manifest" `
  -XPath "//*[local-name()='dependentAssembly'][@dependencyType='install']" |
  ForEach-Object { [PSCustomObject]@{ File = $_.Node.codebase; Present = Test-Path (Join-Path $dir $_.Node.codebase) } } |
  Format-Table -AutoSize
```

The manifest also stores size and hash of each file: see [signing](signing.md) for the order problem this causes.
