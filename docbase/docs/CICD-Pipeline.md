# CI/CD Pipeline

## Branch model

```
dev-001 → dev → main → GitHub Pages (app) + GitHub Wiki (docs)
```

Working branch is always `dev-001`. Promotion is automated by GitHub Actions; never push directly to `dev` or `main`.

### Pipeline overview (Mermaid)

```mermaid
flowchart LR
  subgraph dev-001["dev-001 (working branch)"]
    PUSH["push to dev-001"]
    RELEASE["release\nbump version + CHANGELOG"]
    FAST["fast_checks\ncompose • build • lint"]
    SEC["security_checks\nCodeQL + SonarQube"]
    PROMOTE["promote\ndirect push dev-001→dev→main"]
  end
  subgraph main["main (release)"]
    PAGES["pages\nbuild + deploy Vite app"]
    WIKI["wiki\nsync docbase/ markdown"]
    SONAR["sonar_baseline\ninformational SonarCloud scan"]
  end

  PUSH --> RELEASE --> FAST --> SEC --> PROMOTE
  PROMOTE --> PAGES
  PROMOTE --> WIKI
  PROMOTE --> SONAR
```

### Job sequence (Mermaid)

```mermaid
sequenceDiagram
  participant dev001 as dev-001
  participant R as release
  participant F as fast_checks
  participant S as security_checks
  participant P as promote
  participant PG as pages
  participant WK as wiki
  participant SB as sonar_baseline

  dev001->>R: push (conventional commit)
  R->>R: bump version, update CHANGELOG
  R->>dev001: push release commit
  R->>F: (needs release)
  F->>F: validate compose, build, lint
  F->>S: (needs fast_checks)
  S->>S: CodeQL SAST + SonarQube quality gate
  S->>P: (needs security_checks)
  P->>P: git push dev-001 HEAD → dev
  P->>P: git push dev-001 HEAD → main
  P->>PG: (needs promote, parallel)
  PG->>PG: checkout main, build Vite app, deploy to Pages
  P->>WK: (needs promote, parallel)
  WK->>WK: checkout main, sync docbase/ to GitHub Wiki
  P->>SB: (needs promote, parallel)
  SB->>SB: checkout main, informational SonarCloud scan
```

## Stages

A single workflow run chains all jobs — no PR-triggered gate runs, no separate main-branch runs:

| Job | Needs | Purpose |
| --- | --- | --- |
| `release` | — | Bump version (`major.minor.patch`) from conventional commits and update `CHANGELOG.md`. |
| `fast_checks` | `release` | Validate compose, build the image, lint. Gate. |
| `security_checks` | `fast_checks` | CodeQL (SAST, mandatory hard gate) + SonarQube Cloud (quality gate, fail-closed when `SONAR_TOKEN` is configured; skipped with a notice when absent). Always fails closed on ERROR/NONE — this is a pre-promotion gate. |
| `promote` | `security_checks` | Direct `git push` of the dev-001 HEAD to `dev` and `main` using `GITHUB_TOKEN`. No PRs; gate checks already ran. `--force-with-lease` handles stale merge commits from the previous PR-based flow. |
| `pages` | `promote` | Checkout `main` (promoted commit), build the React + Vite application (`codebase/site/`) and deploy to GitHub Pages. |
| `wiki` | `promote` | Checkout `main`, sync `docbase/` markdown to the GitHub Wiki. Uses `PROMOTE_TOKEN` for the wiki repo push. Non-blocking: warns and skips if the wiki is not yet initialized (needs one-time UI page creation). |
| `sonar_baseline` | `promote` | Checkout `main`, run an informational SonarCloud scan with `GITHUB_REF` overridden to `refs/heads/main` to establish the main-branch baseline. No quality-gate check, no fail-closed. |

`GITHUB_TOKEN` pushes do not trigger new workflow runs (GitHub security feature), which is exactly what we want — the entire pipeline is one run. Previously the PR-based promotion flow triggered 4 separate workflow runs per dev-001 push; the consolidated pipeline runs once.

## Versioning

Versions are strict `major.minor.patch`. The `release` job parses commit subjects since the last `chore(release):` commit:

- `feat:` / `feat!:` → minor (or major if breaking)
- `fix:` → patch
- `BREAKING CHANGE:` in a body → major
- anything else → patch

### Version bump decision (Mermaid)

```mermaid
flowchart TD
  START([Parse commits since last release]) --> CHECK_BREAK{BREAKING CHANGE
  or feat!?}
  CHECK_BREAK -->|yes| MAJOR[major: X+1.0.0]
  CHECK_BREAK -->|no| CHECK_FEAT{any feat: ?}
  CHECK_FEAT -->|yes| MINOR[minor: X.Y+1.0]
  CHECK_FEAT -->|no| PATCH[patch: X.Y.Z+1]
  MAJOR --> WRITE[Prepend entry to CHANGELOG.md]
  MINOR --> WRITE
  PATCH --> WRITE
  WRITE --> COMMIT["chore(release): X.Y.Z"]
  COMMIT --> PUSH[push to dev-001]
```

## Required secrets

| Secret | Used by | Description |
| --- | --- | --- |
| `GITHUB_TOKEN` | release, promote | Built-in token. Release pushes the version commit to dev-001; promote pushes the dev-001 HEAD to dev and main. GITHUB_TOKEN pushes do not trigger new workflow runs (by design). |
| `PROMOTE_TOKEN` | wiki | PAT for pushing docbase/ to the GitHub Wiki (separate `.wiki.git` repo). Scopes: `repo`, `workflow`. |
| `SONAR_TOKEN` | security_checks, sonar_baseline | SonarQube Cloud analysis token. Optional — when absent, security_checks runs CodeQL-only SAST and sonar_baseline is skipped. |

## Repository settings

- Enable GitHub Pages: **Settings → Pages → Source: GitHub Actions**.
- Add `dev-001` and `main` as deployment branches in **Settings → Environments → github-pages**.
- Enable the GitHub Wiki feature: **Settings → General → Features → Wikis** (the `configure-secrets.sh` script does this).
- Initialize the GitHub Wiki (one-time): open **{repo}/wiki** in the browser and create the first page. GitHub does not create the `.wiki.git` repo until this is done. The `wiki` job warns and skips (non-blocking) until then.

### Deployment topology (PlantUML)

```plantuml
@startuml
!theme plain
skinparam componentStyle rectangle

cloud "GitHub" as GH {
  rectangle "dev-001\n(working branch)" as dev001
  rectangle "dev\n(staging)" as dev
  rectangle "main\n(release)" as main
  rectangle "GitHub Pages\n(Vite app)" as pages
  rectangle "GitHub Wiki\n(docbase markdown)" as wiki
}

rectangle "Single workflow run" as runner {
  rectangle "release" as rls
  rectangle "fast_checks" as fc
  rectangle "security_checks" as sec
  rectangle "promote\n(direct git push)" as prom
  rectangle "pages" as pg
  rectangle "wiki" as wk
  rectangle "sonar_baseline" as sb
}

dev001 --> rls : push
rls --> dev001 : release commit
rls --> fc : needs release
fc --> sec : needs fast_checks
sec --> prom : needs security_checks
prom --> dev : git push (GITHUB_TOKEN)
prom --> main : git push (GITHUB_TOKEN)
prom --> pg : needs promote (parallel)
prom --> wk : needs promote (parallel)
prom --> sb : needs promote (parallel)
pg --> pages : checkout main, deploy
wk --> wiki : checkout main, sync
@enduml
```
