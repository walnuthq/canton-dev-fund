# Development Fund Proposal: Versioned Debug Info Metadata for Daml

**Author:** [Walnut](https://walnut.dev)<br>
**Status:** Draft<br>
**Created:** 2026-08-15<br>
**Label:** daml-tooling<br>
**Champion:** Need Champion<br>

---

## Abstract

Walnut proposes to define, prototype, upstream, and validate a versioned debug
metadata format for Daml packages.

The goal is to give Daml/Canton developer tools a stable, portable way to map a
compiled package back to source files, source spans, templates, choices,
interfaces, expressions, failure sites, and value locations. The first
consumers are trace, visualization, test-reporting, coverage, and source
diagnostics tools, starting with the approved
[DPM Trace Transaction Visualization](./dpm-trace-visualization.md)
proposal, which names this work as its planned "Compiler Source Metadata"
follow-on.

This is not a request to expose private participant data or to change Canton
transaction visibility. The metadata describes code, not ledger state. Tools
still operate through authorized participant APIs and only show values visible
to the requesting party context.

The concrete first version of the format, `daml-debug-info/v1`, is specified
in [Appendix A](#appendix-a-daml-debug-infov1-draft-specification) of this
document and is backed by a working reference implementation.

---

## Specification

### 1. Objective

Create a versioned Daml debug metadata standard, upstream the emission paths
that produce it, and validate it end to end with independent consumers.

The first version targets Daml SDK 3.x packages (Daml-LF 2.1 and later) and
answers these questions for any package:

- Which source files produced this package?
- Which source hashes prove those files match the compiled package?
- Which modules, templates, interfaces, exceptions, choices, controllers,
  observers, signatories, keys, and expressions correspond to which source
  spans?
- Which Daml-LF package, module, and entity references correspond to those
  source spans?
- Which values can a tool reasonably associate with those spans from ordinary
  transaction and update data?
- Which values require interpreter or runtime support and therefore must not
  be claimed by trace-only tools?
- Which failure sites (assertions, aborts, `failWithStatus` calls) exist in
  the package, and how can a failed completion be joined back to one of them?
- How can tools detect schema compatibility across SDK versions?

The output is a public, versioned schema plus reference tooling, not a
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

- Absolute local paths leak into artifacts or reports.
- Source matching fails when package source is not local.
- Many identical assertion strings produce ambiguous locations.
- Expression-level replay is easy to overstate, because transaction data does
  not include arbitrary local interpreter state.
- Tooling cannot reliably compare metadata across SDK versions.
- Every debugger, test reporter, profiler, coverage tool, and explorer invents
  its own source map.

Mature ecosystems solve this with explicit debug metadata formats: DWARF for
native toolchains, PDB/CodeView on Windows, source maps for JavaScript, and
ETHDebug-style metadata in EVM tooling. Daml needs the same kind of durable
contract, adapted to Canton privacy and Daml-LF semantics.

### 3. Implementation Mechanics

The proposal has five technical workstreams.

#### A. Debug Metadata Schema

Define `daml-debug-info/v1` as a versioned schema. The concrete encoding is
human-readable JSON with a machine-checkable JSON Schema validator. A future
protobuf encoding can be added without changing the conceptual model.

The schema includes:

- `schema`: stable schema identifier (`daml-debug-info/v1`).
- `producer`: compiler or tool name, version, build mode, and feature flags.
- `package`: package id, package name and version, Daml-LF version, and SDK
  version.
- `sources`: package-relative source paths, SHA-256 content hashes, and the
  module-to-file mapping, plus an explicit record of modules whose sources
  could not be mapped.
- `spans`: source spans for definitions and located expressions.
- `symbols`: modules, templates, interfaces, exceptions, choices, methods,
  data types, and values, each carrying its Daml-LF reference where one
  exists.
- `valueSlots`: value locations a tool may try to populate, each labeled with
  a normative availability class (`transaction-visible` or
  `interpreter-only`).
- `failureSites`: assertion, abort, precondition, and `failWithStatus` sites
  with their spans and statically known messages or error ids, so failed
  completions can be joined to source without text search.
- `steps`: deterministic source-level step descriptors for tool display.

The complete concrete specification, including the producer invariants,
position semantics, and validation rules, is
[Appendix A](#appendix-a-daml-debug-infov1-draft-specification). The schema
will be reviewed with Daml/Canton maintainers before it is declared stable,
and the first implementation is marked experimental while the ecosystem
validates the model.

#### B. Compiler Emission

Upstream debug metadata emission in the Daml compiler.

The emission is strictly additive:

- No Daml-LF semantic change.
- No Canton protocol change.
- No participant node change.
- No mandatory runtime overhead.

The central producer invariant, stated normatively in Appendix A.2: enabling
debug metadata emission MUST NOT change the compiled Daml-LF output. The
package id of a package built with and without the emission flag MUST be
identical, so metadata generated in a debug build applies to the exact
artifact that is deployed. The upstream contribution includes a CI test that
asserts this package-id equality.

Emission modes:

```bash
daml build --experimental-debug-info
```

Packaging modes, specified in Appendix A.1:

- Sidecar artifact next to the DAR.
- Optional DAR member under `META-INF/daml-debug-info/`, ignored by existing
  tools.
- Both copies byte-identical.

The producer normalizes source paths to package-relative paths. Absolute
local paths are never emitted.

Two small compiler fixes from the reference implementation (Appendix A.11)
are prerequisites for choice-level debugging and improve stack traces for all
Daml users independently of this proposal. Walnut will submit them to
`digital-asset/daml` as standalone pull requests ahead of this proposal's
review, together with measurements of DALF size and interpreter overhead.

#### C. Runtime Debug Trace

Extend the Daml Script runner with an opt-in runtime debug trace:

```bash
daml script --debug-trace-file <file> ...
```

The runner writes one JSON object per line (JSONL) describing script
lifecycle, script questions with call-stack locations, submissions, and
ledger events observed by the run, using the same position conventions as the
metadata so events join against spans and steps. The concrete contract is
Appendix A.9.

This workstream covers the Daml Script layer only. Expression-level stepping
inside choice bodies needs one additional hook in the Speedy interpreter. That
hook is out of scope here: Appendix A.10 reserves its shape so a follow-up
proposal can add it without changing the trace contract.

#### D. Reader, Validator, and Test Corpus

Build a reference reader and validator package and CLI. It validates schema
version and feature flags, verifies package id and source hashes, checks span
bounds and internal reference consistency, checks that value-slot
availability labels conform to the normative table in Appendix A.6, verifies
that sidecar and DAR-member copies are byte-identical, produces a specific
diagnostic for line-ending mismatches, and rejects non-portable absolute
paths.

The test corpus includes representative Daml packages:

- Simple create and consuming exercise.
- Nested exercise with child creates.
- Choices with controllers, observers, and signatories.
- Interfaces and interface choices.
- Contract keys where the target LF version supports them.
- Assertions, aborts, and `failWithStatus` sites used by failed-completion
  diagnostics.
- Multi-module and multi-package examples.
- A package upgrade example with two versions of the same package, to
  exercise package-id-keyed resolution.

#### E. Consumer Integration: `dpm trace` and `dpm debug`

Integrate the metadata into two consumers:

- `dpm trace` (approved proposal): source diagnostics for failed completions
  driven by `failureSites`, transaction tree source links, source links for
  comparing prepared with committed transactions, and Daml Script test
  reports.
- `dpm debug`: source-level stepping over the runtime debug trace from
  workstream C, with explicit value availability labels.

This integration must preserve Canton's privacy model. If the
participant-visible transaction does not contain a value, the tools must not
invent it from source metadata. They may show the source span and explain
that the value is `interpreter-only`.

### 4. Architectural Alignment

This proposal aligns with Canton architecture in these ways:

- It respects participant-scoped visibility. Debug metadata maps code to
  source. It does not reveal ledger data outside the requesting party's
  rights.
- It is package-oriented, matching Daml-LF and DAR package distribution, and
  it behaves correctly under smart contract upgrades: metadata is keyed by
  package id, and consumers resolve it strictly by package id, never by
  package name. Under package-name resolution a participant may execute a
  different package version than the developer expects, which is exactly the
  situation where precise per-package metadata matters most.
- The metadata trust model is explicit. Debug metadata is advisory: nothing
  cryptographically binds a sidecar or DAR member to a DALF, because
  `META-INF` members are not part of the package id. For packages a developer
  did not build, the only strong verification is to rebuild from source and
  compare package ids. Consumers treat metadata as trusted only as much as
  its distribution channel.
- The ledger never transports debug metadata. Canton distributes DALFs, not
  DAR members, so counterparties cannot receive a package's debug info
  through the ledger. This is good for privacy, and it is why a future
  source and metadata registry is the natural follow-on for third-party
  packages.
- It complements package upload and package vetting. Vetting says a
  participant is willing to process a package. Debug metadata says which
  source and spans correspond to a package.
- It improves DPM and Daml developer workflows without changing Canton
  consensus, synchronization, privacy, or transaction semantics.
- It gives multiple tools a shared contract instead of locking the ecosystem
  to one debugger or one vendor's trace format.

The proposal directly supports the Development Fund's developer-experience
and shared tooling priorities.

### 5. Backward Compatibility

The first version is additive.

- Existing Daml applications continue to compile and run unchanged.
- Existing DAR consumers ignore optional metadata files they do not
  understand.
- Embedding the metadata changes the DAR file hash (it changes the zip
  bytes) but never the package id. Workflows that pin DAR file hashes must
  pin the debug-enabled DAR.
- Sidecar metadata is optional.
- Tools that do not understand `daml-debug-info/v1` continue to use existing
  package and source mechanisms.
- Schema evolution is versioned. Consumers must ignore unknown fields in a
  supported major version and reject unsupported major versions.

---

## Non-Goals

This proposal does not include:

- A hosted source registry or Sourcify-style verification service.
- A VS Code extension.
- A full DAP-compatible debugger.
- Live breakpoints in Canton participants.
- Daml-LF interpreter changes. The Speedy location hook needed for
  expression-level stepping is reserved in Appendix A.10 and left to a
  follow-up proposal.
- Any mechanism to bypass participant visibility.
- Any claim that failed submissions have update ids.

Those are possible future proposals once the metadata contract is accepted.

---

## Milestones and Deliverables

A working proof of concept already exists (see Reference Implementation
Status below). The funded work is therefore not "make it work once": it is
maintainer alignment, upstreaming, hardening, and independent validation.
Milestone acceptance criteria are deliberately set beyond what the proof of
concept already demonstrates.

### Milestone 1: Maintainer-Aligned Schema and Upstream Groundwork

**Estimated Delivery:** 4 weeks from start<br>
**Focus:** Turn the draft into a maintainer-reviewed Daml debug metadata
specification, and open the upstream conversation with running code.

**Deliverables / Value Metrics:**

- Public `daml-debug-info/v1` specification published in a location agreed
  with Daml/Canton maintainers (Appendix A is the starting draft).
- Machine-checkable JSON Schema for the format.
- Compatibility and versioning rules, position semantics, and path-hygiene
  rules.
- Normative value availability table distinguishing transaction-visible from
  interpreter-only values (Appendix A.6).
- `failureSites` definition for failed-completion diagnostics (Appendix A.7).
- The two compiler location fixes from the reference implementation submitted
  to `digital-asset/daml` as standalone pull requests, with DALF size and
  interpreter overhead measurements on a large public package.
- Review session with Daml/Canton maintainers and at least one ecosystem
  tooling consumer.

**Acceptance Criteria:**

- A Daml/Canton maintainer or delegated reviewer confirms that the schema is
  plausible for compiler emission.
- The schema represents at least these representative Daml constructs:
  template, choice, choice argument, contract payload field, and failed
  assertion source span.
- The schema explicitly distinguishes transaction-visible values from
  interpreter-only values, per slot kind.
- The two standalone compiler fix pull requests are open upstream with
  maintainer review engagement.

### Milestone 2: Compiler and Runtime Emission Upstreaming

**Estimated Delivery:** 6 weeks after Milestone 1 acceptance<br>
**Focus:** Land upstream-quality emission of the metadata and the runtime
debug trace.

**Deliverables / Value Metrics:**

- Metadata emission path submitted to `digital-asset/daml` behind an
  experimental flag, or through an emission path maintainers prefer.
- A CI test asserting that the package id of a package built with and
  without the debug flag is identical (the producer invariant of Appendix
  A.2).
- Runtime debug trace emission in the Daml Script runner (workstream C)
  submitted the same way.
- Source path normalization, source hash generation, and the
  `unmappedModules` record.
- `failureSites` emission for abort, assertion, precondition, and
  `failWithStatus` sites.
- Mapping for modules, templates, choices, fields, and core expression
  spans.
- Documentation explaining how to generate metadata and traces.

**Acceptance Criteria:**

- A developer can compile a representative package with the upstream-submitted
  emission path and produce a `daml-debug-info/v1` artifact containing
  package id, source hashes, source spans, symbols, LF references, value
  slots, failure sites, and steps.
- The package-id invariance test passes in CI.
- The artifact contains no absolute local paths.
- Existing package compilation is unaffected when debug metadata is not
  requested.
- The upstream pull requests are under active maintainer review, or, if
  upstreaming is declined, a maintainer-agreed alternative distribution path
  for the emission tooling is documented and delivered.

### Milestone 3: Reader, Validator, and Corpus

**Estimated Delivery:** 4 weeks after Milestone 2 acceptance<br>
**Focus:** Make the metadata consumable and checkable by independent tools.

**Deliverables / Value Metrics:**

- Open-source reader and validator library and CLI.
- Validation diagnostics for stale source hashes (including a specific
  line-ending mismatch diagnostic), span mismatches, unsupported schema
  versions, availability-label conformance, sidecar/DAR-member divergence,
  and non-portable paths.
- Test corpus covering the representative constructs listed in workstream D,
  including a multi-package example and a package upgrade example.
- Compatibility tests for forward-compatible unknown fields.
- Documentation for tool authors.

**Acceptance Criteria:**

- The validator accepts all valid corpus artifacts and rejects intentionally
  corrupted artifacts.
- A tool author can load the metadata and resolve a package, module,
  template, or choice reference to a source span without using
  Walnut-specific code.
- The corpus includes at least one multi-package example and one upgrade
  example with two versions of the same package.

### Milestone 4: Consumer Integration, Adoption, and Ecosystem Validation

**Estimated Delivery:** 5 weeks after Milestone 3 acceptance<br>
**Focus:** Prove the standard improves real Canton developer workflows.

**Deliverables / Value Metrics:**

- `dpm trace` support for loading `daml-debug-info/v1` sidecar or
  DAR-embedded metadata.
- Failed-completion source diagnostics in `dpm trace` driven by
  `failureSites`, with clearly labeled match quality (exact error id, exact
  static message, or degraded heuristic).
- Source-linked transaction, compare, and test-report output using the
  metadata.
- `dpm debug` source-level stepping over the workstream C runtime trace,
  with explicit labels for transaction-visible and interpreter-only values.
- Example package and walkthrough connecting compiler emission, runtime
  trace, and consumer output.
- Feedback report from at least three Canton developers or organizations
  that test the metadata-backed workflow.
- Follow-up issues filed for the Speedy interpreter location hook and any
  other runtime hooks needed for a future true debugger.

**Acceptance Criteria:**

- `dpm trace` resolves source diagnostics for a failed completion without
  relying on text-search-only heuristics when metadata with `failureSites`
  is available.
- `dpm trace` and `dpm debug` correctly refuse to show values that are not
  present in participant-visible data or the local trace, and label them as
  interpreter-only instead.
- At least two independent testers confirm that metadata-backed source links
  are more reliable than the pre-metadata workflow.

---

## Acceptance Criteria

The Tech & Ops Committee can evaluate completion based on:

- A public, versioned metadata specification with a machine-checkable
  schema.
- Emission paths for metadata and runtime traces submitted upstream, with
  the package-id invariance guarantee tested in CI.
- A reader and validator that independent tools can use.
- A representative Daml test corpus, including an upgrade example.
- `dpm trace` and `dpm debug` integrations demonstrating concrete developer
  value.
- Clear privacy and value-availability semantics.
- Maintainer review of the schema and emission approach.
- Independent ecosystem feedback.

The value to the ecosystem is not the existence of a JSON file. The value is
that source-aware Daml/Canton tools can share one stable metadata contract
instead of building incompatible heuristics.

---

## Funding

**Total Funding Request:** 1,600,000 Canton Coin (CC)

Each payment is tied to a reviewable ecosystem asset rather than an internal
activity.

### Payment Breakdown by Milestone

- Milestone 1, Maintainer-Aligned Schema and Upstream Groundwork: 250,000 CC
  upon committee acceptance.
- Milestone 2, Compiler and Runtime Emission Upstreaming: 500,000 CC upon
  committee acceptance.
- Milestone 3, Reader, Validator, and Corpus: 350,000 CC upon committee
  acceptance.
- Milestone 4, Consumer Integration, Adoption, and Ecosystem Validation:
  500,000 CC upon committee acceptance and ecosystem validation.

### Volatility Stipulation

The expected duration is under six months if maintainer review is available
during the design phase. If the scope expands into interpreter or runtime
hooks, that will be split into a separate proposal rather than extending this
one.

Should the project timeline extend beyond six months due to
Committee-requested scope changes, remaining milestones must be renegotiated
to account for significant USD/CC price volatility.

---

## Co-Marketing

Walnut can collaborate with the Canton Foundation and Daml/Canton maintainers
on:

- A technical post explaining Daml debug metadata and Canton privacy
  boundaries.
- A demo showing metadata-backed `dpm trace` failed-completion diagnostics
  and `dpm debug` stepping.
- Documentation for ecosystem tool authors.
- A follow-up discussion on future debugger and runtime hooks.

---

## Rationale

### Why start with metadata instead of a full debugger?

A full source-level debugger needs runtime and interpreter hooks, breakpoint
semantics, value capture policy, and careful privacy review. Metadata is the
lower-risk foundation that every later tool needs anyway. It improves trace,
test, coverage, profiling, and explorer workflows immediately while giving
the ecosystem a precise way to discuss future runtime hooks.

### Why upstream instead of maintaining a fork?

A debug format only becomes an ecosystem contract if the reference producer
ships with the toolchain developers already use. A forked compiler rots and
fragments the ecosystem. That is why Milestones 1 and 2 anchor acceptance on
upstream submission and maintainer review, and why the two prerequisite
compiler fixes are submitted as standalone pull requests that benefit every
Daml user regardless of this proposal's outcome.

### Why use package-relative paths and hashes?

Daml packages move across machines, CI systems, validators, and
organizations. Absolute local paths are not portable and can leak private
machine details. Package-relative paths plus source hashes let tools verify
source identity without committing to a developer's local filesystem layout.
The specification also defines hashes at the byte level and requires a
specific line-ending mismatch diagnostic, a lesson learned the hard way in
Solidity metadata verification, where Windows checkouts silently fail hash
checks.

### Why a dedicated `failureSites` section?

Failed completions carry a status and a message, not a source location.
Without an indexed table of failure sites, tools must text-search source
files for the message, which is ambiguous whenever messages repeat or are
built dynamically. `failureSites` gives consumers an exact join key when
`failWithStatus` error ids or static messages are used, and an honestly
labeled degraded match otherwise. It also treats `failWithStatus` as the
first-class failure mechanism, matching the direction of Daml 3.x, where
user-defined exceptions are deprecated.

### Why integrate with `dpm trace` first?

`dpm trace` is the approved, concrete consumer that already exercises the
hard parts: participant-scoped updates, failed completions, prepared
transactions, test reports, and source diagnostics. Integrating metadata
there validates the schema against real workflows without requiring a new
hosted product or IDE.

### Why not put this only in `damlc inspect`?

`damlc inspect` is useful, but a durable ecosystem contract should be an
artifact that tools can carry, validate, cache, and consume without invoking
a compiler every time. `damlc inspect` can still be one way to view or derive
the metadata.

### Why not make this a source registry proposal?

A registry may be useful later, especially for packages a developer did not
build locally, and the trust model in section 4 explains why: the ledger
never transports debug metadata, so third-party packages need an out-of-band
distribution and verification channel. A registry depends on a stable
package and source metadata format. This proposal creates that foundation
first.

---

## Risks and Mitigations

- **Compiler mapping complexity:** start with modules, templates, choices,
  fields, and core spans before attempting every expression form.
- **Overstating runtime values:** the normative availability table makes the
  correct classification testable, and consumers must display missing
  interpreter-only values honestly.
- **Maintainer alignment risk:** maintainer review is the first milestone,
  the schema is not frozen before that review, and the standalone compiler
  fixes open the upstream conversation with small, obviously useful changes.
- **Upstreaming declined:** Milestone 2 defines the fallback explicitly: a
  maintainer-agreed alternative distribution path for the emission tooling.
- **SDK evolution:** major-versioned schemas, feature flags, and
  forward-compatible parsing rules.
- **Path and data leaks:** package-relative paths and hashes by default, and
  the validator rejects absolute paths in strict mode. Runtime trace files
  contain ledger data visible to the submitting context and are documented
  as sensitive.
- **Scope creep into debugger and runtime hooks:** runtime-hook findings are
  documented as follow-up work, not as a blocker for the metadata standard.

---

## Maintenance

The specification will live in a Foundation or Daml/Canton-maintainer-approved
public location once accepted. Reference tooling will be open source under
Apache-2.0 unless maintainers request a different standard license. Walnut
will maintain the `dpm trace` and `dpm debug` consumer integrations.
Long-term compiler emission maintenance follows the ownership model agreed
with Daml maintainers, which is a Milestone 1 discussion item.

---

## About the Team

Walnut builds blockchain debugging and observability tooling. The team has
experience with compiler debug information, source mapping, transaction
visualization, simulation, contract verification, and debugger UX across
ecosystems including Starknet, Ethereum/Solidity (debug info generation in
the official Solidity compiler, `solc`), and Arbitrum Stylus.

That experience is directly relevant here: Daml needs debug metadata that is
specific to Daml-LF and Canton privacy, not a generic copy of EVM or native
debugging formats.

---

## Reference Implementation Status

A working reference implementation of the concrete `daml-debug-info/v1`
schema exists and backs Appendix A:

- `daml build --experimental-debug-info`
  (branch `walnuthq/daml@feature/debug-info`) emits the metadata from the
  compiled Daml-LF package (sidecar JSON plus a
  `META-INF/daml-debug-info/<package-id>.json` DAR member), with
  package-relative source paths, SHA-256 source hashes, symbols, spans,
  Daml-LF references, value slots with availability labels, and
  deterministic evaluation steps. Metadata is derived from the compiled
  package, never from textual scanning of sources.
- Two compiler fixes make choice-level debugging possible: choice
  declaration locations are now populated in the Daml-LF AST, and source
  locations are no longer stripped from choice bodies during the GHC to
  Daml-LF conversion (Appendix A.11). Walnut will submit both to
  `digital-asset/daml` as standalone pull requests ahead of this proposal's
  review, together with DALF size and interpreter overhead measurements.
- `daml script --debug-trace-file <file>` (Daml Script runner extension,
  same branch) emits the JSONL runtime debug trace of Appendix A.9.
- `dpm trace` consumes the metadata for source links, and `dpm debug`
  provides source-level stepping over runtime debug traces
  (`walnuthq/dpm-trace`).

Appendix A.13 lists precisely which parts of the specification the reference
implementation emits today and which parts are specified for Milestone 2.
The proof of concept exists to de-risk the proposal. The funded work is the
maintainer-aligned specification, the upstreaming, the validator and corpus,
and the independent ecosystem validation described in the milestones.

---

## References

- Approved consumer proposal:
  [DPM Trace Transaction Visualization](./dpm-trace-visualization.md)
- `dpm trace` proof of concept:
  <https://github.com/walnuthq/dpm-trace>
- Reference implementation branch:
  <https://github.com/walnuthq/daml/tree/feature/debug-info>

---

## Appendix A: `daml-debug-info/v1` draft specification

This appendix is the concrete draft specification of the format, backed by
the reference implementation described above. During Milestone 1 it moves to
a maintainer-agreed public location and gains a machine-checkable JSON
Schema. The key words MUST, MUST NOT, SHOULD, and MAY are used as in RFC
2119. Appendix A.13 records where the current reference implementation still
differs from this text.

### A.1 Artifact placement

A producer emits the metadata twice:

1. **Sidecar:** next to the DAR, named `<dar-basename>.debug-info.json`
   (for example `.daml/dist/asset-demo-1.0.0.debug-info.json`).
2. **DAR member:** `META-INF/daml-debug-info/<package-id>.json`.
   Existing DAR consumers ignore unknown `META-INF` members, so embedding is
   backward compatible. Consumers SHOULD also accept the legacy prototype
   member `META-INF/daml-debug-info.json`.

Rules:

- The two copies MUST be byte-identical. A consumer that reads both and
  finds different bytes MUST treat the metadata as invalid.
- When both are present and identical, the DAR member is authoritative
  (it travels with the artifact). The sidecar exists for convenience.
- v1 producers emit metadata for the main package of the DAR only.
  Dependency packages carry their own metadata in their own DARs.
- Embedding changes the DAR file hash, never the package id.
- Producers SHOULD emit a stable, canonical serialization (UTF-8, LF
  newlines, stable key order for a given producer version), so identical
  inputs produce identical bytes.

### A.2 Top-level document

```json
{
  "schema": "daml-debug-info/v1",
  "producer": { "tool": "damlc", "version": "3.x",
                "buildMode": "experimental",
                "features": ["source-spans", "symbols", "lf-refs",
                             "value-slots", "steps", "failure-sites"] },
  "package": { "packageId": "<hex>", "name": "asset-demo",
               "version": "1.0.0", "lfVersion": "2.1",
               "sdkVersion": "3.x" },
  "sources": [ ... ],
  "unmappedModules": [ ... ],
  "spans": [ ... ],
  "symbols": [ ... ],
  "valueSlots": [ ... ],
  "failureSites": [ ... ],
  "steps": [ ... ],
  "compatibility": { "minConsumerSchema": "daml-debug-info/v1",
                     "ignoreUnknownFields": true }
}
```

**Versioning.** `schema` is the major-versioned identifier. Consumers MUST
reject documents whose major version they do not support and MUST ignore
unknown fields anywhere in a document whose major version they do support.
The `compatibility` object is informative only: producers MAY emit it,
consumers MUST NOT rely on it, and the normative rules are the two sentences
above.

**Producer invariants.**

- Enabling metadata emission MUST NOT change the compiled Daml-LF output.
  The package id of a package built with and without the emission flag MUST
  be identical. This is the property that lets metadata from a debug build
  describe the exact artifact that is deployed.
- Metadata MUST be derived from the compiled Daml-LF package, never from
  textual scanning of sources.

**Positions.** All line and column positions in the document are **1-based**
(Daml-LF stores 0-based source locations, and producers convert). The `end`
line is inclusive. The `end` column is **exclusive**: it points one past the
last character of the span, following the GHC and Daml-LF convention that
the reference implementation preserves. Columns count Unicode code points,
not UTF-8 bytes and not UTF-16 code units. Consumers converting spans to
protocols with different units (for example LSP, whose default is UTF-16)
MUST convert explicitly.

**Supported LF range.** v1 describes Daml-LF 2.1 and later (Daml SDK 3.x).

**Required and optional fields.**

| Object | Required | Optional |
| --- | --- | --- |
| top level | `schema`, `producer`, `package`, `sources`, `spans`, `symbols`, `valueSlots`, `steps` | `unmappedModules`, `failureSites`, `compatibility` |
| `producer` | `tool`, `version`, `buildMode`, `features` | |
| `package` | `packageId`, `name`, `lfVersion`, `sdkVersion` | `version` |
| source | `id`, `module`, `path`, `sha256` | `uri` |
| span | `id`, `source`, `kind`, `start`, `end` | |
| symbol | `id`, `kind`, `module`, `name`, `qualifiedName` | `parent`, `span`, `source`, `lfRef`, `type` |
| value slot | `id`, `symbol`, `kind`, `availability` | `name`, `type` |
| failure site | `id`, `symbol`, `source`, `kind`, `start`, `end` | `message`, `errorId` |
| step | `id`, `symbol`, `index`, `source`, `start`, `end` | |

Arrays MAY be empty. `features` lists the sections the producer emitted, so
consumers can distinguish "not supported by this producer" from "supported
and empty".

### A.3 `sources`

One entry per package module whose source file resolved under the package
source root at build time:

```json
{ "id": "src:Nested.Util", "module": "Nested.Util",
  "uri": "daml://asset-demo/Nested/Util.daml",
  "path": "Nested/Util.daml",
  "sha256": "<content hash>" }
```

- `path` is package-relative (relative to the `source:` root of
  `daml.yaml`). Producers MUST NOT emit absolute local paths.
- `sha256` is the hash of the **exact bytes the compiler read**, with no
  normalization. Consumers verify it before trusting spans (stale-source
  detection). Because a checkout with translated line endings (for example
  git `core.autocrlf` on Windows) has different bytes, consumers SHOULD
  retry a failed verification with newline-normalized content and, on a
  match, report a specific line-ending mismatch diagnostic instead of a
  generic hash failure.
- `module` gives consumers the module-to-file mapping directly (Daml-LF does
  not serialize module source paths).
- `uri` is informative display material only. Its authority component is the
  package name, which is not unique across the ecosystem, so consumers MUST
  NOT use `uri` as a resolution or identity key. `path` plus `sha256` are
  the normative identifiers.
- Modules whose source files did not resolve under the source root (for
  example files included from outside it, or generated modules) are listed
  by module name in the top-level `unmappedModules` array, so consumers can
  distinguish "this module has no mapping" from "this module does not
  exist".

### A.4 `spans`

```json
{ "id": "span:Asset:Asset", "source": "src:Asset",
  "kind": "template-definition",
  "start": {"line": 5, "column": 10}, "end": {"line": 10, "column": 17} }
```

Kinds emitted by the reference producer: `template-definition`,
`choice-definition`, `interface-definition`, `interface-method-definition`,
`exception-definition`, `data-type-definition`, `value-definition`, and
`<slot-kind>-expression` for located value-slot expressions (for example
`signatories-expression`). Spans are only emitted when they provably belong
to the module's own source file. Cross-module inlined spans are dropped
rather than mislabeled.

Position semantics (1-based, end-exclusive columns, code points) are defined
in A.2 and apply to every `start`/`end` pair in the document.

### A.5 `symbols`

```json
{ "id": "sym:Asset:Asset:Transfer", "kind": "choice",
  "module": "Asset", "name": "Transfer",
  "qualifiedName": "Asset:Asset.Transfer",
  "parent": "sym:Asset:Asset", "span": "span:Asset:Asset:Transfer",
  "source": "src:Asset",
  "lfRef": { "packageId": "<pkg>", "module": "Asset",
             "entity": "Asset", "choice": "Transfer" } }
```

- Kinds: `module`, `template`, `choice`, `interface`, `interface-choice`,
  `interface-method`, `exception`, `record`, `variant`, `enum`, `value`.
- `interface-method` and the `interface-method-definition` span kind refer
  to the method **declaration inside the interface definition**. Spans for a
  template's method implementations inside its interface instance are not
  emitted in v1. The kind `interface-instance-method` is reserved for them.
- `qualifiedName` conventions: `Module:Entity` (templates, interfaces,
  types, values) and `Module:Entity.Choice` (choices), matching the
  identifiers that appear in Ledger API events, completions, and error
  messages.
- `lfRef` is the Daml-LF reference where one exists. Tools can join it
  against transaction data without string heuristics.
- `type` (optional) is the rendered LF type for values, methods, and slots.
- Compiler-generated definitions (names containing `$`) are excluded.
- Data types that merely back a template payload, exception, or choice
  argument are not repeated as standalone symbols. Their fields surface as
  value slots of the owning symbol.

### A.6 `valueSlots` and availability

```json
{ "id": "slot:Asset:Asset:Transfer:argument:newOwner",
  "symbol": "sym:Asset:Asset:Transfer", "name": "newOwner",
  "kind": "choice-argument-field", "type": "Party",
  "availability": "transaction-visible" }
```

`availability` is the privacy-honesty contract:

- `transaction-visible`: populatable from participant-visible transaction
  data by a party entitled to see the event.
- `interpreter-only`: only observable with interpreter or runtime support.
  Trace-only tools MUST NOT claim these from transaction data. They should
  show the slot and label the value as not captured.

The values `source-only` and `not-tracked` are reserved for future use.

The availability of each slot kind is **normative**, so a validator can
check conformance:

| Slot kind | Availability | Populated from |
| --- | --- | --- |
| `contract-payload-field` | `transaction-visible` | `CreatedEvent.create_arguments` |
| `choice-argument` | `transaction-visible` | `ExercisedEvent.choice_argument` |
| `choice-argument-field` | `transaction-visible` | `ExercisedEvent.choice_argument` |
| `choice-result` | `transaction-visible` | `ExercisedEvent.exercise_result` |
| `self-contract-id` | `transaction-visible` | event `contract_id` |
| `choice-controllers` | `transaction-visible` | `ExercisedEvent.acting_parties` |
| `choice-observers` | `interpreter-only` | not exposed as a Ledger API event field |
| `choice-authorizers` | `interpreter-only` | not exposed as a Ledger API event field |
| `signatories` | `transaction-visible` | `CreatedEvent.signatories` |
| `observers` | `transaction-visible` | `CreatedEvent.observers` |
| `precondition` | `interpreter-only` | evaluated at interpretation time only |
| `contract-key` | `transaction-visible` | `CreatedEvent.contract_key` |
| `key-maintainers` | `interpreter-only` | maintainers are a subset of signatories but are not identified as maintainers in event data |
| `interface-view` | `transaction-visible` | `CreatedEvent.interface_views` |
| `exception-message` | `interpreter-only` | only failure text reaches completions |

A producer MUST NOT assign a more permissive availability than this table,
and a validator MUST flag violations.

### A.7 `failureSites`

An indexed table of the places a package can fail with a user-relevant
message, so failed completions can be joined to source without free-text
search:

```json
{ "id": "site:Asset:Asset:Transfer:0",
  "symbol": "sym:Asset:Asset:Transfer", "source": "src:Asset",
  "kind": "fail-with-status", "errorId": "AssetError.NotOwner",
  "message": "only the owner can transfer",
  "start": {"line": 31, "column": 5}, "end": {"line": 31, "column": 62} }
```

- Kinds: `abort`, `error`, `assert`, `ensure`, `fail-with-status`, `throw`.
- `errorId` is present when the site is a `failWithStatus` call with a
  statically known error id. `message` is present when the failure text is a
  string literal at the site. Both are omitted when the value is computed
  dynamically.
- `ensure` sites point at the template precondition expression, joinable
  from precondition-failure errors via the owning template symbol.
- Consumers join a failed completion to a site in order of strength: exact
  `errorId` match, then exact static `message` match, then heuristic
  (substring) match. Each degradation MUST be labeled in consumer output, so
  a heuristic match is never presented as an exact one.
- `failWithStatus` is the first-class failure mechanism of this section,
  matching Daml 3.x, where user-defined exceptions are deprecated. `throw`
  sites exist for packages that still use exceptions.

Producers that emit this section include `failure-sites` in
`producer.features`.

### A.8 `steps`

Deterministic evaluation step descriptors: the source spans of the compiled
Daml-LF expression's location nodes in **pre-order**, per owning symbol
(choice bodies and top-level values, which include Daml Script entry
points):

```json
{ "id": "step:Asset:test_transfer:0", "symbol": "sym:Asset:test_transfer",
  "index": 0, "source": "src:Asset",
  "start": {"line": 30, "column": 3}, "end": {"line": 30, "column": 40} }
```

- `index` is **document order** (pre-order of the compiled expression), not
  execution order. Consumers MUST NOT render steps as a timeline.
- De-duplication: when the same `(source, start, end)` span occurs more than
  once in pre-order, only the first occurrence is kept.
- Step ids are stable for a given package id, so runtime events and
  breakpoints can reference them portably. Runtime debug events that carry
  source locations join against these spans by
  `(packageId, module, start, end)`. The package id is part of the join key
  because, under smart contract upgrades, several versions of a package with
  identical module names and often identical spans routinely coexist in one
  run.
- Known limitation: create-time expressions (signatories, observers,
  precondition, key) have spans (A.4) but no steps in v1, so stepping
  through a create does not stop inside them.

### A.9 Runtime debug-trace contract (JSONL)

`daml script --debug-trace-file <file>` writes one JSON object per line.
Consumers MUST ignore unknown `event` kinds and unknown fields. Events are
strictly sequential (Daml Script executes sequentially), and scripts in one
run do not interleave.

```
{"event":"script-start","script":"Asset:test_transfer"}
{"event":"trace","message":"...","location":{LOC}}
{"event":"warning","message":"...","location":{LOC}}
{"event":"question","name":"Submit","version":1,"stackTrace":[{LOC},...]}
{"event":"submission","actAs":["Alice::1220.."],"readAs":[],"location":{LOC}}
{"event":"created","templateId":"<pkgid>:Asset:Asset","contractId":"00..",
 "argument":{...}}
{"event":"exercised","templateId":"<pkgid>:Asset:Asset","interfaceId":null,
 "choice":"Transfer","contractId":"00..","argument":{...},"result":...}
{"event":"script-end","status":"success"}
{"event":"script-end","status":"error","error":"...","location":{LOC}}
```

`LOC = {"packageId","module","definition","startLine","startCol","endLine",
"endCol"}` with 1-based positions and end-exclusive columns (the runtime's
0-based locations are normalized by the emitter), matching
`daml-debug-info/v1` spans. `definition` is the unqualified name of the
top-level definition within `module` (the `script-start` field
`Module:definition` is its qualified form). `location` may be `null` when
the runtime has no location. Trace locations join metadata spans and steps
by `(packageId, module, start, end)`, as in A.8.

**Value encoding.** `argument` and `result` values are encoded with the
daml-lf API JSON codec (`ApiCodecCompressed`, the compact encoding used by
the JSON API for LF values): records as objects, Numerics, dates,
timestamps, parties, and contract ids as strings, and that codec's rules for
optionals and variants. If a value resists that encoding, the producer falls
back to the value's textual rendering as a single JSON string, and consumers
MUST tolerate that fallback.

**Modes.** The `submission`, `created`, and `exercised` events are emitted
in IDE-ledger runs (the reference implementation hooks the IDE ledger
client). When a script runs against a real participant over gRPC, the
reference implementation emits the script-level events only (`script-start`,
`trace`, `warning`, `question`, `script-end`). The MUST-ignore-unknown-events
rule makes adding ledger-mode event emission later backward compatible.

**Stability.** `question` event names and versions (for example `Submit`)
are Daml Script internals with no stability guarantee. They are informative,
and consumers MUST NOT build behavior on specific question names.

**Sensitivity.** Values in `created` and `exercised` events come from a
local script run against the developer's own ledger. The trace never
contains data the submitting context could not see, but it does contain
ledger data (party identifiers, contract payloads). Treat trace files as
sensitive: do not commit them to version control or share them outside the
context entitled to the data. Local variables and intermediate expression
values are **not** captured. Consumers must present them as
`interpreter-only`, not invent them.

### A.10 Runtime follow-up: interpreter location hook

The reference runtime implementation hooks the Daml Script layer: script
questions (with Daml call-stack locations), submissions, ledger events, and
`trace`/`debug` output. Expression-level stepping *within* a choice body
needs one additional hook in the Speedy interpreter, for which this
specification reserves the shape:

```scala
trait MachineLogger {                    // existing trait, existing methods
  def trace(message: String, location: Option[Location]): Unit
  def warn(message: String, location: Option[Location]): Unit
  def onLocation(location: Location): Unit = ()   // NEW, default no-op
}
```

called from the machine's `SELocation` handling, gated so the disabled cost
is one branch. With that hook, the same JSONL contract gains
`{"event":"step","location":{LOC}}` lines that join against `steps` in the
metadata. This is intentionally split out as follow-up work, per this
proposal's scoping.

### A.11 Compiler changes in the reference implementation

Beyond the emitter itself, two compiler fixes were needed to make
choice-level debugging possible, both in the GHC to Daml-LF conversion:

1. `chcLocation` was hard-coded to `Nothing`. It is now populated from the
   desugared choice binder's source span, so choices carry
   `choice-definition` spans.
2. Choice update expressions had **all** source locations stripped
   (`removeLocations`) as a side effect of a structural match. Locations are
   now preserved in the choice body, which both improves runtime stack
   traces and provides `steps` for choices.

Both fixes are **unconditional** compiler changes, deliberately independent
of the debug flag, so that the flag itself never alters the compiled output
(the A.2 producer invariant). Two consequences must be understood:

- Packages built with a fixed compiler have different package ids than
  stock-compiler builds of the same source, exactly as with any compiler
  change. Metadata with choice `steps` therefore exists only for packages
  built by a compiler that includes the fixes. Stepping cannot be
  retrofitted onto packages built before them, because their choice bodies
  contain no location nodes.
- Preserved locations are semantically transparent but not free: they add
  location nodes to the serialized package and to interpretation. The
  standalone upstream pull requests for these fixes include DALF size and
  interpreter overhead measurements on a large public package, so
  maintainers can judge the cost with data.

### A.12 Validation rules for consumers

- Reject unsupported major schema versions. Ignore unknown fields
  otherwise.
- Verify `package.packageId` against the DAR or DALF when both are
  available.
- Verify that sidecar and DAR-member copies are byte-identical when both
  are present. Treat divergence as invalid metadata.
- Verify source `sha256` before trusting spans. On mismatch, retry with
  newline-normalized content and report a line-ending mismatch specifically.
  Otherwise degrade to span-less symbol information (module and
  qualified-name resolution).
- Reject absolute source paths (strict mode) or warn (lenient mode).
- Check value-slot availability labels against the normative table in A.6.
- Never populate an `interpreter-only` slot from transaction data.
- Resolve metadata strictly by package id, never by package name (smart
  contract upgrades make several versions of one package name live at
  once).
- Treat all debug metadata as advisory. It is trusted only as much as its
  distribution channel: nothing binds a sidecar or DAR member to the DALF,
  because `META-INF` members are not part of the package id. For packages
  the consumer did not build, the only strong verification is to rebuild
  from source and compare package ids.

### A.13 Reference implementation status

The reference implementation
(`walnuthq/daml@feature/debug-info`, `walnuthq/dpm-trace`) emits the
following today: `source-spans`, `symbols`, `lf-refs`, `value-slots`,
`steps`, the informative `compatibility` object, the sidecar and DAR-member
copies rendered from one serialization, and the A.9 runtime trace with
IDE-ledger event emission.

Specified in this document and scheduled for Milestone 2:

- `failureSites` (A.7) and the `failure-sites` feature flag.
- `unmappedModules` (A.3).
- Reclassification of `choice-observers` and `key-maintainers` to
  `interpreter-only` in the emitter, per the normative table in A.6 (the
  current prototype labels them `transaction-visible`).
- The CI test asserting package-id equality with and without the debug
  flag (A.2).
- Ledger-mode (gRPC) emission of `submission`/`created`/`exercised` trace
  events, if maintainers want it (A.9 keeps it backward compatible).

This section exists so that no consumer is written against fields the
producer does not yet emit, and so that reviewers can see exactly what the
proof of concept demonstrates versus what the funded milestones deliver.
