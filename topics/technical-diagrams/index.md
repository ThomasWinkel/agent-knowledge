---
title: Technical diagram kinds
description: "Kinds of technical schematics (electric, fluid power, process/P&ID, layout): standards, minimum content, line and port conventions, how CAE tools group them and link one device across diagrams."
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# Technical diagram kinds

Map of the schematic kinds a CAE/diagram tool may support beyond electric: which standards govern them, what a line or port carries there, and how commercial tools group them. Read before designing discipline lists, connector/line data models or symbol libraries that should not stay electric-only. Standards are paid; facts come from public previews, tables of contents and vendor help, each file names its sources.

## Key facts

- Two independent axes: technical area (electric, fluid power, process piping, building services, ...) and document kind (function-oriented schematic, overview/single-line, location-oriented layout, connection-oriented lists/plans). Word lists in tools often mix both; keep them apart in a data model.
- Fluid power = one area: ISO 1219-1/-2 cover hydraulics, pneumatics, cooling, lubrication and technical gases with the same symbols and rules; the medium is a code inside the identification, not another standard: [fluid-power](fluid-power.md).
- Process plants: ISO 10628-1 defines block diagram, process flow diagram (PFD; utility flow diagram as special case) and P&ID with minimum content; instrumentation per IEC 62424 or ANSI/ISA-5.1: [process-pid](process-pid.md).
- Cross-area standards: IEC 81346 designations ([iec-81346](../iec-81346/index.md); IEC 81346-2 letters are used in fluid diagrams too), ISO 14617 symbols for diagrams, ISO 5457 sheets, ISO 7200 title blocks (all normative references of ISO 10628-1).
- Tools: one device, several representations. Overview/single-line are other representations of the same device in the same area, not an own area. Fluid is bundled with electric (EPLAN Fluid, Zuken E3.fluid: machine building); P&ID with plant engineering (EPLAN Preplanning, Aucotec Engineering Base, Siemens COMOS): [tools](tools.md).
- Line data shared by all areas: identification number, size (wire size / nominal size / nominal diameter), type or class, length, remark. Area-specific: color (wires, tubes), medium, pressure, temperature, insulation, flow direction: [tools](tools.md#line-data-across-areas).

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [Fluid power diagrams](fluid-power.md) — Read when supporting hydraulic, pneumatic, cooling or lubrication circuit diagrams - ISO 1219 parts, identification of components, ports and piping, line kinds, how EPLAN Fluid models lines.
- [Process flow diagrams and P&ID](process-pid.md) — Read when supporting process plant diagrams - ISO 10628-1 diagram kinds and minimum content, line widths, flow arrows, crossings, pipeline data (DEXPI, Visio), ISA-5.1 signal lines.
- [Diagram kinds in CAE tools](tools.md) — Read when deciding which diagram kinds/disciplines a tool supports and how one device appears in several diagrams - EPLAN, Zuken E3, Aucotec EB, COMOS, Visio; line data compared across areas.
<!-- END GENERATED CONTENTS -->
