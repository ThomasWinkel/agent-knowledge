# Contributing

Rules for writing to the knowledge base. Paths and commands are relative to the knowledge base root; when embedded in a project, prefix them with its folder (e.g. `python agent-knowledge/.knowledge-base/manage.py lint`). Use `python`, `python3` or `py` (Python ≥ 3.11).

## Principles

- **Only what an agent cannot know.** Not in training data, not trivially derivable from code or docs at hand. In a project: not what the code shows, but what it does not — reasons, pitfalls, external systems, working procedures. Link authoritative sources instead of copying them; record what they lack: undocumented behavior, the working recipe, gotchas.
- **One place per fact.** Search before writing (`grep -rli "<term>" topics/`). Extend the existing place or link to it; never repeat.
- **Edit, don't append.** Rewrite so the file reads as if written today. Delete what is wrong or obsolete. No changelogs, "updated on" notes or history — git has that.
- **Compact.** English, terse, imperative, bullets. Commands and code over prose. No introductions, no filler, no general knowledge.
- **Precise.** Exact names, paths, versions ("applies to API v2"). Prefer facts you verified during the task; mark unverified ones.
- **Durable.** Volatile values (current IDs, prices, people) go stale — describe where to look them up instead.
- **No secrets, no personal data.** Public repositories are public; lint scans for common token formats.

## Where it belongs

1. **Useful beyond this project or scope**, and you know a shared knowledge base for it (from the "## Agent knowledge" section, the user's instructions or links): contribute it there following its guide, and link it from here. Without access, keep it here; it can move later.
2. **A topic covers the subject:** add it there.
3. **Embedded knowledge base, no matching topic:** `project` for what was decided (architecture, decisions, conventions), `learnings` for what had to be learned (external systems, tools, environment, workarounds). One file per subject.
4. **Otherwise:** do not create a topic; mention the knowledge in your final line to the user.

## File format

```markdown
---
title: Authentication
description: Read when obtaining or refreshing tokens, or debugging 401/403 responses.
---

# Authentication

Short summary line, then steps or reference.

## Gotchas

- ...
```

- Frontmatter: exactly `title` and `description`, single line each. Quote values containing `: ` or starting with YAML special characters.
- `description` is what agents choose files by. Detail files start with "Read when ...". Topic indexes state what the topic covers. Max ~200 characters.
- One concern per file. File and folder names: lowercase kebab-case.
- Size: warning above 300 lines, error above 500 — split into files or a subfolder.
- Links: relative paths (`../other-topic/file.md`). Embedded in a project, link the project's files too (`../../../src/client.py`) instead of copying code. Other repositories: absolute URLs.
- Other files (examples, schemas, images) may live next to the Markdown files; link them.

## Topic entry point (`index.md`)

```markdown
---
title: Billing API
description: Internal billing REST API - authentication, endpoints, error semantics.
---

> **Agents:** read [the knowledge base guide](../../AGENTS.md) first.

# Billing API

2-5 sentences: what it is, when an agent needs it.

## Key facts

- The few facts needed for almost every task, with links to details.

<!-- BEGIN GENERATED CONTENTS ... -->
<!-- END GENERATED CONTENTS -->
```

- Hand-written part under ~80 lines. Never edit the generated block; `manage.py index` rewrites it from the frontmatter of the files.
- Large topics may have subfolders; each needs its own `index.md` of the same shape.

## New topic (only when the user asks)

A topic is an area an agent is pointed to as a whole: a system, an API, a process. Agree on slug and scope with the user, then:

```sh
python .knowledge-base/manage.py new-topic <slug> --title "<Title>" --description "<what it covers>"
```

Use `<slug>/<sub>` for a subtopic. Replace the TODO in the created `index.md`.

## Workflow

1. Start:
   - Own repository (`pull-request`, `direct-push`), in its clone (e.g. `~/.agent-knowledge/<repository-name>`): `git fetch origin && git switch -c knowledge/<topic>-<short-description> origin/main`.
   - Embedded (`with-project`): stay on the current branch; no separate branch, fetch or pull.
2. Edit files.
3. `python .knowledge-base/manage.py index`, then `python .knowledge-base/manage.py lint`. Fix all errors; address warnings where reasonable. Renamed or deleted a file: fix the links lint reports.
4. Publish according to `contribution` in `knowledge-base.toml`:
   - **`with-project`:** the knowledge changes are part of your task's changes. Commit and publish them together, the way the project handles the task (same commit or pull request), so they are reviewed with the code.
   - **`pull-request`:** commit (`<topic>: <what changed>`), `git fetch origin && git rebase origin/main`, `git push -u origin HEAD`, then `gh pr create --fill` (without `gh`: give the user the link `git push` prints). Describe what was wrong or missing and how you verified it. PRs by `trusted_authors` that only change `topics/` and `index.md` are merged automatically once checks pass; all others wait for review. No push access: `gh repo fork --remote` and open the PR from the fork.
   - **`direct-push`:** commit, `git fetch origin && git rebase origin/main`, `git push origin HEAD:main`. If rejected because `main` moved: fetch, rebase, rerun `index` and `lint`, push again.
   - In `pull-request` and `direct-push`, **changes outside `topics/` and `index.md`** (tooling, guides, configuration) always need a review: open a PR, or in `direct-push` mode push the branch (`git push -u origin HEAD`) and ask the user to review and merge it.
   - Rebase conflicts in generated blocks or `index.md`: take either side, then rerun `manage.py index`.
5. Run `python .knowledge-base/manage.py check-template`. If it reports an update, mention it in your final line to the user; ignore failures.
