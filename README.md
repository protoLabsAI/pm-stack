# Portfolio Manager — a protoAgent plugin bundle

A **bundle** (ADR 0040): a curated, pinned set of plugins you install with one command.
`pm-stack` stands up a **Portfolio Manager** — the manager-of-teams tier of the
portfolio→team hierarchy (ADR 0055). It orchestrates work across **many Lead Engineer
teams**: it spins a team up per project, dispatches features to each team's board over
A2A, rolls up a bounded cross-board view, sequences cross-board dependencies, and disposes
a team once its board drains. It runs **no board of its own** — that's the team tier.

The project-team counterpart is the **[Lead Engineer](https://github.com/protoLabsAI/leadEngineer)**
bundle (a single repo + its coding agents). This bundle is the manager that delegates to them.

## What's in it
| Plugin | Role | On the PM? |
|---|---|---|
| **delegates** | the ACP/A2A spawn spine (ships with protoAgent) | **enabled** |
| **[portfolio](https://github.com/protoLabsAI/portfolio-plugin)** | orchestrate + **spawn** teams: `portfolio_spinup_team`, `dispatch`, `rollup`, `diff`/`watch`, `link`/`plan`/`autodispatch`, `autodispose` | **enabled** |
| **[project_board](https://github.com/protoLabsAI/projectBoard-plugin)** | beads-backed board + ACP spawn loop — the **team's** board | installed, off |
| **[agent_browser](https://github.com/protoLabsAI/agent-browser-plugin)** | browser automation — the **team's** browser | installed, off |

### Why the team plugins are installed but off
A Portfolio Manager doesn't run a board. But the **Lead Engineer teams it spawns do** —
and a spawned team's `plugins.dir` defaults to **this host's** plugins dir. So pm-stack
installs `project_board` (+ `agent_browser`) here, off by default, and every team the PM
spins up discovers them on this host without a per-team reinstall. **Install once, provision
many teams.**

The external plugins are **pinned to release tags** — `pm-stack` is a *tested combo*, not
"whatever's latest." A pin means **"last verified working"** and only moves through a passing
verification (ADR 0049): CI installs this manifest's pin set into a scratch agent and probes
every declared console view on each PR + weekly, and a scheduled job opens a pin-bump PR when
a member tags a new release. `verified_against:` records the core version the set was last
verified on.

## Install
```bash
python -m server plugin install https://github.com/protoLabsAI/pm-stack
```
Install ≠ enable ≠ trust — the installer prints the suggested `plugins.enabled` + config;
apply them to your `config/langgraph-config.yaml`:
```yaml
plugins:
  enabled: [delegates, portfolio]
# That's it — no team_template needed. Spawned teams INHERIT the PM's own gateway
# (portfolio v0.14+): the PM's resolved model.api_base + its OPENAI_API_KEY (via the team's
# environment) carry over, so a team boots ready to think with zero creds prep. Only set a
# team_template for a team you want on a DIFFERENT gateway than the PM's:
# portfolio:
#   team_template: /path/to/your/team-template   # optional — a per-team gateway/config
```
then restart. The board the PM's teams run needs the `br` (beads) CLI on this host.

**One-pick setup:** installing the bundle also registers a **"Portfolio Manager"** archetype
in the new-agent picker (ADR 0042) — pick it and you get a PM with `delegates`+`portfolio`
enabled and the manager persona set, no YAML. With gateway inheritance there's nothing else
to configure: spin up a team and it runs on your gateway.

## Spinning up teams
Once enabled, the Portfolio Manager spins up an ephemeral Lead Engineer team per project:
```
portfolio_spinup_team(name="docs-team", repo="/abs/path/to/repo")   # boots + registers a board
portfolio_dispatch(board="docs-team", title=…, spec=…)              # send it work over A2A
portfolio_autodispose()                                             # once its board drains, dispose it
```
See the [portfolio plugin](https://github.com/protoLabsAI/portfolio-plugin) for the full tool
set + team-template details, and [Lead Engineer](https://github.com/protoLabsAI/leadEngineer)
for the team tier. For the full picture — tiers, templates, the two-axis coder escalation,
and a worked example — see **[docs/teams.md](docs/teams.md)**.

## Maintenance
```bash
# verify locally (from a protoAgent checkout, deps synced)
uv run --no-sync python /path/to/pm-stack/scripts/verify_bundle.py /path/to/pm-stack

# check members for newer release tags (rewrites refs in place)
python3 scripts/check_bundle_updates.py protoagent.bundle.yaml
```
