# Software Requirements Specification (SRS)

## Functional requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-001 | The repository shall provide a seven-step workflow documented in `AGENTS.md`: User Requirement, Plan & Track, Implement, Document, CHANGELOG, Git, CI/CD. | Must |
| FR-002 | Step 2 (Plan & Track) shall maintain three separation-of-concerns: backlog (milestones, labels, board), items (one GitHub issue per unit of work), and tasklist (session execution checklist). | Must |
| FR-003 | Implementation shall reside in `codebase/` and documentation in `docbase/`; the two shall not be mixed. | Must |
| FR-004 | The `release` job shall parse conventional commit subjects and bump the version: `feat:` → minor, `fix:` → patch, `feat!:` / `BREAKING CHANGE:` → major. | Must |
| FR-005 | The `release` job shall prepend an entry to `CHANGELOG.md` and commit with `chore(release): X.Y.Z`. | Must |
| FR-006 | The `promote` job shall directly `git push` the `dev-001` HEAD to `dev` and `main` using `GITHUB_TOKEN`, gated by `fast_checks` and `security_checks` before it runs. No PRs are created. | Must |
| FR-007 | The `fast_checks` job shall validate `docker-compose.yml`, build the image, and run lint. It gates promotion (runs before the `promote` job). | Must |
| FR-008 | The `security_checks` job shall run CodeQL SAST as a mandatory pre-promotion hard gate (runs before the `promote` job). | Must |
| FR-009 | The `security_checks` job shall run SonarQube Cloud when `SONAR_TOKEN` is configured (fail-closed) and skip with a notice when absent. | Should |
| FR-010 | The `pages` job shall build the React + Vite application in `codebase/site/` and deploy to GitHub Pages after the `promote` job pushes to `main`. | Must |
| FR-011 | The `wiki` job shall sync `docbase/` markdown to the GitHub Wiki after the `promote` job pushes to `main`, running in parallel with `pages`. | Must |
| FR-012 | The `wiki` job shall warn and exit 0 when the wiki repository does not yet exist (non-blocking). | Must |
| FR-013 | `scripts/configure-secrets.sh` shall read a local `.env` and push values into GitHub's encrypted secret store, enable Pages source, and enable the Wiki feature. | Should |
| FR-014 | The Dockerfile shall be multi-stage: a `node` stage builds the Vite app, an `nginx` stage serves `dist/`. | Must |
| FR-015 | The runtime container shall run nginx as a non-root user (`USER nginx`) with a custom `nginx.conf` listening on port 8080. | Must |
| FR-016 | All `npm ci` and `npm install` invocations (Dockerfiles and CI workflows) shall use `--ignore-scripts` to prevent supply-chain script execution during dependency installation. | Must |
| FR-017 | Third-party GitHub Actions shall be pinned to a full commit SHA, not a floating tag (e.g. `@v8`). | Must |
| FR-018 | The SonarQube Cloud Quality Gate check shall fail-closed (`exit 1` on ERROR status) when `SONAR_TOKEN` is configured. | Must |
| FR-019 | The repository shall provide an OpenCode sandbox (`Dockerfile.opencode` + `docker-compose.yml` `opencode` service) connecting to the HKO AI model (`zai-org/GLM-5.2-FP8` via the LiteLLM gateway), with the web UI published on host port `OPENCODE_PORT` (default 61211) and `HKOAI_API_KEY` injected from `codebase/.env`. | Must |
| FR-020 | The `sonar_baseline` job shall run an informational SonarCloud scan on `main` (after promotion) to establish the main-branch baseline. It shall not fail-closed. | Should |

## Non-functional requirements

| ID | Requirement | Category |
| --- | --- | --- |
| NFR-001 | The container (`docker compose up`) and GitHub Pages shall deploy an identical `dist/` artifact. | Reproducibility |
| NFR-002 | No secrets shall be committed to the repository; `.env` is gitignored. | Security |
| NFR-003 | Promotion shall use direct `git push` with `GITHUB_TOKEN` (not PRs). `PROMOTE_TOKEN` (PAT) shall be used only for wiki repo pushes. `GITHUB_TOKEN` pushes do not trigger new workflow runs (by design). | Security |
| NFR-004 | Code and docs shall be strictly separated to enable independent deployment and replacement. | Maintainability |
| NFR-005 | The `pages`, `wiki`, and `sonar_baseline` jobs shall run in parallel after promotion to minimize deployment latency. | Performance |
| NFR-006 | Every released version shall have a git tag and a CHANGELOG entry for traceability. | Traceability |
| NFR-007 | `codebase/` shall be replaceable per project without affecting the reusable infrastructure (`AGENTS.md`, `docbase/`, pipeline, versioning). | Extensibility |
| NFR-008 | All containers (app via `USER nginx` and opencode sandbox via `USER node`) shall operate under least privilege (non-root user). | Security |
| NFR-009 | `HKOAI_API_KEY` shall be local-only (`codebase/.env`), never pushed to CI or GitHub secrets. | Security |

## Constraints

- GitHub is the only hosting platform (Actions, Pages, Wiki, Issues, Milestones).
- The working branch is always `dev-001`; direct pushes to `dev` or `main` are prohibited (the `promote` job handles promotion via `GITHUB_TOKEN`).
- The GitHub Wiki is not created until a user saves the first page through the web UI.
- The `PROMOTE_TOKEN` PAT requires `repo` scope (for wiki repo pushes only).

## Assumptions

- Consumers replace `codebase/` with their own implementation and update `docbase/docs/*.md` accordingly.
- The Vite app in `codebase/site/` is a sample that demonstrates the deployment path; it may be replaced entirely.
- GitHub Actions runners provide Node.js 24 and Docker support.
