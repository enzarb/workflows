# Changelog

## 1.0.0 (unreleased)

First version of the standard workflow library (ADR-0022 milestone M2): today's compiled
`issue`, `change_request` and `mention` behaviour expressed as YAML.

### Added

- `runtimes/claude-code`: Claude Code over ACP (`npm:@agentclientprotocol/claude-agent-acp@0.92.0`, node 22, via mise).
- Steps and prompts: `triage`, `plan`, `breakdown`, `implement`, `review`, `address`,
  `ci_fix`, `resolve_conflicts`, `task`; command steps `context` (merge base, changed files,
  affected components), `prepare` and `checks`.
- Phases: `dependencies`, `start_gate`, `triage`, `plan`, `breakdown`, `implement`, `review`
  (with the internal `review_rounds`), `merge` (`human` or `auto_when_green`), `release`, `mention`.
- Workflows: `issue` (epics through its `epic` param), `pull_request`, `mention`.
- `board.yaml`, `bindings.yaml`, scenario tests under `tests/`.
- `lint/` composite action (`uses: enzarb/workflows/lint@v1`) and the `lint` CI workflow.

### Uses the interpreter building blocks (ADR-0022 implementation notes)

- `decide` nodes: `dependencies` passes at once when its known dependencies are finished;
  `release` with `when: []` is done at once (no more waiting on a refreshed `merged` fact);
  `plan` leaves through `epic` when `vars.epic` is set.
- Task variables: `vars.fix_rounds` (reset by a person's comment or review, a push, `/retry`),
  `vars.conflict_base`/`vars.conflict_attempts` (attempts per base sha), `vars.epic`
  (remembered across phases, also from the `epic` label), `vars.reviewed_sha` (passed to
  the review prompt as `previous_reviewed_sha` for a delta review), `vars.merged`.
- Params for literal fields: `max_visits: {param: max_rounds}` (implement),
  `{param: max_ci_rounds}` (review), triage timer `stale_after` (`14d`), every button's
  label and command.
- Filters: `create_issues`/`open_pull_request` args are structured (typed leaves),
  `context` shell-quotes the base ref with `shell_quote` and records `base_sha`,
  `ci_ignore` is a `regex_match` pattern (default `(?i)copilot`).
- Actions: `open_pull_request` has `subject: true` and `as: pull_request`;
  `create_issues` has `as: children` and relates every created issue (`relate.from`).
- Bindings: `epics` runs `workflows/issue` with `epic: true` (`workflows/epic` is removed);
  `issues` passes `epic_allowed: false` for sub-issues (a `parent` relation); `mentions`
  matches `event.mentioned` instead of the text `@enzarb`.
- Workflow `on:` handlers: closing the issue (unless by the merge) or closing the PR without
  merging ends the task anywhere, stopping a running step.
- Workflow params: `issue` (`epic`, `epic_allowed`, `epic_label`, `stale_after`, `max_rounds`,
  `release_when`), `pull_request` (`max_rounds`, `release_when`).
- Runtime secret `{name: claude_token, scope: user, env: CLAUDE_CODE_OAUTH_TOKEN}`.
- Prompts and texts use `instance.bot.handle` instead of the name `enzarb`.
- Write workspaces: `implement` on `enzarb/issue-<n>` from the default branch; `address`,
  `ci_fix` and `resolve_conflicts` on the PR head against its base.
- Scenarios for each of these (`dependencies_known`, `epic_label`, `epic_label_later`,
  `epic_not_allowed`, `cancel_while_running`, `pr_closed_while_fixing`, `issue_release_when`,
  `review_fix_rounds_param`, `review_conflict_per_base`, and checks added to existing ones).
