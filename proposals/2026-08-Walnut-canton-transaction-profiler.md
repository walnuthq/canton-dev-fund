# Development Fund Proposal: Canton Transaction Profiler

**Author:** [Walnut](https://walnut.dev)<br>
**Status:** Draft<br>
**Created:** 2026-08-19<br>
**Label:** daml-tooling<br>
**Champion:** Curtis Hrischuk, Digital Asset (curtis.hrischuk@digitalasset.com)<br>

---

## Abstract

Walnut proposes `dpm profile`, an open-source cost and execution profiler
for Canton transactions and Daml Script runs.

It reports transaction size per node, template, and payload field, with an
estimated traffic cost. It reports which source lines a Daml Script run
executed. Every command is non-interactive, with JSON output and defined
exit codes, so a cost regression can fail a pull request.

```bash
dpm profile tx --prepared prepared.json    # where the bytes go
dpm profile run run.jsonl                  # which source lines ran
dpm profile check --budget budgets.json    # CI gate, non-zero on regression
```

Canton's Daml profiler measures interpretation time only; this proposal
builds on its output rather than replacing it. The cost profile and the
execution profile are prototyped, and the samples in this document are tool
output.

This is the third tool in a family: the approved
[DPM Trace Transaction Visualization](./2026-08-Walnut-dpm-trace-visualization.md)
inspects transactions,
[`dpm debug`](https://github.com/canton-foundation/canton-dev-fund/pull/743)
steps through them, and `dpm profile` measures them.

---

## What exists today, and what is missing

Canton ships one profiler, the Daml profiler. It measures Daml
interpretation time inside the engine, per expression. No tool measures the
serialized transaction, which is the layer Canton meters and bills.

The **[Daml profiler](https://docs.canton.network/sdks-tools/development-tools/daml-profiler)**
emits one speedscope JSON file per command. Setup: set `profile-dir` under
`participants.<name>.features`, start `dpm sandbox`, run a script. Its
output uses Daml-LF identifiers. Profiling a create of the `Asset` template
used later in this document gives:

```
create @Asset:Asset
ensures @Asset:Asset
Asset:$censure
signatories @Asset:Asset
Asset:$csignatory
DA.Internal.Template.Functions:$ctoParties
observers @Asset:Asset
Asset:$cobserver
```

`$censure`, `$csignatory`, and `$cobserver` are compiler-generated, and an
exercise profile of the same package adds `Asset:$$sc_Asset_3` and
`<lambda>`. None of these names appear in the package source. The
documentation notes that reading this "does take some experience" and
points to `dpm damlc inspect`.

Per its documentation, it does not "take time into account that is spent
outside of pure interpretation, e.g., time needed to fetch a contract from
the database". It requires participant configuration and Sandbox, and it
profiles local runs only, not committed transactions.

Not available from any tool today:

- transaction size per node, template, or payload field, or its traffic
  cost
- a comparison between two package versions or two runs
- a CI check on size, node count, or time
- profiles of committed transactions
- time spent outside interpretation

The Daml profiler measures interpretation time. `dpm profile` measures
transaction cost, covers the items above, and post-processes the Daml
profiler's output instead of replacing it.

---

## RFP Alignment

This proposal responds to two RFPs in the Foundation's
[2026-2028 roadmap](../2026-2028-strategic-roadmap.md).

**DPM Components and Extension Ecosystem (RFP 19)** asks for reusable DPM
components, naming fee estimators, observability tools, and testing
utilities among the examples. `dpm profile` is all three.

**Integration into SDLCs (RFP 18)** asks for tooling that fits Canton
development into existing CI/CD pipelines and testing frameworks.
Milestone 3 delivers that as budget checks and a worked pipeline example.

The roadmap also records what Canton developers have been asking for.
Transaction Debugging and Observability was the lowest-rated area in the
Foundation's Q1 DevRel survey at 2.55 and remained tied for lowest in Q2 at
3.26. Simulation and dry-run tooling appears in both quarters as a repeated
request, described in the roadmap as the longest-standing unmet need in the
dataset. The prepare, profile, and diff loop in Milestone 1 is that dry-run,
applied to cost.

---

## Specification

### 1. Objective

Build a profiler that answers, for any Canton transaction or Daml Script
run:

- Which lines of Daml did a run execute, and where did its wall-clock go?
- Which nodes, templates, and payload fields account for the transaction's
  size, and what does that size cost in traffic?
- How did cost and time change between two versions of a package, or two
  runs of a workflow?
- Did this change make anything worse, in a form a CI job can act on?

The result is a CLI, a versioned profile report format, and documentation,
all open source.

### 2. Motivation

Canton transactions are slow and expensive in a way that is measurable and
billable. Sequenced traffic is metered in bytes and paid for in Canton Coin,
and the network is formalizing exactly who pays for it (see the approved
[user-paid traffic accounting](./2026-07-DA-user-paid-traffic-accounting.md)
proposal). Latency matters too: interpretation and confirmation time bound
how fast a workflow can run.

A developer who wants to make a workflow cheaper today has no way to find
out where the cost is. The traffic total is available before submission,
and the interpretation profile is available in Daml-LF identifiers after
configuring Sandbox, and neither points at a line of code. Optimization
becomes guesswork: change the model, measure the total again, compare two
numbers by hand.

Measured on a live participant: a contract with two parties, a text field,
and an integer serializes to 918 bytes. The field values account for 9
bytes. The rest is party ids and protocol framing. This ratio is not
visible from the source.

### 3. Implementation Mechanics

#### A. Which lines a Daml run executed

This workstream addresses the Daml profiler's missing source attribution
and its uncounted non-interpretation time. Inputs are the profiler's
speedscope files and the runtime trace Daml Script writes.

`dpm profile run` reads that trace and reports which source lines ran, and
how often. This part is built:

```
DPM execution profile
  measure:      executions per source location
  steps:        9  (7 located, 2 without a source location)

Hot spots
     2x  Asset.daml:9       Asset:Asset            (created x2)
     1x  Asset.daml:18:5    Asset:Asset.Transfer   (exercised x1)
```

The trace carries source coordinates for its own script steps, so some lines
resolve with nothing else supplied. `daml-debug-info/v1` adds the template
creates and choice exercises the trace cannot place on its own, which on
that run takes four located steps to seven. The metadata is the subject of a
separate Walnut proposal,
[PR #743](https://github.com/canton-foundation/canton-dev-fund/pull/743).
This proposal does not require it to be funded.

What this reports today is how often a line ran, not how long it took. The
trace carries no timestamps, and adding them is a change to the emitter in
our own fork, which Milestone 2 makes and carries upstream with the rest of
the compiler work.

That gives wall-clock per step: the interval spanning a submission,
including contract fetch, validation, and sequencing. This is coarser than
per-expression timing and covers the time the Daml profiler excludes.
Output stays speedscope-compatible.

```bash
daml script --debug-trace-file run.jsonl --script Test:allTests
dpm profile run run.jsonl --debug-info pkg.debug-info.json
```

#### B. Cost Profile

Profile the size and estimated traffic cost of a transaction using only
existing APIs. The input is either a prepared transaction from
`InteractiveSubmissionService.PrepareSubmission`, which yields the exact
serialized transaction before any traffic is spent, or a committed update
from `UpdateService.GetUpdateById`. The output is a size tree: per node, per
template, and per payload field, with node counts, serialized sizes, and an
estimated traffic cost at a configurable price per byte. Two profiles can be
diffed, so a developer can compare package versions or candidate designs
before submitting anything.

```bash
dpm profile tx --prepared prepared.json
dpm profile tx <update-id> --submitter <url> --read-as <party>
dpm profile diff before.json after.json
```

This is built and runs today. Profiling a real `Asset` create prepared on a
local Canton participant prints:

```
DPM cost profile
  subject:      dpm-trace-prepare-1934d5a7515d
  origin:       prepared-command
  nodes:        1
  modeled:      229 B  (envelope 26 B, payload 203 B)
  measured:     918 B  (from preparedTransaction)
  unattributed: 689 B  (protocol framing, not attributable to a field)
  Canton says:  0  (totalTrafficCostEstimation, from the participant)
  est. cost:    9.18e-05 CC  (estimate at 1e-07 CC/byte, basis measuredWireBytes)

By template
       229 B    1 node(s)  Asset:Asset

Largest payload fields
        77 B  x1   Asset:Asset.issuer
        77 B  x1   Asset:Asset.owner
         6 B  x1   Asset:Asset.name
         3 B  x1   Asset:Asset.quantity
```

Of the 918 measured bytes, the model attributes 229 to the command. The
two party ids account for 154 and the field values `GOLD` and `100` for 9,
with the remainder in field names and the command envelope. The other 689
bytes are protocol framing. Measured and modeled bytes are reported
separately and never summed.

`PrepareSubmission` also returns the participant's own `costEstimation`,
which the profiler prints as the authoritative traffic figure. On a local
node without traffic pricing it reads zero.

This part needs no compiler, engine, node, or protocol changes.

#### C. Built for CI

The CLI is designed for CI:

- `--format json` on every command, carrying the same versioned schema the
  aggregate report uses, so nothing has to parse human-oriented text.
- Exit codes that carry meaning. Zero for pass, a distinct non-zero code for
  a budget breach, and separate codes for tool failure, so a pipeline can
  tell a broken run from a failing check.
- No daemon, no interactive prompts, no configuration file to author. The
  tool runs headless against a local Canton participant in a container or an
  authorized remote one.
- A baseline profile committed to the repository, so `dpm profile diff`
  reports what a pull request changed in bytes, and in milliseconds once
  the trace carries timestamps.

The diff and the budget check are built. Adding one field to a template:

```
DPM cost diff
  before:       update-000000000001
  after:        update-000000000002
  nodes:        0
  modeled:      +456 B
  envelope:     0 B
  payload:      +456 B

Changed templates
      +456 B  Asset:Asset
```

Same node count, same envelope, 456 bytes more payload, attributed to one
template. Against a budget file that becomes a build failure:

```
$ dpm profile check after.json --budget budgets.json
budget check failed: 2 breach(es)
  maxTotalBytes: transaction models 2330 B, over the 2000 B budget
  maxTemplateBytes: Asset:Asset models 1904 B, over its 1600 B budget
$ echo $?
2
```

#### D. Suite Profiling and Budgets

Aggregate profiles across a Daml Script test suite into one report: cost and
time per template and per choice, with a versioned JSON schema so other
tools can consume it. On top of that, budgets: a check that fails when a
choice's transaction grows beyond a configured size, node count, or time
threshold, with a diff against a committed baseline. Coverage tooling such
as [DamlCov](https://github.com/canton-foundation/canton-dev-fund/pull/323)
shows what a test suite executes. The profiler shows what that execution
costs, and the two reports join on the same template and choice identifiers.

### 4. Architectural Alignment

The profiler works through authorized participant endpoints and respects
participant-scoped visibility. Profiles are derived only from data the
requesting context is entitled to see: its own prepared transactions,
updates, and local script runs. The default report contains identifiers,
counts, sizes, and timings, not payload values. Reports that embed values
for drill-down are marked sensitive, the same discipline the dpm family
already applies to trace files.

It consumes `daml-debug-info/v1` metadata when present and degrades
gracefully without it, which makes it another independent consumer of the
shared metadata format alongside `dpm trace`, `dpm debug`, and coverage
tooling. It requires no Canton protocol changes, no node changes, and no
interpreter changes. The timing work adds timestamps to the Daml Script
runtime trace and asks nothing of Canton.

Traffic prices are configurable inputs. The tool reports bytes and node
counts as ground truth and labels every Canton Coin figure as an estimate,
so it stays useful whatever the fee schedule does.

### 5. Backward Compatibility

The tool is additive. It changes no existing Daml applications, Canton
nodes, APIs, or package workflows. The profile report schema is versioned
with the same rules as the rest of the family: consumers ignore unknown
fields within a supported major version and reject unsupported majors.

---

## Milestones and Deliverables

Milestone 1 depends on nothing beyond existing Ledger APIs. Milestone 2
resolves more of a run when the debug metadata format is available, and
reports the locations the runtime trace carries itself when it is not.

### Milestone 1: Transaction Cost Profile

**Estimated Delivery:** 5 weeks from start<br>
**Focus:** Report transaction size and estimated fees per component.

**Deliverables:**

- `dpm profile tx` for prepared transactions and committed updates, with the
  size tree broken down by node, template, and payload field.
- Estimated traffic cost at a configurable price per byte, labeled as an
  estimate.
- `dpm profile diff` comparing two cost profiles.
- JSON output and documented exit codes on every command.
- Documentation with a local Canton example and an authorized remote
  participant example.

**Acceptance Criteria:**

- A developer can profile a prepared transaction before submitting it,
  identify the largest cost contributor, and compare two package versions.
- The same commands work against a local Canton participant and an
  authorized remote endpoint.
- Every command runs non-interactively and returns machine-readable output.

### Milestone 2: Profiles a developer can read

**Estimated Delivery:** 5 weeks after Milestone 1 acceptance<br>
**Focus:** Put the interpretation profile on source lines, and measure the
time it leaves out.

**Deliverables:**

- `dpm profile run` reading the profiles the SDK profiler writes and
  putting them on source through `daml-debug-info/v1`: frames that name a
  template or choice resolve to their file and line, and compiler-generated
  helpers such as `Asset:$csignatory` are attributed to the template that
  produced them, through the call stack they appear in.
- Timestamps in the Daml Script runtime debug trace, submitted to
  `digital-asset/daml` with the rest of our compiler work.
- Wall-clock per step from that trace, covering the ledger round trip the
  interpretation profile excludes, alongside the execution counts reported
  today.
- Speedscope-compatible output preserved, so profiles still open in the
  viewer developers are pointed at, plus a per-choice and per-template table.
- A written account of what each measure captures and at what granularity,
  so developers read the numbers correctly.

**Acceptance Criteria:**

- A representative script run produces a profile whose frames carry Daml
  source locations, with compiler-generated helpers attributed to the
  template that produced them, from a single command on a clean checkout,
  with no participant configuration written by hand.
- The timing table identifies the slowest step in a representative
  multi-submission workflow, and the number accounts for time spent outside
  interpretation.
- With metadata absent the run still reports every location the trace itself
  carries, rather than failing.
- The trace change is open as a pull request against `digital-asset/daml`,
  and we help the merging process by actively addressing review comments.

### Milestone 3: Suite Profiling and CI Budgets

**Estimated Delivery:** 4 weeks after Milestone 2 acceptance<br>
**Focus:** Fail CI on cost and time regressions.

**Deliverables:**

- Aggregate profile report across a Daml Script test suite, with a
  versioned JSON schema documented for other tools.
- Budget checks that fail when a choice exceeds configured size, node
  count, or time thresholds, with a diff against a committed baseline.
- A worked reference pipeline, published as a GitHub Actions workflow and
  documented so it can be ported to other CI systems.

**Acceptance Criteria:**

- A representative test suite produces one aggregate report.
- An intentionally introduced regression trips its budget in CI with a
  non-zero exit code and output that names the choice and the threshold.
- The reference pipeline runs end to end on a clean checkout, and a
  developer can adopt it by copying it and setting thresholds.

### Milestone 4: Adoption and Ecosystem Validation

**Estimated Delivery:** 4 weeks after Milestone 3 acceptance<br>
**Focus:** Public release and validation with Canton teams.

**Deliverables:**

- Public release with installation instructions and a getting-started
  guide.
- A walkthrough that finds a cost hotspot in a realistic package, fixes it,
  and shows the before-and-after diff.
- Feedback from at least three Canton developers or organizations using the
  profiler on their own workflows.
- Follow-up issues filed for anything the runtime trace or the debug metadata
  format should add, raised with the maintainers who own them.

**Acceptance Criteria:**

- At least three independent organizations run the profiler on their own
  projects.
- At least two of them confirm a finding they did not previously know: a
  cost hotspot, a time hotspot, or a regression caught by a budget.
- At least one of them runs the budget check in their own CI.
- Any critical issue blocking the documented happy path is fixed or
  documented with a workaround.

---

## Acceptance Criteria

The Tech & Ops Committee can evaluate completion based on:

- Cost profiles, and execution profiles that name the source lines a run
  executed, against representative Canton examples, installable in a clean
  environment.
- A versioned, documented profile report schema that other tools can
  consume.
- Budget checks that catch regressions in CI, with a reference pipeline
  anyone can copy.
- Independent ecosystem feedback confirming real findings, per Milestone 4.

---

## Funding

**Total Funding Request:** 1,400,000 Canton Coin (CC)

Each payment is tied to something the committee can inspect, not to
internal activity. Milestone 2 is the smallest amount because its
execution profile already runs: what remains is the frame attribution, the
timestamps in the runtime trace, and the review that carries them upstream.

### Payment Breakdown by Milestone

- Milestone 1, Transaction Cost Profile: 400,000 CC upon committee
  acceptance.
- Milestone 2, Profiles a developer can read: 250,000 CC upon committee
  acceptance.
- Milestone 3, Suite Profiling and CI Budgets: 350,000 CC upon committee
  acceptance.
- Milestone 4, Adoption and Ecosystem Validation: 400,000 CC upon committee
  acceptance and ecosystem validation.

### Volatility Stipulation

The expected duration is under six months. Should the project timeline
extend beyond six months due to Committee-requested scope changes, remaining
milestones must be renegotiated to account for significant USD/CC price
volatility.

---

## Co-Marketing

Walnut can work with the Canton Foundation on a technical post about the
Canton cost model from a developer's seat, a demo video of finding and
fixing a traffic hotspot, and documentation for teams adopting budgets in
CI.

---

## Rationale

### Why not just use the Daml profiler?

We do. This proposal post-processes its output. Its frames are Daml-LF
identifiers, it measures pure interpretation only, and it has no notion of
transaction size, diffing, or CI. This proposal asks nothing of the
engine.

### Why does source mapping need another proposal?

Nothing in the DAR today maps a Daml-LF identifier back to a file and a
line. `daml-debug-info/v1`
([PR #743](https://github.com/canton-foundation/canton-dev-fund/pull/743))
defines that mapping. The two proposals are independent, and each is useful
without the other.

### Why profile cost before time?

Nothing measures cost per component today, and measuring it needs only
existing APIs. Traffic is metered and paid in Canton Coin, so a size
regression recurs on every submission. Milestone 2 then adds wall-clock,
which includes the database and sequencing time the Daml profiler
excludes.

### Why prepared transactions?

`PrepareSubmission` yields the exact serialized transaction without
committing or spending traffic. That turns cost optimization into a dry-run
loop: prepare, profile, change the model, prepare again, diff. The approved
`dpm trace` proposal already exercises the prepare flow, so the plumbing is
proven.

### Why a versioned report schema instead of just terminal output?

Budgets and CI need a stable format, and other tools such as dashboards,
coverage reports, and review bots should be able to consume profiles without
parsing human-oriented text.

### Why the dpm family?

`dpm trace` inspects a transaction, `dpm debug` steps through it, and
`dpm profile` measures it. They install together and take the same
connection flags, and each reads what the others produce: `dpm profile`
profiles a trace artifact `dpm trace` exported, and resolves source through
the same metadata `dpm debug` reads.

---

## Risks and Mitigations

- **Wall-clock depends on a change we make:** rewriting profile frames onto
  source lines needs nothing from anyone, and execution counts work today.
  Wall-clock per step needs timestamps added to the runtime trace in our fork
  and carried upstream, so it carries the same review risk as the rest of the
  compiler work. If it lands late, the readable profile and the counts still
  ship.
- **Metadata dependency:** source mapping is best with `daml-debug-info/v1`.
  Every milestone here works without it: the runtime trace carries source
  coordinates for its own steps, and the cost profile falls back to template
  and choice names, so this proposal does not depend on another being
  funded.
- **Estimate accuracy:** traffic prices change. The tool treats bytes and
  node counts as ground truth and labels every Canton Coin figure as an
  estimate at a stated price.
- **Overlap with other tooling:** coverage (DamlCov) and tracing (dpm trace)
  answer different questions and share identifiers with the profiler, so the
  tools compose instead of competing.

---

## Maintenance

The profiler is open source under Apache-2.0. Walnut maintains it alongside
the `dpm trace` and `dpm debug` integrations, so one team and one release
process cover the whole family. For twelve months following Milestone 4 we
keep it working against each Daml SDK and Canton release published in that
window, triage external issues and pull requests, and hold the profile
report schema stable under the same versioning discipline as the rest of the
family. It is absorbed into the amounts above and does not carry its own
milestone. Everything is open, so stewardship can transfer to the Foundation
or another maintainer if it ever needs to.

---

## About the Team

Walnut has spent four years building debugging and observability tooling for
blockchains, and we build it with the teams behind the platforms themselves:

- **Canton.** The Development Fund approved our
  [DPM Trace Transaction Visualization](./2026-08-Walnut-dpm-trace-visualization.md)
  proposal and we are building `dpm trace` now.
- **Ethereum Foundation / Argot.** We own debug info generation in
  [`solc`](https://github.com/argotorg/solidity), the official Solidity
  compiler. One-year partnership, being extended.
- **Starkware / Starknet.** We build the
  [Walnut Starknet Debugger](https://walnut.dev/), covering debug info
  generation, tracing, simulation, verification, and the hosted debugger
  itself. Three-year partnership.
- **Miden.** We build the compiler and the debugger for
  [Miden](https://github.com/0xMiden).
- **Tempo.** We work with them on
  [solar](https://github.com/paradigmxyz/solar), a Solidity compiler written
  in Rust.
- **Arbitrum / Offchain Labs.**
  [StylusDB](https://github.com/OffchainLabs/stylus-sdk-rs/blob/main/cargo-stylus/docs/StylusDebugger.md),
  the official debugger for Stylus.

We have turned compiled-code profiles into source-level ones before, on
other compilers.

---

## What already exists

Milestone 1 and the execution profile are prototyped in
[walnuthq/dpm-trace](https://github.com/walnuthq/dpm-trace/tree/feature/profiler)
on the `feature/profiler` branch. The samples above are tool output.
Working today:

- `dpm profile tx` over a committed update, an exported trace artifact, or a
  prepared transaction, with the size tree, the per-template rollup, and the
  ranked payload fields.
- `dpm profile diff` between two profiles, separating a node-count change
  from an envelope change from a payload change.
- `dpm profile check` against a budget file, exiting 0 when it fits, 2 on a
  breach, and 1 on a tool error, so a pipeline can tell a failed check from a
  broken run.
- Canton's own `costEstimation` from `PrepareSubmission`, reported as the
  authority on traffic cost, alongside the unattributed remainder.
- `dpm profile run`, reporting executions per source location from a Daml
  Script runtime trace, on the Canton edition that ships with the SDK.
- A versioned `dpm-profile/v1` JSON document behind `--json` and `--export`.
- Tests covering the render path, the sizing invariants, the diff, the
  budgets, and every exit code.

The numbers above come from an integration test that boots a real Canton,
prepares a transaction, and profiles it, so the measured wire size and
Canton's own estimate are what the participant returned. It runs in
`daml-tests/itests/asset-profile-cost.test`.

Reporting rules: a measured wire size appears only when the artifact
carries one (a prepared transaction does, a JSON-fetched update does not),
modeled bytes are never added to measured bytes, and unattributed bytes are
printed as unattributed. The test node has no traffic pricing, so Canton
Coin figures in the samples are estimates at the stated price.

`dpm trace` already speaks to a participant through the Ledger API and
exercises the `PrepareSubmission` flow that Milestone 1 profiles, so the cost
profile builds on plumbing that works today. `dpm profile run` reports
execution counts per source line from the Daml Script runtime trace, which is
the base Milestone 2 adds timing to.

---

## References

- Foundation roadmap and RFPs:
  [2026-2028 Roadmap and Request for Proposals](../2026-2028-strategic-roadmap.md)
- Existing SDK profiler:
  [Daml profiler documentation](https://docs.canton.network/sdks-tools/development-tools/daml-profiler)
- Approved consumer proposal:
  [DPM Trace Transaction Visualization](./2026-08-Walnut-dpm-trace-visualization.md)
- Traffic cost context:
  [Enabling User-Paid Traffic Accounting on Canton Network](./2026-07-DA-user-paid-traffic-accounting.md)
- Debug metadata format (Walnut, in submission):
  [Versioned Debug Info Metadata for Daml, PR #743](https://github.com/canton-foundation/canton-dev-fund/pull/743)
- Related coverage tooling:
  [DamlCov proposal, PR #323](https://github.com/canton-foundation/canton-dev-fund/pull/323)
- dpm tooling:
  [github.com/walnuthq/dpm-trace](https://github.com/walnuthq/dpm-trace)
