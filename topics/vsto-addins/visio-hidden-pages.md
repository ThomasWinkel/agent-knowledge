---
title: Hidden Visio pages (UIVisibility)
description: Read when Visio code hides pages with the PageSheet cell UIVisibility - what it hides (page tabs only), print and PDF export still include them, hidden active page.
---

# Hidden Visio pages (UIVisibility)

`page.PageSheet.CellsU["UIVisibility"].FormulaU = "1"` hides a page (also a background page) from the page tabs. It hides nothing else:

- **Printed and exported:** `Document.PrintOut` (all pages) and `Document.ExportAsFixedFormat` (PDF, `visPrintAll`) include hidden pages, and a hidden background page under its foreground page.
- `Window.Page = hiddenPage` works: the window shows the hidden page.
- Hiding the page the window shows: the window stays on it.

To keep a page out of print or PDF, its shapes must be on non-printing layers ([visio-layers](visio-layers.md)), or the code must print/export a page range that skips it.

Verified with Visio Pro 2019 C2R x64 by COM automation (`Visio.InvisibleApp`): `PrintOut` to `Microsoft Print to PDF` and `ExportAsFixedFormat`, PDF pages and text checked (2026-10). _Unverified:_ the page tabs in the UI, the UI print and Save as PDF dialogs.
