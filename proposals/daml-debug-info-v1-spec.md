# `daml-debug-info/v1` — Draft Specification and Reference Implementation Notes

| Field | Value |
| --- | --- |
| Author | Walnut |
| Status | Draft / implementation-backed |
| Created | 2026-07-02 |
| Label | daml-tooling |
| Companion | `daml-versioned-debug-info-metadata-draft.md` |

This document specifies the first concrete version of the versioned Daml debug
metadata format proposed in *Versioned Debug Info Metadata for Daml*, together
with the runtime debug-trace contract that source-level debuggers (`dpm
debug`) consume. It is backed by a working reference implementation:

- **Producer:** `damlc build --experimental-debug-info`
  (branch `walnuthq/daml@feature/debug-info`). Metadata is derived from the
  compiled Daml-LF package, never from textual scanning of sources.
- **Runtime:** `daml script --debug-trace-file <file>` emits a JSONL debug
  trace of a script run (Daml Script runner extension, same branch).
- **Consumers:** `dpm trace` (source links, availability labels) and
  `dpm debug` (source-level stepping) in `walnuthq/dpm-trace`.

---

## 1. Artifact placement

A producer emits the metadata twice, with identical content:

1. **Sidecar:** next to the DAR, named `<dar-basename>.debug-info.json`
   (e.g. `.daml/dist/asset-demo-1.0.0.debug-info.json`).
2. **DAR member:** `META-INF/daml-debug-info/<package-id>.json`.
   Existing DAR consumers ignore unknown `META-INF` members, so embedding is
   backward compatible. (Consumers should also accept the legacy prototype
   member `META-INF/daml-debug-info.json`.)

## 2. Top-level document

```json
{
  "schema": "daml-debug-info/v1",
  "producer": { ... },
  "package": { ... },
  "sources": [ ... ],
  "spans": [ ... ],
  "symbols": [ ... ],
  "valueSlots": [ ... ],
  "steps": [ ... ],
  "compatibility": { "minConsumerSchema": "daml-debug-info/v1",
                     "ignoreUnknownFields": true }
}
```

- `schema` is the major-versioned identifier. Consumers MUST reject documents
  whose major version they do not support and MUST ignore unknown fields
  anywhere in a document whose major version they do support.
- `producer`: `tool`, `version` (SDK/compiler version), `buildMode`
  (`experimental` until the schema is declared stable), `features` — a list of
  feature flags; v1 reference emits
  `["source-spans", "symbols", "lf-refs", "value-slots", "steps"]`.
- `package`: `packageId` (hex, as computed for the main DALF), `name`,
  `version`, `lfVersion` (e.g. `"2.1"`), `sdkVersion`.

All line/column positions in the document are **1-based** (Daml-LF stores
0-based source locations; producers convert).

## 3. `sources`

One entry per package module whose source file resolved under the package
source root at build time:

```json
{ "id": "src:Nested.Util", "module": "Nested.Util",
  "uri": "daml://asset-demo/Nested/Util.daml",
  "path": "Nested/Util.daml",
  "sha256": "<content hash>" }
```

- `path` is package-relative (relative to the `source:` root of `daml.yaml`).
  Producers MUST NOT emit absolute local paths; files outside the source root
  are omitted entirely.
- `sha256` lets consumers verify that a local checkout matches the compiled
  package before trusting spans (stale-source detection).
- `module` gives consumers the module→file mapping directly (Daml-LF does not
  serialize module source paths).

## 4. `spans`

```json
{ "id": "span:Asset:Asset", "source": "src:Asset",
  "kind": "template-definition",
  "start": {"line": 5, "column": 10}, "end": {"line": 10, "column": 17} }
```

Kinds emitted by the reference producer: `template-definition`,
`choice-definition`, `interface-definition`, `interface-method-definition`,
`exception-definition`, `data-type-definition`, `value-definition`, and
`<slot-kind>-expression` for located value-slot expressions (e.g.
`signatories-expression`). Spans are only emitted when they provably belong to
the module's own source file; cross-module inlined spans are dropped rather
than mislabeled.

## 5. `symbols`

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
- `qualifiedName` conventions: `Module:Entity` (templates, interfaces, types,
  values), `Module:Entity.Choice` (choices) — matching the identifiers that
  appear in Ledger API events, completions, and error messages.
- `lfRef` is the Daml-LF reference where one exists; tools can join it against
  transaction data without string heuristics.
- `type` (optional) is the rendered LF type for values, methods, and slots.
- Compiler-generated definitions (names containing `$`) are excluded.
- Data types that merely back a template payload, exception, or choice
  argument are not repeated as standalone symbols; their fields surface as
  value slots of the owning symbol.

## 6. `valueSlots` and availability

```json
{ "id": "slot:Asset:Asset:Transfer:argument:newOwner",
  "symbol": "sym:Asset:Asset:Transfer", "name": "newOwner",
  "kind": "choice-argument-field", "type": "Party",
  "availability": "transaction-visible" }
```

Slot kinds: `contract-payload-field`, `choice-argument`,
`choice-argument-field`, `choice-result`, `self-contract-id`,
`choice-controllers`, `choice-observers`, `choice-authorizers`,
`signatories`, `observers`, `precondition`, `contract-key`,
`key-maintainers`, `interface-view`, `exception-message`.

`availability` is the privacy-honesty contract:

- `transaction-visible` — populatable from participant-visible transaction
  data by a party entitled to see the event (payload fields, choice arguments
  and results, signatories, observers, acting parties, keys, interface
  views).
- `interpreter-only` — only observable with interpreter/runtime support
  (preconditions, exception messages, authorizers, intermediate expressions).
  Trace-only tools MUST NOT claim these from transaction data; they should
  show the slot and label the value as not captured.

Reserved for future use: `source-only`, `not-tracked`.

## 7. `steps`

Deterministic evaluation step descriptors: the source spans of the compiled
Daml-LF expression's location nodes in pre-order, de-duplicated, per owning
symbol (choice bodies and top-level values, which include Daml Script
entry points):

```json
{ "id": "step:Asset:test_transfer:0", "symbol": "sym:Asset:test_transfer",
  "index": 0, "source": "src:Asset",
  "start": {"line": 30, "column": 3}, "end": {"line": 30, "column": 40} }
```

Step ids are stable for a given package id, so runtime events and breakpoints
can reference them portably. Runtime debug events that carry source locations
join against these spans by `(module, start, end)`.

## 8. Runtime debug-trace contract (JSONL)

`daml script --debug-trace-file <file>` writes one JSON object per line.
Consumers MUST ignore unknown `event` kinds and unknown fields.

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
"endCol"}` with 1-based positions (the runtime's 0-based locations are
normalized by the emitter), matching `daml-debug-info/v1` spans. `location`
may be `null` when the runtime has no location.

Values in `created`/`exercised` events come from a **local** script run
against the developer's own ledger (IDE ledger or an authorized participant);
the trace never contains data the submitting context could not see. Local
variables and intermediate expression values are **not** captured — consumers
must present them as `interpreter-only`, not invent them.

## 9. Runtime follow-up: interpreter location hook

The reference runtime implementation hooks the Daml Script layer: script
questions (with Daml call-stack locations), submissions, ledger events, and
`trace`/`debug` output. Expression-level stepping *within* a choice body needs
one additional hook in the Speedy interpreter (Canton repository), for which
this spec reserves the shape:

```scala
trait MachineLogger {                    // existing trait, existing methods
  def trace(message: String, location: Option[Location]): Unit
  def warn(message: String, location: Option[Location]): Unit
  def onLocation(location: Location): Unit = ()   // NEW, default no-op
}
```

called from the machine's `SELocation` handling, gated so the disabled cost is
one branch. With that hook, the same JSONL contract gains
`{"event":"step","location":{LOC}}` lines that join against `steps` in the
metadata. This is intentionally split out as follow-up work, per the
proposal's scoping.

## 10. Compiler changes in the reference implementation

Beyond the emitter itself, two compiler fixes were needed to make choice-level
debugging possible, both in the GHC→LF conversion:

1. `chcLocation` was hard-coded to `Nothing`; it is now populated from the
   desugared choice binder's source span, so choices carry
   `choice-definition` spans.
2. Choice update expressions had **all** source locations stripped
   (`removeLocations`) as a side effect of a structural match; locations are
   now preserved in the choice body (they are evaluation-transparent), which
   both improves runtime stack traces and provides `steps` for choices.

## 11. Validation rules for consumers

- Reject unsupported major schema versions; ignore unknown fields otherwise.
- Verify `package.packageId` against the DAR/DALF when both are available.
- Verify source `sha256` before trusting spans; degrade to span-less symbol
  information (module/qualified-name resolution) on mismatch.
- Reject absolute source paths (strict mode) or warn (lenient mode).
- Never populate an `interpreter-only` slot from transaction data.
