---
title: Satellite assemblies (localized .resx) in a VSTO add-in
description: Read when a VSTO add-in or a library it references uses .resx per language (de\<Assembly>.resources.dll) - build output, manifest, MSI, lookup at runtime.
---

# Satellite assemblies (localized .resx) in a VSTO add-in

They work without any extra setup, in the debug build and when installed by MSI.

Setup that was tested: SDK-style net48 library `MyLib` with `Texts.resx` + `Texts.de.resx`, referenced by a classic VSTO add-in project.

- The build writes `de\MyLib.resources.dll` into the library output and into the output of every VSTO project that references it.
- The VSTO `.dll.manifest` lists the file on its own as `<dependentAssembly dependencyType="install" codebase="de\MyLib.resources.dll">` with `language="de"`. Because of that, the file must be installed ([manifest-dependencies](manifest-dependencies.md)).
- MSI: a recursive harvest of the whole output (`<Files Include="!(bindpath.X)\**">`, [wix-toolset/harvesting-project-output](../wix-toolset/harvesting-project-output.md)) also ships the `de\` subfolder. No extra WiX entries are needed.
- At runtime VSTO shadow-copies the add-in (`AppDomain.ShadowCopyFiles` = true, under `%LOCALAPPDATA%\assembly\dl3\`), and the satellite assembly too. Lookups find it from bin\Debug (HKCU dev registration) and from Program Files (MSI).

## Lookup

```csharp
var rm = new ResourceManager("MyLib.Texts", typeof(MyLib.SomeType).Assembly);
rm.GetString("Hello", new CultureInfo("de-AT")); // German (de-AT -> de)
rm.GetString("Hello", new CultureInfo("fr"));    // neutral Texts.resx
```

- Base name = `<RootNamespace>.<folder path>.<file name without .resx>`.
- Pass the culture to each lookup. The thread's `CurrentUICulture` can differ from the Office UI language. Seen once: Visio `Application.Language` = 1033 (en-US), thread `CurrentUICulture` = en-GB (Windows display language).

Verified 2026-10: Visio 2019 C2R x64, VS 2026 (MSBuild 18), WiX 7. Other Office apps not tested; the mechanism is the same.
