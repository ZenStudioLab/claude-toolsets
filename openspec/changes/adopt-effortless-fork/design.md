# Design

## Context

See `proposal.md` (Why) and the two delta specs for requirements. The relevant facts about upstream at the reviewed commit `bfb85cda60055a89946f610959b479d82609c67a` (upstream `main`, still its head on 2026-10-16):

- **Update paths in `hooks/register.tsx`.** There are three ways new code can arrive:
  - The 6-hour timer at `startTimers` (`:2706`) and `afterLoad` (`:2759`) both call `checkUpdate` (`:2761`). That runs `latestAvailable` (`:2682`), which runs `latestFromGit` (`:2668`, a `git fetch` of HeyCubit `stable`), falls back to `RELEASES_URL` (`:2618`), and runs a background `claude plugin marketplace update effortless` (`:2694`).
  - `runUpdate` (`:2840`) runs `claude plugin marketplace update` and `claude plugin update`, then `syncRunningCopy` (`:2883`, `cp -R` over the running copy) and `/reload-plugins`. It is reached from the update card buttons (`:2935`, `:2947`), the settings panel (`:4713`) and `/effortless update` (`:3924-3929`).
  - `checkUpdateNow` (`:2784`) sits behind the settings "Check for updates" button (`:4717`).
- **Install identity.** `pluginId` (`:3146`) reads the marketplace name from the install path (`…/cache/<marketplace>/…`) and defaults to `effortless` when the path has no `cache` segment. That is the case under the upstream test harness: `tests/effortless.test.ts` expects `effortless@effortless` (`:3579`, `:3714`).
- **Upstream tests.** `tests/effortless.test.ts` (3986 lines, `describe('updates')` at `:3356`) exercises the updater. `tools/test.sh` runs every `*.test.ts` except `textless/` through `claude plugin test`.
- **Marketplace format.** The marketplace JSON accepts `{"source":"url","url":…,"sha":<40 hex>}`. Upstream uses it (with `ref`) and 164 entries in `claude-plugins-official` use it, so the pin format is proven on Claude Code 2.1.301.
- **`claude-toolsets` itself.** It has no commits and no remote yet, and `docs/adr/` does not exist. The operator's GitHub account is `ZenStudioLab`, and `ZenStudioLab/effortless` does not exist yet.

## Goals / Non-Goals

**Goals:**

- The code that runs is exactly the pinned fork commit, and it changes only through a deliberate pin bump.
- Each upstream sync costs one gate run, one review and one merge, with almost no conflicts.

**Non-Goals:**

- Restructuring the 345 KB `hooks/register.tsx`. Every structural edit becomes a merge conflict while we track upstream. This file-size exception is deliberate and lasts as long as we track upstream.
- Changing Jev, the Haiku judge, the informational links (`REPO_URL`, `SITE_URL`, `REPORT_BUG_URL`) or the uninstall button. None of them can deliver new code.
- Hosting or replacing the textless art service, since textless is not installed.
- CI automation for the gate. It is an operator-run script.

## Decisions

### D1 — Fork starting point: upstream `main` at `bfb85cd`, not `stable`

`stable` (`d4cd7d6`) split from `main` after `d45501c` and was never reviewed. Tracking `main` means every gate run diffs against one line of history.
- *Alternative:* `stable`. Rejected: it was never reviewed, and upstream copies release commits onto it.

Create the fork with `gh repo fork HeyCubit/effortless --clone=false --default-branch-only`. Then check that `ZenStudioLab/effortless` `main` equals `bfb85cd` before adding any commit. If upstream has moved by then, reset the fork's `main` to `bfb85cd`; the fork is brand new, so a force-push there loses nothing.

### D2 — How the lockdown switches on: by install identity

Add one exported pure function, `updatesLocked(root: string)`. It takes the plugin's install folder (`$.plugin.root`) and returns `true` unless that folder's marketplace part is `effortless` or `effortless-dev`, upstream's own names. To get the marketplace part, it uses the same `cache/<marketplace>/` regex as `pluginId`. A root without a `cache/` segment resolves to upstream's `effortless`, which is the same fallback `pluginId` uses. The update functions call `updatesLocked($.plugin.root)`. The test calls it with the harness's `$.plugin.root`, so there is a single, testable source of the identity. The guards are:

1. an early return at the top of `checkUpdate`, which covers the timer, `afterLoad`, `/effortless update` and `checkUpdateNow`;
2. an early return at the top of `runUpdate`, which covers every button and the settings panel and is the only caller of `syncRunningCopy`;
3. when locked, `checkUpdateNow` and `/effortless update` reply "updates are managed by the pinned zenstudiolab marketplace".

Why identity rather than an always-on constant:
- Upstream's own update tests run as `effortless@effortless`, so they keep passing unchanged. We never edit `tests/effortless.test.ts`, the largest source of conflicts.
- Any install under a different marketplace name, ours included, is locked. That is the safe direction.

Alternatives:
- *An always-on constant.* Rejected: it breaks the upstream `describe('updates')` tests, which we would then have to edit on every merge.
- *An environment-variable switch.* Rejected: it gives a second way to turn updates back on.

### D3 — Lockdown test lives in fork-only files and runs under the locked identity

The update functions read their identity from `$.plugin.root` (`:3146`) and take no identity argument. So the integration test must run the plugin from a folder that resolves to `effortless@zenstudiolab`. `claude plugin test` runs every `*.test.ts` in one run with one plugin root, so the locked and unlocked runs are separate.

`tests/lockdown.test.ts` uses the same `claude-code/testing` harness:
- a unit test of `updatesLocked` for roots under `cache/effortless/`, `cache/effortless-dev/` and `cache/zenstudiolab/`, and for a root with no `cache/` segment; it always runs;
- an integration test. It fires `session.start`, advances the clock past `UPDATE_CHECK_MS`, runs `/effortless update` and presses "Check for updates". Then it asserts no `http.fetch` to `github.com`/`api.github.com`, no `process.run` of `git` or `claude`, a refusal reply, and no update card. It runs only when `updatesLocked($.plugin.root)` is `true` and reports itself skipped otherwise, so upstream's own `tools/test.sh` run (identity `effortless`) stays green.

How the locked run gets its identity, decided in task 2.1:
1. if the harness can override `$.plugin.root`, the test sets it to a path under `cache/zenstudiolab/`;
2. otherwise, `tools/lockdown-test.sh` (fork-only) copies the plugin files into `$(mktemp -d -p /tmp)/cache/zenstudiolab/effortless/<version>/`.
   - **What it copies:** tracked files without `textless/` and without `tests/effortless.test.ts`, so `tests/lockdown.test.ts` is the only test file. Upstream's update tests would fail under the locked identity by design.
   - **How it runs:** `claude plugin test "<absolute path>"`.
   - **Why the identity holds:** the regex matches the substring `cache/` anywhere. A `/tmp` parent that contains no `cache` substring, plus an absolute path, make the identity resolve to `zenstudiolab`.

If neither path yields the identity `effortless@zenstudiolab` inside the test, the worker stops and the Orchestrator decides. Any locked run that skips the integration test counts as a failure.
- *Alternative:* adding the test to `tests/effortless.test.ts`. Rejected: conflicts on merge.

### D4 — Gate: `tools/upstream-gate.sh` in the fork, inspect-only

1. Add remote `upstream` (`https://github.com/HeyCubit/effortless.git`) if it is missing.
2. `git fetch upstream main`, then list `HEAD..upstream/main`.
The script takes one optional argument, the upstream ref to inspect (default `upstream/main`), so it can be checked against a throwaway local ref.

3. Diff `HEAD...<ref>` and sort every changed path into one of three groups. The marketplace installs the whole repository root, so every path not in group (a) or (b) lands in the installed copy.
   - **(a) Not loaded or run:** `textless/`, `site/`, `docs/`, `report-worker/`, `.github/`. Listed, not flagged.
   - **(b) Run on the host during verification:** `tests/`, `tools/`. Listed in their own section, to review before running `tools/test.sh`, and they make the gate exit non-zero. Pattern flags don't apply here, because the test mocks are full of `http.fetch`/`process.run`.
   - **(c) Installed:** everything else, `bench/` included, since `/effortless bench` reads `bench/judge-cases.json` (`:4019`).
4. In group (c), always flag:
   - any newly added path;
   - any change to a plugin component file: `.claude-plugin/`, `hooks/hooks.json`, `commands/`, `.mcp.json`, `agents/`, `skills/`, `output-styles/`, `settings.json`, `.lsp.json`, `bin/`.

   Also flag any added line matching:
   - network or remote addresses: URLs, `fetch(`, `$.http`;
   - processes: `$.process`, `child_process`, `spawn`, `exec`;
   - file writes: `$.fs.write`, `$.fs.remove`;
   - environment and credentials: `$.env`, `process.env`, `.ssh`, `.aws`, `token`, `secret`;
   - code evaluation and encoding: `eval`, `new Function`, `atob`, `base64`, `fromCharCode`;
   - new code loading or command paths through existing helpers: `import `, `require(`, `runClaude(`, `$.command`, and any reference to a group (a) folder name (`textless`, `site/`, `docs/`, `report-worker`).
5. Exit non-zero if anything is flagged or group (b) is non-empty. The script never runs, builds or tests upstream code. `tools/test.sh` runs only after the merge, on our own branch, once the group (b) listing has been reviewed.

The gate lives in the fork, as the user chose (Q9a). Upstream has no file at that path, so an upstream change to it would show up as a merge conflict or in the gate's own group (b) listing.
- *Alternative:* a written checklist. Rejected by the user in favour of a repeatable script.

### D5 — Marketplace in `claude-toolsets`, name `zenstudiolab`

`claude-toolsets/.claude-plugin/marketplace.json` lists only `effortless`, as `{"source":"url","url":"https://github.com/ZenStudioLab/effortless.git","sha":"<40 hex>"}`. There is no `ref`, so the commit `sha` is the only selector.

The name `zenstudiolab` doesn't collide with upstream's `effortless`. That also defuses the hardcoded `marketplace update effortless` at `:2694` if a future merge brings it back.

Add the marketplace from the local `claude-toolsets` folder first (Q8a). Switch to the GitHub-hosted repository after `claude-toolsets` is published (Q8b).
- *Alternative:* a marketplace inside the fork. Rejected: the pin and the code it pins would live in the same repo.

### D6 — Submodule is for editing, not installing

`plugins/effortless` → `https://github.com/ZenStudioLab/effortless.git`, with a local `upstream` remote added by the gate. Claude Code installs from the marketplace's pinned URL, never from the submodule's working tree, so uncommitted edits there can never be installed.

### D7 — An ADR records the rule

`docs/adr/0001-pinned-forks-for-third-party-plugins.md` records the rule for future plugins in `claude-toolsets`: fork, lock down self-update, gate syncs, and pin by `sha`.

## Risks / Trade-offs

- **[Risk] Upstream renames or restructures the update functions.** The early returns could end up in dead code while a new updater runs.
  → Mitigation: the lockdown test, the gate's `$.process`/`fetch` flags, and the rule that the pin never moves without both passing.
- **[Risk] A development load from outside the plugin cache** (e.g. `--plugin-dir plugins/effortless`) gets the identity `effortless` and is not locked, so it would contact upstream.
  → Mitigation: install only through `zenstudiolab`; the ADR says so. A dev load fetches only a version number and never installs anything unless the Update button is pressed.
- **[Risk] Pattern-based gate flags can miss obfuscated changes.**
  → Mitigation: the gate lists every incoming commit for review, flags encoded and evaluated code explicitly, and manifest changes always flag.
- **[Trade-off] A public fork exposes our lockdown changes.** It holds no secrets, and the user accepted this.
- **[Trade-off] Jev stays.** In `auto` mode, a set `TYPESAFE_API_KEY` sends prompt excerpts to `api.typesafe.ai`. Accepted by the user (Q4).

## Migration Plan

1. Write the ADR.
2. Get one explicit operator approval covering: creating the public fork, pushing to it, running plugin tests locally, and installing the plugin.
3. Create the fork at `bfb85cd` and add the submodule.
4. In the fork, add the lockdown, its test and the gate. Run `tools/test.sh`, and also `tools/lockdown-test.sh` if it exists (D3 path 2). Confirm the locked integration test was executed, not skipped, then push.
5. Write the `claude-toolsets` marketplace with the lockdown commit `sha`, add it locally, and install `effortless@zenstudiolab`.
6. Once `claude-toolsets` is published, re-add the marketplace from GitHub.

**Rollback:** `claude plugin uninstall effortless@zenstudiolab` and `claude plugin marketplace remove zenstudiolab`. The fork and submodule can stay, since they never execute on their own.

## Open Questions

- Visibility of the published `claude-toolsets` repository (step 6). It is decided when that step runs and doesn't affect earlier steps.
