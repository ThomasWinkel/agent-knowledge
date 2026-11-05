---
title: Visio application settings vs document settings
description: Read when code or a template changes Visio editing options (AutoConnect, connector split/delete, snap/glue to, snap strength, dynamic grid) - registry or document, persistence.
---

# Visio application settings vs document settings

`Application.Settings.*` and `Application.LiveDynamics` are per-user options: Visio writes them to `HKCU\Software\Microsoft\Office\16.0\Visio\Application` on quit, and every later session reads them back. A change by code is a change for **all** drawings of that user, not only yours. Prefer a document or shape cell where one exists (template, stencil).

Measured on Visio 2019 C2R x64 (16.0), 2026-11: Visio started as `VISIO.EXE` (not by COM), attached via `GetActiveObject`, values set, `Quit()`, registry compared, Visio restarted, values read. All values below survived the restart.

| Property | UI | Registry value (`Application` key, `REG_SZ` unless noted) | Default |
|---|---|---|---|
| `Settings.EnableAutoConnect` | Options > Advanced > Enable AutoConnect | `EnableAutoConnect` | `True` |
| `Settings.ConnectorSplittingEnabled` | Options > Advanced > Enable connector splitting | `AllowShapeSplitting` | `True` |
| `Settings.DeleteConnectorsEnabled` | Options > Advanced > Delete connectors when deleting shapes | `DeleteConnectorsEnabled` | `True` |
| `Settings.TransitionsEnabled` | Options > Advanced > Enable transitions (animated re-layout, Pan & Zoom) | `TransitionsEnabled` | `True` |
| `Settings.DrawingAids` | Snap & Glue > General > Drawing aids | `ToolSnapLines` | `True` |
| `Settings.SnapStrengthGridX`/`Y` | Snap & Glue > Advanced > Snap strength > Grid (pixels, 0-999) | `SnapGridX`/`SnapGridY` | `5` |
| `Settings.DeveloperMode` | Options > Advanced > Run in developer mode | `DeveloperTools` (`REG_DWORD`) | - |
| `LiveDynamics` | none | `LiveMouseDynamics` | `True` |

- `Settings.DeveloperMode` set by code shows (`True`) or hides (`False`) the Developer ribbon tab at once, also while a modal add-in dialog is open (UI Automation tab list, Visio 2019, 2026-11).
- A value at its default is absent from the registry; Visio adds it when set by code. To restore a user's state exactly, delete the added values while Visio is not running (setting the old value by code leaves the value behind, harmless).
- Visio writes the values only on quit: a second running Visio instance overwrites them on its own quit.
- `Document.DiagramServicesEnabled` is not persisted (docs) and only affects code-triggered behaviors. The save/set/restore block around it in recorded macros is macro recorder boilerplate, not needed for settings.

## No effect on code

Measured by COM in a new blank drawing (rectangles, `Shape.AutoConnect`):

- `DeleteConnectorsEnabled = True`: `Window.Selection.Delete()` and `Shape.Delete()` of a glued shape left all connectors in place. The option acts on UI deletes only (_unverified_ by hand).
- Connector splitting never happens by code: [visio-connector-splitting](visio-connector-splitting.md).

## Document and shape level

- `Document.DynamicGridEnabled` is saved in the file (`document.xml` `DocumentSettings/DynamicGridEnabled`) and comes from the template; new blank drawing: `True`. Set it in the template instead of per session.
- Snap & Glue dialog, "Snap to" / "Glue to" check boxes (shape geometry, vertices, intersections, connection points, ...) and the Snap/Glue switches are per document: `Document.SnapSettings`, `GlueSettings`, `SnapExtensions`, saved in `document.xml` `DocumentSettings` and copied from the template into a new drawing (measured). Bits (`VisSnapSettings`): RulerSubdivisions 1, Grid 2, Guides 4, Handles 8, Vertices 16, ConnectionPoints 32, Geometry 256, AlignmentBox 512, Extensions 1024, Intersections 65536, Disabled 32768 (Snap off). `VisGlueSettings`: Guides 1, Handles 2, Vertices 4, ConnectionPoints 8, Geometry 32, Disabled 32768. New blank drawing: `SnapSettings = 65847`, `GlueSettings = 9`, `SnapExtensions = 34`.
- Connector splitting: application option plus page and shape cells: [visio-connector-splitting](visio-connector-splitting.md).
