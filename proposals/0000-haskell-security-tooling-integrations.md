# Haskell security tooling integrations

* Author: Gautier DI FOLCO
* Date: 2026-06-20
* Category: RFC
* Status: Draft

## Abstract

This proposal describes how to make package vulnerability information a first-class part of the Haskell development workflow, without centralising that workflow in any single tool.
We propose to provide an out-of-the-box `cabal audit` experience built on the existing Cabal external-command/plugin system and the community `cabal-audit` tool, and to surface security advisories directly on Hackage package pages.
The advisory database remains the single source of truth; we deliberately do *not* propose extending the `.cabal` or `cabal.project` syntax to carry security metadata (see Alternatives Considered).

## Background

The Haskell Security Response Team (SRT) was established following [Tech Proposal #37](./accepted/037-advisory-db.md) to manage security risks in the ecosystem.
The team maintains the official [Haskell Security Advisory Database](https://github.com/haskell/security-advisories/) which provides a standardized source of vulnerability information.
For details on the initial constitution and reports of the SRT, refer to the [Haskell Security page](https://haskell.org/security) and the [Q2 2023 Report](https://github.com/haskell/security-advisories/blob/main/reports/2023-07-10-ann-q2-report.md).

The database itself is organized around a set of core Haskell libraries and tools developed by the SRT to automate vulnerability tracking:
* `hsec-core`: The core library defining security advisory types, parsing from Markdown, and validation.
* `hsec-tools`: A CLI utility to check, reserve, and query advisories in the database.
* `hsec-sync`: A CLI utility and library to synchronize a local snapshot of the advisory database.

### Command-Line Usage and Tutorial

To query vulnerability data locally, developers first synchronize the snapshot repository and then query the database:

```bash
# Synchronize the local advisory snapshot
$ hsec-sync sync

# Check if a package is affected by known vulnerabilities
$ hsec-tools query is-affected aeson
Affected by:
* [HSEC-2023-0001] Hash flooding vulnerability in aeson
* [HSEC-2026-0007] Denial of Service and Memory Exhaustion in aeson and text-iso8601
```

This checking process can also be done programmatically inside Haskell applications by using the corresponding libraries published on Hackage.

## Problem Statement

Currently, security auditing is fragmented and not discoverable.
Developers must learn about, install, and run third-party tools such as `cabal-audit` themselves, or configure custom CI actions, before they can verify that their build plan is free of known vulnerabilities.
There is no signal at the point of consumption: Hackage does not surface active vulnerabilities on package pages, so package users are often unaware of security issues until they go looking for them.
The goal of this proposal is to make vulnerability checking part of the default experience, so that it reaches developers who would never seek it out on their own.

Requirements a solution should be evaluated against:
* **Discoverability**: a developer should see advisory information without having to know a specific tool exists.
* **Low friction / out-of-the-box**: checks should work in a fresh `cabal-install` setup with no manual installation step.
* **Single source of truth**: the advisory database must remain the authoritative record; nothing should duplicate or compete with it.
* **No syntax lock-in**: avoid extending long-lived formats (`.cabal`) with data that does not belong to them.
* **Minimal toolchain bloat**: prefer reusing existing extension mechanisms over growing core tools.

## Prior Art and Related Efforts

Several efforts already exist in this space, and this proposal is built on top of them rather than replacing them:

* **[`cabal-audit`](https://github.com/MangoIV/cabal-audit/)** (MangoIV): audits Cabal build plans against the Haskell Security Advisory database. This is the primary tool this proposal aims to make discoverable and out-of-the-box.
* **[cabal-plan-submit](https://github.com/dancewithheart/cabal-plan-submit)**: uploads a build plan (`plan.json`) to the GitHub Dependency Submission API, making Haskell dependencies visible to GitHub's own security tooling. Complementary to the Haskell-native checks.
* **Cabal's [external-command system](https://cabal.readthedocs.io/en/stable/external-commands.html)**: Cabal already supports discovering and invoking external commands/plugins. This is the integration point this proposal relies on, rather than baking new commands into the `cabal` executable.
* **Hackage improvement efforts**: there is an ongoing tech proposal about improving the `hackage-server` codebase. Surfacing advisories on Hackage should be sequenced against that work, since the maintainability of `hackage-server` determines how feasible any UI addition is.
* **[flora.pm](https://flora.pm/)**: its maintainer (@hecate) has planned to publish advisory data on the flora platform. Hackage and flora should present consistent information drawn from the same database.

## Technical Content

We propose two integrations. Both reuse existing tooling and keep the advisory database as the single source of truth.

### 1. An out-of-the-box `cabal audit` via the external-command/plugin system

The user-facing goal is simple: typing `cabal audit` in a project checks the current build plan against the advisory database.
Rather than building this command into the `cabal` executable, we propose to deliver it through Cabal's existing **external-command / plugin system**, backed by `cabal-audit`, and to make it available with no manual install step — for example by having distributors such as `ghcup` bundle it alongside `cabal-install`.

This keeps `cabal-install` focused and avoids growing an already large codebase, while still giving developers a zero-configuration experience. It also lets the audit tooling evolve independently of Cabal releases.

Deeper, built-in integration would only be warranted if a long-term, accepted plan tied advisory data into existing cabal commands (for example, `cabal build` or `cabal outdated` warning on affected dependencies), or if the plugin model proved unable to deliver the desired experience. That is explicitly out of scope for this proposal and would be a separate decision for the Cabal maintainers.

Configuration such as ignoring specific advisory IDs would live in `cabal.project` only where it is genuinely project-local (for example, documenting an accepted, mitigated exception). Global concerns — such as which advisory sources an organisation trusts — are handled by the audit tool's own configuration, not by per-project cabal metadata.

### 2. Surface security advisories on Hackage

We propose integrating `hsec-sync` and `hsec-tools query` into the `hackage-server` infrastructure so that every package page displays a dedicated security section, and a package version affected by an active advisory shows a warning banner.

This should be sequenced against, and coordinated with, the ongoing Hackage improvement work: the feasibility and maintenance cost of a UI addition depend directly on the state of the `hackage-server` codebase. The data shown must be drawn exclusively from the advisory database (no manual per-package entry on Hackage), and should be consistent with what `flora.pm` publishes.

### Alternatives Considered

**Extending `.cabal` and `cabal.project` syntax with security metadata** (for example a `security-advisories` field, or `source-security-advisories` stanzas) was considered and is **rejected**:

* `.cabal` files exist to provide the information needed to install and run a package, and are intended to be essentially fixed at upload time; revisions exist only to update that install/run information. New versions are presumed advisory-free, so advisory data would enter only via revisions, distorting their purpose.
* It would create a competing, second, less-reliable source of truth alongside the authoritative advisory database.
* Security source selection is a global/organisational concern (for example a bank's environment), not a per-project one, so `cabal.project` is the wrong home for it.
* Syntax, once added, is extremely hard to remove.

These points were raised consistently across the discussion (see the pull request thread) by hasufell, MangoIV, and gbaz.

## Drawbacks and Risks

* **Bundling reliance**: an out-of-the-box experience depends on distributors (ghcup and others) bundling the audit plugin; users on bare or minimal installs may still need to install it manually.
* **Hackage maintenance cost**: adding a security UI to `hackage-server` adds surface area to a codebase whose maintainability is itself under discussion.
* **Coverage of the advisory database**: the value of any of this depends on the advisory database itself being kept current and complete — a problem this proposal does not solve.
* **Plugin discoverability**: relying on the external-command system means the experience is only as smooth as that system's discovery and invocation behaviour; gaps there would need follow-up work in Cabal.

## Needs Assessment and Open Questions

This proposal intentionally raises, rather than answers, several questions that should be resolved during discussion:

* **Who uses `cabal-audit` (or the advisory database) today, and who needs to?** We lack a clear picture of current adoption and demand. Quantifying this would sharpen which improvements are actually worth the effort.
* **How do significant Haskell codebases already do security audits?** For example, some teams generate a Software Bill of Materials from a Nix closure and run a generic, language-agnostic third-party scanner over it, doing nothing Haskell-specific at all. Understanding existing practice prevents reinventing it.
* **Should we instead ensure existing audit tooling works well with Haskell**, rather than building Haskell-specific integrations? (For example, better SBOM / dependency-graph export so that generic scanners cover Haskell.)
* **Coordination with the Hackage improvement proposal and `flora.pm`**: what is the realistic sequencing, and who owns the Hackage-side work?

## Stakeholders

* **Haskell Security Response Team**: maintainers of the advisory database and core querying tools.
* **Cabal maintainers**: owners of the external-command/plugin system and any bundling or distribution story.
* **`cabal-audit` maintainer (MangoIV)**: the tool this proposal aims to make default.
* **Hackage administrators / the Hackage improvement effort**: responsible for any server-side advisory display.
* **Distributors (e.g. ghcup)**: responsible for bundling the audit plugin for an out-of-the-box experience.
* **Haskell developers**: beneficiaries of integrated, discoverable security checks.

## People

_To be filled by the proposer._ Indicate who intends to do the work for each integration (for example the `cabal-audit` packaging/bundling story, and the Hackage-side display), and whether this stays within the proposer's own resources (RFC) or requires additional help.

## Time

_To be filled by the proposer._ Rough estimates per integration, noting that the Hackage-side work is gated on the Hackage improvement effort's timeline.

## Success

The project will be considered a success when:
1. `cabal audit` works out of the box in a standard `cabal-install`/`ghcup` setup, via the external-command/plugin system, with no manual installation step.
2. Package pages on Hackage display warnings for active security advisories, sourced from the advisory database.
3. The advisory database remains the single source of truth, with no competing security metadata introduced into the `.cabal` format.
