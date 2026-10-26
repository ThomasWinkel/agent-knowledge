---
title: VSTO Office add-ins
description: "VSTO Office add-ins: registration, MSI, signing, load failures, settings, localization, SDK refs; Visio: ShapeSheet, windows, layers, hidden pages, ungroup, performance, data; Excel tables; ribbon."
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# VSTO Office add-ins

VSTO add-ins are .NET Framework COM add-ins for desktop Office (Word, Excel, Outlook, Visio, ...). Read when deploying one with an MSI, when an installed add-in does not appear or load, when it uses `Properties.Settings` or localized `.resx`, or when other projects use its VSTO types. Building the MSI itself: [wix-toolset](../wix-toolset/index.md).

## Key facts

- VSTO is maintained but frozen: .NET Framework 4.8 is the last runtime, no .NET (Core) support. Microsoft points new cross-platform work to Office Web Add-ins.
- **Visio uses its own key** `Software\Microsoft\Visio\Addins`, not `...\Office\Visio\Addins`. MSI installs need `Manifest = file:///<path>.vsto|vstolocal`: [registration](registration.md).
- Take the registry KeyName from `keyName` in the generated `.dll.manifest`, never from the class name.
- Every build on a dev machine writes an HKCU registration that overrides the installed add-in. Delete it before installer tests: [dev-build-registration](dev-build-registration.md).
- Files next to the add-in DLL: use `Assembly.CodeBase`, not `Location` (shadow copy): [addin-folder](addin-folder.md).
- Sign the assembly before `VisualStudioForApplicationsBuild` generates the manifests: [signing](signing.md).
- Office updates empty `Properties.Settings` unless `Upgrade()` runs: [user-settings](user-settings.md).
- Localized `.resx` (satellite assemblies `de\*.resources.dll`) work in debug and MSI without setup; VSTO lists them in the manifest: [satellite-assemblies](satellite-assemblies.md).
- An SDK-style library using VSTO ribbon types needs an extra search path and VS MSBuild: [sdk-project-office-tools-reference](sdk-project-office-tools-reference.md).
- Visio: a ShapeSheet cell reaches the add-in only by a marker event, and only `RUNADDONWARGS("QueueMarkerEvent",...)` tells the shape: [visio-shapesheet-calls](visio-shapesheet-calls.md).
- Visio: `ActiveWindow` is `null` without a document; its `Selection` is empty in a ShapeSheet window, holds master shapes in a master edit window and skips shapes selected inside a group; `CreateSelection(visSelTypeAll)` takes shapes on locked and hidden layers: [visio-active-window](visio-active-window.md).
- Visio: every `Window.Page = ...` by code is an Undo step (`Go To Page`): wrap page loops in one undo scope; zoom (`ViewFit`) is kept per page and saved in the file, but does not make the document dirty: [visio-page-view](visio-page-view.md).
- Visio: a ShapeSheet cell set by code holds millions of characters (no 64k limit); 1,000,000 survived save and reopen: [visio-cell-string-length](visio-cell-string-length.md).
- Visio: a changed stencil master never reaches drawings that use it: a drop adds `Name.<n>`, old instances keep the old copy; edit the document master to update them; only edit in place keeps `BaseID`: [visio-master-versions](visio-master-versions.md).
- Visio: `Pages[..]!Sheet.n!` references work across pages and update live; deleting or cut/pasting the target silently turns them into constants, copying the referencing shape alone keeps or breaks them; `SETATREF` writes through across pages: [visio-cross-shape-references](visio-cross-shape-references.md).
- Visio: `ConnectionsAdded`/`Deleted` fire per glue change (also on Undo, `IsUndoingOrRedoing`), not on paste or on delete of the target; a cell written in the handler joins the glue's Undo step: [visio-glue-events](visio-glue-events.md).
- Visio: typed values in cells (text, number, bool, date), what survives save; writing a `GUARD` cell throws `Cell is guarded.` unless forced: [visio-cell-values](visio-cell-values.md).
- Visio: Shape Data lists (`Prop.Format`) and `INDEX`/`LOOKUP` split at `;` in every region; `LISTSEP()` breaks them, `ResultStr` formats numbers by region: [visio-region-settings](visio-region-settings.md).
- Visio: a shape on several layers is visible/printed while any layer is visible/printing (print ignores visibility), locked if any is locked; a locked layer blocks paste and drop, and cannot be deleted, nor its shapes; layers travel by name on paste; `;` breaks layer names: [visio-layers](visio-layers.md).
- Visio: a dropped undo scope takes back everything inside, also committed nested scopes; a dropped nested scope only its own changes. Undo of `Layer.Name = x` is broken: rename via `NameU` + the `visLayerName` cell: [visio-undo-scopes](visio-undo-scopes.md), [visio-layers](visio-layers.md).
- Visio: `UIVisibility` = 1 hides a page only from the page tabs; print and PDF export still include it: [visio-hidden-pages](visio-hidden-pages.md).
- Visio: `Shape.Ungroup()` on a master instance shows a dialog also when called by code; set `Application.AlertResponse = 1` for the call: [visio-ungroup](visio-ungroup.md).
- Visio: `ShowChanges = false` deselects all shapes and blocks selection changes by code; `UndoEnabled = false` clears the undo history; `EventsEnabled = 0` hits all add-ins; in one measurement the switches gave nothing, `Page.SetResults` 3-5x: [visio-performance-switches](visio-performance-switches.md).
- Visio: release every `Shape` and `Cell` object while scanning many shapes (`Marshal.ReleaseComObject`): exact scan of 20,000 shapes 4.6 s instead of 17-18.5 s; a batch read by master is faster, but a failing `Page.GetResults` can block that page for the session: [visio-scan-shapes](visio-scan-shapes.md).
- Visio: `DataRecordset.GetRowData` returns typed values (`string`, `int`, `double`, `DateTime`, `bool`, `decimal`, `DBNull`); row 0 = column names: [visio-data-recordsets](visio-data-recordsets.md).
- Visio: `DataRecordsets.Add` sets no primary key (refresh matches by row position); key matching ignores case; duplicate keys unlink shapes; `Refresh()` and UI Refresh All resolve a relative `Data Source` against the process's current folder (set by File Open), not the document's: [visio-data-linking](visio-data-linking.md).
- Excel by COM: write a table as one array; strings like `007` turn into numbers unless formatted `@` first; the table name is `ListObject.DisplayName`, not `Name`; with a filter on the sheet a multi-cell write fills only visible cells, all with the first value; a filter of another table makes a row delete take whole sheet rows; find a workbook open in any instance by path in the ROT, not by `GetObject(path)`: [excel-write-table](excel-write-table.md).
- Excel data validation by COM: `Formula1` uses the region's list separator; a typed list over 255 characters breaks the saved file: [excel-data-validation](excel-data-validation.md).
- A test can click a custom ribbon button by UI Automation: [ribbon-ui-test](ribbon-ui-test.md).
- WinForms dialog with an editable `DataGridView` and a `CancelButton`: Esc while typing in a cell closes the whole form: [winforms-dialogs](winforms-dialogs.md).
- Ctrl+click = help on every ribbon button without code per handler: wrap VSTO's private handler field; `file://` URLs lose their `#anchor` in `Process.Start`: [ribbon-click-help](ribbon-click-help.md).
- Ribbon texts can be changed at runtime (e.g. in `Load`), except `RibbonSeparator.Title`; an exception in `Load` silently hides the whole tab: [ribbon-runtime-properties](ribbon-runtime-properties.md).
- Built-in icon for a ribbon button (`imageMso`): `GetImageMso` from another process returns no picture, but its error tells whether an ID exists: [ribbon-office-icons](ribbon-office-icons.md).

## "Installed, but the add-in does not appear or load"

In one real case (Visio 2019 x64) all five causes occurred one after another; each had to be fixed. Check in this order, details in [load-diagnostics](load-diagnostics.md):

1. Not in `COMAddIns`: wrong key path or KeyName ([registration](registration.md)).
2. Keys in `WOW6432Node` while Office is 64-bit, or the reverse ([wix-toolset/package-bitness](../wix-toolset/package-bitness.md), [registration](registration.md)).
3. Files listed in the manifest missing from the install folder ([manifest-dependencies](manifest-dependencies.md)).
4. `Manifest` value without `file:///` ([registration](registration.md)).
5. Expired manifest certificate, or assembly signed after manifest generation ([signing](signing.md)).

On a dev machine also check that the HKCU dev registration is not overriding the installed one ([dev-build-registration](dev-build-registration.md)).

Verified with Visio Pro 2019 C2R x64, WiX 7.0.0, VS2022 (2026-07); registration and loading also with Excel 2019 C2R x64 (2026-10). Other Office apps are not tested. Statements marked _Unverified_ are untested.

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [Folder of the add-in DLL at runtime](addin-folder.md) — Read when a VSTO add-in must find files next to its DLL or in its install folder (help, data, config) - Location is the shadow copy, CodeBase the real folder.
- [Dev build registration overrides the installed add-in](dev-build-registration.md) — Read when testing an installed VSTO add-in on a development machine, or when Office loads the add-in from bin\Debug instead of the install folder.
- [Excel data validation by COM](excel-data-validation.md) — Read when code adds Excel data validation (drop-down lists, custom formulas) by COM - list separator by region, 255-character limit that breaks the file, booleans, case.
- [Writing a table to Excel by COM](excel-write-table.md) — Read when code writes to an Excel table by COM - array write, text turning into numbers, dates, append, delete rows, add columns, filtered or protected sheets, names, open workbook, cleanup.
- [Load diagnostics](load-diagnostics.md) — Read when a VSTO add-in is installed but does not appear or does not load, or to check that a dependency loaded - COMAddIns via COM, LoadBehavior, VSTO event log, VSTO_LOGALERTS.
- [Manifest dependencies must be installed](manifest-dependencies.md) — Read when an installed VSTO add-in silently does not load although registration is correct, or when deciding which DLLs the installer must ship.
- [Installer prerequisite checks](prerequisite-checks.md) — Read when adding launch conditions to a VSTO add-in installer for the VSTO runtime or the Office application.
- [Registry registration via MSI](registration.md) — Read when registering a VSTO add-in from an MSI/WiX installer - registry path (Visio differs), required values, file:/// and |vstolocal, key name, HKMU, 32- and 64-bit Office.
- [Ctrl+click on a ribbon button opens help](ribbon-click-help.md) — Read when a VSTO ribbon should open help on Ctrl+click on all buttons without code per handler - wrap the click handlers, open a local help page at an anchor.
- [Office icons for ribbon buttons](ribbon-office-icons.md) — Read when choosing a built-in Office icon (imageMso, OfficeImageId) for a custom ribbon button - check that an ID exists, view the icons, no icon for some commands.
- [Changing ribbon properties at runtime](ribbon-runtime-properties.md) — Read when code sets properties of a VSTO designer ribbon after it is created (e.g. translated Label/ScreenTip/SuperTip in Load) - which can be set, "Component is read only", how to check.
- [Clicking a ribbon button from a test script](ribbon-ui-test.md) — Read when a test must click a control of an add-in's custom ribbon tab (VSTO designer or XML) without a person, or drive the dialog it opens - UI Automation and Win32 in PowerShell.
- [Satellite assemblies (localized .resx) in a VSTO add-in](satellite-assemblies.md) — Read when a VSTO add-in or a library it references uses .resx per language (de\<Assembly>.resources.dll) - build output, manifest, MSI, lookup at runtime.
- [Referencing VSTO assemblies from an SDK-style project](sdk-project-office-tools-reference.md) — Read when a non-VSTO SDK-style library (net48) must use VSTO types such as RibbonBase or RibbonButton, e.g. shared ribbon code - search path, dotnet build limit.
- [Signing](signing.md) — Read when signing a VSTO add-in (Authenticode, manifests, MSI), choosing the MSBuild hook for signtool, separating debug/release certificates, or when an expired test certificate blocks loading.
- [User settings location and Office updates](user-settings.md) — Read when a VSTO add-in uses Properties.Settings - where user.config lives, why settings vanish after Office updates, and a safe Settings.Upgrade() pattern.
- [Visio ActiveWindow and Selection by window type](visio-active-window.md) — Read when a Visio add-in command uses ActiveWindow or its Selection, or selects shapes by code - ShapeSheet, master and group edit window, no document, group selection, select all.
- [Long strings in Visio ShapeSheet cells](visio-cell-string-length.md) — Read when storing long text (JSON, encoded data) in Visio ShapeSheet cells from code - no 64k limit via the API, what survives save and reopen, sources claiming 64k.
- [Typed values in Visio ShapeSheet cells](visio-cell-values.md) — Read when Visio code writes or reads text, numbers, booleans or dates in ShapeSheet cells (empty cells, types, errors, new rows, master instances, GUARD, read-only documents).
- [Visio cross-shape references - pages, delete, copy, SETATREF, containers](visio-cross-shape-references.md) — Read when a ShapeSheet formula reads another shape (Sheet.n!, Pages[..]!, SETATREF, CONTAINERSHEETREF) and must survive page rename, delete, cut/paste, copy or another document.
- [Linking Visio shapes to data rows and refreshing](visio-data-linking.md) — Read when Visio code links shapes to data recordset rows or refreshes linked data - primary key, row matching, refresh conflicts, shape data matching, edited values, column changes.
- [Reading Visio data recordsets](visio-data-recordsets.md) — Read when Visio code reads linked external data (DataRecordsets, GetRowData, DataColumns) or links an Excel sheet by code - value types, column names, empty fields, ACE connection, relative paths.
- [Visio glue and container events - what fires, Undo, paste, delete](visio-glue-events.md) — Read when an add-in reacts to glue (ConnectionsAdded/Deleted) or container membership events, writes cells in the handler, or must handle Undo, paste and delete.
- [Hidden Visio pages (UIVisibility)](visio-hidden-pages.md) — Read when Visio code hides pages with the PageSheet cell UIVisibility - what it hides (page tabs only), print and PDF export still include them, hidden active page.
- [Visio layers by code](visio-layers.md) — Read when Visio code creates, names, hides, locks or deletes layers, or shapes are on several layers - which layer wins for visible/print/lock, names, copy/paste and master drop.
- [Visio master versions - stencil edits and documents with old copies](visio-master-versions.md) — Read when shipping a changed master in a stencil, or when code must bring a new master version into drawings that already use the old one - BaseID, UniqueID, drop, MatchByName.
- [Visio page turns and zoom by code](visio-page-view.md) — Read when Visio add-in code turns pages (Window.Page) or zooms (ViewFit, Zoom) - Undo steps per page turn, zoom kept per page and in the file, selection, dirty flag.
- [Visio performance switches for bulk changes](visio-performance-switches.md) — Read when Visio code changes many shapes or cells and you weigh ScreenUpdating, ShowChanges, DeferRecalc, EventsEnabled, UndoEnabled or batch calls - effects, side effects, one measurement.
- [Visio ShapeSheet values and Windows region settings](visio-region-settings.md) — Read when Visio code builds Shape Data lists (Prop.Format), uses INDEX/LOOKUP/LISTSEP, or reads numbers with ResultStr - what depends on the list separator and decimal symbol.
- [Scanning all shapes of a Visio document fast](visio-scan-shapes.md) — Read when Visio code reads cells of many shapes across all pages (find by type or key, build an index) - release COM objects, batch reads, GetResults failures.
- [Visio shape names of copies](visio-shape-names.md) — Read when Visio code compares Shape.Name/NameU, e.g. to find shapes of the same kind - the .<number> suffix of copies, when it is not the ID, names that lose it.
- [Visio shape UniqueID on copy, cut and paste](visio-shape-uniqueid.md) — Read when Visio code needs a stable shape identity, e.g. to tell a copied shape from the original or a moved one - which edits and Undo keep, drop or never set Shape.UniqueID.
- [Calling a Visio add-in from the ShapeSheet](visio-shapesheet-calls.md) — Read when a ShapeSheet cell (EventDrop, EventDblClick, Actions) must run code in a Visio VSTO/COM add-in - marker events, context string format, QUEUEMARKEREVENT vs RUNADDONWARGS, redo.
- [Visio undo scopes - rollback and nesting](visio-undo-scopes.md) — Read when Visio code uses BeginUndoScope/EndUndoScope (one Undo step, rollback, nesting) or must store document data that Undo does not take back.
- [Visio Shape.Ungroup from code](visio-ungroup.md) — Read when a Visio add-in or script calls Shape.Ungroup - confirmation dialog for master instances, AlertResponse, errors for non-groups and locked groups, what is left of the group.
- [WinForms dialogs in add-ins - DataGridView gotchas](winforms-dialogs.md) — Read when an add-in dialog edits values in a DataGridView: Esc closes the whole form (CancelButton), error icons that do not show, check boxes that apply late.
<!-- END GENERATED CONTENTS -->
