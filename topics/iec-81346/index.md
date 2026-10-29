---
title: IEC 81346 reference designation
description: "IEC 81346 reference designations (device tags) for schematics: aspects and syntax, letter-code classification, how EPLAN implements it."
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# IEC 81346 reference designation

IEC 81346 defines reference designations (German: Referenzkennzeichen, formerly BMK) for objects in technical systems, e.g. `=AP+ST1-M1` in a circuit diagram. Not only electrical: IEC 81346-2:2019 is meant for all technical areas (electric, mechanical, process, civil) and branches (energy, chemical, construction, automotive, marine) (its introduction). Read before designing how a schematic tool stores, composes, numbers or displays device tags. The standards are paid; facts here come from public previews and secondary sources, each file names them.

## Key facts

- Three aspects: `=` function, `+` location, `-` product; levels by concatenation along the path (`+H1+A1-KF3`): [syntax](syntax.md).
- One level = letter code of the class + number; codes, not words. Readable names live next to the code as descriptions: [syntax](syntax.md), [eplan](eplan.md).
- Letter codes from IEC 81346-2:2019 (products by inherent function, spaces, infrastructure; 1-3 letters); project-specific tables allowed if built the same way: [classification](classification.md).
- The standard fixes syntax, not the number or meaning of levels, nor the order of aspects; each project documents its structure.
- EPLAN model: `== = ++ + -`, identity ("Seitenstruktur") separate from display ("Vollständiges BMK"), identifier + description lists: [eplan](eplan.md).

<!-- BEGIN GENERATED CONTENTS: do not edit, run `python .knowledge-base/manage.py index` -->
## Contents

- [Classification and letter codes](classification.md) — Read when choosing letter codes for products, locations or functions - IEC 81346-2:2019 schemes, code levels, edition differences, project-specific tables.
- [EPLAN implementation](eplan.md) — Read when designing a tool's device tag model after EPLAN - structure identifiers (== = ++ +), tag parts, page structure, display vs identity, code + description pairs.
- [Aspects and syntax](syntax.md) — Read when composing, parsing or displaying an IEC 81346 reference designation - aspects and prefix signs, single and multi level, allowed content, order.
<!-- END GENERATED CONTENTS -->
