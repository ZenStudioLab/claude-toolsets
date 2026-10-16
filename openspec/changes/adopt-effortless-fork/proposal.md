# Proposal

## Why

We want the `effortless` Claude Code plugin (upstream `HeyCubit/effortless`), but its publisher can't be independently verified. The GitHub org is 10 days old and has one repo, and nearly all of its 715 commits come from a single account. The plugin also updates itself: every 6 hours it checks HeyCubit's mutable `stable` branch, and its Update button reinstalls from there and copies the new files over the running copy. A static review of `main` at `bfb85cda60055a89946f610959b479d82609c67a` found no malware. That review only means something if what runs on our machine is exactly the code we reviewed, and it changes only when we decide.

## What Changes

- Create a public fork, `ZenStudioLab/effortless`, from the reviewed upstream commit `bfb85cd`. Track upstream `main` (not `stable`) and keep our own changes small, so each upstream sync stays cheap.
- Turn off the plugin's built-in updater in the fork. That means no update check against upstream, no Update or "Check for updates" install path, and no overwriting of the running copy. Add a test so an upstream merge can't quietly bring the updater back.
- Add an upstream sync gate to the fork (`tools/upstream-gate.sh`). It diffs pending upstream commits, flags new network, process, filesystem, environment, manifest and code-evaluation changes, and must be reviewed clean before any merge.
- Add `ZenStudioLab/effortless` as a submodule at `claude-toolsets/plugins/effortless`, for editing only. It is not the install source.
- Host a Claude Code plugin marketplace named `zenstudiolab` in `claude-toolsets`. It lists only `effortless`, pinned to a full commit `sha` of the fork, and the pin moves only through a deliberate, gate-passed commit.
- Install `effortless@zenstudiolab` from that marketplace, from the local folder first and later from GitHub once `claude-toolsets` is published.
- Install only `effortless`. The upstream `textless` plugin and its community art service are not installed. Its folder stays untouched in the fork so upstream merges don't conflict.
- Keep the Jev/TypeSafe judge as upstream ships it (opt-in).
- Record the "install third-party plugins only from a pinned, gated fork" rule as ADR-0001.

## Capabilities

### New Capabilities

- `vetted-plugin-marketplace`: the `zenstudiolab` marketplace in `claude-toolsets`. It covers which third-party plugins we install, from which source, how each is pinned to a reviewed commit, and how a pin may change.
- `effortless-fork-lockdown`: how our `effortless` fork behaves. Its built-in updater is off, a test guards that, and upstream changes only enter through the reviewed sync gate.

### Modified Capabilities

None. `openspec/specs/` is empty.

## Impact

- **New repo on GitHub:** a public fork, `ZenStudioLab/effortless`. This is an outward-facing action and needs explicit user approval when the work is carried out.
- **`claude-toolsets`:** new `.gitmodules`, `plugins/effortless` (submodule), `.claude-plugin/marketplace.json`, and `docs/adr/0001-*.md`.
- **Fork:** small edits to `hooks/register.tsx` (the update functions), plus new files `tests/lockdown.test.ts`, `tools/upstream-gate.sh` and, if the test harness requires it, `tools/lockdown-test.sh`.
- **User Claude Code config (`~/.claude`):** a new marketplace entry `zenstudiolab` and the installed plugin `effortless@zenstudiolab`. Installing and running the plugin's tests both execute third-party code and need explicit user approval.
- **Data flow kept on purpose:** the Haiku judge (runs on your own Claude login) and the opt-in Jev judge, which sends prompt excerpts to `api.typesafe.ai`. In the default `auto` mode Jev turns on whenever `TYPESAFE_API_KEY` is set.
