# Development Fund Proposal Draft: Versioned Debug Info Metadata for Daml

**Author:** Walnut
**Status:** Draft / follow-on proposal
**Created:** 2026-07-01
**Label:** daml-tooling
**Champion:** Need Champion

---

## Abstract

Walnut proposes to define, prototype, and validate a versioned debug metadata
format for Daml packages.

The goal is to give Daml/Canton developer tools a stable, portable way to map a
compiled package back to source files, source spans, templates, choices,
interfaces, expressions, and value locations. The first consumers are trace,
visualization, test-reporting, coverage, and source diagnostics tools, including
the already-submitted `dpm trace` visualization proposal.

This is not a request to expose private participant data or to change Canton
transaction visibility. The metadata describes code, not ledger state. Tools
still operate through authorized participant APIs and only show values visible
to the requesting party context.

---

## Relationship to the Submitted `dpm trace` Proposal

Walnut has already submitted the Development Fund proposal
`DPM Trace Transaction Visualization`. That proposal is intentionally scoped to
transaction inspection, visualization, prepared transaction comparison, failed
completion diagnostics, and adoption. It uses existing Ledger API, JSON Ledger
API, completion, package, and local project metadata. It does **not** require
Daml compiler, Daml-LF interpreter, Canton node, or protocol changes.

This proposal is the follow-on that the `dpm trace` proposal identifies under
"Compiler Source Metadata".

The boundary is:

- `dpm trace` proves the user workflow and gives the ecosystem a useful first
  source-aware consumer.
- This proposal defines the metadata standard and compiler/tooling support that
  makes source-aware consumers reliable instead of heuristic.

Without this metadata, tools can still show useful participant-visible
transaction trees, but source-level explanations depend on local file discovery,
`damlc inspect`, and text matching. That is good enough for a proof of concept
and for early adoption. It is not enough for a durable Canton debugging,
profiling, test, coverage, and explorer ecosystem.

---

## Specification

### 1. Objective

Create a versioned Daml debug metadata standard and working reference
implementation that can be emitted by Daml tooling and consumed by ecosystem
tools.

The first version should answer these questions for any package:

- Which source files produced this package?
- Which source hashes prove those files match the compiled package?
- Which modules, templates, interfaces, exceptions, choices, controllers,
  observers, signatories, keys, and expressions correspond to which source spans?
- Which Daml-LF package/module/entity references correspond to those source
  spans?
- Which values can a tool reasonably associate with those spans from ordinary
  transaction/update data?
- Which values require interpreter/runtime support and therefore must not be
  claimed by trace-only tools?
- How can tools detect schema compatibility across SDK versions?

The output should be a public, versioned schema plus reference tooling, not a
Walnut-private format.

### 2. Motivation

Daml and Canton already expose important primitives:

- Canton participants expose authorized, participant-scoped updates and
  completions.
- DARs and DALFs identify the packages being interpreted.
- Daml-LF carries source location information.
- `damlc inspect`, project metadata, package manifests, and local source trees
  can provide useful source hints.

Those primitives are not yet a stable debug metadata contract.

Tool builders currently have to reconstruct intent by combining package ids,
local paths, source spans, text search, Daml Script output, error strings, and
transaction trees. That has several failure modes:

- absolute local paths leak into artifacts or reports;
- source matching fails when package source is not local;
- many identical assertion strings produce ambiguous locations;
- expression-level replay is easy to overstate because transaction data does not
  include arbitrary local interpreter state;
- tooling cannot reliably compare metadata across SDK versions;
- every debugger, test reporter, profiler, coverage tool, and explorer invents
  its own source map.

Mature ecosystems solve this with explicit debug metadata formats: DWARF for
native toolchains, PDB/CodeView on Windows, source maps for JavaScript, and
ETHDebug-style metadata in EVM tooling. Daml needs the same kind of durable
contract, adapted to Canton privacy and Daml-LF semantics.

### 3. Implementation Mechanics

The proposal has four technical workstreams.

#### A. Debug Metadata Schema

Define `daml-debug-info/v1` as a versioned schema. The concrete encoding should
be decided with Daml/Canton maintainers, but the initial reference should be
human-readable JSON with a JSON Schema validator. A future protobuf encoding can
be added without changing the conceptual model.

The schema should include:

- `schema`: stable schema identifier, e.g. `daml-debug-info/v1`.
- `producer`: compiler/tool name, version, build mode, feature flags.
- `package`: package id, package name/version where available, Daml-LF version,
  SDK/compiler version, package hash, and optional DAR hash.
- `sources`: package-relative source URIs, normalized display paths, SHA-256
  content hashes, optional redacted original paths, and source-root identity.
- `spans`: source id, start/end line/column, span kind, and optional parent span.
- `symbols`: modules, templates, interfaces, exceptions, choices, methods,
  data constructors, fields, keys, controllers, observers, signatories, and
  other user-visible definitions.
- `lfRefs`: Daml-LF references for package/module/entity/value-level constructs
  where the compiler can identify them.
- `steps`: optional source-level evaluation step descriptors for tool display,
  not runtime values.
- `valueSlots`: value locations that a tool may populate from visible
  transaction data, such as contract payload fields, choice arguments, choice
  results, created payloads, exercise actors, signatories, observers, and keys.
- `availability`: classification for each value slot, such as
  `transaction-visible`, `source-only`, `interpreter-only`, or `not-tracked`.
- `compatibility`: minimum consumer version, feature flags, and forward-
  compatibility rules.

Example shape:

```json
{
  "schema": "daml-debug-info/v1",
  "producer": {
    "tool": "damlc",
    "version": "3.x",
    "features": ["source-spans", "value-slots"]
  },
  "package": {
    "packageId": "<package-id>",
    "name": "asset-demo",
    "version": "1.0.0",
    "lfVersion": "<lf-version>"
  },
  "sources": [
    {
      "id": "src:Asset",
      "uri": "daml://asset-demo/daml/Asset.daml",
      "path": "daml/Asset.daml",
      "sha256": "<source-sha256>"
    }
  ],
  "symbols": [
    {
      "id": "sym:Asset:Asset:Transfer",
      "kind": "choice",
      "module": "Asset",
      "template": "Asset",
      "choice": "Transfer",
      "span": "span:transfer-choice",
      "lfRef": {
        "packageId": "<package-id>",
        "module": "Asset",
        "entity": "Asset",
        "choice": "Transfer"
      }
    }
  ],
  "spans": [
    {
      "id": "span:transfer-choice",
      "source": "src:Asset",
      "kind": "choice-definition",
      "start": {"line": 24, "column": 3},
      "end": {"line": 39, "column": 20}
    }
  ],
  "valueSlots": [
    {
      "id": "slot:transfer-argument:newOwner",
      "symbol": "sym:Asset:Asset:Transfer",
      "name": "newOwner",
      "kind": "choice-argument-field",
      "availability": "transaction-visible"
    }
  ]
}
```

The exact schema should be reviewed with Daml/Canton maintainers before it is
declared stable. The first implementation can be marked experimental while the
ecosystem validates the model.

#### B. Compiler and Packaging Prototype

Prototype debug metadata emission in Daml tooling.

The preferred first step is additive:

- no Daml-LF semantic change;
- no Canton protocol change;
- no participant node change;
- no mandatory runtime overhead.

Candidate emission modes:

```bash
damlc build --debug-info-json <file>
damlc build --debug-info-dir <dir>
damlc inspect --debug-info-json <dar-or-dalf>
```

Candidate packaging modes:

- sidecar artifact: `<package-id>.daml-debug-info.json`;
- optional DAR member ignored by existing tools, e.g.
  `META-INF/daml-debug-info/<package-id>.json`;
- both, with identical content hashes and validation rules.

The prototype should normalize source paths to package-relative URIs by default.
Absolute local paths must not be emitted unless an explicit developer-only flag
is added, and those paths must be marked non-portable.

#### C. Reader, Validator, and Test Corpus

Build a small reference reader/validator package and CLI. It should:

- validate schema version and feature flags;
- verify package id and package hash where available;
- verify source hashes when sources are present;
- check span bounds against source files;
- check that symbol ids and LF references are internally consistent;
- produce useful diagnostics for stale or mismatched metadata;
- reject or warn on non-portable absolute paths by default.

The test corpus should include representative Daml packages:

- simple create and consuming exercise;
- nested exercise with child creates;
- choices with controllers, observers, and signatories;
- interfaces and interface choices where available;
- contract keys;
- assertions and aborts used by failed-completion diagnostics;
- multi-module packages;
- dependencies across multiple packages.

#### D. First Consumer Integration

Integrate the metadata into `dpm trace` as the first end-to-end consumer.

The integration should improve:

- source diagnostics for failed completions;
- transaction tree source links;
- prepared-vs-committed comparison source links;
- Daml Script test reports;
- expression/source step display with explicit value availability labels.

This integration must preserve Canton's privacy model. If the participant-visible
transaction does not contain a value, `dpm trace` must not invent it from source
metadata. It may show the source span and explain that the value is
`interpreter-only` or `not transaction-visible`.

### 4. Architectural Alignment

This proposal aligns with Canton architecture in these ways:

- It respects participant-scoped visibility. Debug metadata maps code to source;
  it does not reveal ledger data outside the requesting party's rights.
- It is package-oriented, matching Daml-LF and DAR package distribution.
- It complements package upload and package vetting. Vetting says a participant
  is willing to process a package; debug metadata says which source and spans
  correspond to a package.
- It improves DPM/Daml developer workflows without changing Canton consensus,
  synchronization, privacy, or transaction semantics.
- It gives multiple tools a shared contract instead of locking the ecosystem to
  one debugger or one vendor's trace format.

The proposal directly supports the Development Fund's developer-experience and
shared tooling priorities.

### 5. Backward Compatibility

The first version is additive.

- Existing Daml applications continue to compile and run unchanged.
- Existing DAR consumers should ignore optional metadata files they do not
  understand.
- Sidecar metadata is optional.
- Tools that do not understand `daml-debug-info/v1` continue to use existing
  package and source mechanisms.
- Schema evolution is versioned. Consumers must ignore unknown compatible fields
  and reject unsupported major versions.

---

## Non-Goals

This proposal does not include:

- a hosted source registry or Sourcify-style verification service;
- a VS Code extension;
- a full DAP-compatible debugger;
- live breakpoints in Canton participants;
- Daml-LF interpreter changes for arbitrary local variable capture;
- any mechanism to bypass participant visibility;
- any claim that failed submissions have update ids.

Those are possible future proposals once the metadata contract is accepted.

---

## Milestones and Deliverables

### Milestone 1: Maintainer-Aligned Schema Draft

**Estimated Delivery:** 4 weeks from start
**Focus:** Turn the concept into a reviewable Daml debug metadata specification.

**Deliverables / Value Metrics:**

- Public `daml-debug-info/v1` draft specification.
- JSON Schema or equivalent machine-checkable schema.
- Compatibility and versioning rules.
- Path-hygiene rules for portable source paths and hashes.
- Value availability model explaining what trace tools can and cannot populate.
- Review session with Daml/Canton maintainers and at least one ecosystem tooling
  consumer.

**Acceptance Criteria:**

- A Daml/Canton maintainer or delegated reviewer confirms that the schema is
  plausible for compiler emission.
- The schema can represent at least five representative Daml constructs:
  template, choice, choice argument, contract payload field, and failed assertion
  source span.
- The schema explicitly distinguishes transaction-visible values from
  interpreter-only values.

### Milestone 2: Compiler Emission Prototype

**Estimated Delivery:** 6 weeks after Milestone 1 acceptance
**Focus:** Emit real debug metadata from real Daml packages.

**Deliverables / Value Metrics:**

- Prototype `damlc` or adjacent compiler-tool emission path.
- Sidecar JSON output for at least one package.
- Optional DAR embedding prototype if maintainers agree it is appropriate.
- Source path normalization and source hash generation.
- Mapping for modules, templates, choices, fields, and core expression spans.
- Documentation explaining how to generate metadata.

**Acceptance Criteria:**

- A developer can compile a representative package and produce a
  `daml-debug-info/v1` artifact.
- The artifact includes package id, source hashes, source spans, symbols, and LF
  references for the representative package.
- The artifact does not contain local absolute paths by default.
- Existing package compilation remains unaffected when debug metadata is not
  requested.

### Milestone 3: Reader, Validator, and Corpus

**Estimated Delivery:** 4 weeks after Milestone 2 acceptance
**Focus:** Make the metadata consumable by independent tools.

**Deliverables / Value Metrics:**

- Open-source reader/validator library or CLI.
- Validation diagnostics for stale source hashes, span mismatches, unsupported
  schema versions, and non-portable paths.
- Test corpus covering representative Daml constructs.
- Compatibility tests for forward-compatible unknown fields.
- Documentation for tool authors.

**Acceptance Criteria:**

- The validator accepts all valid corpus artifacts and rejects intentionally
  corrupted artifacts.
- A tool author can load the metadata and resolve a package/module/template/
  choice reference to a source span without using Walnut-specific code.
- The corpus includes at least one multi-package example.

### Milestone 4: `dpm trace` Consumer Integration and Ecosystem Validation

**Estimated Delivery:** 5 weeks after Milestone 3 acceptance
**Focus:** Prove the standard improves a real Canton developer workflow.

**Deliverables / Value Metrics:**

- `dpm trace` support for loading `daml-debug-info/v1` sidecar or DAR-embedded
  metadata.
- Source-linked transaction, failed-completion, compare, and test-report output
  using the metadata.
- Clear UI labels for transaction-visible, source-only, and interpreter-only
  values.
- Example package and walkthrough connecting compiler emission to trace output.
- Feedback report from at least three Canton developers or organizations that
  test the metadata-backed workflow.

**Acceptance Criteria:**

- `dpm trace` resolves source diagnostics for a failed completion without relying
  on text-search-only heuristics when metadata is available.
- `dpm trace` correctly refuses to show local variable values that are not
  present in participant-visible transaction data.
- At least two independent testers confirm that metadata-backed source links are
  more reliable than the pre-metadata workflow.
- Follow-up issues are filed for any compiler/runtime hooks needed for a future
  true debugger.

---

## Acceptance Criteria

The Tech & Ops Committee can evaluate completion based on:

- a public, versioned metadata specification;
- a working compiler/tooling prototype that emits metadata for real packages;
- a reader/validator that independent tools can use;
- a representative Daml test corpus;
- a `dpm trace` integration demonstrating concrete developer value;
- clear privacy and value-availability semantics;
- maintainer review of the schema and emission approach;
- independent ecosystem feedback.

The value to the ecosystem is not the existence of a JSON file. The value is that
source-aware Daml/Canton tools can share one stable metadata contract instead of
building incompatible heuristics.

---

## Funding

**Draft Total Funding Request:** 1,600,000 Canton Coin (CC)

This amount should be reviewed with the Champion and Tech & Ops Committee before
formal submission, but the proposal is structured so each payment is tied to a
reviewable ecosystem asset rather than an internal activity.

### Payment Breakdown by Milestone

- Milestone 1, Maintainer-Aligned Schema Draft: 250,000 CC upon committee
  acceptance.
- Milestone 2, Compiler Emission Prototype: 500,000 CC upon committee
  acceptance.
- Milestone 3, Reader, Validator, and Corpus: 350,000 CC upon committee
  acceptance.
- Milestone 4, `dpm trace` Consumer Integration and Ecosystem Validation:
  500,000 CC upon committee acceptance and ecosystem validation.

### Volatility Stipulation

The expected duration is under six months if maintainer review is available
during the design phase. If the scope expands into interpreter/runtime hooks,
that should be split into a separate proposal rather than extending this one.

Should the project timeline extend beyond six months due to Committee-requested
scope changes, remaining milestones should be renegotiated to account for
significant USD/CC price volatility.

---

## Co-Marketing

Walnut can collaborate with the Canton Foundation and Daml/Canton maintainers on:

- a technical post explaining Daml debug metadata and Canton privacy boundaries;
- a demo showing metadata-backed `dpm trace` source diagnostics;
- documentation for ecosystem tool authors;
- a follow-up discussion on future debugger/runtime hooks.

---

## Rationale

### Why start with metadata instead of a full debugger?

A full source-level debugger needs runtime/interpreter hooks, breakpoint
semantics, value capture policy, and careful privacy review. Metadata is the
lower-risk foundation that every later tool needs anyway. It improves trace,
test, coverage, profiling, and explorer workflows immediately while giving the
ecosystem a precise way to discuss future runtime hooks.

### Why use package-relative paths and hashes?

Daml packages move across machines, CI systems, validators, and organizations.
Absolute local paths are not portable and can leak private machine details.
Package-relative paths plus source hashes let tools verify source identity
without committing to a developer's local filesystem layout.

### Why integrate with `dpm trace` first?

`dpm trace` is the submitted, concrete consumer that already exercises the hard
parts: participant-scoped updates, failed completions, prepared transactions,
test reports, and source diagnostics. Integrating metadata there validates the
schema against real workflows without requiring a new hosted product or IDE.

### Why not put this only in `damlc inspect`?

`damlc inspect` is useful, but a durable ecosystem contract should be an
artifact that tools can carry, validate, cache, and consume without invoking a
compiler every time. `damlc inspect` can still be one way to view or derive the
metadata.

### Why not make this a source registry proposal?

A registry may be useful later, especially for packages a developer did not
build locally. It depends on a stable package/source metadata format. This
proposal creates that foundation first.

---

## Risks and Mitigations

- **Compiler mapping complexity:** start with modules, templates, choices,
  fields, and core spans before attempting every expression form.
- **Overstating runtime values:** require explicit `availability` labels and make
  consumers display missing interpreter-only values honestly.
- **Maintainer alignment risk:** make maintainer review the first milestone and
  do not freeze the schema before that review.
- **SDK evolution:** use major-versioned schemas, feature flags, and forward-
  compatible parsing rules.
- **Path leaks:** default to package-relative paths and hashes; reject local
  absolute paths in validator strict mode.
- **Scope creep into debugger/runtime hooks:** document runtime-hook findings as
  follow-up work, not as a blocker for the metadata standard.

---

## Maintenance

The specification should live in a Foundation or Daml/Canton-maintainer-approved
public location once accepted. Reference tooling should be open source under
Apache-2.0 unless maintainers request a different standard license. Walnut can
maintain the `dpm trace` consumer integration. Long-term compiler emission
maintenance should follow the ownership model agreed with Daml maintainers.

---

## About the Team

Walnut builds blockchain debugging and observability tooling. The team has
experience with compiler debug information, source mapping, transaction
visualization, simulation, contract verification, and debugger UX across
ecosystems including Starknet, Ethereum/Solidity, and Arbitrum Stylus.

That experience is directly relevant here: Daml needs debug metadata that is
specific to Daml-LF and Canton privacy, not a generic copy of EVM or native
debugging formats.

---

## References

- Submitted consumer proposal: `DPM Trace Transaction Visualization`
- `dpm trace` proof of concept: <https://github.com/walnuthq/dpm-trace>
- Initial Walnut Daml debug-info branch:
  <https://github.com/walnuthq/daml/tree/feature/debug-info>
