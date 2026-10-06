# Quick Start

## Prerequisites

- Docker
- (Optional) `jq` for pretty JSON output

## Run locally

```mermaid
flowchart LR
  A["cp .env.example .env"] --> B["docker compose up --build"]
  B --> C["open http://127.0.0.1:8080"]
```

1. Copy the environment file and adjust values:

   ```bash
   cp codebase/.env.example codebase/.env
   ```

2. Build and start the service:

   ```bash
   docker compose --env-file codebase/.env --project-directory codebase up --build
   ```

3. Open <http://127.0.0.1:8080> in a browser.

## Stop

```bash
docker compose --env-file codebase/.env --project-directory codebase down
```

## Run the OpenCode sandbox

The sandbox is an OpenCode **web UI** connected to the HKO AI model (`zai-org/GLM-5.2-FP8`). It runs as a daemon alongside the app service and publishes its web interface on host port `OPENCODE_PORT` (default `61211`; the container uses opencode's default internal port).

1. Ensure `HKOAI_API_KEY` is set in `codebase/.env` (local-only — see `codebase/.env.example`).

2. Start the sandbox (and app) with Docker Compose:

   ```bash
   docker compose --env-file codebase/.env --project-directory codebase up -d
   ```

3. Open <http://127.0.0.1:61211> in a browser.

   The service passes `HKOAI_API_KEY` from `.env`.

To run only the sandbox:

```bash
docker compose --env-file codebase/.env --project-directory codebase up -d opencode
```

## CI/CD setup (PlantUML)

```plantuml
@startuml
!theme plain
skinparam actorStyle awesome

actor Developer as dev
participant "git push" as git
participant "release" as rls
participant "fast_checks" as fc
participant "security_checks" as sec
participant "promote" as prom
participant "pages" as pg
participant "wiki" as wk
participant "sonar_baseline" as sb

dev -> git : commit to dev-001
git -> rls : push
rls -> rls : bump version + CHANGELOG
rls -> fc : (needs release)
fc -> fc : compose + build + lint
fc -> sec : (needs fast_checks)
sec -> sec : CodeQL + SonarQube gate
sec -> prom : (needs security_checks)
prom -> prom : git push dev-001 HEAD → dev (GITHUB_TOKEN)
prom -> prom : git push dev-001 HEAD → main (GITHUB_TOKEN)
prom -> pg : (needs promote, parallel)
prom -> wk : (needs promote, parallel)
prom -> sb : (needs promote, parallel)
pg -> pg : checkout main, build Vite app
pg -> dev : deploy to GitHub Pages
wk -> wk : checkout main, sync docbase/
wk -> dev : publish to GitHub Wiki
sb -> sb : checkout main, informational scan
@enduml
```
