# Development Fund Proposal: DPM Trace and Source-Aware Debugging

**Author:** Walnut (https://walnut.dev)
**Status:** Draft  
**Created:** 2026-05-08  
**Label:** daml-tooling  
**Champion:** Need Champion

---

## Abstract

We propose to build `dpm trace`, an open-source DPM plugin for inspecting and debugging Canton transactions.

A developer should be able to start from an already committed update id, run a single CLI command, and get a readable transaction trace.

From there, the tool adds interactive inspection and source-aware debugging workflows. To make that reliable, we also propose compiler-generated debug metadata so transaction traces and debugger steps can link back to Daml source.

---

## Specification

### 1. Objective

Build the core tooling needed for source-aware Canton transaction debugging.

- Add `dpm trace <update-id>` so developers can inspect already committed updates from the CLI.
- Add an interactive debugger so developers can inspect committed updates and replay captured trace bundles.

We are open to adjusting the details with Canton developers, Daml/Canton maintainers, and the Tech & Ops Committee so the work solves the right problems.

The Canton developer survey mentions Tenderly-like or Foundry-like tooling. We agree with the direction, but a 1:1 copy does not fit Canton. We propose implementing only what matters for Canton.

We first looked at the current Canton stack and found useful primitives already exist:

- Transaction/update data is available through participant Ledger APIs, including the JSON Ledger API.
- Daml Script and IDE flows can produce useful local traces while tests or scripts are running.
- Participants already upload DARs and use package vetting to control which packages they are willing to process.

The gaps are:

- There is no CLI command for inspecting an already committed update by update id.
- There is no interactive debugger for local Daml Script/test workflows.
- There is no interactive inspection workflow for committed updates.
- Current package metadata can help decode Daml values, templates, choices, and fields, but it does not give debuggers reliable source locations or expression-level information.
- There is no **standard** format for representing Daml/Canton debugging information.

This proposal focuses on those gaps.

### 2. Implementation Mechanics

The work is split into three core components.

#### DPM Trace CLI

Add a DPM plugin command:

```bash
dpm trace <update-id> --participant <participant-url> --read-as <party>
```

The command connects to an authorized participant endpoint, fetches the participant-visible update, decodes it, and renders a readable trace tree.

It uses existing APIs:

- `UpdateService.GetUpdateById`
- JSON Ledger API `/v2/updates/update-by-id`
- `PackageService.GetPackage` where package metadata is available
- and optional event/query APIs for related visible contract payloads

The output is explicitly labeled as a participant-visible projection. The tool does not claim to reconstruct private data that the authorized party cannot see.

#### Cross-Participant Debugging

Canton debugging is participant-scoped. `dpm trace` shows the update projection available to the authorized participant and party context provided by the user.

For workflows involving multiple participants, the tool can query multiple authorized participant endpoints when the user provides access to them, and show the resulting projections as separately labeled views. It should always label which participant/party context each view came from, and it should be explicit when counterparty-private data is not available.

#### Interactive Debugger

Build interactive debugging around trace bundles. A trace bundle is a portable artifact that contains the transaction trace with the context available to the tool: participant-visible events, party context, package metadata, source/debug metadata, visible contract state where available, and privacy/projection labels.

The trace bundle format will be documented, versioned, and JSON-based, with explicit privacy labels, optional redaction guidance, and hashes or references for attached package/debug metadata.

```bash
dpm trace <update-id> --interactive
dpm trace test <script-or-test> --interactive
dpm trace bundle <update-id> --participant <participant-url> --read-as <party> --out <trace-bundle>
dpm trace record <script-or-test> --out <trace-bundle>
dpm trace replay <trace-bundle> --interactive
```

For committed updates, the simplest workflow is `dpm trace <update-id> --interactive`, which immediately opens an interactive trace inspector. The update already happened, so this mode cannot pause the original execution. It can step event by event, inspect values and state changes, set breakpoints on templates/choices/source locations, and show source links where metadata exists.

For local Daml Script/test workflows, `dpm trace test <script-or-test> --interactive` starts an interactive debugging session directly from the local workflow. This can provide more insight than committed-update inspection because the tool can observe the workflow while it is running.

The debugger has three entry points: inspect, record, and replay. A developer can inspect a committed update immediately, capture a committed update with `dpm trace bundle`, or record a local workflow with `dpm trace record`. The resulting bundle can be reopened later with `dpm trace replay <trace-bundle> --interactive` without reconstructing the original environment manually.

Milestones 2 and 3 focus on trace-level debugging, bundle capture, and replay. Full expression-level Daml-LF stepping is not assumed in these milestones, since it depends on the debug metadata and lower-level runtime hooks described in Milestone 4.

#### Debug Information Format and Compiler Integration

Current DAR metadata helps with package ids, modules, templates, choices, fields, and typed values. That is enough for decoding many traces, but it is not enough for robust source-level debugging.

We propose defining a versioned Canton debug information format for Daml contracts. The format can be embedded in a DAR or emitted as a sidecar build artifact.

Initial data:

- package id
- package name and version
- source root
- source file hashes
- module spans
- template spans
- choice spans
- expression spans where feasible
- deterministic debug step IDs that map source spans to Daml-LF/runtime execution points

Later extensions:

- richer local variable information
- profiling and coverage metadata
- future debugger, profiler, and coverage extensions

The preferred compiler entry point is:

```bash
damlc build --debug-info
```

This is similar in spirit to mature debug-info ecosystems such as DWARF on Unix-like systems and PDB/CodeView on Windows. In web3, Ethereum has recently started standardizing this problem through the `ethdebug` format. A shared Canton debug-info format would give compiler, debugger, profiler, and other tooling authors a common target to build against.

### 3. Architectural Alignment

The design follows Canton architecture:

- It works through authorized participant endpoints and respects participant-scoped visibility.
- The DPM plugin milestones require no Canton protocol changes.
- It uses existing Ledger API and JSON Ledger API primitives before proposing lower-level runtime work.
- It treats multi-participant transactions as multiple authorized projections, not as one globally visible transaction.
- It complements package upload and package vetting instead of replacing them.
- It uses DPM as the developer entry point, which keeps the UX close to existing Canton developer workflows.
- Compiler-generated debug information is opt-in. Existing Daml projects and DARs continue to work unchanged, and debugging tools use the metadata only when it is available.

The proposal aligns directly with Daml Language & Developer Tooling, Canton APIs, and DAR Package Management priorities.

### 4. Backward Compatibility

The DPM plugin is additive and does not change existing Daml applications, Canton nodes, Ledger API semantics, or package vetting.

The compiler debug-info flag is opt-in. DARs built without debug information continue to work exactly as they do today. Tools should degrade gracefully: with no debug info, they still show decoded package data and transaction structure; with debug info, they add source links and richer stepping.

---

## Milestones and Deliverables

### Milestone 1: Transaction Trace CLI

**Estimated Delivery:** 4 weeks from start
**Focus:** One useful command to inspect committed updates.

**Deliverables / Value Metrics:**

- `dpm trace <update-id>` plugin command.
- Support for local and remote participant JSON Ledger API endpoints.
- Support for `--read-as` party context and bearer-token based access.
- Human-readable transaction tree for already committed updates.
- Create, exercise, archive, and reassignment event rendering.
- Contract ids, parties, witnesses, signatories, observers, choice arguments, return values, and payloads where visible.
- Participant-visible projection labeling.
- JSON trace artifact for downstream tools.
- Documentation with a local Canton example and a remote participant example.

**Acceptance Criteria:**

- A developer can run `dpm trace <update-id> --participant <url> --read-as <party>` against a local Canton participant and inspect a committed update.
- The same command shape works against an authorized remote participant endpoint.
- The CLI renders a readable transaction tree for create, exercise, archive, and reassignment events where present.
- The CLI can emit a normalized JSON trace artifact.
- Output labels the result as a participant-visible projection and does not imply access to non-visible private data.
- At least three representative Daml examples are included: create, exercise with child create, and archive/consuming exercise.

### Milestone 2: Trace Bundles

**Estimated Delivery:** 4 weeks after Milestone 1 acceptance
**Focus:** Make committed updates portable and reusable.

**Deliverables / Value Metrics:**

- `dpm trace bundle <update-id>` to save a committed update and its available context as a trace bundle.
- `dpm trace replay <trace-bundle>` to load a saved trace bundle and render the saved trace.
- Portable, versioned trace bundle format.
- Bundle contents: transaction trace, participant/read-as context, visible events, package metadata, reserved source/debug metadata slots, visible contract state where available, and privacy/projection labels. The richer debug-info format is defined in Milestone 4.
- Bundle format designed to support both committed-update bundles and local workflow recordings without a breaking change.
- Bundle privacy guidance covering what is included, what can be redacted, and how participant-visible labels are preserved.
- Bundle validation so corrupted or incompatible artifacts fail clearly.
- Documentation explaining trace bundles, participant projections, and privacy limits.

**Acceptance Criteria:**

- A developer can create a trace bundle from a committed update and reopen it later.
- The trace bundle schema is documented, versioned, and includes privacy/redaction guidance.
- A developer can replay a committed-update bundle from the CLI and see the same participant-visible trace data.
- Invalid or unsupported bundle versions produce clear errors.
- Documentation clearly states that a trace bundle is a participant-visible artifact, not a full global transaction record.
- Privacy labeling is present in every debug session.

### Milestone 3: Interactive Debugger

**Estimated Delivery:** 4 weeks after Milestone 2 acceptance  
**Focus:** Add interactive debugging for committed updates and local Daml Script/test workflows.

**Deliverables / Value Metrics:**

- `dpm trace <update-id> --interactive` for immediate committed-update inspection.
- `dpm trace replay <trace-bundle> --interactive` for committed-update and local workflow bundles.
- `dpm trace test <script-or-test> --interactive` for local Daml Script/test workflows.
- `dpm trace record <script-or-test> --out <trace-bundle>` for local workflows.
- Event-by-event replay over the participant-visible transaction tree for committed updates.
- Local workflow bundles with richer recorded context where available.
- REPL-style debugger with next/previous/continue/source/context commands.
- Breakpoints on event index, template, choice, and source location where metadata exists.
- Source snippet display where metadata exists.
- Variable and contract inspector for visible values.
- State-diff view for created and archived contracts.
- Documentation covering the two debugger modes: committed-update inspection and local Daml Script/test debugging.

**Acceptance Criteria:**

- A developer can open an immediate interactive inspector from a committed update without first writing a bundle.
- A developer can replay a committed-update bundle interactively from the CLI.
- A developer can run an immediate interactive debugger for a local Daml Script/test workflow.
- A developer can record a local Daml Script/test workflow into a trace bundle.
- A developer can replay a locally recorded bundle interactively from the CLI.
- Breakpoints work for event id, template, choice, and source location when debug metadata is present.
- Local workflow debugging provides more context than committed-update inspection when the runtime exposes it.
- The debugger degrades cleanly when source metadata is missing.
- Documentation explains the supported debugger modes and their requirements.

### Milestone 4: Debug Information Format and Compiler Integration

**Estimated Delivery:** 6 weeks after Milestone 3 acceptance
**Focus:** Make source-aware debugging reliable by defining and emitting debug metadata.

**Deliverables / Value Metrics:**

- Draft Canton/Daml debug-info format.
- Reference implementation for generating debug-info artifacts.
- Preferred compiler UX: `damlc build --debug-info`, subject to review with Daml maintainers.
- Sidecar JSON output, with DAR embedding as the preferred long-term path if accepted by maintainers.
- Source file hashes and package-id matching.
- Template and choice source spans.
- Initial expression span support where compiler data is available.
- Define deterministic debug step IDs for source expressions and emit them in the debug-info artifact, so runtime/debugger events can reference exact Daml source spans.
- `dpm trace --debug-info <path>` support, where the path can be a file, directory, or manifest for multi-package traces.
- Runtime integration note describing what Daml-LF/interpreter debug events are needed for full expression-level stepping.
- Documentation for generating debug-info artifacts and using them with `dpm trace`.

**Acceptance Criteria:**

- A sample Daml project can be built with debug info.
- `dpm trace` can load the debug-info artifact and link trace nodes to source locations.
- Debug-info output includes deterministic debug step IDs for supported source spans.
- The format is documented and versioned.
- The design is shared with Daml/Canton maintainers.

---

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- The delivered tools work on the current stable DPM/Daml SDK version at the time of development.
- The CLI commands can be installed and run in a clean environment.
- Trace and debugger output is useful for committed updates, not only for locally executed scripts.
- Participant-scoped privacy is correctly represented in command syntax, output, artifacts, and documentation.
- Debug-info artifacts are versioned, documented, and matched to package ids.
- All software is released as open source under Apache-2.0 unless otherwise agreed.
- Documentation includes local development and authorized remote participant workflows.

Ecosystem value will be measured by:

- Working CLI trace and interactive debugger demos against representative Canton examples.
- Feedback from Canton ecosystem developers, Daml/Canton maintainers, or Tech & Ops reviewers.
- A reviewed and documented debug-info format that can be reused by future tools.

---

## Potential Follow-Ons

The work in this proposal is intended to be a foundation. Once the core toolchain is in place, several extensions become natural follow-ons. They are listed here for context only and are **not part of this funding request**. Any of them can be picked up later as separate proposals once the core deliverables have been validated by the ecosystem.

### Source and Debug Metadata Registry

An open-source registry for verified Daml source and debug metadata, so source-aware debugging works for packages a developer has not built locally.

This would not replace Canton package upload or package vetting. Vetting says a participant is willing to process a package; source/debug verification would say that a given source tree and debug-info artifact correspond to a given package id.

Likely shape:

- Hosted or permissioned instance for open-source Canton packages, examples, and ecosystem packages.
- Self-hosted/private instance for institutions that cannot publish source externally.
- Storage of DARs where permitted, source files, package ids, debug-info artifacts, template/choice/field metadata, source spans, and optional repository metadata.
- `dpm trace` lookup by package id to make historical traces source-aware.

Open questions to resolve before this becomes its own proposal:

- Reproducibility of `damlc` builds across SDK versions, OS, and build environments, and a verification status model that handles partial matches.
- Threat model: who can publish, anti-squatting on package ids, source poisoning, and spam control.
- Licensing of registry data, separate from the Apache-2.0 code license.
- How pre-existing DARs without debug info are represented.

### Other Likely Follow-Ons

- A web UI on top of the trace bundle format and debug-info format.
- A DAP-compatible adapter and VS Code extension for the interactive debugger.
- Simulation and what-if tooling on top of replay bundles.
- Profiling and coverage extensions built on top of the debug-info format.

---

## Funding

**Total Funding Request:** 1,580,000 Canton Coin

### Payment Breakdown by Milestone

- Milestone 1, Transaction Trace CLI: 320,000 CC upon committee acceptance
- Milestone 2, Trace Bundles: 320,000 CC upon committee acceptance
- Milestone 3, Interactive Debugger: 320,000 CC upon committee acceptance
- Milestone 4, Debug Information Format and Compiler Integration: 620,000 CC upon committee acceptance

---

## Co-Marketing

Walnut is happy to collaborate with Canton on co-marketing if there is interest. Examples of content we could create together:

- A technical blog post explaining transaction debugging under Canton’s participant privacy model.
- A short demo video showing `dpm trace`, interactive debugging, and source-aware traces.
- A follow-up writeup comparing local script traces, committed update traces, and source-aware debugging.

---

## Motivation

Better debugging tooling is one of the highest-leverage improvements for developer experience. Canton already has strong primitives, but they are not packaged into a workflow that feels like modern smart-contract debugging.

Today, developers can get useful traces while running Daml Script tests, and authorized participant APIs expose committed update data. The missing workflow is what developers expect from mature blockchain tooling: start from a transaction/update id, inspect what happened, navigate the trace, connect it back to source, and share a reproducible debugging artifact.

This work benefits application developers, auditors, package reviewers, participant operators, and teams onboarding to Canton from other ecosystems.

---

## Rationale

We are not starting with a hosted UI as the first deliverable.

The right first step is to build the debugging toolchain in layers:

1. A small trace command that proves committed updates can become developer-readable.
2. A trace bundle format for saving and replaying committed updates.
3. An interactive debugger for committed updates and local workflows.
4. A debug-info format so source mapping is based on compiler-generated metadata.

This approach fits Canton’s architecture. It starts with participant-visible data, keeps privacy boundaries explicit, reuses DPM and Ledger APIs, and only proposes compiler work where the current metadata is not enough.

This proposal is not the whole "Tenderly for Canton" story. It focuses on the lower-level building blocks first: update tracing, replay bundles, and debug metadata. Follow-on work could include a Web UI, DAP/VS Code support, simulation tools, or a source/debug metadata registry, all areas where Walnut already has relevant experience.

Walnut is well suited for this work. Our team has four years of experience building blockchain debugging and observability tooling, including:

- [Walnut Starknet Debugger](https://walnut.dev/), a multi-year partnership with StarkWare covering debug-info generation, transaction simulation, contract verification, network forking, and hosted debugger UI.
- [Debug info generation in the official Solidity compiler, `solc`](https://github.com/argotorg/solidity), a one-year collaboration with the Ethereum Foundation / Argot Collective, to be extended.
- [StylusDB](https://github.com/OffchainLabs/stylus-sdk-rs/blob/main/cargo-stylus/docs/StylusDebugger.md), a CLI debugger for Arbitrum Stylus and cross-VM Solidity/Rust debugging workflows.

---

## References

- `dpm trace` proof of concept: https://github.com/walnuthq/dpm-trace
- Initial `damlc --debug-info` support branch: https://github.com/walnuthq/daml/tree/feature/debug-info
- JSON Ledger API OpenAPI: https://docs.digitalasset.com/build/3.4/reference/json-api/openapi.html
- Canton package management and vetting: https://docs.daml.com/canton/usermanual/packagemanagement.html#understanding-package-vetting
- ETHDebug format: https://ethdebug.github.io/format/
