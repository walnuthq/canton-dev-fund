# Development Fund Proposal: DPM Trace Transaction Visualization

**Author:** [Walnut](https://walnut.dev)<br>
**Status:** Draft<br>
**Created:** 2026-05-08<br>
**Label:** daml-tooling<br>
**Champion:** Need Champion<br>

---

## Table of Contents

- [Abstract](#abstract)
- [Specification](#specification)
  - [1. Objective](#1-objective)
  - [2. Motivation](#2-motivation)
  - [3. Implementation Mechanics](#3-implementation-mechanics)
  - [4. Architectural Alignment](#4-architectural-alignment)
  - [5. Backward Compatibility](#5-backward-compatibility)
- [Proof of Concept Implementation](#proof-of-concept-implementation)
- [Milestones and Deliverables](#milestones-and-deliverables)
  - [Milestone 1: Transaction Trace CLI and Artifact](#milestone-1-transaction-trace-cli-and-artifact)
  - [Milestone 2: Interactive Transaction Visualizer](#milestone-2-interactive-transaction-visualizer)
  - [Milestone 3: Prepare and Compare Workflows](#milestone-3-prepare-and-compare-workflows)
  - [Milestone 4: Adoption and Ecosystem Validation](#milestone-4-adoption-and-ecosystem-validation)
- [Acceptance Criteria](#acceptance-criteria)
- [Potential Follow-Ons](#potential-follow-ons)
- [Funding](#funding)
- [Co-Marketing](#co-marketing)
- [Rationale](#rationale)
- [About the Team](#about-the-team)
- [References](#references)

---

## Abstract

We propose to build `dpm trace`, an open-source DPM plugin for visualizing and comparing Canton transactions and command outcomes.

A developer should be able to start from an already committed update id, run a single CLI command, and get a readable participant-visible transaction trace.

On top of that trace, `dpm trace` will add an interactive visualizer, reusable trace artifacts, and comparison workflows for prepared commands, successful updates, failed completions, and repeated business operations.

---

## Specification

### 1. Objective

Build the core tooling needed for Canton transaction visualization and comparison.

- Add `dpm trace <update-id>` so developers can inspect already committed updates from the CLI.
- Add an interactive transaction visualizer so developers can navigate moderate transaction trees without reading raw JSON.
- Add prepare and compare workflows so developers can compare intended command outcomes with actual committed or failed outcomes.
- Add a portable trace artifact so a participant-visible transaction projection can be saved, shared, reopened, or consumed by future UI tooling.

We are open to adjusting the details with Canton developers, Daml/Canton maintainers, and the Tech & Ops Committee.

### 2. Motivation

The Canton developer survey mentions Tenderly-like or Foundry-like tooling. We agree with the direction, but a 1:1 copy does not fit Canton. We propose implementing only what matters for Canton.

We analysed the current Canton stack and found useful primitives that already exist:

- Transaction/update data is available through participant Ledger APIs, including the JSON Ledger API.
- Command completions expose successful and failed command outcomes for authorized clients.
- Daml Script and IDE flows can produce useful local traces while tests or scripts are running.
- Participants already upload DARs and use package vetting to control which packages they are willing to process.

The gaps are:

- There is no single DPM command for inspecting the trace of an already committed update.
- There is no interactive visualization workflow for navigating a transaction tree, state changes, parties, arguments, payloads, and return values.
- There is no standard portable artifact for saving a participant-visible update projection and reopening it later.
- There is no developer-friendly diff between a prepared command and an actual successful update.
- There is no developer-friendly diff between an intended command and a failed completion, with error context and related logs where available.
- There is no simple way to compare two updates that represent the same business operation and understand why they differ.

This proposal focuses on those gaps.

### 3. Implementation Mechanics

The work is split into four core components.

#### DPM Trace CLI

Add a DPM plugin command:

```bash
dpm trace <update-id> \
  --participant-url <participant-url> \
  --read-as <party> \
  --access-token-file <token-file>
```

The command connects to an authorized participant endpoint, fetches the participant-visible update, decodes it, and renders a readable trace tree with templates, choices, arguments, payloads, return values, parties, and contract ids where visible.

The participant endpoint is the Ledger API or JSON Ledger API endpoint. The authorization model is explicit: the user supplies a participant URL, an access token, and a party context such as `--read-as` or `--act-as`, depending on the operation.

It uses existing APIs:

- `UpdateService.GetUpdateById`
- JSON Ledger API `/v2/updates/update-by-id`
- `PackageService.GetPackage` where package metadata is available
- optional event/query APIs for related visible contract payloads

The CLI can also export the normalized trace:

```bash
dpm trace <update-id> \
  --participant-url <participant-url> \
  --read-as <party> \
  --access-token-file <token-file> \
  --export trace.json
```

#### Interactive Transaction Visualizer

Add an interactive mode:

```bash
dpm trace <update-id> --visualize
```

This opens an interactive visualizer for the committed update. The update already happened, so the tool is not pausing live execution. It is helping the developer understand what happened.

The visualizer should make a moderate transaction tree easy to navigate:

- Expandable event tree for create, exercise, archive, and reassignment events.
- Event details panel with arguments, payloads, return values, actors, witnesses, signatories, observers, and contract ids.
- State diff panel for contracts created, archived, reassigned, or otherwise affected.
- Search and filters by template, choice, party, contract id, event type, and package id.
- Source hints where existing package/project metadata can provide them.
- Clear participant/projection labels so the user knows which participant and party rights produced the view.

The same visualizer can also reopen exported trace artifacts.

#### Prepare and Compare

Add a workflow for visualizing a prepared command before it is submitted:

```bash
dpm trace prepare \
  --participant-url <participant-url> \
  --act-as <party> \
  --access-token-file <token-file> \
  --commands commands.json \
  --export prepared.json
```

This command runs Canton's non-committing prepare flow. The result can be visualized and compared with a committed update or failed completion, but it is not itself a committed transaction.

Add compare commands:

```bash
dpm trace compare --prepared prepared.json --update <update-id>
dpm trace compare --prepared prepared.json --completion <command-id>
dpm trace compare <update-id-a> <update-id-b>
```

The comparison workflow should support:

- Prepared command vs actual successful update.
- Prepared command vs failed completion, including completion status and error details.
- Two successful updates that represent the same business operation.
- State diff comparison for creates, archives, reassignments, payload changes, parties, and return values.
- Log correlation where the operator or developer provides logs that can be matched by command id, update id, trace context, or time window.

It uses existing APIs:

- `InteractiveSubmissionService.PrepareSubmission`
- JSON Ledger API interactive submission equivalent where available
- `CommandCompletionService.CompletionStream` for successful and failed completions
- `UpdateService.GetUpdateById` for successful committed updates
- `PackageService.GetPackage` for value/package decoding where available

Failed submissions may not have an `update_id`. In that case the tool works from the command completion and any available operator-provided logs, rather than pretending there is a committed transaction to fetch.

#### Trace Artifacts and Privacy

The trace artifact is a portable JSON document for a participant-visible transaction projection or prepared command result.

It may contain:

- committed update data or prepared command output
- participant URL label, party context, and visibility/projection labels
- visible transaction events
- completion status and error details where available
- package metadata references where available
- optional log attachments or references
- redaction metadata for fields the user chooses not to share

`dpm trace` should always label the endpoint and party context used to fetch data, and should not imply a global transaction view.

### 4. Architectural Alignment

The design follows Canton architecture:

- It works through authorized participant endpoints and respects participant-scoped visibility.
- It uses DPM as the developer entry point, which keeps the UX close to existing Canton developer workflows.
- It uses existing Ledger API and JSON Ledger API primitives.
- It supports bearer-token based authorization and explicit party context.
- It complements package upload and package vetting instead of replacing them.
- It does not require Canton protocol changes, Canton node changes, Daml compiler changes, or Daml-LF interpreter changes.
- It handles failed submissions through completion data, without inventing a committed update that does not exist.

The proposal aligns directly with Daml Language & Developer Tooling, Canton APIs, and DAR Package Management priorities.

### 5. Backward Compatibility

The proposed solution is additive. It does not change existing Daml applications, Canton nodes, Ledger API semantics, package upload, or package vetting.

---

## Proof of Concept Implementation

As part of this proposal work, Walnut built a proof of concept to validate the core technical assumptions before asking for funding.

- `dpm trace` proof of concept: [https://github.com/walnuthq/dpm-trace](https://github.com/walnuthq/dpm-trace)

The proof of concept fetches committed updates from an authorized participant, renders a readable trace tree, exports trace data, and demonstrates the shape of an interactive CLI inspection flow.

---

## Milestones and Deliverables

### Milestone 1: Transaction Trace CLI and Artifact

**Estimated Delivery:** 4 weeks from start<br>
**Estimated Effort:** 8 engineer-weeks<br>
**Focus:** One useful command to inspect committed updates and export a reusable trace artifact.

**Deliverables / Value Metrics:**

- `dpm trace <update-id>` plugin command.
- Support for local and remote participant JSON Ledger API endpoints.
- Support for `--participant-url`, `--read-as`, and bearer-token based access.
- Human-readable transaction tree for already committed updates.
- Create, exercise, archive, and reassignment event rendering.
- Contract ids, parties, witnesses, signatories, observers, choice arguments, return values, and payloads where visible.
- Participant-visible projection labeling.
- `--export trace.json` support for a normalized trace artifact.
- Documentation with a local Canton example and an authorized remote participant example.

**Acceptance Criteria:**

- A developer can run `dpm trace <update-id> --participant-url <url> --read-as <party>` against a local Canton participant and inspect a committed update.
- The same command shape works against an authorized remote participant endpoint.
- The CLI renders a readable transaction tree for create, exercise, archive, and reassignment events where present.
- The CLI can emit a normalized JSON trace artifact.
- Output labels the result as a participant-visible projection and does not imply access to non-visible private data.
- At least three representative Daml examples are included: create, exercise with child create, and archive/consuming exercise.

### Milestone 2: Interactive Transaction Visualizer

**Estimated Delivery:** 4 weeks after Milestone 1 acceptance<br>
**Estimated Effort:** 8 engineer-weeks<br>
**Focus:** Make transaction trees easy to navigate and understand.

**Deliverables / Value Metrics:**

- `dpm trace <update-id> --visualize` for committed-update visualization.
- `dpm trace open trace.json` for exported trace artifacts.
- Expandable transaction tree.
- Event details panel.
- State diff panel.
- Search and filters by template, choice, party, contract id, event type, and package id.
- Participant/projection labels shown in the visualizer.
- Best-effort symbol/source hints when local project metadata is available.
- Documentation explaining visualizer usage and privacy limits.

**Acceptance Criteria:**

- A developer can open an interactive visualizer from a committed update without manually reading JSON.
- A developer can reopen an exported trace artifact and see the same participant-visible transaction view.
- The visualizer can navigate a representative transaction tree with nested exercise/create events.
- Search and filters work for at least template, choice, party, and contract id.
- The state diff view clearly shows created and archived contracts where present.
- Documentation clearly states that the visualizer shows a participant-visible projection, not a global transaction record.

### Milestone 3: Prepare and Compare Workflows

**Estimated Delivery:** 5 weeks after Milestone 2 acceptance<br>
**Estimated Effort:** 10 engineer-weeks<br>
**Focus:** Help developers understand why intended and actual outcomes differ.

**Deliverables / Value Metrics:**

- `dpm trace prepare --commands commands.json` for preparing and visualizing prepared transaction data before submission.
- Prepared command artifacts labeled as prepared, not committed.
- `dpm trace compare --prepared prepared.json --update <update-id>`.
- `dpm trace compare --prepared prepared.json --completion <command-id>` using the authorized completions endpoint.
- `dpm trace compare --prepared prepared.json --completion-file completion.json` for captured completion/error artifacts.
- `dpm trace compare <update-id-a> <update-id-b>`.
- Diff view for event tree changes, state diff changes, parties, arguments, payloads, return values, and completion status.
- Completion integration for failed command outcomes where available.
- Failed-command comparison based on completion status, error details, submission id, trace context, offset, parties, and synchronizer time.
- Documentation explaining what can and cannot be compared.

**Acceptance Criteria:**

- A developer can prepare a command without committing it and inspect the prepared result.
- A developer can compare a prepared result with a successful committed update.
- A developer can compare a prepared result with a failed completion where the completion is available.
- A developer can compare two successful updates and see event/state differences.
- Failed-command views show completion status and error details without claiming that a failed command has a committed update id.
- Failed-command comparison works from a documented completion/error artifact and does not require log access.

### Milestone 4: Adoption and Ecosystem Validation

**Estimated Delivery:** 4 weeks after Milestone 3 acceptance<br>
**Estimated Effort:** 4 engineer-weeks plus support during the adoption window<br>
**Focus:** Prove that the tooling is useful to Canton developers outside Walnut.

**Deliverables / Value Metrics:**

- Public release of the `dpm trace` plugin with installation instructions.
- Getting-started guide covering local Canton, authorized remote participant usage, exported trace artifacts, visualization, prepare, and compare.
- At least one short demo video or recorded walkthrough.
- Outreach to Canton ecosystem developers, Daml/Canton maintainers, and Tech & Ops reviewers for hands-on testing.
- Feedback collection from at least three independent organizations using Canton or building Canton applications.
- Fixes or follow-up issues for adoption-blocking bugs found during the adoption window.
- Short adoption report summarizing who tested the tool, what workflows were tested, whether it sped up development, and what should be improved next.

**Acceptance Criteria:**

- At least three independent organizations run the trace workflow against a dApp project and confirm whether it sped up development and how.
- At least two independent organizations test the visualizer or compare workflow, confirmed by written feedback, public issue/PR activity, or other verifiable evidence.
- The getting-started guide lets a new user produce their first trace without Walnut assistance.
- Adoption feedback is published as a short report with concrete follow-up items.
- Any critical issue that blocks the documented happy path is fixed or explicitly documented with a workaround.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- The delivered tools work on the current stable DPM/Daml SDK version at the time of development.
- The CLI commands can be installed and run in a clean environment.
- Trace and visualizer output is useful for committed updates.
- Prepare and compare workflows are useful for understanding intended vs actual outcomes.
- Participant-scoped privacy is correctly represented in command syntax, output, artifacts, and documentation.
- Failed command handling uses completion data and available logs without implying a committed update exists.
- All software is released as open source under Apache-2.0 unless otherwise agreed.
- Documentation includes local development and authorized remote participant workflows.

Ecosystem value will be measured by:

- Working CLI trace, visualizer, prepare, and compare demos against representative Canton examples.
- Feedback from Canton ecosystem developers, Daml/Canton maintainers, or Tech & Ops reviewers.
- Adoption outcomes described in Milestone 4.

---

## Potential Follow-Ons

The work in this proposal is intentionally limited to transaction visualization, artifacts, prepare, compare, and adoption. Several follow-ons become natural once these primitives are validated. They are listed here for context only and are **not part of this funding request**.

### Compiler Source Metadata

A versioned source/debug metadata schema and richer compiler-emitted debug information may be valuable follow-on work. This should build on existing LF spans and be scoped as a separate proposal with Daml/Canton maintainers.

### Source and Metadata Registry

A Sourcify-like registry may be useful later for verified Daml source and metadata, especially for packages a developer has not built locally. This would not replace Canton package upload or package vetting. Vetting says a participant is willing to process a package; source/metadata verification would say that a given source tree or artifact corresponds to a given package id.

### Other Likely Follow-Ons

- A hosted web UI on top of trace artifacts.
- A DAP-compatible adapter and VS Code extension if source-level tooling becomes a separate workstream.
- More advanced simulation and what-if tooling.
- Profiling and coverage tooling if a debug-info format is standardized later.

---

## Funding

**Total Funding Request:** 1,900,000 Canton Coin

The funding is split so that roughly half is tied to delivery and roughly half is tied to ecosystem adoption.

### Payment Breakdown by Milestone

- Milestone 1, Transaction Trace CLI and Artifact: 320,000 CC upon committee acceptance
- Milestone 2, Interactive Transaction Visualizer: 320,000 CC upon committee acceptance
- Milestone 3, Prepare and Compare Workflows: 310,000 CC upon committee acceptance
- Milestone 4, Adoption and Ecosystem Validation: 950,000 CC upon committee acceptance and adoption criteria

---

## Co-Marketing

Walnut is happy to collaborate with Canton on co-marketing if there is interest. Examples of content we could create together:

- A technical blog post explaining transaction visualization under Canton's participant privacy model.
- A short demo video showing `dpm trace`, the transaction visualizer, and compare workflows.
- A follow-up writeup explaining lessons from the adoption window.

---

## Rationale

We are not starting with a hosted UI as the first deliverable.

The right first step is to build the transaction inspection toolchain in layers:

1. A trace command that makes committed updates developer-readable.
2. A portable trace artifact for saving and reopening participant-visible projections.
3. An interactive visualizer for navigating transaction trees and state changes.
4. Prepare and compare workflows for intended vs actual outcomes.
5. Adoption validation with independent Canton organizations.

This gives Canton developers useful tooling quickly without requiring protocol, node, compiler, or runtime changes.

## About the Team

Walnut is well suited for this work. Our team has four years of experience building blockchain debugging and observability tooling. We partner with leading ecosystems:

- Starkware / Starknet. We are building [Walnut Starknet Debugger](https://walnut.dev/), aka Tenderly for Starknet. Our work is end-to-end and involves debug info generation, tracing, transaction simulation, contract verificaiton, network forking and infra for hosting the debugger web app. 3-year ongoing partnership.
- Ethereum Foundation / Argot: [Debug info generation in the official Solidity compiler, ](https://github.com/argotorg/solidity)[`solc`](https://github.com/argotorg/solidity). We own debug info generation in solc - the official Solidity compiler. 1-year partnership, to be extended.
- Arbitrum/Offchain Labs: [StylusDB](https://github.com/OffchainLabs/stylus-sdk-rs/blob/main/cargo-stylus/docs/StylusDebugger.md), an official debugger for Stylus (Rust) and debugging Solidity \<\> Rust interoperability transactions.

---

## References

- JSON Ledger API OpenAPI: [https://docs.digitalasset.com/build/3.4/reference/json-api/openapi.html](https://docs.digitalasset.com/build/3.4/reference/json-api/openapi.html)
- gRPC Ledger API Reference: [https://docs.digitalasset.com/build/3.4/reference/lapi-proto-docs.html](https://docs.digitalasset.com/build/3.4/reference/lapi-proto-docs.html)
- Canton package management and vetting: [https://docs.daml.com/canton/usermanual/packagemanagement.html#understanding-package-vetting](https://docs.daml.com/canton/usermanual/packagemanagement.html#understanding-package-vetting)
