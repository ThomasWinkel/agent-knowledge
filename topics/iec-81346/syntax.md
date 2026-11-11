---
title: Aspects and syntax
description: Read when composing, parsing or displaying an IEC 81346 reference designation - aspects and prefix signs, single and multi level, allowed content, order.
---

# Aspects and syntax

Standard: IEC 81346-1:2022 (replaces the 2009 edition). The full text is paid; public previews show only the table of contents. Facts below come from secondary sources and ISO/TS 81346-3:2012, which cites IEC 81346-1:2009.

## Aspects

| Prefix | Aspect | Question |
|---|---|---|
| `=` | function | what does the object do? |
| `+` | location | where is the object? |
| `-` | product | what is the object built of / what is it? |

- Aspects are independent views of the same object: one designation per aspect, combined as needed (`=N-T1+L11`). The location aspect is optional (myelectrical.com).
- German term: Referenzkennzeichen, formerly Betriebsmittelkennzeichen (BMK) (Wikipedia DE).

## Single level

- Prefix sign + letter code of the object class + number, e.g. `-KF3`, `+A1`. The number tells apart objects of the same class inside the same parent object (ISO/TS 81346-3:2012, 4.2).
- Also allowed: letter code alone, or number alone (myelectrical.com).
- A single-level designation contains no designation of upper or lower objects (ISO/TS 81346-3:2012, 3.17).
- Content is codes, not words: meaning comes from the class letter code ([classification](classification.md)) and the structure. A readable name ("control cabinet powertrain") is kept as a description next to the code; tools hold code + description pairs ([eplan](eplan.md)).

## Multi level

- Concatenate the single levels along the path, starting at the top: `=N1-A1-Q1` = breaker `Q1` inside assembly `A1` of function `N1` (ISO/TS 81346-3:2012, 4.2; example myelectrical.com).
- Levels exist in every aspect: `+H1+A1` (building, cabinet), `=A1=B2`, `-U1-K1`. The standard fixes syntax, not how many levels there are or what they mean; the project defines and documents its structure and notation (ISO/TS 81346-3:2012, 4.1.3-4.1.4).
- `+U1-K1` (K1 sits in U1) and `-U1-K1` (K1 is part of product U1) are different statements.
- Product levels nest like the others: `-QA2-MB1` = solenoid 1 of valve 2 (FRAME webinar, 2.3).
- One path may change aspect: IEC 81346-1:2009 has "Figure 22 - Examples of multi-level reference designations with multiple prefix signs", "Figure 19 - ... object identified by means of one aspect and with sub-objects identified by means of another aspect" and Annex D "Interpretation of reference designations using different aspects" (shifts function <-> product, product <-> location, function <-> location; the 1996 term "transition" was dropped) (table of contents and foreword of the preview). The rule text itself: _unverified_ (not in the preview).
- Shortened form without repeated prefix (`=N-A1Q1`) appears in examples (myelectrical.com); _Unverified_ as a rule.

## Order

- No fixed order of aspects in the standard (agent's reading of sources; examples differ: `=N-T1+L11` myelectrical.com, `=AP+ST1-M1` EPLAN, `=A1-K3+U4` industrialmonitordirect.com).
- EPLAN order: `== = ++ + -` ([eplan](eplan.md)); the usual practice.
- An object may carry one designation per aspect side by side (reference designation set, clause 7).

## Objects, names and reference designation sets (IEC 81346-1:2022, Annex B and C, preview text)

- The norm prescribes no design process (C.1). An object is created when the designer needs it, "just a 'placeholder' for information, given a name and possibly identified by means of its reference designation" (C.2.1). Early on it "can be named in plain language, based on its place in the functional process, and codified by the reference designation based on the function-oriented structure" (B.2.3).
- Location designation comes when the location structure exists; together with the function-oriented designation it "establishes a reference designation set" (B.2.4). A location designation that names only the room/area, not the space the object occupies, "is said to be ambiguous" (B.2.4).
- One thing defined from three aspects (requirement = function, implementation = product, placement = location) = three closely related objects (C.2.3). Two ways: (a) keep them separate, relations kept by the CAx tool, three reference designations; (b) merge into one object addressed by a reference designation set; its designations are "alternative 'addresses' to the object under consideration, all equally valid" (C.2.4). How relations are kept: "a computer system implementation issue" (C.2.3).
- Each designation of a set comes from its own structure (function tree, location tree, product tree), so each ends in a code of that structure; whether practice uses different counters per aspect for one object: _unverified_ (figures not in the preview).

## Gotchas

- A free name with a hyphen (`AB12-1`) inside a composed designation reads as two product levels (`-AB12-1` = `1` inside `AB12`). Use another separator in names that may end up in a designation.
- _Unverified_ (common practice, not found in a source): `:` before a terminal (`-X1:3`, terminals: IEC 61666), `.` for a part of one object (`-K3.2` = contact 2 of relay K3), `&` before a document code (IEC 61355).

## Sources

- ISO/TS 81346-3:2012 preview: https://cdn.standards.iteh.ai/samples/59598/edd5fcaf82554181a625e70929f4365e/ISO-TS-81346-3-2012.pdf
- IEC reference designations: https://myelectrical.com/notes/entryid/24/iec-reference-designations
- Wikipedia DE: https://de.wikipedia.org/wiki/EN_IEC_81346
- IEC 81346-1:2009 preview (contents, foreword): https://www.normservis.cz/download/view/iec/info_iec81346-1%7Bed1.0%7Db.pdf, https://shop.standards.ie/preview/98697315321.pdf?sku=873844_SAIG_NSAI_NSAI_2077624
- FRAME Consulting, "Arbeiten nach IEC 81346", EPLAN webinar 2021-11-08: https://frame-engineering.de/wp-content/uploads/2021/11/Webinar-IEC-81346.pdf
- IEC 81346-1:2022 preview (pages 57-58, 63-67): https://www.normsplash.com/Samples/BSI/136456632/BS-EN-IEC-81346-1-2022-en-2.pdf
- Function-location-component guide: https://industrialmonitordirect.com/blogs/knowledgebase/iec-81346-aspect-notation-function-location-component-setup
