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

- Lookup: local `Name` `Dynamic connector` first, then `NameU`. So a built-in connector that gets into the drawing later (paste from another drawing; a drop of `ConnectorToolDataObject`, below) arrives as Name `Dynamic connector`, NameU `Dynamic connector.<n>` and may take over. _Unverified_ for paste and for the Connector tool.
- Keep the local `Name` free (e.g. your product name); only `NameU` must be `Dynamic connector`.
- `Application.ConnectorToolDataObject` is read-only and returns Visio's built-in connector in every case above; `Page.Drop` of it adds a new master `Dynamic connector` / `Dynamic connector.<n>` next to yours. It does not tell which master the tool will use.
- _Unverified:_ non-English UI (the built-in master has a localized `Name`, e.g. German `Dynamischer Verbinder`); a connector master selected in a stencil window before using the Connector tool; the blue AutoConnect arrows clicked by mouse.

Test: draw two rectangles, `DoCmd(1225)` (`visCmdDRConnectorTool`), drag by real mouse from one to the other, `DoCmd(1219)` (pointer tool), then read `Master.NameU` of the new 1-D shape. Mouse helpers: [visio-connector-splitting](visio-connector-splitting.md) used the same approach.
