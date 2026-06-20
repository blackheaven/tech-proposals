# Haskell security tooling integrations

* Author: Gautier DI FOLCO
* Date: 2026-06-20
* Status: Draft

## Abstract

This proposal describes the integration of security auditing tools into the core Haskell development workflow.
We propose to natively integrate package vulnerability checking in Cabal, expose package advisories on Hackage, and introduce metadata fields in the Cabal specification to declare, manage, and query security advisories.

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

### Existing Ecosystem Tools

Several community tools have been built around this infrastructure:
* [MangoIV/cabal-audit](https://github.com/MangoIV/cabal-audit/): A tool to audit Cabal build plans against the Haskell Security Advisory database.
* [dancewithheart/cabal-plan-submit](https://github.com/dancewithheart/cabal-plan-submit): A utility to upload Cabal build plans (`plan.json`) to the GitHub Dependency Submission API, making Haskell dependencies visible to GitHub.

## Problem Statement

Currently, security auditing is fragmented.
Developers must install third-party tools such as `cabal-audit` or configure custom CI actions to verify that their build plan is free of known vulnerabilities.
Furthermore, Hackage does not show active vulnerabilities on package pages, meaning package consumers are often unaware of security issues until they run external audit tools.
Finally, there is no standardized way in `.cabal` metadata files or `cabal.project` configurations to declare advisory mappings or customize security database sources.

## Technical Content and Proposals

We propose three integrations to embed security checks natively into the Haskell build toolchain.

### 1. Integrate `cabal-audit` into Cabal

We propose to add a native command to the Cabal build tool: `cabal audit`.
This command will read the active build plan and check the dependencies against the Haskell Security Advisory database.
This native integration eliminates the need for developers to install external binaries. 
The configuration, such as ignoring specific advisory IDs, will be defined directly in `cabal.project`.

### 2. Expose Security Advisories on Hackage

We propose to integrate `hsec-sync` and `hsec-tools query` within the `hackage-server` infrastructure.
Every package page on Hackage will display a dedicated security section. 
If a package version is affected by an active advisory, a warning banner will be displayed on the page.
This will prevent developers from unknowingly using vulnerable versions when browsing Hackage.

### 3. Extend Cabal Syntax for Security Metadata

We propose to add dedicated fields to the Cabal package description syntax (`cabal-syntax`), allowing package maintainers to link advisories directly.
This will also allow curators to edit package metadata on Hackage to append security information.

#### Package Metadata Fields (`.cabal` format)

```cabal
security-advisories:
  haskell:HSEC-2026-0001
  haskell:HSEC-2026-0005
  file:HSEC-2026-0001
  my-company:HSEC-2026-0016
```

#### Project Configuration Fields (`cabal.project` format)

We propose the ability to declare custom security databases in `cabal.project` so that organizations can audit dependencies against internal or third-party advisory feeds:

```cabal
source-security-advisories
  name: my-company
  type: git
  location: https://gitlab.com/my-company/security-advisories
  branch: generated/snapshot-export
  
source-security-advisories
  name: a-provider
  type: url
  location: https://github.com/a-provider/security-advisories/archive/refs/heads/generated/snapshot-export.zip
```

## Stakeholders

* **Haskell Security Response Team**: Maintainers of the advisory database and core querying tools.
* **Cabal Maintainers**: Responsible for implementing `cabal audit` and parsing the new fields in `cabal-syntax`.
* **Hackage Administrators**: Responsible for integrating warning displays in `hackage-server`.
* **Haskell Developers**: Beneficiaries of integrated security checks in their build processes.

## Success

The project will be considered a success when:
1. `cabal audit` is a standard command in Cabal.
2. Package pages on Hackage display banners for active security advisories.
3. The `security-advisories` and `source-security-advisories` stanzas are parsed and supported by `cabal-install` and `cabal-syntax`.
