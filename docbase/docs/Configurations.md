# Configurations

All configurable knobs live in `codebase/.env.example`. Copy it to `codebase/.env` and adjust. Real `.env` files are never committed.

## Application variables

| Variable | Default | Description |
| --- | --- | --- |
| `PORT` | `8080` | Host port mapped to the container. |
| `NIC` | `127.0.0.1` | Network interface the port binds to. Use `0.0.0.0` to expose publicly. |
| `IMAGE_TAG` | `latest` | Tag applied to the built image. |
| `HKOAI_API_KEY` | _(empty)_ | API key for the HKO AI model (`zai-org/GLM-5.2-FP8`) via the LiteLLM gateway. Read by `opencode.json` as `{env:HKOAI_API_KEY}`. Never commit a real value. |
| `OPENCODE_PORT` | `61211` | Host port for the OpenCode web UI (container uses opencode's default internal port). |

## CI/CD secrets

These are stored as encrypted **repository secrets** (GitHub-hosted runners cannot read a local `.env` at runtime). Configure them from a local `.env` via the auto-config script:

```bash
cp .env.example .env          # fill in real token values
./scripts/configure-secrets.sh
```

The script pushes each value into GitHub's secret store with `gh secret set`, enables GitHub Pages (Source = GitHub Actions), and registers `dev-001` as a deployment branch for the `github-pages` environment. Real `.env` is gitignored.

### Token secrets

| Secret | Used by | Description |
| --- | --- | --- |
| `GIT_PUSH_TOKEN` | release | PAT for pushing release commits to `dev-001`. Scopes: `repo`, `workflow`. |
| `PROMOTE_TOKEN` | promote, wiki | PAT that creates/merges promotion PRs (GITHUB_TOKEN PRs do not trigger checks) and pushes `docbase/` to the GitHub Wiki. Scopes: `repo`, `workflow`. |
| `SONAR_TOKEN` | security gate | SonarQube Cloud analysis token. Optional — when absent, the dev→main gate runs CodeQL-only SAST; when present, Sonar runs and fails closed on its quality gate. |

### Notification secrets

| Secret | Used by | Description |
| --- | --- | --- |
| `NOTIFICATION_ADDRESS` | pages | Recipient email for deployment notifications. |
| `NOTIFICATION_HEADER` | pages | Email subject header (e.g. `GitHub - [HKO] opencode-workflow-demo`). |
| `NOTIFICATION_ACTIVE` | pages | `"true"` to enable notifications, `"false"` to disable. Currently `false`. |

### Secrets configuration flow (Mermaid)

```mermaid
flowchart TB
  A[".env.example (template)"] -->|cp| B[".env (filled in)"]
  B --> C["configure-secrets.sh"]
  C -->|gh secret set| D["GitHub Secrets"]
  C -->|gh api| E["Pages: Source = GitHub Actions"]
  C -->|gh api| F["Env github-pages: dev-001 branch"]
  D --> G["CI/CD workflow reads secrets at runtime"]
```

## OpenCode sandbox

This repo's primary deliverable is an **OpenCode sandbox**: an OpenCode agent that connects to the HKO AI model. It is configured by `opencode.json` at the repo root:

- **Provider `hko`** — OpenAI-compatible (`@ai-sdk/openai-compatible`) pointed at the HKO LiteLLM gateway `https://litellm.services.hko.gov.hk`.
- **Model** — `zai-org/GLM-5.2-FP8` (100k context / 100k output).
- **API key** — injected from the `HKOAI_API_KEY` environment variable (`{env:HKOAI_API_KEY}`); never hardcoded in `opencode.json`. **Local-only**: set it in `codebase/.env` (see `codebase/.env.example`); it is never pushed to CI or GitHub secrets.
- **Instructions** — `AGENTS.md` is loaded as project instructions.
- **Permissions** — `*` allow (full tool access inside the sandbox).

### Prerequisites

1. Copy `codebase/.env.example` to `codebase/.env` and fill in `HKOAI_API_KEY`:

   ```bash
   cd codebase
   cp .env.example .env
   # edit .env → HKOAI_API_KEY=<your-key>
   ```

2. Build the sandbox image (`codebase/Dockerfile.opencode` — `node:24-alpine` + `opencode-ai@1.18.34` + `git`):

   ```bash
   docker compose build opencode
   ```

### Running the sandbox

The `opencode` service is a **web UI daemon** that starts alongside `app` with `docker compose up`. It runs `opencode web --hostname 0.0.0.0` (using opencode's default internal port), publishing the web interface on the host port defined by `OPENCODE_PORT` (default `61211`):

```bash
cd codebase
docker compose up -d
```

Open <http://127.0.0.1:61211> in a browser. The service passes `HKOAI_API_KEY` from `.env`.

To run only the sandbox (without the app):

```bash
docker compose up -d opencode
```

`HKOAI_API_KEY` stays local — it is not among the CI/CD secrets pushed by `scripts/configure-secrets.sh`.

## Application site

The React + Vite application (`codebase/site/`) is deployed to GitHub Pages at the path matching the repository name (`/opencode-workflow-demo/`). If you rename the repo, update `base` in `codebase/site/vite.config.ts`.

The multi-stage `codebase/Dockerfile` runs the same Vite build (`npm ci --ignore-scripts && npm run build`) in a `node` stage and serves the resulting `dist/` from an `nginx` stage. The nginx stage uses a custom `codebase/nginx.conf` (listening on port 8080) and runs as the non-root `nginx` user (`USER nginx`). The container and the Pages deploy therefore ship an identical artifact.

## Documentation (GitHub Wiki)

`docbase/` is markdown-only and is published to the repository's GitHub Wiki by the `wiki` job on push to `main`.

- The `wiki` job clones `{repo}.wiki.git`, copies `docbase/TOCTREE.md` and `docbase/docs/*.md` to the wiki root (GitHub Wiki serves pages by filename — subdirectories are not supported in URLs), generates a minimal `Home.md` and a `_Sidebar.md` (from `TOCTREE.md`), rewrites `docs/X.md` links to `X` for wiki resolution, and pushes with `--force-with-lease` (docbase is the source of truth).
- It reuses `PROMOTE_TOKEN` (its `repo` scope covers the wiki repo).
- **One-time prerequisite:** GitHub does not create the `.wiki.git` repo until the first page is saved through the web UI (there is no API to bootstrap it). Until then the `wiki` job prints a warning and exits 0 — non-blocking, does not fail the pipeline. Once initialized, subsequent runs sync normally.
- Wiki-side edits made through the browser are overwritten on the next sync; edit `docbase/` instead.
