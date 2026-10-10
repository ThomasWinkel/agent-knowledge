---
title: WiX Toolset
description: "WiX Toolset v4+ (v7) MSI builds: SDK-style projects, HeatWave VS integration, OSMF EULA, file harvesting, package bitness."
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# WiX Toolset

WiX Toolset v4+ builds MSI packages and bundles from SDK-style `.wixproj` projects; Visual Studio integration is FireGiant's HeatWave extension. Read before creating, migrating or debugging a WiX installer. General WiX syntax is assumed known; this topic holds version facts, gotchas and verified recipes.

**Installer for a Visio/Excel solution (add-ins, stencils, templates): start with [office-solution-installer](office-solution-installer.md)**, the steps in order with a complete example.

**Installer for a VSTO/Office add-in: also read [vsto-addins](../vsto-addins/index.md).** Registration, manifest dependencies and signing live there, and the add-in fails silently without them.

**Installer shipping Visio stencils/templates:** read [visio-content-deployment](../visio-content-deployment/index.md).

## Key facts

- **Current versions** (checked 2026-10-08): WiX 7.0.0 (2026-04-06), HeatWave 1.0.8. Check for newer: `https://api.nuget.org/v3-flatcontainer/wixtoolset.sdk/index.json`, https://github.com/wixtoolset/wix/releases.
- **WiX v7 fails with WIX7015 until the OSMF EULA is accepted.** Never accept it yourself; ask the user: [osmf-eula](osmf-eula.md).
- **heat.exe is removed in v7** (last `WixToolset.Heat`: 6.0.2). Harvest with `<Files>`: [harvesting-project-output](harvesting-project-output.md).
- v3 and v4+ differ in project format, schema namespace, extensions and VS extension; `wix convert` migrates v3 sources: [versions-and-project-format](versions-and-project-format.md).
- C# custom actions: `WixToolset.Dtf.CustomAction`, own EULA check, x86 shim by default: [managed-custom-actions](managed-custom-actions.md).
- Packages default to x86; per-machine registry values then land in `WOW6432Node`: [package-bitness](package-bitness.md).
- `<Files>` with zero matches is only warning WIX8600. Treat it as an error.
- Verified with WiX 7.0.0, HeatWave 1.0.8, VS2022 17.14 (2026-07). Statements marked _Unverified_ are untested.

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [Harvesting project output](harvesting-project-output.md) — Read when adding another project's build output to a WiX v4+ package - ProjectReference, bindpath, <Files> syntax, Exclude, WIX8600, and checking the built MSI's contents.
- [HeatWave templates missing in New Project dialog](heatwave-templates-missing.md) — Read when HeatWave is installed but WixMsiPackage/WixBundle templates do not appear in Visual Studio's "Create a new project" dialog.
- [Managed custom actions (DTF)](managed-custom-actions.md) — Read when writing a C# custom action for a WiX v4+ MSI - WixToolset.Dtf.CustomAction project, .CA.dll, OSMF EULA, shim bitness, deferred action as SYSTEM, logging.
- [Installer for a Visio/Excel solution](office-solution-installer.md) — Read first when building an MSI for an Office solution with WiX v7 - VSTO add-ins (Visio, Excel), Visio stencils and templates - steps in order, complete example, test procedure.
- [OSMF EULA acceptance (WiX v7)](osmf-eula.md) — Read when a WiX v7 build or wix.exe command fails with WIX7015 "You must accept the Open Source Maintenance Fee (OSMF) EULA", or when setting up WiX v7 in CI.
- [Package bitness and registry redirection](package-bitness.md) — Read when choosing x86 vs x64 for a WiX package, when per-machine registry values land in WOW6432Node, or when Platform=x64 breaks a ProjectReference build.
- [Versions and project format](versions-and-project-format.md) — Read when choosing between WiX v3 and v4+, migrating a v3 project, picking the Visual Studio extension, or replacing heat.exe.
- [Visual Studio extension diagnostics](vs-extension-diagnostics.md) — Read when a Visual Studio extension (e.g. HeatWave) or its project templates misbehave - log locations, template caches, ActivityLog, correct vswhere workload check.
<!-- END GENERATED CONTENTS -->
