---
title: Protecting Visio shapes against formatting and themes
description: Read when users must not change fill, line or font of a (group) shape - LockFormat on groups vs ribbon, page themes ignore all locks, THEMEVAL vs fixed formulas.
---

# Protecting Visio shapes against formatting and themes

Measured 2026-11-05, Visio 2019 C2R x64 build 20527 (English UI), own visible Visio by COM. Ribbon buttons pressed by UI Automation (Home tab: `Fill RGB(...)`, `Line RGB(...)`, `Font Color RGB(...)` = apply last color), theme by the Design tab gallery (`Linear`) and by `Page.SetTheme(i)`. Test shape: group (`SelectMode 0`) of a frame rectangle (fill) and a text rectangle (`FillPattern 0`, `LinePattern 0`).

## Ribbon formatting

- Formatting a group applies to the group **and every sub-shape**: the text sub-shape got `FillPattern 1`, `LinePattern 1`, `FillForegnd = THEMEGUARD(RGB(255,255,0))`.
- `LockFormat = 1` **on the group only** blocks Fill, Line and Font Color for the group and all sub-shapes. No message, the buttons stay enabled, nothing changes.
- `LockFormat = 1` on the sub-shapes only: the group's own cells still change, the sub-shapes not.
- Sub-selected member of a group without `LockFormat` (`Window.Select(sub, 3)`): Fill changes only that member.
- Code is not blocked: `CellsU('FillForegnd').FormulaU = ...` on a `LockFormat` shape works.
- Not measured: Format Shape pane, Format Painter, Quick Styles gallery.

## Themes

- A page theme rewrites the **results of every cell holding `THEMEVAL()`**: fill, line color, line weight, text color, font (`Char.Font` 4 -> 118). Default shapes have `THEMEVAL()` in these cells.
- Neither `LockFormat` nor `LockThemeColors`, `LockThemeEffects`, `LockThemeFonts`, `LockThemeIndex`, `LockVariation`, `LockThemeConnectors` (on group and sub-shapes) stop it, by Design tab or `Page.SetTheme`.
- Fixed formulas resist: `LineColor = RGB(0,0,0)`, `LineWeight = 0.75 pt`, `Char.Color = RGB(0,0,0)` kept their values. Ribbon formats are stored as `THEMEGUARD(RGB(...))` and resist too.
- So a shape that must look the same in any theme needs fixed formulas in every cell that would read the theme (replace `THEMEVAL()` by its current result).
- Gotcha: theme `Linear` gave a white line on a white-filled frame: the frame disappears.
- `Page.SetThemeVariant(5, 2)` after `SetTheme(5)` threw `Requested operation is presently disabled.` (cause unknown).

## Related

- `LockGroup = 1`: no ungroup, no delete of members ([visio-glue-events](visio-glue-events.md#keeping-glue-out-of-a-built-group-shape)); it also makes `Shape.Parent = group` fail silently ([visio-build-shapes-by-code](visio-build-shapes-by-code.md)).
