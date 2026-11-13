---
title: Visio content deployment
description: "Installing Visio stencils and templates so Visio lists them: MSI PublishComponent (Solution Publishing), ConfigChangeID, content cache, file paths."
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# Visio content deployment

Copying `.vssx`/`.vstx` files to disk is not enough: Visio shows stencils under "More Shapes" and templates under "New" only if they are published or lie in a configured file path. Read before writing an installer that ships Visio stencils or templates. VSTO add-in registration is a separate topic: [vsto-addins](../vsto-addins/index.md); MSI build basics: [wix-toolset](../wix-toolset/index.md).

## Key facts

- Two mechanisms:
  - **Publishing** (Solution Publishing): rows in the MSI `PublishComponent` table, machine-wide, the Microsoft-recommended way: [publish-component](publish-component.md).
  - **Path discovery**: Visio file-path options (Options → Advanced → File Locations) = per-user registry values. Unfit for per-machine installs.
- Microsoft documents publishing only for Visio 2003/2007. **Visio 2016+ reads only the undocumented `...B30x` GUIDs** (verified on 2019 C2R); the "version-neutral" IDs from the docs are ignored.
- The installer must change `ConfigChangeID` after install and uninstall, or Visio keeps its old cache. MSI cannot do this without a custom action.
- Visio caches installed content per user in `%LocalAppData%\Microsoft\Visio\content16.dat` (thumbnails: `thumbs.dat`). Rebuilt when `HKLM\Software\Microsoft\Office\Visio\ConfigChangeID` changes; deleting the file with Visio closed forces a rebuild.
- Stencils need a unit suffix (`_m`/`_M` metric, `_U` or none = US): More Shapes filters by the active drawing's units ([publish-component](publish-component.md), AppData).
- Path discovery wins over publishing: a stencil/template in a file path option (e.g. a developer's repo folder) with the same file name as a published one keeps the published entry out of `content16.dat`. Clear the paths for installer tests. A document saved while its workspace stencil is missing at its path gets the path of the published file with that name in `visio/windows.xml` (verified on 2019).
- Statements marked _Unverified_ are untested.

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [Preview picture of a template or stencil](preview-picture.md) — Read when a Visio template/stencil/drawing shows a blurry or empty picture in File > New or Explorer - where the picture lives, what Visio writes, replace it, keep it in drawings.
- [Publishing via the PublishComponent table](publish-component.md) — Read when publishing Visio stencils/templates from an MSI - PublishComponent GUIDs per Visio version, Qualifier and AppData format, WiX <Category>, ConfigChangeID bump.
<!-- END GENERATED CONTENTS -->
