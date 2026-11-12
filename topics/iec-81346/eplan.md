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

## Function text and extended reference designation (EPLAN help 2.9)

- Function text (Funktionstext) = descriptive text of a function (symbol), not identifying: manual (property 20011), common (20120: automatic text of the function, else of its main function), automatic composed by block property (20545, since 2.8). Path function text: on logic pages the topmost/leftmost path text along the schematic path; connection lists show the text of the symbol the connection is attached to; device lists and parts lists output only the main function and its text.
- Main function / auxiliary functions: one device appears as several functions (symbols); lists use the main function only (path text help).
- "Erweiterte Referenzkennzeichnung" (EN 81346 mode): function aspect = Anlage `=`, location = Einbauort `+`, product = Produktkennzeichen `-`. The page structure is then not used to structure devices; structure boxes (Strukturkästen) pass the product designation on to elements inside; functions without visible device tag take the parent structures. Help tip: put a structure box around the whole page.

## Identity vs display

- Project property "Seitenstruktur" (page structure) sets which parts identify a device uniquely.
- Dialog "Vollständiges BMK" (full device tag) sets how the tag is shown (e.g. with or without leading digit, sub-counter); display format is independent of the project structure.
- _Unverified_: parts known from the page (title block) are left out at the symbol, the full tag stays internal; structure identifiers inherit project -> page -> symbol, empty = inherit.

## Code + description pairs

- Structure identifier management (Project data > Structure identifier management) holds lists of identifiers or of identifier + description pairs, e.g. `+A1` = "control cabinet powertrain".
- Descriptions are multilingual, format `'de_DE@Beschreibung A;en_US@Description A;'`; list syntax `=List{Pair{'A', '<description>'}, ...}` (EPLAN EEC).
- Use for tools: users think in descriptions, the designation holds the short code.

## Terminals and terminal strips

- Terminal strip definition (Klemmenleistendefinition) identifies the strip and holds its device tag (`-X1`); a strip may exist without terminals (help 2.9, glossary).
- Each terminal is defined by its device tag; terminals with the same tag + accessories form the strip (EPLAN blog). Terminal = strip tag + terminal designation, e.g. `=A+O-X100:1` (TechTip saddle jumpers).
- Terminal designation is free text; sorting copes with mixed values (`PE`, `A10`, `10A`; dialog "Klemmenleiste bearbeiten"). Order in the strip: device position orders the terminal devices, sort ID inside one; new terminals go to the end (help 2.9, Verwaltung von Klemmen).
- Multi-level terminal: terminals with the same tag, next to each other in the sort order, level values rising or falling; ascending sort -> lowest level = 1 (EPLAN blog).
- Connection points of one terminal: several connection designations per terminal (EPLAN: separated by paragraph marks); a partial terminal (Teilklemme) symbol shows one or more connections of it, so one terminal can appear in several places of the schematic. Terminal targets are evaluated only at connections that have a designation (help 2.9). 4-connection terminals commonly get connections `a`..`d` (CAD.de forum, old). Terminals have an internal and an external side.
- Jumper types (help 2025; German help 2.9/2022): jumper, wire jumper (Drahtbrücke), insertable jumper (Einlegebrücke), saddle jumper (Stegbrücke), switching jumper (Schaltbrücke).
  - Stegbrücke: default when two **adjacent** terminals of a strip are connected; needs terminals with a jumper option, not PE/PEN, no accessory between them. Otherwise (not adjacent, or a terminal without jumper option): Drahtbrücke, a real wire.
  - Einlegebrücke: laid into the clamping space (Phoenix data sheet: "Einlegebrücken im Klemmraum"; Stegbrücke = fixed jumper in the terminal middle); forum: needs an external wire jumper definition where it is clamped.
  - Schaltbrücke: switchable link to the next terminal, states open/closed, defined separately for the internal and external side.
  - Manual Stegbrücke is defined at its start (since 2.5); terminals inserted between two jumpered terminals change its target (TechTip).
- Other practice (vendor blog, secondary, not checked against NFPA 79): Europe addresses a terminal point as `X1:14`; machine builders often use cable designation + terminal number instead of per-wire numbers; US-style wire numbers stay the same at every termination, terminal blocks prefixed `TB`.

## Sources

- Aufbau des Betriebsmittelkennzeichens: https://www.eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/devicetaggui_k_aufbauvonbm.htm
- Structure identifier management in P8: https://eplan.help/en-us/Infoportal/Content/EECPro/2.9/Content/htm/eececad_k_structure_identifier_management_in_p8.htm
- Kennzeichnungsblöcke (designation blocks, "beliebig kombinierbar"): https://www.eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/pleditorgui_k_grundlagen.htm
- Extended reference designation: https://www.eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/projectstructure_k_referenzkennzeichnung.htm
- Path function texts: https://www.eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/gededitgui_k_pfadtextauswertung.htm
- Properties list (function text IDs): https://www.eplan.help/de-DE/Infoportal/Content/Plattform/2022/Content/htm/properties_o_main.htm
- FRAME Consulting webinar: https://frame-engineering.de/wp-content/uploads/2021/11/Webinar-IEC-81346.pdf
- CTB, IEC 81346 reference designation system: https://www.ctb.co.at/en/knowledge/iec-81346-reference-designation-system/
- Klemmenleistendefinition (glossary 2.9): https://eplan.help/de-de/infoportal/content/Plattform/2.9/Content/htm/Glossary_o_klemmenleistendefinitionen.htm
- Verwaltung von Klemmen (2.9): https://www.eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/terminalgui_k_verwaltung.htm
- Dialog Klemmenleiste bearbeiten (2.9): https://eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/stripmanagementgui_d_klemmenleistebearbeiten.htm
- Jumper types (2025): https://eplan.help/en-us/infoportal/content/Plattform/2025/Content/htm/terminalgui_k_verwendungbruecken.htm
- TechTip manual saddle jumpers: https://www.eplan.help/techtipps/de-DE/Klemmen/TechTip-Behaviour-of-manual-saddle-jumpers.pdf
- EPLAN blog, partial and multi-level terminals: https://www.eplan.com/de-de/blog/software/teilklemmen-tipps
- Wire identification DIN vs UL 508A (vendor blog): https://industrialmonitordirect.com/blogs/knowledgebase/wire-identification-in-industrial-control-panels-din-vs-ul-508a
- Brücken erstellen (2.9): https://www.eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/terminalgui_h_brueckenarbeiten.htm
- Verwendung von Brücken (2022): https://www.eplan.help/de-DE/Infoportal/Content/Plattform/2022/Content/htm/terminalgui_k_verwendungbruecken.htm
- Teilklemmen einfügen (2.9): https://www.eplan.help/de-de/Infoportal/Content/Plattform/2.9/Content/htm/terminalgui_h_teilklemmenzeichnen.htm
- CAD.de forum, connection designations of terminals: https://ww3.cad.de/foren/ubb/Forum467/HTML/009766.shtml
