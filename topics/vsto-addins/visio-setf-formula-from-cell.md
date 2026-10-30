---
title: Visio SETF - a formula text from another cell, kept live
description: Read when shapes should take a formula kept as text in a document cell (display rules, SETF and GETREF) - prefix, silent failures, copy, masters in stencils, Undo, speed.
---

# Visio SETF: a formula text from another cell, kept live

A document cell holds a formula **as text**; every shape writes it into its own cell by `SETF` in a trigger cell. When the text changes, Visio rewrites the formula in every shape, without add-in code. Reference: [SETF](https://learn.microsoft.com/en-us/office/client-developer/visio/setf-function). Measured with Visio Pro 2019 C2R x64, invisible Visio by COM from PowerShell.

```text
TheDoc!User.RuleA        = """+""&User.Unit&""-""&User.Name"      (value = the formula text)
Shape  User.Text         = (written by SETF)
Shape  User.Text.Prompt  = SETF(GETREF(User.Text),"="&IF(STRSAME(User.Show,"A"),TheDoc!User.RuleA,TheDoc!User.RuleB))
```

Result: `User.Text` formula `"+"&User.Unit&"-"&User.Name`, evaluated in the shape (own cells). The trigger cell's value is `0`.

## Behavior

- Live: a change of the document cell or of `User.Show` rewrites the target formula. A change of `User.Name` only re-evaluates it (no `SETF`).
- `"="&` is required: without it the text is written as a string constant (`"""+""&..."`), the shape shows the formula text.
- A user value typed into the target cell stays until an input of the trigger cell changes.
- Undo: change the document cell in one undo scope -> one Undo takes back the document cell **and** every shape's formula; Redo brings both.
- Copy (`Page.Drop(shape)`) into another document: the shape takes that document's text at once. Target document without the cell: no error, the source's formula stays; adding the cell later makes it live at once. Clipboard paste: _unverified_.
- Master: the trigger formula cannot be written in a document without the referenced cell (`#NAME?`) -> a stencil must carry the document cells too. The master shows the stencil's text; an instance takes the drawing's text at drop, its target cell formula is then local.
- Speed (`ScreenUpdating` 0, invisible): change of the document cell with 1,024 shapes 136-299 ms, 8,192 shapes 2,052-2,396 ms; its Undo about the same. Writing the same formulas by `Page.SetFormulas` in one undo scope is not faster (8,192: 2,498-2,648 ms).

## Gotchas

- **Broken text fails silently**: syntax error (`"+"&User.Unit&`), a missing cell (`User.Missing`), a reference to the target cell itself: `SETF` writes nothing, no error in any cell, the old formula stays. Check a text before storing it, e.g. write it once by code (`Cell.FormulaU` throws `An exception occurred.` for a syntax error).
- Empty text -> empty formula, the cell shows `0` (number). For "show nothing" store `""""` (= formula `""`).
