---
id: RFC-0001
title: Prism cross-platform product specification framework (deepseek proposal)
date: 2026-11-13
status: proposed
---

# RFC-0001: Prism cross-platform product specification framework (deepseek proposal)

## Context

Product teams and coding agents need one source of truth for an application that targets iOS, Android, Web, Windows, Linux, and macOS. Today the same behavior and interface are restated in platform code, design files, and prose specifications. The copies drift, and an LLM asked to implement a platform must infer missing behavior, invent names, and guess at UI structure.

Prism is a working name for a declarative specification language and a verification CLI. A Prism specification declares the clean-architecture layers as first-class constructs: entities, use cases, ports and adapters, and framework-level screens, components, design tokens, navigation, and targets. The specification is readable without tooling, compact enough for LLM context, and checked before any platform code is written. A reference interpreter executes scenarios, and target generators produce a typed core plus a platform-neutral UI intermediate representation (IR). The LLM implements adapter bodies and platform integration from task-scoped briefs.

The v0.1 scope is the language grammar and semantics, the `prism` CLI with `check`, `lint`, and `format` as mandatory commands plus `run`, `build`, `brief`, and `verify`, a reference application, and end-to-end backends for iOS (SwiftUI) and Web (React, TypeScript). The language declares target profiles for Android, Windows, Linux, and macOS so those backends can be added without changing application specifications. The reference application is a habit tracker with eight screens: Today, HabitDetail, NewHabit, EditHabit, History, Archive, Settings, and About. It exercises entities, use cases, ports, adapters, tokens, themes, components, navigation, and scenarios.

Non-goals for v0.1: end-to-end delivery on all six targets, pixel-perfect visual parity across platforms, a general-purpose programming language, a visual editor, a hosted service, localization, offline synchronization, and a plugin marketplace.

## Requirements and constraints

### Requirements

| ID | Priority | Requirement |
| --- | --- | --- |
| FR-001 | MUST | The language MUST declare entities, value objects, enums, errors, invariants, and pure domain functions with explicit types. |
| FR-002 | MUST | The language MUST declare use cases with typed inputs, outputs, declared errors, required ports, and an executable behavior body. |
| FR-003 | MUST | The language MUST declare ports and adapters, and `prism check` MUST verify that each adapter implements every member of its port with compatible types. |
| FR-004 | MUST | The language MUST declare a platform-neutral screen tree with a fixed widget vocabulary, typed data bindings, actions, states, and navigation edges. |
| FR-005 | MUST | The language MUST declare layout, named styles, design tokens, themes, and responsive variants, and `prism check` MUST reject references to undeclared tokens, styles, widgets, or invalid widget properties. |
| FR-006 | MUST | The language MUST declare target profiles for iOS, Android, Web, Windows, Linux, and macOS, and v0.1 MUST generate and verify complete implementations for iOS and Web. |
| FR-007 | MUST | The language MUST declare reusable components with typed parameters and slots, and `prism check` MUST reject recursive component cycles. |
| FR-008 | MUST | The CLI MUST provide `check` for parsing, name resolution, type checking, architecture checking, and completeness; `lint` for quality rules and token budgets; and `format` for canonical formatting, each with documented exit codes and machine-readable output. |
| FR-009 | MUST | The CLI MUST enforce the clean-architecture dependency rule: entities depend only on entities; use cases depend on entities and ports; adapters depend on ports, entities, and errors; frameworks depend on all inward layers. It MUST reject outward references and cycles. |
| FR-010 | MUST | The CLI MUST provide a reference interpreter that executes use-case and UI-flow scenarios and exits nonzero when observed results, errors, side effects, widget presence, or navigation differ from expectations. |
| FR-011 | MUST | The CLI MUST emit a deterministic, versioned intermediate representation for each target, and repeated generation from the same specification and CLI version MUST produce byte-identical output. |
| FR-012 | MUST | The CLI MUST emit task-scoped context packs within a requested token budget, containing the exact declarations, target mappings, file list, and acceptance scenarios needed for one implementation task. |
| FR-013 | MUST | The CLI MUST verify traceability between specification declarations and implementation artifacts using stable IDs, content hashes, and implementation anchors, and reject missing, duplicate, or stale implementations. |
| FR-014 | MUST | The CLI MUST expose a machine-readable schema of the grammar, widget catalog, target capabilities, and diagnostic codes for LLM consumption. |
| FR-015 | MUST | The language MUST be edition-versioned, and the CLI MUST reject unknown editions and preserve the meaning of existing diagnostic codes across editions. |
| FR-016 | MUST | The language MUST require accessibility semantics for interactive widgets and alternative text for informative images, and `prism check` MUST reject omissions. |
| FR-017 | MUST | The language MUST isolate effects and nondeterminism behind ports, so use-case and UI scenarios execute reproducibly. |
| FR-018 | MUST | The CLI MUST run offline without network calls or telemetry. |
| NFR-001 | MUST | The reference application specification of at least eight screens MUST be at most 20,000 tokens under a pinned tokenizer, and each task context pack MUST be at most 4,000 tokens. |
| NFR-002 | MUST | `prism check` MUST complete in under 2 seconds for a 10,000-line specification on a 2024-class laptop. |
| NFR-003 | MUST | Every diagnostic MUST include a stable code, severity, file, line, column, message, and remediation help, and `--json` output MUST conform to a versioned schema. |
| NFR-004 | MUST | `prism format` MUST be idempotent: formatting an already formatted file MUST produce identical bytes. |
| NFR-005 | MUST | A full `prism check` of the reference application MUST stay below 500 MB of peak resident memory. |

### Constraints

| ID | Constraint |
| --- | --- |
| C-001 | v0.1 MUST deliver end-to-end iOS (SwiftUI) and Web (React, TypeScript) implementations of the reference application. |
| C-002 | The shared specification MUST contain no platform-specific widget names, APIs, or file paths. Platform differences MUST live in target profiles and adapters. |
| C-003 | The CLI MUST be a single native binary that works offline and sends no telemetry. |
| C-004 | The expression language MUST be total: no recursion, no unbounded iteration, no user-defined mutable global state, and every expression terminates. |
| C-005 | The language and CLI MUST NOT depend on a specific LLM provider or model. The LLM contract is expressed through files and JSON. |
| C-006 | v0.1 target toolchains are pinned to Swift 6 with SwiftUI and Swift Package Manager for iOS, and TypeScript strict mode with React, Vite, and Vitest for Web. |

## Options

Option assessments are design-time judgments. They identify expected compliance and known tradeoffs; they do not report measured performance, token counts, or target build results. The adoption phase validates those targets.

### Language surface and type model

| Criterion ID | Braced declarations with explicit types | Indentation-scoped declarations with inferred types | Schema-first data model with embedded expressions |
| --- | --- | --- | --- |
| FR-001 | Fully meets - keyword-led declarations give each domain construct an explicit, readable form. | Fully meets - entities and fields remain readable, but deep nesting relies on whitespace. | Partially meets - types are expressible, but invariants and pure functions become embedded strings or expression blobs. |
| FR-002 | Fully meets - use-case signatures and bodies have a stable clause order and clear boundaries. | Fully meets - use cases remain readable at shallow nesting, but long bodies are harder to scan. | Partially meets - behavior bodies are hard to read and review inside a data document. |
| FR-004 | Fully meets - braces make screen and widget boundaries explicit in deep UI trees. | Fully meets - the tree is readable, but indentation mistakes change meaning. | Partially meets - a widget tree becomes verbose and difficult to review as data. |
| FR-007 | Fully meets - component parameters and slots have explicit delimiters. | Fully meets - components are expressible, but slots are visually ambiguous at depth. | Partially meets - slots and typed parameters require custom schema conventions. |
| FR-015 | Fully meets - the edition is a top-level declaration and the grammar can reject unknown editions. | Fully meets - the edition is a top-level declaration. | Fully meets - the edition is a top-level field. |
| FR-017 | Fully meets - effectful calls are port calls, independent of block style. | Fully meets - effectful calls are port calls, independent of block style. | Fully meets - effects can remain restricted to declared calls. |
| NFR-001 | Fully meets - the 20,000-token budget leaves room for braces and explicit types. | Fully meets - indentation removes brace tokens and lowers the count. | Does not meet - repeated keys and quoting make the reference specification exceed the budget. |
| NFR-004 | Fully meets - a canonical printer fixes spacing, clause order, and line breaks. | Fully meets - a canonical printer can normalize indentation. | Fully meets - canonical serialization is straightforward. |
| C-004 | PASS - braces do not affect the total expression subset. | PASS - indentation does not affect the total expression subset. | PASS - the embedded expression language can remain total. |

**Chosen option:** Braced declarations with explicit types

Braces make the full UI tree and nested use-case bodies unambiguous for humans and for the formatter. Explicit types support closed-world checking and precise LLM briefs. The token cost is acceptable within NFR-001. Indentation would be more compact, but whitespace-sensitive meaning raises the cost of LLM-generated edits and diagnostic repair. The schema-first model is easier to parse but weak for behavior and UI review.

Considered and excluded: a visual block editor does not produce a text artifact that can be diffed, reviewed, or placed directly in an LLM context.

### UI specification and rendering

| Criterion ID | Normalized widget tree with reference renderers | Per-platform native screen templates | Free-form screen descriptions with LLM-authored UI |
| --- | --- | --- | --- |
| FR-004 | Fully meets - one typed tree declares structure, data bindings, actions, states, and navigation. | Fully meets - each platform has a typed tree, but the trees duplicate structure. | Partially meets - prose or sketches do not produce a checkable widget tree. |
| FR-005 | Fully meets - tokens, styles, themes, and responsive variants are shared and checked once. | Partially meets - styles and tokens diverge per platform and cannot be checked once. | Does not meet - there is no machine-checkable token or style model. |
| FR-006 | Fully meets - target profiles map the same primitives to each platform. | Fully meets - native templates map directly, with duplicated maintenance. | Partially meets - target capabilities are not declared, so unsupported UI is discovered late. |
| FR-007 | Fully meets - components have typed parameters and slots with cycle checks. | Partially meets - components are redefined per platform and can drift. | Does not meet - reusable typed components are not expressible. |
| FR-016 | Fully meets - semantic role, label, and alternative text are required properties. | Partially meets - each platform enforces its own semantics. | Does not meet - accessibility can be omitted without detection. |
| NFR-001 | Fully meets - one shared tree avoids platform duplication. | Does not meet - repeated screen trees exceed the token budget. | Partially meets - short descriptions fit, but they omit required detail. |
| C-001 | PASS - reference renderers for SwiftUI and React cover the v0.1 catalog. | PASS - native templates can target both platforms, at higher cost. | FAIL - end-to-end behavior and UI cannot be guaranteed from free-form descriptions. |
| C-002 | PASS - the shared tree contains only normalized primitives. | FAIL - the shared specification contains native widget names. | FAIL - platform-specific instructions enter the shared specification. |
| C-006 | PASS - the SwiftUI and React renderers use the pinned toolchains. | PASS - each template uses its platform toolchain. | FAIL - the toolchain contract is not tied to a declared UI model. |

**Chosen option:** Normalized widget tree with reference renderers

One tree with a fixed primitive catalog keeps the specification portable and within the token budget. Reference renderers for SwiftUI and React make generation deterministic. Per-platform templates duplicate the interface and break C-002. Free-form descriptions leave the most important verification target, the UI structure, uncheckable.

Considered and excluded: a per-platform design-token pipeline duplicates the tree and weakens the single-source-of-truth property.

### Platform implementation ownership

| Criterion ID | Hybrid generation and LLM implementation | Full generation of all artifacts | LLM implementation with CLI checks only |
| --- | --- | --- | --- |
| FR-003 | Fully meets - the CLI generates port interfaces and adapter stubs, and the checker verifies conformance. | Partially meets - interfaces are generated, but native adapter bodies need platform APIs that the specification does not express. | Fully meets - the LLM implements adapters and the checker verifies their shape. |
| FR-006 | Fully meets - the CLI generates the core and UI, and the LLM fills platform bindings. | Partially meets - native storage, clock, identity, and lifecycle integrations are not fully expressible. | Partially meets - target output depends on model behavior rather than a deterministic generator. |
| FR-008 | Fully meets - the same CLI checks every artifact regardless of owner. | Fully meets - the CLI checks generated artifacts. | Fully meets - the CLI checks LLM output. |
| FR-010 | Fully meets - the reference interpreter runs the specification scenarios. | Fully meets - generated code can be tested against the same scenarios. | Fully meets - the interpreter remains the behavioral oracle. |
| FR-011 | Fully meets - generated core and UI are byte-identical for the same inputs. | Fully meets - generation is deterministic. | Does not meet - LLM output is not byte-identical across runs. |
| FR-012 | Fully meets - briefs contain only the adapter or binding work the LLM owns. | Fully meets - briefs can describe generation tasks. | Partially meets - the LLM owns large, weakly bounded tasks. |
| FR-013 | Fully meets - generated files carry hashes and LLM files carry anchors. | Fully meets - generated files carry hashes. | Partially meets - anchors can be added, but the generated baseline is absent. |
| NFR-002 | Fully meets - generated artifacts and checks use the same bounded pipeline. | Fully meets - generation and checking are local operations. | Fully meets - the CLI operations remain local. |
| C-001 | PASS - the hybrid split covers both targets. | PASS - full generation can target both, with higher delivery risk. | PASS - an LLM can implement both, with no determinism guarantee. |
| C-003 | PASS - one offline binary performs generation and verification. | PASS - one offline binary performs generation. | PASS - one offline binary performs checks. |
| C-005 | PASS - ownership is defined by files and anchors, not by a model. | PASS - ownership is defined by generated files. | PASS - the workflow is model-agnostic even though output varies. |

**Chosen option:** Hybrid generation and LLM implementation

The CLI owns everything that can be derived deterministically: domain types, validation, use-case code, ports, UI IR, screen code, tokens, fakes, and conformance tests. The LLM owns adapter bodies and platform integration, which require knowledge of native APIs and cannot be generated from the specification alone. This split keeps the LLM surface small and verifiable. Full generation is rejected because native adapter bodies are not expressible in v0.1. LLM-only implementation is rejected because nondeterministic output defeats traceability and reproducible verification.

Considered and excluded: a plugin-based renderer marketplace expands the trust surface before the core contract and verification ladder are proven.

### Verification and anti-hallucination

| Criterion ID | Layered checks, executable scenarios, briefs, and traceability | Static checks and formatting only | Static checks plus generated tests without executable scenarios |
| --- | --- | --- | --- |
| FR-008 | Fully meets - check, lint, and format run in one pipeline with stable diagnostics. | Fully meets - check, lint, and format are present. | Fully meets - check, lint, and format are present. |
| FR-009 | Fully meets - layer rules are checked before generation. | Fully meets - layer rules are checked. | Fully meets - layer rules are checked. |
| FR-010 | Fully meets - scenarios execute in the reference interpreter. | Does not meet - the CLI cannot execute behavior. | Partially meets - generated tests exist, but specification scenarios do not execute. |
| FR-012 | Fully meets - task briefs are dependency-closed and token-bounded. | Does not meet - no task briefs are produced. | Partially meets - tests are generated, but context is not bounded per task. |
| FR-013 | Fully meets - hashes, anchors, and stale-file checks cover every artifact. | Does not meet - no implementation traceability exists. | Partially meets - tests can be traced, but generated and LLM-owned files are not fully covered. |
| FR-014 | Fully meets - the schema command exposes grammar, widgets, capabilities, and diagnostics. | Partially meets - a grammar schema can be exported, but the runtime contract is missing. | Partially meets - grammar and tests can be exported, but target capabilities are missing. |
| FR-017 | Fully meets - scenarios control ports and observe calls. | Fully meets - effectful calls are syntactically restricted to ports. | Fully meets - generated tests can control ports. |
| FR-018 | Fully meets - every command is local. | Fully meets - every command is local. | Fully meets - every command is local. |
| NFR-001 | Fully meets - briefs are capped and the full specification is measured. | Partially meets - the full specification can be measured, but no task budget is enforced. | Partially meets - tests add output rather than reducing context. |
| NFR-003 | Fully meets - all stages emit the same diagnostic schema. | Fully meets - static diagnostics share the schema. | Fully meets - static and test diagnostics share the schema. |
| NFR-004 | Fully meets - the formatter is part of the gate. | Fully meets - the formatter is present. | Fully meets - the formatter is present. |
| NFR-005 | Fully meets - the pipeline is bounded and local. | Fully meets - the pipeline is bounded and local. | Fully meets - the pipeline is bounded and local. |
| C-003 | PASS - no network or hosted verification is required. | PASS - no network is required. | PASS - no network is required. |
| C-004 | PASS - scenario execution respects the total expression subset. | PASS - static checking respects the subset. | PASS - test generation respects the subset. |
| C-005 | PASS - the workflow is model-agnostic. | PASS - the workflow is model-agnostic. | PASS - the workflow is model-agnostic. |

**Chosen option:** Layered checks, executable scenarios, briefs, and traceability

Static checks catch unknown names, type errors, layer violations, missing target bindings, and invalid UI references. Executable scenarios catch behavioral drift that no static rule can detect. Traceability ties generated and LLM-owned artifacts back to declarations. Briefs keep the LLM context small and exact. Static-only checking is rejected because it cannot detect behavior drift. Generated tests without executable scenarios are rejected because the specification itself must remain the behavioral oracle.

Considered and excluded: a hosted verification service conflicts with C-003 and adds a network dependency to the repair loop.

### CLI implementation stack

| Criterion ID | Rust workspace with a single binary | Go module with a single binary | TypeScript on Node with a bundled runtime |
| --- | --- | --- | --- |
| FR-008 | Fully meets - the command set, diagnostics, and JSON output are ordinary application code. | Fully meets - the command set, diagnostics, and JSON output are ordinary application code. | Fully meets - the command set, diagnostics, and JSON output are ordinary application code. |
| FR-014 | Fully meets - the schema can be emitted from typed grammar and catalog data. | Fully meets - the schema can be emitted from typed grammar and catalog data. | Fully meets - the schema can be emitted from typed grammar and catalog data. |
| FR-018 | Fully meets - the binary performs all work locally. | Fully meets - the binary performs all work locally. | Fully meets - the bundled runtime performs all work locally. |
| NFR-002 | Fully meets - native code and predictable memory support the 2-second check target. | Fully meets - compiled code and a small runtime support the 2-second check target. | Partially meets - V8 startup and heap behavior make the latency target depend on tuning. |
| NFR-005 | Fully meets - ownership and compact data structures keep memory bounded. | Fully meets - a small runtime keeps memory bounded. | Partially meets - a bundled JavaScript runtime raises the memory floor. |
| C-003 | PASS - produces a native binary with no runtime dependency. | PASS - produces a native binary with no runtime dependency. | FAIL - a bundled Node runtime is not a native binary and adds a runtime dependency. |
| C-005 | PASS - the stack is independent of any model provider. | PASS - the stack is independent of any model provider. | PASS - the stack is independent of any model provider. |

**Chosen option:** Rust workspace with a single binary

Rust provides sum types, exhaustive pattern matching, and ownership, which fit a compiler that must represent an AST, diagnostics, typed declarations, and target IR without shared mutable state. It produces a native binary that satisfies C-003 and supports the NFR-002 and NFR-005 targets. Go is eligible and simpler to staff, but its type system makes exhaustive declaration and diagnostic modeling more verbose. The Node option is rejected because a bundled runtime conflicts with C-003.

Considered and excluded: a C++ implementation was excluded because manual memory management raises the cost of a long-lived compiler without improving the product contract.

### Language governance and compatibility

| Criterion ID | Edition-based language versioning | CLI semantic version as language version | No formal versioning |
| --- | --- | --- | --- |
| FR-015 | Fully meets - each specification names an edition, and unknown editions are rejected. | Partially meets - the language changes with every CLI release, and a specification cannot name a stable target. | Does not meet - no mechanism rejects incompatible syntax or semantics. |
| NFR-003 | Fully meets - diagnostic codes are permanent within and across editions. | Partially meets - codes can churn with tool releases. | Does not meet - diagnostic meanings are unstable. |
| C-005 | PASS - editions are independent of any model provider. | PASS - CLI versions are independent of any model provider. | PASS - the absence of versioning is independent of any model provider. |

**Chosen option:** Edition-based language versioning

Editions separate language changes from CLI release cadence. A specification declares the edition it targets, and the CLI rejects unknown editions. Diagnostic codes are never renumbered; removed rules are deprecated with a warning and a documented replacement. CLI semantic versions remain free to ship parser, performance, and diagnostic improvements without forcing specification changes.

Considered and excluded: calendar versioning without editions does not define which syntax and semantics a specification targets.

The chosen options are compatible. Braced declarations carry the normalized widget tree. The tree compiles to a versioned IR that reference renderers consume. The hybrid ownership model keeps platform code small. The verification ladder checks each stage. The Rust CLI implements the parser, checker, interpreter, renderers, and generators as one offline binary. Together the choices satisfy all MUST requirements and constraints.

## Design

### Language and layer model

A Prism project contains one `app.prism` manifest and any number of `.prism` source files in a single project-wide namespace. Declarations are order-independent and unique. The manifest declares the edition and application name. Source files may be grouped by layer for readability, but layer membership comes from the declaration kind, not the directory.

The language uses `//` comments. The formatter preserves leading and inline comments, uses two-space indentation, one declaration per line, a fixed clause order, a maximum line width of 100 columns, and a single trailing newline.

The four clean-architecture layers map to declaration kinds:

| Layer | Declarations | May reference |
| --- | --- | --- |
| Entities | `entity`, `value`, `enum`, `error`, pure `fun` and `invariant` | Entities only |
| Use cases | `port`, `usecase`, `scenario` | Entities, errors, ports |
| Adapters | `adapter` | Entities, errors, ports, adapter configuration |
| Frameworks | `screen`, `component`, `flow`, `target`, `theme`, `style`, `token`, `app` | All inward layers |

The checker rejects outward references, cycles between layers, duplicate declarations, and adapters that do not implement a port. Ports belong to the use-case layer because they express what a use case needs. Adapters implement ports. Frameworks bind ports to adapters per target. An adapter declaration names one port, its target set, optional configuration, and the member signatures it implements. The checker compares each signature with the port and rejects missing, extra, or incompatible members. The CLI generates an anchored stub for each member; the LLM owns the body.

The type system includes `Bool`, `Int`, `Float`, `Text(min..max)`, `Date`, `Instant`, `Duration`, `Color`, `Font`, `Url`, `[T]`, `T?`, `Id<T>`, records, and enums. Entities and values are immutable. Constructors use named fields. `.with(field: value)` produces an updated copy. Invariants run on construction and update.

Use cases are the only effectful units. They declare inputs, output, required ports, typed errors, a `do` body, and optional scenarios. Effects enter only through ports, including time, identity, storage, and network. The expression subset supports `let`, `if`, `else`, `for` over finite collections, `return`, `fail`, `ensure`, `if let`, `??`, field access, constructors, built-in collection operations, and single-expression lambdas. It excludes recursion, unbounded loops, user-defined mutable globals, classes, and escape hatches. The checker enforces C-004.

Example (abridged; `NewHabit`, `HabitDetail`, `ListToday`, the remaining use cases, and adapter bodies are declared elsewhere):

```prism
edition 2026.1
app Habits

target ios {
  platform ios
  ui SwiftUI
  root "targets/ios"
  bind Habits = LocalHabits
  bind CheckIns = LocalCheckIns
  bind Clock = SystemClock
  bind Ids = UuidIds
}

target web {
  platform web
  ui React
  root "targets/web"
  bind Habits = LocalHabits
  bind CheckIns = LocalCheckIns
  bind Clock = SystemClock
  bind Ids = UuidIds
}

enum Cadence { Daily, Weekdays }

entity Habit {
  id: Id<Habit>
  name: Text(1..60)
  cadence: Cadence = Daily
  archived: Bool = false

  invariant name.trim().length > 0

  fun dueOn(day: Date) -> Bool {
    if cadence == Weekdays {
      return day.weekday != Sat && day.weekday != Sun
    }
    return true
  }
}

entity CheckIn {
  id: Id<CheckIn>
  habit: Id<Habit>
  day: Date
}

error HabitNotFound(id: Id<Habit>)
error AlreadyCheckedIn(day: Date)

port Habits {
  byId(id: Id<Habit>) -> Habit?
  save(habit: Habit) -> Habit
  all() -> [Habit]
}

port CheckIns {
  on(habit: Id<Habit>, day: Date) -> CheckIn?
  save(checkIn: CheckIn) -> CheckIn
}

port Clock {
  today() -> Date
}

port Ids {
  new<T>() -> Id<T>
}

usecase CheckInHabit(habit: Id<Habit>) -> CheckIn {
  needs Habits, CheckIns, Clock, Ids
  errors HabitNotFound, AlreadyCheckedIn

  do {
    let h = Habits.byId(habit) ?? fail HabitNotFound(habit)
    let day = Clock.today()
    if CheckIns.on(habit, day) != none {
      fail AlreadyCheckedIn(day)
    }
    let c = CheckIn(id: Ids.new(), habit: habit, day: day)
    CheckIns.save(c)
    return c
  }

  scenario "duplicate check-in is rejected" {
    given {
      let h = Habit(id: id1, name: "Read")
      Habits.byId(h.id) -> h
      Clock.today() -> 2026-11-13
      CheckIns.on(h.id, 2026-11-13) -> CheckIn(id: id2, habit: h.id, day: 2026-11-13)
    }
    when CheckInHabit(h.id)
    then fails AlreadyCheckedIn(2026-11-13)
    then no call CheckIns.save
  }
}
```

### UI model

A screen declares a route, typed data bindings, a widget tree, and optional loading, empty, and error states. The widget tree is built from a fixed primitive catalog: `Column`, `Row`, `Stack`, `Grid`, `List`, `Scroll`, `Card`, `Label`, `Button`, `Input`, `Toggle`, `Image`, `Icon`, `Spacer`, `Divider`, and `SafeArea`. The standard library adds composite components such as `EmptyState`, `ErrorBanner`, and `SkeletonList`.

Every widget has typed properties. Layout properties include `gap`, `padding`, `align`, `justify`, `grow`, `width`, `height`, and `wrap`. Components may declare typed parameters and slots. The checker rejects unknown properties, unknown widgets, recursive component cycles, and type mismatches.

Each interactive widget has an implicit semantic role from its type and requires a `label` property. Generic interactive containers must declare a `role`. Informative images require `alt` text. The checker rejects missing labels, missing roles where required, and missing alternative text.

Design tokens are typed declarations for color, typography, spacing, radius, elevation, and motion. Named styles reference tokens. Screens reference styles. Themes override token values for light, dark, and target-specific variants. A target selects a theme at runtime from the system appearance or a declared default; the specification does not encode platform appearance APIs. Target-specific token and style overrides are allowed; target-specific widget names are not. Lint rejects raw color, spacing, and radius values outside token declarations. Lint also checks declared foreground and background token pairs against WCAG AA contrast where both values are colors.

Responsive variants use width breakpoints:

```prism
ui {
  when width >= 600 {
    Row { content }
  } else {
    Column { content }
  }
}
```

The UI tree example continues the reference application:

```prism
value HabitStatus {
  habit: Habit
  due: Bool
  checked: Bool
  streak: Int
}

token color.bg: Color = #FFFFFF
token color.fg: Color = #111827
token color.muted: Color = #6B7280
token color.accent: Color = #2563EB
token color.onAccent: Color = #FFFFFF
token space.sm: Int = 8
token space.md: Int = 16
token radius.card: Int = 12
token radius.control: Int = 8
token type.title: Font = system(28, bold)
token type.bodyStrong: Font = system(16, semibold)
token type.caption: Font = system(12, regular)

theme Light {
  color.bg = #FFFFFF
  color.fg = #111827
}

theme Dark {
  color.bg = #0B1020
  color.fg = #F9FAFB
}

style title {
  font: type.title
  color: color.fg
}

style bodyStrong {
  font: type.bodyStrong
  color: color.fg
}

style caption {
  font: type.caption
  color: color.muted
}

style primary {
  background: color.accent
  foreground: color.onAccent
  padding: space.sm
  radius: radius.control
}

style card {
  background: color.bg
  foreground: color.fg
  padding: space.md
  radius: radius.card
}

component HabitRow(status: HabitStatus, onTap: Action) {
  ui {
    Row(gap: space.md, align: center, onTap: onTap) {
      Column(grow: 1) {
        Label(status.habit.name, style: bodyStrong)
        Label(status.checked ? "Checked in" : "Due today", style: caption)
      }
      Button(
        status.checked ? "Done" : "Check in",
        action: CheckInHabit(status.habit.id),
        disabled: status.checked,
        style: primary,
        label: status.checked ? "Already checked in" : "Check in habit"
      )
    }
  }
}

screen Today {
  route "/today"
  data statuses: [HabitStatus] from ListToday

  ui {
    Column(gap: space.md, padding: space.md) {
      Row(justify: spaceBetween, align: center) {
        Label("Today", style: title)
        Button(
          "New habit",
          action: navigate(NewHabit),
          style: primary,
          label: "Create a new habit"
        )
      }
      if statuses.isEmpty {
        EmptyState("No habits yet", detail: "Create your first habit.")
      } else {
        List(statuses) { status ->
          HabitRow(status, onTap: navigate(HabitDetail(status.habit.id)))
        }
      }
    }
  }

  states {
    loading { SkeletonList(rows: 3) }
    error { ErrorBanner(message) }
  }
}

flow Main {
  start Today
  screens Today, NewHabit, HabitDetail
}
```

The checker builds navigation edges from `navigate` actions and row taps, verifies that every target screen exists, and verifies that every screen in a flow is reachable from its start screen. Screen states are checked for widget validity and accessibility. Screen actions return declared use-case results. The runtime maps declared domain errors to the screen's error state and adapter failures to an infrastructure error state. UI scenarios assert widget presence, state selection, and navigation targets; layout geometry is not asserted in v0.1.

The CLI compiles the tree to a target-neutral UI IR with resolved tokens, theme values, styles, accessibility metadata, data bindings, actions, and navigation. Reference renderers for SwiftUI and React consume the IR and generate native UI code. The renderer capability matrix declares which primitive and property combinations each target supports. An unsupported combination is a `P5xxx` error before generation, not a runtime surprise.

### CLI and verification

The CLI is a single native binary named `prism`, implemented as a Rust workspace with crates for syntax, semantics, IR, UI, generators, and command-line composition. The parser is hand-written recursive descent so the grammar reference and the implementation cannot drift. Commands:

| Command | Purpose |
| --- | --- |
| `prism init` | Create a project, configuration, and agent skill files. |
| `prism check` | Parse, resolve names, check types, enforce layers, check port and adapter conformance, validate UI and target completeness. |
| `prism lint` | Apply naming, unused declaration, accessibility, raw style value, scenario coverage, and token budget rules. |
| `prism format` | Print or write canonical formatting; `--check` fails when files differ. |
| `prism run` | Execute use-case and UI-flow scenarios in the reference interpreter. |
| `prism build --target` | Generate the typed core, UI IR, target code, adapter stubs, fakes, and conformance tests. |
| `prism brief --target --task --budget` | Emit a deterministic task context pack within a token budget. |
| `prism verify --target` | Check anchors, hashes, stale generated files, target builds, and conformance tests. |
| `prism schema` | Emit the grammar, widget catalog, target capability matrix, and diagnostic catalog as JSON. |
| `prism stats` | Report specification, brief, and generated-code token and line counts. |
| `prism explain CODE` | Explain a diagnostic code with remediation guidance. |

All commands accept `--json`. Exit codes are 0 for success, 1 for findings, and 2 for usage or internal errors. Diagnostics use stable groups: `P1xxx` syntax, `P2xxx` names, `P3xxx` types, `P4xxx` architecture and bindings, `P5xxx` UI and completeness, `P6xxx` scenarios, `P7xxx` lint and budgets, `P8xxx` formatting, and `P9xxx` verification and traceability. Every diagnostic includes file, span, line, column, severity, code, message, help text, and related locations.

The pipeline is:

1. Lex and parse into an AST with spans.
2. Resolve names in a closed world and report the closest match for unknown names.
3. Build typed declarations and check arity, nullability, constraints, generics, and enum variants.
4. Enforce layer direction, port and adapter conformance, component cycles, and target bindings.
5. Validate the UI tree, tokens, styles, themes, accessibility, navigation, and flow reachability.
6. Execute scenarios in the reference interpreter.
7. Compile the checked model to a target-neutral IR.
8. Generate target code and conformance tests from the IR.
9. Verify generated hashes, LLM anchors, target builds, and conformance results.

Common failure cases and their diagnostics:

| Failure | Diagnostic group |
| --- | --- |
| Unknown declaration or misspelled reference | `P2xxx` with a suggested name |
| Type mismatch or missing field | `P3xxx` |
| Entity references a framework declaration | `P4xxx` layer violation |
| Required port has no target binding | `P4xxx` |
| Adapter misses a port member | `P4xxx` |
| Unknown widget, property, token, or style | `P5xxx` |
| Missing accessibility label or alternative text | `P5xxx` |
| Recursive component cycle | `P5xxx` |
| Scenario result or side effect differs | `P6xxx` |
| Formatter would change a file | `P8xxx` |
| Stale generated file or missing anchor | `P9xxx` |

### LLM contract and ownership

The LLM reads a task brief, edits only the listed files, and runs `prism verify`. A brief contains the edition, application, target, task, specification hash, exact file list, dependency closure of the task in canonical form, port signatures, type definitions, error types, acceptance scenarios, generated files that must not be edited, rules, and the measured token count. Briefs are deterministic: stable ordering, no timestamps, project-relative paths, Markdown for models and JSON for tools.

Ownership is explicit:

| Artifact | Owner | Regeneration |
| --- | --- | --- |
| Domain types, validation, use cases, ports, errors | CLI | Always overwritten; hash-guarded |
| UI IR, screen code, tokens, styles, navigation, fakes, conformance tests | CLI | Always overwritten; hash-guarded |
| Adapter stubs, composition root, app shell | CLI | Scaffolded once; overwritten only with `--force` |
| Adapter bodies, platform integration | LLM | Anchored and verified |
| Specification, design tokens, themes | Human or LLM author | Source of truth |

Generated files carry a header with the application, edition, target, and content hash:

```swift
// @prism-generated app=Habits edition=2026.1 target=ios hash=...
```

LLM-owned files carry an implementation anchor:

```swift
// @prism impl adapter:LocalHabits spec=...
```

`prism verify` rejects missing anchors, duplicate implementations, stale hashes, edits to generated files, and target builds that do not compile. The reference interpreter and generated target code share fixtures so behavioral drift is detected by differential tests.

Token efficiency comes from four mechanisms: a small keyword vocabulary and fixed clause order, defaults for common properties, a task-scoped brief instead of the whole repository, and canonical formatting that keeps patches small. `prism stats` reports actual token counts with a pinned tokenizer. `prism lint` enforces the NFR-001 budgets. The reference application spec is measured against equivalent native Swift and TypeScript implementations; the measurement report records both token counts and line counts.

### Target profiles

| Target | Platform and UI profile | v0.1 status |
| --- | --- | --- |
| `ios` | Swift 6, SwiftUI, Swift Package Manager, Swift Testing | End-to-end |
| `web` | TypeScript strict, React, Vite, Vitest | End-to-end |
| `android` | Kotlin, Jetpack Compose | Profile and capability matrix only |
| `windows` | C# and .NET, WinUI 3 | Profile and capability matrix only |
| `linux` | Linux, UI backend pending (GTK 4 or Tauri) | Profile and capability matrix only |
| `macos` | Swift 6, SwiftUI | Profile and capability matrix only |

A target profile declares its platform, UI backend, root path, supported capabilities, and port bindings. The checker requires a binding for every port a use case needs. The capability matrix records which primitives, properties, input methods, and platform services the target supports. Target-specific build and test commands live in `prism.toml`, outside the language, so the shared specification remains platform-neutral. Adding a backend requires a renderer, a capability matrix, and conformance tests; it does not require changing application specifications.

### Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Native UI generation across platforms is the largest delivery risk. | Ship two backends first, use a fixed primitive catalog, and reject unsupported combinations at check time. |
| A structurally valid UI tree can still look wrong. | Combine structural checks, widget presence scenarios, screenshot tests on both targets, and manual review of native idiom. |
| The expression language grows into a general-purpose language. | Freeze the v0.1 subset in the language reference. Additions require a new edition and a demonstrated need from the reference applications. |
| Interpreter and generated code drift. | Run differential tests on shared fixtures and require conformance tests in `prism verify`. |
| Token budgets vary by tokenizer and model. | Pin one tokenizer for measurement, report relative comparisons, and avoid absolute claims about model context windows. |
| An LLM edits generated files or invents symbols. | Closed-world checks, hash guards, anchors, exact brief file lists, and `prism verify`. |
| Native adapter APIs differ more than expected. | Keep adapter contracts declarative, generate stubs, and let the LLM implement bodies against platform capability ports. |
| Accessibility is treated as decoration. | Require role and label properties in the grammar, reject omissions, and lint token contrast. |

## Adoption

### Delivery phases

| Phase | Deliverable | Exit criteria |
| --- | --- | --- |
| P0 | Language reference, grammar, diagnostic catalog, reference application specification | The specification exercises every v0.1 construct and fits the token budget. |
| P1 | Lexer, parser, resolver, type checker, layer checker, formatter, check, lint, and format commands | `prism check`, `prism lint`, and `prism format --check` pass on the reference application; golden corpus covers valid and invalid input. |
| P2 | Reference interpreter, use-case scenarios, UI-flow scenarios, run command | Every reference scenario passes; failures report the expected and observed values. |
| P3 | UI IR, SwiftUI and React renderers, token and style generation, build command | Generated iOS and Web applications compile, render the reference screens, and pass widget presence and navigation tests. |
| P4 | Briefs, anchors, verify command, stats command, agent skill | Briefs fit 4,000 tokens; stale or missing anchors fail; `prism verify` is green on both targets. |
| P5 | Pilot application, measurement report, public alpha | An independent team writes a small application from the language reference and ships it on iOS and Web. |

### Compatibility and rollout

v0.1 freezes edition `2026.1`. A specification that names another edition fails with a `P1xxx` diagnostic that lists supported editions. Future editions may add constructs, change defaults, or deprecate rules; removed rules produce warnings for one edition before removal. Diagnostic codes are never renumbered. CLI semantic versions can ship parser, performance, and diagnostic improvements without changing the edition.

The rollout starts with the reference application, then a pilot team, then a public alpha. Existing projects are not migrated because no Prism specifications exist before v0.1. Generated artifacts are reproducible from the specification and CLI version. Rollback consists of reverting the specification and generated directories in Git and rebuilding; v0.1 stores no application data format that requires migration.

### Verification plan

- Unit tests for lexer, parser, resolver, type checker, layer checker, formatter, interpreter, and generators.
- Golden corpus of valid and invalid specifications with expected diagnostic codes and spans.
- Property tests for parser round-trip and formatter idempotence.
- Snapshot tests for canonical formatting, diagnostics, UI IR, and generated Swift and TypeScript.
- Differential tests that run the same fixtures through the reference interpreter and generated target code.
- Target build tests on macOS for iOS and on macOS and Linux for Web.
- Determinism tests that run generation twice and compare bytes and token counts.
- Token benchmark that compares the reference specification, task briefs, and equivalent native code with a pinned tokenizer.
- Manual review of the reference application on iOS and Web for native idiom, accessibility, and visual quality.

## Open questions

- The working name `Prism` and the `.prism` extension need a trademark and package registry review before a public alpha. Resolution: complete the review before P5.
- Visual parity needs a measurable method. Resolution: prototype screenshot comparison in P3 and set per-target thresholds before the pilot.
- The pinned tokenizer for NFR-001 is not selected. Resolution: choose one tokenizer, record it in the tooling documentation, and use it in `prism stats`.
- The Linux backend choice between GTK 4 and a Tauri-based renderer is deferred. Resolution: evaluate both against the primitive catalog after iOS and Web ship.
- UI scenarios assert widget presence, state, and navigation in v0.1. Geometry assertions may be added if screenshot and layout data prove stable. Resolution: revisit after P3.
- Distribution channels for the single binary (Homebrew, package archives, or language package managers) are undecided. Resolution: choose before the public alpha.
- Platform-specific UI needs outside the primitive catalog have no extension mechanism in v0.1. Resolution: collect pilot requests and decide whether an edition adds a constrained native component contract.

## References

- [ADR-0001: Use Makespec to manage decisions and specifications](../decisions/0001-use-makespec-to-manage-decisions-and-specifications.md) defines the record workflow used for this specification.