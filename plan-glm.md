# Blueprint — Cross-platform product specification framework

**Status:** POC plan — Phase 0 (design docs) pending execution
**Edition:** 2026.1
**Working name:** Blueprint (`blue` CLI, `*.blp` files)

## 1. Decisions

- **Language scope:** the blueprint is a full app blueprint — domain, use cases, adapters, and an abstract UI layer (screens, widgets, navigation).
- **Delivery model:** hybrid — the CLI deterministically generates the provable core (domain, ports, use case contracts); the LLM implements adapters, UI, and platform integration.
- **Syntax:** indentation-based, YAML-like, no braces, terse keywords, inferred defaults.
- **CLI stack:** Rust workspace, single `blue` binary.
- **First proof:** the Todo app on Web (TypeScript/React); Swift and Kotlin follow.
- **Phase 0 deliverable:** design docs and a reference spec, before any code.

## 2. Thesis

The product specification is the source of truth. Platform code is an implementation artifact, not a separate design.

The framework is three products:

1. **The spec language** — clean architecture is expressed as grammar: `domain`, `usecase`, `adapters`, `platforms`.
2. **The verifier CLI** — parses, resolves, typechecks, enforces architecture rules, formats canonically, and reports structured diagnostics.
3. **The LLM contract** — the language is written for token efficiency and closed-world name resolution, so an LLM authoring a spec cannot invent references; the CLI proves it.

The language is intentionally not a general-purpose programming language. Its core is small enough for precise semantics and deterministic verification.

## 3. The language — sketch

Indentation-based, no braces, terse keywords, defaults inferred:

```
app Todo v1

domain
  entity Task
    id Uuid key
    title Text 1..200
    done Bool = false
    createdAt Instant
    invariant done -> title.len > 0

usecase
  flow AddTask(title Text)
    require title.len > 0 fail "Title required"
    task = Task.new(title)
    TaskStore.put(task)
    emit TaskAdded(task)

  flow ToggleTask(id Uuid)
    task = TaskStore.get(id) ?? fail NotFound
    task.done = !task.done
    TaskStore.put(task)

adapters
  port TaskStore
    put(Task)
    get(id Uuid) -> Task?
    list() -> [Task]
    changes() -> [Task] stream

platforms
  screen TaskList
    tasks = TaskStore.changes
    view
      list tasks -> TaskRow
      fab -> AddTask

  screen AddTask
    form title -> AddTask
```

## 4. Clean architecture mapping

Clean architecture is structural, not advisory. Four fixed top-level sections; the checker rejects any cross-layer shortcut, so an LLM cannot express an architecture violation.

| Clean architecture | Blueprint construct | CLI produces |
|---|---|---|
| Entities | `entity`, `value`, `enum`, `error` | Domain types, validation, pure methods |
| Use cases | `flow` | Flow interfaces, result types, fakes, contract tests |
| Adapters | `port` | Port protocols, adapter stubs, wiring |
| Frameworks | `screen`, `nav` | UI scaffolding, navigation graph, composition root |

Enforced dependency rules:

- `domain` may reference only `domain`.
- `usecase` may reference `domain` and declared `error` types.
- `adapters` may reference `usecase`, `domain`, and errors.
- `platforms` may reference everything below through view bindings.

No outward references, no cycles between layers.

## 5. Language v0.1

### 5.1 File model

- One `app.blp` declares the app name, edition, and target list.
- Other `.blp` files contain declarations in a single project-wide namespace.
- Declarations are order-independent; resolution is by name, not position.
- Sections appear in the fixed order `domain`, `usecase`, `adapters`, `platforms` — this is a token-efficiency rule (see §9) as much as a structural one.
- Comments use `#`. The formatter preserves leading and inline comments on declarations.
- Canonical format: 2-space indentation, one statement per line, fixed clause order, single trailing newline.
- Indentation is the only nesting mechanism: no braces, no terminators, no line continuations. Tabs are rejected with a dedicated diagnostic.

### 5.2 Constructs and semantics

- **Closed world:** every name must resolve; every declaration is unique; no implicit behavior.
- **Types:** `Bool`, `Int`, `Float`, `Text`, `Uuid`, `Instant`, `Date`, `Duration`, `[T]`, `T?`, `Id<T>`, records, enums.
- **Entities and values:** immutable. Constructors use named fields. `.with(field: value)` produces an updated copy.
- **Pure methods:** `fun` blocks inside entities/values hold domain behavior. No ports, no mutation, no recursion.
- **Invariants:** expressions checked on construction and `.with()`.
- **Flows (use cases):** the only effectful units. Declare inputs, output, `needs` ports, typed `errors`, a `do` body, and `scenario` blocks.
- **Ports:** async by default. Adapter failures surface as an implicit infrastructure error outside the domain error set.
- **Error handling:** `fail E`, `require cond else E`, `?? fail E`, `if let`. Domain errors are typed and declared.
- **Executable core:** `let`/`var`, `if`/`else`, `for` over finite lists, `return`, `fail`, `require`, field access, constructors, built-ins, and port calls. No recursion, no lambdas, no classes, no escape hatches.
- **Screens:** declare route, loaded data, actions, navigation, states, and copy. Layout is deliberately unspecified and left to the LLM.
- **Navigation:** `nav` declares the start screen and the reachable graph; the checker verifies every screen is reachable.
- **Targets:** declare framework profile only (`ui react`, `ui swiftui`). Paths and commands live in `blue.toml`.
- **Determinism:** no ambient time or randomness. Everything enters through ports, so scenarios are reproducible.

### 5.3 Example (expanded)

`Task`, `TaskStore`, and the flows live in one file; the `NotFound` error is declared in `domain`:

```
app Todo v1

domain
  error NotFound(id Uuid)

  enum Priority
    low
    normal
    high

  entity Task
    id Uuid key
    title Text 1..200
    done Bool = false
    priority Priority = normal
    createdAt Instant

    fun isOverdue(now Instant) -> Bool
      return not done and createdAt < now

  value TaskFilter
    done Bool?
    priority Priority?

usecase
  flow AddTask(title Text, priority Priority)
    require title.len > 0 else NotFound
    task = Task.new(title, priority)
    TaskStore.put(task)
    emit TaskAdded(task)

  flow ToggleTask(id Uuid)
    task = TaskStore.get(id) ?? fail NotFound
    TaskStore.put(task.with(done: not task.done))

  flow ListTasks(filter TaskFilter?) -> [Task]
    all = TaskStore.list()
    if filter == none
      return all
    return all.filter(task -> task.done == filter.done)
```

## 6. Verification ladder

| Level | Command | Proves |
|---|---|---|
| V0 syntax | `blue check` | Parses with spans and recoverable diagnostics |
| V1 names | `blue check` | Closed world; no invented references |
| V2 types | `blue check` | Arity, nullability, constraints, enum variants |
| V3 architecture | `blue check` | Layer direction, port/conformance rules |
| V4 completeness | `blue check`, `blue lint` | Targets, ports, screens, errors, copy, reachability |
| V5 behavior | `blue run` | Scenarios execute; calls and failures match |
| V6 traceability | `blue verify` | Anchors, hashes, generated freshness, target tests |
| V7 conformance | `blue verify` | Same fixtures pass on every target |

Diagnostic codes are stable and grouped:

- `BL1xxx` syntax and lexing
- `BL2xxx` name resolution
- `BL3xxx` types
- `BL4xxx` architecture
- `BL5xxx` completeness
- `BL6xxx` scenario execution
- `BL7xxx` lint and budgets
- `BL8xxx` formatting
- `BL9xxx` verification and traceability

Every diagnostic carries file, span, line, column, severity, code, message, help text, and related locations. A name-resolution failure suggests the closest match:

```
error[BL2041]: unknown port `TaskStor`
  --> app.blp:28:5
   |
28 |     TaskStor.put(task)
   |     ^^^^^^^^ did you mean `TaskStore`?
```

## 7. CLI

### 7.1 Core commands

| Command | Purpose |
|---|---|
| `blue check` | Syntax, names, types, architecture, completeness |
| `blue lint` | Naming, dead specs, ambiguity, scenario coverage, token budgets |
| `blue fmt` | Canonical formatting; `--check` for CI |
| `blue run` | Execute scenarios in the reference interpreter |
| `blue gen` | Generate core, scaffolds, fakes, and conformance tests for a target |
| `blue brief` | Emit a task-scoped context pack for the LLM |
| `blue verify` | Anchors, hashes, stale files, target tests, coverage |
| `blue graph` | Layer and dependency graph |
| `blue stats` | Spec and brief token counts |
| `blue explain` | Explain an error code |
| `blue manual` | Compact language reference for LLMs |
| `blue schema` | Machine-readable grammar and diagnostic catalog |

All commands support `--json`. Exit codes: `0` success, `1` findings, `2` usage error, `3` internal error.

### 7.2 JSON diagnostics

```json
{
  "version": 1,
  "command": "check",
  "ok": false,
  "summary": { "errors": 1, "warnings": 2 },
  "diagnostics": [
    {
      "code": "BL2041",
      "severity": "error",
      "message": "unknown port `TaskStor`",
      "file": "app.blp",
      "line": 28,
      "column": 5,
      "span": { "start": 412, "end": 419 },
      "help": "did you mean `TaskStore`?",
      "related": []
    }
  ]
}
```

### 7.3 Configuration

Spec metadata lives in `app.blp`. Tooling paths and commands live in `blue.toml`:

```toml
[targets.web]
backend = "typescript"
root = "targets/web"
test = ["npm", "--prefix", "targets/web", "run", "test"]

[targets.ios]
backend = "swift"
root = "targets/ios"
test = ["swift", "test", "--package-path", "targets/ios"]

[lint]
max_brief_tokens = 8000
max_line_width = 100
```

The checker requires every declared target to have a matching config section.

## 8. Hybrid ownership

| Artifact | Owner | Regeneration |
|---|---|---|
| Domain types, runtime helpers, ports, flow contracts, errors | CLI | Always overwritten; hash-guarded |
| In-memory fakes, conformance tests, fixtures | CLI | Always overwritten |
| Adapter stubs, screen stubs, composition root, app shell | CLI | Scaffolded once; never overwritten without `--force` |
| Adapter bodies, UI, platform integration | LLM | Anchored; verified |
| Spec and design decisions | Human | Source of truth |

Generated files carry a header with edition, target, and content hash:

```ts
// @blue-generated app=Todo edition=2026.1 target=web hash=...
```

LLM-owned files carry implementation anchors:

```ts
// @blue impl adapter:LocalStore spec=...
```

`blue verify` rejects missing anchors, duplicate implementations, stale hashes, and edits to generated files.

## 9. Token efficiency

The language is designed for token-efficient LLM input. Concretely:

- Indentation replaces braces; keywords stay ≤ 8 characters.
- Fixed section order (`domain` → `usecase` → `adapters` → `platforms`) keeps the prompt prefix byte-stable, maximizing provider prompt-cache reuse across turns.
- Short, stable identifiers with checker-resolved references; no long qualified names inside bodies.
- Defaults are inferred rather than declared (`Bool` defaults to `false`, ids default to `Uuid key`).
- One declaration per line, rare comments, no decorative punctuation.
- Diagnostics are one line each with a stable code, so repair loops cost few tokens.

Token goals:

- Todo app spec: under ~2k tokens.
- Per-screen or per-adapter brief: under ~1.5k tokens.
- `blue stats` reports actual counts; lint enforces budgets.

## 10. Repository layout

```
.
├── Cargo.toml
├── rust-toolchain.toml
├── crates/
│   ├── blueprint-syntax/    # spans, lexer, parser, AST, diagnostics
│   ├── blueprint-check/     # resolver, types, architecture, completeness, lint
│   ├── blueprint-fmt/       # canonical printer
│   ├── blueprint-interp/    # reference interpreter (scenarios)
│   ├── blueprint-gen/       # TS/Swift codegen, briefs, manifest, verify
│   └── blueprint-cli/       # commands, config, JSON output
├── docs/
│   ├── 00-vision.md         # problem, principles, non-goals, success criteria
│   ├── 01-language.md       # full reference: grammar, constructs, types, tokens
│   ├── 02-architecture.md   # layer rules, bindings, conformance contract
│   ├── 03-toolchain.md      # CLI UX, diagnostics catalog, canonical format
│   ├── 04-codegen-targets.md# generated vs LLM-owned, per platform
│   └── 05-llm-protocol.md   # authoring rules, briefs, repair loop
├── examples/todo/
│   ├── blue.toml
│   ├── spec/
│   │   ├── app.blp
│   │   ├── domain.blp
│   │   ├── usecase.blp
│   │   ├── adapters.blp
│   │   └── platforms.blp
│   └── targets/web/
└── tests/
    ├── corpus/
    └── e2e/
```

## 11. Demo: Todo app

**Domain**

- `Task`: id, title, done, priority, createdAt
- `Priority`: low, normal, high
- `TaskFilter` value: done, priority
- `NotFound` error

**Ports:** `TaskStore` (put, get, list, changes).

**Flows:** `AddTask`, `ToggleTask`, `ListTasks`, `DeleteTask`.

**Screens:** `TaskList` (list + filter + add action), `TaskEdit` (title, priority), `TaskDetail`.
**Navigation:** start at `TaskList`; every screen reachable.

**Targets:** Web first (TypeScript strict, React, Vite, Vitest), then iOS (Swift 6, SwiftUI), then Android (Kotlin, Compose), then desktop.

**Adapters:** `LocalStore` (localStorage on Web, file JSON on iOS), `SystemClock`, `UuidIds`.

Stubs are generated; bodies are LLM-implemented from briefs.

**Scenarios:** create valid/invalid, toggle, delete, filter, not-found, empty list.

**Acceptance**

- `blue check`, `lint`, `fmt --check`, `run` green.
- Generated Web code compiles and its conformance tests pass.
- `blue verify` reports full traceability.
- The app runs and matches the spec manually.

**Out of scope:** sync, auth, notifications, localization, design tokens, accessibility audits.

## 12. Phases

| Phase | Deliverable | Exit criteria |
|---|---|---|
| P0 | Design docs + reference `todo` spec | Docs self-consistent; every construct exercised; grammar frozen |
| P1 | Lexer, parser, formatter, name/type/architecture checker | `check` and `fmt` work on the example; corpus tests; JSON diagnostics |
| P2 | Flows, pure methods, interpreter, scenarios | `blue run` passes every Todo scenario; failures are precise |
| P3 | Screens, navigation, targets, completeness, lint, graph | Missing copy, unreachable screens, and layer violations are caught |
| P4 | TypeScript codegen, runtime helpers, fakes, conformance tests | Generated code compiles; tests pass; `gen --check` is idempotent |
| P5 | Swift + Kotlin codegen | Same fixtures pass on all three targets |
| P6 | Briefs, anchors, manifest, verify, stats | Briefs fit budget; stale or missing anchors fail; verify is green |
| P7 | End-to-end LLM demo, playbook, measurement report | App built from briefs; verify green; token metrics recorded |

P0 is the immediate next step. A thin vertical slice should land after P2: one entity, one flow, one scenario, one screen, one generated TypeScript type.

## 13. Testing strategy

- Unit tests per crate.
- Snapshot tests for diagnostics and generated code.
- Golden corpus: valid and invalid specs with expected codes.
- Property tests: `parse(print(ast)) == ast`; formatter idempotence.
- Interpreter tests for value semantics and call traces.
- Differential tests: interpreter results versus generated TypeScript results on shared fixtures.
- CLI integration tests with temporary projects.
- Determinism tests: run twice, compare bytes and token counts.
- CI on macOS and Ubuntu.

## 14. Risks and non-goals

**Risks**

- Codegen breadth for TypeScript, Swift, and Kotlin is the largest risk. Mitigate with a narrow typed AST and vertical slices.
- Interpreter and generated code can drift. Mitigate with differential tests on shared fixtures.
- UI correctness is not fully verifiable. Anchors and manual review cover structure; visual behavior stays human-reviewed.
- Scope creep toward a general-purpose language. Freeze v0.1 and reject escape hatches.
- Token metrics are model-specific. Report relative comparisons and use a fixed tokenizer.
- Indentation syntax is unforgiving of inconsistent authoring. Mitigate with `fmt`, precise diagnostics, and tabs rejected outright.

**Non-goals for the POC**

- All six targets. Web proves the model; iOS, Android, and desktop follow.
- Pixel-perfect UI or a UI toolkit.
- A general-purpose programming language.
- Full generation of database/network adapters.
- Localization, design tokens, accessibility, offline sync.
- LSP, editor extension, and tree-sitter grammar. These are hardening work.

## 15. Phase 0 deliverables (immediate)

| Doc | Contents |
|---|---|
| `docs/00-vision.md` | Problem, principles, non-goals, hybrid delivery model, success criteria |
| `docs/01-language.md` | Full reference: EBNF grammar, constructs, type system v0, token-efficiency rules, worked example |
| `docs/02-architecture.md` | Layer rules, binding model, platform conformance, port/adapter contract |
| `docs/03-toolchain.md` | Rust workspace layout, CLI commands, diagnostic codes, `--json` output, canonical formatting |
| `docs/04-codegen-targets.md` | Generated vs LLM-owned per platform, conformance contract; Web/TS detailed first |
| `docs/05-llm-protocol.md` | Authoring rules for LLMs, brief format, repair loop, prompt templates |
| `examples/todo/spec/*.blp` | Reference spec exercising every construct in the docs |

## 16. Open decisions

- Language name **Blueprint**; renaming is cheap now and expensive later.
- Pure entity methods (`fun`) are in v0.1. If schedule slips, duplicate small predicates in flows instead.
- Screen view models are LLM-owned in the POC. Generating reactive view models is a later optimization.
- Android (Kotlin/Compose) is the next backend after Web; iOS follows; desktop via Tauri or native profiles.
- Whether `blue run` (interpreter) stays in v0.1 or is deferred to v0.2 once scenarios are proven.

## 17. Next step

Switch to a build agent and execute P0: write the six docs, freeze the grammar, and land the reference `todo` spec. Then bootstrap the Rust workspace and deliver the walking skeleton (`check`, `fmt`, one scenario, one generated TypeScript type).

---

## Appendix A — Divergence from `plan-deepseek.md`

A sibling plan exists in this repo (`plan-deepseek.md`, working name **Weave**). The two plans were authored independently and differ on the following points. Reconcile before coding, or choose one and retire the other.

| Dimension | This plan (Blueprint) | Sibling plan (Weave) |
|---|---|---|
| Syntax | Indentation-based, no braces | Braced, Rust/Swift-like |
| File extension | `.blp` | `.weave` |
| Layer naming | `domain` / `usecase` / `adapters` / `platforms` | `entity` / `port` / `usecase` / `adapter` / `screen` / `target` |
| First target | Web (TypeScript/React) | iOS (SwiftUI) and Web (React) |
| Domain model | Entities + values + enums + errors, pure `fun` methods | Same, with `fun` methods optional |
| Use case construct | `flow` with `do` body, `needs` ports, `scenario` blocks | `usecase` with `do` body, `needs` ports, `scenario` blocks |
| Verification | check / lint / fmt / run / gen / brief / verify | Same command set, `weave` binary |
| Priorities | Docs first, then interpreter, then codegen | Spec + codegen early, interpreter in P2 |
| Demo app | Todo | Habit tracker |

Shared ground: hybrid CLI-codegen + LLM-implementation model, Rust workspace, closed-world name resolution, structured JSON diagnostics, scenario-based behavioral oracle, generated conformance tests, token-budget goals, and the same four clean-architecture layers.
