# Changelog

All notable changes to this project are documented here. Versions follow semver.

## 0.8.2 (2026-10-06)

### Fixed
- redirect error messages to stderr in configure-secrets.sh (S7677)

## 0.8.1 (2026-10-06)

### Fixed
- resolve SonarCloud S6471 + align all docs with consolidated pipeline (0.8.1)

## 0.8.0 (2026-10-06)

### Added
- consolidate CI/CD to single workflow run + fix Dockerfile S6505 (0.8.0)

## 0.7.6 (2026-10-06)

### Fixed
- bind-mount opencode.json read-only into sandbox container (0.7.6)

## 0.7.5 (2026-10-06)

### Fixed
- run security_checks on push to main for SonarCloud baseline (0.7.5)

## 0.7.4 (2026-10-06)

### Fixed
- narrow sonar sources and exclude coverage for demo app (0.7.3)

## 0.7.3 (2026-10-06)

### Fixed
- store regex in variable to avoid bash parser error (0.7.3)
- correct repo name references and release job regex (0.7.3)

## 0.7.2 (2026-10-06)

### Fixed
- correct Mermaid edge label syntax in Architecture.md (0.7.2)

## 0.7.1 (2026-10-06)

### Other
- docs: remove template boilerplate, keep 7 project-relevant docs (0.7.1)

## 0.7.0 (2026-10-06)

### Added
- add OpenCode sandbox connecting to HKO AI model (0.7.0)

### Other
- Initial commit

## 0.6.4 (2026-10-05)

### Other
- fix(security): resolve 4 SonarCloud vulnerabilities + fail-closed QG + branch attribution (0.5.4)

## 0.6.3 (2026-10-05)

### Other
- docs: fill template placeholders — Charter, Stories, PRD, SRS, RTM, CRM (0.5.3)

## 0.6.2 (2026-10-05)

### Fixed
- migrate to Node 24 — upload-pages-artifact v5, node-version 24 (0.5.2)

## 0.6.1 (2026-10-05)

### Other
- docs: reframe Step 2 as separation of concerns (backlog, items, tasklist) (0.5.1)

## 0.6.0 (2026-10-05)

### Added
- add Plan & Track step to workflow (seven-step, 0.4.5)

## 0.5.0 (2026-10-05)

### Added
- use README.md as wiki Home, add sonar.branch.name=main (0.4.4)

## 0.4.3 (2026-10-05)

### Fixed
- flatten wiki pages to root — GitHub Wiki ignores subdirectories (0.4.3)

## 0.4.2 (2026-10-05)

### Fixed
- make wiki job non-blocking when wiki is not initialized (0.4.2)

## 0.4.1 (2026-10-05)

### Fixed
- bootstrap wiki on first run instead of failing closed (0.4.1)

## 0.4.0 (2026-10-05)

### Added
- deploy codebase to Pages, docbase to Wiki (0.4.0)

## 0.3.2 (2026-10-05)

### Fixed
- manual tar to preserve dotfiles in Pages artifact (0.3.2)

## 0.3.1 (2026-10-05)

### Fixed
- add .nojekyll so Pages serves dotfiles like .env.example (0.3.1)

## 0.3.0 (2026-10-05)

### Added
- deploy docbase/ and codebase/ to GitHub Pages subpaths (0.3.0)

## 0.2.0 (2026-10-05)

### Added
- add notification secrets, GIT_PUSH_TOKEN, Mermaid/PlantUML docs (0.2.0)

## 0.1.6 (2026-10-05)

### Other
- fix(ci): close stale PRs before creating fresh ones (0.1.6)

## 0.1.5 (2026-10-05)

### Other
- fix(ci): use --watch then query named gate state (0.1.5)

## 0.1.4 (2026-10-05)

### Other
- fix(ci): non-blocking Sonar QG + targeted check watching (0.1.4)

## 0.1.3 (2026-10-05)

### Other
- fix(ci): make Sonar optional, keep CodeQL as hard gate (0.1.3)

## 0.1.2 (2026-10-05)

### Other
- feat(ci): add .env-driven secret auto-config script (0.2.0)

## 0.1.1 (2026-10-05)

### Other
- fix(ci): fail fast on missing secrets and bump deprecated actions (0.1.1)

## 0.1.0 (2026-10-05)

### Added
- initial opencode project template (0.1.0)

### Other
- Initial commit
