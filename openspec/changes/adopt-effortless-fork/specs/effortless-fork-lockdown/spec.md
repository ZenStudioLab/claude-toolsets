# Spec Delta

## Purpose

Makes sure our `effortless` fork never updates itself and only takes upstream code that has passed a reviewed security gate. The code that runs is then always the code we pinned.

## ADDED Requirements

### Requirement: No automatic update checks
When installed from any marketplace other than upstream's own (`effortless`, `effortless-dev`), the forked plugin SHALL NOT contact any update source by itself. That covers no fetch of upstream `public.json` or GitHub contents, no `git fetch` of any upstream branch, and no background `claude plugin marketplace update`, whether at session start, on reload, or on a timer.

#### Scenario: Session start and timers stay quiet
- **WHEN** a session starts with the forked plugin installed from `zenstudiolab` and its periodic timers have fired
- **THEN** the plugin has made no HTTP request to `api.github.com` or `github.com`
- **AND** it has run no `git` or `claude` process for update purposes

### Requirement: No self-update or self-overwrite
When installed from any marketplace other than `effortless` or `effortless-dev`, the forked plugin SHALL NOT install or update itself through its update UI or the `/effortless update` command, and SHALL NOT copy files over its running copy. Updates happen only by moving the marketplace pin.

#### Scenario: Update command is refused
- **WHEN** the operator runs `/effortless update`
- **THEN** the plugin answers that updates are managed by the pinned `zenstudiolab` marketplace
- **AND** no `claude plugin` process, file copy or plugin reload is started

#### Scenario: Check for updates button
- **WHEN** the operator presses "Check for updates" in the plugin settings
- **THEN** no network request or process is started
- **AND** the button reports that updates are managed by the pinned marketplace

#### Scenario: No update card appears
- **WHEN** a newer upstream version exists
- **THEN** the plugin never shows an update offer card

#### Scenario: A development load is not a trusted install
- **WHEN** the fork is loaded from outside the plugin cache, for example with `--plugin-dir`, so its identity resolves to upstream's `effortless`
- **THEN** the lockdown does not apply, and that load does not count as an installation under this capability

### Requirement: Lockdown is guarded by a test
The fork SHALL contain an automated test of these no-update requirements, in files upstream does not have. It SHALL run the plugin under the identity `effortless@zenstudiolab` and SHALL fail when an update-related network request or process starts there. Runs under upstream's identity SHALL skip its locked-only checks.

#### Scenario: Merged upstream code re-enables updates
- **WHEN** a merge brings in code that fetches an update source at session start or on a timer
- **THEN** the lockdown test fails
- **AND** the merge is not pinned in the `zenstudiolab` marketplace

### Requirement: Upstream changes pass a security gate
Before any upstream merge, the operator SHALL run the fork's upstream gate on an upstream ref (default `upstream/main`). It SHALL list incoming commits and flag added lines in installed files that touch network, processes, commands, filesystem writes, environment or credentials, code evaluation, encoded payloads, or new imports. It SHALL always flag new installed paths and plugin component files.

#### Scenario: Clean report
- **WHEN** the gate runs against pending upstream commits that add none of the flagged patterns, no new installed paths, no component-file changes and no host-run changes
- **THEN** it lists the incoming commits, reports no flags and exits with status 0

#### Scenario: New installed path or component file
- **WHEN** incoming commits add a new path outside the not-loaded folders, or change a component file such as `.claude-plugin/`, `hooks/hooks.json`, `commands/`, `.mcp.json`, `agents/`, `skills/`, `output-styles/`, `settings.json`, `.lsp.json` or `bin/`
- **THEN** the gate flags it whatever its content and exits non-zero

#### Scenario: Changes that run on the host during verification
- **WHEN** incoming commits change `tests/` or `tools/`
- **THEN** the gate lists them in a separate "runs under tools/test.sh" section for review before tests are run, and exits non-zero

#### Scenario: Flagged report blocks the merge
- **WHEN** pending upstream commits add a flagged pattern, such as a new URL, a `$.process.run`, a call to an existing process helper such as `runClaude(`, a `$.command` call, a new `import`, a `$.fs.write` or a manifest change
- **THEN** it prints each flagged line with file and position and exits non-zero
- **AND** the merge proceeds only after the operator has reviewed each flagged line

#### Scenario: Files the plugin never loads or runs
- **WHEN** incoming upstream commits change only `textless/`, `site/`, `docs/`, `report-worker/` or `.github/`
- **THEN** the gate lists those changes as not loaded and does not flag them

#### Scenario: The gate inspects without executing
- **WHEN** the gate runs
- **THEN** it only fetches and diffs; it does not run, build or test any upstream code

### Requirement: Fork tracks upstream main with small changes
The fork SHALL track upstream `main`. Our changes SHALL stay confined to the lockdown edits, the lockdown test and the gate, and the upstream `textless` folder SHALL stay unmodified.

#### Scenario: Fork starting point
- **WHEN** the fork is created
- **THEN** its `main` starts from the reviewed upstream commit `bfb85cda60055a89946f610959b479d82609c67a`, plus the lockdown commit(s)

#### Scenario: Diff against upstream stays small
- **WHEN** the operator diffs the fork's `main` against the merged upstream commit
- **THEN** only the update code in `hooks/register.tsx`, `tests/lockdown.test.ts`, `tools/upstream-gate.sh` and the lockdown test runner `tools/lockdown-test.sh` differ
