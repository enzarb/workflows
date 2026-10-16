# enzarb/workflows

The standard enzarb workflow library (ADR-0022). The engine knows no workflow: every phase,
step, prompt, state name, board column and runtime enzarb runs by default is YAML in this
repository, and a Project or repository inherits it, overrides it, or copies parts of it.
The formats are specified in enzarb/enzarb `docs/workflows/format.md`; prompts are MiniJinja
templates over the task document (`docs/workflows/task-document.md`).

## Layout

| Path | What |
|---|---|
| `runtimes/claude-code.yaml` | Claude Code over ACP: `npm:@agentclientprotocol/claude-agent-acp` and node, installed with mise; secret `{name: claude_token, scope: user, env: CLAUDE_CODE_OAUTH_TOKEN}` |
| `steps/` | One unit of sandbox work each. Agent steps: `triage`, `plan`, `breakdown`, `implement`, `review`, `address`, `ci_fix`, `resolve_conflicts`, `task`. Command steps: `context` (merge base, changed files, affected components), `prepare`, `checks` |
| `prompts/` | The agent prompts, one per agent step |
| `phases/` | Subgraphs with params and exits: `dependencies`, `start_gate`, `triage`, `plan`, `breakdown`, `implement`, `review` (runs the internal `review_rounds`), `merge`, `release`, `mention` |
| `workflows/` | `issue` (epics too, via its `epic` param), `pull_request`, `mention` |
| `board.yaml` | Backlog, Plan, In progress, Blocked, Review, Done (limit 20) |
| `bindings.yaml` | Library default bindings |
| `tests/` | Scenario tests, run by `enzarb-lint test` |
| `examples/` | `enzarb.yaml` (the proposed `.enzarb/workflows.yaml` of enzarb/enzarb), `auto-merge.yaml` |
| `lint/action.yml` | The `enzarb/workflows/lint@v1` GitHub Action |

## Workflows

- **`issue`**: dependencies → start gate (`/plan`) → triage → plan (approval) → implement →
  review → merge → release. Triage `epic`, `/epic` during plan approval, the `epic` label
  (from a writer, any time) or the `epics` binding's `epic: true` go to the breakdown
  instead, and `vars.epic` remembers it across phases: sub-issues are created on approval,
  related to the epic's task as children (`parent` from their side), and the epic ends when
  they all ended. Sub-issues run with `epic_allowed: false`, so they never nest.
- **`pull_request`**: a PR that references no issue: review → merge → release. A PR that
  references an issue joins that issue's task (the engine links it), so it is reviewed by
  the issue workflow's review phase. The PR enzarb opens is linked with `subject: true`.
- **`mention`**: `@<bot handle> …` (the comment's `mentioned`) on an object with no live
  task: one read-only `task` step, reply posted.

Both `issue` and `pull_request` end the task anywhere (stopping a running step) when the
pull request is closed without merging; `issue` also when the issue is closed, unless it
was closed by the merge (`vars.merged`).

### Workflow params

Set by a binding's `with` (or a scenario's `params`).

| Workflow | Param (default) | What |
|---|---|---|
| `issue` | `epic` (false) | Treat the issue as an epic (the `epics` binding sets it) |
| | `epic_allowed` (true) | May become an epic; the `issues` binding sets false for sub-issues of an epic |
| | `epic_label` (epic) | Label that marks an epic later on |
| | `stale_after` (14d) | Triage questions go stale after this duration |
| | `max_rounds` (3) | Automatic fix rounds on the review's must-fix findings |
| | `release_when` ([]) | Release events that end the task; empty ends it at the merge |
| `pull_request` | `max_rounds` (3), `release_when` ([]) | As above |

## Using the library

A repository opts in with `.enzarb/workflows.yaml`:

```yaml
version: 1
imports: {std: enzarb/workflows@v1}
bindings: []           # inherit every library binding
```

`std/<path>` then names a file of this repository at `v1` (a moving major tag; pin a tag
or sha to opt out of upgrades). A repository workflow composes the phases:

```yaml
# .enzarb/workflows/issue.yaml
version: 1
imports: {std: enzarb/workflows@v1}
board: std/board
on:
  - {events: [{event: issue, if: "event.action == 'closed' and not vars.merged"}], do: "end:closed"}
phases:
  - {use: std/phases/dependencies}
  - {use: std/phases/start_gate, with: {command: start, button: Start}}
  - {use: std/phases/triage, with: {stale_after: 7d}, exits: {epic: breakdown}}
  - {use: std/phases/plan, with: {prompt: {file: .enzarb/prompts/plan.md}}, exits: {epic: breakdown}}
  - {use: std/phases/implement, with: {max_rounds: 5}}
  - {use: std/phases/review, with: {ci_ignore: "(?i)copilot|codecov"}, exits: {merged: release}, max_entries: 10}
  - {use: std/phases/merge, with: {mode: auto_when_green}, exits: {changed: review}, max_entries: 10}
  - {use: std/phases/release, with: {when: [release.published]}, exits: {done: {end: done}}}
  - {use: std/phases/breakdown}
```

Often a binding's `with` is enough (see `examples/enzarb.yaml`):

```yaml
bindings:
  - {id: issues, on: issue, if: "event.action == 'opened'", use: std/workflows/issue, with: {release_when: [release.published]}}
```

To change something a phase does not expose as a param, copy the phase file into the
repository and `use` the copy.

### Bindings layering

Bindings map triggers to workflows in three layers: this library's `bindings.yaml`, the
Project's `spec.workflows.bindings`, and the repository's `.enzarb/workflows.yaml`. The merged
list is ordered repo, Project, library, and the first binding whose `on` and `if` match wins.
A binding with the same `id` as a lower one replaces it; `{id: <id>, disabled: true}` removes it.

| id | on | if | workflow |
|---|---|---|---|
| `epics` | `issue` | opened with the `epic` label, not a sub-issue | `workflows/issue` with `epic: true` |
| `issues` | `issue` | opened; `epic_allowed` is false when `relations` has a `parent` (a sub-issue) | `workflows/issue` |
| `adopted_prs` / `adopted_issues` | `adopted` | by subject kind | `workflows/pull_request` / `workflows/issue` |
| `pull_requests` | `pull_request` | opened, references no issue | `workflows/pull_request` |
| `mentions` | `comment` | `event.mentioned` (the instance's bot handle), not a bot | `workflows/mention` |

## Phase params and exits

Every phase also has the implicit exits `done` (next phase) and `limit_reached`.

| Phase | Params (default) | Exits | Waits (reason · column) |
|---|---|---|---|
| `dependencies` | `done_statuses` ([done, merged, released]), `unblock_command` (unblock) | | `blocked_by_dependencies` · blocked; button "Start anyway". Dependencies already known to be finished pass at once (`decide`) |
| `start_gate` | `command` (plan), `label` (plan), `button` (Start planning) | `closed` | `awaiting_start` · backlog |
| `triage` | `prompt`, `epic` (false), `epic_allowed` (true), `stale_after` (14d) | `epic`, `duplicate`, `closed`, `failed` | `awaiting_answers` (timer `stale_after`), `stale` · blocked |
| `plan` | `prompt`, `approve_command` (approve), `approve_label` (Approve), `epic_command` (epic), `epic_allowed` (true) | `epic`, `closed`, `failed` | `awaiting_approval` · plan; any other comment re-plans. `vars.epic` leaves through `epic` at once |
| `breakdown` | `prompt`, `max_subissues` (8), `approve_command` (approve), `approve_label` (Create sub-issues) | `closed`, `failed` | `awaiting_approval` · plan; `awaiting_children` · in progress. `create_issues` result is `outputs.children`; each issue is related (`parent`) |
| `implement` | `prompt`, `prepare` (mise run prepare), `check` (mise run check), `retry_command` (retry), `retry_label` (Retry), `draft` (false), `max_rounds` (3: attempts against failing checks) | `closed` | `implement_blocked` · blocked. The opened PR is `outputs.pull_request` and a subject; branch `enzarb/issue-<n>` |
| `review` | `review_prompt`, `address_prompt`, `ci_fix_prompt`, `resolve_prompt`, `ci_ignore` (`(?i)copilot`, a regex), `retry_command` (retry), `max_rounds` (3), `max_ci_rounds` (3), `max_conflict_attempts` (2) | `merged`, `closed` | `awaiting_ci` · review; `awaiting_review`, `round_limit` · review; `ci_failed`, `conflict_parked` · blocked |
| `merge` | `mode` (human \| auto_when_green), `command` (merge), `merge_label` (Merge), `method` (squash), `delete_branch` (true), `retry_command` (retry), `retry_label` (Retry) | `changed` (back to review), `closed` | `checking_merge`; `ready_to_merge` · review; `merge_failed` · blocked. Sets `vars.merged` |
| `release` | `when` ([]: `<type>` or `<type>.<action\|state>`, e.g. `release.published`, `package.published`, `deployment.success`) | `failed` | `awaiting_release` · done; `when: []` is done at once (`decide`) |
| `mention` | `prompt` | `failed` | |

Every button's label and command come from the phase's params.

Review policy: the bot's must-fix findings get `max_rounds` automatic fix rounds, counted
in `vars.fix_rounds` (a person's comment or review, a push by someone else and `/retry`
reset it). Red GitHub Actions checks of the repository get `max_ci_rounds` CI-fix rounds
per cycle with the failing logs (checks whose name matches the `ci_ignore` regex never count;
other failing checks wait on a person). A merge conflict gets `max_conflict_attempts`
resolution attempts (merge, never rebase) per base sha: `vars.conflict_base` and
`vars.conflict_attempts`, so only a moved base or `/retry` brings fresh attempts. Each
review stores the head it looked at in `vars.reviewed_sha`, and the next review gets it as
`previous_reviewed_sha` and reviews the delta. Nothing merges without a writer's Merge click
or `/merge` unless the merge phase runs with `mode: auto_when_green`; the merge is pinned to
the head CI passed on (`head_sha`).

### Task variables

| Var | Set by | Read by |
|---|---|---|
| `epic` | triage/plan `epic` exit, the `epic` label handler | triage, plan |
| `fix_rounds` | review (`address` +1; reset on a person's round, a push, `/retry`) | review |
| `conflict_base`, `conflict_attempts` | review `conflict_gate` | review |
| `reviewed_sha` | review step | the next review (delta) |
| `merged` | merge `done`, review `merged` | the `issue` workflow's close handler |

### Still open

- `dependencies` cannot pass without a refresh when an issue has no dependencies: an empty
  `facts.dependencies` is indistinguishable from an unknown one (the engine omits empty
  lists), so the first refresh still decides.
- The shapes of the `create_issues` result (`{issues: [{kind, repo, number, …}]}`) and of
  `open_pull_request`'s result (`{kind, number, title, url}`) are what the library assumes;
  the connector contract (M3) has to match them.

## Checking

```sh
enzarb-lint check . --repo enzarb/workflows
enzarb-lint test  . --repo enzarb/workflows
```

In another repository's CI:

```yaml
- uses: enzarb/workflows/lint@v1
  with:
    token: ${{ secrets.ENZARB_LINT_TOKEN }}   # read access to enzarb/enzarb while it is private
```

The action builds `enzarb-lint` from enzarb/enzarb at a pinned ref (`enzarb-ref`), serves
`enzarb/workflows` imports from a checkout of `library-ref` (default `v1`), and runs `check`
and `test`. Without a token it skips with a notice.
