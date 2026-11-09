---
title: Diagram kinds in CAE tools
description: Read when deciding which diagram kinds/disciplines a tool supports and how one device appears in several diagrams - EPLAN, Zuken E3, Aucotec EB, COMOS, Visio; line data compared across areas.
---

# Diagram kinds in CAE tools

## Grouping

- EPLAN: Electric P8, Fluid, Preplanning/P&ID, Pro Panel on one platform. Page types decide the representation: e.g. `Schematic multi-line`, `Schematic single-line`, `Fluid power schematic`, `Panel layout`, `Topology`, `Graphic` (help 2.8/2.9 page type and cross-reference pages; full lists not read). Fluid covers hydraulics, pneumatics, lubrication, cooling, cooling lubricant, gas engineering ([fluid-power](fluid-power.md)).
- Zuken E3.series: E3.fluid for hydraulic, pneumatic, cooling and lubrication schematics, standalone or mixed with electric in one project and database; one device in pneumatic schematic, installation diagrams and manifold layouts; hose/tube sizing checks (zuken.com product pages, vendor claims).
- Aucotec Engineering Base, Siemens COMOS: plant engineering, one data model for process (P&ID), electrical and automation; P&ID is the start, circuit diagrams and tag sheets generated from typicals (vendor pages, marketing level; object model not read).
- Visio: separate templates per area (P&ID, Fluid Power, HVAC, Plumbing, electrical), no link between diagrams.

## One device, several representations

- EPLAN: a function is created once (device navigator) and placed several times; representation (single-line, multi-line) follows the page type; properties synchronized across single-line, multi-line and overview pages (help 2.9 `singlepole_k_hintergrund`). Panel layout parts cross-reference their main function in the schematic both ways.
- So overview, single-line and plant overview are not own disciplines: they are further representations of a device that has its main representation in the schematic.
- Typical split of one machine part: pump body in P&ID or fluid diagram, its motor in the electric diagram, the real part in the layout; a process valve in the P&ID, its actuator air in the fluid diagram, its solenoid coil in the electric diagram.

## Line data across areas

| Meaning | Electric wire | Fluid line (ISO 1219-2, EPLAN) | Process pipe (ISO 10628-1, DEXPI, Visio) |
|---|---|---|---|
| identification | wire number / connection designation | optional line number (`2-Ø20`) | pipeline number, `SegmentNumber` |
| size | cross-section mm² or AWG | nominal size / outer diameter x wall | nominal diameter (`DN 25`), `LineSize` |
| type, class | wire type (e.g. HD 361 `H07V-K`) | pipe vs hose, hose type | piping class, schedule, material |
| color | IEC 60757 codes | line kind color (EPLAN) | - |
| medium | potential, signal | trade H/P/C/K/L/G, derived from the ends | `FluidCode` |
| length, remark | yes | length (conduits) | description |
| process values | - | pressure level index (optional code) | design pressure, temperature, insulation, flow direction |

- Shared by all: identification, size, type/class, length, remark. Area-specific: color, medium, pressure, temperature, insulation, flow direction.
- Kind of line (decides look and valid ends): electric wire, cable, bridge, function line; fluid pressure, control/pilot, drain, return, working line; process main/secondary pipe, utility line, signal lines (electric, pneumatic, data, capillary).
