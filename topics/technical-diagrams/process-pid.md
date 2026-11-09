---
title: Process flow diagrams and P&ID
description: Read when supporting process plant diagrams - ISO 10628-1 diagram kinds and minimum content, line widths, flow arrows, crossings, pipeline data (DEXPI, Visio), ISA-5.1 signal lines.
---

# Process flow diagrams and P&ID

## ISO 10628-1 (Diagrams for the chemical and petrochemical industry, Part 1; read: ISO/DIS 10628-1 2012 sample, iTeh; published as ISO 10628-1:2014 / EN ISO 10628-1:2015)

- Not for electrical engineering diagrams. Part 2 = graphical symbols (ISO 10628-2). Normative: ISO 128, ISO 5457, ISO 7200, ISO 14617, ISO 15519, IEC 62424 (process control requests in P&ID).
- Three kinds: block diagram (rectangular frames + flow lines), process flow diagram (symbols + lines; utility flow diagram = special PFD for energy/utility systems), P&ID (based on the PFD: equipment + piping + measuring/control functions).
- P&ID minimum content (4.4.2): function and type of equipment incl. drives and reserves; identification numbers incl. drives; characteristic data (may be in separate lists); "nominal sizes, pressure ratings, materials and type of piping, e.g. by stating the pipeline number, piping class or identification number"; details of piping, valves, fittings; measuring/control functions with identification number. Optional: energy flows, material of construction, elevations, reference designations of valves and fittings.
- Equipment, valves, fittings per ISO 10628-2; measuring and control per IEC 62424.
- Drafting: sheet A1 preferred; title block per ISO 7200. Line widths on module M = 2.5 mm: 1.0 mm main flow lines; 0.5 mm equipment symbols, frames, subsidiary flow, energy and auxiliary lines; 0.25 mm valves, fittings, measuring/control symbols, signal lines, reference lines; never thinner. Parallel lines >= 2 x widest width and >= 1 mm apart, flow lines > 10 mm.
- Flow: main direction left to right, top to bottom; arrows inside lines show flow direction; inlet/outlet arrows per ISO 80416-2; lines leaving a sheet continue at the same height on the next.
- Crossings without connection: same width -> the vertical line is interrupted; different widths -> the thinner one is interrupted.

## Pipeline data

- DEXPI P&ID specification 1.4 (dexpi.org, CC BY 4.0), class `PipingNetworkSegment`: `NominalDiameterRepresentation` (`DN 25`), `NominalDiameterNumericalValueRepresentation`, `NominalDiameterTypeRepresentation` (`DN`), `NominalDiameterStandard`, `PipingClassCode`, `FluidCode`, `InsulationType`, `InsulationThickness`, `FlowDirection`, `SegmentNumber`, `PressureTestCircuitNumber`, `OperatingTemperature`, `HeatTracingType`, `JacketedPipe`, `Slope`. Pressure rating and material not seen there (partly read). Current spec list: DEXPI 2.0.1, P&ID 1.4, Process 1.0.
- Visio 2019 P&ID stencil `PEPIPE_M` (masters Major/Minor Pipeline, with L/R variants; read 2026-11-09): shape data `Description`, `LineSize`, `Schedule`, `Material`, `DesignPressure`, `DesignTemperature`. Instruments (`PEINST_M`): `Instrument type`, `Local / remote`, `Description`, `Connection Size`, `Service`, `Manufacturer`, `Model`. Equipment (`PEEQP_M`): `Description`, `Material`, `Manufacturer`, `Model`.
- Line number formats in practice (size - fluid code - sequence - piping class - insulation): common, no standard found; unverified.

## Instrumentation (ANSI/ISA-5.1)

- Current editions: ANSI/ISA-5.1-2024, ISA-TR5.1.02-2024 (identification), ISA-TR5.1.03-2024 (graphic symbols) (isa.org). Paid.
- Signal lines (secondary sources, controleng.com, automation.com, open-exam-prep; unverified against the standard): electric signal dashed; pneumatic signal solid with double slashes `//`; capillary tube solid with small `X`; data/software link line with small circles; hydraulic signal and electromagnetic lines exist, look not confirmed. These are not standard line patterns of drawing programs: plan custom line patterns.
- Tags: first letter measured variable, following letters functions, then loop number, e.g. `FIC 101`, `TIC 101` (common; unverified against the standard).
