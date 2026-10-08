# Upgrade to a newer template version

Triggered by the user, by an agent's hint after contributing, or on GitHub by the issue "Template update available" from the weekly workflow. Works the same for knowledge bases in their own repository and embedded in a project.

1. Start:
   - Own repository: `git fetch origin && git switch -c knowledge/template-upgrade origin/main`.
   - Embedded: create a branch the way the project handles changes.
2. Find the latest version: `python .knowledge-base/manage.py check-template` (in the knowledge base root).
3. Clone that release of the template (repository from `.knowledge-base/template.toml`), e.g. into your scratchpad:
   `git clone --depth 1 --branch v<version> <template-repository> <scratchpad>/template`
4. Run the **new** template's script against the knowledge base root (repository root or the project folder, e.g. `agent-knowledge`):
   ```sh
   python <scratchpad>/template/.knowledge-base/manage.py upgrade --target <knowledge-base-root>
   ```
   It replaces all template-owned files (`owned` in `template.toml`; when embedded, without `standalone_only`) and prints the migration steps from `MIGRATIONS.md` for every version in between.
5. Apply the printed migration steps in order.
6. `python .knowledge-base/manage.py index`, then `python .knowledge-base/manage.py lint`; fix all errors.
7. Review `git diff`. Local edits to template-owned files are overwritten by design; if any were lost, tell the user.
8. Commit ("Upgrade knowledge base template to <version>") and publish for review — never directly to `main`, because upgrades change agent instructions and tooling:
   - `pull-request`: open a PR (on GitHub referencing the issue: `Closes #<number>`); it is never merged automatically.
   - `direct-push`: push the branch (`git push -u origin HEAD`) and ask the user to review and merge it.
   - `with-project`: publish like any project change (e.g. a pull request).
