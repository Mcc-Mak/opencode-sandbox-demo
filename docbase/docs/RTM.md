# Requirements Traceability Matrix (RTM)

Maps each requirement to its implementation and tests.

| Req ID | Requirement | Implemented in | Verified by |
| --- | --- | --- | --- |
| FR-001 | Seven-step workflow | `AGENTS.md` § Workflow | Manual: confirm `AGENTS.md` lists 7 steps in order. |
| FR-002 | Plan & Track: backlog / items / tasklist | `AGENTS.md` § Workflow step 2 | Manual: confirm step 2 defines three separation-of-concerns. |
| FR-003 | Code/docs separation | `codebase/`, `docbase/` directory structure | Manual: confirm no docs in `codebase/` and no code in `docbase/`. |
| FR-004 | Conventional-commit versioning | `.github/workflows/ci-cd.yml` `release` job | CI: push a `feat:` commit, confirm minor bump in `CHANGELOG.md`. |
| FR-005 | CHANGELOG entry + release commit | `.github/workflows/ci-cd.yml` `release` job | CI: confirm `chore(release): X.Y.Z` commit appears after push. |
| FR-006 | Direct-push promotion `dev-001 → dev → main` | `.github/workflows/ci-cd.yml` `promote` job | CI: confirm `promote` job pushes dev-001 HEAD directly to `dev` and `main` via `GITHUB_TOKEN`. |
| FR-007 | Fast checks (compose, build, lint) | `.github/workflows/ci-cd.yml` `fast_checks` job | CI: confirm job runs and gates promotion (before `promote`). |
| FR-008 | CodeQL SAST hard gate | `.github/workflows/ci-cd.yml` `security_checks` job | CI: confirm CodeQL runs and blocks promotion on findings. |
| FR-009 | SonarQube Cloud (optional, fail-closed) | `.github/workflows/ci-cd.yml` `security_checks` job | CI: with `SONAR_TOKEN` → runs and fails closed; without → skips with notice. |
| FR-010 | GitHub Pages deployment | `.github/workflows/ci-cd.yml` `pages` job | CI: after promote pushes to `main`, confirm app live at Pages URL. |
| FR-011 | GitHub Wiki deployment | `.github/workflows/ci-cd.yml` `wiki` job | CI: after promote pushes to `main`, confirm `docbase/` pages on Wiki. |
| FR-012 | Non-blocking wiki when uninitialized | `.github/workflows/ci-cd.yml` `wiki` job | CI: delete wiki, push to dev-001, confirm pipeline succeeds with warning. |
| FR-013 | Secret auto-configuration | `scripts/configure-secrets.sh` | Manual: run script, confirm secrets appear in GitHub Settings → Secrets. |
| FR-014 | Multi-stage Dockerfile | `codebase/Dockerfile` | Manual: `docker compose up --build`, confirm app served on port 8080. |
| FR-015 | Non-root nginx container | `codebase/Dockerfile` (`USER nginx`), `codebase/nginx.conf` | Manual: `docker compose up --build`, confirm nginx process runs as `nginx` user (`docker exec <container> ps aux`). |
| FR-016 | `npm ci`/`npm install --ignore-scripts` | `codebase/Dockerfile`, `codebase/Dockerfile.opencode`, `.github/workflows/ci-cd.yml` | CI: confirm build succeeds with `--ignore-scripts` flag in logs. |
| FR-017 | SHA-pinned third-party actions | `.github/workflows/ci-cd.yml` | CI: confirm SonarSource action uses full SHA, not `@v8`. |
| FR-018 | Fail-closed SonarQube QG | `.github/workflows/ci-cd.yml` `security_checks` job | CI: with `SONAR_TOKEN` and ERROR status, confirm pipeline fails. |
| FR-019 | OpenCode sandbox (HKO AI model) | `codebase/Dockerfile.opencode`, `codebase/docker-compose.yml` `opencode` service, `opencode.json` | Manual: `docker compose up`, confirm web UI at `http://127.0.0.1:61211`. |
| FR-020 | SonarCloud main-branch baseline | `.github/workflows/ci-cd.yml` `sonar_baseline` job | CI: after promote, confirm informational SonarCloud scan runs on `main` (non-blocking). |
| NFR-001 | Identical `dist/` (container vs Pages) | `codebase/Dockerfile` + `pages` job | Manual: compare `dist/` hash from container build and Pages artifact. |
| NFR-002 | No secrets committed | `.gitignore` | CI: confirm `.env` is gitignored; scan history for leaked tokens. |
| NFR-003 | Direct-push promotion (`GITHUB_TOKEN`); `PROMOTE_TOKEN` for wiki only | `.github/workflows/ci-cd.yml` `promote` job (GITHUB_TOKEN), `wiki` job (PROMOTE_TOKEN) | CI: confirm `promote` pushes directly via `GITHUB_TOKEN`; confirm `wiki` uses `PROMOTE_TOKEN` for wiki repo push. |
| NFR-004 | Code/docs separation | Directory structure | Manual: same as FR-003. |
| NFR-005 | Parallel pages + wiki + sonar_baseline | `.github/workflows/ci-cd.yml` job dependencies | CI: confirm `pages`, `wiki`, and `sonar_baseline` run concurrently after `promote`. |
| NFR-006 | Git tags for releases | `git tag` | Manual: `git tag -l` confirms annotated tags for each version. |
| NFR-007 | Replaceable `codebase/` | Template structure | Manual: replace `codebase/`, confirm pipeline still works. |
| NFR-008 | Non-root containers | `codebase/Dockerfile` (`USER nginx`), `codebase/Dockerfile.opencode` (`USER node`) | Manual: `docker exec <app-container> whoami` returns `nginx`; `docker exec <opencode-container> whoami` returns `node`. |
| NFR-009 | `HKOAI_API_KEY` local-only | `codebase/.env.example`, `codebase/.env` (gitignored), `opencode.json` (`{env:HKOAI_API_KEY}`) | Manual: confirm `.env` is gitignored; confirm key absent from CI secrets and GitHub Secrets. |
