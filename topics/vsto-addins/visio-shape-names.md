---
title: Visio shape names of copies
description: Read when Visio code compares Shape.Name/NameU, e.g. to find shapes of the same kind - the .<number> suffix of copies, when it is not the ID, names that lose it.
---

# Visio shape names of copies

Visio keeps shape names unique per page (or master) by adding `.<number>` to a copy. To find shapes "of the same kind" by name, strip a trailing `.<digits>`:

```csharp
static readonly Regex CopyNumber = new(@"\.\d+$");
static string BaseName(string name) => CopyNumber.Replace(name, "");   // "Relay.17" -> "Relay"
```

| Action | Result |
|---|---|
| New shape without a name | `Sheet.<ID>` |
| `shape.Name = "Relay"` by code | `Name` and `NameU` both `Relay` |
| `Duplicate()` of `Relay` or `Relay.2` | `Relay.<new ID>` (base kept, number replaced) |
| `Duplicate()` of `Fuse 1.5` (name ends in `.<digits>`) | `Sheet.<new ID>`: the name is lost |
| Copy + `Page.Paste()` to another page (name free there) | name kept: `Relay.3` gets ID 4, `Relay` stays `Relay` |

## Gotchas

- **The suffix is not always the ID.** After a paste to another page it is the ID from the source page. Do not strip only `"." + shape.ID`.
- Stripping `.<digits>` also cuts a name that ends in digits after a dot (`Fuse 1.5` -> `Fuse 1`). Visio treats such names the same way (row above).
- All unnamed shapes, groups included, have the base name `Sheet`.
- `Name` is the local name, `NameU` the universal one; in a localized Visio, built-in shapes differ (`Rechteck.5` / `Rectangle.5`). Pick one and compare only that.
- Shapes next to a shape: `shape.ContainingShape.Shapes`. For a top-level shape `ContainingShape` is the page sheet `ThePage` (of the page, or of the master in a master), and its `Shapes` are the top-level shapes; in a group it is the group.

Verified with Visio Pro 2019 C2R x64 (English UI) by COM automation (2026-10). _Unverified:_ paste onto a page where the name is already taken; names of master instances.
