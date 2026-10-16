# Review Log: adopt-effortless-fork

## Review Context

- review_mode: change_bundle
- review_subject: adopt-effortless-fork
- selected reviewer: oracle
- selected default implementation specialist: worker
- focus mode: focus
- orchestrator mode: normal
- ADR check: needed. ADR-0001 (pinned forks for third-party plugins) is task 1.1, owned by the Orchestrator, and must be done before implementation proceeds. `docs/adr/` does not exist yet and is created by that task.
- User decisions (explore session, 2026-10-16):
  - Q1: track upstream with small changes and a security gate before each sync.
  - Q2: fork from `bfb85cd`.
  - Q3: originally both plugins; revised by the user to `effortless` only ("we don't need textless").
  - Q4: keep Jev opt-in.
  - Q5: marketplace in `claude-toolsets`, pinned by `sha`.
  - Q6: public fork under the current account (`ZenStudioLab`).
  - Q7: moot once textless was dropped.
  - Q8: local folder first, then GitHub.
  - Q9: gate as a script in the fork.

## Frozen Baseline

BASELINE_TARGET=openspec/changes/adopt-effortless-fork (apply-ready bundle: proposal.md, design.md, tasks.md, specs/effortless-fork-lockdown/spec.md, specs/vetted-plugin-marketplace/spec.md)
BASELINE_TARGET_REVISION=sha256 proposal=6e9b23f1 design=f26dc8b0 tasks=0c07aed6 lockdown-spec=5049038b marketplace-spec=107f0244
BASELINE_SOURCE=user decisions Q1-Q9 recorded above; upstream HeyCubit/effortless at bfb85cda60055a89946f610959b479d82609c67a (static review copy at /tmp/effortless-review.*/repo); Claude Code 2.1.301 marketplace format as used in ~/.claude/plugins/marketplaces/claude-plugins-official
BASELINE_INCLUSION_PREDICATE=the five bundle files listed in BASELINE_TARGET
BASELINE_EXCLUSIONS=implementation in the fork or claude-toolsets (not yet started); upstream code quality beyond the update and lockdown surface; textless and its market service; Jev behaviour

---

<!-- Oracle round 1 section transcribed by the caller from the oracle run output (oracle had no file-write tool); run 6c39ea00. -->

## Round 1 - Oracle Review

Review mode: `FULL` · Reviewer: `oracle` · Target: change bundle `adopt-effortless-fork` (5 files). Before reviewing I recomputed the sha256 prefixes: proposal 6e9b23f1, design f26dc8b0, tasks 0c07aed6, lockdown-spec 5049038b, marketplace-spec 107f0244. They equal `BASELINE_TARGET_REVISION`. Evidence was read only, and no upstream code was run.

### Coverage

**Full inventory: the update entry points** (the spec claims every one is covered, so this boundary can be listed completely). Source is upstream `hooks/register.tsx` at `bfb85cd`. No other file in `hooks/` contains `process.run`, `http.fetch` or `command.run`.

| Entry point | Reaches | Covered by D2 guard |
|---|---|---|
| 6-hour timer `startTimers` (:2706) | `checkUpdate` | `checkUpdate` |
| `afterLoad` on session start or first draw (:2759) | `checkUpdate` | `checkUpdate` |
| Settings "Check for updates", `checkUpdateNow` (:2789, button :4717) | `checkUpdate` | `checkUpdate` + locked reply |
| `/effortless update` (:3925 / :3928) | `checkUpdate`, then `runUpdate` | both, plus locked reply |
| Update card buttons (:2935, :2947), settings panel (:4713) | `runUpdate` | `runUpdate` |
| `latestAvailable` (:2682) → `latestFromGit` (:2668), `RELEASES_URL` fetch (:2685), background `marketplace update effortless` (:2694) | only called from `checkUpdate` (:2767) | `checkUpdate` |
| `syncRunningCopy` (:2883, `cp -R`/robocopy) | only called from `runUpdate` (:2859) | `runUpdate` |
| `runClaude` other use: uninstall (:2529) | not an update path | not guarded (consistent with design Non-Goals) |

Result: the two guards cover every entry point. I found no update path outside them.

**Other checks:**
- Marketplace pin format: `claude-plugins-official` has 164 entries of the form `{"source":"url","sha":…}` with no `ref`, so the format in D5 works.
- Install layout: `installed_plugins.json` entries record `installPath = cache/<marketplace>/<plugin>/<version>` plus `gitCommitSha` and `version`.
- Upstream heads at review time: `main` is still `bfb85cd` and `stable` is `d4cd7d6`.
- The untracked `.claude/` and `.pi/` folders hold only generated OpenSpec commands and skills; I saw no secrets.

**Not covered:**
- The `claude-code/testing` harness API. Its type definitions (`.claude-plugin/types/`) are generated and absent from the clone, and the binary was not inspected.
- How `claude plugin update` behaves when the `sha` changes but the version stays the same.
- The full list of component files Claude Code 2.1.301 auto-loads. I used the known plugin layout, not official docs.
- Jev and textless, both excluded by the baseline.

### Findings

#### F1 — High — family `lockdown-test-feasibility` (root: the integration test depends on an unconfirmed harness ability) — in scope
- **Evidence:**
  - Spec `effortless-fork-lockdown`, "Lockdown is guarded by a test", says the test SHALL fail when an update-related request or process starts.
  - Task 2.3 includes a check that the test fails once the `checkUpdate` guard is removed.
  - Design D3's fallback reads "calls the guarded entry points with a locked identity instead".
  - The identity is computed inside the plugin from the synchronous `$.plugin.root` (`hooks/register.tsx:3146`), and no update function takes an identity argument.
  - The upstream tests mock only engine calls (`fs.read`, `http.fetch`, `session.root`, `env.get`; `tests/effortless.test.ts:3356-3372`, `:3886`). Nothing shows `$.plugin.root` can be overridden.
  - So if task 2.1's answer is no, the stated fallback can't be built. Only the unit test of `updatesLocked` would remain, which neither meets the requirement nor fails in 2.3's guard-removal check.
- **SUGGESTED_CORRECTION:**
  - Make task 2.1 a decision gate and name a concrete fallback now. For example: a second `claude plugin test` run on a copy placed under `…/cache/zenstudiolab/effortless/<version>/` that holds only the plugin plus `tests/lockdown.test.ts`. Upstream tests keep running under the default `effortless` identity.
  - If neither the override nor the folder-placed copy produces `effortless@zenstudiolab`, the worker stops and the Orchestrator decides.
  - If a runner script is added, add it to the "Diff against upstream stays small" scenario's list of allowed files.

#### F2 — High — family `gate-path-classification` (root: D4's installed / not-installed split doesn't match what Claude Code loads and runs) — in scope
- **Evidence:**
  - D4 step 3 and the spec scenario "Files that are not installed" exclude `tools/`, `bench/` and others. The always-flagged manifest set is only `.claude-plugin/`, `hooks/hooks.json` and `commands/`.
  - The marketplace installs the whole repo root, so every file lands in `cache/zenstudiolab/effortless/<version>/`.
  - Claude Code auto-loads other plugin component files too (`.mcp.json`, `agents/`, `skills/`, `output-styles/`, `settings.json`, `.lsp.json`, `bin/`). A new `.mcp.json` server with a `command` field matches none of D4's patterns, so the gate would exit 0 although code runs on load.
  - `bench/` is labelled not installed, but `/effortless bench` reads `${$.plugin.root}/bench/judge-cases.json` (`hooks/register.tsx:4019`).
  - `tests/` is in pattern scope, and `tests/effortless.test.ts` contains many `http.fetch`/`process.run` mocks. Nearly every upstream test change would be flagged, and `tools/test.sh` runs those tests on the host after the merge.
- **SUGGESTED_CORRECTION:**
  - Always flag any newly added path outside the excluded docs and site folders.
  - Always flag any change to the component files listed above.
  - Move `bench/` out of the "not installed" list, or relabel the category "not loaded or executed by the plugin" and keep only folders that really aren't.
  - List `tests/` changes in their own "runs under `tools/test.sh`" section to review before running the tests, instead of pattern-flagging them.
  - Keep the pattern list for hook code as it is. This adds no new tooling.

#### F3 — Medium — family `spec-design-scope-mismatch` — in scope
- **Evidence:**
  - Spec `effortless-fork-lockdown`, "No automatic update checks" and "No self-update or self-overwrite", says "The forked plugin SHALL NOT …" with no condition.
  - Design D2 deliberately keeps updates on under the `effortless`/`effortless-dev` identity. That is how upstream tests stay green, and design Risks bullet 2 accepts it for development loads with `--plugin-dir`.
- **SUGGESTED_CORRECTION:** Limit both requirements to installs whose marketplace is anything other than `effortless`/`effortless-dev` (in practice, `zenstudiolab`). Also add one scenario saying a development load is not a trusted install, as the ADR in task 1.1 already states.

#### F4 — Medium — family `gate-verification-criteria` — in scope
- **Evidence:**
  - Task 2.4 verifies the gate "against a throwaway local ref", but D4 and the spec fix the comparison to `HEAD...upstream/main`, with no ref parameter.
  - Task 2.4 also expects "0 incoming commits". That holds only while upstream `main` stays at `bfb85cd`. It is true right now, but upstream moves daily.
- **SUGGESTED_CORRECTION:**
  - Give the gate one optional ref argument, defaulting to `upstream/main`.
  - Check the gate's incoming-commit count against `git rev-list --count HEAD..upstream/main` instead of expecting 0.
  - Check its exit code against whether anything was flagged.

#### F5 — Medium — family `pin-bump-version-keying` — in scope
- **Evidence:**
  - `installed_plugins.json` keys installs by version (`cache/<marketplace>/<plugin>/<version>`) and records both `version` and `gitCommitSha`.
  - The fork keeps upstream's `version` 1.71.0, because the spec's small-diff scenario forbids touching `plugin.json`.
  - It is unverified whether `claude plugin update` installs a new `sha` when the version is unchanged, as with a lockdown-only fix.
  - Tasks 3.2 and 3.3 say "at the pinned `sha`" without naming the field that records it.
- **SUGGESTED_CORRECTION:**
  - In tasks 3.2 and 3.3, verify against `gitCommitSha`.
  - In the Workflow follow-up, note that if a pin bump with the same version leaves `gitCommitSha` unchanged, the fix is to uninstall and reinstall `effortless@zenstudiolab`.
  - No new tooling is needed.

#### F6 — Low — family `design-claim-accuracy` — in scope
- **Evidence:**
  - D4 says "the lockdown test catches a gate that has been tampered with". The D3 test only exercises `updatesLocked` and the update entry points, never `tools/upstream-gate.sh`.
  - D2 says "about 50 upstream update tests"; the 52 is a count of matching grep lines, not of tests.
- **SUGGESTED_CORRECTION:**
  - Replace the D4 claim with the protection that actually exists: an upstream change to `tools/upstream-gate.sh` shows up as a merge conflict or a listed diff.
  - Write "the upstream `describe('updates')` tests" instead of a number.

#### F7 — Low — family `approval-and-publish-scope` — in scope
- **Evidence:**
  - Design D1 and task 1.3 may need a force-reset of the fork's `main`, but approval 1.2(b) covers only "pushing lockdown commits".
  - Task 4.1 "make the first commit" doesn't say which untracked paths (`.claude/`, `.pi/`, `openspec/`, `docs/plans/reviews/`) go into a public repo. I found no secrets in them.
- **SUGGESTED_CORRECTION:**
  - Add "force-reset of the new fork's `main` to `bfb85cd` if upstream has moved" to approval 1.2(b).
  - Have task 4.1 list exactly which paths go into the first commit.

#### F8 — Low — family `fork-marketplace-trap` — in scope
- **Evidence:** The fork keeps upstream's root `.claude-plugin/marketplace.json` unchanged, and its entries point at `HeyCubit/effortless` `stable`. Running `claude plugin marketplace add ZenStudioLab/effortless` would install unreviewed upstream code. The spec's small-diff rule correctly keeps that file unedited.
- **SUGGESTED_CORRECTION:** Have ADR-0001 (task 1.1) state that the fork repo must never be added as a marketplace; only `zenstudiolab` is.

PROPOSED_REVIEW_ACTION=PATCH

---

## Caller Assessment — Round 1

### Review Mode
- `change_bundle`

### Convergence Mode
- `FULL`

### Review Subject
- adopt-effortless-fork

### Per-Issue Evaluation

#### Issue F1 — Lockdown integration test may not be buildable
- **Oracle Severity:** High
- **Caller Assessment:**
  - **Family:** `lockdown-test-feasibility`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** spec `effortless-fork-lockdown` requirement "Lockdown is guarded by a test", together with user decision Q1 ("check for security before each merge/sync"). Without a buildable test, nothing enforces that requirement.
  - **Decision:** `ACCEPT_BLOCKING`
  - **CORRECTION:**
    - D3 is rewritten. The integration test runs only under the locked identity and reports itself skipped otherwise, so upstream's `tools/test.sh` stays green.
    - The locked identity comes from path 1 (harness override of `$.plugin.root`) or path 2 (fork-only `tools/lockdown-test.sh`, a folder-placed copy under `mktemp -d/cache/zenstudiolab/effortless/<version>/`). If neither works, the worker stops and the Orchestrator decides. A locked run that skips the integration test is a failure.
    - Task 2.1 is now a decision gate. Task 2.3 verifies the skipped and executed runs and both guard-removal failures.
    - The spec's test requirement and the small-diff scenario now name the runner. The proposal's Impact lists it.

#### Issue F2 — Gate path classification misses component files, bench and host-run paths
- **Oracle Severity:** High
- **Caller Assessment:**
  - **Family:** `gate-path-classification`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** spec requirement "Upstream changes pass a security gate" and user decision Q1. Concrete security risk: an added `.mcp.json` server would execute on load while the gate exits 0, because the whole repository root is installed.
  - **Decision:** `ACCEPT_BLOCKING`
  - **CORRECTION:**
    - D4 now uses three groups: (a) not loaded or run (`textless/`, `site/`, `docs/`, `report-worker/`, `.github/`); (b) run on the host during verification (`tests/`, `tools/`), listed and making the gate exit non-zero; (c) installed, everything else, `bench/` included.
    - In group (c), new paths and component files (`.claude-plugin/`, `hooks/hooks.json`, `commands/`, `.mcp.json`, `agents/`, `skills/`, `output-styles/`, `settings.json`, `.lsp.json`, `bin/`) always flag, and the pattern flags apply to added lines.
    - The spec gate requirement gains scenarios for new or component paths and for host-run paths. The not-loaded scenario no longer lists `bench/`.

#### Issue F3 — Spec says "never" while D2 keeps upstream identities unlocked
- **Oracle Severity:** Medium
- **Caller Assessment:**
  - **Family:** `spec-design-scope-mismatch`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** no independent blocking source. The F1 correction, however, depends on the locked versus unlocked identity being stated in the spec, because the test runs only under the locked identity.
  - **Decision:** `ACCEPT_NONBLOCKING`, applied as part of the F1 correction.
  - **CORRECTION:** both no-update requirements are scoped to installs from any marketplace other than `effortless`/`effortless-dev`. Added the scenario "A development load is not a trusted install".

#### Issue F4 — Task 2.4 gate verification is not runnable as written
- **Oracle Severity:** Medium
- **Caller Assessment:**
  - **Family:** `gate-verification-criteria`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** no independent blocking source. The F2 correction needs it, because the new groups can only be verified against throwaway refs.
  - **Decision:** `ACCEPT_NONBLOCKING`, applied as part of the F2 correction.
  - **CORRECTION:**
    - The gate takes an optional ref argument (default `upstream/main`).
    - Task 2.4 checks the incoming-commit count against `git rev-list --count HEAD..upstream/main`, and the exit code against the flagged state.
    - Five throwaway-ref cases cover hooks, `.mcp.json`, `bench/`, `tests/` and `textless/`.

#### Issue F5 — Pin bumps with an unchanged version may not reinstall
- **Oracle Severity:** Medium
- **Caller Assessment:**
  - **Family:** `pin-bump-version-keying`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level. The first install in wave 3 is unaffected, and only later pin bumps could hit this.
  - **Decision:** `ACCEPT_NONBLOCKING`
  - **CORRECTION:** not edited this round, because only accepted blocking findings authorize bundle edits. Carried to `/opsx-apply-loop`:
    - In tasks 3.2, 3.3 and 4.2, verify the pin via `gitCommitSha` in `installed_plugins.json`.
    - If a pin bump with an unchanged version leaves `gitCommitSha` unchanged, uninstall and reinstall `effortless@zenstudiolab`.

#### Issue F6 — Inaccurate design claims
- **Oracle Severity:** Low
- **Caller Assessment:**
  - **Family:** `design-claim-accuracy`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level.
  - **Decision:** `ACCEPT_NONBLOCKING`
  - **CORRECTION:**
    - The D4 "lockdown test catches a tampered gate" claim was replaced. It sat in the paragraph rewritten for F2, which now says an upstream change to the gate shows up as a merge conflict or in group (b).
    - The D2 "about 50 upstream update tests" wording is left for apply-time cleanup.

#### Issue F7 — Approval scope is missing the force-reset, and the first public commit is unscoped
- **Oracle Severity:** Low
- **Caller Assessment:**
  - **Family:** `approval-and-publish-scope`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level. The global kernel already requires confirming hard-to-reverse and outward-facing actions, so apply must ask regardless.
  - **Decision:** `ACCEPT_NONBLOCKING`
  - **CORRECTION:** carried to `/opsx-apply-loop`:
    - The task 1.2 approval prompt must include "force-reset the new fork's `main` to `bfb85cd` if upstream moved".
    - Task 4.1 must list the exact paths in the first public commit before asking.

#### Issue F8 — The fork's own upstream marketplace file is a trap
- **Oracle Severity:** Low
- **Caller Assessment:**
  - **Family:** `fork-marketplace-trap`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level.
  - **Decision:** `ACCEPT_NONBLOCKING`
  - **CORRECTION:** carried to `/opsx-apply-loop`. ADR-0001 (task 1.1) must state that the fork repository is never added as a marketplace; only `zenstudiolab` is.

### Summary
- **Issues accepted blocking:** 2 (F1, F2)
- **Issues accepted nonblocking:** 6 (F3 and F4 applied with their parent corrections, F6 partly applied, F5, F7 and F8 carried to apply)
- **Issues deferred out of scope:** 0
- **Issues rejected:** 0

### Caller Result
PATCH_BASELINE_REVISION=sha256 proposal=f2402711 design=8e2b641d tasks=002ee7f5 lockdown-spec=18ba09e7 marketplace-spec=107f0244
REVIEW_ACTION=PATCH
CONSENSUS_STATUS=NEEDS_REVISION

### Notes for Oracle (next round)
- Fixed: F1 (D3, tasks 2.1, 2.3 and 2.5, the spec test requirement, the small-diff scenario, proposal Impact) and F2 (D4, task 2.4, the spec gate requirement and its scenarios). F3 and F4 were applied together with them.
- Verify that the D3 path-2 identity claim holds against `pluginId`'s regex (`hooks/register.tsx:3146`) for a `mktemp -d` root.
- Run a bounded whole-bundle regression sweep. The boundary fields are unchanged; only the candidate revision changed.

---

<!-- Oracle round 2 section transcribed by the caller from oracle run be7a2091 output (oracle had no file-write tool). -->

## Round 2 - Oracle Review

Mode: `PATCH`. Reviewer: `oracle`. Target: change bundle `adopt-effortless-fork`. I recomputed the sha256 prefixes and they match `PATCH_BASELINE_REVISION`: proposal f2402711, design 8e2b641d, tasks 002ee7f5, lockdown-spec 18ba09e7, marketplace-spec 107f0244. All evidence was read only, and I executed no upstream code.

The accepted Round 1 fixes hold, but the sweep found two new High issues and two new Medium issues. All four are fixable in the artifacts, so I propose `PATCH`.

### Verification of accepted corrections

**F1 (`lockdown-test-feasibility`): resolved in substance.**
- D3 now names two concrete paths and a stop-and-escalate rule. Task 2.1 is a real decision gate. Task 2.3 checks both the skipped and the executed runs and runs two guard-removal checks. The spec and proposal name `tools/lockdown-test.sh`.
- I checked the path-2 identity claim against `hooks/register.tsx:3146`, which computes the install identity as `` `effortless@${$.plugin.root.match(/cache[\\/]([^\\/]+)[\\/]/)?.[1] ?? 'effortless'}` ``.
  - For a root such as `/tmp/tmp.XXXX/cache/zenstudiolab/effortless/1.71.0/`, the first match captures `zenstudiolab`, so the claim is correct.
  - Two conditions the design doesn't state:
    - The regex is unanchored. It matches any substring ending in `cache/`, not only a whole folder named `cache`. A temp directory under `~/.cache/…` would resolve to a different name.
    - It only matches if `$.plugin.root` is an absolute path. Run as `claude plugin test .`, the root may be relative.
  - Task 2.3's "integration test actually executed" check would catch either failure, so this goes into F11 rather than counting as a recurrence.
- The remaining gaps sit in how the corrected design is wired into later syncs (F9) and how the test learns its identity (F11). Neither is the Round 1 root cause (no buildable path).

**F2 (`gate-path-classification`): resolved.**
- Component files, `bench/` and `tests/`/`tools/` are now classified correctly. New installed paths always flag.
- Upstream has no `.github/` folder, so listing it in group (a) is harmless.
- One remaining hole is in the line-pattern list, not the path groups. It is a different root cause, reported as F10.

**F3: resolved.**
- Both no-update requirements are now limited to installs under a marketplace other than `effortless`/`effortless-dev`. The scenario "A development load is not a trusted install" was added.
- Nit only, not a finding: the spec's Purpose line and the proposal's "Turn off the plugin's built-in updater" are still unconditional.

**F4: resolved.**
- The gate now takes an optional ref argument. The incoming-commit count is checked against `git rev-list --count`, and five throwaway-ref cases are listed.

**F6:**
- The D4 tamper claim is now accurate: an upstream file at `tools/upstream-gate.sh` would show up as an add/add merge conflict or in the group (b) listing.
- The D2 "about 50" wording is unchanged, as agreed (carried to apply).

**F5, F7, F8:** carried to apply. The edits did not change their evidence.

### Regression sweep coverage

- **Whole bundle (5 files).** I re-checked consistency between:
  - proposal Impact ↔ spec small-diff scenario ↔ task 2.5 file list;
  - D3 ↔ task 2.1/2.3 ↔ spec "guarded by a test";
  - D4 ↔ task 2.4 ↔ spec gate requirements;
  - Migration Plan and Workflow follow-up ↔ D3;
  - marketplace spec "pin gated" ↔ D4 exit semantics.
- **Upstream checks:**
  - `pluginId` is not exported (`:3146` is a `const`, no `export`).
  - Hook modules import from `../types` and `./…`, so imports already cross folder boundaries.
  - `runClaude(` and `$.command.run` appear 4 times combined in `hooks/register.tsx`.
- **Not covered:** the `claude-code/testing` harness API (type definitions are absent from the clone), and whether `$.plugin.root` is absolute under `claude plugin test`.

### Findings

#### F9 — High — family `sync-workflow-lockdown-run` (root: the locked integration test isn't part of the sync procedure) — in scope

**Evidence:**
- Under D3 path 2, `tools/test.sh` runs with identity `effortless`, so the lockdown integration test reports itself skipped there. Only `tools/lockdown-test.sh` runs it under the locked identity.
- `tasks.md` "Workflow follow-up" says only "Merge `upstream/main`, run `tools/test.sh`, and push the fork."
- `design.md` Migration Plan step 4 still says "Run `tools/test.sh` (upstream tests plus the lockdown test)", which was true before the D3 rewrite and is false under path 2.
- Result: on every upstream sync after the first, the spec scenario "Merged upstream code re-enables updates → the lockdown test fails" never runs. That holds the marketplace spec's pin gate cannot be met either: its requirement asks that the lockdown test passed, and there's no run that could pass it.
- Task 2.3 verifies the locked run once, during the initial wave only.

**SUGGESTED_CORRECTION:**
- In the Workflow follow-up and Migration step 4, say: run `tools/test.sh`, and also run `tools/lockdown-test.sh` when 2.1 chose path 2. In both cases, confirm the locked integration test executed and was not skipped.
- If 2.1 chose path 1, a single `tools/test.sh` run is enough. Record which path applies in the ADR or the fork README note so later syncs know which command to run.

#### F10 — High — family `gate-pattern-coverage` (root: the group (c) line patterns miss ways new code can be loaded or run through code that already exists) — in scope

**Evidence:**
- D4 step 4 patterns cover URLs, `fetch(`, `$.http`, `$.process`, `child_process`, `spawn`, `exec`, `$.fs.write`/`remove`, `$.env`, credentials words, and eval/encoding.
- Not covered:
  - **New imports.** An added `import … from '../docs/x'` (or `../site/…`, `../textless/…`) line in `hooks/register.tsx` matches no pattern. The imported file sits in group (a), "listed, not flagged". The gate can exit 0 while installed code now loads an unflagged file. Hooks already import across folders (`hooks/register.tsx:4` from `'../types'`).
  - **Wrapper calls.** New calls through existing wrappers contain no flagged token, for example `runClaude($, ['plugin', 'install', …])` (wrapper at `:2826`, holding the only `$.process.run`) or a `git(…)` helper (`:2670`). Today's background `marketplace update effortless` at `:2694` is exactly such a line.
  - **`$.command.run`.** It runs slash commands and skills (`:1107`, `:1518`, `:1692`, `:1752`, `:2538`).
- The lockdown test only catches these when they fire on session start, the timer, `/effortless update` or the check button. A call triggered from another event or command passes both the test and a clean gate exit.
- This is a security gap against user decision Q1. Round 1 explicitly endorsed the pattern list; this is my miss, not a recurrence of F2's path-classification root.

**SUGGESTED_CORRECTION:**
- In D4 step 4, the spec gate requirement and task 2.4, add to the group (c) added-line patterns: `import `, `require(`, `runClaude(`, `$.command`, and any reference to a group (a) folder name.
- Add one throwaway-ref case to task 2.4: an added `runClaude(` line in `hooks/`, expected flagged with a non-zero exit.
- This is a pattern-list edit only; no new tooling.

#### F11 — Medium — family `lockdown-test-identity-source` (root: the test's skip/run decision has no defined source of the identity) — in scope

**Evidence:**
- D3: the integration test "runs only when the plugin's identity is locked and reports itself skipped otherwise". But `pluginId` is a module-private `const` (`:3146`). D2 exports only `updatesLocked(pluginId)`, which takes the identity as input.
- Task 2.2 limits the `register.tsx` diff to "those functions and the new export", which rules out exporting `pluginId` as well.
- The test then has no stated way to compute the identity it gates on.
- D3 path 2 also says "no earlier `cache/` segment", but the regex matches the substring `cache/` anywhere. It also depends on `$.plugin.root` being an absolute path.

**SUGGESTED_CORRECTION:**
- Let D2 export the identity source: either export `pluginId`, or make the exported function `updatesLocked(root: string)` and have `pluginId` use it. Allow that in task 2.2's diff check.
- The test then reads `$.plugin.root` through the same function.
- In D3 path 2, require `mktemp -d -p /tmp` (no `cache/` substring before the cache segment) and `claude plugin test "<absolute dir>"`.

#### F12 — Medium — family `gate-pass-definition` (root: "gate passed" isn't defined now that reviewed flags are the normal case) — in scope

**Evidence:**
- Marketplace spec "Pin changes are deliberate and gated" requires a fork commit "whose upstream sync gate and lockdown test both passed". Its scenarios say "gate report was reviewed clean".
- The lockdown spec scenario "Flagged report blocks the merge" says the merge proceeds after review of each flagged line, while the gate still exits non-zero.
- After F2, any new installed path or component change flags, and upstream routinely adds files.
- So most syncs end with exit status ≠ 0 and are reviewed and accepted. Read literally ("passed" as exit 0), the pin could never move on those syncs.
- The marketplace spec file is unchanged since Round 1 (107f0244); this tension became material with the F2 correction.

**SUGGESTED_CORRECTION:** In the marketplace spec requirement text, define a passed gate as either an exit-0 report, or a report whose flagged lines and group (b) listing were all reviewed and accepted by the operator. Keep the existing scenarios.

#### F13 — Low — family `lockdown-runner-copy-scope` — in scope

**Evidence:**
- D3 path 2: the runner "copies the plugin files plus `tests/lockdown.test.ts`".
- `claude plugin test` runs every `*.test.ts` under the folder (`tools/test.sh` header comment). If the copy includes `tests/effortless.test.ts`, upstream's `describe('updates')` tests run under the `zenstudiolab` identity. The lockdown would make them fail, by design.

**SUGGESTED_CORRECTION:** State that the path-2 copy excludes `tests/effortless.test.ts` and `textless/`, and contains `tests/lockdown.test.ts` as its only test file.

### Carried, unchanged (not re-raised)

F5, F7, F8 and the D2 wording part of F6 remain accepted non-blocking and are carried to apply. Their evidence is unchanged.

### Proposal rationale

F9 and F10 are new High findings. F9 is a regression introduced by the F1 correction's wiring. F10 is a new pattern-coverage family I missed in Round 1. Neither repeats the Round 1 roots (no buildable test path; folder-based classification), so a rebuild isn't indicated.

All four non-Low findings can be fixed with wording and pattern-list edits to D3/D4, task 2.2/2.4, the Workflow follow-up and the two specs, without changing the baseline or the structure.

PROPOSED_REVIEW_ACTION=PATCH
---

## Caller Assessment — Round 2

### Review Mode
- `change_bundle`

### Convergence Mode
- `PATCH`

### Review Subject
- adopt-effortless-fork

### Verification of Round 1 corrections
- Oracle confirms F1, F2, F3 and F4 are resolved, and the F6 tamper claim is now accurate.
- The F5, F7 and F8 carries are unchanged.

### Per-Issue Evaluation

#### Issue F9 — Locked lockdown test is not part of the sync procedure
- **Oracle Severity:** High
- **Caller Assessment:**
  - **Family:** `sync-workflow-lockdown-run`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** spec `effortless-fork-lockdown`, scenario "Merged upstream code re-enables updates" (the lockdown test fails, so the merge is not pinned), together with spec `vetted-plugin-marketplace`, "Pin changes are deliberate and gated". Under D3 path 2 neither can be met on later syncs.
  - **Decision:** `ACCEPT_BLOCKING`
  - **CORRECTION:** Migration step 4 and the Workflow follow-up now run `tools/test.sh` plus `tools/lockdown-test.sh` if it exists, and confirm the locked integration test was executed, not skipped. Because the condition is whether the runner file exists, nothing extra needs to record which path was chosen.

#### Issue F10 — Gate line patterns miss imports, helper calls and command runs
- **Oracle Severity:** High
- **Caller Assessment:**
  - **Family:** `gate-pattern-coverage`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** spec requirement "Upstream changes pass a security gate" and user decision Q1. Concrete security risk: a new `runClaude(` or `$.command.run` call, or an import of a group (a) file, passes with exit 0.
  - **Decision:** `ACCEPT_BLOCKING`
  - **CORRECTION:**
    - D4 group (c) patterns now also cover `import `, `require(`, `runClaude(`, `$.command`, and references to group (a) folder names.
    - The spec gate requirement text and the "Flagged report" scenario now name commands and imports.
    - Task 2.4 adds a throwaway-ref case for `runClaude(` and a `../docs/` import.

#### Issue F11 — No defined source of the identity for the test's skip/run decision
- **Oracle Severity:** Medium
- **Caller Assessment:**
  - **Family:** `lockdown-test-identity-source`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level. Applied because the F9 correction depends on the test's run/skip decision being implementable.
  - **Decision:** `ACCEPT_NONBLOCKING`, applied as part of the F9 correction.
  - **CORRECTION:**
    - D2's export is now `updatesLocked(root)`, which reuses `pluginId`'s regex; the test calls it with the harness's `$.plugin.root`.
    - The unit-test inputs are now roots.
    - Task 2.2's diff check allows `pluginId` if the regex is shared.
    - D3 path 2 now requires `mktemp -d -p /tmp` and an absolute path for `claude plugin test`.

#### Issue F12 — "Gate passed" is undefined when reviewed flags are the normal case
- **Oracle Severity:** Medium
- **Caller Assessment:**
  - **Family:** `gate-pass-definition`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level. Applied together with F9, because both touch the same pin requirement sentence ("whose gate passed and whose locked lockdown test ran").
  - **Decision:** `ACCEPT_NONBLOCKING`, applied as part of the F9 correction.
  - **CORRECTION:** the marketplace pin requirement now defines a passed gate as exit 0, or as every flagged line and host-run listing reviewed and accepted by the operator. It also requires the locked lockdown test to have actually run.

#### Issue F13 — The path-2 copy could include upstream's update tests
- **Oracle Severity:** Low
- **Caller Assessment:**
  - **Family:** `lockdown-runner-copy-scope`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level. Applied with F11, because both rewrite the same D3 path-2 sentence.
  - **Decision:** `ACCEPT_NONBLOCKING`, applied as part of the F11 rewrite.
  - **CORRECTION:** the D3 path-2 copy now excludes `textless/` and `tests/effortless.test.ts`, so `tests/lockdown.test.ts` is its only test file.

#### Nit — the Purpose line and the proposal wording are unconditional
- Not a finding and not edited. The requirement bodies carry the precise scope.

### Summary
- **Issues accepted blocking:** 2 (F9, F10)
- **Issues accepted nonblocking:** 3 (F11, F12, F13, each applied with a parent correction), plus F5, F7, F8 and the D2 wording part of F6 still carried to apply
- **Issues deferred out of scope:** 0
- **Issues rejected:** 0

### Caller Result
PATCH_BASELINE_REVISION=sha256 proposal=f2402711 design=27040ea4 tasks=7cd15873 lockdown-spec=0af425f8 marketplace-spec=8dc018ca
REVIEW_ACTION=PATCH
CONSENSUS_STATUS=NEEDS_REVISION

### Notes for Oracle (next round)
- Fixed: F9 (Migration step 4, Workflow follow-up) and F10 (D4 patterns, spec gate text and scenario, task 2.4 case). F11, F12 and F13 were applied with them.
- Round 3 of at most 4. Run a bounded whole-bundle regression sweep. The boundary fields are unchanged.
- No structural family has recurred. F9 and F10 are new roots, as you noted.

---

<!-- Oracle round 3 section transcribed by the caller from oracle run f4eb3f4e output (oracle had no file-write tool). -->

## Round 3 - Oracle Review

Mode: `PATCH`. Reviewer: `oracle`. Target: the change bundle `adopt-effortless-fork`. The recomputed sha256 prefixes match `PATCH_BASELINE_REVISION`: proposal f2402711, design 27040ea4, tasks 7cd15873, lockdown-spec 0af425f8, marketplace-spec 8dc018ca. `openspec validate --strict` passes. All evidence was read only, and no upstream code was executed.

### Verification of accepted corrections

- **F9 (`sync-workflow-lockdown-run`): resolved.** Three places now run the lockdown test and require proof that the locked integration test executed rather than being skipped:
  - design Migration step 4;
  - the tasks Workflow follow-up;
  - the marketplace spec requirement "Pin changes are deliberate and gated", which reads "whose locked lockdown test ran (not skipped) and passed".
  
  All three key on whether `tools/lockdown-test.sh` exists, so no path record is needed. Every sync and every pin bump is now tied to the locked run.
- **F10 (`gate-pattern-coverage`): resolved for the reported root.** The new patterns are listed in three places: D4 step 4, the spec gate requirement and its "Flagged report" scenario. Task 2.4 has a throwaway-ref check for `runClaude(` and for a `../docs/` import. One residual variant is folded into F10 below. It is not a structural recurrence.
- **F11: resolved.**
  - D2 now exports `updatesLocked(root)` and reuses the regex in `pluginId`, with the same fallback when the root has no `cache/` segment. The test and the update functions share this one source of identity.
  - The unit-test inputs are now plugin roots. Task 2.2's diff allowance covers `pluginId` when the regex is shared.
  - D3 path 2 specifies `mktemp -d -p /tmp` and an absolute path. I re-checked this against `hooks/register.tsx:3146`, where the regex is `/cache[\\/]([^\\/]+)[\\/]/` with no anchor. A `/tmp/tmp.XXXXXXXXXX/cache/zenstudiolab/effortless/<v>/` root resolves to `zenstudiolab`. This holds as long as the root `claude plugin test` reports is the directory passed to it. That is unverified, and the check in task 2.3 that the integration test executed covers it.
- **F12: resolved in the requirement text.** "Passes" is now defined as either exit 0 or a fully reviewed and accepted report. A small wording leftover is noted below.
- **F13: resolved.** The path-2 copy excludes `textless/` and `tests/effortless.test.ts`.
- **Carried items** (F5, F7, F8, and the D2 wording in F6): not re-raised. F7's evidence is updated below.

### Regression sweep coverage

I checked consistency across the whole bundle:
- D2/D3 against tasks 2.1–2.3 and the spec requirement "Lockdown is guarded by a test";
- D4 against task 2.4 and the spec gate requirement and its scenarios;
- proposal Impact against the "Diff against upstream stays small" scenario and the task 2.5 file list;
- Migration and Workflow follow-up against D3;
- the marketplace pin requirement against the lockdown "Flagged report" scenario.

On the upstream side, I re-traced every route into `runUpdate` (`:2935`, `:2947`, `:4713`, `:3928`) and found one the tasks don't exercise: `/effortless card offer` (`:3908-3921`). It builds a live offer card whose Update button calls `runUpdate($, card)` (`:2935`), with no update check involved.

**Not covered:** the `claude-code/testing` harness API is still unread, and whether `claude plugin test` reports the passed directory as `$.plugin.root` is still unconfirmed. Both are left to the task 2.1 decision gate.

### Findings

#### F14 — Medium — family `runupdate-guard-test-reachability` — in scope
**Evidence:**
- The third verification bullet of task 2.3 says removing the `runUpdate` guard "makes the locked run fail".
- The integration test steps listed in task 2.3 and in D3 never reach `runUpdate` while the `checkUpdate` guard is in place:
  - `session.start`, the timer and "Check for updates" all stop at the locked `checkUpdate`, so no offer card is created;
  - `/effortless update` returns the locked reply (D2 point 3) before `runUpdate` (`:3928`).
- With only the `runUpdate` guard removed, the test as described would still pass, so that check cannot be met.
- A real, reachable route exists: `/effortless card offer` (`:3908-3921`) creates an offer card even when locked, and its Update button calls `runUpdate` (`:2935`). That route is exactly what the `runUpdate` guard protects.

**SUGGESTED_CORRECTION:** In task 2.3 and D3, add an integration step: run `/effortless card offer`, press the card's Update button, then assert that no `claude` `process.run`, no file copy and no reload happens.

#### F15 — Medium — family `lockdown-test-path-conditional-criteria` — in scope
**Evidence:** The first two verification bullets of task 2.3 assume D3 path 2:
- "`tools/test.sh` … the lockdown integration test reported as skipped there";
- "the locked run passes".

Under path 1 (the harness overrides `$.plugin.root`), the test sets a `cache/zenstudiolab/` root itself. The integration test therefore runs, not skips, inside `tools/test.sh`, which is also the locked run. The first bullet would fail on a correct path-1 implementation.

**SUGGESTED_CORRECTION:** Make the bullets conditional on the path:
- Path 1: `tools/test.sh` passes with upstream tests unchanged and the integration test executed.
- Path 2: as written now.

The Workflow follow-up wording already fits both paths.

#### F10 (tracked, residual variant) — Low — family `gate-pattern-coverage` — in scope
**Evidence:**
- D4 step 4 matches `import ` (with a trailing space) and `require(`. It does not match a dynamic `import(` with a computed path, and a computed path also defeats the group (a) folder-name pattern.
- No hook uses `import(` today.
- This is a variant of the same root, not a structural recurrence. The gate still lists every incoming commit, and design Risks already accepts that a pattern list cannot be complete.

**SUGGESTED_CORRECTION:** Add `import(` to the group (c) pattern list, which is a single-token edit. No other change.

#### F12 (tracked, residual wording) — Low — family `gate-pass-definition` — in scope
**Evidence:** The marketplace spec's requirement text now defines a passing gate as "exit 0, or every flagged line and host-run listing reviewed and accepted". Its two scenarios still say "the gate report was reviewed clean" and "no gate report reviewed clean", which could be read as exit 0 only.

**SUGGESTED_CORRECTION:** Change "reviewed clean" to "passed (as defined above)" in both scenarios.

### Carried, evidence updated (not re-raised)
- **F7 (`approval-and-publish-scope`).** Approval 1.2(c) names only `tools/test.sh`. Under path 2, `tools/lockdown-test.sh` also runs plugin code on the host, so it should be added to the 1.2(c) wording when F7 is applied.
- **F5, F8, and the D2 wording in F6:** unchanged.

### Proposal rationale
No Critical or High finding remains:
- F9 and F10 are fixed at their roots. F10's residual is a Low variant, not a repeated structural family, so no `REBUILD` is called for.
- F14 and F15 are Medium. They make task 2.3's acceptance criteria unachievable or path-dependent, but they don't affect the design. They can be fixed with small task wording edits, or carried to apply. The wave 2 Oracle gate (task 2.6) would catch either one during execution.

PROPOSED_REVIEW_ACTION=PASS
---

## Caller Assessment — Round 3

### Review Mode
- `change_bundle`

### Convergence Mode
- `PATCH`

### Review Subject
- adopt-effortless-fork

### Verification
- Oracle confirms F9, F10, F11, F12 and F13 are resolved at their roots.

### Per-Issue Evaluation

#### Issue F14 — The `runUpdate` guard-removal check is unreachable as written
- **Oracle Severity:** Medium
- **Caller Assessment:**
  - **Family:** `runupdate-guard-test-reachability`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level.
  - **Decision:** `ACCEPT_NONBLOCKING`
  - **CORRECTION:** carried to `/opsx-apply-loop` task 2.3. Add an integration step that runs `/effortless card offer` (`hooks/register.tsx:3908-3921`), presses the card's Update button, and asserts no `claude` `process.run`, no file copy and no reload. The `runUpdate` guard-removal check is measured against this step.

#### Issue F15 — Task 2.3 test-run criteria assume D3 path 2
- **Oracle Severity:** Medium
- **Caller Assessment:**
  - **Family:** `lockdown-test-path-conditional-criteria`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level.
  - **Decision:** `ACCEPT_NONBLOCKING`
  - **CORRECTION:** carried to task 2.3.
    - Path 1: `tools/test.sh` passes with upstream tests unchanged and the integration test executed.
    - Path 2: as written.

#### Issue F10 (residual) — dynamic `import(` is not matched
- **Oracle Severity:** Low
- **Caller Assessment:**
  - **Family:** `gate-pattern-coverage`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level. This is a variant of the same root, not a structural recurrence.
  - **Decision:** `ACCEPT_NONBLOCKING`
  - **CORRECTION:** carried to task 2.4. Add `import(` to the group (c) patterns.

#### Issue F12 (residual) — scenarios still say "reviewed clean"
- **Oracle Severity:** Low
- **Caller Assessment:**
  - **Family:** `gate-pass-definition`
  - **In-scope:** YES
  - **GOVERNING_SOURCE:** none at blocking level.
  - **Decision:** `ACCEPT_NONBLOCKING`
  - **CORRECTION:** carried to apply. The requirement text governs, so read "reviewed clean" in both pin scenarios as "passed as defined in the requirement".

### Carried to `/opsx-apply-loop` (accepted, non-blocking; not edited in the bundle)
- **F5:** verify pins via `gitCommitSha` in tasks 3.2, 3.3 and 4.2. If a bump with an unchanged version leaves `gitCommitSha` unchanged, uninstall and reinstall.
- **F6 (D2 wording):** say "the upstream `describe('updates')` tests" instead of "about 50".
- **F7:** the task 1.2 approval must include:
  - force-resetting the new fork's `main` to `bfb85cd` if upstream moved;
  - running `tools/lockdown-test.sh` (path 2) as well as `tools/test.sh`.
  
  Task 4.1 must list the exact paths in the first public commit before asking.
- **F8:** ADR-0001 must state that the fork repository is never added as a marketplace.
- **F10 residual, F12 residual, F14, F15:** as recorded above.

### Summary
- **Issues accepted blocking:** 0
- **Issues accepted nonblocking:** 4 this round (F14, F15, F10 residual, F12 residual)
- **Issues deferred out of scope:** 0
- **Issues rejected:** 0

### Terminal baseline validation
- **Scope fields:** `BASELINE_TARGET`, `BASELINE_SOURCE`, `BASELINE_INCLUSION_PREDICATE` and `BASELINE_EXCLUSIONS` are unchanged from the frozen baseline.
- **Revision:** the current sha256 prefixes (proposal f2402711, design 27040ea4, tasks 7cd15873, lockdown-spec 0af425f8, marketplace-spec 8dc018ca) equal the latest recorded `PATCH_BASELINE_REVISION`, so the result is not stale.

### Caller Result
REVIEW_ACTION=PASS
CONSENSUS_STATUS=MOSTLY_GOOD

---
