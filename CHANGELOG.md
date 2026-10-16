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
- Workflows: `issue`, `epic`, `pull_request`, `mention`.
- `board.yaml`, `bindings.yaml`, scenario tests under `tests/`.
- `lint/` composite action (`uses: enzarb/workflows/lint@v1`) and the `lint` CI workflow.
