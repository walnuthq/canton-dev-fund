# Development Fund Proposal: DPM Trace and Source-Aware Debugging

**Author:** Walnut  
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

The Canton developer survey mentions Tenderly-like or Foundry-like tooling. We agree with the direction, but Canton is not EVM. Canton has participant-scoped privacy, Daml packages, package vetting, and Ledger API projections. A direct copy of EVM forking or global transaction debugging would be the wrong model.

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
- There is no common source/debug metadata registry comparable to source verification systems in other smart contract ecosystems. This is useful, but can be treated as optional follow-on work after the core trace/debug-info tooling is in place.

This proposal focuses on those gaps.

### 2. Implementation Mechanics

The work is split into three core components and one optional follow-on component.

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

#### Interactive Debugger

Build interactive debugging around trace bundles. A trace bundle is a portable artifact that contains the transaction trace with the context available to the tool: participant-visible events, party context, package metadata, source/debug metadata, visible contract state where available, and privacy/projection labels.

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

#### Debug Information Standard and `damlc` Support

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

The compiler entry point can be:

```bash
damlc build --debug-info
```

This is similar in spirit to mature debug-info ecosystems such as DWARF on Unix-like systems and PDB/CodeView on Windows. In web3, Ethereum has recently started standardizing this problem through the `ethdebug` format. A shared Canton debug-info schema would give compiler, debugger, profiler, and other tooling authors a common target to build against.

#### Optional Follow-On: Source and Debug Metadata Registry

As an optional follow-on, build an open-source registry for verified Daml source and debug metadata.

This does not replace Canton package upload or package vetting. Vetting says a participant is willing to process a package. Source/debug verification says that a given source tree and debug-info artifact correspond to a given package id.

The registry should support two operating modes:

- Hosted or permissioned instance for open-source Canton packages, examples, and ecosystem packages.
- Self-hosted/private instance for institutions that cannot publish source externally.

The registry stores DARs where permitted, source files, package ids, debug-info artifacts, template/choice/field metadata, source spans, and optional repository metadata. `dpm trace` can then look up metadata by package id to make historical traces source-aware.

### 3. Architectural Alignment

The design follows Canton architecture:

- It works through authorized participant endpoints and respects participant-scoped visibility.
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

**Estimated Delivery:** 2 weeks from start  
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

### Milestone 2: Interactive Debugger

**Estimated Delivery:** 5 weeks after Milestone 1 acceptance  
**Focus:** One interactive debugger with three entry points: inspect, record, and replay.

**Deliverables / Value Metrics:**

- Immediate interactive inspection for committed updates.
- Immediate interactive debugging for local Daml Script/test workflows.
- Record/bundle support for committed updates and local workflows.
- Interactive replay for captured trace bundles.
- Portable trace bundle format.
- Bundle contents: transaction trace, participant/read-as context, visible events, package metadata, source/debug metadata, visible contract state where available, and privacy/projection labels.
- REPL-style debugger with next/previous/continue/source/context commands.
- For committed updates: event-by-event replay over the participant-visible transaction tree.
- For local tests/scripts: interactive debugging over the local workflow, with richer recorded context where available.
- Breakpoints on event index, template, choice, and source location where metadata exists.
- Source snippet display.
- Variable and contract inspector for visible values.
- State-diff view for created and archived contracts.
- Documentation explaining immediate inspection, recording, replay bundles, and the difference between committed-update replay and local workflow debugging.

**Acceptance Criteria:**

- A developer can create a trace bundle from a committed update and reopen it later.
- A developer can record a local Daml Script/test workflow into a trace bundle.
- A developer can open an immediate interactive inspector from a committed update without first writing a bundle.
- A developer can run an immediate interactive debugger for a local Daml Script/test workflow.
- A developer can replay a bundle interactively from the CLI.
- Breakpoints work for event id, template, choice, and source location when debug metadata is present.
- The debugger degrades cleanly when source metadata is missing.
- Privacy labeling is present in every debug session.

### Milestone 3: Debug Information Standard and Compiler Support

**Estimated Delivery:** 4 weeks after Milestone 2 acceptance  
**Focus:** Make source-aware debugging reliable by defining and emitting debug metadata.

**Deliverables / Value Metrics:**

- Draft Canton/Daml debug-info schema.
- `damlc build --debug-info` support.
- Sidecar JSON output or DAR-embedded metadata, subject to review with Daml maintainers.
- Source file hashes and package-id matching.
- Template and choice source spans.
- Initial expression span support where compiler data is available.
- Define deterministic debug step IDs for source expressions and emit them in the debug-info artifact, so runtime/debugger events can reference exact Daml source spans.
- `dpm trace --debug-info <file>` support.
- Runtime integration note describing what Daml-LF/interpreter debug events are needed for full expression-level stepping.

**Acceptance Criteria:**

- A sample Daml project can be built with debug info.
- `dpm trace` can load the debug-info artifact and link trace nodes to source locations.
- Debug-info output includes deterministic debug step IDs for supported source spans.
- The schema is documented and versioned.
- The design is shared with Daml/Canton maintainers.

### Optional Milestone 4: Source and Debug Metadata Registry

**Estimated Delivery:** 4 weeks after Milestone 3 acceptance  
**Focus:** Optional follow-on to make source-aware debugging usable beyond a local checkout.

**Deliverables / Value Metrics:**

- Open-source registry service for package/source/debug metadata.
- CLI upload and lookup commands.
- Lookup by package id.
- Verification flow that checks source/debug-info correspondence to package id.
- Hosted Walnut demo instance for public or permissioned packages.
- Self-hosting guide for private institutional deployments.
- Documentation explaining how this complements, and does not replace, package vetting.

**Acceptance Criteria:**

- A package can be uploaded or verified and then resolved by package id.
- `dpm trace` can fetch metadata from the registry and use it for source-aware traces.
- The self-hosted deployment path can be run by an institution without sending private source to Walnut.
- At least one open-source/sample package is published to the hosted instance as a working example.

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
- If Optional Milestone 4 is accepted, documentation also includes private/self-hosted registry workflows.

Ecosystem value will be measured by:

- Working CLI trace and interactive debugger demos against representative Canton examples.
- Feedback from Canton ecosystem developers, Daml/Canton maintainers, or Tech & Ops reviewers.
- A reviewed and documented debug-info schema that can be reused by future tools.
- If Optional Milestone 4 is accepted, a working metadata registry example for both hosted and self-hosted workflows.

---

## Funding

**Core Funding Request:** 750,000 Canton Coin

**Optional Add-On Funding Request:** 200,000 Canton Coin

**Total Funding Request if Optional Milestone 4 is accepted:** 950,000 Canton Coin

### Payment Breakdown by Milestone

- Milestone 1, Transaction Trace CLI: 150,000 CC upon committee acceptance
- Milestone 2, Interactive Debugger: 350,000 CC upon committee acceptance
- Milestone 3, Debug Information Standard and Compiler Support: 250,000 CC upon committee acceptance
- Optional Milestone 4, Source and Debug Metadata Registry: 200,000 CC upon final release and acceptance

### Volatility Stipulation

The project duration is expected to be under 6 months. Should the project timeline extend beyond 6 months due to Committee-requested scope changes, any remaining milestones must be renegotiated to account for significant USD/CC price volatility.

---

## Co-Marketing

Upon release, Walnut will collaborate with the Canton Foundation on:

- A technical blog post explaining transaction debugging under Canton’s participant privacy model.
- A short demo video showing `dpm trace`, interactive debugging, and source-aware traces.
- Developer documentation and examples for local and remote participant workflows.
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
2. An interactive debugger that makes traces navigable and useful during real development.
3. A debug-info standard so source mapping is based on compiler-generated metadata.
4. And optionally, a metadata registry so source-aware debugging works outside a single local checkout.

This approach fits Canton’s architecture. It starts with participant-visible data, keeps privacy boundaries explicit, reuses DPM and Ledger APIs, and only proposes compiler work where the current metadata is not enough.

Walnut is well suited for this work. Our team has four years of experience building blockchain debugging and observability tooling, including:

- Starknet Debugger, a multi-year partnership with StarkWare covering debug-info generation, transaction simulation, contract verification, network forking, and hosted debugger UI.
- ETHDebug work in the Solidity compiler ecosystem, helping standardize debug information for EVM smart contracts.
- StylusDB, a CLI debugger for Arbitrum Stylus and cross-VM Solidity/Rust debugging workflows.

The end result is not just a nicer CLI. It is a shared debugging substrate that can later support a full web UI, richer replay, profiling, coverage, and more advanced Canton development workflows.

---

## References

- JSON Ledger API OpenAPI: https://docs.digitalasset.com/build/3.4/reference/json-api/openapi.html
- Canton package management and vetting: https://docs.daml.com/canton/usermanual/packagemanagement.html#understanding-package-vetting
- ETHDebug format: https://ethdebug.github.io/format/
