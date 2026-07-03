# How your Roxy's teams work

Roxy is a **Portfolio Manager** (the `pm-stack` archetype): one agent that orchestrates
work across *many* project teams instead of writing code herself. This guide covers how
those teams come into being, what each one carries, and how a feature climbs from "spec"
to "merged PR" once she's dispatched it.

## The two tiers

`pm-stack` stands up the **manager tier** of a two-tier hierarchy (ADR 0055):

| Tier | Runs | Board? |
|---|---|---|
| **Roxy** (the PM) | `delegates` + `portfolio`, enabled | **None — board-less by design** |
| **A team** (per repo) | its own `project_board`, in its own protoAgent instance | **One board per team, one repo per board** |

Roxy doesn't hold a `.beads` database, doesn't open worktrees, and doesn't dispatch a
coding agent directly. She uses the **`portfolio`** plugin to spin up a **team** — a
separate, fully-fledged protoAgent instance running `project_board` for exactly one
repo — and talks to it over **A2A**. `project_board` + `agent_browser` ship *installed
but disabled* on Roxy's own instance for exactly one reason: a spawned team's plugin
discovery defaults to Roxy's host, so every team she spins up finds them there with no
per-team install. Roxy never enables them herself.

```
portfolio_spinup_team(name="docs-team", repo="/abs/path/to/repo")
  → clones a team template, boots a new protoAgent instance bound to that repo,
    registers it on the fleet as a board named "docs-team"

portfolio_dispatch(board="docs-team", title=…, spec=…, acceptance_criteria=…)
  → sent over A2A; docs-team's OWN lead agent creates + readies the feature on
    its OWN board — Roxy never touches its repo or its .beads directly

portfolio_rollup()  /  portfolio_diff()  /  portfolio_watch()
  → bounded, cross-board visibility: lane counts + only what's blocked or just
    changed, never a raw dump of every team's every feature

portfolio_autodispose()
  → tears down a spawned team once its board has drained (all work done)
```

Each team is genuinely isolated: its own `.beads/*.db`, its own git worktrees, its own
coder delegates. A bug or a stuck loop on one team's board can't touch another's, and
Roxy's context stays a rollup, never a merge of every team's raw state.

## Templates: what a team is cloned from

`portfolio_spinup_team` doesn't build a team from scratch — it clones a **team
template**: a base `langgraph-config.yaml` with three sentinels filled in per spawn
(a plain, comment-preserving string replace, so the template stays readable):

| Sentinel | Filled with |
|---|---|
| `{{REPO}}` | the `repo` argument — the repo this team's board manages |
| `{{TEAM_NAME}}` | the `name` argument |
| `{{GATE}}` | the `gate` argument — the pre-PR check command (empty **auto-detects** one from the repo's build files — `ruff check . && ruff format --check .` for Python, `npm ci && npm run docs:build` for a VitePress site, `npm ci && npm test` for any other Node project; truly empty only when none of those match) |

The bundle's default template (from the `portfolio` plugin's
[`examples/team-template`](https://github.com/protoLabsAI/portfolio-plugin/tree/main/examples/team-template))
enables `project_board` + `delegates`, declares an ACP coder (`proto`, the first-class
protoLabs coding agent) scoped to `{{REPO}}`, and wires it into the board's `coders`
ladder. **Empty is the happy path**: leave `model.api_base` blank in the template and a
spawned team inherits Roxy's own resolved gateway + `OPENAI_API_KEY` through its
environment, so it runs real turns with zero creds prep. Only fill in a gateway (and a
sibling `secrets.yaml`) for a team you want on a *different* gateway than Roxy's.

Point Roxy at **your own** template once you outgrow the generic default:

```yaml
# your langgraph-config.yaml
portfolio:
  team_template: /path/to/your/team-template   # a langgraph-config.yaml (+ secrets.yaml)
```

or configure named **archetypes** — repo/gate (+ template) presets for a repo you spin
up often, so it's one word instead of a full path each time:

```yaml
portfolio:
  team_archetypes:
    protolibrary: { repo: ~/dev/protoLibrary, gate: "npm run docs:build" }
    protoagent:   { repo: ~/dev/protoAgent,   gate: "ruff check ." }
```

```
portfolio_spinup_team(name="protolibrary-team", archetype="protolibrary")
```

An archetype only presets *what* gets built (repo/gate/template) — it says nothing
about lifecycle. `portfolio_spinup_team` still defaults to `auto_dispose=True`, so an
archetype-spun team is torn down by `portfolio_autodispose()` exactly like an ad hoc one
the moment its board drains. For a repo you want to keep as a **standing** team, pass
`auto_dispose=False` explicitly (see the worked example below).

A template is also where you extend the coding roster: add another ACP delegate (e.g. a
`claude` entry alongside `proto`) to the template's `delegates:` list and reference it in
`project_board.coders` to give that team's escalation ladder a second coding agent, not
just a second model tier on the same one.

## The two-axis coder escalation

A team's board doesn't hand a feature to one coder and hope. It escalates on **two
independent axes** when a shot stalls, and they compose rather than compete:

### Axis 1 — model tier (live today)

`project_board.coders` is a map of **tier name → delegate**, e.g.:

```yaml
project_board:
  coders:
    fast: proto          # first attempt
    smart: proto         # climbs here on a capability failure
    reasoning: proto     # then here
    # opus: claude       # add another ACP agent as the top rung
```

A **capability failure** (no diff produced, or a timeout — *did the agent do anything
at all?*) climbs the ladder one rung (`fast → smart → reasoning → …`) and blocks at the
top if the last rung also fails. Transient failures (rate-limit, a merge conflict) retry
the *same* tier with backoff instead of climbing — a bigger model doesn't fix a flaky
network call. This ladder is throwing progressively bigger brains at the same shot; it
ships today in `project_board`.

### Axis 2 — execution-grounded search (landing per ADR 0064)

The second axis doesn't change *which model* answers — it changes *how hard the board
searches* on a fixed model, gated on whether the result actually **passes tests**:

```
greedy        1-shot                                            cheap; solves most
best-of-k     k candidates → run tests → keep the passing one    headroom recovery
tree-search   refine on the failing tests, bounded depth         grounded fix loop
fusion        richer candidate generation → execute-select       hardest, priciest
```

Each rung fires **only when the cheaper one fails its tests** — the board's Ready gate
already requires acceptance criteria before a feature is dispatchable; this axis compiles
those criteria into runnable tests and uses them as the selector, instead of shipping
whatever the coder produced and finding out from PR review. `project_board` already ships
an initial execution-grounded step here (best-of-N candidate selection, gated on your
`local_gate_cmd` when one's configured) — the full ladder (tree-search refinement, the
fusion rung) is landing as the standalone **`coder`** plugin, which composes into the
board loop.

**The two axes together:** tier escalation asks *"can a smarter model do this?"*; the
execution-grounded ladder asks *"can this same model find a passing answer if it searches
harder?"* — a hard, well-specified feature can climb tiers *and* search rungs before it
blocks. Neither axis is Roxy's concern: she only ever sees the outcome (merged / blocked)
and the cost in `portfolio_rollup` — the ladder is entirely a `project_board` (team-tier)
concern.

## A worked example

Say you want a standing team for two of your own repos, each with its own gate command:

```yaml
# langgraph-config.yaml — on Roxy
plugins:
  enabled: [delegates, portfolio]

portfolio:
  team_template: ~/dev/my-team-templates/default   # your own template, not the bundle's generic one
  team_archetypes:
    api:  { repo: ~/dev/my-api,  gate: "make check" }
    docs: { repo: ~/dev/my-docs, gate: "npm run build" }
```

Restart, then:

```
portfolio_spinup_team(name="api-team", archetype="api", auto_dispose=False)
  → clones your template, binds ~/dev/my-api, boots the team, registers "api-team" as a board

portfolio_spinup_team(name="docs-team", archetype="docs", auto_dispose=False)
  → same, for ~/dev/my-docs

# auto_dispose=False on both — these are STANDING teams for repos you work often, not
# one-shot spawns. Leave it at its default (True) for a finite project's team instead.

portfolio_dispatch(board="api-team", title="Add rate-limit headers to /v2",
                    spec="Return X-RateLimit-* headers on every /v2 response.",
                    acceptance_criteria="Existing rate-limit tests plus a new one asserting the headers are present.")
  → api-team's own lead creates + readies the feature on ITS board; its coder ladder
    (tier escalation, and — once landed — execution-grounded search) builds it,
    opens a PR, and ships it through `make check`

portfolio_rollup()
  → [ {"board": "api-team",  "total": 5, "counts": {"ready": 1, "in_progress": 1, "done": 3},
       "blocked": [], "critical_path": []},
      {"board": "docs-team", "total": 2, "counts": {"done": 2}, "blocked": [], "critical_path": []} ]
  # bounded lane counts (counts) + only the blocked/foundation items — never every feature

portfolio_autodispose()
  → leaves both alone — auto_dispose=False means they're never a candidate, regardless
    of how drained their board is; use portfolio_teardown_team(name) if you want one
    gone anyway
```

## Further reading

- **[ADR 0055 — Multi-team orchestration: federated boards over A2A](https://github.com/protoLabsAI/protoAgent/blob/main/docs/adr/0055-multi-team-orchestration-federated-boards.md)**
  — why teams scale *out* (one board per protoAgent instance, addressed over A2A) instead
  of *up* (one process holding every repo's board), and the rollup/diff/dependency-graph
  phasing.
- **[ADR 0064 — `coder`: execution-grounded code-solve](https://github.com/protoLabsAI/protoAgent/blob/main/docs/adr/0064-coder-execution-grounded-code-solve.md)**
  — why the board needed a *second* escalation axis beyond model tier, how the
  greedy → best-of-k → tree-search → fusion ladder gates on tests instead of an LLM
  judge, and how it composes with (rather than replaces) the tier ladder above.
- **[portfolio-plugin](https://github.com/protoLabsAI/portfolio-plugin)** — the full tool
  reference + `examples/` templates.
- **[projectBoard-plugin](https://github.com/protoLabsAI/projectBoard-plugin)** — the
  team-tier board + coder ladder this guide describes.
