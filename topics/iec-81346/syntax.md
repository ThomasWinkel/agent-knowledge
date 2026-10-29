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
- Shortened form without repeated prefix (`=N-A1Q1`) appears in examples (myelectrical.com); _Unverified_ as a rule.

## Order

- No fixed order of aspects in the standard (agent's reading of sources; examples differ: `=N-T1+L11` myelectrical.com, `=AP+ST1-M1` EPLAN).
- EPLAN order: `== = ++ + -` ([eplan](eplan.md)).

## Gotchas

- A free name with a hyphen (`AB12-1`) inside a composed designation reads as two product levels (`-AB12-1` = `1` inside `AB12`). Use another separator in names that may end up in a designation.
- _Unverified_ (common practice, not found in a source): `:` before a terminal (`-X1:3`, terminals: IEC 61666), `.` for a part of one object (`-K3.2` = contact 2 of relay K3), `&` before a document code (IEC 61355).

## Sources

- ISO/TS 81346-3:2012 preview: https://cdn.standards.iteh.ai/samples/59598/edd5fcaf82554181a625e70929f4365e/ISO-TS-81346-3-2012.pdf
- IEC reference designations: https://myelectrical.com/notes/entryid/24/iec-reference-designations
- Wikipedia DE: https://de.wikipedia.org/wiki/EN_IEC_81346
