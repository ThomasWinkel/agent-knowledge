---
title: Visio cross-shape references - pages, delete, copy, SETATREF, containers
description: "Read when a ShapeSheet formula reads another shape or page (Sheet.n!, Pages[..]!, ThePage!, SETATREF, CONTAINERSHEETREF): names, delete, copy, nesting, stale values, speed."
---

# Visio cross-shape references

A formula can read a cell of another shape on the same page (`Sheet.5!Prop.X`) or on another page (`Pages[PageA]!Sheet.5!Prop.X`). The value updates live, also without any add-in. Visio keeps the reference only while the target shape keeps its identity; then it silently turns the formula into a constant or `REF()`.

Measured 2026-10-26, Visio 2019 C2R x64, COM from PowerShell (`Cut`/`Copy`/`Paste`, `Shape.Delete`, `Duplicate`).

## What Visio does with the reference

| Action | Formula afterwards |
|---|---|
| Change target cell | unchanged, value follows |
| Page `Name` changed (local name only) | unchanged: `Pages[..]` holds `NameU` |
| Page `NameU` changed | rewritten to the new `NameU` |
| Target deleted | **constant** with the last value (`"v1"`); Undo restores the reference |
| Page of the target deleted | **constant**, like target deleted; Undo of the page delete restores the reference |
| Target cut + pasted (same or other page, new ID) | **constant** with the last value |
| Referencing shape copied alone, cross-page ref | unchanged: copy points to the original target |
| Referencing shape duplicated (`Duplicate`) | unchanged |
| Referencing shape copied alone, same-page ref (`Sheet.n!`) | `REF()`; shows the old value until the cell recalculates, then `#REF!` (`Cell.Error` 55) |
| Referencing shape + target copied together (same or other page) | rewritten to the copied target |
| Referencing shape pasted into another document | `REF()` => `#REF!` |

- Nothing errors and nothing reports the change: a lost link looks like a normal value. An add-in that relies on links must store the target's identity (e.g. a GUID cell) next to the formula to detect and repair them.
- Keep the formula instead of a constant: in `BeforeShapeDelete` of the target, find the referencing cells by `Cell.Dependents` of the target's cells (also across pages; it also lists formulas inside the target's own sub-shapes, skip those) and write them before Visio does, e.g. `FormulaForceU = ""` = master formula again. Same Undo step as the delete: Undo brings target and reference back, Redo deletes again with the master formula (skip the handler while `IsUndoingOrRedoing`). Works when both shapes are deleted in one selection. A page delete gives no `BeforeShapeDelete` (event list of the document). Measured 2026-10-28 in process: [visio-cell-change-events](visio-cell-change-events.md#handlers-in-process).
- A `Prompt` cell can be the target like a value cell: `Pages[B]!Sheet.5!User.x.Prompt` (also same page) gives its value, `Cell.Precedents` lists `User.x.Prompt`, a page rename and a copy of reference + target together behave as above (Visio 2019 build 20527, 2026-11-11). So one row can carry two values others read, e.g. value + `.Prompt` = a hyperlink target + its page name.
- Glue (`BeginX = PAR(PNT(Sheet.n!Connections.X1.X, ...))`) behaves like a same-page reference: a 1D shape copied alone is pasted unglued.
- Formula text is not a stable key: a same-page reference written as `Sheet.1!Prop.X` reads back as `Labbox!Prop.X` from `FormulaU` once the target has the `NameU` `Labbox` (`Formula` shows the local `Name`); a cross-page `Pages[A]!Sheet.1!Prop.X` kept `Sheet.1` after the rename. Not always: written as `SETATREF(Pages[Page-1]!Sheet.151!Prop.X)` to a master instance named `My Part 1/A.151`, it read back as `SETATREF(Pages[Page-1]!'My Part 1/A.151'!Prop.X)` (name quoted with `'`), while one to `Sheet.8` (named `Symbol`) stayed `Sheet.8` (seen 2026-10-28; rule unknown). To check where a formula points, use `Cell.Precedents` (array of the referenced `Cell`s: `.Shape`, `.Name`), not string compare. Measured 2026-10, Visio 2019 C2R x64.

## Page names in `Pages[..]`

Measured 2026-10, Visio 2019 C2R x64, COM from PowerShell, values read back after save and reopen.

- **No quoting.** Write the page name as it is: `Pages[My Page]!Sheet.1!Prop.X`. Works with space, leading space, `-`, `.`, `'`, `"`, `!`, `;`, `,`, `+`, umlauts, leading digit, and a page named `Pages`. `Pages['My Page']` and `Pages["My Page"]` throw `#NAME?`.
- **`]` in a page name breaks the reference.** Writing `Pages[a]b]!...` or `Pages[a[b]]!...` throws `Error in formula.`. Renaming the target page to such a name makes Visio rewrite existing references to `Pages[a]b]!...` itself; they keep working in the session, but **after save and reopen they are constants** (last value, or `0`). `[` alone (`a[b`) survives. Check page names for `]` before relying on cross-page references.
- **`NameU` vs `Name`.** `FormulaU` shows the page's `NameU`, `Formula` its local `Name`. `FormulaU = "Pages[x]!..."` resolves `x` by `NameU` (a `Name` is accepted too); `Formula` resolves by `Name`. Write `FormulaU` with `NameU`.
- One name space: setting a page's `NameU` to another page's `Name` throws `The name '...' is already in use.`; the reverse (`Name` = another page's `NameU`) is allowed, so `Pages[X1]` can mean different pages in `FormulaU` and `Formula`.
- A reference to the own page is shortened: `Pages[<own page>]!Sheet.2!Prop.X` reads back as `Sheet.2!Prop.X`, to the own shape as `Prop.X`.
- Background pages and hidden pages (`UIVisibility` = 1) work as targets; renaming a background page's `NameU` rewrites the references.
- Missing page, shape or cell in the reference: the write throws `#NAME?`.
- `Cell.Precedents` returns the target cell across pages (`.Shape.ContainingPage`, `.Shape`, `.Name`) for a plain reference, `SETATREF(..)`, `GUARD(SETATREF(..))`, `IFERROR(<ref>, "none")` and `SETATREF(..)&""`; for `<ref1>&<ref2>` both cells. Same after reopen.

## SETATREF across pages

`Prop.P = SETATREF(Pages[A]!Sheet.1!Prop.X)` shows the target value; writing the cell writes through to the target and keeps the `SETATREF` formula. Verified for `FormulaU = '"w"'`, `FormulaU = '9'` and `ResultIU = 7`. Use it for "edit here, stored there" (one value, several views).

Chains: an edit is forwarded only through cells that hold `SETATREF` themselves (measured, Visio 2019, COM writes `FormulaU`/`ResultIU` on `Prop.P`, text and number rows):

| `Prop.P` | `User.L` | Edit of `Prop.P` |
|---|---|---|
| `SETATREF(User.L)` | `Pages[A]!Sheet.1!Prop.X` | lands in `User.L` as a constant: the link is gone, target unchanged |
| `SETATREF(User.L)` | `SETATREF(Pages[A]!Sheet.1!Prop.X)` | lands in the target; both formulas stay |
| `SETATREF(User.L)` | `GUARD(Pages[A]!Sheet.1!Prop.X)` | error `Cell is guarded`, nothing changed |
| `SETATREF(User.L)` | `GUARD(SETATREF(Pages[A]!Sheet.1!Prop.X))` | lands in the target; both formulas stay |

- One Undo takes back the forwarded write (target value restored, formulas kept).
- A user edit in the Shape Data window behaves the same (`SETATREF` link cell; text and number row; one Undo step each; [visio-shape-data-window-test](visio-shape-data-window-test.md)).
- Shape Data window on a source `Prop.X = SETATREF(Pages[A]!Sheet.1!Prop.X)` (Visio 2019 build 20527, measured):
  - List choice (fixed or variable list) lands in the target as `INDEX(n,Pages[<source page>]!Sheet.n!Prop.X.Format)`: a position in the **source's** list. A value typed into a variable list that is not in it rewrites the source's `Format` to a constant (`";a;b;c;zz"`); the target gets `INDEX(4,...)`. Make the target's value text before you repair the source's `Format`, else the position points past the end (`""`).
  - Cleared number row (empty formula): target cleared **and** the source's own formula emptied: link lost. Cleared text row: `""` lands in the target, link kept.
  - Every forwarded edit (also a COM write into the target, Undo, Redo) fires `CellChanged` for both cells, target first. With two shapes selected (one linked, one not), one choice writes both.
- `SETATREF(<ref>,SETATREFEVAL(SETATREFEXPR("")))` stores the edit in the source's own formula (`SETATREFEXPR(INDEX(2,Prop.X.Format))`) and pushes the evaluated value: a list choice lands as text (`"b"`). But a typed value not in a variable list lands as `""` (value lost), and a cleared number row still loses the link (target unchanged). Plain `SETATREF` plus a handler that makes the target text is the safer choice.
- Copy keeps the link: a pasted source (same or other page) and a duplicated page still point to the old target (same page reads back `SETATREF(<target NameU>!Prop.X)`). Reset links on copy in code.
- Code that rewrites a `SETATREF` cell itself (new target, constant, or back to the master) must use `FormulaForceU` or `Shape.SetFormulas` with `visSetBlastGuards` (2): that replaces the cell's own formula and forwards nothing. Plain `FormulaU` and `SetFormulas` without flag 2 always forward (measured on a master instance with local `User.L = SETATREF(Sheet.1!Prop.X)`):

  | Write to `User.L` | `FormulaU` / `SetFormulas` 0 or 8 | `FormulaForceU` / `SetFormulas` 2 or 10 |
  |---|---|---|
  | `SETATREF(Sheet.1!Prop.Y)` | `Prop.X` becomes `Prop.Y`; `User.L` unchanged | `User.L` = `SETATREF(Sheet.1!Prop.Y)`; `Prop.X` unchanged |
  | `"t"` | `Prop.X` becomes `"t"`; `User.L` unchanged | `User.L` = `"t"` |
  | `""` | `Prop.X` emptied **and** `User.L` back to the master | `User.L` back to the master; `Prop.X` unchanged |

  Force also skips `GUARD`: check for it before writing. Writing `GUARD(<own SETATREF formula>)` with `FormulaU` forwards it into the target: `Circular formula reference.`
- The target may be a `Prompt` cell: `Prop.P = SETATREF(User.U.Prompt)` writes the edit into the Prompt (COM `FormulaU`; one Undo step in a scope), so `User.U` can derive its value from it, e.g. a code `INDEX(0,User.U.Prompt," ")` from a chosen `S1 Cabinet` (Visio 2019, 2026-10-31).
- Pattern: one link cell holds `SETATREF(<target>)`; cells that should be editable use `SETATREF(<link cell>)`, read-only cells a plain `<link cell>` reference.

With that pattern (measured the same way, three pages):

- Chain over two shapes (A links to B's cell, B's cell links to C): an edit in A lands in C, all formulas stay. A broken middle link (constant in B's link cell) makes A show B's constant.
- Target cell holds a formula (`Prop.W&"A"`): the edit replaces it with a constant; Undo restores it. Target `GUARD(...)`: `Cell is guarded`, nothing changed.
- Target deleted: the link cell becomes a constant (the `SETATREF` is gone too); a later edit stays in the link cell.
- Cycle (S takes M's cell, M takes S's cell): no error, no hang, nothing reports it. Plain references freeze at the last value. With `SETATREF` on both sides an edit of S ends in M's cell as a constant (M's `SETATREF` formula is replaced), which breaks the cycle.

## CONTAINERSHEETREF

```text
User.Loc = IFERROR(CONTAINERSHEETREF(1)!Prop.Loc, "none")
User.Loc = IFERROR(CONTAINERSHEETREF(1, "Cabinet")!Prop.Loc, "none")   ' container's User.msvShapeCategories = "Cabinet"
```

Measured 2026-10-30 unless noted, Visio 2019 C2R x64, invisible Visio by COM, members added by `ContainerProperties.AddMember`.

- Container shape: any shape with `User.msvStructureType = "Container"` is one (`ContainerProperties` not null).

- Works in Visio 2019: value follows the container live; not a member -> `IFERROR` fallback.
- Membership is not geometry. What sets it (measured 2026-11-12, Visio 2019 build 20527, own visible Visio, real mouse and keyboard; container = rectangle with `User.msvStructureType = "Container"` + `User.msvShapeCategories`):
  - Makes a member: `AddMember`; UI drag of a shape into the container; drop of a master from the stencil window into it; Ctrl+C/Ctrl+V, Ctrl+D, Ctrl+X/Ctrl+V and Ctrl+drag of a member (copy lands inside, offset); Ctrl+C/Ctrl+V of the selected container copies it with its members, which are members of the copy; `Page.DropContainer(master, selection)` wraps the selection.
  - Ends membership: UI drag out; moving out by code; UI Delete of the container deletes its members too (Undo brings all back).
  - Does **not** make a member (shape inside, value from the fallback): `Page.Drop` (master or shape) inside, `Selection.Move` in by code, `Window.Copy`/`Window.Paste`, `Window.Duplicate`, `Selection.Duplicate` (measured 2026-11-12, also in a visible Visio), `Window.Cut`/`Paste` by API (a selected container is then copied without members); container dropped from the stencil, moved (mouse) or resized (mouse handle or `Height`) over free shapes. `Window.Delete` of a selected container by API deletes the container only.
- List container (`User.msvStructureType = "List"`, `msvSDListDirection` 0, `msvSDListSpacing` 0, `msvSDListAlignment` 1; measured 2026-11-12, Visio 2019 build 20527, COM): `ContainerProperties.InsertListMember(shape, pos)` lines members up edge to edge, middles in line, any widths/heights; `LISTORDER()` is live and changes for all after an insert. As with containers, placement by code makes no member (shape moved onto the list end, shape placed by glue code next to the last member). A member moved away by code leaves the list and its gap stays open. The list alone moved by `Selection.Move` loses all members (Undo brings them back). Cost out of process: 200 `InsertListMember` 14 s; insert at position 1 with 200 members 0.85 s, its Undo 0.4 s. Not measured: mouse.
- Nested by UI: dragging container I (in front of O) into O makes I a member of O, and I's members become members of **both** I and O; then index 1 = I (front). Same with `AddMember` of the member to O.
- Take in the shapes lying inside a container (membership by geometry, e.g. an action after the container was placed over shapes): `container.SpatialNeighbors(visSpatialContain, 0, visSpatialIncludeContainerShapes)` (2, 0, 128) gives the top-level shapes wholly inside, also a connector between two of them; not a shape crossing the edge, not a connector going out. Without flag 128 an inner container is left out. Same result with the container at the back, in the middle, in front, filled or not. Then `ContainerProperties.AddMember(selection, visMemberAddDoNotExpand)`; filter existing members first. `visSpatialOverlap` (1) gives only shapes crossing the edge (plus a big shape around, e.g. a page frame); `Selection.AddToContainers` of those added the crossing one. Measured 2026-11-12 (Visio 2019 build 20527, invisible, COM).
- Z-order by `AddMember`: a plain shape as member leaves the z-order as it is (member may stay behind the container); a **container** as member (alone or in a selection) sends the receiving container to the very back of the page, behind every shape (same measurement). To keep a page frame at the back, send it to back again afterwards.
- Category match is **case-sensitive**: `CONTAINERSHEETREF(1, "MyCat")` does not find a container with category `MYCAT`; a category list (`"Cabinet;MyCat"`) matches each entry.
- **Nesting is not passed on**: a shape in container I, I in container O, is member of I only (`MemberOfContainers` = I, `CONTAINERCOUNT()` = 1, `CONTAINERSHEETREF(2)` -> fallback). `CONTAINERSHEETREF(1, "<O's category>")` finds nothing. To inherit over levels, each container reads its own container with the same formula (chain: `User.Loc = <parent via CONTAINERSHEETREF or ThePage> & ";" & own code` works over two levels, also after save + reopen); a container without that formula cuts the chain only for a shape that is member of it alone (by code); a member of both (UI drag) still finds the categorized container: `CONTAINERSHEETREF(1, "<cat>")` skips containers without the category, whatever their z-order (measured 2026-11-12).
- Member of two overlapping containers: index 1 = the one on top in z-order; `BringToFront` of the other switches it live. So in nested containers (member of all) `CONTAINERSHEETREF(1)` reads the **outer** one once it is brought to front. Index order is not reliably z-order: after `RemoveMember`/`AddMember` of an inner container by code the member's list read `R,Q,T` (z-order `Q<R<T`, T on top), `MemberOfContainers` `Q,R,T` (measured 2026-11-12, Visio 2019 build 20527, invisible, COM).
- Innermost of nested containers, whatever the z-order: each container carries its depth, the member picks the deepest of the first k indexes. Index may be a cell reference (works, Visio 2019 build 20527). Recipe (k = 4; `d(i)` = `IFERROR(CONTAINERSHEETREF(i,"MyArea")!User.Depth,0)`):

  ```text
  container: User.Depth = 1+MAX(d(1),d(2),d(3),d(4))
  both:      User.Pick  = IF(AND(d(1)>=d(2),d(1)>=d(3),d(1)>=d(4)),1,IF(AND(d(2)>=d(3),d(2)>=d(4)),2,IF(d(3)>=d(4),3,4)))
  read:      IF(User.Pick=1,IFERROR(CONTAINERSHEETREF(1,"MyArea")!User.X,ThePage!User.X),IF(User.Pick=2,...,IFERROR(CONTAINERSHEETREF(4,"MyArea")!User.X,ThePage!User.X)))
  ```

  Measured 2026-11-12 (same setup; 3 nested containers, member of all, all 6 z-orders, code change, Undo, save + reopen): right in all. Use the fixed-index `IF` chain, not `CONTAINERSHEETREF(User.Pick,...)`: when the member's index order changes without its own membership changing (inner container removed from and added again to the outer ones by code), the computed index kept the old container's value until the formula was written again; the `IF` chain depends on all k containers and stayed right. Speed, 1,000 members in 3 nested containers, invisible, 3 rounds interleaved (plain `CONTAINERSHEETREF(1)` / computed index / `IF` chain): outer code change 166-206 / 173-188 / 183-250 ms, Undo 91-94 / 96-97 / 95-173 ms, outer `BringToFront` 42-46 / 486-527 / 492-538 ms, `SendToBack` 107-110 / 539-555 / 553-639 ms.
- **Stale values**: the dependency is made when the member's formula is evaluated. If the container gets the category or the cell only after `AddMember`, or its cell is deleted and added again, the member keeps the old value (no error) until it recalculates itself: change of one of its own input cells, of membership or of z-order. A deleted container cell gives no `REF()`; Visio stores the unresolved name upper case (`CONTAINERSHEETREF(1)!Prop.LOC`) and resolves it later.
- Speed: changing the container cell recalculates all members, ~0.1 ms each (1,024 members 116-124 ms, 8,192 members 974-1,008 ms; `ScreenUpdating` 0). `AddMember` of a `Selection`: 1,024 shapes 390 ms, 8,192 shapes 5,978 ms. Visible Visio, `ScreenUpdating` on, no event sink (2026-11-12): 1,000 members with one formula: container cell change 24-25 ms, its Undo 29-34 ms, `AddMember` 352 ms; with an out-of-process `CellChanged` sink 2,054 ms (the events cost the time).

## ThePage and missing page cells

Measured 2026-10-30, same setup.

- `ThePage!User.X` where the page has no `User.X`: the write throws `#NAME?`, also inside `IFERROR(...)`.
- Page cell deleted (measured 2026-10-30 and 2026-11-02, Visio 2019 C2R x64 build 20527): adding the cell again never brings the dependency back by itself.
  - **Local formula** (shape without master, or a cell overridden in the instance): Visio rewrites it for good. Seen: the reference became `REF()` (`IF(LEN(IFERROR(REF(),""))>0,REF(),...)`), and in another formula (`IF(STRSAME(...),ThePage!User.X,INDEX(...))`) the whole formula became a constant of the last value. The original text is lost; only code that knows it can write it again.
  - **Inherited formula** (master instance): the formula text stays unchanged (`Cell.IsInherited` true, no `REF()`), but the value freezes: page cell added again and changed, the instance keeps the old value. Writing `""` (`FormulaU` or `FormulaForceU`) or rewriting an input cell recalculates it once, then it is frozen again. Fixes that make it follow again: save + reopen the document (all instances), writing the formula in the master again (`Master.Open()`, same `FormulaU`, `Close()`: all instances of that master), or a local copy of the formula in the instance (loses inheritance). A new instance dropped after the cell is back follows normally.
- Frozen inherited chain, no cell deleted (seen once 2026-11-09 in a user's drawing, Visio 2019 C2R x64 build 20527; origin unknown, not reproduced by COM or by hand): instance cells `LineColor = GUARD(RGB(INDEX(0,User.Look,","),...))` and `User.Look = IF(TheDoc!User.Show,...)`, both inherited. Changing `TheDoc!User.Show` recalculated `User.Look`, but not `LineColor`; other inherited chains of the same shape (`User.Index` -> `LineWeight`) worked, a second instance of the same master on the page worked. `Cell.Precedents`/`Dependents` were the same as in the working instance (`User.Look` lists `LineColor`). Stayed frozen after save + reopen; in the file the instance had `<Cell N='LineColor' V='#ff00ff' F='Inh'/>` (stale value cached). No effect: reading `User.Look` first, `Cell.Trigger` on `User.Look`, writing an input cell of `User.Look` with the same value, a local `LineColor` with the same formula after an Undo. `Cell.Trigger` on `LineColor` recalculates it once, then it is frozen again. Fixes that made it follow again: `User.Look` changing its value while one of its own-shape inputs is local (e.g. a local `Prop` cell set, then the document switch flipped so the result changes), any new local row in the `User` section (`AddNamedRow` + `User.Probe = 1`), or a local `User.Look`. Similar community reports, none explained: [visguy 5644](https://visguy.com/vgforum/index.php?topic=5644.0) (new master User row not evaluated in instances until edited once; replies suggest `DEPENDSON`, not tried here), [bVisual 2021-10-20](https://bvisual.net/2021/10/20/forcing-display-update-of-inherited-data-values/) (inherited `SETATREF(TheDoc!..)` shown stale).
- Document cell deleted (measured 2026-11-02, same setup): a formula reading `TheDoc!User.X` in a master of the document stencil becomes a constant of its last value; all instances inherit the constant. Adding the cell again changes nothing; write the master formula again (`Master.Open()` ... `Close()`), the instances then follow.
- Shape with `ThePage!User.X` copied (`Page.Drop(shape, x, y)`) onto a page without `User.X`: no error, shows the source page's last value (also after save + reopen), follows that page's cell once it is added.
- **Master** whose formulas read `TheDoc!User.L` and `ThePage!User.X`, dropped (`Page.Drop(master, x, y)`) into a document without these cells (measured 2026-10-31, Visio 2019 C2R x64 build 20527, with and without `EventsEnabled`): Visio adds both itself. The document cell gets the **source document's** value, also its `Prompt` when the master reads `TheDoc!User.L.Prompt` (measured with a 105-char Prompt); a target document that has the cell keeps its own value and the instance uses it; the page gets a bare `User.X` row with the master page's value (not the source page's), no other rows. Same for `ThePage!Prop.X` (measured 2026-11-05): the page gets a `Prop.X` row; label not checked (the master page's row had none), so code that wants a label writes it. One Undo removes the page row (and the master), the document cell stays. So a stencil with such masters must hold the document cells with neutral values, or each drop seeds them into the drawing; code that adds the page cells on drop must not take a bare `User.X` row as "already set up".
- **Master copied into another document** (`Document.Masters.Drop(master, 0, 0)`, e.g. a drawing's document master into a new document saved as `.vssx`; measured 2026-11-12, Visio 2019 C2R x64 build 20527): Visio adds the document `User` rows too, values from the source document; a row the target has keeps its value. It may add more rows than the master reads (seen: 16 rows for a master reading 6; on `Page.Drop` from a script-made stencil all 16 rows of that stencil, from a stencil made by `Masters.Drop` only the 6 read ones); the rule is unknown, so do not count on extra rows. Dropping the same master again adds `Name.<n>`, it does not replace.
- New pages and page rows (measured 2026-10-31, Visio 2019 C2R x64): `Pages.Add()` (also with `Background = 1`) and the UI command `DoCmd(2361)` (`visCmdNewForegroundPage`) copy none of the active page's `User`/`Prop` rows. `Page.Duplicate()` and `DoCmd(2383)` (`visCmdDuplicatePage`) copy rows, formulas and values. So in `PageAdded`: rows missing = new page, rows present = duplicated page. Code that relies on a page cell must add it to new pages.
- Gotcha: `DoCmd(2202)` (`visCmdInsertPageTab`) from an out-of-process script hung twice (Visio blocked, COM calls of a second client returned nothing; process killed). Use 2361 to insert a page by command.
- `TheDoc!User.X` behaves like `ThePage!`: writing a formula (seen for `Prop.X.Format = TheDoc!User.X`) fails with `#NAME?` while the document cell is missing. Add the document cell first.
- Speed: one page cell change recalculates all dependent shapes on the page, ~0.1 ms each (1,024 shapes 96-118 ms, 8,192 shapes 824-952 ms, plain `ThePage!User.X` or a nested `IF` of 5 sources alike; invisible Visio, `ScreenUpdating` 0).

Events for these changes (glue, container): [visio-glue-events](visio-glue-events.md).
