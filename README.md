# Agent Knowledge

Shared knowledge for AI agents: internal processes, private APIs, conventions and verified recipes.

A knowledge base for AI agents (Claude Code, GitHub Copilot), created from
[agent-knowledge-template](https://github.com/ThomasWinkel/agent-knowledge-template). Agents read it to solve tasks that need knowledge
outside their training data, and keep it up to date themselves.

## Usage

Tell an agent, for example:

> Use the knowledge base https://github.com/ThomasWinkel/agent-knowledge, topic TOPIC. Read its AGENTS.md first.

Or share the link to a topic entry point: https://github.com/ThomasWinkel/agent-knowledge/blob/main/topics/TOPIC/index.md

A link is for the task at hand. To have agents consult a topic in every session of a project:

> Include topic TOPIC from https://github.com/ThomasWinkel/agent-knowledge in this project.

All topics are listed in [index.md](index.md). Rules for agents: [AGENTS.md](AGENTS.md).

## Maintenance

- **New topic:** ask an agent working in a clone of this repository to create it.
- **Changes:** pull requests by `trusted_authors` (see `knowledge-base.toml`) that only change
  topics are merged automatically once checks pass. Everything else needs a review.
- **Template updates:** a weekly workflow opens an issue when a new template version is available.
  Assign it to an agent (e.g. Copilot) or ask an agent to follow `.knowledge-base/UPGRADE.md`.
