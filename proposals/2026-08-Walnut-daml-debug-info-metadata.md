# Development Fund Proposal: Versioned Debug Info Metadata for Daml

**Author:** [Walnut](https://walnut.dev)<br>
**Status:** Draft<br>
**Created:** 2026-08-15<br>
**Label:** daml-tooling<br>
**Champion:** Need Champion<br>

---

## Abstract

Walnut proposes a versioned debug metadata format for Daml packages, together
with the compiler support to emit it and the tooling to consume it.

The goal is to give Daml/Canton developer tools a stable, portable way to map
a compiled package back to its source: which files produced it, which spans
correspond to which definitions, where it can fail, and which values a tool
can actually show. The first consumer is the approved
[DPM Trace Transaction Visualization](./dpm-trace-visualization.md)
proposal, which names this work as its planned "Compiler Source Metadata"
follow-on. Test, coverage, and explorer tools can build on the same
contract.

The first version of the format, `daml-debug-info/v1`, is specified
in [Appendix A](#appendix-a-daml-debug-infov1-draft-specification) of this
document and is backed by a working reference implementation.

---

## Specification

### 1. Objective

Create a versioned Daml debug metadata standard, upstream its emission
paths, and validate it with independent consumers.

The first version targets Daml SDK 3.x packages (Daml-LF 2.1 and later) and
answers these questions for any package:

- Which source files produced this package, and which hashes prove a local
  checkout matches it?
- Which definitions and Daml-LF references correspond to which source
  spans?
- Which values can a tool populate from transaction data, and which need
  interpreter support and must not be claimed by trace-only tools?
- Which failure sites exist, and how can a failed completion be joined back
  to one of them?

The result is a public schema and reference tooling.

### 2. Motivation

Daml and Canton already provide most of the raw material. Participants
expose authorized, participant-scoped updates and completions. DARs and
DALFs identify the packages being interpreted. Daml-LF carries source
location information, and `damlc inspect`, package manifests, and local
source trees can fill in hints.

What is missing is a contract. A tool builder today reconstructs intent by
combining package ids, local paths, source spans, text search, Daml Script
output, error strings, and transaction trees. That breaks in predictable
ways. Absolute paths leak into reports. Source matching fails when the
package source is not local. A dozen identical assertion strings map to a
dozen possible locations. Tools overstate what they can replay, because
transaction data contains no interpreter state. Nothing can be compared
across SDK versions, and every debugger, test reporter, and explorer ends
up inventing its own source map.

Other ecosystems solved this with explicit debug metadata formats: DWARF
for native toolchains, PDB/CodeView on Windows, source maps for JavaScript,
ETHDebug in EVM tooling. Daml needs its own, adapted to Canton privacy and
Daml-LF semantics.

### 3. Implementation Mechanics

The proposal has five technical workstreams.

#### A. Debug Metadata Schema

Define `daml-debug-info/v1` as a versioned schema. The concrete encoding is
human-readable JSON with a machine-checkable JSON Schema validator. A future
protobuf encoding can be added without changing the conceptual model.

The schema includes:

- `package` and `producer`: package id, package name and version, Daml-LF
  and SDK versions, emitting tool, and feature flags.
- `sources`: package-relative paths, SHA-256 hashes, the module-to-file
  mapping, and a record of modules that could not be mapped.
- `spans` and `symbols`: source spans for definitions and located
  expressions, and the templates, choices, interfaces, and values they
  belong to, each with its Daml-LF reference.
- `valueSlots`: value locations a tool may try to populate, each labeled
  `transaction-visible` or `interpreter-only`.
- `failureSites`: assertion, abort, precondition, and `failWithStatus`
  sites with spans and statically known messages or error ids.
- `steps`: deterministic source-level step descriptors.

[Appendix A](#appendix-a-daml-debug-infov1-draft-specification) is the
complete specification, including producer invariants, position semantics,
and validation rules. The schema will be reviewed with Daml/Canton
maintainers before it is declared stable, and the first implementation is
marked experimental until then.

#### B. Compiler Emission

Upstream debug metadata emission in the Daml compiler.

The emission is strictly additive: no Daml-LF semantic change, no Canton
protocol or node change, no mandatory runtime overhead.

One rule is central, and Appendix A.2 makes it binding:
turning on metadata emission does not change the compiled Daml-LF output. A
package built with the flag has the same package id as one built without it,
so metadata from a debug build describes the exact artifact that gets
deployed. The upstream contribution includes a CI test for this package-id
equality.

The flag is `daml build --experimental-debug-info`. It writes a sidecar
file next to the DAR and an optional DAR member under
`META-INF/daml-debug-info/` that existing tools ignore, both byte-identical
(Appendix A.1). Source paths are package-relative, and absolute local paths
are never emitted.

Two small compiler fixes are prerequisites for choice-level debugging
(Appendix A.11). They go to `digital-asset/daml` as standalone pull
requests ahead of this proposal's review.

#### C. Runtime Debug Trace

Extend the Daml Script runner with an opt-in runtime debug trace:

```bash
daml script --debug-trace-file <file> ...
```

The runner writes one JSON object per line: script start and end, script
questions with call-stack locations, submissions, and the ledger events the
run observed. Positions use the same conventions as the metadata, so trace
events join against spans and steps. The full contract is Appendix A.9.

This workstream covers the Daml Script layer only. Expression-level stepping
inside choice bodies needs one additional hook in the Speedy interpreter. That
hook is out of scope here: Appendix A.10 reserves its shape so a follow-up
proposal can add it without changing the trace contract.

#### D. Reader, Validator, and Test Corpus

Build a reference reader and validator package and CLI implementing the
consumer rules of Appendix A.12: schema and hash verification, span and
availability conformance, byte-identical copies, path hygiene, and a
dedicated diagnostic for line-ending mismatches.

The test corpus covers creates and consuming exercises, nested exercises
with child creates, controllers, observers and signatories, interfaces and
interface choices, contract keys where the target LF version has them,
assertion and `failWithStatus` sites, multi-module and multi-package
layouts, and a package upgrade example with two versions of the same
package.

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

The design follows Canton architecture:

- It respects participant-scoped visibility. Debug metadata maps code to
  source and reveals no ledger data outside the requesting party's rights.
- It is package-oriented, matching how Daml-LF and DARs are distributed.
- It complements package upload and vetting. Vetting says a participant is
  willing to process a package. Debug metadata says which source and spans
  correspond to it.
- It requires no changes to Canton consensus, synchronization, privacy, or
  transaction semantics.

Metadata is keyed by package id, so smart contract upgrades need no special
handling: consumers resolve it by package id, never by package name. A
participant resolving by name may execute a different version of a package
than the developer expects, which is exactly why the format keys nothing on
names.

Nothing cryptographically binds a sidecar or DAR member to a DALF, since
`META-INF` members are not part of the package id. Debug metadata is
therefore advisory, trusted as much as its distribution channel. For
packages a developer did not build, the strong check is to rebuild from
source and compare package ids. The ledger itself never transports the
metadata: Canton distributes DALFs, not DAR members, so counterparties
cannot receive a package's debug info through the ledger. That protects
privacy, and it also makes a registry for third-party packages the natural
follow-on.

The result is one contract for many tools instead of a single debugger or
one vendor's trace format.

### 5. Backward Compatibility

The first version is additive. Existing Daml applications compile and run
unchanged, existing DAR consumers ignore metadata members they do not
understand, and tools that do not know the format keep using today's
package and source mechanisms.

Embedding the metadata changes the DAR file hash (the zip bytes change) but
never the package id, so workflows that pin DAR file hashes must pin the
debug-enabled DAR. Schema evolution is versioned: consumers ignore unknown
fields within a supported major version and reject unsupported major
versions.

---

## Non-Goals

This proposal stops at the metadata contract. It does not fund an IDE
debugger (no DAP adapter, no VS Code extension, no live breakpoints in
Canton participants), a hosted source registry or Sourcify-style
verification service, or changes to the Daml-LF interpreter (the stepping
hook is reserved in Appendix A.10 for a follow-up). All of these are
candidates for later proposals once the metadata contract is accepted.

---

## Milestones and Deliverables

A working proof of concept already exists (see Reference Implementation
Status below). The milestones are set beyond it, so no payment covers work
the prototype already demonstrates. The funding goes to maintainer
alignment, upstreaming, and independent validation.

### Milestone 1: Maintainer-Aligned Schema and Upstream Groundwork

**Estimated Delivery:** 4 weeks from start<br>
**Focus:** Get the specification reviewed by maintainers and the first
upstream pull requests opened.

**Deliverables / Value Metrics:**

- Public `daml-debug-info/v1` specification and machine-checkable JSON
  Schema, published in a location agreed with maintainers (Appendix A is
  the starting draft).
- Versioning, position, and path-hygiene rules, the per-slot-kind
  availability table (Appendix A.6), and the `failureSites` definition
  (Appendix A.7).
- The two compiler location fixes submitted to `digital-asset/daml` as
  standalone pull requests, with DALF size and interpreter overhead
  measurements on a large public package.
- Review session with Daml/Canton maintainers and at least one ecosystem
  tooling consumer.

**Acceptance Criteria:**

- A maintainer or delegated reviewer confirms the schema is plausible for
  compiler emission and covers template, choice, choice argument, payload
  field, and failed assertion spans.
- The schema distinguishes transaction-visible from interpreter-only values
  per slot kind.
- The two compiler fix pull requests are open upstream and under review.

### Milestone 2: Compiler and Runtime Emission Upstreaming

**Estimated Delivery:** 6 weeks after Milestone 1 acceptance<br>
**Focus:** Get the metadata and runtime trace emission into the upstream
toolchain.

**Deliverables / Value Metrics:**

- Metadata emission submitted to `digital-asset/daml` behind an
  experimental flag (or through an emission path maintainers prefer), with
  a CI test that the package id is identical with and without the flag
  (the Appendix A.2 invariant).
- Runtime debug trace emission in the Daml Script runner (workstream C),
  submitted the same way.
- Path normalization, source hashes, `unmappedModules`, `failureSites`
  emission, and span mapping for modules, templates, choices, and fields.
- Documentation for generating metadata and traces.

**Acceptance Criteria:**

- A representative package built with the upstream-submitted path produces
  a complete `daml-debug-info/v1` artifact with no absolute paths, and
  builds without the flag are unaffected.
- The package-id invariance test passes in CI.
- The upstream pull requests are under active review, or an alternative
  distribution path agreed with maintainers is documented and delivered.

### Milestone 3: Reader, Validator, and Corpus

**Estimated Delivery:** 4 weeks after Milestone 2 acceptance<br>
**Focus:** Make the metadata consumable and checkable by independent tools.

**Deliverables / Value Metrics:**

- Open-source reader and validator library and CLI, with diagnostics for
  stale hashes (including line-ending mismatches), span and availability
  violations, sidecar/DAR-member divergence, unsupported versions, and
  non-portable paths.
- Test corpus covering the workstream D constructs, including multi-package
  and upgrade examples, plus forward-compatibility tests for unknown
  fields.
- Documentation for tool authors.

**Acceptance Criteria:**

- The validator accepts all valid corpus artifacts and rejects
  intentionally corrupted ones.
- A tool author can resolve a package, module, template, or choice
  reference to a source span without Walnut-specific code.

### Milestone 4: Consumer Integration, Adoption, and Ecosystem Validation

**Estimated Delivery:** 5 weeks after Milestone 3 acceptance<br>
**Focus:** Prove the standard improves real Canton developer workflows.

**Deliverables / Value Metrics:**

- `dpm trace` loads sidecar or DAR-embedded metadata and uses
  `failureSites` for failed-completion diagnostics, with labeled match
  quality (exact error id, exact message, or heuristic).
- Source-linked transaction, compare, and test-report output.
- `dpm debug` source-level stepping over the workstream C runtime trace,
  labeling transaction-visible and interpreter-only values.
- Example package and walkthrough from compiler emission to consumer
  output.
- Feedback from at least three Canton developers or organizations, and
  follow-up issues for the Speedy location hook.

**Acceptance Criteria:**

- `dpm trace` resolves failed-completion diagnostics without text-search
  heuristics when `failureSites` metadata is available.
- Both tools refuse to show values absent from participant-visible data or
  the local trace, labeling them interpreter-only.
- At least two independent testers confirm the metadata-backed source links
  beat the pre-metadata workflow.

---

## Acceptance Criteria

The Tech & Ops Committee can evaluate completion based on:

- A public, versioned specification with a machine-checkable schema,
  reviewed by maintainers.
- Emission paths submitted upstream, with the package-id invariance tested
  in CI.
- A reader, validator, and test corpus that independent tools can use.
- `dpm trace` and `dpm debug` integrations that improve real developer
  workflows, with independent ecosystem feedback.

The value to the ecosystem is not the existence of a JSON file. The value is
that source-aware Daml/Canton tools can share one stable metadata contract
instead of building incompatible heuristics.

---

## Funding

**Total Funding Request:** 1,600,000 Canton Coin (CC)

Each payment is tied to something the committee can inspect, not to
internal activity.

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

Walnut can work with the Canton Foundation and Daml/Canton maintainers on a
technical post about Daml debug metadata and Canton privacy boundaries, a
demo of metadata-backed diagnostics and `dpm debug` stepping, and
documentation for ecosystem tool authors.

---

## Rationale

### Why start with metadata instead of a full debugger?

A full source-level debugger needs runtime and interpreter hooks, breakpoint
semantics, value capture policy, and careful privacy review. Metadata is the
lower-risk foundation that every later tool needs anyway. It improves
tracing, test reporting, and coverage work now, and it makes the later
conversation about runtime hooks specific instead of hypothetical.

### Why upstream instead of maintaining a fork?

A debug format is only a shared contract if the compiler everyone already
uses can emit it. A forked emitter would drift out of date and split the
ecosystem, so Milestones 1 and 2 tie acceptance to upstream submission and
maintainer review. The two prerequisite compiler
fixes go upstream as standalone pull requests in any case, since they
improve stack traces for every Daml user whatever happens to this proposal.

### Why use package-relative paths and hashes?

Daml packages move across machines, CI systems, validators, and
organizations. Absolute local paths are not portable and can leak private
machine details. Package-relative paths plus source hashes let tools verify
source identity without committing to a developer's local filesystem layout.
The specification also defines hashes at the byte level and requires a
specific line-ending mismatch diagnostic, because checkouts with translated
line endings are the most common cause of spurious hash failures in Solidity
metadata verification.

### Why a dedicated `failureSites` section?

Failed completions carry a status and a message, not a source location.
Without an indexed table of failure sites, tools must text-search source
files for the message, which is ambiguous whenever messages repeat or are
built dynamically. `failureSites` gives consumers an exact join key when
`failWithStatus` error ids or static messages are used, and a clearly
labeled weaker match otherwise. It also puts `failWithStatus` at the center
of the failure model, matching Daml 3.x, where user-defined exceptions are
deprecated.

### Why integrate with `dpm trace` first?

`dpm trace` is the approved consumer that already exercises the
hard parts: participant-scoped updates, failed completions, prepared
transactions, test reports, and source diagnostics. Integrating metadata
there validates the schema against real workflows without requiring a new
hosted product or IDE.

---

## Risks and Mitigations

- **Maintainer alignment:** review is Milestone 1, the schema is not frozen
  before it, and the standalone fixes open the conversation with small,
  useful changes. If upstreaming is declined, Milestone 2 falls back to a
  distribution path agreed with maintainers.
- **Overstating runtime values:** the availability table makes the correct
  classification testable, and consumers must show a missing
  interpreter-only value as missing rather than guessing it.
- **Path and data leaks:** package-relative paths and hashes by default,
  absolute paths rejected in strict mode, and runtime trace files
  documented as sensitive.
- **Scope creep:** runtime-hook findings become follow-up proposals, not
  blockers for the metadata standard.

---

## Maintenance

The specification will live in a maintainer-approved public location once
accepted, and reference tooling is Apache-2.0 unless maintainers prefer
another license. Walnut maintains the `dpm trace` and `dpm debug`
integrations. Compiler emission ownership is a Milestone 1 discussion item.

---

## About the Team

Walnut builds blockchain debugging and observability tooling. The team has
experience with compiler debug information, source mapping, transaction
visualization, simulation, contract verification, and debugger UX across
ecosystems including Starknet, Ethereum/Solidity (debug info generation in
the official Solidity compiler, `solc`), and Arbitrum Stylus.

---

## Reference Implementation Status

A working reference implementation of the `daml-debug-info/v1` schema
exists and backs Appendix A. On branch `walnuthq/daml@feature/debug-info`,
`daml build --experimental-debug-info` emits the metadata from the compiled
Daml-LF package (sidecar JSON plus a DAR member), and
`daml script --debug-trace-file` emits the Appendix A.9 runtime trace. The
metadata is derived from the compiled package, never from textual scanning
of sources. Two compiler fixes make choice-level debugging possible
(Appendix A.11), and Walnut will submit both to `digital-asset/daml` as
standalone pull requests ahead of this proposal's review, with DALF size
and interpreter overhead measurements. In `walnuthq/dpm-trace`, `dpm trace`
consumes the metadata for source links and `dpm debug` steps through
runtime traces. Appendix A.13 lists what the implementation emits today
versus what is Milestone 2 work.

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

This appendix is the draft specification, backed by the reference
implementation. During Milestone 1 it moves to a maintainer-agreed public
location and gains a machine-checkable JSON Schema. MUST, SHOULD, and MAY
are used as in RFC 2119, and A.13 records where the implementation still
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

- The two copies MUST be byte-identical, and a consumer that finds
  different bytes MUST treat the metadata as invalid. The DAR member is
  authoritative, and the sidecar exists for convenience.
- v1 producers emit metadata for the main package only. Dependencies carry
  their own metadata in their own DARs.
- Embedding changes the DAR file hash, never the package id.
- Producers SHOULD emit a canonical serialization (UTF-8, LF newlines,
  stable key order per producer version), so identical inputs produce
  identical bytes.

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
reject unsupported major versions and MUST ignore unknown fields in a
supported one. The `compatibility` object is informative only: producers
MAY emit it, and consumers MUST NOT rely on it.

**Producer invariants.**

- Enabling metadata emission MUST NOT change the compiled Daml-LF output.
  The package id of a package built with and without the emission flag MUST
  be identical, so that metadata from a debug build describes the exact
  artifact that is deployed.
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
source root at build time.

- `path` is package-relative (relative to the `source:` root of
  `daml.yaml`). Producers MUST NOT emit absolute local paths.
- `sha256` is the hash of the **exact bytes the compiler read**, with no
  normalization, verified by consumers before trusting spans. A checkout
  with translated line endings (git `core.autocrlf` on Windows) has
  different bytes, so consumers SHOULD retry with newline-normalized
  content and, on a match, report a line-ending mismatch instead of a
  generic hash failure.
- `module` gives consumers the module-to-file mapping directly (Daml-LF
  does not serialize module source paths).
- `uri` is informative display material only. Its authority is the package
  name, which is not unique, so consumers MUST NOT use `uri` as a
  resolution key. `path` plus `sha256` are the normative identifiers.
- Modules whose sources did not resolve under the source root (files
  included from outside it, generated modules) are listed by name in the
  top-level `unmappedModules` array, so "no mapping" is distinguishable
  from "no such module".

### A.4 `spans`

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
- `interface-method` and its span kind refer to the declaration inside the
  interface definition. Implementation spans in a template's interface
  instance are not emitted in v1 (`interface-instance-method` is reserved).
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

`availability` is the privacy-honesty contract. A `transaction-visible`
slot is populatable from participant-visible transaction data by a party
entitled to see the event. An `interpreter-only` slot is observable only
with interpreter or runtime support: trace-only tools MUST NOT claim it
from transaction data, and should show the slot with the value marked as
not captured. (`source-only` and `not-tracked` are reserved.) The
availability of each slot kind is normative, so a validator can check
conformance:

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
  `errorId`, exact static `message`, then heuristic substring. Each
  degradation MUST be labeled, so a heuristic match is never presented as
  exact.
- `failWithStatus` is the primary failure mechanism of this section,
  matching Daml 3.x, where user-defined exceptions are deprecated. `throw`
  sites exist for packages that still use exceptions.

Producers that emit this section include `failure-sites` in
`producer.features`.

### A.8 `steps`

Deterministic evaluation step descriptors: the source spans of the compiled
Daml-LF expression's location nodes in **pre-order**, per owning symbol
(choice bodies and top-level values, which include Daml Script entry
points).

- `index` is **document order** (pre-order of the compiled expression), not
  execution order. Consumers MUST NOT render steps as a timeline.
- De-duplication: when the same `(source, start, end)` span occurs more than
  once in pre-order, only the first occurrence is kept.
- Step ids are stable for a given package id, so runtime events and
  breakpoints can reference them portably. Runtime events join these spans
  by `(packageId, module, start, end)`: under smart contract upgrades,
  several versions of a package with identical module names and spans can
  coexist in one run, so the package id is part of the key.
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
in IDE-ledger runs. Against a real participant over gRPC, the reference
implementation emits the script-level events only. The ignore-unknown-events
rule makes adding ledger-mode emission later backward compatible.

**Stability.** `question` event names and versions (for example `Submit`)
are Daml Script internals with no stability guarantee. They are informative,
and consumers MUST NOT build behavior on specific question names.

**Sensitivity.** Trace values come from a local script run and never
contain data the submitting context could not see, but they do contain
ledger data (party identifiers, payloads). Treat trace files as sensitive
and keep them out of version control. Local variables and intermediate
expression values are **not** captured. Consumers must present them as
`interpreter-only`, not invent them.

### A.10 Runtime follow-up: interpreter location hook

The reference runtime hooks the Daml Script layer only. Expression-level
stepping *within* a choice body needs one additional Speedy interpreter
hook, for which this specification reserves the shape: a
`MachineLogger.onLocation(location: Location): Unit` method with a default
no-op body, called from the machine's `SELocation` handling and gated so
the disabled cost is one branch. With that hook, the same JSONL contract
gains `{"event":"step","location":{LOC}}` lines that join against `steps`.
This is follow-up work, per this proposal's scoping.

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

Both fixes are **unconditional** compiler changes, kept independent of the
debug flag so that the flag itself never alters the compiled output (the
A.2 producer invariant). Two consequences follow:

- Packages built with a fixed compiler have different package ids than
  stock-compiler builds of the same source, as with any compiler change.
  Choice `steps` therefore exist only for packages built with the fixes.
  Stepping cannot be retrofitted onto older builds, whose choice bodies
  contain no location nodes.
- Preserved locations are semantically transparent but not free: they add
  location nodes to the serialized package and to interpretation. The
  upstream pull requests include DALF size and interpreter overhead
  measurements, so maintainers can judge the cost with data.

### A.12 Validation rules for consumers

- Reject unsupported major schema versions. Ignore unknown fields
  otherwise.
- Verify `package.packageId` against the DAR or DALF, and verify that
  sidecar and DAR-member copies are byte-identical. Treat divergence as
  invalid metadata.
- Verify source `sha256` before trusting spans. On mismatch, retry with
  newline-normalized content and report a line-ending mismatch
  specifically. Otherwise degrade to span-less symbol information.
- Reject absolute source paths (strict mode) or warn (lenient mode).
- Check availability labels against the table in A.6, and never populate an
  `interpreter-only` slot from transaction data.
- Resolve metadata strictly by package id, never by package name.
- Treat all metadata as advisory, trusted as much as its distribution
  channel. For packages the consumer did not build, the strong check is to
  rebuild from source and compare package ids.

### A.13 Reference implementation status

The reference implementation
(`walnuthq/daml@feature/debug-info`, `walnuthq/dpm-trace`) emits the
following today: `source-spans`, `symbols`, `lf-refs`, `value-slots`,
`steps`, the informative `compatibility` object, the sidecar and DAR-member
copies rendered from one serialization, and the A.9 runtime trace with
IDE-ledger event emission.

Specified in this document and scheduled for Milestone 2: `failureSites`
(A.7), `unmappedModules` (A.3), the package-id invariance CI test (A.2),
reclassifying `choice-observers` and `key-maintainers` to
`interpreter-only` in the emitter (the prototype labels them
`transaction-visible`), and optional ledger-mode emission of trace events
(A.9).

Consumers should detect what a given artifact contains from
`producer.features`, not from this list, which describes one implementation
as of this writing.
