---
title: OSMF EULA acceptance (WiX v7)
description: Read when a WiX v7 build or wix.exe command fails with WIX7015 "You must accept the Open Source Maintenance Fee (OSMF) EULA", or when setting up WiX v7 in CI.
---

# OSMF EULA acceptance (WiX v7)

Since v6, WiX binaries are released under the Open Source Maintenance Fee (OSMF) EULA. v7 enforces an explicit acceptance: without it every build fails.

```
error WIX7015: You must accept the Open Source Maintenance Fee (OSMF) EULA to use WiX Toolset v7. For instructions, see https://wixtoolset.org/osmf/
```

**Do not accept it on your own.** Accepting binds the user or company to a payment obligation. Stop, show the user the error and the terms below, and let them decide and choose the method.

## Terms (OSMF EULA v1.1, WiX v7)

- Fee applies only to users who use WiX in revenue-generating activities with annual gross revenue of US$10,000 or more. Users paying FireGiant for support/maintenance are exempt.
- Amount and payment are set by the project, not the EULA: GitHub Sponsors of `wixtoolset`. Currently reported as US$10/month (not stated in the EULA; check https://wixtoolset.org/osmf/).
- Self-attested; nothing is checked technically. Source code stays freely licensed; the EULA covers the binary release.
- Text: https://github.com/wixtoolset/wix/blob/v7.0.0/OSMFEULA.txt

## Ways to accept (value = `wix<major>`, e.g. `wix7`)

| Scope | How |
|---|---|
| Per user, once (writes `%USERPROFILE%\.wix\wix7-osmf-eula.txt`) | `wix eula accept wix7` or `dotnet build -t:AcceptEula -p:EulaId=wix7` |
| Per project (versioned, works in CI) | `.wixproj`: `<PropertyGroup><AcceptEula>wix7</AcceptEula></PropertyGroup>` |
| Per command | `wix build -acceptEula wix7 ...` |

- The MSBuild check is target `CheckLicenseAcceptance` (before `CoreCompile`). It is skipped when `AcceptEula` is set to any value, otherwise it requires the acceptance file.
- `WixToolset.Dtf.CustomAction` (C# custom action projects) runs the same check before `CoreCompile`: that `.csproj` fails with WIX7015 too and takes the same `AcceptEula` property.
- CI agents run under another user profile: use the project property or `-p:AcceptEula=wix7` there.
- Source: `src/wix/WixToolset.Sdk/tools/wix.targets` and `src/wix/WixToolset.Core/CommandLine/EulaCommand.cs` in tag `v7.0.0`.
