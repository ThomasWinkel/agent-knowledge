---
title: EPLAN implementation
description: Read when designing a tool's device tag model after EPLAN - structure identifiers (== = ++ +), tag parts, page structure, display vs identity, code + description pairs.
---

# EPLAN implementation

EPLAN Electric P8 is the common reference for IEC 81346 in electrical CAD. Useful as a model for own tools.

## Device tag (BMK) parts

| Prefix | EPLAN name (DE) | Meaning |
|---|---|---|
| `==` | Funktionale Zuordnung | functional assignment |
| `=` | Anlage | plant / system |
| `++` | Aufstellungsort | installation site |
| `+` | Einbauort | mounting location |
| `-` | BMK | Vorziffer (leading digit), Kennbuchstabe (letter code), Zähler (counter), Trennzeichen (separator), Unterzähler (sub-counter) |

- Also "Benutzerdefiniert: freie Hierarchie" (user-defined, free hierarchy).
- Example: `=AP+ST1-M1` = motor `M1` in mounting location `ST1` of plant `AP`.
- `==` and `++` are EPLAN's way to give the top level of the function and location aspects its own field; IEC 81346-1 itself defines only `=`, `+`, `-` ([syntax](syntax.md)).

## Identity vs display

- Project property "Seitenstruktur" (page structure) sets which parts identify a device uniquely.
- Dialog "Vollständiges BMK" (full device tag) sets how the tag is shown (e.g. with or without leading digit, sub-counter); display format is independent of the project structure.
- _Unverified_: parts known from the page (title block) are left out at the symbol, the full tag stays internal; structure identifiers inherit project -> page -> symbol, empty = inherit.

## Code + description pairs

- Structure identifier management (Project data > Structure identifier management) holds lists of identifiers or of identifier + description pairs, e.g. `+A1` = "control cabinet powertrain".
- Descriptions are multilingual, format `'de_DE@Beschreibung A;en_US@Description A;'`; list syntax `=List{Pair{'A', '<description>'}, ...}` (EPLAN EEC).
- Use for tools: users think in descriptions, the designation holds the short code.

## Sources

- Aufbau des Betriebsmittelkennzeichens: https://www.eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/devicetaggui_k_aufbauvonbm.htm
- Structure identifier management in P8: https://eplan.help/en-us/Infoportal/Content/EECPro/2.9/Content/htm/eececad_k_structure_identifier_management_in_p8.htm
