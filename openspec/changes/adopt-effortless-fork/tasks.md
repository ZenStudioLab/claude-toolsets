# Tasks

Routing for `/opsx-apply-loop`:
- **Reviewer:** `oracle`.
- **Default implementation owner:** `worker`.
- **Modes:** focus mode `focus`, orchestrator mode `normal`.

The Orchestrator owns ADR and spec edits, every outward-facing or global-config action (GitHub fork and push, `~/.claude` marketplace and plugin changes) and every approval checkpoint. Each wave uses at most one `oracle` instance plus up to three other instances, and reuses an eligible instance when one is enough.

**Oracle wave gate.** These rules apply to every task labelled "Oracle wave gate" below:
- `oracle` reviews the whole completed wave, never a single task, and emits findings plus exactly one advisory `PROPOSED_REVIEW_ACTION`.
- The Orchestrator reconciles every finding (`ACCEPT_BLOCKING` requires a cited governing source) and records the only authoritative `REVIEW_ACTION`: `PASS`, `PATCH`, `REBUILD` or `HUMAN_DECISION`.
- `PATCH` or `REBUILD` returns work to the same wave.
- A sequence allows at most four Oracle rounds and one recorded rebuild.
- Before a terminal `PASS` or `HUMAN_DECISION`, check the latest candidate revision and the frozen baseline.

## 1. Decision record, approval, fork and submodule (Owner: Orchestrator)

- [x] 1.1 Write `docs/adr/0001-pinned-forks-for-third-party-plugins.md` (D7): third-party plugins are installed only from a `ZenStudioLab` fork, with self-update locked down, upstream syncs gated, and the marketplace pinned by full `sha`. Development loads from outside the plugin cache are not trusted installs. Verify the file exists and that its decision matches design D2–D5.
- [x] 1.2 Approval checkpoint. Show the operator the exact commands and ask one explicit yes/no covering: (a) `gh repo fork HeyCubit/effortless --clone=false --default-branch-only` as a public repo; (b) pushing lockdown commits to `ZenStudioLab/effortless` `main`; (c) running `tools/test.sh` locally, which executes the reviewed upstream code plus our changes; (d) adding the `zenstudiolab` marketplace and installing `effortless@zenstudiolab` into `~/.claude`. Verify the answer is recorded in the review log. Without approval, stop the affected items.
- [x] 1.3 Create the fork. Verify `gh api repos/ZenStudioLab/effortless --jq .fork` is `true` and `git ls-remote https://github.com/ZenStudioLab/effortless.git refs/heads/main` returns `bfb85cda60055a89946f610959b479d82609c67a`. If it doesn't, reset the fork's `main` to that commit (D1) and verify again.
- [x] 1.4 Run `git submodule add https://github.com/ZenStudioLab/effortless.git plugins/effortless` in `claude-toolsets` and check out `bfb85cd`. Verify `.gitmodules` names the fork URL, `git -C plugins/effortless rev-parse HEAD` is `bfb85cd`, and the untracked `.claude/`, `.pi/` and `openspec/` folders are untouched.
- [x] 1.5 Oracle wave gate (`oracle`) for wave 1: review the ADR, the fork state and the submodule against the specs and design, following the wave-gate rules above.

## 2. Fork lockdown, lockdown test and upstream gate (Owner: worker, focus-grouped; push by Orchestrator)

- [x] 2.1 Decision gate for D3. In `plugins/effortless`, find out whether `claude-code/testing` lets a test override `$.plugin.root`. If it does, use D3 path 1. If not, use D3 path 2 (`tools/lockdown-test.sh` with a folder-placed copy). If neither yields the identity `effortless@zenstudiolab` inside the test, stop and hand the decision to the Orchestrator. Verify by recording the finding, with file and line evidence, and the chosen path in the wave report.
- [x] 2.2 In `hooks/register.tsx`, add the exported pure `updatesLocked(root)`, reusing `pluginId`'s `cache/<marketplace>/` regex (D2), plus the early returns in `checkUpdate` and `runUpdate` and the locked replies for `checkUpdateNow` and `/effortless update` (D2). Verify that `git diff bfb85cd -- hooks/register.tsx` touches only those functions, `pluginId` if the regex is shared, and the new export.
- [x] 2.3 Add `tests/lockdown.test.ts`, plus `tools/lockdown-test.sh` if 2.1 chose path 2 (D3). The unit test covers `updatesLocked` for roots under `cache/effortless/`, `cache/effortless-dev/` and `cache/zenstudiolab/`, and for a root without `cache/`. The integration test runs only when `updatesLocked($.plugin.root)` is true (D3) and asserts: no `http.fetch` to `github.com`/`api.github.com` and no `git`/`claude` `process.run` after `session.start` and a clock advance past `UPDATE_CHECK_MS`; a refusal reply from `/effortless update` and from "Check for updates" with no process started; and no update card. Verify that:
  - `tools/test.sh` shows no failure absent from a baseline run of the same upstream commit (upstream has timing-sensitive tests that fail at baseline; re-run a differing one alone before calling it a regression), with upstream's `tests/effortless.test.ts` unchanged and the lockdown integration test reported as skipped there;
  - the locked run passes with its integration test actually executed, not skipped;
  - temporarily removing the `checkUpdate` guard and then the `runUpdate` guard each makes the locked run fail; restore both afterwards.
- [x] 2.4 Add `tools/upstream-gate.sh` (D4), inspect-only. It takes an optional ref argument (default `upstream/main`), adds the `upstream` remote if missing, fetches `main`, lists incoming commits, sorts paths into groups (a)/(b)/(c), flags, and exits non-zero on any flag or any group (b) change. Verify on the real upstream:
  - the reported incoming-commit count equals `git rev-list --count HEAD..upstream/main`;
  - the exit code matches whether anything was flagged.

  Then verify against throwaway local refs, one each:
  - adding `$.process.run` in `hooks/`: flagged with file and line, non-zero exit;
  - adding a `runClaude(` call or an `import … from '../docs/…'` line in `hooks/`: flagged, non-zero exit;
  - adding `.mcp.json`: flagged as a component file, non-zero exit;
  - adding a new file under `bench/`: flagged as a new installed path, non-zero exit;
  - changing only `tests/`: listed under "runs under tools/test.sh", non-zero exit;
  - changing only `textless/`: listed as not loaded, exit 0.

  Delete the throwaway refs afterwards.
- [x] 2.5 Commit the lockdown, test and gate in the fork. Verify that `git diff --stat bfb85cd HEAD` lists only `hooks/register.tsx`, `tests/lockdown.test.ts`, `tools/upstream-gate.sh`, and `tools/lockdown-test.sh` if 2.1 chose path 2. Then the Orchestrator pushes to `ZenStudioLab/effortless` `main` (approved in 1.2) and verifies the remote `main` sha equals the local `HEAD`.
- [x] 2.6 Oracle wave gate (`oracle`) for wave 2: review the fork diff, the lockdown test, the gate script and its recorded verification against the `effortless-fork-lockdown` spec, following the wave-gate rules above.

## 3. Pinned marketplace and local install (Owner: Orchestrator)

- [x] 3.1 Write `claude-toolsets/.claude-plugin/marketplace.json` (D5), with name `zenstudiolab` and one plugin, `effortless`, sourced as `{"source":"url","url":"https://github.com/ZenStudioLab/effortless.git","sha":"<full sha of the pushed fork HEAD from 2.5>"}` with no `ref`. Verify the JSON parses, the `sha` is 40 hex characters and equals the fork's remote `main`, and `textless` is not listed.
- [x] 3.2 Run `claude plugin marketplace add /mnt/031D13CB0120451A/Src/Tools/claude/claude-toolsets` and then `claude plugin install effortless@zenstudiolab` (approved in 1.2). Verify that `claude plugin marketplace list` shows `zenstudiolab` and no `effortless` or `effortless-dev`, and that `~/.claude/plugins/installed_plugins.json` records `effortless@zenstudiolab` with an install path under `cache/zenstudiolab/` at the pinned `sha`.
- [x] 3.3 Check the running plugin. In a new Claude Code session, `/effortless update` must reply that updates are managed by the pinned `zenstudiolab` marketplace, and `/effortless debug` must show no update check. Verify that `claude plugin update effortless@zenstudiolab` leaves the installed `sha` unchanged. If a behaviour can only be observed interactively, record it as operator-observed.
- [x] 3.4 Stage the `claude-toolsets` changes (`.gitmodules`, `plugins/effortless`, `.claude-plugin/marketplace.json`, `docs/adr/0001-*.md`) without staging unrelated untracked files. Verify that `git status --short` shows exactly those staged paths.
- [x] 3.5 Oracle wave gate (`oracle`) for wave 3: review the marketplace file, the install evidence and the staged scope against the `vetted-plugin-marketplace` spec, following the wave-gate rules above.

## 4. Switch the marketplace to the published `claude-toolsets` (Owner: Orchestrator)

- [x] 4.1 Ask the operator to choose a visibility for the `claude-toolsets` GitHub repository and to confirm creating it and pushing to it (outward-facing). After approval, make the first commit, create `ZenStudioLab/claude-toolsets` and push. Verify `git ls-remote origin` matches the local `HEAD`.
- [x] 4.2 Replace the local marketplace with the GitHub-hosted one under the same name: `claude plugin marketplace remove zenstudiolab`, then `claude plugin marketplace add ZenStudioLab/claude-toolsets`, then reinstall `effortless@zenstudiolab` if Claude Code requires it. Verify that `known_marketplaces.json` shows the GitHub source for `zenstudiolab` and the installed `effortless` is at the same pinned `sha` as in 3.2.
- [x] 4.3 Oracle wave gate (`oracle`) for wave 4: review the published marketplace switch against the "Marketplace location" requirement, following the wave-gate rules above.

## Workflow follow-up

- On every later upstream sync: in `plugins/effortless`, run `tools/upstream-gate.sh` and review every flagged line and the "runs under tools/test.sh" listing. Merge `upstream/main` and run `tools/test.sh`; it must show no failure absent from a baseline run of the merged upstream commit (re-run a differing timing-sensitive test alone). Run `tools/lockdown-test.sh` if it exists and confirm it passes with the locked integration test executed, not skipped. Then push the fork. Then bump the `sha` in `claude-toolsets/.claude-plugin/marketplace.json` in its own commit and run `claude plugin marketplace update zenstudiolab` and `claude plugin update effortless@zenstudiolab`.
- Archive the change once the review requirements are met.
