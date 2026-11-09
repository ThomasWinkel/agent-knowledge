---
title: Visio default connector - own master for the Connector tool and AutoConnect
description: Read when drawn connectors (Connector tool, AutoConnect) should be your own connector master instead of Visio's Dynamic connector - NameU rule, lookup order, ConnectorToolDataObject.
---

# Visio default connector

Visio's Connector tool and `Shape.AutoConnect` (without a connector argument) do not always use the built-in Dynamic connector: a **document master whose `NameU` is `Dynamic connector`** takes its place. Put your connector master into the template that way and every drawn connector is an instance of it.

Measured 2026-11, Visio 2019 C2R x64 build 20527, English UI; masters made by `Document.Masters.Drop(connectorShape, 0, 0)`, then `NameU` and `Name` set.

| Document masters | `Shape.AutoConnect(other, 0)` | Connector tool (real mouse drag) |
|---|---|---|
| none | built-in, adds master `Dynamic connector` | built-in |
| own: Name + NameU `Dynamic connector` | own | _unverified_ |
| own: Name `MyConnector`, NameU `Dynamic connector` | own | own (drawing made from a `.vstx` holding it); no built-in master added |
| own: Name + NameU `MyConnector` | built-in | _unverified_ |
| own (Name `MyConnector`, NameU `Dynamic connector`) + another master Name `Dynamic connector`, NameU `Other` (a plain line) | takes the **other** one (Name match wins), then fails without message (`HRESULT -2032466954`), a line left unglued | _unverified_ |

- Connector tool drag from a connection point to a connection point (real mouse, shapes with `GlueType 8`, drawing from the template above): both ends glued **static** to the points (`BeginX = PAR(PNT(A!Connections.P1.X,A!Connections.P1.Y))`), for named ABCD rows and normal rows alike; the connector is the own master. Drag from shape interior to interior (no points, `GlueType 0`): dynamic glue to the shapes (`PinX`).
- Lookup: local `Name` `Dynamic connector` first, then `NameU`. So a built-in connector that gets into the drawing later (paste from another drawing; a drop of `ConnectorToolDataObject`, below) arrives as Name `Dynamic connector`, NameU `Dynamic connector.<n>` and may take over. _Unverified_ for paste and for the Connector tool.
- Keep the local `Name` free (e.g. your product name); only `NameU` must be `Dynamic connector`.
- `Application.ConnectorToolDataObject` is read-only and returns Visio's built-in connector in every case above; `Page.Drop` of it adds a new master `Dynamic connector` / `Dynamic connector.<n>` next to yours. It does not tell which master the tool will use.
- Switching a drawing's own connector on later (rename the document master's `NameU` to `Dynamic connector`): instances stay linked, but the master gets a new `UniqueID`, so a later drop from the stencil adds a second master `MyConnector.<n>` ([visio-master-versions](visio-master-versions.md)). Renaming back does not help (again a new `UniqueID`), nor does Undo (NameU back, `UniqueID` stays new); `Master.UniqueID` is read-only. Fix: set `MatchByName = true` on the document master in the same step: stencil drops then reuse it by its local `Name` (keep the local `Name` equal to the stencil master's name), also after renaming back and after save + reopen. Measured 2026-11-09, build 20527.
- With the built-in `Dynamic connector` master already in the drawing, the rename does not take: once `The name 'Dynamic connector' is already in use.`, usually silently the master became `Dynamic connector.5` (not the default then). Check `NameU` after the rename. Measured 2026-11-08 and 2026-11-09, build 20527.
- Connector tool with an own default master that has `GlueType 4` (real mouse, 2026-11-09): point to point and point to empty space work as above (static glue to the point). A drag that starts inside a shape (`GlueType 8`) draws no connector but moves the shape, same with the built-in connector.
- `AutoConnect` needs a default connector that allows walking glue: own master with `GlueType 4` (`visGlueTypeNoWalking`) -> fails (`HRESULT -2032466954`), connector left unglued; same master with `GlueType 0` or `2` -> ok. Measured 2026-11-09, build 20527, invisible Visio, shapes with connection points.
- `AutoConnect` always glues **dynamically to the whole shape** (`Connect.ToCell` = `PinX`, `ToPart 3`, `BeginX = _WALKGLUE(...)`), never to a connection point, even when the shapes have some. So it cannot make point-to-point connections.
- `Application.Settings.EnableAutoConnect` (the AutoConnect option in Visio Options) does not affect `Shape.AutoConnect` from code: identical results with False and True (same measurement).
- _Unverified:_ the blue AutoConnect arrows clicked by mouse (with `GlueType 4`, and whether they also glue to the whole shape); non-English UI (the built-in master has a localized `Name`, e.g. German `Dynamischer Verbinder`); a connector master selected in a stencil window before using the Connector tool.

Test: draw two rectangles, `DoCmd(1225)` (`visCmdDRConnectorTool`), drag by real mouse from one to the other, `DoCmd(1219)` (pointer tool), then read `Master.NameU` of the new 1-D shape. Mouse helpers: [visio-connector-splitting](visio-connector-splitting.md) used the same approach.
