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
   The [current draft](https://github.com/walnuthq/daml/blob/feature/debug-info/sdk/compiler/damlc/daml-debug-info-v1.md) is written against a working prototype.
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

The [full draft specification](https://github.com/walnuthq/daml/blob/feature/debug-info/sdk/compiler/damlc/daml-debug-info-v1.md) lives with the prototype compiler
and covers the document structure, the position and hashing rules, the
runtime trace format, and what a consumer must validate.

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

`dpm debug` is new work in this proposal. It runs a Daml Script under the
debugger, stops at breakpoints set on Daml lines, steps from there, and
shows the values it has while labeling the ones it does not.

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

- `daml build --experimental-debug-info` emits the metadata, taken from
  the compiled package rather than by reading source text.
- The two compiler fixes that restore source locations for choices.
- `daml script --debug-trace-file` writes the runtime trace.

In [walnuthq/dpm-trace](https://github.com/walnuthq/dpm-trace) we have
prototyped both sides of the tooling: `dpm trace` reading the metadata for
source links, and an early `dpm debug` stepping through the traces.

The specification's last section says exactly which parts the prototype
emits today and which parts are Milestone 2 work.

---

## References

- Approved proposal this builds on:
  [DPM Trace Transaction Visualization](./dpm-trace-visualization.md)
- Draft specification:
  [daml-debug-info-v1.md](https://github.com/walnuthq/daml/blob/feature/debug-info/sdk/compiler/damlc/daml-debug-info-v1.md)
- Prototype compiler and runtime:
  [github.com/walnuthq/daml (branch feature/debug-info)](https://github.com/walnuthq/daml/tree/feature/debug-info)
- Prototype tooling:
  [github.com/walnuthq/dpm-trace](https://github.com/walnuthq/dpm-trace)
