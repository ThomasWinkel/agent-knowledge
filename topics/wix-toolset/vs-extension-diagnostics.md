---
title: Visual Studio extension diagnostics
description: Read when a Visual Studio extension (e.g. HeatWave) or its project templates misbehave - log locations, template caches, ActivityLog, correct vswhere workload check.
---

# Visual Studio extension diagnostics

Paths for VS2022 (`17.0_<instance>`); VS2026 analogous.

| What | Where |
|---|---|
| Install log | `%TEMP%\dd_VSIXInstaller_<timestamp>_<hash>.log`: metadata, supported products, prerequisites, abort reason |
| Per-machine extensions | `<VS install>\Common7\IDE\Extensions\<random>\extension.vsixmanifest` |
| Per-user extensions | `%LocalAppData%\Microsoft\VisualStudio\<ver>_<instance>\Extensions\<random>\` |
| Registered template IDs | `%LocalAppData%\Microsoft\VisualStudio\<ver>_<instance>\InstalledTemplates.json` |
| New Project search index | same folder, `NpdProjectTemplateCache_en-US` |
| ActivityLog | `%AppData%\Microsoft\VisualStudio\<ver>_<instance>\ActivityLog.xml` |

- Extension folder names are random: find one by full-text search for its ID in all `extension.vsixmanifest` files.
- A template in `InstalledTemplates.json` is not necessarily in the search index.
- Rebuild caches: close VS, `devenv /updateconfiguration`. Compare the cache file timestamp to confirm it ran.
- ActivityLog: close VS, `& "<path>\devenv.exe" /log`, reproduce, close VS, read the log. It is UTF-16: test your search with a term that must occur before concluding "not found". It shows package/MEF load errors, not silent filtering.

## vswhere: checking a workload

Correct (empty output = not installed):

```
vswhere -requires Microsoft.VisualStudio.Workload.ManagedDesktop -format value -property displayName
```

Wrong: filtering `vswhere -all -products * -format json` output on `packages.id`. Without `-packages` the `packages` array is empty or partial and reports installed workloads as missing.
