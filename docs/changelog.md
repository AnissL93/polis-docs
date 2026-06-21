# Changelog

All notable changes to this project will be documented in this file.
Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- **Onboard an existing repo** — `/polis:set-up` now has two modes: scaffold a new project,
  or adopt an existing repo by URL. `scripts/onboard-existing.sh` injects the pipeline
  machinery non-destructively (keeps the repo's README, CI, and any existing
  `build.sh`/`test.sh`; seeds missing build/test scripts from the detected stack) on a
  `chore/onboard-polis` branch and opens a PR. The skill then analyzes the code and open
  issues and recommends a starting label for each.
- **`/polis:server-setup` skill** — SSH into a remote server and configure a self-hosted
  GitHub Actions runner end-to-end: tests connectivity, installs prerequisites (Node.js,
  git, curl) per OS (Ubuntu/Debian, RHEL, macOS), downloads and registers the runner with
  a one-time token (never echoed), installs it as a systemd/launchd service, sets
  `RUNNER_LABEL`, and polls until online.
- **Self-hosted runner support** — all pipeline jobs now read `vars.RUNNER_LABEL`
  (falls back to `ubuntu-latest`); set one repo variable to route every job to your own
  machine.
- **Agent skills injection** — `polis.yml` accepts a `skills:` block; content from
  `skills/<name>/SKILL.md` is injected into agent system prompts (`--append-system-prompt`
  for Claude, prepended text for Codex/Aider).
- **Auto tech-stack detection** (`skills: {auto: true}`) — `scripts/detect-skills.sh`
  fingerprints the repo and downloads matching SKILL.md files from `everything-claude-code`
  into `skills/`. Detects: C++, Python, Django, Go, Rust, TypeScript, Next.js,
  Java/Spring, Kotlin, Android, Swift, Laravel, Perl, Docker, Postgres; always adds
  `security-review`.
- **Pipeline auto mode** (`pipeline: {mode: auto}`) — skips human label gates: applies
  `agent:code` after spec and `agent:decompose` after arch automatically; auto-merges the
  PR when reviews converge and tests pass. Falls back to `needs-human-review` on
  cap-reached or test failures.
- **Pipeline visualization site** with animations and install commands (GitHub Pages).
- **`/polis:docs` skill** — syncs README, CHANGELOG, inline docs, and doc site.
- **`/polis:page` skill** — publishes documentation to the public GitHub Pages site.

### Changed

- `install-harnesses.sh`: yq installation is now cross-platform — Linux x86_64/arm64
  (wget) and macOS (Homebrew).
- `polis.yml.example`: new `skills:` and `pipeline:` sections with auto and explicit options.
- `/polis:docs` and `/polis:page` split from a single combined skill into two.

### Maintenance

- CI: Pages workflow enabled automatically; Node 20 deprecation warning silenced.
- Tests: 26 pipeline tests covering skills injection, auto-detection, auto-proceed, and
  harness dispatch.
