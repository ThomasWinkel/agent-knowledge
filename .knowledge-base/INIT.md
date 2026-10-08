# Set up a knowledge base

Two cases:

- **Own repository:** a knowledge base shared across projects, e.g. for internal APIs and processes. Target: the root of an (empty) repository.
- **Embedded in a project:** knowledge specific to one project, versioned and reviewed with its code. Target: a folder in the project repository, by default `agent-knowledge`.

## Steps

1. **Get the template** (skip if the user created the repository with GitHub's "Use this template" and you work in its clone — `knowledge-base.toml` contains `is_template = true`):
   - Find the latest release: `git ls-remote --tags --refs <template-url>`, take the highest `vMAJOR.MINOR.PATCH`.
   - `git clone --depth 1 --branch <tag> <template-url> <scratchpad>/template`
2. **Ask the user once**, proposing defaults so a short "ok" suffices:
   - name and one-sentence purpose (embedded: e.g. "<Project> knowledge" / "Project knowledge for agents working on <project>");
   - embedded: target folder (`agent-knowledge`), and consent to add a pointer to the project's agent instructions and, if the project has CI, a lint step;
   - own repository on GitHub with pull requests: GitHub logins whose PRs may be merged automatically (default: the repository owner; `Copilot` for PRs of the Copilot coding agent).
3. **Install:**
   ```sh
   python <scratchpad>/template/.knowledge-base/manage.py install --target <target> --name "<name>" --description "<purpose>" [--trusted-author <login> ...]
   ```
   With "Use this template" run `python .knowledge-base/manage.py init` with the same options instead.
   It copies the template, writes `knowledge-base.toml` and `README.md`, removes the example topic and generates `index.md`. It detects embedding (target is not the repository root) and chooses `contribution`: `with-project` when embedded, `pull-request` on GitHub, `direct-push` elsewhere (override with `--contribution`). Embedded, it skips the GitHub workflows and `LICENSE`. Existing `LICENSE`, `.gitignore` and `.gitattributes` are kept.
4. **Embedded only:** `init` also creates the starter topics `project` (what was decided) and `learnings` (what had to be learned) and prints two snippets with the actual folder (again with `python <target>/.knowledge-base/manage.py embed`). If the user agreed:
   - Add the "## Agent knowledge" section to the project's instructions, following steps 3–4 of [CONNECT.md](CONNECT.md).
   - Add the lint step to the project's CI.
5. Run `python <target>/.knowledge-base/manage.py lint`.
6. **Commit:**
   - Own repository: commit ("Initialize knowledge base") and push to `main`.
   - Embedded: commit as a normal project change and publish it the way the project handles changes.
7. **Tell the user**, briefly:
   - `pull-request` on GitHub: auto-merge uses the workflow token. It fails if branch protection on `main` requires reviews, or if the organization restricts workflow permissions to read-only (Settings → Actions → General → Workflow permissions). A weekly workflow opens an issue when a new template version is available.
   - `direct-push`: whoever may push to `main` may change topics; agents push other changes as branches for review. The GitHub workflow files are inert on other servers.
   - `with-project`: knowledge changes arrive with the code changes of each task.
   - Own repository: `LICENSE` is the template's MIT license; adjust holder or license for the content if needed. To use it in a project, link it for a single task, or ask an agent to include it (or some topics) permanently in the project or globally.
   - New topics: ask an agent to create them. Agents mention new template versions after contributing.
