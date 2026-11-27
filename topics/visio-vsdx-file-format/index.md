---
title: Visio VSDX file format
description: "Reading a saved Visio .vsdx without Visio: package parts, top-level shapes, master inheritance of rows, cell value meaning vs COM, reading while open, speed."
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# Visio VSDX file format

A `.vsdx` is an Open Packaging (ZIP) file of XML parts. Reading it directly needs no Visio and skips Visio's page loading. Read this topic before code reads a saved drawing without Visio, or compares XML values with what COM returns. COM-side cell values: [visio-cell-values](../vsto-addins/visio-cell-values.md); COM scan speed and page loading: [visio-scan-shapes](../vsto-addins/visio-scan-shapes.md).

## Key facts

- Parts: `visio/document.xml` -> rels -> `pages/pages.xml`, `masters/masters.xml`; each `<Page>`/`<Master>` has `<Rel r:id>` to its own part. Top-level shapes = direct `Shape` children of `PageContents/Shapes`. `pages.xml` lists background pages first; COM `Pages` has the foreground pages first: [package-and-inheritance](package-and-inheritance.md).
- An instance gets its rows from the master's top-level shape; merge rows by `N`, cells by `N`, `Del="1"` removes. With the right merge order the row index equals COM's: [package-and-inheritance](package-and-inheritance.md).
- `V` is the cached result. Strings (`U="STR"`) and Booleans map 1:1 to `ResultStr`; number texts cannot be rebuilt from `V`: [cell-values](cell-values.md).
- No `F` attribute = the formula is the constant `V`; `F="No Formula"` = empty formula; `F="Inh"` = formula inherited, `V` still local: [cell-values](cell-values.md).
- The file of a document open in Visio can be read (`FileShare.ReadWrite | FileShare.Delete`, measured on a local file): it is the last saved state, so use it only while `Document.Saved` is true. Opening a drawing and idling did not set `Saved` to false. No Visio API saves a copy without renaming the document (`SaveAs`/`SaveAsEx`); `IPersistFile` on a `Document` was reported as `E_NOINTERFACE` out of process (not tried in process).
- Speed (measured 2026-11, 379 KB file, 20 pages, 4,001 top-level shapes, 27,521 shapes): all User rows of the document and all top-level shapes (135,031 rows) by `XDocument` in 0.3 s on one thread (net48), 0.47 s as a new process. COM on the same drawing in process: ~7 s for Visio to load the pages at first access, plus the reads.

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [VSDX cell values vs COM results](cell-values.md) — Read when code turns a .vsdx <Cell V F U> into the text or value COM gives (ResultStr, FormulaU), or must tell empty, constant and inherited cells apart in the XML.
- [VSDX package parts and master inheritance](package-and-inheritance.md) — Read when code finds pages, masters and top-level shapes in a .vsdx, or merges a shape's ShapeSheet rows with its master's rows as Visio does.
<!-- END GENERATED CONTENTS -->
