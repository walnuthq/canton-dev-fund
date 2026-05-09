# Review: DPM Trace and Source-Aware Debugging Proposal

**Reviewed file:** [dpm-trace-source-aware-debugging.md](dpm-trace-source-aware-debugging.md)
**Reviewer:** Roman Mazur
**Date:** 2026-05-09

---

**Verdict (moderate confidence):** The proposal is well-structured and the team is credible, but it is materially over-promising on Milestones 2 and 3 and under-specifying the hardest technical risks. The pricing for M1 is fat, M2/M3 are aggressive on time. The most fundamental problem — Canton's privacy model makes a "Tenderly-like" debugger structurally weaker than what Ethereum developers expect, and the proposal does not honestly confront that limitation. I would push back on it before approving.

---

## 1. The privacy collision is hand-waved (high confidence, biggest issue)

The proposal's single most important sentence is buried in §1: "The output is explicitly labeled as a participant-visible projection." That label is not a feature — it is an admission that this tool cannot do what Tenderly does, and the rest of the document does not internalize that.

What Tenderly/Foundry actually deliver is the ability to replay the *entire* execution of a transaction across all contracts and inspect any state at any step. The reason this is valuable is precisely because Ethereum has a single global view. Canton, by design, does not. A debugger that can only show one party's projection is structurally limited for the most common debugging case in a multi-party application: "why did the counterparty's choice fail / produce the wrong result?"

The proposal mentions multi-participant transactions in §3 ("It treats multi-participant transactions as multiple authorized projections, not as one globally visible transaction") but then never explains what the debugging UX is in that case. Do you open N debuggers, one per authorized projection, and mentally stitch? Do you fetch from N participants if you have credentials for all of them? Is there a multi-projection synced replay mode? This is the central design question and it is absent.

**Required fix:** Add a section called "Cross-participant debugging" that describes exactly what the developer experience is for a transaction that touched three participants, and is honest about which problems this tool *cannot* solve. Failing to do this risks shipping a tool that wins demos but disappoints in production debugging sessions, which is exactly the credibility risk the Canton ecosystem can least afford right now.

## 2. Milestone 2 is two milestones masquerading as one (high confidence)

M2 bundles, in 5 weeks:

- Immediate inspector for committed updates
- Immediate interactive debugger for local Daml Script/test workflows
- Record/bundle for both committed updates and local workflows
- Replay
- Portable bundle format (undefined)
- REPL-style commands
- Breakpoints on event id, template, choice, **and source location**
- Source snippet display
- Variable + contract inspector
- State-diff view

The committed-update inspector is a paginated tree-walker over data you already fetched in M1 — that's a few days' work. The local-workflow interactive debugger is qualitatively different work: you have to hook into the Daml Script runner or the Daml-LF interpreter, and that requires either Digital Asset cooperation, a fork, or living within whatever tracing surface `Debug.trace`/`debug` already provides. The proposal does not say which of these three roads it is taking, and the choice dominates the schedule.

Similarly, "breakpoints on source location" in M2 contradicts the dependency on M3, which is where source spans get emitted by the compiler. M2 says "where metadata exists" — but no debug-info-emitting compiler exists yet at M2 time, so in practice source-location breakpoints are vapor in M2 and only become real at the end of M3.

**Required fix:** Split M2 into M2a (committed-update inspector + bundle format + replay) and M2b (live local Daml Script/test debugger). Be explicit about whether the live debugger requires `damlc`/Daml-LF runtime changes; if so, M2b should depend on M3, not the other way around.

## 3. Milestone 3 has a hard external dependency that is not priced in (high confidence)

`damlc build --debug-info` is the headline deliverable. `damlc` is owned by Digital Asset. Three outcomes are possible:

1. DA accepts upstream PRs to add the flag and emit the schema. Best case, but governed by their roadmap, not Walnut's. There is no LOI / champion / co-sign in the proposal indicating DA has agreed.
2. Walnut maintains a fork of `damlc`. This is open-source-allowed but operationally heavy and creates split-brain risk for users — which compiler do they install?
3. Walnut emits a sidecar via static analysis without modifying `damlc`. This is feasible for module/template/choice spans but breaks down for "expression spans where feasible" and is borderline impossible for "deterministic debug step IDs that map source spans to Daml-LF/runtime execution points," which require either compiler cooperation or interpreter cooperation, ideally both.

The proposal hedges with "subject to review with Daml maintainers" and "Runtime integration note describing what Daml-LF/interpreter debug events are needed" — i.e., the hardest part of the work (interpreter-level debug events) is documented as a *note*, not delivered. That is the actual prerequisite for "real" stepping. Without it, M3's deliverable is closer to "a JSON schema and a static analyzer" than "DWARF for Daml."

**Required fix:** Either secure (and quote in the proposal) explicit DA buy-in for the `damlc` flag and the Daml-LF debug events, or rescope M3 to "static debug-info sidecar + schema proposal + reference implementation," and stop calling it a debug-info standard until DA co-owns it. A "standard" issued unilaterally is just a JSON file.

## 4. The "trace bundle" format is undefined (moderate confidence)

The proposal mentions trace bundles ~10 times and never specifies them. Things a reviewer needs to see:

- Wire format (JSON? CBOR? protobuf with .proto files?)
- Versioning / forward-compat strategy
- Hashing / integrity (so a bundle can't be forged or tampered with)
- Embedded metadata vs. external references (does the bundle include the DAR / debug-info, or reference it by hash?)
- Privacy: bundle redaction. If I'm Bank A and I record a script trace, the bundle may contain customer PII or contract terms — under what conditions can I share it with a vendor like Walnut for support?

The redaction question is non-trivial and is exactly the kind of thing institutional Canton users will block adoption on. This belongs in the proposal, not in an implementation detail conversation later.

## 5. Pricing analysis (low–moderate confidence on CC USD)

I do not have a high-confidence number for Canton Coin's USD price as of 2026-05-09 — I have not verified it and will not invent one. So I'll evaluate the structure rather than the absolute amount.

Per-milestone effort:

- M1 (2 weeks, 150k CC): wrapping `GetUpdateById` + JSON Ledger API into a CLI with pretty-printing. ~1 engineer-month at most. This is the thinnest deliverable in the proposal and is paid 20% of the core budget.
- M2 (5 weeks, 350k CC): the biggest scope, the most risk, paid 47% of the budget. Internally proportional, but absolute scope is too large for 5 weeks (see §2).
- M3 (4 weeks, 250k CC): blocked on DA. Paying for a deliverable whose acceptance is partly outside the contractor's control is a governance smell.
- M4 (4 weeks, 200k CC, optional): a registry service. 4 weeks for a service with verification, hosted instance, self-hosted guide, and a public sample is aggressive.

Structural concern: payment is "upon committee acceptance" per milestone, but acceptance criteria contain subjective bars ("useful for committed updates," "feedback from … developers"). This invites disputes. Tighten to objective tests (e.g., "renders X event types correctly on Y reference DARs," "produces non-empty source links on Z packages built with `--debug-info`").

## 6. Existing primitives are slightly oversold (moderate confidence)

§1 claims "There is no interactive debugger for local Daml Script/test workflows." This is mostly true in the strict stepping sense, but understates what is already there: Daml Studio's script results UI shows transaction trees and ledger state at every commit, and `Debug.trace`/`Debug.traceRaw` plus the Script trace facilities give a structured-print debugging loop. Calling that "no interactive debugger" is true literally; calling it "no debugging story today" — which is roughly what the Motivation section implies — is unfair and weakens credibility with reviewers who actually use the SDK.

Be precise: today, Canton developers have *tree inspection* (Daml Studio) and *committed-update fetching* (Ledger API), but no *unified, source-aware, post-hoc* debugger. That is the real gap.

## 7. Reproducibility of source verification is hard, and it's hidden in M4 (moderate-high confidence) — **DONE**

**Resolved 2026-05-09:** Optional Milestone 4 was removed from the funding request. The registry was relocated to a new `## Potential Follow-Ons` section in the proposal, with the build-reproducibility, hermeticity, partial-match, and pre-existing-DAR concerns called out as open questions to resolve before any future registry proposal.

---

Original critique:

M4's "Verification flow that checks source/debug-info correspondence to package id" is essentially: "given a source tree, rebuild the DAR and check that the package id matches." Daml/`damlc` builds are not guaranteed bit-reproducible across SDK versions, OS, or build environments unless very carefully constrained (this is the same hell EVM verifiers live in with solc nightlies). Sourcify-style pain.

The proposal treats this as a 4-week deliverable and does not discuss the build environment hermeticity required, the SDK pinning story, or the failure modes ("verified" / "metadata-only-match" / "unverified"). It also says nothing about how to handle pre-built DARs that were not built with `--debug-info` (which will be all of them at registry launch).

## 8. Smaller issues

- **No champion.** The proposal says "Need Champion." Without one, this won't pass committee. Acknowledged but unresolved.
- **§2 referenced "Canton developer survey"** but does not cite or link it. Either link it or drop the appeal to it.
- **References section is thin** — only three links and no citation of the Daml/Canton SDK version targeted. Acceptance Criteria says "current stable DPM/Daml SDK version at the time of development," which is fine but should also be pinned at start-of-work.
- **`dpm trace --debug-info <file>`** — passing a single file conflicts with the multi-package nature of Canton transactions; you'd want a directory or a manifest. Minor.
- **Apache-2.0 commitment** is good. Make sure registry data licensing is also addressed for M4 (CC0 metadata? Apache for code, separate license for the corpus of source uploads? Right now it's silent.).
- **No threat model for the registry.** Who can publish? Is there spam control? Squatting on package ids? Source poisoning? At minimum mention the model.
- **Co-Marketing section is fine** but is filler in a technical proposal. I'd cut it or move it to an appendix.
- **"Walnut" is referenced** without disclosure that this is the proposing party. It is signed at the top, so this is fine, but inside §Rationale "Walnut already has relevant experience" reads cleaner if framed as past credentials with links to the Starknet Debugger / ETHDebug / StylusDB work. Verifiable claims > self-praise.

## 9. What I would actually want to see before approving

1. A 2–4 paragraph cross-participant debugging story (§1).
2. M2 split into M2a/M2b, with the live local-workflow debugger explicitly conditional on `damlc`/Daml-LF cooperation.
3. A signed indication from Digital Asset (or at least a named maintainer) that they are willing to review and merge `damlc --debug-info`. Failing that, an alternative architecture that does not depend on `damlc` modifications, with reduced scope honestly stated.
4. A trace bundle spec sketch: format, versioning, integrity, redaction.
5. Objective acceptance tests per milestone replacing the subjective "useful" / "feedback" bars.
6. A named champion.

## 10. Should this be funded?

If you fix items 1–6 above: yes, conditionally. The team has the right pedigree (Starknet Debugger, ETHDebug, StylusDB are real and relevant, high confidence), the layered approach is correct, and the gap is genuine. Canton needs this category of tool.

If you ship the proposal as-is: I would vote no, or fund only M1 + the M3 schema work as a scoped pilot (~250k–300k CC range structurally), and gate M2 and the registry on demonstrated DA cooperation and a credible cross-participant story. Funding M2 at 350k CC right now buys a transaction-tree explorer with hopeful breakpoints, not a debugger.

Bottom line: the proposal is the *right* proposal, but it is selling Foundry while the architecture only supports a participant-scoped event browser with source links. Re-pitch it for what it actually is, tighten the deliverables, and it deserves to pass.
