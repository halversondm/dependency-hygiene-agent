# Dependency Hygiene Agent

A Claude Code agent setup that keeps open source dependencies up to date across many repositories. Give it a list of repos and it raises a pull request with updated dependencies for each one, then notifies you in Slack.

## What it does

For each repository in `config.yml`:

1. Clones the repository.
2. Creates a `CLAUDE.md` if one doesn't exist.
3. Updates dependencies to their latest stable versions:
   - Maven projects (`pom.xml`) from Maven Central
   - NPM projects (`package.json`) from the NPM registry
4. Builds the project and fixes errors, with up to 5 fix attempts.
5. If there are real changes and the build passes, pushes a `deps/update-<date>` branch.
6. Opens a PR against the default branch.
7. Posts the PR to Slack.

Repositories are processed by up to 5 subagents at a time. The orchestrator collects each worker's status and prints a final summary.

## Prerequisites

- [Claude Code](https://claude.com/claude-code)
- `gh` CLI, authenticated with push access to the target repositories
- `git`, plus `mvn` and/or `node`/`npm` for the project types you use
- The Slack MCP (`plugin:engineering:slack`), authenticated via `/mcp`

## Configuration

Edit `config.yml`:

```yaml
slack_channel: "#dependency-updates"
workdir: ./workspace            # optional, where repos are cloned

repositories:
  - url: https://github.com/your-org/example-maven-service
  - url: https://github.com/your-org/example-npm-app
    default_branch: main                          # optional
    build_command: npm ci && npm run build        # optional override
```

If `build_command` is omitted, the agent uses `mvn -B clean verify` for Maven or `npm ci && npm run build && npm test` for NPM, running only the scripts that exist.

## Usage

From the repository root:

```
claude --agent dependency-orchestrator
```

It must run as the top-level session. Claude Code subagents can't spawn other subagents, so the orchestrator can't be called from another agent. Agents in `.claude/agents/` are discovered at session start, so restart the session after editing them.

## Agents

| Agent | File | Role |
|---|---|---|
| `dependency-orchestrator` | `.claude/agents/dependency-orchestrator.md` | Reads `config.yml`, runs up to 5 workers concurrently, sends Slack notifications, and produces the final summary |
| `dependency-updater` | `.claude/agents/dependency-updater.md` | Handles one repository end to end and returns a status report |

Each worker's report includes `status` (`pr-created`, `no-changes`, `failed` or `skipped`), the PR URL, the number of dependencies updated, fix attempts, and notes.

## Behavior notes

- Only stable releases are used. Alpha, beta, RC, milestone and snapshot versions are skipped.
- Major-version bumps are included and called out in the PR.
- A repository with no dependency changes gets no PR.
- If the build can't be fixed in 5 attempts, nothing is pushed for that repository and it's reported as `failed`.
- The agents never push to the default branch, never force-push, and never merge PRs. Review each PR before merging.
