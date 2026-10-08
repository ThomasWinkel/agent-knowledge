---
title: HeatWave templates missing in New Project dialog
description: Read when HeatWave is installed but WixMsiPackage/WixBundle templates do not appear in Visual Studio's "Create a new project" dialog.
---

# HeatWave templates missing in New Project dialog

Seen with VS2022 Community 17.14.37516.0 + HeatWave 1.0.8 (2026-07): searching "WiX" shows only the v3 templates ("Setup Project for WiX v3"), none of `WixMsiPackage`, `WixBundle`, ...

## Fix that worked

1. Uninstall HeatWave via the **Visual Studio Installer** (not via Manage Extensions).
2. Reinstall via **Extensions → Manage Extensions → Online**.

Not sufficient: VS restart, `devenv /updateconfiguration`, dialog filters set to "All". Why the installer route matters is unknown.

## Ruled out (do not repeat)

- Extension not installed: VSIXInstaller reported `AlreadyInstalledException`; files and `ProjectTemplates\WixMsiPackage\*.vstemplate` were present.
- Missing workload `ManagedDesktop`: was installed (beware the wrong `vswhere` check, see [vs-extension-diagnostics](vs-extension-diagnostics.md)).
- `ActivityLog.xml` had no mention of HeatWave/FireGiant at all.
- Template IDs (`FireGiant.HeatWave.PackageProject`, `...BundleProject`) were listed in `InstalledTemplates.json` but not in the search index. Hypothesis (unverified): the templates' `<WizardExtension>` (MEF wizard `FireGiant.HeatWave.Wix.ProjectSystem.MsiPackage.MsiPackageProjectWizard`) fails silently; the v3 templates have none.

## Workaround: create the project without the wizard

The templates only use two placeholders.

1. Find the HeatWave folder: search `extension.vsixmanifest` files under `<VS install>\Common7\IDE\Extensions\*\` for `FireGiant.HeatWave`; templates are in `ProjectTemplates\<TemplateName>\`.
2. Copy `Package.wixproj`, `Package.wxs`, `Folders.wxs`, `ExampleComponents.wxs`, `Package.en-us.wxl` into the new project folder.
3. Replace `$WixToolsetSdkVersion$` with e.g. `/7.0.0` (→ `Sdk="WixToolset.Sdk/7.0.0"`), and `$safeprojectname$`/`$projectname$` with the project name.
4. Add it via **File → Add → Existing Project**.
