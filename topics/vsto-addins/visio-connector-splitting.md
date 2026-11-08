---
title: Visio connector splitting - cells, mouse only, glue to the whole shape
description: Read when a shape dropped onto a connector should split it (e.g. insert a device into a wire), or splitting must be prevented - cells, what triggers it, where the halves glue.
---

# Visio connector splitting

A 2D shape dropped onto a connector splits it into two connectors, one on each side of the shape. Visio allows it only if all four levels allow it ([PageShapeSplit docs](https://learn.microsoft.com/en-us/office/client-developer/visio/pageshapesplit-cell-page-layout-section)):

| Level | Setting | On | Default |
|---|---|---|---|
| Application | `Application.Settings.ConnectorSplittingEnabled` (per-user registry: [visio-application-settings](visio-application-settings.md)) | - | `True` |
| Page | `PageShapeSplit` (Page Layout) | page | `1` |
| Splitting shape | `ShapeSplit` (Shape Layout): "can split shapes that are splittable" | the 2D shape that is dropped | drawn rectangle: `0` |
| Split shape | `ShapeSplittable` (Shape Layout): "this 1-D shape can be split" | the connector | `Shape.AutoConnect` connector: `1` |

To rule splitting out for your own shapes regardless of the user's option, set `ShapeSplit = 0` or `GlueType = 8` on your 2D masters, `ShapeSplittable = 0` on your connectors, or `PageShapeSplit = 0` in the template pages.

Measured 2026-11, Visio 2019 C2R x64 build 20527, new blank metric drawing, two rectangles joined by `Shape.AutoConnect`, a 20 x 8 mm rectangle with two connection points (left/right middle, or top/bottom middle for a vertical connector) and `ShapeSplit = 1`:

## Mouse only

- By code nothing splits, even with all four levels on: `Page.Drop(shape, x, y)` onto the connector, or moving a shape there by `PinX`/`PinY`.
- A real mouse drag of the shape onto the connector splits it: 2 connectors (`Dynamic connector` + a new `Dynamic connector.<n>`). With `ShapeSplit = 0` on the shape, or `ShapeSplittable = 0` on the connector, the same drag splits nothing.
- Connector from an own master (the document's default connector, [visio-default-connector](visio-default-connector.md)): both halves are instances of that master and keep the old connector's local cells (a local `ShapeSplittable = 1` was copied to the new half).
- _Unverified:_ drop from a stencil window by mouse (only an existing shape was dragged); whether local Shape Data values are copied too.

## Halves glue to the whole shape, not to connection points

- Both halves are glued with dynamic (walking) glue to the shape: `Connect.ToCell` = `PinX`, `BeginX`/`EndX` = `_WALKGLUE(...)`. No `Connections.X<n>` glue.
- Right after the split the ends sit exactly on the two connection points.
- After the shape moved (+10/+10 mm by `PinX`/`PinY`), Visio reroutes: one end stayed on its connection point, the other walked to the middle of another side of the outline (horizontal case: bottom middle; vertical case: left middle).
- `GlueType = 8` (`visGlueTypeNoWalkingTo`) on the dropped shape blocks splitting completely: the same mouse drag left 1 connector, unglued from the shape, whatever the connector had (`GlueType` 0 or 4, `ConFixedCode` 0 or 2; horizontal and vertical). Splitting needs dynamic glue to the shape; a shape that allows glue only to its connection points cannot split connectors with Visio's own feature.
- So the information "which end on which connection point" is not kept. If it matters (e.g. a device inserted into a wire), re-glue each end to its connection point after the split (`Cell.GlueTo(shape.CellsSRC(visSectionConnectionPts, row, 0))`; _unverified_ for this case), e.g. in a `ConnectionsAdded` handler ([visio-glue-events](visio-glue-events.md)).
