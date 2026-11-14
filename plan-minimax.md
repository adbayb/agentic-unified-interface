# Strata — plan

**Project:** Strata — a declarative specification language + verification CLI for the AI era.
**Repo root:** `/Users/ayoubadib/Developer/poc-new-language`
**Status:** Approved direction; M1 scope agreed. Open questions in §11.
**Date:** 2026-11-13

---

## 1. Positioning

`makespec` governs *records* (ADRs/RFCs in Markdown). `poc-openspec` / `poc-speckit` govern *process*. Strata is the missing third piece: **an executable specification language** where the spec *is* the application, and the compiler is the oracle that proves it.

| | Makespec | OpenSpec / Spec-Kit | **Strata** |
|---|---|---|---|
| Output | Markdown records | Markdown deltas/tasks | **Typed, layered, compilable source** |
| Validation | Structure only | Structure only | **Types + architecture rules + totality** |
| Produces code | No | No | **Yes (M2: native per platform)** |
| Agent contract | Skill + CLI | Skill + bash | **Skill + CLI + JSON diagnostics + context API** |

Core thesis: **the spec is the only artifact an LLM writes; everything else is compiler output.** Token efficiency follows from deleting boilerplate, not from compressing syntax.

---

## 2. Naming

`strata` is heavily occupied on public registries:

- npm `strata` — HTTP server (mjackson/strata)
- npm `strata-lang` — statically-typed scripting language, owns the `.strata` extension
- crates.io `strata`, `strata-rs` — binary format / search
- GitHub `strata-core/strata` — "a general-purpose, strongly typed programming language designed for safe automation, explainable AI" (direct competitor)

Verified registry state:

| Name | crates.io | npm | Note |
|---|---|---|---|
| `strata` | taken | taken | x |
| `stratum` | taken | taken | x |
| `mantle` / `bedrock` / `tessera` / `palisade` | taken | taken | x |
| `strata-lang` | **free** | taken | ok crate name |
| `strata-spec` | **free** | **free** | ok |
| `stratalang` | **free** | **free** | ok |

**Decision:** keep the **binary and language name `strata`** (binaries need no registry uniqueness), publish the crate as `strata-lang` and the npm package as `strata-spec` (or `@scope/strata`). Source extension `.strata`.

---

## 3. The language

Four layers, one keyword set, one file per unit. The layer *is* the directory — the compiler reads the path to enforce the dependency rule.

| Layer dir | Declarations | May depend on |
|---|---|---|
| `spec/entities/` | `entity`, `value`, `enum`, `event`, `error` | **nothing** |
| `spec/use_cases/` | `use case`, `port` | entities |
| `spec/adapters/` | `adapter` (implements a port), `presenter` | entities, use_cases |
| `spec/frameworks/` | `view`, `framework` (composition root), `theme` | all four |

### Domain

```strata
# spec/entities/task.strata
entity Task {
  id:        TaskId
  title:     Text(1..120)
  done:      Bool = false
  priority:  Priority = Priority.normal
  tags:      [Tag] = []
  createdAt: Instant = now()

  rule title_not_blank { title.trim != "" }
}

value TaskId { raw: Uuid }
enum Priority { low, normal, high }
```

### Port

```strata
# spec/use_cases/task_repository.strata
port TaskRepository {
  fn all(filter: TaskFilter = TaskFilter.all) -> [Task]
  fn byId(id: TaskId) -> Task?
  fn save(task: Task) -> Task
  fn delete(id: TaskId) -> Void
}
```

### Use case

```strata
# spec/use_cases/add_task.strata
use case AddTask {
  requires { repo: TaskRepository }
  input  { title: Text(1..120) }
  output { task: Task }
  errors { InvalidTitle, DuplicateTitle }

  do {
    if input.title.trim == "" { fail InvalidTitle }
    if repo.all().filter(t => t.title.lower == input.title.lower).len > 0 {
      fail DuplicateTitle
    }
    yield repo.save(Task(title: input.title))
  }
}
```

### Adapter

```strata
# spec/adapters/sqlite_task_repository.strata
adapter SqliteTaskRepository : TaskRepository {
  requires { db: Database }
  backend: sqlite

  fn all(filter) -> [Task] { db.select(Task).filter(t => filter.matches(t)) }
  fn byId(id) -> Task?          { db.find(Task, id) }
  fn save(task) -> Task         { db.upsert(Task, task) }
  fn delete(id) -> Void         { db.remove(Task, id) }
}
```

### Framework (composition root)

```strata
# spec/frameworks/app.strata
framework App {
  targets: ios, android, web, macos, windows, linux
  binds {
    repo: TaskRepository = ios: SqliteTaskRepository, web: IndexedDbTaskRepository, default: SqliteTaskRepository
    db:   Database      = sqlite
  }
  screens { TaskList }
}
```

### Constrained expression sublanguage

The anti-hallucination lever. This is the totality guarantee.

**Allowed:** literals (Int, Float, Text, Bool, Instant, Duration, Color, Hex) · field access `a.b` · index `a[i]` · calls `f(args)` · operators `+ - * / % == != < <= > >= && || !` · `if` / `else` · `let` · `yield` · `fail` · single-expression lambdas `x => expr` · list/map literals · string interpolation.

**Forbidden:** `while` · user-defined functions · recursion · mutable state · dynamic dispatch · user-defined generics · closures capturing mutable state.

**Consequence:** every Strata program is total. No infinite loops, no stack overflow, no unbounded inference. The compiler states this, and `strata check` proves it mechanically.

---

## 4. Clean architecture as a compiler invariant

Layer violations are *errors*, not review comments.

| Code | Rule |
|---|---|
| `E2001` | `entities` importing anything but `entities` -> layer violation |
| `E2001` | `use_cases` importing `adapters` / `frameworks` -> layer violation |
| `E2001` | `adapters` importing `frameworks` -> layer violation |
| `E2002` | adapter not implementing a declared port member |
| `E3001` | required port with no binding for a target (highest-value error) |
| `E3002` | binding to an unknown adapter |

---

## 5. CLI

```
strata init [name]          # scaffold + install .agents/skills/strata
strata check [paths...]      # parse + typecheck + lint        (exit 0/1/2)
strata lint [paths...]       # lint only; --rule <id> --fix
strata format [paths...]     # canonical format; --check
strata schema [--json]       # grammar + stdlib signatures   <- LLM grounding
strata context <selector>    # minimal verified context slice; --budget <tokens>
strata explain <CODE|path>   # diagnostic or symbol explainer
strata compile --target <t>  # native codegen (M2)
strata doctor                # toolchain check
```

Every command accepts `--json`. Exit codes: `0` clean, `1` diagnostics found, `2` bad usage — matching makespec.

### Diagnostics catalogue (stable, never renumbered)

`E1001` unknown identifier · `E1002` unknown type · `E1003` type mismatch · `E1004` unknown member · `E1005` arity mismatch · `E1006` invalid constraint · `E1100` duplicate declaration · `E1200` unknown constructor field · `E1300` missing required field · `E1400` invalid literal · `E1500` use case yields nothing · `E2001` layer violation · `E2002` port member unimplemented · `E3001` unresolved binding · `E3002` unknown adapter binding · `E4001` parse error · `E4002` unexpected token · `W1001` unused import · `W1002` unused declaration · `W3001` target not bound

---

## 6. Anti-hallucination: three commands the agent calls instead of guessing

This is the actual product.

1. **`strata schema --json`** — complete grammar + stdlib signature index, version-pinned. The agent loads this *once* instead of inventing `repo.fetchAll()` that doesn't exist. Every unresolvable name is a hard `E1001`, so the compiler is ground truth, not the agent's memory.
2. **`strata context <selector> --budget N`** — returns the *transitive dependency closure* of a declaration, deduplicated, canonicalized, hard-capped at N tokens (measured, not estimated). Biggest token lever: the agent pulls exactly the verified slice it needs instead of re-reading the repo each turn.
3. **`strata explain E2001`** — plain-text, agent-readable explanation of any diagnostic plus the offending span.

Plus: `--json` diagnostics with stable codes and spans, formatter idempotence (minimal diffs -> minimal patches), golden-file tests proving codegen determinism.

---

## 7. Token efficiency — mechanisms and proof

| Mechanism | Effect |
|---|---|
| Compiler generates constructors, validation, serialization, equality, DI wiring, HTTP routes, mappers, test skeletons | Deletes the bulk of native boilerplate |
| Fixed, small keyword vocabulary | Concentrated token distribution -> fewer tokens for the same content |
| `strata context --budget N` | Agent reads a slice, not the repo |
| `strata schema --json` | One-time load instead of repeated guessing |
| Canonical formatting | Small, reviewable patches |
| `include` + stdlib imports | No repeated declarations |

**Measured, not asserted.** `strata bench tokens` will tokenize (o200k_base reference) 3 reference apps (todo, notes+offline sync, weather+API) three ways:

- (a) LLM writes Swift + Kotlin + TS natively
- (b) LLM writes the Strata spec
- (c) Strata spec + compiler output

and reports spec tokens vs. native tokens, ratio, and spec lines vs. generated lines. The number goes in the RFC, with the harness committed. If the ratio isn't compelling, the thesis is wrong and we should know in M1, not M4.

---

## 8. Repository layout

Mirrors makespec — one Rust binary, hexagonal boundaries inside it, offline, no telemetry.

```
Cargo.toml  AGENTS.md  CONTRIBUTING.md  README.md  CHANGELOG.md  LICENSE
src/
  main.rs
  startup/
    cli.rs
    commands/{init,check,lint,format,schema,context,explain}.rs
  core/
    syntax/{source,lexer,parser,ast}.rs
    semantics/{symbols,types,resolve,layer}.rs
    diagnostics.rs
    fmt.rs
    lint/{mod,rules}.rs
    project.rs
  adapters/
    filesystem.rs
    embedded.rs
tests/{cli.rs,fixtures/}
benches/
docs/{decisions/,specifications/}          # already initialized: ADR-0001 = makespec adoption
.agents/skills/makespec/                   # installed by `makespec init`
.agents/skills/strata/{SKILL.md,references/{language,cli,diagnostics}.md}
examples/taskflow/
```

Dependencies: `clap`, `serde` / `serde_json`, `thiserror`, `toml`. No parser generator — hand-written recursive descent, so the grammar in `references/language.md` and the code cannot drift.

---

## 9. Milestones

### M1 — Language + verification CLI (agreed MVP)

1. Records, authored *with `makespec`* (dogfooding). `makespec init` has already been run in this directory — ADR-0001 ("Use Makespec to manage decisions and specifications") is the adoption decision and `docs/specifications/` is empty, so Strata's own records start at **ADR-0002** and **RFC-0001**: ADR-0002 (Rust), ADR-0003 (four layers), ADR-0004 (offline / agent-driven), ADR-0005 (constrained expressions and totality), ADR-0006 (diagnostic code stability), RFC-0001 (Strata PRD).
2. Grammar reference doc.
3. Lexer, parser, AST, spans.
4. Symbol table, type system, resolver, type checker.
5. Layer rule engine.
6. Lint rules (v1 set).
7. Canonical formatter.
8. Diagnostics — codes, JSON output, `explain`.
9. CLI: `init`, `check`, `lint`, `format`, `schema`, `context`, `explain`.
10. Bundled agent skill (`.agents/skills/strata/`).
11. Test suite — golden formatter, parser round-trip, diagnostics, CLI integration, formatter idempotence property test.
12. `examples/taskflow/`.

**Exit criteria**

- `strata check` clean on the example project.
- Formatter idempotent (property test).
- Every diagnostic code documented and reachable via `strata explain`.
- `strata context --budget N` provably never exceeds N tokens.
- Every layer rule provably fires on a negative fixture.

### M2 — Native codegen

- Platform-neutral IR + backend trait.
- Backends: Swift/SwiftUI (iOS + macOS), Kotlin/Compose (Android), TypeScript/React (Web), .NET (Windows), Linux via the TS backend under Tauri.
- Hard problem is UI parity — solved with a fixed ~15-primitive widget vocabulary and a compile error `E3003 unsupported_widget_for_target` when a view uses something a target cannot express.
- Golden-file tests per backend.

### M3 — Runtime + platform adapters + agent integration

- Platform capability ports: Http, Storage, SecureStorage, Notifications, Camera, Biometrics, FileSystem — with per-target implementations.
- LSP server for editor diagnostics.
- MCP server exposing `check` / `schema` / `context`.

### M4 — Measurement + ecosystem

- Token benchmark suite with published numbers.
- Stdlib adapter registry.
- VS Code extension.

---

## 10. Risks

1. **UI parity across 6 platforms is the hard part.** Mitigated by the fixed widget vocabulary + explicit unsupported-widget error, and by shipping M2 backend-by-backend rather than all-at-once.
2. **Expression sublanguage scope creep.** Mitigation: freeze the allowed set in ADR-0004; additions are breaking changes.
3. **"No loops" is ergonomically hostile.** Mitigation: rich combinators (`map`, `filter`, `fold`, `sort_by`, `group_by`, `first`, `any`, `all`) so most real logic is expressible. The totality guarantee is worth more than the convenience.
4. **Diagnostic codes become a public contract.** Mitigation: ADR-0005 — codes never renumbered, documented deprecation path.
5. **Name collision** — see section 2.

---

## 11. Open questions

1. **Name** — proceed with binary `strata` + crate `strata-lang` / npm `strata-spec`, or pick another?
2. **Views in `adapters/` or `frameworks/`?** The plan puts views + composition root in `frameworks/` (outermost), but the existing `clean-architecture` repo puts views in `frameworks/` and gateway *interfaces* in `adapters/` — the inverse split. Worth aligning with the existing convention.
3. **Should M1 include a minimal `strata compile` stub** that dumps a JSON IR, proving the IR shape early rather than designing it in M2?
4. **Where should `strata` live** — a new `strata-lang` repo, or here in `poc-new-language`?

---

## 12. Conventions inherited from makespec

- One Rust Cargo binary; dependencies locked in `Cargo.lock`; MSRV declared in `Cargo.toml`.
- Hexagonal boundaries inside one binary: `src/core/` owns language and rules, `src/adapters/` owns filesystem and embedded assets, `src/startup/` composes and handles terminal IO.
- Offline. No network, no LLM-provider calls, no telemetry.
- Exit codes `0` / `1` / `2`; diagnostics on stderr.
- Bounded input sizes; bodies read from stdin, never argv.
- `docs/decisions/` (ADR) and `docs/specifications/` (RFC) as numbered Markdown records with YAML front matter; status only in front matter.
- Agent skill is a router: `SKILL.md` holds the workflow, detail lives in `references/`; semver `metadata.version`.
- Conventional Commits; release archives per OS/CPU.
