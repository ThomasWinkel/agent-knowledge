# Use a knowledge base in a project

Makes agents consult a knowledge base, or some of its topics, in every session of a project. Only when the user asks for it ("include", "always use", "add to this project"); a link alone is for the task at hand.

1. **Get the knowledge base** as described under Access in [AGENTS.md](../AGENTS.md): the clone `~/.agent-knowledge/<repository-name>`, or the folder of an embedded knowledge base.
2. **Generate the entry** in the knowledge base root:
   ```sh
   python .knowledge-base/manage.py pointer [--topic <slug> ...]
   ```
   - With `--topic`: the project needs a few topics. Preferred: their descriptions tell agents exactly when to consult them.
   - Without: the project needs the whole knowledge base, or the user linked it as a whole.
3. **Choose the file** — the project's always-loaded instructions:
   - `AGENTS.md`; `CLAUDE.md` if there is no `AGENTS.md` or `CLAUDE.md` does not import it (`@AGENTS.md`).
   - Neither exists: create `AGENTS.md` with the section and `CLAUDE.md` containing `@AGENTS.md`.
   - For all projects, only if the user asks: the user's global instructions (e.g. `~/.claude/CLAUDE.md`).
4. **Merge** into the "## Agent knowledge" section; there is exactly one per file:
   - No section: add the printed section.
   - Section exists: keep its other entries, add the new ones, replace an entry with the same location.
   - Text above the list without `agent-knowledge pointer vN` or with a lower N than printed: replace it with the printed text.
5. **Publish** a project file like any project change (part of the task's commit or pull request). Tell the user in one line what is included.

An outdated section (see Projects in [AGENTS.md](../AGENTS.md)) needs no request: run step 2 and replace only the text above the list, then step 5. Entries keep their shape across versions.

To remove a knowledge base or topic, delete its entry; delete the section when its list is empty.
