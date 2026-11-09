---
title: Fluid power diagrams
description: Read when supporting hydraulic, pneumatic, cooling or lubrication circuit diagrams - ISO 1219 parts, identification of components, ports and piping, line kinds, how EPLAN Fluid models lines.
---

# Fluid power diagrams

## Standards

- ISO 1219-1: graphical symbols. ISO 1219-2:2012: circuit diagrams. ISO 1219-3: symbol modules and connected symbols. ISO 1219-2 covers hydraulics, pneumatics, cooling, lubrication and technical gas circuits (genorma.com summary); its annexes B-D are example hydraulic, pneumatic and lubrication diagrams (TOC).
- DIN ISO 1219-2:1995-12 withdrawn, replaced by ISO 1219-2:2012-09; no DIN ISO edition planned (Festo Didactic, "Reference identification to ISO 1219-2 and/or EN 81346-2", 02/2014).
- Port identification of valves: hydraulic ISO 9461 (letters; under revision), pneumatic ISO 11727 (numbers; confirmed 2020) (standard catalog pages; ifps.org forum). Common meanings, unverified against the texts: pneumatic 1 supply, 2/4 outlets, 3/5 exhaust, 12/14 pilot; hydraulic P pressure, T tank, A/B work ports, X/Y pilot, L drain.

## ISO 1219-2:2012 structure (TOC, DIN download of the contents)

- 5.1 identification code of components and hose assemblies: installation code, medium code, circuit number, component number (each a clause); Annex A relationship of the parts.
- 5.2 port identification; 5.3 identification code for piping (optional identification number, technical information, examples); 5.4 optional application code for piping (medium code, line code, pressure level index).
- 6 technical information on the circuit diagram: circuit function, 6.3 electrical reference designation, per component kind (reservoirs, air supply, pumps, prime movers, valves, cylinders, motors, accumulators, filters, piping, indicators, timers); Annex E parts list example.

## Identification

- Component code (Festo 2014, quoting ISO 1219-2:2012): system designation, `-`, medium code, circuit number (mandatory), `.`, component number (mandatory). Medium codes: `H` hydraulics, `P` pneumatics, `C` cooling, `K` cooling lubricants, `L` lubrication, `G` gas technology. Energy supply components preferably circuit `0`; components numbered bottom to top, left to right within a circuit.
- The 2012 code has no letter for the component's function. Festo Didactic therefore uses EN 81346-2:2009 letters in fluid diagrams since 2014, mostly two letters, e.g. `MM` cylinder, `QM` directional valve (extraction scrambled, likely), `GP` pump, `GQ` compressor, `RM` non-return valve, `RN` flow control valve, `BP` pressure sensor, `BG` limit switch, `AZ` air service unit, `MB` actuation coil. ISO 1219-2:2012 itself requires IEC/EN 81346 for electric components (Festo; TOC 6.3).
- Piping code (EPLAN help 2.9 `ftechnic_k_verbindungen`, citing DIN ISO 1219-2): optional line number, `-`, technical information starting with `Ø`. Tube: nominal size (`2-Ø20`); pipe: outer diameter x wall (`1-Ø30x5`) or outer/inner diameter (`3-Ø10/6`).

## Lines

- Line kinds (EPLAN help 2.9 `ftechnic_h_fluidleitungen_farblich_hervorheben`, per DIN ISO 1219-1): pressure, control (pilot), drain, reservoir (return), working lines; each kind has its own line type, color and layer. EPLAN defaults: pressure line orange, reservoir line light blue. The line kind follows the port: property `Pressure / control port` of the function connection point (dialog Connection point logic) sets the kind of the connected line.
- Pipe vs tube/hose is a property of the connection, not of the port (EPLAN `ftechnic_k_verbindungen`).
- Medium ("trade") of a fluid connection is derived from source and target and cannot be edited (EPLAN). Trade values: Hydraulics, Pneumatics, Lubrication, Cooling, Cooling lubricant, Gas engineering (EPLAN `ftechnic_k_einstellungen_fluidverbindungen`).
- EPLAN models parallel hoses as a conduit definition, used like a cable definition; piping and hose lines are devices with device tags (macros `Pipe1`-`Pipe5`, `Hose`); properties Length, Cross-section / diameter, Unit; report "Conduit / line plan" (EPLAN `ftechnic_k_schlauchpakete`). Cable reports do not apply to them.

## Visio's own Fluid Power stencils (Visio 2019, `FPEQP_M`, `FPVALV_M`, `FPASSM_M`, read 2026-11-09)

- Symbols only, no line master: lines are drawn with the normal connector. Shape data are symbol variants (e.g. pump `Displacement` `Fixed;Variable`, `Flow Directions`, `Control Mechanism`), no identification or line data.
