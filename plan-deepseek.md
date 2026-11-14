# Weave — Cross-platform product specification framework

**Status:** POC plan  
**Edition:** 2026.1  
**Working name:** Weave (`weave` CLI, `.weave` files)

## 1. Decisions

- **Hybrid implementation:** the CLI deterministically generates the provable core; the LLM implements adapters, UI, and platform integration.
- **UI scope:** screens and flows are declared; layout and native idiom remain with the LLM.
- **CLI stack:** Rust workspace, single `weave` binary.
- **First proof:** habit tracker on iOS (SwiftUI) and Web (React).

## 2. Thesis

The product specification is the source of truth. Platform code is an implementation artifact, not a separate design.

The framework is three products:

1. **The spec language** — clean architecture is expressed as grammar: entities, use cases, adapters, frameworks.
2. **The verifier CLI** — parses, resolves, typechecks, enforces architecture and completeness, formats canonically, executes scenarios, and verifies implementations.
3. **The LLM contract** — task-scoped briefs, generated core code, traceability anchors, and conformance tests make drift detectable.

The language is intentionally not a general-purpose programming language. Its executable core is small enough for precise semantics and deterministic verification.

## 3. Clean architecture mapping

| Clean architecture | Weave construct | CLI produces |
|---|---|---|
| Entities | `entity`, `value`, `enum`, `error` | Domain models, validation, pure methods |
| Use cases | `port`, `usecase` | Port protocols, interactors, fakes, contract tests |
| Adapters | `adapter` | Stubs, wiring contracts |
| Frameworks | `screen`, `flow`, `target` | UI stubs, composition root, app shell |

The dependency rule is enforced by the checker:

- Entities may reference only entities.
- Use cases may reference entities, ports, and errors.
- Adapters may reference ports, entities, errors, and configuration.
- Screens may reference use cases, entities, flows, and targets.
- Targets may reference screens and adapters.

No outward references, no cycles between layers.

## 4. Language v0.1

### 4.1 File model

- One `main.weave` declares `edition`, `app`, and `targets`.
- Other `.weave` files contain declarations in a single project-wide namespace.
- Declarations are order-independent.
- The parser is hand-written recursive descent with Pratt expressions, spans, and recovery at newline or `}`.
- Comments use `#`; the formatter preserves leading and inline comments on declarations and statements.
- Canonical format: 2-space indentation, one statement per line, fixed clause order, single trailing newline.

### 4.2 Constructs and semantics

- **Closed world:** every name must resolve; every declaration is unique; no implicit behavior.
- **Types:** `Bool`, `Int`, `Text`, `Date`, `Instant`, `Duration`, `[T]`, `T?`, `Id<T>`, records, enums.
- **Entities and values:** immutable. Constructors use named fields. `.with(field: value)` produces updates.
- **Pure methods:** `fun` blocks inside entities/values hold domain behavior. No ports, mutation, or recursion.
- **Invariants:** expressions checked on construction and `.with()`.
- **Use cases:** the only effectful units. Declare inputs, output, `needs` ports, typed `errors`, a `do` body, and `scenario` blocks.
- **Ports:** async by default. Adapter failures surface as an implicit infrastructure error outside the domain error set.
- **Error handling:** `fail E`, `ensure cond else E`, `or fail E`, `if let`. Domain errors are typed and declared.
- **Executable core:** `let`/`var`, `if`/`else`, `for` over finite lists, `return`, `fail`, `ensure`, field access, constructors, built-ins, and port calls. No recursion, lambdas, classes, or escape hatches.
- **Screens:** declare route, loaded data, actions, navigation, states, and copy. Layout is deliberately unspecified.
- **Flows:** declare start screen, navigation edges, and reachability.
- **Targets:** declare framework profile only (`ui SwiftUI`, `ui React`). Paths and commands live in `weave.toml`.
- **Determinism:** no ambient time or randomness. Everything enters through ports, so scenarios are reproducible.

### 4.3 Example (abridged)

`CheckIn`, `Habits`, `Clock`, and `Ids` are declared elsewhere.

```weave
edition 2026.1
app Habits
target ios { ui SwiftUI }
target web { ui React }

enum Cadence { Daily, Weekdays }

entity Habit {
  id: Id<Habit>
  name: Text(1..60)
  cadence: Cadence = Daily
  archived: Bool = false

  fun dueOn(day: Date) -> Bool {
    if cadence == Weekdays { return day.weekday != Sat and day.weekday != Sun }
    return true
  }
}

error AlreadyDone(day: Date)

port CheckIns {
  on(habit: Id<Habit>, day: Date) -> CheckIn?
  save(checkIn: CheckIn)
}

usecase CheckInHabit(habit: Id<Habit>) -> CheckIn {
  needs Habits, CheckIns, Clock, Ids
  errors HabitNotFound, AlreadyDone

  do {
    h := Habits.byId(habit) or fail HabitNotFound(habit)
    day := Clock.today()
    if CheckIns.on(habit, day) != none { fail AlreadyDone(day) }
    c := CheckIn(id: Ids.new(), habit: habit, day: day)
    CheckIns.save(c)
    return c
  }

  scenario "duplicate check-in is rejected" {
    given {
      h := Habit(id: id1, name: "Read")
      Habits.byId(h.id) -> h
      Clock.today() -> 2026-11-13
      CheckIns.on(h.id, 2026-11-13) -> CheckIn(id: id2, habit: h.id, day: 2026-11-13)
    }
    when CheckInHabit(h.id)
    then fails AlreadyDone(2026-11-13)
    then no call CheckIns.save
  }
}
```

Scenarios stub port calls at the value level. The interpreter records every call and asserts results, errors, and side effects.

## 5. Verification ladder

| Level | Command | Proves |
|---|---|---|
| V0 syntax | `weave check` | Parses with spans and recoverable diagnostics |
| V1 names | `weave check` | Closed world; no invented references |
| V2 types | `weave check` | Arity, nullability, constraints, enum variants |
| V3 architecture | `weave check` | Layer direction, port/adapter conformance |
| V4 completeness | `weave check`, `lint` | Targets, ports, screens, errors, copy, reachability |
| V5 behavior | `weave run` | Scenarios execute; calls and failures match |
| V6 traceability | `weave verify` | Anchors, hashes, generated freshness, target tests |
| V7 conformance | `weave verify` | Same fixtures pass on every target |

Diagnostics use stable codes:

- `WV1xxx` syntax and lexing
- `WV2xxx` name resolution
- `WV3xxx` types
- `WV4xxx` architecture
- `WV5xxx` completeness
- `WV6xxx` scenario execution
- `WV7xxx` lint and budgets
- `WV8xxx` formatting
- `WV9xxx` verification and traceability

Every diagnostic carries file, span, line, column, severity, code, message, help text, and related locations.

## 6. CLI

### 6.1 Core commands

| Command | Purpose |
|---|---|
| `weave check` | Syntax, names, types, architecture, completeness |
| `weave lint` | Naming, dead specs, ambiguity, scenario coverage, token budgets |
| `weave fmt` | Canonical formatting; `--check` for CI |
| `weave run` | Execute scenarios in the reference interpreter |
| `weave gen` | Generate core, scaffolds, fakes, and conformance tests for a target |
| `weave brief` | Emit a task-scoped context pack for the LLM |
| `weave verify` | Anchors, hashes, stale files, target tests, coverage |
| `weave graph` | Layer and dependency graph |
| `weave stats` | Spec and brief token counts |
| `weave explain` | Explain an error code |
| `weave manual` | Compact language reference for LLMs |
| `weave schema` | Machine-readable grammar and diagnostic catalog |

All commands support `--json`. Exit codes: `0` success, `1` findings, `2` usage error, `3` internal error.

### 6.2 JSON diagnostics

```json
{
  "version": 1,
  "command": "check",
  "ok": false,
  "summary": { "errors": 1, "warnings": 2 },
  "diagnostics": [
    {
      "code": "WV2041",
      "severity": "error",
      "message": "unknown use case `Checkin`",
      "file": "spec/application.weave",
      "line": 12,
      "column": 4,
      "span": { "start": 218, "end": 225 },
      "help": "did you mean `CheckInHabit`?",
      "related": []
    }
  ]
}
```

### 6.3 Configuration

Spec metadata lives in `main.weave`. Tooling paths and commands live in `weave.toml`:

```toml
[targets.ios]
backend = "swift"
root = "targets/ios"
test = ["swift", "test", "--package-path", "targets/ios"]

[targets.web]
backend = "typescript"
root = "targets/web"
test = ["npm", "--prefix", "targets/web", "run", "test"]

[lint]
max_brief_tokens = 8000
max_line_width = 100
```

The checker requires every declared target to have a matching config section.

## 7. Repository layout

```
.
├── Cargo.toml
├── rust-toolchain.toml
├── crates/
│   ├── weave-syntax/    # spans, lexer, parser, AST, diagnostics
│   ├── weave-check/     # resolver, types, architecture, completeness, lint
│   ├── weave-fmt/       # canonical printer
│   ├── weave-interp/    # reference interpreter
│   ├── weave-gen/       # Swift/TS codegen, briefs, manifest, verify
│   └── weave-cli/       # commands, config, JSON output
├── docs/
│   ├── language.md
│   ├── semantics.md
│   ├── diagnostics.md
│   └── llm-protocol.md
├── examples/habits/
│   ├── weave.toml
│   ├── spec/
│   │   ├── main.weave
│   │   ├── domain.weave
│   │   ├── application.weave
│   │   ├── adapters.weave
│   │   └── surfaces.weave
│   └── targets/
│       ├── ios/
│       └── web/
└── tests/
    ├── corpus/
    └── e2e/
```

## 8. Hybrid ownership

| Artifact | Owner | Regeneration |
|---|---|---|
| Domain types, runtime helpers, ports, use case implementations, errors | CLI | Always overwritten; hash-guarded |
| In-memory fakes, conformance tests, fixtures | CLI | Always overwritten |
| Adapter stubs, screen stubs, composition root, app shell | CLI | Scaffolded once; never overwritten without `--force` |
| Adapter bodies, UI, platform integration | LLM | Anchored; verified |
| Spec and design decisions | Human | Source of truth |

Generated files carry a header with edition, target, and content hash:

```swift
// @weave-generated app=Habits edition=2026.1 target=ios hash=...
```

LLM-owned files carry implementation anchors:

```ts
// @weave impl adapter:LocalStore spec=...
```

`weave verify` rejects missing anchors, duplicate implementations, stale hashes, and edits to generated files.

## 9. Briefs and the LLM loop

`weave brief --target ios --task screen:Today --budget 1500` emits:

- edition, app, target, task, spec hash
- exact list of files to create or edit
- dependency closure of the task, formatted canonically
- port signatures, type definitions, error types
- acceptance scenarios and fixtures
- generated files that must not be edited
- rules: no new dependencies, no renamed symbols, add anchors, run `weave verify`
- token count and remaining budget

Briefs are deterministic: stable ordering, no timestamps, project-relative paths. Markdown for the model; JSON for tools.

The loop:

1. Author or change the spec.
2. Run `weave check`, `weave lint`, `weave fmt --check`.
3. Generate a brief for one task.
4. LLM implements only the listed files.
5. Run `weave verify --target ...`; feed structured diagnostics back.
6. Repeat until green.

Anti-hallucination mechanisms:

- Closed-world name resolution.
- Pinned edition; unknown syntax is rejected.
- Stable names and content hashes.
- Minimal task-scoped context instead of the whole spec.
- Deterministically generated core.
- Executable scenarios as the behavioral oracle.
- Generated conformance tests, not LLM-authored tests.
- Traceability anchors and stale-file detection.

Token goals:

- Habit tracker spec: under ~2k tokens.
- Per-screen or per-adapter brief: under ~1.5k tokens.
- `weave stats` reports actual counts; lint enforces budgets.
- The POC report benchmarks Weave against equivalent YAML/JSON and prose specs.

## 10. Demo: habit tracker

**Domain**

- `Habit`: id, name, cadence, archived, createdAt
- `CheckIn`: id, habit, day
- `HabitStatus` value: habit, due, checked, streak
- `Cadence`: `Daily`, `Weekdays`
- Errors: `HabitNotFound`, `AlreadyCheckedIn`, `HabitArchived`

**Ports:** `Habits`, `CheckIns`, `Clock`, `Ids`.

**Use cases**

- `CreateHabit`
- `CheckInHabit`
- `ListToday`
- `HabitHistory`
- `ArchiveHabit`

**Screens and flow**

- `Today`: list statuses, check in, navigate to detail/new
- `HabitDetail`: history and streak
- `NewHabit`: create form
- `flow Main`: start at `Today`; every screen reachable

**Targets**

- iOS: Swift 6, SwiftUI, Swift Package Manager, Swift Testing
- Web: TypeScript strict, React, Vite, Vitest

**Adapters**

- `LocalStore` for `Habits` and `CheckIns` (file JSON on iOS, localStorage on Web)
- `SystemClock` for `Clock`
- `UUIDIds` for `Ids`

Stubs are generated; bodies are LLM-implemented from briefs.

**Scenarios:** create valid/invalid, check in, duplicate check-in, weekday cadence, streak calculation, archive hides from Today, not-found cases.

**Acceptance**

- `weave check`, `lint`, `fmt --check`, `run` green.
- Generated iOS and Web code compiles.
- Generated conformance tests pass on both targets.
- `weave verify` reports full traceability.
- Both apps run and match the spec manually.

**Out of scope:** sync, auth, notifications, localization, design tokens, accessibility audits, weekly cadence.

## 11. Phases

| Phase | Deliverable | Exit criteria | Rough effort |
|---|---|---|---|
| P0 | Language contract, example spec, diagnostic catalog | Example is human-reviewable; edition frozen | 2–3 days |
| P1 | Lexer, parser, formatter, name/type/architecture checker | `check` and `fmt` work on the example; corpus tests; JSON diagnostics | 1–1.5 weeks |
| P2 | Use cases, pure methods, interpreter, scenarios | `weave run` passes every habit scenario; failures are precise | 1.5–2 weeks |
| P3 | Screens, flows, targets, completeness, lint, graph | Missing copy, unreachable screens, and layer violations are caught | 1 week |
| P4 | Swift/TS codegen, runtime helpers, fakes, conformance tests | Generated code compiles; tests pass on both; `gen --check` is idempotent | 2 weeks |
| P5 | Briefs, anchors, manifest, verify, stats | Briefs fit budget; stale or missing anchors fail; verify is green | 1 week |
| P6 | End-to-end LLM demo, playbook, measurement report | Both apps built from briefs; verify green; metrics recorded | 1–1.5 weeks |

Total: roughly 8–10 weeks solo. Phases can overlap.

A thin vertical slice should land after P2: one entity, one use case, one scenario, one screen, one Swift type. This validates the full pipeline before broadening.

## 12. Testing strategy

- Unit tests per crate.
- Snapshot tests for diagnostics and generated code.
- Golden corpus: valid and invalid specs with expected codes.
- Property tests: `parse(print(ast)) == ast`; formatter idempotence.
- Interpreter tests for value semantics, date math, and call traces.
- Differential tests: interpreter results versus generated Swift/TS results on shared fixtures.
- CLI integration tests with temporary projects.
- End-to-end script: `check`, `fmt --check`, `run`, `gen --check`, target tests, `verify`.
- Determinism tests: run twice, compare bytes and token counts.
- CI on macOS and Ubuntu; Swift target tests on macOS.

## 13. Risks and non-goals

**Risks**

- Codegen breadth for Swift and TypeScript is the largest risk. Mitigate with a narrow typed AST and vertical slices.
- Interpreter and generated code can drift. Mitigate with differential tests on shared fixtures.
- UI correctness is not fully verifiable. Anchors and manual review cover structure; visual behavior stays human-reviewed.
- Date and text semantics differ across platforms. Use generated runtime helpers and edge-case fixtures.
- Scope creep toward a general-purpose language. Freeze v0.1 and reject escape hatches.
- Token metrics are model-specific. Report relative comparisons and use a fixed tokenizer.

**Non-goals for the POC**

- All six targets. iOS and Web prove the model; Android and desktop follow.
- Pixel-perfect UI or a UI toolkit.
- A general-purpose programming language.
- Full generation of database/network adapters.
- Localization, design tokens, accessibility, offline sync.
- LSP, editor extension, and tree-sitter grammar. These are hardening work.

## 14. Open decisions

- Working name: **Weave**; rename is cheap now and expensive later.
- Pure entity methods (`fun`) are in v0.1. If schedule slips, duplicate small predicates in use cases instead.
- Use case bodies are generated for the supported core subset. If codegen slips, fall back to generated stubs plus LLM bodies while keeping conformance tests.
- Screen view models are LLM-owned in the POC. Generating reactive view models is a later optimization.
- Android (Kotlin/Compose) is the next backend after iOS/Web; desktop follows via Tauri or native profiles.

## 15. Next step

Switch to a build agent and start P0/P1: freeze `docs/language.md`, write the habit spec, bootstrap the Rust workspace, and land the walking skeleton (`check`, `fmt`, one scenario, one generated Swift type).