---
title: Visio cross-shape references - pages, delete, copy, SETATREF, containers
description: "Read when a ShapeSheet formula reads another shape (Sheet.n!, Pages[..]!, SETATREF, CONTAINERSHEETREF) and must survive page rename, delete, cut/paste, copy or another document."
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
| Target cut + pasted (same or other page, new ID) | **constant** with the last value |
| Referencing shape copied alone, cross-page ref | unchanged: copy points to the original target |
| Referencing shape duplicated (`Duplicate`) | unchanged |
| Referencing shape copied alone, same-page ref (`Sheet.n!`) | `REF()`; shows the old value until the cell recalculates, then `#REF!` (`Cell.Error` 55) |
| Referencing shape + target copied together (same or other page) | rewritten to the copied target |
| Referencing shape pasted into another document | `REF()` => `#REF!` |

- Nothing errors and nothing reports the change: a lost link looks like a normal value. An add-in that relies on links must store the target's identity (e.g. a GUID cell) next to the formula to detect and repair them.
- Glue (`BeginX = PAR(PNT(Sheet.n!Connections.X1.X, ...))`) behaves like a same-page reference: a 1D shape copied alone is pasted unglued.

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
- `FormulaForceU` on `Prop.P` replaces `Prop.P`'s own `SETATREF` formula; nothing is forwarded.
- Pattern: one link cell holds `SETATREF(<target>)`; cells that should be editable use `SETATREF(<link cell>)`, read-only cells a plain `<link cell>` reference.

## CONTAINERSHEETREF

```text
User.Loc = IFERROR(CONTAINERSHEETREF(1)!Prop.Loc, "none")
User.Loc = IFERROR(CONTAINERSHEETREF(1, "Cabinet")!Prop.Loc, "none")   ' container's User.msvShapeCategories = "Cabinet"
```

- Works in Visio 2019: value follows the container live; not a member -> `IFERROR` fallback.
- Membership is not geometry: `ContainerProperties.AddMember` makes a member; moving it out by code removes it; moving it back in by code does not add it again (UI drag: _unverified_).
- Visio stores an unresolved name upper case in the formula (`CONTAINERSHEETREF(1)!Prop.LOC`); the lookup still works.
- Nested containers: the first match is by z-order, not nesting (_unverified_, from a design note).

Events for these changes (glue, container): [visio-glue-events](visio-glue-events.md).
