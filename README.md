# enzarb/workflows

The standard enzarb workflow library (ADR-0022). The engine knows no workflow: every phase,
step, prompt, state name, board column and runtime enzarb runs by default is YAML in this
repository, and a Project or repository inherits it, overrides it, or copies parts of it.
The formats are specified in enzarb/enzarb `docs/workflows/format.md`; prompts are MiniJinja
templates over the task document (`docs/workflows/task-document.md`).

## Layout

| Path | What |
|---|---|
| `runtimes/claude-code.yaml` | Claude Code over ACP: `npm:@agentclientprotocol/claude-agent-acp` and node, installed with mise; secret `claude_token` (user scope, exposed as `CLAUDE_CODE_OAUTH_TOKEN`) |
| `steps/` | One unit of sandbox work each. Agent steps: `triage`, `plan`, `breakdown`, `implement`, `review`, `address`, `ci_fix`, `resolve_conflicts`, `task`. Command steps: `context` (merge base, changed files, affected components), `prepare`, `checks` |
| `prompts/` | The agent prompts, one per agent step |
| `phases/` | Subgraphs with params and exits: `dependencies`, `start_gate`, `triage`, `plan`, `breakdown`, `implement`, `review` (runs the internal `review_rounds`), `merge`, `release`, `mention` |
| `workflows/` | `issue`, `epic`, `pull_request`, `mention` |
| `board.yaml` | Backlog, Plan, In progress, Blocked, Review, Done (limit 20) |
| `bindings.yaml` | Library default bindings |
| `tests/` | Scenario tests, run by `enzarb-lint test` |
| `examples/` | `enzarb.yaml` (the proposed `.enzarb/workflows.yaml` of enzarb/enzarb), `auto-merge.yaml` |
| `lint/action.yml` | The `enzarb/workflows/lint@v1` GitHub Action |

## Workflows

- **`issue`**: dependencies → start gate (`/plan`) → triage → plan (approval) → implement →
  review → merge → release. Triage `epic` (or `/epic` during plan approval) goes to the
  breakdown instead: sub-issues are created on approval and the epic ends when they all ended.
- **`epic`**: an issue opened with the `epic` label; triage always continues to the breakdown.
- **`pull_request`**: a PR that references no issue: review → merge → release. A PR that
  references an issue joins that issue's task (the engine links it), so it is reviewed by
  the issue workflow's review phase.
- **`mention`**: `@enzarb …` on an object with no live task: one read-only `task` step, reply posted.

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
phases:
  - {use: std/phases/dependencies}
  - {use: std/phases/start_gate, with: {command: start}}
  - {use: std/phases/triage, exits: {epic: breakdown}}
  - {use: std/phases/plan, with: {prompt: {file: .enzarb/prompts/plan.md}}, exits: {epic: breakdown}}
  - {use: std/phases/implement}
  - {use: std/phases/review, exits: {merged: release}, max_entries: 10}
  - {use: std/phases/merge, with: {mode: auto_when_green}, exits: {changed: review}, max_entries: 10}
  - {use: std/phases/release, with: {when: [release.published]}, exits: {done: {end: done}}}
  - {use: std/phases/breakdown}
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
| `epics` | `issue` | opened with the `epic` label | `workflows/epic` |
| `issues` | `issue` | opened | `workflows/issue` |
| `adopted_prs` / `adopted_issues` | `adopted` | by subject kind | `workflows/pull_request` / `workflows/issue` |
| `pull_requests` | `pull_request` | opened, references no issue | `workflows/pull_request` |
| `mentions` | `comment` | body contains `@enzarb`, not a bot | `workflows/mention` |

## Phase params and exits

Every phase also has the implicit exits `done` (next phase) and `limit_reached`.

| Phase | Params (default) | Exits | Waits (reason · column) |
|---|---|---|---|
| `dependencies` | `done_statuses` ([done, merged, released]), `unblock_command` (unblock) | | `blocked_by_dependencies` · blocked; button "Start anyway" |
| `start_gate` | `command` (plan), `label` (plan) | `closed` | `awaiting_start` · backlog; button "Start planning" |
| `triage` | `prompt`, `epic` (false), `epic_allowed` (true) | `epic`, `duplicate`, `closed`, `failed` | `awaiting_answers`, `stale` · blocked (stale after 14 days) |
| `plan` | `prompt`, `approve_command` (approve), `epic_command` (epic) | `epic`, `closed`, `failed` | `awaiting_approval` · plan; button "Approve"; any other comment re-plans |
| `breakdown` | `prompt`, `max_subissues` (8), `approve_command` (approve) | `closed`, `failed` | `awaiting_approval` · plan; `awaiting_children` · in progress |
| `implement` | `prompt`, `prepare` (mise run prepare), `check` (mise run check), `retry_command` (retry), `draft` (false) | `closed` | `implement_blocked` · blocked; button "Retry" |
| `review` | `review_prompt`, `address_prompt`, `ci_fix_prompt`, `resolve_prompt`, `ci_ignore` (Copilot) | `merged`, `closed` | `awaiting_ci` · review; `awaiting_review`, `round_limit` · review; `ci_failed`, `conflict_parked` · blocked |
| `merge` | `mode` (human \| auto_when_green), `command` (merge), `method` (squash), `delete_branch` (true), `retry_command` (retry) | `changed` (back to review), `closed` | `checking_merge`; `ready_to_merge` · review, button "Merge"; `merge_failed` · blocked |
| `release` | `when` ([]: `<type>` or `<type>.<action\|state>`, e.g. `release.published`, `package.published`, `deployment.success`) | `failed` | `awaiting_release` · done |
| `mention` | `prompt` | `failed` | |

Review policy (from the compiled `change_request` machine): enzarb's must-fix findings get
3 automatic fix rounds, red GitHub Actions checks of the repository get 3 CI-fix rounds with
the failing logs (checks whose name starts with `ci_ignore` never count; other failing
checks wait on a person), and a merge conflict gets 2 resolution attempts (merge, never
rebase). A person's comment or review is addressed; a push by someone else and `/retry`
start a fresh cycle with fresh caps. Nothing merges without a writer's Merge click or
`/merge` unless the merge phase runs with `mode: auto_when_green`; the merge is pinned to the
head CI passed on (`head_sha`).

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
