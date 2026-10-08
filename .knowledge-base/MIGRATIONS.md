# Migrations

Steps an agent applies to a knowledge base during an upgrade, in addition to the file replacement done by `manage.py upgrade`. One `## <version>` section per template version (headings are parsed by the script). Write "None." when a version needs no steps.

## 0.8.0

None. Reading agents refuse content that reaches beyond their task and remove it as a security fix (`Security:` title, `Suspicious-Commit` trailer).

## 0.7.0

None. Contributions carry an `Agent-Model` trailer in commit messages and pull request descriptions.

## 0.6.0

None. Contribution rules now cover writing for other readers and handling sensitive data.

## 0.5.0

Only for embedded knowledge bases (`contribution = "with-project"`); otherwise none. Projects that use a shared knowledge base update their pointer themselves (see Projects in `AGENTS.md`).

1. Run `python <folder>/.knowledge-base/manage.py embed`. It prints the new "## Agent knowledge" section, which lists knowledge bases as entries.
2. In the project's `AGENTS.md` or `CLAUDE.md`, replace the text of the "## Agent knowledge" section with the printed text and entry, keeping entries of other knowledge bases; add the section if missing and the user agrees.

## 0.4.0

Only for embedded knowledge bases (`contribution = "with-project"`); otherwise none.

1. Run `python <folder>/.knowledge-base/manage.py embed`. It creates the starter topics `project` and `learnings` if missing and prints the "## Agent knowledge" section for the project.
2. Replace the "## Agent knowledge" section in the project's `AGENTS.md` or `CLAUDE.md` with the printed one; add it if missing and the user agrees.

## 0.3.0

No steps. New: knowledge bases embedded in a project (`manage.py install`, `contribution = "with-project"`).

## 0.2.0

Works with any git server; template updates are detected via version tags.

1. In `knowledge-base.toml`, add below `repository` (without it, `pull-request` is assumed):
   ```toml
   # How agents publish changes:
   #   "pull-request" - pull requests; auto-merged for trusted_authors (GitHub), otherwise reviewed
   #   "direct-push"  - push to main; the git server's access rights decide who may write
   contribution = "pull-request"
   ```
   Use `"direct-push"` if the knowledge base is not on GitHub or the user prefers pushing to `main`.

## 0.1.0

Initial version. None.
