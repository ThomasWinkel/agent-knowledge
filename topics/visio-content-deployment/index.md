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
- Statements marked _Unverified_ are untested.

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [Publishing via the PublishComponent table](publish-component.md) — Read when publishing Visio stencils/templates from an MSI - PublishComponent GUIDs per Visio version, Qualifier and AppData format, WiX <Category>, ConfigChangeID bump.
<!-- END GENERATED CONTENTS -->
