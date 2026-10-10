---
title: VSTO Office add-ins
description: "VSTO Office add-ins (incl. Visio): registration, MSI deployment, signing, load failures, user settings."
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# VSTO Office add-ins

VSTO add-ins are .NET Framework COM add-ins for desktop Office (Word, Excel, Outlook, Visio, ...). Read when deploying one with an MSI, when an installed add-in does not appear or load, or when it uses `Properties.Settings`. Building the MSI itself: [wix-toolset](../wix-toolset/index.md).

## Key facts

- VSTO is maintained but frozen: .NET Framework 4.8 is the last runtime, no .NET (Core) support. Microsoft points new cross-platform work to Office Web Add-ins.
- **Visio uses its own key** `Software\Microsoft\Visio\Addins`, not `...\Office\Visio\Addins`. MSI installs need `Manifest = file:///<path>.vsto|vstolocal`: [registration](registration.md).
- Take the registry KeyName from `keyName` in the generated `.dll.manifest`, never from the class name.
- Every build on a dev machine writes an HKCU registration that overrides the installed add-in. Delete it before installer tests: [dev-build-registration](dev-build-registration.md).
- Sign the assembly before `VisualStudioForApplicationsBuild` generates the manifests: [signing](signing.md).
- Office updates empty `Properties.Settings` unless `Upgrade()` runs: [user-settings](user-settings.md).

## "Installed, but the add-in does not appear or load"

In one real case (Visio 2019 x64) all five causes occurred one after another; each had to be fixed. Check in this order, details in [load-diagnostics](load-diagnostics.md):

1. Not in `COMAddIns`: wrong key path or KeyName ([registration](registration.md)).
2. Keys in `WOW6432Node` while Office is 64-bit, or the reverse ([wix-toolset/package-bitness](../wix-toolset/package-bitness.md), [registration](registration.md)).
3. Files listed in the manifest missing from the install folder ([manifest-dependencies](manifest-dependencies.md)).
4. `Manifest` value without `file:///` ([registration](registration.md)).
5. Expired manifest certificate, or assembly signed after manifest generation ([signing](signing.md)).

On a dev machine also check that the HKCU dev registration is not overriding the installed one ([dev-build-registration](dev-build-registration.md)).

Verified with Visio Pro 2019 C2R x64, WiX 7.0.0, VS2022 (2026-07); registration and loading also with Excel 2019 C2R x64 (2026-10). Other Office apps are not tested. Statements marked _Unverified_ are untested.

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [Dev build registration overrides the installed add-in](dev-build-registration.md) — Read when testing an installed VSTO add-in on a development machine, or when Office loads the add-in from bin\Debug instead of the install folder.
- [Load diagnostics](load-diagnostics.md) — Read when a VSTO add-in is installed but does not appear or does not load - COMAddIns check via COM, LoadBehavior, VSTO event log, VSTO_LOGALERTS, diagnosis order.
- [Manifest dependencies must be installed](manifest-dependencies.md) — Read when an installed VSTO add-in silently does not load although registration is correct, or when deciding which DLLs the installer must ship.
- [Installer prerequisite checks](prerequisite-checks.md) — Read when adding launch conditions to a VSTO add-in installer for the VSTO runtime or the Office application.
- [Registry registration via MSI](registration.md) — Read when registering a VSTO add-in from an MSI/WiX installer - registry path (Visio differs), required values, file:/// and |vstolocal, key name, HKMU, 32- and 64-bit Office.
- [Signing](signing.md) — Read when signing a VSTO add-in (Authenticode, manifests, MSI), choosing the MSBuild hook for signtool, separating debug/release certificates, or when an expired test certificate blocks loading.
- [User settings location and Office updates](user-settings.md) — Read when a VSTO add-in uses Properties.Settings - where user.config lives, why settings vanish after Office updates, and a safe Settings.Upgrade() pattern.
<!-- END GENERATED CONTENTS -->
