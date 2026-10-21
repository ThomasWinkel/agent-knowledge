---
title: Visio layers with several layers per shape
description: Read when Visio code hides, shows or deletes shapes by layers (Layer.Delete, visibility cells) and shapes can be on more than one layer - which layer wins, what Delete(1) deletes.
---

# Visio layers with several layers per shape

A shape can be on any number of layers of its page (`Shape.LayerCount`, `Shape.Layer[i]`; `Layer.Add(shape, 0)`, `Layer.Remove(shape, 0)`). Visibility and deletion then follow rules that are easy to get wrong.

## Rules

- **Visible while any of its layers is visible.** Hiding one layer (`visLayerVisible` = 0) hides a shape only if all its other layers are hidden too. To hide shapes by a layer, they must be on no other visible layer.
- **`Layer.Delete(1)` deletes only shapes on no other layer**, of any kind ([Layer.Delete](https://learn.microsoft.com/en-us/office/vba/api/visio.layer.delete), Remarks). Shapes also on another layer just lose this layer. To delete by your own rule (e.g. "on no other layer of my kind"), delete the shapes yourself, then `Layer.Delete(0)`.
- Layers come in without code: _unverified_ Visio puts connectors drawn with the Connector tool on a layer `Connector`, and masters can bring layers along. So "on no other layer" is rarely true for shapes of a real drawing.

## Gotchas

- Removing shapes from all other layers makes hiding by layer reliable, but loses the user's layer assignments for good.

Visible rule verified by a manual test in Visio Pro 2019 C2R x64 (2026-10); `Layer.Delete` from Microsoft docs. _Unverified:_ printing (`visLayerPrint`) and locking (`visLayerLock`) with several layers; which layer color wins (`visLayerColor`) on two colored layers.
