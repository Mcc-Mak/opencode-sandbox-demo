# AGENTS.md

Compact guidance for OpenCode sessions working in this repo. Read before editing.

## Repository purpose

A Docker project that builds an **OpenCode sandbox** connecting to the HKO AI model — `zai-org/GLM-5.2-FP8` served by the HKO LiteLLM gateway (`https://litellm.services.hko.gov.hk`). The sandbox is configured by `opencode.json` (repo root), which wires the `hko` provider (OpenAI-compatible via `@ai-sdk/openai-compatible`) and reads the API key from `HKOAI_API_KEY`. The key is **local-only** — set it in `codebase/.env` (see `codebase/.env.example`); it is never pushed to CI or GitHub secrets.

The repo retains its reusable infrastructure: a seven-step coding workflow, a progressive GitHub Actions CI/CD pipeline, strict semantic versioning, and a sample React + Vite app (`codebase/site/`) deployed to GitHub Pages. Implementation lives in `codebase/`; `AGENTS.md`, `opencode.json`, `docbase/`, the workflow, and versioning are reusable infrastructure.

## Layout

Code and docs are strictly separated. Do not mix them.

- `codebase/` — all implementation (project-specific; replace per project)
  - `.env.example` — every env var (PORT, NIC, image tag, `HKOAI_API_KEY`, etc.) with safe defaults
  - `docker-compose.yml` — two services: `app` (Vite→nginx daemon on `PORT`) and `opencode` (web UI daemon on `OPENCODE_PORT`/61211, container uses opencode's default internal port). Reads env for ports, NIC, and `HKOAI_API_KEY`.
  - `Dockerfile` / `Dockerfile.*` — the default `Dockerfile` is multi-stage: a `node` stage builds the Vite app, an `nginx` stage serves `dist/`. `Dockerfile.opencode` builds the OpenCode sandbox image (`node:24-alpine` + `opencode-ai` + `git`).
  - `site/` — React + Vite application, built and deployed to GitHub Pages on push to `main` (the same `dist/` is produced by the multi-stage `Dockerfile`, so container and Pages deploy an identical artifact)
- `docbase/` — all documentation (markdown only; published to the GitHub Wiki on push to `main`)
  - `TOCTREE.md` — index linking every doc below (also drives the wiki `TOCTREE` page and `_Sidebar.md`)
  - `docs/SRS.md`, `Architecture.md`, `QuickStart.md`,
    `Configurations.md`, `CICD-Pipeline.md`, `RTM.md`, `CRM.md`
  - **CRM** = Cross-Reference Matrix (maps requirements → docs → tests). Keep it updated when requirements change.
- `opencode.json` (repo root) — OpenCode sandbox config: defines the `hko` provider (OpenAI-compatible → LiteLLM), pins `zai-org/GLM-5.2-FP8`, loads `AGENTS.md` as instructions, and grants `*` permissions. The API key is injected from `HKOAI_API_KEY`.
- `CHANGELOG.md` (repo root) — one entry per change, versioned `major.minor.patch`
- `.env.example` (repo root) — documents CI/CD secrets (`PROMOTE_TOKEN`, `SONAR_TOKEN`); copy to gitignored `.env` and run `scripts/configure-secrets.sh`
- `scripts/configure-secrets.sh` — pushes local `.env` secret values into GitHub's encrypted store and enables Pages
- `.github/workflows/*.yml` — CI/CD pipelines

When adding a doc, also add it to `docbase/TOCTREE.md`. When adding an env var, also add it to `codebase/.env.example`.

## Workflow (follow in order)

Every task follows these 7 steps, in order:

1. **User Requirement** — capture the ask before coding.
2. **Plan & Track** — act as project manager before implementing. Maintain three distinct concerns, each at its own level of granularity. These are a separation of **concerns** (distinct roles, granularity, and lifecycles), not a separation of **actions** (a temporal sequence) — a backlog is not an item, an item is not a tasklist:
   - **Backlog** — the portfolio of work: milestones, labels, tags, and the project board, reflecting version history and roadmap. Manage via `gh`.
   - **Items** — one GitHub issue per distinct unit of work, with acceptance criteria and a verifiable state (open/closed). Open new issues for work discovered during planning; close issues only when the work is verifiably complete.
   - **Tasklist** — the granular checklist of steps within the current session, tracking execution progress per item.
   
   **Never perform a destructive action** (closing or reopening issues, deleting labels/milestones/tags, removing board items) without explicit confirmation — pause and ask first.
3. **Implement** into `codebase/{.env.example,docker-compose.yml,Dockerfile.*}`. Ports and NIC come from `.env`.
4. **Document** into `docbase/` (all files listed above, TOCTREE updated). CRM must reflect new/changed requirements.
5. **Update `CHANGELOG.md`** — add entry with version `major.minor.patch`.
6. **Git** — commit to branch `dev-001`:
   - Subject includes the version: `major.minor.patch` (e.g. `<scope>: ... (0.1.0)`)
   - Body explains the what and why
   - Push `dev-001` to `origin/dev-001`. Do **not** push to `dev` or `main` manually.
7. **CI/CD** runs automatically (see below).

Do not skip steps 4–5. Implementation without docs + changelog is incomplete.

## Git & branching

- Working branch is always **`dev-001`**. Commit and push there only.
- Promotion is automated by GitHub Actions, not manual:
  `origin/dev-001` → `origin/dev` → `origin/main` → **GitHub Pages** (app) + **GitHub Wiki** (docs)
- Never force-push. Never commit directly to `main` or `dev`.

## CI/CD pipelines (`.github/workflows/*.yml`)

A single workflow run on every `dev-001` push (plus `workflow_dispatch`). All jobs chain in one run — no PR-triggered gate runs, no separate main-branch runs:

1. **`release`** — bump version and update `CHANGELOG.md` from conventional commits.
2. **`fast_checks`** (needs: release) — validate compose, build the image, lint. Gate.
3. **`security_checks`** (needs: fast_checks) — **CodeQL** (SAST, mandatory hard gate) + **SonarQube Cloud** (quality gate, fail-closed when `SONAR_TOKEN` is configured; skipped with a notice when absent).
4. **`promote`** (needs: security_checks) — direct `git push` of the dev-001 HEAD to `dev` and `main` using `GITHUB_TOKEN`. No PRs; gate checks already ran in steps 2–3. `--force-with-lease` handles stale merge commits from the previous PR-based flow.
5. **`pages`** + **`wiki`** + **`sonar_baseline`** (needs: promote, parallel) — check out `main` (the promoted commit): build the React + Vite app and deploy to Pages; sync `docbase/` to the GitHub Wiki (uses `PROMOTE_TOKEN` for the wiki repo push); run an informational SonarCloud scan to establish the main-branch baseline.

`GITHUB_TOKEN` pushes do not trigger new workflow runs (GitHub security feature), which is exactly what we want — the entire pipeline is one run. GitHub does not create the `.wiki.git` repo until the first page is saved through the web UI; until then the `wiki` job warns and exits 0 (non-blocking).

When editing workflows, preserve the job chain (release → fast_checks → security_checks → promote → pages/wiki/sonar_baseline) and the CodeQL hard gate. SonarQube Cloud runs (and fails closed) when `SONAR_TOKEN` is configured; it is skipped when absent.

## Conventions

- Versioning is strict `major.minor.patch`; bump per CHANGELOG entry.
- `.env.example` is the source of truth for configurable knobs. Real `.env` is never committed.
- `opencode.json` references the API key via `{env:HKOAI_API_KEY}` — never hardcode a key in it. `HKOAI_API_KEY` is **local-only**: set it in `codebase/.env`, never in the root `.env`, and it is never pushed to CI or GitHub secrets.
- Prefer editing existing files over creating new ones; only create files listed in the Layout section.
