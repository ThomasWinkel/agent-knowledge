---
title: Visio SpatialNeighbors and SpatialRelation - touching 2D shapes
description: Read when finding which shapes touch another (2D next to 2D, or a 2D mark across connectors; no glue), choosing the tolerance, or re-checking touch after a move or glue.
---

# Visio touching shapes: SpatialNeighbors, SpatialRelation

Visio glues only 1D to 2D. For two 2D shapes that should belong together when placed side by side (e.g. a small shape whose line ends on the bar of another), find the partner by geometry. Measured 2026-11, Visio 2019 C2R x64 build 20527: shape A = vertical bar line (open geometry, `NoFill`), shape B = horizontal line ending at the bar.

```csharp
// visSpatialTouching = 8; tolerance in inches (internal units).
Visio.Selection hits = shapeB.SpatialNeighbors[8, 0.01 / 25.4, 0];
short rel = shapeB.SpatialRelation[shapeA, 0.01 / 25.4, 0];   // 8 = touching, 0 = none
```

## What counts

- Line end exactly on the bar: `Touching` (8) with tolerance 0. Gap 0.001 mm: nothing with tolerance 0.
- Tolerance t finds gaps below t: t = 0.01 mm finds 0.005 mm, not 0.01; t = 0.1 mm finds 0.05, not 0.1.
- Line crossing the bar (0.01 to 4.9 mm past it): still `Touching`, never `Overlap` (4). For open lines, ask for `Touching` only.
- Line exactly at an end of the bar touches; 0.05 mm beyond only with t = 0.1 mm.
- Only geometry counts: text blocks do not, nor bounding boxes that touch without geometry.
- Rotated (90 deg) and flipped (`FlipX`) shapes: found as expected.
- `SpatialNeighbors` returns every touching top-level shape: also a connector glued to the shape, other shapes along the same bar. Filter by an own cell (e.g. a `User` row naming the shape's role) and treat several hits as ambiguous.

## 2D shape over connectors (mark across wires)

Measured 2026-11, same build: open 2D shape (`NoFill` line, Z polyline, oval) laid across dynamic connectors (right-angle, open or glued ends).

- Connector crossing the shape's line: `Touching` at tolerance 0, never `Overlap`; also collinear with a segment, crossing on any segment of a bent route, and with the shape turned 90 deg. A connector ending on the line touches (tolerance rules as above).
- Oval with `NoFill` (solid or dashed line alike): finds only connectors crossing or touching its outline; one lying fully inside: nothing. Filled oval: the inside one is `ContainedIn` (from the connector's side `Overlap`); crossing ones stay `Touching`.
- A connector glued to a connection point of the shape also touches it: exclude it via `Shape.FromConnects` if it must not count.
- Cost in process: `SpatialNeighbors` + 2 cell reads per hit to filter: 7-11 ms for 40 connectors under one shape.

## Moves and cost

- `CellChanged` with `SetFilterSRC` on section 1 (object), row 1 (transform), cells 0..8 (`PinX` .. `FlipY`): a move fires `PinX`/`PinY` per moved shape, plus `Width`, `PinX`, `LocPinX` per rerouted connector; height change `Height`, `LocPinY`; Rotate90 `PinX`, `PinY`, `Angle`; flip `FlipX`. Duplicate, delete, drop: none (use `ShapeAdded`, `BeforeShapeDelete`). Undo/Redo: same events, `IsUndoingOrRedoing` true.
- A shape dropped onto others (`Page.Drop` of a master): `ShapeAdded` already sees its touching shapes. `BeforeShapeDelete` still sees them; Undo of the delete: `ShapeAdded` sees them. Duplicate of several shapes: the copies' transform events come before their `ShapeAdded`; at `ShapeAdded` all copies are placed.
- A connector's own route is **not** final in `ConnectionsAdded`: the reroute's `Width`, `PinX`, `PinY`, `Height` ... events of that connector follow in the same action, and the touch changes between them ([glue events](visio-glue-events.md)). Check a newly glued connector at those events, not only in `ConnectionsAdded`.
- A shape that moves away fires; the shapes it leaves behind do not. Re-check the partners it had before (keep them in your own index), not only the ones touching now.
- In process, several shapes moved in one action (`Selection.Move`, also its Undo/Redo): the first event already sees all shapes at their new place. So a handler can decide at once and write in the user's Undo step ([visio-cell-change-events](visio-cell-change-events.md)).
- Cost: `SpatialNeighbors` inside a `CellChanged` handler (VSTO, Debug) 0.3-0.9 ms, first call of an action 2-6 ms. Out of process 11-14 ms per call, alike on pages with 21, 1,001 and 6,001 shapes.

_Unverified:_ mouse drag with snap (whether a shape placed by hand ends exactly on the other's line), connector drawn by hand with the Connector tool (event order), shapes inside groups.
