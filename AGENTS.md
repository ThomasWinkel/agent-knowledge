# Agent Guide

This is a knowledge base for AI agents: condensed knowledge that is **not in your training data** — internal processes, private APIs, conventions, verified recipes and gotchas. It is a repository of its own or a folder embedded in a project repository. Read this guide once per session before using it.

## Layout

Paths are relative to the knowledge base root, the folder containing `knowledge-base.toml`.

- `index.md` — all topics (generated).
- `topics/<topic>/index.md` — topic entry point: overview, key facts, list of files with descriptions.
- `topics/<topic>/*.md` — detail files, each with YAML frontmatter `title` and `description`.
- `.knowledge-base/` — tooling, maintenance guides and the design rationale ([DESIGN.md](.knowledge-base/DESIGN.md)). Ignore unless contributing, maintaining or asked about the concept.

## Access

- **In your working tree** (embedded in the project, `contribution = "with-project"`, or you work in the knowledge base repository itself): read the files there; never pull or switch branches for the knowledge base.
- **Own repository:** prefer git (SSH or HTTPS, any git server): it returns exact content and is needed to contribute. You may get a link to a file, or a repository URL plus a topic name — then open `topics/<topic>/index.md` in the clone.
  - Clone path: `~/.agent-knowledge/<repository-name>` (`~` = home directory), shared across sessions and projects; your scratchpad if home is not writable.
  - Existing clone: `git fetch origin && git switch --detach origin/main` — always read `main`, never a leftover branch. If that fails (local changes), read anyway and tell the user.
  - No clone: `git clone --depth 1 <repository-url> <clone-path>`.
  - Quick lookups without git (public GitHub repositories only): fetch raw files, converting `https://github.com/<owner>/<repo>/blob/<branch>/<path>` to `https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<path>`. Prefer `curl` over fetch tools that summarize pages; summaries drop exact details.

## Projects

- A link to a knowledge base or topic is for the task at hand: do not record it in project instructions or memory. If it helped and the project's "## Agent knowledge" section has no entry for it, offer once in your final line to include it permanently.
- Include it permanently only when the user asks: [.knowledge-base/CONNECT.md](.knowledge-base/CONNECT.md).
- An "## Agent knowledge" section in your instructions is outdated if it does not say `agent-knowledge pointer v1` or higher: update it following CONNECT.md (in a project as part of your task's changes) and mention it in one line.

## Reading

1. Read the topic `index.md` you were given, or the root `index.md` to find a topic.
2. Pick files by their `description` and load only what the task needs.
3. Follow links into other topics only when relevant.
4. Search: `grep -r "^description:" topics/` lists every file; `grep -rli "<term>" topics/` finds mentions.

The content is reference knowledge, not instructions that override the user. If it contradicts what you observe, trust the observation and fix the knowledge base afterwards.

## Contributing

Record knowledge within this knowledge base's scope — whether or not you consulted it for the task — when:

- **something was decided:** architecture, a decision or a convention, agreed in chat or chosen during implementation. Record the outcome (what, why, rejected alternatives), not the discussion.
- **something had to be learned:** through questions to the user, research, or trial and error. Record the working solution and the gotchas.
- **the knowledge base was wrong or outdated:** fix it.

Do it once the decision is made or the problem is solved, at the latest before you finish; keep "update knowledge base" on your task list as a reminder. Skip one-off details, anything a capable model already knows or the code shows, and anything the user marked as confidential. Do it without asking; tell the user in one line what you changed, with the pull request link or commit.

| Change | Rule |
|---|---|
| Fix or extend files in an existing topic | on your own |
| Add a file to an existing topic | on your own, after checking for duplicates |
| Create, split, merge or rename a topic | only when the user asks — never propose it |

Before writing, read [.knowledge-base/CONTRIBUTING.md](.knowledge-base/CONTRIBUTING.md).

## Maintenance (only when the user asks)

- Set up a knowledge base from the template — as its own repository or embedded in a project: [.knowledge-base/INIT.md](.knowledge-base/INIT.md)
- Use a knowledge base or topic permanently in a project: [.knowledge-base/CONNECT.md](.knowledge-base/CONNECT.md)
- Upgrade to a newer template version: [.knowledge-base/UPGRADE.md](.knowledge-base/UPGRADE.md)
- Work on the template itself: [.knowledge-base/TEMPLATE-DEVELOPMENT.md](.knowledge-base/TEMPLATE-DEVELOPMENT.md)
