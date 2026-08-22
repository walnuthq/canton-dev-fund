# Development Fund Proposal: Versioned Debug Info Metadata for Daml

**Author:** [Walnut](https://walnut.dev)<br>
**Status:** Draft<br>
**Created:** 2026-08-19<br>
**Label:** daml-tooling<br>
**Champion:** Need Champion<br>

---

## Abstract

When a Daml workflow fails on Canton, the developer sees an error message
and a package id. They do not see which line of Daml failed, which values
reached it, or how execution got there. The compiler knows all of this, but
nothing carries it from the compiler to the tools developers actually use.

Walnut proposes to fix that. We will define a debug metadata format for
Daml, make the Daml compiler emit it, and build the tooling that uses it. The metadata maps a compiled package back to its source code, so a
tool can point at the exact line that failed instead of guessing.

We hit this problem building our own tool. The Development Fund approved
our
[DPM Trace Transaction Visualization](./dpm-trace-visualization.md)
proposal, and we are building `dpm trace` now. It can show what a
transaction did, but not which Daml code did it. This proposal supplies the
missing metadata, uses it to add source-level views to `dpm trace`, and
adds `dpm debug`, a new debugger that steps through a Daml run in the
source.

---

## What we will build

1. **The format.** A published, versioned specification for Daml debug
   metadata, plus a JSON Schema so tools can check a file automatically.
   Appendix A is the current draft, written against a working prototype.
2. **Compiler support.** `damlc` emits the metadata when asked, and can
   produce a debug build that a debugger is able to stop inside. Both go
   into the official Daml repository.
3. **A runtime trace.** Daml Script writes a log of what a run did and
   where in the source each step happened.
4. **A reader library.** Code that loads the metadata, verifies it, and
   answers "which source line is this?", with example Daml packages we test
   against.
5. **The tooling that uses it.** Source-level views added to `dpm trace`,
   whose transaction inspection and visualization are already funded by the
   approved DPM Trace proposal, and `dpm debug`, a new command-line
   debugger with breakpoints and stepping over Daml source.

---

## Specification

### 1. Objective

Give Daml developers accurate, source-level error messages and debugging.

To do that, the metadata has to answer four questions about any compiled
package:

- Which source files produced it, and do the files on my disk still match?
- Which line of source does each template, choice, and expression come
  from?
- Which values may a tool show, and which values does it not have?
- Where can this package fail, so a failure can be traced back to the line
  that caused it?

The format and the tooling are public and open source, so any Daml tool can
use them. We are building the first user ourselves.

### 2. Motivation

Debugging Daml today is harder than it should be. A failed submission
returns a status and a message. Finding the cause means searching the
source for the error text, which breaks down as soon as two assertions
share a message or a message is built at runtime. Nothing tells a developer
which line ran, in which package version, with which values.

Everything needed to fix this already exists inside the compiler. Daml-LF
carries source locations, and the compiler knows which file every
definition came from. That information is simply thrown away before it
reaches any tool.

Every serious language toolchain solves this the same way, by writing debug
metadata next to the compiled artifact: DWARF for native code, source maps
for JavaScript, ETHDebug for EVM contracts. Daml has no equivalent yet. The
result is that each Canton tool, including ours, has to guess at source
locations, and different tools guess differently.

The value is straightforward. Clear error messages that name a file and a
line. A debugger that steps through Daml source instead of ledger events.
Test and coverage reports that point at real code. Less time spent guessing
why a workflow failed.

### 3. Implementation Mechanics

#### A. The format

We publish `daml-debug-info/v1` as a versioned specification with a JSON
Schema. A metadata file describes one compiled package: its source files
and their hashes, the source span of every template, choice, interface and
value, the Daml-LF references that tie those back to ledger data, the
places the package can fail, and step markers a debugger can walk.

It also records which values a tool is allowed to show. Some values are in
the transaction and any entitled party can see them. Others exist only
inside the interpreter and never appear in ledger data. The format labels
each one, so tools can show what they have and say plainly when they do not
have something, instead of inventing it.

Appendix A is the full draft, backed by a working prototype
([walnuthq/daml](https://github.com/walnuthq/daml/tree/feature/debug-info)).

#### B. Compiler support

`daml build --experimental-debug-info` writes the metadata next to the
compiled DAR, and optionally inside it, where existing tools ignore it.

The flag never changes the compiled package. A package built with it has
the same package id as one built without it, which is what makes the
metadata safe to trust: it describes exactly the artifact you deploy. We
prove this with a test that builds both ways and compares package ids.

Two small fixes to the compiler are needed first. Source locations for
choices are currently dropped during compilation, so no tool can point at a
choice body. Both fixes are useful on their own, because they also improve
Daml stack traces for everyone, so we send them upstream as separate pull
requests before this proposal is voted on.

#### C. Debug builds and the runtime trace

A run has to leave a record before a debugger can show anything.

`daml script --debug-trace-file <file>` writes that record: which script
started, what it submitted, which contracts were created and exercised,
what was traced, and where each of those happened in the source. This uses
an extension point the interpreter already provides, so it needs no change
to Canton.

To follow execution inside a choice body, `daml build --debug` produces a
debug build of the package carrying a step marker at each source location.
Section 4 explains how a debugger uses those markers to stop. Marking is
opt-in and separate from metadata emission, so an ordinary build still
produces an identical package id.

#### D. Reader library and examples

A small open-source library and command-line tool that reads a metadata
file and checks it is trustworthy before anything relies on it: that it
matches the package, that the source files on disk are the ones that were
compiled, that the spans point inside those files, and that no absolute
paths from someone's laptop leaked into it. Given a template or choice, it
returns the source location.

Any tool that wants to use the metadata starts here instead of writing its
own parser. It ships with example Daml packages covering the cases that
break naive implementations, including interfaces, contract keys, several
packages at once, and two versions of the same package.

#### E. The tooling that uses it

Metadata on its own helps nobody, so this proposal also builds the two
pieces that turn it into something a developer uses.

`dpm trace`, funded by our approved
[DPM Trace proposal](./dpm-trace-visualization.md), can already show what a
transaction did. Here we add the source side of it: the Daml line behind
each transaction node, and for a failed submission, the assertion that
rejected it instead of a text search for the error message.

`dpm debug` is new work in this proposal. It replays the runtime trace from
workstream C and steps through the run in the Daml source, showing the
values it has and labeling the ones it does not have.

Between them they are also the proof. If the metadata cannot drive a real
debugger, it is not good enough.

### 4. How the debugger works

C and C++ have used the same debugging model for decades, and this proposal
copies it on purpose.

A C++ toolchain writes debug info next to the compiled binary. It maps
machine addresses back to source lines and records where values live. The
debugger reads that file, and when the developer asks to break on a line it
plants a trap at the matching address. When execution reaches the trap, the
debugger takes over and shows them their own source.

The pieces line up:

| C++ | Daml, in this proposal |
| --- | --- |
| DWARF file beside the binary | `daml-debug-info/v1` beside the DAR |
| compiling with `-g` | `daml build --experimental-debug-info` |
| a build the debugger can stop inside | `daml build --debug`, which adds step markers |
| the trap firing | a marker calling the interpreter's existing logging callback |
| `gdb` or `lldb` | `dpm debug` |

The one real difference is where the trap comes from. In C++ the operating
system provides it, so the binary is untouched and the debugger stops the
process from outside. Canton has no equivalent, and adding one would mean
changing the Daml interpreter, the component that computes transactions on
every validator. We are not proposing that. Nobody wants a debugger that
can pause a live ledger, and the review burden on a consensus-critical
component would be out of proportion to what it buys.

So the compiler plants the traps instead, which is why interactive
debugging uses a debug build. A debug build has a different package id from
the production build, exactly as a `-g -O0` binary differs from a release
binary, and it is used the same way: locally, on your own package, while
you are working on it. The production build still carries debug info and
still has an unchanged package id, which is what lets `dpm trace` put
source locations on real transactions.

Breakpoints then work the way a developer expects. Each marker calls the
interpreter's logging callback synchronously, on the thread evaluating the
choice body. `dpm debug` receives the call, compares it against the
breakpoints the developer set, and either returns at once or holds until
they step. Holding that call holds interpretation, which is what makes a
real pause possible without touching Canton.

Two limits are worth stating plainly. Values are shown when they appear in
the trace and labeled as not captured otherwise, so this is not yet a full
variable inspector. And it debugs Daml Script runs against a local ledger,
not transactions running on someone else's validator, which no participant
operator would allow in any case.

### 5. Architectural Alignment

Nothing here changes Canton. No protocol change, no node change, no change
to how transactions are executed or what any party can see. The metadata
describes code, not ledger data, and the tools that read it keep working
through authorized participant APIs.

The metadata belongs to a package and is keyed by package id, which is what
makes it correct under smart contract upgrades: two versions of a package
have different ids and different metadata, so a tool always resolves
against the version that actually ran.

### 6. Backward Compatibility

No backward compatibility impact. The metadata is new, optional, and
written alongside the DAR rather than inside the package, so existing
builds, tools, and deployments are unaffected.

---

## Milestones and Deliverables

### Milestone 1: Published specification and compiler fixes merged

**Estimated Delivery:** 4 weeks from start<br>
**Focus:** Agree the format with Daml maintainers and land the groundwork
in the official compiler.

**Deliverables:**

- The `daml-debug-info/v1` specification published, with a JSON Schema.
- The two compiler location fixes merged into `digital-asset/daml`,
  including measurements of their effect on package size and interpreter
  speed.
- A review of the format with Daml/Canton maintainers.

**Acceptance Criteria:**

- The specification is public and covers, at minimum, templates, choices,
  choice arguments, payload fields, and failure sites.
- Daml maintainers have reviewed the format and agree it is emittable by
  the compiler.
- The two compiler fixes are **merged** into `digital-asset/daml`.

### Milestone 2: Compiler and Daml Script emission merged

**Estimated Delivery:** 8 weeks after Milestone 1 acceptance<br>
**Focus:** Ship metadata emission and the runtime trace in the official
toolchain.

**Deliverables:**

- `damlc` emits `daml-debug-info/v1` behind a flag.
- `daml build --debug` produces a debug build carrying step markers.
- Daml Script writes the runtime debug trace.
- The test proving the metadata flag does not change the compiled package.
- Documentation for all three.

**Acceptance Criteria:**

- Both changes are **merged** into `digital-asset/daml`.
- A developer can build any Daml package with a released or nightly build
  of the compiler and get valid metadata, with no absolute paths and no
  change to the package id.
- If Daml maintainers decline the upstream change, the committee decides
  whether to accept delivery through a documented alternative distribution
  instead. We do not treat an open pull request as delivery.

### Milestone 3: Reader library and examples

**Estimated Delivery:** 4 weeks after Milestone 2 acceptance<br>
**Focus:** Make the metadata usable by any tool, not just ours.

**Deliverables:**

- Open-source reader and validator library and command-line tool.
- Example Daml packages covering interfaces, contract keys, multi-package
  builds, and package upgrades.
- Documentation for tool authors.

**Acceptance Criteria:**

- The library accepts every valid example and rejects deliberately broken
  ones, including stale sources and mismatched package ids.
- Someone who is not us can go from a template or choice name to the right
  source line using only the published library and documentation.

### Milestone 4: Source-level debugging in our tools, and adoption

**Estimated Delivery:** 5 weeks after Milestone 3 acceptance<br>
**Focus:** Prove it works by shipping it to Canton developers.

**Deliverables:**

- `dpm trace` gains source locations for transactions and failed
  submissions.
- `dpm debug`, the new command-line debugger, sets breakpoints on Daml
  lines and steps through a run in the source.
- A worked example and walkthrough, from building a package to debugging a
  failure in it.
- Feedback from at least three Canton developers or teams outside Walnut.

**Acceptance Criteria:**

- A developer can take a failing Daml workflow and see the exact line that
  rejected it, without searching the source for the error message.
- `dpm debug` stops at a breakpoint on a line inside a choice body, steps
  from there, and shows the values it has while labeling the ones it does
  not.
- At least two testers outside Walnut confirm this is better than
  debugging without it.

---

## Acceptance Criteria

Each milestone above carries its own criteria, and those are the bar. Two
apply throughout: the compiler work is delivered by being merged into the
official Daml repository, not by being proposed, and every deliverable is
open source under Apache-2.0.

---

## Funding

**Total Funding Request:** USD [amount to confirm], paid in Canton Coin.

Milestones are priced in US dollars. When a milestone is accepted, the
payment is made in Canton Coin at the CC/USD rate at the time of payment,
so the amount of CC varies and the value delivered does not. This removes
the price risk that a fixed CC amount puts on both sides, since CC can move
sharply in a week and this project runs for months.

### Payment Breakdown by Milestone

- Milestone 1, Published specification and compiler fixes merged:
  USD [amount] on acceptance.
- Milestone 2, Compiler and Daml Script emission merged:
  USD [amount] on acceptance.
- Milestone 3, Reader library and examples:
  USD [amount] on acceptance.
- Milestone 4, Source-level debugging in our tools, and adoption:
  USD [amount] on acceptance.

---

## Co-Marketing

Walnut can work with the Canton Foundation on a technical post about how
Daml debug metadata works, a demo showing a failing workflow debugged down
to the line, and documentation for teams building Daml tools.

---

## Rationale

### Why put this in the compiler instead of working it out from source?

Only the compiler knows which source produced which part of a package. Any
tool working from the outside has to guess by matching text, and it guesses
wrong exactly when it matters most: when two assertions share a message, or
the source on disk is not the source that was built. Writing the answer
down at compile time is what makes it reliable.

### Why get it merged upstream instead of shipping our own compiler?

A debug format is only useful if the compiler people already use emits it.
A patched compiler that only we ship would be a fork nobody adopts, and
would fall behind on the first SDK release. That is why Milestones 1 and 2
are only accepted when the changes are merged into `digital-asset/daml`,
and why the two prerequisite fixes go upstream on their own regardless of
whether this proposal is funded.

### Why publish the format instead of keeping it internal?

We would benefit either way, but the ecosystem only benefits if the format
is public. Coverage tools, test reporters, profilers, and explorers all
need the same source mapping. If we keep it private, every one of them
either builds its own or does without.

### Why record what a tool may and may not show?

Canton is private by design, and a debugger that quietly invents a value it
cannot actually see is worse than one that shows nothing. Labeling each
value as visible in the transaction or interpreter-only makes honesty
something the format enforces rather than something each tool remembers to
do.

---

## Risks and Mitigations

- **Upstream review takes time.** Merging into `digital-asset/daml` is not
  fully in our control. We open the two small fixes before this proposal is
  voted on, so review starts early and maintainers can weigh in on the
  approach before the larger changes arrive. Milestone 2 names the fallback
  if they decline.
- **The format misses something real.** The specification is not frozen
  until maintainers have reviewed it, and the prototype already covers real
  packages, so gaps show up as concrete cases rather than as opinions.
- **Tools overstate what they know.** The value availability rules are part
  of the specification and are checked by the reader library, so a tool
  that claims a value it cannot have fails validation.

---

## Maintenance

Everything we produce is open source under Apache-2.0. Walnut maintains the
specification, the reader library, `dpm debug`, and the source-level
features in `dpm trace`. Once the compiler changes are merged, they are maintained in
`digital-asset/daml` like the rest of the compiler, which is the point of
upstreaming them.

---

## About the Team

Walnut has spent four years building debugging and observability tooling
for blockchains, and we build it with the teams behind the platforms
themselves:

- **Canton.** The Development Fund approved our
  [DPM Trace Transaction Visualization](./dpm-trace-visualization.md)
  proposal and we are building `dpm trace` now. This proposal comes
  directly out of that work: we hit the missing metadata while building
  it.
- **Starkware / Starknet.** We build the
  [Walnut Starknet Debugger](https://walnut.dev/), covering debug info
  generation, tracing, simulation, verification, and the hosted debugger
  itself. Three-year partnership.
- **Ethereum Foundation / Argot.** We own debug info generation in
  [`solc`](https://github.com/argotorg/solidity), the official Solidity
  compiler. One-year partnership, being extended.
- **Arbitrum / Offchain Labs.**
  [StylusDB](https://github.com/OffchainLabs/stylus-sdk-rs/blob/main/cargo-stylus/docs/StylusDebugger.md),
  the official debugger for Stylus.

Debug info generation in a production compiler is the specific thing we
have done before, in `solc` and in Starknet, and it is the specific thing
this proposal asks for in Daml.

---

## What already exists

We built a working prototype before asking for funding, so this proposal
describes something we have already proven rather than something we hope
will work. In
[walnuthq/daml](https://github.com/walnuthq/daml/tree/feature/debug-info):

- `daml build --experimental-debug-info` emits the metadata described in
  Appendix A, taken from the compiled package rather than by reading source
  text.
- The two compiler fixes that restore source locations for choices.
- `daml script --debug-trace-file` writes the runtime trace.

In [walnuthq/dpm-trace](https://github.com/walnuthq/dpm-trace) we have
prototyped both sides of the tooling: `dpm trace` reading the metadata for
source links, and an early `dpm debug` stepping through the traces.

Appendix A.12 says exactly which parts of the specification the prototype
emits today and which parts are Milestone 2 work.

---

## References

- Approved proposal this builds on:
  [DPM Trace Transaction Visualization](./dpm-trace-visualization.md)
- Prototype compiler and runtime:
  [github.com/walnuthq/daml (branch feature/debug-info)](https://github.com/walnuthq/daml/tree/feature/debug-info)
- Prototype tooling:
  [github.com/walnuthq/dpm-trace](https://github.com/walnuthq/dpm-trace)

---

## Appendix A: `daml-debug-info/v1` draft specification

This appendix is the draft specification. It is backed by the working
prototype in
[walnuthq/daml](https://github.com/walnuthq/daml/tree/feature/debug-info),
not written from theory. During Milestone 1 it moves to a public location
agreed with Daml maintainers and gains a machine-checkable JSON Schema. MUST, SHOULD, and MAY
are used as in RFC 2119, and A.12 records where the implementation still
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

`availability` says what a tool is allowed to show. A `transaction-visible`
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

### A.9 Runtime debug-trace format (JSONL)

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

### A.10 Compiler changes in the reference implementation

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

### A.11 Validation rules for consumers

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

### A.12 Reference implementation status

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
