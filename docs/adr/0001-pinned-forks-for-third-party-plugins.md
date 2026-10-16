# ADR-0001: Install third-party Claude Code plugins only from pinned, locked-down forks

**Status**: Accepted
**Date**: 2026-10-16
**Change**: `openspec/changes/adopt-effortless-fork`

## Context

We want to use third-party Claude Code plugins, starting with `effortless` (upstream `HeyCubit/effortless`). The upstream publisher can't be independently verified. Its update flow also installs new code from a branch that can change at any time and copies it over the running plugin. A static review only protects us if the code that runs is exactly the code we reviewed and changes only when we decide.

## Decision

Third-party plugins are installed on this machine only through the `zenstudiolab` marketplace hosted in `claude-toolsets` (`.claude-plugin/marketplace.json`).

1. **Fork.** Each plugin comes from a fork owned by `ZenStudioLab`, never from the upstream repository.
2. **Lock down self-update.** The fork turns off the plugin's own update checks and self-install whenever it is installed under any marketplace other than upstream's. A fork-only test guards this, and it must actually run under the locked identity.
3. **Gate syncs.** Upstream changes enter the fork only after the fork's upstream gate (`tools/upstream-gate.sh`) has been run and every flagged line and host-run listing has been reviewed.
4. **Pin by full `sha`.** Each marketplace entry pins its fork by a full 40-character commit `sha`, never by a branch or tag. The pin moves only in its own `claude-toolsets` commit, to a fork commit whose gate passed and whose locked lockdown test ran and passed.

Two ways of loading a fork are not trusted installs:

- **Never add the fork repository itself as a marketplace.** Forks keep upstream's own `.claude-plugin/marketplace.json`, which points at the upstream repository. Adding it would install unreviewed upstream code.
- **A development load from outside the plugin cache** (for example `--plugin-dir plugins/effortless`) resolves to upstream's identity and is not locked. It is for development only.

## Consequences

- **Good:** what runs is always a reviewed, pinned commit. Upstream can't push code to this machine, and each upgrade is a deliberate, reviewable commit.
- **Bad:** every upstream sync costs a gate run, a review, a merge, a test run and a pin bump. The fork keeps upstream's large files unrestructured, to keep merges cheap.
- **Mitigations:** fork changes stay confined to a few fork-only files plus small guards. The gate lists every incoming commit, so a review never depends on the pattern flags alone.
