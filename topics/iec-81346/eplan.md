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

## In practice (FRAME webinar 2021, an EPLAN consultant)

- Most used project schema: "Anlage und Einbauort" (`=` + `+`); `==` and `++` set to "not available". `==` only for nested functions in complex plants and machines, `++` for nested locations in large sites.
- Function `=` examples: infrastructure `=PSS` power supply, `=PLC`; real functions `=CB1` conveyor 1. Location `+`: `+CC1` control cabinet 1, `+FC1` fluid cabinet 1.
- Recommended: function-oriented projects: all pages of one function in a row (drive, control, fault signals of conveyor 1), built from a macro pool; old style was location/build-oriented (power part, control part, PLC card by card), easier to wire from.
- Device tag = structure identifiers + letter code + counter; the counter starts at 1 again for each structure identifier (no page prefix needed). Page numbers start at 1 per structure too.
- Each block is "identifying" (part of identity, used to tell pages and devices apart) or "descriptive" (shown only).
- CTB (vendor article): a designation is unique within its plant/function aspect: `-K3` once inside `=A1`, again inside `=A2` allowed.
- _Unverified_ how the location reaches a device in practice: pages carry `=` and `+` and devices inherit them (help says the blocks define the hierarchy of pages and of objects in the schematic); setting it from the panel layout placement not seen in a source.

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
- Kennzeichnungsblöcke (designation blocks, "beliebig kombinierbar"): https://www.eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/pleditorgui_k_grundlagen.htm
- FRAME Consulting webinar: https://frame-engineering.de/wp-content/uploads/2021/11/Webinar-IEC-81346.pdf
- CTB, IEC 81346 reference designation system: https://www.ctb.co.at/en/knowledge/iec-81346-reference-designation-system/
