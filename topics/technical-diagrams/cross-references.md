---
title: Cross-references in circuit diagrams
description: Read when showing where the other parts of a device are drawn (coil <-> contacts, contact image, interruption points) - IEC 61082-1 clauses, EPLAN, AutoCAD Electrical options.
---

# Cross-references in circuit diagrams

A device drawn in parts (coil on one page, its contacts on others) needs texts that say where the other parts are. Tools agree on the shape: main part refers to all others, each other part refers back, location = sheet + column of a reference grid. Facts read 2026-11 from vendor help; IEC text is paid.

## Standard

- IEC 61082-1:2014, contents only (public preview): 5.7.3 Content area with Figure 8 "Example of a reference grid", 5.8 Cross-references with Figure 9 "Examples of the application of cross-references", 5.9 Hyperlinks, 7.4.3 Methods for representation of components, 7.4.4 Representation of components with movable parts, 11.3 Referencing. Their rules: _unverified_ (body not in the preview).

## EPLAN Electric P8 (eplan.help, Platform 2023; interruption points 2025)

- Device cross-references: the main function refers to all auxiliary functions, each auxiliary function back to the main function. Also single-line <-> multi-line, to device tag lists; panel layout parts both ways ([tools](tools.md)).
- Contact image `In path` or `On component`, per device; shows all placed functions plus unplaced device functions. Options: `Show in tabular form` (headers from `Contact image table: Labeling`, default `NO`/`NC`), `Display as symbol image`, `Show symbols`, `Hide left-hand / right-hand connection point designation`, `Display own cross-reference`. Component settings override project settings.
- Reference = page + column (+ row if the plot frame has rows). Separators `Ahead of the page` (`/`), `Between page and column` (`.`), e.g. `=AA1+BC1/1.8`. `Display full structure identifier in cross-reference` off = page and column only. `Without page number`, `Suppress identical page`, `Swap row and column information` (`20.1.A` -> `20.A.1`), `Number per row / column` (references as a matrix).
- Interruption points: pair, star or chain; `Only display page names in cross-reference` (`7.8/L-` -> `7/L-`).
- Not stated on these pages: how the column is computed, whether references update live.

## AutoCAD Electrical (help.autodesk.com 2025; table format 2016)

- Referencing styles `X-Y Grid`, `X Zones` (no Y axis; "In IEC, the most commonly used format is X Zones"), ladder. IEC cross-reference format `%S-%N` (`%S` sheet, `%N` reference value of the drawing's referencing style).
- Parent (coil) formats `Text`, `Graphical` (IEC or JIC symbols for NO, NC, Form C; one line per reference), `Table`. Unused contacts: `Display Unused Children (Contacts)` as `Separate Reference` or `Contact Count Totals`, `Fill Reference With` (e.g. `SP`). Children match the parent by equal TAG values.
- Update: table "automatically gets updated in real time"; text: "You must run the Component Cross-reference command to update any existing cross-referencing text."

## Others

- WSCAD 5.4 (docu.wscad.com): contact reflector = copies of the contacts under the coil, rotated 90 degrees, with the coordinates of the original contacts; vacant contacts "cannot be displayed"; pin numbers must be numeric.
- QElectroTech (manual 0.7/0.8, read as a summary): master with many slaves, slave with one master; label `Under the label of the element` or `Footer`; folio variables `%{f}` number, `%{l}` line, `%{c}` column.

## Design notes

- A contact image that lists unused parts needs the device's part list (which contacts exist); a tool that only knows placed symbols cannot show spares (WSCAD).
- PDF export of Visio keeps one hyperlink per shape: a clickable reference per line needs one shape per line ([visio-hyperlinks](../vsto-addins/visio-hyperlinks.md)).
