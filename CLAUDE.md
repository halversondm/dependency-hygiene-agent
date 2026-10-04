# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

There is no application code, build, lint, or test tooling here. The repo is a Claude Code agent setup that updates open source dependencies (Maven and NPM) across the repositories listed in `config.yml`, then opens PRs and notifies Slack. The behavior lives entirely in two agent prompt files, so changing behavior means editing Markdown.

## Running

```
claude --agent dependency-orchestrator
```

Run from the repo root. It must run as the top-level session: Claude Code subagents cannot spawn subagents, so the orchestrator cannot itself be invoked as a subagent. Agents in `.claude/agents/` are only discovered at session start. If you edit or add them mid-session, `dependency-updater` will not be available as a `subagent_type`, and workers must be general-purpose agents told to read `.claude/agents/dependency-updater.md`.

Prerequisites: `gh` authenticated with push access, `mvn` and `node`/`npm` installed, and the Slack MCP (`plugin:engineering:slack`) authenticated via `/mcp`.

## Architecture

- `.claude/agents/dependency-orchestrator.md` reads `config.yml`, runs at most 5 `dependency-updater` workers concurrently (starting the next repo whenever one finishes), and posts a Slack message per PR plus a final summary. It never does repository work itself and never merges.
- `.claude/agents/dependency-updater.md` handles exactly one repo. It clones into `<workdir>/<repo>`, ensures a `CLAUDE.md` exists, updates dependencies, builds (max 5 fix attempts), and pushes a `deps/update-<date>` branch. It then opens a PR against the default branch. It returns a fixed-format report (`repo`, `status`, `pr_url`, `updated`, `fix_attempts`, `notes`) that the orchestrator parses. Changing that format means updating both files.
- Slack is deliberately sent only by the orchestrator, not the workers.
- `config.yml` has `slack_channel`, optional `workdir` (default `./workspace`, gitignored), and `repositories[]` with `url` and optional `default_branch` and `build_command`. The Slack channel is referenced by name in the config but the Slack tool needs its channel ID, so look it up with `slack_search_channels`.

## Behavior to know

- Only stable versions are used. Pre-release versions (alpha, beta, RC, milestone, snapshot) are skipped.
- A repo with no real dependency changes gets no PR, even if a `CLAUDE.md` was added. The worker reports `no-changes`.
- If the build can't be fixed in 5 attempts, the worker doesn't push. The partial approach used so far has been to hold back the offending major bump (for example keeping msw on 2.x) and still open the PR for the rest.
- HTTPS pushes may fail without credentials. Workers have worked around this with SSH or a one-off `gh auth git-credential` helper passed via `git -c`, without changing git config.
