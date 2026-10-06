# Architecture

## Overview

This repository builds an **OpenCode sandbox** that connects to the HKO AI model (`zai-org/GLM-5.2-FP8` via LiteLLM), packaged as a Docker image. It also retains reusable infrastructure: a sample React + Vite app deployed to GitHub Pages, documentation published to the GitHub Wiki, and a progressive CI/CD pipeline. It separates **implementation** (`codebase/`) from **documentation** (`docbase/`) and **CI/CD infrastructure** (`.github/`). The pipeline progressively promotes code through three branch gates before deploying the Vite application to GitHub Pages and documentation to the GitHub Wiki.

## System structure (Mermaid)

```mermaid
flowchart TB
  subgraph repo["Repository"]
    subgraph codebase["codebase/ (project-specific)"]
      ENV[".env.example"]
      DC["docker-compose.yml"]
      DF["Dockerfile (multi-stage)"]
      DFOC["Dockerfile.opencode"]
      APP["site/ (React + Vite app)"]
    end
    subgraph docbase["docbase/ (documentation, markdown only)"]
      DOCS["docs/*.md (7 docs)"]
      TOC["TOCTREE.md"]
    end
    subgraph cicd[".github/workflows/"]
      WF["ci-cd.yml (6 jobs)"]
    end
    AGENTS["AGENTS.md"]
    OCJSON["opencode.json"]
    CL["CHANGELOG.md"]
  end

  ENV --> DC
  DC --> DF
  DC --> DFOC
  DF --> APP
  DFOC -->|uses| OCJSON
  TOC --> DOCS
  WF -->|promotes| repo
```

## Components (PlantUML)

```plantuml
@startuml
!theme plain
skinparam componentStyle rectangle

package "codebase/" {
  [Dockerfile] as df
  [Dockerfile.opencode] as dfoc
  [docker-compose.yml] as dc
  [.env.example] as env
  [site/ (Vite app)] as app
}

package "docbase/" {
  [docs/*.md] as docs
  [TOCTREE.md] as toc
}

package ".github/workflows/" {
  [ci-cd.yml] as wf
}

[AGENTS.md] as agents
[opencode.json] as ocjson
[CHANGELOG.md] as cl

env --> dc : ports + NIC + HKOAI_API_KEY
dc --> df : build (app)
dc --> dfoc : build (opencode)
df --> app : npm build → nginx serve
dfoc --> ocjson : uses
toc --> docs : index
wf --> cl : version bump
@enduml
```

- **App service** — containerised application defined by the multi-stage `Dockerfile` (a `node` stage builds the Vite app, an `nginx` stage serves `dist/` as the non-root `nginx` user via a custom `nginx.conf` on port 8080) + `docker-compose.yml`; ports and NIC come from `.env`.
- **OpenCode sandbox** — web UI daemon defined by `Dockerfile.opencode` (`node:24-alpine` + `opencode-ai@1.18.34` + `git`), running `opencode web --hostname 0.0.0.0` (opencode's default internal port) in `docker-compose.yml`. `HKOAI_API_KEY` is passed from `.env` (local-only). Publishes the web interface on host port `OPENCODE_PORT` (default `61211`). Starts alongside the app service with `docker compose up`.
- **App site** — React + Vite SPA in `codebase/site/`, deployed to GitHub Pages. The container and Pages ship an identical `dist/` artifact.
- **Documentation** — markdown-only `docbase/`, published to the GitHub Wiki by the `wiki` job.
- **Pipeline** — six-job workflow (`release → fast_checks → promote → security_checks → pages + wiki`) that progressively promotes code from `dev-001` to GitHub Pages (app) and the GitHub Wiki (docs).

## CI/CD data flow (Mermaid)

```mermaid
flowchart LR
  A["dev-001 push"] --> B["release\nversion bump"]
  B --> C["fast_checks\ncompose + build + lint"]
  C --> D["promote\nPR dev-001→dev"]
  D -->|gate: Fast Checks| E["merge to dev"]
  E --> F["PR dev→main"]
  F -->|gate: Security & Quality| G["merge to main"]
  G --> H["pages\nbuild + deploy"]
  G --> W["wiki\nsync docbase/"]
  H --> I["GitHub Pages (app)"]
  W --> J["GitHub Wiki (docs)"]
```

## Deployment

The system deploys in four ways:

1. **App container** — `docker compose up --build` from `codebase/`; the multi-stage `Dockerfile` builds the Vite app and serves `dist/` from nginx, binding to the host port and NIC defined in `.env`.
2. **OpenCode sandbox** — `docker compose up` from `codebase/`; builds `Dockerfile.opencode` and starts the OpenCode web UI (`opencode web --hostname 0.0.0.0`) connected to the HKO AI model. Published on host port `OPENCODE_PORT` (default `61211`). Local-only (`HKOAI_API_KEY` in `.env`).
3. **Application** — GitHub Actions builds the Vite app in `codebase/site/` and deploys the static artifact to GitHub Pages on every push to `main`. Identical `dist/` to the container.
4. **Documentation** — GitHub Actions syncs `docbase/` markdown to the GitHub Wiki on every push to `main` (parallel with Pages).

### Deployment flow (PlantUML)

```plantuml
@startuml
!theme plain

artifact "Docker Image" as image
artifact "Sandbox Image" as sbimage
folder "Container" as container
folder "OpenCode web UI" as sandbox
cloud "GitHub Pages" as pages
cloud "GitHub Wiki" as wiki
folder "codebase/site/" as source
folder "docbase/" as docs

source --> image : build (Dockerfile)
image --> container : docker compose up
sbimage --> sandbox : docker compose up
source --> pages : build (Vite) + deploy (Actions)
docs --> wiki : sync (Actions)

note right of container
  Binds to PORT and NIC
  from .env
end note

note right of sandbox
  HKOAI_API_KEY from .env
  Web UI on OPENCODE_PORT
  Local-only, daemon
end note

note right of pages
  base: /opencode-workflow-demo/
  Triggered on push to main
end note

note right of wiki
  Reuses PROMOTE_TOKEN
  Needs one-time UI init
  Non-blocking until then
end note
@enduml
```
