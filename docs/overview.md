# Polis

An issue-driven agent pipeline. Install the plugin (or fork it), and turn an idea into
shipped software — in any language — with configurable human gates at each step or fully
autonomous operation.

## Get started (Claude Code plugin)

Polis ships as a Claude Code plugin. Install it once, then use it from any directory:

```
/plugin marketplace add AnissL93/Polis
/plugin install polis@polis
```

That adds five skills: `/polis:set-up`, `/polis:drive`, `/polis:docs`, `/polis:page`, and `/polis:server-setup`.

> Prefer to set things up by hand on GitHub instead of using the plugin? See
> [Manual setup](#manual-setup-without-the-plugin) at the end.

## Skills

### `/polis:set-up` — create a new project

Run it from anywhere to stand up a brand-new Polis-powered repo. It walks you through, asking
as it goes — you never edit files by hand:

1. **Repo** — asks for a project name, target directory, owner, and visibility; clones the
   template and pushes a clean copy to a **new** GitHub repo (Actions on by default).
2. **Secrets** — guides you to create `PIPELINE_PAT` and set each backend's auth
   (`CLAUDE_CODE_OAUTH_TOKEN`, and optionally `CODEX_AUTH` / provider API keys). *You* run
   every secret/login command in your own session, so tokens never pass through the agent.
3. **Config** — shows the default `polis.yml` and asks whether to customize it.
4. **Labels** — creates the pipeline's trigger labels in the new repo.

The new repo contains the pipeline machinery plus a bundled `/drive` skill, so collaborators
can drive it without installing the plugin. No manual clone needed.

### `/polis:drive` — run the pipeline from the terminal

A cockpit for the whole pipeline — no GitHub web UI:

- a **state panel** of your issues/PRs with their current stage and the suggested next label;
- **create** an issue from an idea, and apply any `agent:*` trigger — plan / spec / code / fix,
  plus on-demand review with a chosen backend or profile;
- **watch** the triggered Actions run, **read** the AI/human review feedback, and **merge** a
  finished PR — all via `gh`, confirming before anything that spends a run or merges.

### `/polis:docs` — sync in-repo documentation

Keeps three in-repo documentation surfaces in sync with code changes since the last git tag:

| Step | What it does |
|------|-------------|
| **README** | Adds / updates / removes sections that reflect changed features, CLI flags, and APIs. Shows a unified diff before writing. |
| **CHANGELOG** | Drafts a new `[Unreleased]` section by categorising commits; asks before prepending. |
| **Inline docs** | Flags public symbols whose docstrings are missing or stale, then writes concise fixes on approval. |

Run it from the project root after any meaningful code change, or when cutting a release.

### `/polis:page` — publish to the public documentation site

Syncs official documentation and the pipeline visualization to a dedicated public GitHub
repository and deploys it to GitHub Pages — keeping the main project repo private.

| Step | What it does |
|------|-------------|
| **Visualization** | Updates `site/index.html` in the docs repo to reflect any pipeline or label changes. |
| **Docs sync** | Copies README, CHANGELOG, agent personas, and config examples to the docs repo. |
| **Pages deploy** | Commits, pushes, and triggers the GitHub Actions Pages workflow. |

The official Polis documentation lives at **[AnissL93/polis-docs](https://github.com/AnissL93/polis-docs)**
and is served at `https://AnissL93.github.io/polis-docs/`.

### `/polis:server-setup` — set up a self-hosted runner via SSH

SSH into a remote server and configure it as a GitHub Actions runner so every Polis
pipeline job runs on your own machine instead of GitHub's cloud:

1. **Connectivity** — tests SSH key-based access; guides you through `ssh-copy-id` if not yet configured.
2. **Prerequisites** — detects the OS (Ubuntu/Debian, RHEL/Fedora, macOS) and installs Node.js ≥ 18, git, and curl if missing.
3. **Runner** — downloads the latest GitHub Actions runner, registers it against your repo with a one-time token (fetched via `gh api` — the token is never printed), and installs it as a `systemd`/`launchd` service.
4. **Wire-up** — sets `RUNNER_LABEL` as a repo variable so the pipeline immediately routes to your server.
5. **Verify** — polls until the runner appears online, then summarises how to revert to cloud runners.

Credentials (Claude token, API keys) still come from GitHub Secrets — no extra auth setup on the machine.

## How it works — two layers

**Planning:** open an issue describing an idea, then drive it with labels:

| Add label | The agent… |
|-----------|------------|
| `agent:arch`      | writes an architecture doc (with a phased work breakdown) in a draft PR |
| `agent:rearch`    | rewrites the arch doc from your PR comments (repeat as needed) |
| `agent:decompose` | creates one milestone per phase and one issue per breakdown line |

**Execution:** on each generated issue:

| Add label | The agent… |
|-----------|------------|
| `agent:spec`   | writes a spec (incl. a Test plan) in a draft PR |
| `agent:respec` | rewrites the spec from your PR comments (repeat as needed) |
| `agent:code`   | implements code + tests, runs an AI review loop, marks the PR ready |
| `agent:fix`    | revises the code from your PR comments — or, with no comment, fixes the failing tests — then re-runs tests and updates the `tests-failing` label (repeat as needed) |

The pattern is always **comment → label → regenerate → repeat → advance**. The terminal
action is always a human **merge**. Only the repo owner can trigger the pipeline.

## Customizing

- `scripts/{build,test,deploy}.sh` — agent-owned; start as no-op placeholders.
  `build.sh` runs immediately before `test.sh` (in CI and the pipeline) on a clean runner,
  so it's where dependencies get installed (`pip install …`, `npm ci`, …). If `test.sh`
  calls a runner that isn't installed there, tests fail with `command not found`.
- `.github/agents/*.md` — the agent personas (system prompts). Tune to taste.
- `scripts/pipeline.sh` — the orchestrator. Covered by `tests/pipeline/`.

### Multi-backend configuration (`polis.yml`)

By default every stage runs Claude. To run stages on other models, copy
`polis.yml.example` → `polis.yml` and edit it. You can:

- define **backends** (`harness` + `model` + optional `base_url`/`api_key_env`);
  harnesses are `claude` (Anthropic), `codex` (OpenAI), and `aider` (universal —
  DeepSeek, Ollama, and most litellm-supported models);
- map each **role** (`architect`, `spec`, `code`, `fix`) to a backend;
- configure the **review loop**: `review.max_rounds` and any number of reviewers,
  each on its own backend (e.g. Claude writes, DeepSeek reviews).

For non-Claude backends, add the provider's API key as a repo **secret** and a
matching line in `.github/workflows/agent-pipeline.yml`'s `env:` block (named by the
backend's `api_key_env`). The workflow installs only the harnesses your config
references. With no `polis.yml`, nothing changes — claude everywhere, 3 rounds.

### On-demand & backend-selectable review

Beyond the automatic review during `agent:code`, you can trigger an AI review of the
**code**, **spec**, or **arch** doc with a label, choosing the reviewer at trigger time:

- `agent:code_review` — code review with the default recipe (the `review:` block).
- `agent:spec_review:codex` — review the spec using the `codex` backend.
- `agent:arch_review:thorough` — review the arch doc with the named `thorough` profile.

A label suffix is either a **backend** (your default reviewers forced onto it) or a
**profile** under `review_profiles:` (its own reviewers/mode/rounds; `mode: comment` posts
and stops, `mode: iterate` revises and re-reviews). Labels are generated from `polis.yml`
by `bootstrap-labels.sh`: one per configured backend and per valid profile — so a claude-only
fork never sees a codex label. Spec/arch reviews default to `comment` (they keep the human
gate); code defaults to `iterate`.

**Codex with a ChatGPT subscription (instead of a metered key).** Just as Claude
runs on a subscription token (`CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token`),
the `codex` harness can run on your ChatGPT plan. Run `codex login` locally, then
store the resulting `~/.codex/auth.json` **contents** in a secret named `CODEX_AUTH`
(already wired into the workflow env). When set, each agent job restores it to
`~/.codex/auth.json` before Codex runs, so you don't need `OPENAI_API_KEY`. Leave
`CODEX_AUTH` unset to fall back to a metered `OPENAI_API_KEY`. (The auth token is
refreshed in-run but can't persist back to the secret, so re-capture it if it expires.)

### Agent skills (`polis.yml`)

Agents can be given extra domain knowledge by injecting skill content into their system
prompts at job start. For Claude the content is appended via `--append-system-prompt`; for
Codex and Aider it is prepended to the task prompt.

Each skill is a `SKILL.md` file stored under `skills/<name>/`. Two ways to configure them:

**Option A — auto-detect (recommended).**
`install-harnesses.sh` calls `scripts/detect-skills.sh`, which fingerprints the repo and
downloads matching `SKILL.md` files from
[everything-claude-code](https://github.com/affaan-m/everything-claude-code) into `skills/`.
All downloaded skills are injected into every role automatically.

```yaml
skills:
  auto: true
```

Detection rules:

| Fingerprint | Skills added |
|---|---|
| `CMakeLists.txt`, `*.cpp/cc/cxx` | `cpp-coding-standards`, `cpp-testing` |
| `requirements.txt`, `pyproject.toml`, `setup.py` | `python-patterns`, `python-testing` |
| + `manage.py` / `django` in requirements | `django-patterns`, `django-tdd` |
| `go.mod` | `golang-patterns`, `golang-testing` |
| `Cargo.toml` | `rust-patterns`, `rust-testing` |
| `tsconfig.json` / `package.json` w/ typescript | `frontend-patterns` |
| + `next.config.*` | `nextjs-turbopack` |
| `pom.xml` / `build.gradle` (+ spring) | `java-coding-standards`, `springboot-patterns`, `springboot-tdd` |
| `*.kt` / `*.kts` | `kotlin-patterns`, `kotlin-testing` |
| + `AndroidManifest.xml` | `android-clean-architecture` |
| `*.swift` | `swift-concurrency-6-2`, `swiftui-patterns` |
| `artisan` / `composer.json` w/ laravel | `laravel-patterns`, `laravel-tdd` |
| `Dockerfile`, `docker-compose.yml` | `docker-patterns`, `backend-patterns` |
| + `postgres` in compose | `postgres-patterns` |
| always | `security-review` |

**Option B — explicit list.**
Name skills directly; they must already exist as `skills/<name>/SKILL.md` in the repo.
`global` skills go to every role; `roles` entries add role-specific ones.

```yaml
skills:
  global: [drive]
  roles:
    code: [drive, docs]
```

Both options can be combined: `auto: true` plus explicit `global`/`roles` entries are merged.

### Pipeline mode (`polis.yml`)

By default (`mode: human`) the pipeline pauses at every gate and waits for a human to
apply the next label and to merge the final PR. Set `mode: auto` to remove those gates:

```yaml
pipeline:
  mode: auto   # human (default) | auto
```

| Gate | `human` | `auto` |
|---|---|---|
| After `agent:spec` | wait for you to add `agent:code` | applies `agent:code` automatically |
| After `agent:arch` | wait for you to add `agent:decompose` | applies `agent:decompose` automatically |
| After `agent:code` finalize | labels `needs-human-review`, you merge | merges via `gh pr merge --squash` |

Auto mode only merges when **reviews converge and tests pass**. If either fails the PR is
still labeled `needs-human-review` so a human can intervene. You can interrupt at any point
by manually adding `agent:respec` or `agent:rearch` before the auto label fires.

### Self-hosted runners

By default the pipeline runs on GitHub-hosted `ubuntu-latest` runners. To use your own
server instead, set a repository **variable** (Settings → Secrets and variables → Actions →
Variables) named `RUNNER_LABEL` to your runner's label (e.g. `self-hosted`).

Every job in `agent-pipeline.yml` reads `vars.RUNNER_LABEL || 'ubuntu-latest'`, so a single
variable switches the whole pipeline to your machine. Useful when:

- you want to avoid GitHub Actions minutes;
- you're running a local model via the `aider` harness and `base_url: http://localhost:...`;
- you need custom hardware (GPU, large RAM) or network access to internal services.

**Setup checklist for a self-hosted runner:**

1. [Register the runner](https://docs.github.com/en/actions/hosting-your-own-runners) on your
   machine and give it a label (e.g. `self-hosted`).
2. Set `RUNNER_LABEL=self-hosted` as a repo variable.
3. Ensure Node.js ≥ 18, `npm`, `git`, and `curl`/`wget` are installed — `install-harnesses.sh`
   handles the rest (Claude Code, yq, Codex, Aider) and auto-detects Linux x86_64/arm64
   or macOS (via Homebrew).
4. Set the same repo **secrets** as a cloud setup (`CLAUDE_CODE_OAUTH_TOKEN`, `PIPELINE_PAT`,
   plus any backend keys).

For multi-label runner selectors (e.g. `[self-hosted, linux, x64]`) you can edit the
`runs-on` line in `.github/workflows/agent-pipeline.yml` directly.

## Manual setup (without the plugin)

Prefer to fork on GitHub and wire it up yourself? `/polis:set-up` automates all of this, but
here are the manual steps:

1. **Secrets** (Settings → Secrets and variables → Actions):
   - `CLAUDE_CODE_OAUTH_TOKEN` — Claude Code auth.
   - `PIPELINE_PAT` — a fine-grained PAT with `contents`, `issues`, `pull-requests` write.
     (Used instead of `GITHUB_TOKEN` so the agent's PRs/pushes can trigger CI.)
   - **Org-owned forks only:** also add a *variable* (not secret) `PIPELINE_OWNER` =
     your GitHub username. The pipeline only triggers when the person who applies the
     `agent:*` label is the repo owner or matches `PIPELINE_OWNER`; on an org the
     owner is the org itself, so no human ever matches without this.
2. **Enable Actions** — open the **Actions** tab and click *I understand my workflows,
   enable them* (GitHub disables Actions on every fork by default).
3. **Labels** — created automatically the first time you open an issue (the **Bootstrap
   Labels** workflow runs on `issues: opened`). To create them ahead of time, run it
   manually: Actions → Bootstrap Labels → Run workflow.
4. You never edit `scripts/build.sh` / `test.sh` / `deploy.sh` by hand — the agent writes
   them to match whatever language it builds.

## Developing the pipeline itself

Run the harness tests: `bash tests/pipeline/run.sh`

The skills live under `skills/` (plugin layout). To use `/polis:set-up` / `/polis:drive` while
hacking on Polis itself, add the local checkout as a marketplace: `/plugin marketplace add ./`
then `/plugin install polis@polis`.
