---
title: Visio undo scopes - rollback and nesting
description: "Read when Visio code uses BeginUndoScope/EndUndoScope to make several changes one Undo step, to roll back on error or cancel, or nests scopes (a dialog session with steps)."
---

# Visio undo scopes - rollback and nesting

`Application.BeginUndoScope(name)` returns an ID; `EndUndoScope(id, commit)` ends it. `commit = true`: all changes inside are one Undo step named `name`. `commit = false`: Visio takes all changes inside back at once, and no Undo step is left.

```csharp
int scope = app.BeginUndoScope("My Action");
bool done = false;
try { /* changes */ done = true; }
finally { app.EndUndoScope(scope, done); }   // exception -> everything taken back
```

## Dialog session: one scope, steps nested

A dialog that applies each step at once, with OK = one Undo step and Cancel = nothing changed:

```csharp
int session = app.BeginUndoScope("Manage Things");      // dialog opens
// each step:
int step = app.BeginUndoScope("Step");
try { /* changes */ app.EndUndoScope(step, true); }
catch { app.EndUndoScope(step, false); /* show error, session goes on */ }
// OK:     app.EndUndoScope(session, true);
// Cancel: app.EndUndoScope(session, false);
```

Measured (each case once):

- Outer dropped: everything of the session is back, also committed inner scopes: deleted shapes, deleted pages, deleted and added layers, layer membership, Shape Data rows and cells, document cells. The next `Undo()` takes back the step **before** the session (no empty step left).
- Outer committed: one Undo step; one `Undo()` takes back the whole session, `Redo()` brings it again.
- Inner dropped inside the session: only that step is taken back; earlier committed steps stay, and the outer commit makes them one Undo step.
- A step that fails halfway (an exception from Visio, e.g. `Cell is guarded.`): dropping its inner scope takes back what it changed before the error.

## Gotchas

- `Layer.Name = x` is not undone cleanly: after Undo (also a dropped scope) the layer still reports the new `Name`, `NameU` is empty and its cells throw `An exception occurred.`; only save + reopen shows the right old name. Rename layers another way: [visio-layers](visio-layers.md#names).

Verified with Visio Pro 2019 C2R x64 by COM automation from PowerShell (`Visio.InvisibleApp`, out of process), 2026-10. _Unverified:_ the same calls from an add-in (in process) with a modal dialog open; scopes over several documents.
