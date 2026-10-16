# Spec Delta

## Purpose

Lets this machine install third-party Claude Code plugins only from a marketplace we control. Each plugin is pinned to an exact commit of a fork we reviewed, so nothing changes until we move the pin.

## ADDED Requirements

### Requirement: Marketplace identity
`claude-toolsets` SHALL publish one Claude Code plugin marketplace named `zenstudiolab`. That name SHALL differ from every upstream marketplace name of the plugins it lists, including `effortless`.

#### Scenario: Marketplace is recognized under its own name
- **WHEN** the operator runs `claude plugin marketplace list` after adding `claude-toolsets` as a marketplace
- **THEN** a marketplace named `zenstudiolab` is listed
- **AND** no marketplace named `effortless` or `effortless-dev` is listed as a result of this change

### Requirement: Plugins come only from our forks
Every plugin entry in the `zenstudiolab` marketplace SHALL take its source from a repository owned by `ZenStudioLab`, never from an upstream publisher's repository.

#### Scenario: effortless points at the fork
- **WHEN** the marketplace file is read
- **THEN** the `effortless` entry's source URL is `https://github.com/ZenStudioLab/effortless.git`

### Requirement: Plugins are pinned to a full commit
Every plugin entry SHALL pin its source with a full 40-character commit `sha`. The pinned commit SHALL contain the reviewed lockdown, so Claude Code installs and updates only that exact commit.

#### Scenario: Update without a pin change installs nothing new
- **WHEN** the operator runs `claude plugin update effortless@zenstudiolab` and the pinned `sha` has not changed
- **THEN** the installed plugin stays at the pinned commit
- **AND** commits pushed to the fork after that `sha` are not installed

#### Scenario: Abbreviated or moving references are rejected in review
- **WHEN** a marketplace entry names only a branch or tag, or a `sha` shorter than 40 characters
- **THEN** the entry does not meet this requirement and must not be committed

### Requirement: Pin changes are deliberate and gated
The `sha` of a plugin entry SHALL change only in its own commit to `claude-toolsets`, to a fork commit whose gate passed and whose locked lockdown test ran (not skipped) and passed. A gate passes when it exits 0, or when the operator has reviewed and accepted every flagged line and every host-run listing in its report.

#### Scenario: Pin bump after an upstream sync
- **WHEN** an upstream sync has been merged into the fork, after the gate report was reviewed clean and the lockdown test passed
- **THEN** a single `claude-toolsets` commit moves the `effortless` `sha` to the merged fork commit

#### Scenario: Pin bump without a passing gate
- **WHEN** a candidate fork commit has no gate report reviewed clean or no passing lockdown test
- **THEN** the pin is not moved to that commit

### Requirement: Only the listed plugin is installed
The `zenstudiolab` marketplace SHALL list `effortless` only. The upstream `textless` plugin SHALL NOT be listed or installed by this change.

#### Scenario: textless is unavailable
- **WHEN** the operator runs `claude plugin install textless@zenstudiolab`
- **THEN** the install fails because the marketplace does not list `textless`

### Requirement: Marketplace location
The operator SHALL first add the `zenstudiolab` marketplace from the local `claude-toolsets` folder. Once `claude-toolsets` is published to GitHub, they SHALL switch to the published repository, keeping the same marketplace name and pins.

#### Scenario: Local marketplace first
- **WHEN** `claude-toolsets` has no GitHub remote
- **THEN** the marketplace is added from the absolute local path of `claude-toolsets`
- **AND** `effortless@zenstudiolab` installs from the pinned fork commit

#### Scenario: Switch to the published repository
- **WHEN** `claude-toolsets` has been pushed to GitHub
- **THEN** the local marketplace entry is replaced with the GitHub-hosted one under the same name `zenstudiolab`
- **AND** the installed `effortless` stays at the same pinned `sha`
