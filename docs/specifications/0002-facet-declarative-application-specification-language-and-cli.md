---
id: RFC-0002
title: Facet declarative application specification language and CLI
date: 2026-11-14
status: proposed
---

# RFC-0002: Facet declarative application specification language and CLI

## Context

Product developers and coding agents need a checked, versioned contract for application behavior and UI that can be used to implement native applications without copying the specification into prose, platform code, or duplicated acceptance tests. This proposal defines Facet, a declarative application specification language and offline verification CLI. The intended users are individual developers and small product teams using human developers or external, provider-independent coding models to implement applications.

This RFC is a proposal derived from `plan-gpt-6-astra.md`; it is not an accepted decision or an implementation report. Facet is a working name and has not been checked for public package or trademark availability. The plan describes the source workspace as lacking application implementation, but that status is not independently asserted here. Token reduction, readability, CLI speed, and native performance are hypotheses to measure, not demonstrated results.

The proposed product overlaps with RFC-0001, which describes Prism, but is a separate alternative: Facet assigns all native application source to humans or external coding models and does not generate native source code. Its initial proof is a local task manager implemented for iOS, Android, and Web, rather than generated implementations for iOS and Web of a habit tracker. Both proposals explore typed application specifications, clean architecture, scenario checking, and bounded LLM context. RFC-0001 remains proposed and unchanged.

Proposal assumptions: the initial user needs an offline local application, not a hosted service; application data persists locally across restarts; platform storage errors remain explicit; and a useful first release consists of the language, Rust CLI, task-manager specification, and native implementations on the three named targets. This RFC proposes measurable acceptance gates and leaves syntax selection and device-specific performance limits to explicit evaluation before implementation expands.

The initial release excludes cloud sync, authentication, collaboration, a hosted service, native source generation, a shared rendering runtime, arbitrary platform code in specifications, and desktop application implementations. Editor extensions, language servers, MCP servers, plugin marketplaces, and general-purpose language features are deferred.

## Requirements and constraints

### Requirements

| ID | Priority | Requirement |
| --- | --- | --- |
| FR-001 | MUST | The Facet language MUST express typed entities, values, enums, errors, invariants, ports, use cases, adapters, targets, and a single unnamed shared `host` entry point. |
| FR-002 | MUST | The checker MUST enforce clean-architecture references, type and name correctness, complete port contracts, target bindings, and inferred use-case dependencies and failures. |
| FR-003 | MUST | The language MUST express a typed shared UI tree with reusable components and slots, local state and forms, explicit query reloads, navigation, responsive layout, themes, design tokens, and accessibility metadata. |
| FR-004 | MUST | The CLI MUST provide `check`, `lint`, `format`, `run`, `schema`, `brief`, `verify`, and `stats` commands with documented human and versioned JSON output. |
| FR-005 | MUST | `run` MUST execute bounded domain and UI-state scenarios using controlled fakes and report mismatched results, errors, calls, logs, or navigation without using real application storage or network services. |
| FR-006 | MUST | `brief` MUST emit deterministic, dependency-complete task context with the relevant contracts, target bindings, tests, and exact token count; it MUST fail rather than truncate required context when the requested budget is exceeded. |
| FR-007 | MUST | `verify` MUST compare native test reports against the complete expected case set derived from colocated Facet tests, report stale or missing artifact mappings, and distinguish automated conformance from source-boundary and manual UI evidence. |
| FR-008 | MUST | The reference product MUST be implemented as native iOS, Android, and Web applications from the same Facet contracts, with local persistence and no shared application rendering runtime. |
| FR-009 | MUST | The reference task manager MUST support creating, editing, completing, deleting, filtering, listing, and viewing tasks, and MUST preserve data across application restarts. |
| FR-010 | MUST | Presentation actions MUST support local structured logging with fixed fields and literal event names; logs MUST exclude dynamic task data, and domain, query, and rendering code MUST NOT log. |
| NFR-001 | MUST | Formatting and machine-readable exports MUST be deterministic; formatting MUST preserve comments and semantics and be idempotent. |
| NFR-002 | MUST | Static CLI commands and scenario evaluation MUST work offline and MUST send no telemetry. |
| NFR-003 | MUST | Parsing, evaluation, input sizes, collections, child processes, output, and execution time MUST have documented bounds; limit failures MUST be explicit and MUST NOT appear as successful results. |
| NFR-004 | MUST | Token, CLI, and application performance claims MUST be backed by reproducible measurements with tokenizer, toolchain, device, OS, and fixture details where applicable. Device-specific performance gates MUST be agreed before declaring a target complete. |
| NFR-005 | MUST | Diagnostics MUST have stable codes, severity, project-relative source locations, explanatory messages, and remediation guidance; JSON output MUST follow a versioned schema. |
| FR-011 | MUST | Before full parser implementation, the project MUST compare equivalent braced and indentation-based language samples, record token counts and controlled editing/repair results, and select the initial syntax based on those results. |

### Constraints

| ID | Constraint |
| --- | --- |
| C-001 | The CLI MUST be implemented in Rust and released as native host binaries for macOS, Linux, and Windows. |
| C-002 | Humans or external coding models own native application source and native test harnesses; Facet and its CLI MUST NOT generate native source code or execute model inference. |
| C-003 | The initial application proof MUST target iOS, Android, and Web. Windows, Linux, and macOS application profiles are planned or unsupported, never reported as verified targets. |
| C-004 | The reference application MUST be local-only. Cloud sync, authentication, collaboration, and remote application services are out of scope. |
| C-005 | Facet source MUST remain platform-neutral: no native platform API snippets, arbitrary embedded platform code, network imports, or shell directives. |

## Options

Assessments below are design-time judgments based on this proposal and inspected project records. They are not measurements, completed implementation work, or evidence that the combined options have been validated in practice.

### Language surface and syntax selection

| Criterion ID | Evidence-gated selection between braced and indentation syntax | Adopt braced syntax without measurement | Adopt indentation syntax without measurement | Schema-first declarations with embedded expressions |
| --- | --- | --- | --- | --- |
| FR-001 | Fully meets - either candidate must express the same typed declaration and behavior contract before selection. | Fully meets - explicit blocks can express the required declarations. | Fully meets - indentation can express the required declarations. | Partially meets - data is representable, but executable contracts need custom embedded conventions. |
| FR-002 | Fully meets - both candidates are evaluated for type checking, architecture rules, full-port contracts, and inferred dependencies/failures. | Fully meets - block notation can express the declarations and checker rules. | Fully meets - indentation can express the declarations and checker rules. | Partially meets - the data schema needs custom mechanisms for resolved behavior and architectural checks. |
| FR-011 | Fully meets - the project measures equivalent samples and records results before selection. | Does not meet - no comparison or evidence informs the syntax choice. | Does not meet - no comparison or evidence informs the syntax choice. | Does not meet - it does not compare the two requested language candidates. |
| NFR-001 | Fully meets - the selected grammar receives a canonical, deterministic formatter. | Fully meets - a canonical formatter can normalize braces. | Fully meets - a canonical formatter can normalize indentation. | Fully meets - canonical serialization is possible. |
| C-005 | PASS - the grammar comparison does not permit platform-specific source escapes. | PASS - the grammar can forbid platform-specific source escapes. | PASS - the grammar can forbid platform-specific source escapes. | PASS - the schema can forbid platform-specific source escapes. |

**Chosen option:** Evidence-gated selection between braced and indentation syntax

The source plan includes a provisional braced example and a partial indentation alternative, but no equivalent full samples, parser results, or token measurements. Before full parser implementation, expand both candidates to the same nested UI and behavior sample. Compare the complete serialized inputs with equivalent JSON/YAML and prose baselines using a pinned tokenizer. Record controlled human editing outcomes, parser/formatter feasibility, first-pass validity, diagnostics, repair rounds, and total input tokens; label optional external-model experiments with their model and prompt settings. Select and document one syntax before full parser work. The braced examples below are illustrative only, not a syntax decision or claim of token efficiency.

### Shared interface model

| Criterion ID | One typed, shared UI and state model | Duplicated platform-specific Facet UI trees | Prose-only screen descriptions |
| --- | --- | --- | --- |
| FR-003 | Fully meets - one checked tree declares reusable elements, state, routes, themes, layout, and accessible semantics. | Partially meets - each target can specify its interface, but structure and behavior can drift. | Does not meet - prose does not provide typed bindings or checkable UI states. |
| FR-010 | Fully meets - action-scoped logs and their observable records can be specified and tested alongside UI behavior. | Partially meets - logging rules must be repeated and checked separately on each target. | Does not meet - log placement, fields, and payload restrictions are not machine-checkable. |
| C-005 | PASS - platform differences are in target profiles and bindings, not shared screen declarations. | FAIL - platform-specific UI declarations violate the platform-neutral Facet source constraint. | PASS - prose can omit platform APIs, but does not meet the functional UI requirement. |

**Chosen option:** One typed, shared UI and state model

The shared tree gives the checker and UI-state tests one platform-neutral contract. The initial primitive catalog stays bounded and expands only when the reference app or a focused fixture requires it. Each target profile maps supported properties and capabilities to native implementations; the tree does not promise pixel-identical rendering.

### Native implementation ownership

| Criterion ID | Human or external-model native implementation with Facet verification | Generate native source from Facet | Generate shared core and UI, with native integration supplied externally |
| --- | --- | --- | --- |
| FR-006 | Fully meets - briefs give implementation agents the exact contracts and dependencies they need. | Partially meets - generated code can be included, but the chosen ownership boundary is lost. | Partially meets - briefs can cover the remaining integration, but much native code is still generated. |
| FR-007 | Fully meets - native harnesses report conformance against test cases derived from Facet. | Partially meets - generated tests can run, but native source ownership and independent test sensitivity are weakened. | Partially meets - verification is possible, but generated source adds another artifact contract. |
| C-002 | PASS - humans and external models own native source; the CLI checks and exports data only. | FAIL - native source generation violates the constraint. | FAIL - generating core or UI produces native application source. |
| C-005 | PASS - target APIs stay in native projects and target bindings. | PASS - platform-specific APIs can remain outside Facet source. | PASS - platform-specific APIs can remain outside Facet source. |

**Chosen option:** Human or external-model native implementation with Facet verification

Facet does not emit Swift, Kotlin, TypeScript, SQL, or other native application source. It may export deterministic, versioned checked-model data and test fixtures. The model provider is outside the CLI, and the CLI remains provider-independent. The verifier checks reported results and artifact mappings but does not claim that hashes or model-authored tests prove behavior; harness review, deliberate faults, native builds, and platform tests supply separate evidence.

### CLI implementation stack

| Criterion ID | Rust workspace and native CLI | Go module and native CLI | TypeScript CLI with bundled runtime |
| --- | --- | --- | --- |
| C-001 | PASS - Rust is the required implementation and supports native host binaries. | FAIL - Go violates the specified implementation constraint. | FAIL - TypeScript with a bundled runtime is not the required Rust native CLI. |
| NFR-002 | Fully meets - commands can run locally without network services or telemetry. | Fully meets - commands can run locally without network services or telemetry. | Fully meets - commands can be implemented locally, subject to packaging. |
| NFR-003 | Fully meets - the CLI can enforce explicit input, evaluator, and process bounds. | Fully meets - the CLI can enforce explicit bounds. | Fully meets - the CLI can enforce explicit bounds. |
| NFR-005 | Fully meets - the CLI can publish one stable diagnostic contract across commands. | Fully meets - the CLI can publish one stable diagnostic contract. | Fully meets - the CLI can publish one stable diagnostic contract. |

**Chosen option:** Rust workspace and native CLI

The Rust stack is a scope constraint from the source plan, not a benchmark result. Keep compiler logic in a core crate and filesystem, process, and terminal I/O in a CLI crate. Dependencies and parser strategy remain implementation choices subject to the language evaluation and reproducible builds.

### Verification and application proof

| Criterion ID | Layered checks, bounded evaluator, briefs, and native conformance | Static checks and formatting only | Run scenarios only in the Facet reference evaluator |
| --- | --- | --- | --- |
| FR-004 | Fully meets - the command set includes checking, formatting, schema, briefs, verification, and measurements. | Partially meets - static commands work, but runtime and verification commands are absent. | Partially meets - behavior runs, but static diagnostics, formatting, and context export are absent. |
| FR-005 | Fully meets - evaluator scenarios control ports and observe outcomes without real services. | Does not meet - static checks do not execute scenarios. | Fully meets - scenarios can be evaluated against controlled fakes. |
| FR-006 | Fully meets - deterministic dependency closure and explicit budget failure are part of the pipeline. | Does not meet - static checks alone do not produce task-scoped context. | Does not meet - an evaluator alone does not produce verified task briefs. |
| FR-007 | Fully meets - target verification compares native reports with complete derived case sets and reports evidence boundaries. | Does not meet - no native conformance or traceability check exists. | Does not meet - reference evaluation is not native implementation evidence. |
| FR-008 | Fully meets - the shared contract is checked against all three target implementations. | Partially meets - static support does not demonstrate that each native implementation conforms. | Partially meets - evaluator scenarios do not establish native UI or persistence behavior. |
| FR-009 | Fully meets - application-level tests cover operations and restart persistence on each target. | Partially meets - the product can be built, but this approach does not verify its full behavior. | Partially meets - the evaluator can cover domain rules but not real native storage or UI. |
| NFR-001 | Fully meets - deterministic formatting and exports are checked across the pipeline. | Fully meets - static formatting and exports can be deterministic. | Partially meets - scenario output may be deterministic, but canonical formatting is not included. |
| NFR-003 | Fully meets - evaluator and process bounds produce explicit failures. | Partially meets - parser inputs can be bounded, but evaluator/process behavior is outside the approach. | Fully meets - evaluator bounds can be tested explicitly. |
| C-002 | PASS - CLI exports contracts and derived fixtures but no native source. | PASS - static tools do not generate native source. | PASS - the evaluator does not generate native source. |
| C-003 | PASS - the reference proof covers iOS, Android, and Web and marks desktop unsupported. | PASS - application scope can target the three platforms. | PASS - application scope can target the three platforms. |
| C-004 | PASS - application adapters use local persistence only. | PASS - application scope can remain local-only. | PASS - evaluator fakes avoid remote services. |

**Chosen option:** Layered checks, bounded evaluator, briefs, and native conformance

Static validation, reference scenarios, native tests, source-boundary review, and manual UI review establish different kinds of evidence. A passing scenario suite does not prove all-input equivalence, a brief does not prove correct model behavior, and a native SDK choice does not prove performance. The verifier MUST report missing test cases and unsupported targets rather than filling gaps with defaults.

### Initial application target scope

| Criterion ID | Three native targets: iOS, Android, and Web | iOS and Web only | Six targets: iOS, Android, Web, Windows, Linux, and macOS |
| --- | --- | --- | --- |
| FR-008 | Fully meets - it implements each target required for the initial proof. | Partially meets - Android proof is absent. | Fully meets - it includes the required targets plus desktop. |
| FR-009 | Fully meets - the same task-manager behavior and local persistence are required on all three. | Partially meets - the product proof omits Android. | Fully meets - it can implement the task manager on all required targets. |
| NFR-004 | Fully meets - named-device gates can be established per target before completion claims. | Partially meets - measurement omits one required mobile platform. | Partially meets - six platform gates expand scope before the three-target proof is validated. |
| C-003 | PASS - matches the required initial target set and keeps desktop status explicit. | FAIL - omits Android from the initial proof. | FAIL - treats desktop application targets as initial deliverables. |
| C-004 | PASS - local storage adapters can serve the task manager on each target. | PASS - the targets can use local storage. | PASS - target choice does not require cloud services. |

**Chosen option:** Three native targets: iOS, Android, and Web

The selected scope is the smallest candidate that includes every required initial target while keeping native desktop work out of the proof. CLI host binaries remain a separate concern and support macOS, Linux, and Windows. Native application completion requires the corresponding toolchain or device evidence; an emulator result cannot substitute for an unavailable physical-device performance measurement.

The selected options are compatible at the proposal level: the typed UI model remains platform-neutral, native source stays human- or model-owned, Rust tooling produces checked data and tests rather than source, and the layered verifier checks the same scenarios across all three targets. These judgments do not establish that the syntax, CLI, or performance targets are feasible; adoption gates below require that evidence before target completion.

## Design

### Product and language model

Facet uses `.facet` source files and a project manifest to select an edition, source root, target profiles, and native implementation roots. The proposed source layout is feature-oriented: each `src/<feature>/` owns its domain contracts, use cases, presentation declarations, and colocated `test` declarations. Feature folders do not create namespaces. Native projects live under configurable implementation roots and follow their SDK conventions. Compiler fixtures remain separate from application acceptance tests.

The model has four checked layers: entities and values; use cases and inner-owned ports; adapters; and framework-level host, screens, components, navigation, and target composition. The checker resolves declaration kinds and references rather than trusting directory names. Entities cannot depend on ports or presentation. Use cases cannot name concrete adapters. Screens cannot bypass use cases to reach repositories. Target bindings must supply compatible adapters and required capabilities. Adapter contracts cover every operation of the referenced port.

| Layer | Owned declarations | Allowed source references |
| --- | --- | --- |
| Entities | Entities, values, enums, invariants, domain errors | Domain types and pure built-ins |
| Use cases | Use cases, application errors, inner-owned ports | Entities and port contracts |
| Adapters | Adapter declarations and native implementation mappings | Inner contracts, entities, and declared adapter configuration |
| Frameworks | Shared host, screens, components, navigation, and target composition | Inward contracts and adapters through explicit target wiring |

The checker rejects outward source dependencies even when runtime calls use injected ports. Declaration initialization, component expansion, and composition dependency graphs are acyclic; navigation may cycle. Colocated test stubs and assertions are test-driver roots and do not create production dependencies or target binding requirements.

A project declares exactly one unnamed `host`; platform targets provide composition and startup. The shared declaration namespace is closed, order-independent, and case-sensitive. The first type set includes booleans, portable exact integers, text, typed identifiers, records, enums, lists, and optional values. Integer, text, whitespace, ordering, and wire semantics must be fixed before conformance implementation. UI lengths, colors, typography, radius, and motion use separate validated catalog types. Floating-point domain arithmetic, dates, time zones, package imports, recursion, unbounded loops, dynamic evaluation, arbitrary native snippets, network imports, shell directives, and user-defined higher-order functions are deferred or excluded.

The portable integer proposal is the exact JSON/JavaScript integer range, from -9007199254740991 through 9007199254740991, with checked overflow. Candidate text semantics for the syntax study use `trim` over ASCII space, tab, carriage return, and newline; `length` counts Unicode scalar values; and text ordering compares Unicode scalar sequences. These are provisional semantics from the worked example and must be retained or revised with explicit fixtures before edition 0.1 is frozen. Numeric wire encodings, equality, nullability, and line-ending behavior must also be specified before native conformance work.

Use the following naming conventions in both syntax candidates: kebab-case source filenames and feature directories (`task-editor.facet`); PascalCase types and type-like declarations (`TaskTitle`); camelCase fields, parameters, members, actions, and target names (`errorMessage`, `ios`); dot-separated camelCase token and literal log-event names (`color.background`, `task.saveFailed`); and natural-language test descriptions. Names are case-sensitive. Treat acronyms as words (`TaskId`, `uiState`). Catalog names such as `onChange` and `fieldError` retain exact spelling without aliases. The formatter preserves names and does not rename files or declarations. The linter reports noncanonical but otherwise valid names; it does not rewrite them.

Use cases contain executable bindings, constructors, conditionals, bounded collection operations, port calls, `return`, and typed `fail`. Dependencies are inferred from resolved calls; failures are inferred from fallible constructors, `try`, and declared port failures. Port signatures declare their external failures; use cases do not duplicate inferred failure lists. `try` propagates a typed failure through a use case and transfers it to exactly one matching presentation action handler. A failed action skips remaining success statements, runs its matching handler once, and does not resume the success path. Handlers cover every inferred action failure and cannot themselves perform fallible calls in edition 0.1. Optional absence is distinct from failure. Cancellation and evaluator resource limits are not domain errors.

Effectful port calls execute serially in source order; the language has no implicit parallelism, retry, or compensating transaction. Combine failures across typed branches conservatively, including branches that appear unreachable from current prose or examples. Native storage atomicity remains an adapter contract tested against the real implementation.

Use one `test` declaration for domain and UI scenarios. The first `when` subject selects the domain or screen test driver, and the checker resolves it before validating `given` setup. Domain scenarios invoke use cases and assert results, failures, and port calls; screen scenarios interact through accessible controls and observe semantic state, navigation, focus, and logs. Once a screen is open, interactions use the active navigation state; a screen test cannot switch to direct domain invocation. Test names are unique in their owning declaration scope. Unconfigured calls, invalid fixtures, absent subjects, and unsupported assertions fail explicitly. Native test harnesses consume versioned fixture data derived from these tests, report results by qualified case name, and cannot omit expected cases.

#### Provisional Facet syntax examples

The following connected fragments make the proposal concrete. The manifest is `facet.toml`; task contracts and screens belong under the feature-owned `src/tasks/` root, while the shared host and platform composition belong in `src/host.facet` and `src/targets.facet`. Tests stay beside the contract they exercise. The blocks below are grouped by concern for readability and can be split across those files. They use the braced candidate, but are not a finalized grammar, complete runnable project, or compiler-validated sample. Equivalent indentation-based examples and measurements are required by FR-011 before full parser implementation.

The manifest selects the edition, source root, targets, and native project roots:

```toml
edition = "0.1"
sources = ["src"]
targets = ["ios", "android", "web"]

[implementations.ios]
root = "platforms/ios"

[implementations.android]
root = "platforms/android"

[implementations.web]
root = "platforms/web"
```

The domain and port declarations own the title invariant and external failure contracts:

```facet
enum TaskFilter { All, Active, Completed }

error InvalidTitle
error TaskNotFound(id: Id<Task>)
error DuplicateTask(id: Id<Task>)
error StorageFailed
error IdentityFailed

value TaskTitle {
  text: Text
  invariant text == trim(text) else InvalidTitle
  invariant length(text) >= 1 && length(text) <= 120 else InvalidTitle
}

entity Task {
  id: Id<Task>
  title: TaskTitle
  completed: Bool
}

port TaskIds {
  next() -> Id<Task> errors IdentityFailed
}

port Tasks {
  create(task: Task) -> Task errors DuplicateTask, StorageFailed
  get(id: Id<Task>) -> Task? errors StorageFailed
  getAll() -> [Task] errors StorageFailed
  rename(id: Id<Task>, title: TaskTitle) -> Task
    errors TaskNotFound, StorageFailed
  setCompleted(id: Id<Task>, completed: Bool) -> Task
    errors TaskNotFound, StorageFailed
  remove(id: Id<Task>) -> Unit errors StorageFailed
}
```

The use case shows inferred dependencies and failures, plus a colocated scenario:

```facet
usecase CreateTask(title: Text) -> Task {
  let name = try TaskTitle(text: trim(title))
  let identity = try TaskIds.next()
  let task = Task(id: identity, title: name, completed: false)
  return try Tasks.create(task)
}

usecase FindTasks(filter: TaskFilter) -> [Task] {
  let tasks = try Tasks.getAll()
  let matching = tasks where task =>
    filter == TaskFilter.All ||
    (filter == TaskFilter.Active && !task.completed) ||
    (filter == TaskFilter.Completed && task.completed)
  return matching order task => task.title.text asc, task.id asc
}

usecase SetTaskCompleted(id: Id<Task>, completed: Bool) -> Task {
  return try Tasks.setCompleted(id, completed)
}

usecase RenameTask(id: Id<Task>, title: Text) -> Task {
  let name = try TaskTitle(text: trim(title))
  return try Tasks.rename(id, name)
}

usecase GetTask(id: Id<Task>) -> Task {
  let task = try Tasks.get(id)
  if let found = task {
    return found
  }
  fail TaskNotFound(id: id)
}

usecase GetEditorTask(id: Id<Task>?) -> Task? {
  if let identity = id {
    let task = try Tasks.get(identity)
    if task == none {
      fail TaskNotFound(id: identity)
    }
    return task
  }
  return none
}

usecase RemoveTask(id: Id<Task>) -> Unit {
  return try Tasks.remove(id)
}

test "invalid title causes no identity or storage effects" {
  when CreateTask(title: "   ")
  then fails InvalidTitle
  then no calls TaskIds
  then no calls Tasks
  then no logs
}

test "create trims and persists an incomplete task" {
  given {
    let identity = id<Task>("t1")
    let expected = Task(
      id: identity,
      title: TaskTitle(text: "Read"),
      completed: false
    )
    TaskIds.next() returns identity
    Tasks.create(expected) returns expected
  }
  when CreateTask(title: "  Read  ")
  then returns expected
  then calls TaskIds.next() once
  then calls Tasks.create(expected) once
}
```

`CreateTask` requires `TaskIds` and `Tasks`; its inferred failures are `InvalidTitle`, `IdentityFailed`, `DuplicateTask`, and `StorageFailed`. The scenario drives the reference evaluator with fakes and proves validation occurs before identity or storage effects. Production identities come from `TaskIds`; `id<Task>("t1")` is available only in test fixtures.

A reusable component declares a typed parameter and a content slot:

```facet
component Panel(title: Text) {
  slot content
  ui {
    Column(gap: space.medium) {
      Text(title, style: heading, role: heading)
      slot content
    }
  }
}
```

A screen declares local state, an explicit query trigger, an action with typed failure handlers, and semantic loading/error/empty/content states:

```facet
screen TaskList {
  route "/tasks"
  state filter: TaskFilter = TaskFilter.All
  state errorMessage: Text? = none
  data tasks = FindTasks(filter: filter)
    reload on enter, change(filter)

  action complete(id: Id<Task>, completed: Bool) {
    set errorMessage = none
    try SetTaskCompleted(id: id, completed: completed)
    refresh tasks

    failure TaskNotFound { refresh tasks }
    failure StorageFailed {
      log error "task.updateFailed"
      set errorMessage = "Could not update task."
    }
  }

  ui {
    Column(gap: space.medium) {
      if width >= 600dp {
        Row {
          Text("Tasks", role: heading)
          Button("New task", action: navigate(TaskEditor(id: none)))
        }
      } else {
        Column {
          Text("Tasks", role: heading)
          Button("New task", action: navigate(TaskEditor(id: none)))
        }
      }
      if let message = errorMessage {
        Text(message, role: alert)
      }
      states tasks {
        loading { Text("Loading tasks...", role: status) }
        error StorageFailed {
          Text("Could not load tasks.", role: alert)
        }
        empty { Text("No tasks match this filter.") }
        content {
          List(tasks, key: task.id) { task ->
            Row {
              Toggle(
                value: task.completed,
                label: "Completed: " + task.title.text,
                onChange: complete(id: task.id, completed: event.value)
              )
              Button(task.title.text, action: navigate(TaskDetail(id: task.id)))
            }
          }
        }
      }
    }
  }
}
```

A form screen demonstrates typed local input, validation feedback, explicit focus, inferred failure handlers, and navigation after success:

```facet
screen TaskEditor(id: Id<Task>?) {
  route "/tasks/edit"
  state errorMessage: Text? = none
  data task = GetEditorTask(id: id) reload on enter
  form draft {
    title: Text = task?.title.text ?? ""
  }

  action save {
    set errorMessage = none
    if let identity = id {
      try RenameTask(id: identity, title: draft.title)
    } else {
      try CreateTask(title: draft.title)
    }
    navigate TaskList()

    failure InvalidTitle {
      field draft.title "Enter a title between 1 and 120 characters."
      focus draft.title
    }
    failure TaskNotFound {
      set errorMessage = "This task no longer exists."
    }
    failure DuplicateTask {
      set errorMessage = "Could not create task. Try again."
    }
    failure IdentityFailed {
      set errorMessage = "Could not create task. Try again."
    }
    failure StorageFailed {
      set errorMessage = "Could not save task. Try again."
    }
  }

  ui {
    states task {
      loading { Text("Loading editor...", role: status) }
      error TaskNotFound { Text("This task no longer exists.", role: alert) }
      error StorageFailed { Text("Could not load task.", role: alert) }
      content {
        Panel(title: id == none ? "New task" : "Edit task") {
          content {
            Input(bind: draft.title, label: "Title", required: true)
            if let message = errorMessage {
              Text(message, role: alert)
            }
            Button("Save", action: save, disabled: save.pending)
          }
        }
      }
    }
  }
}

test "invalid input keeps the editor open" {
  given {
    GetEditorTask(id: none) returns none
    CreateTask(title: "   ") fails InvalidTitle
  }
  when open TaskEditor(id: none)
  when fill input("Title") with "   "
  when tap button("Save")
  then screen TaskEditor(id: none)
  then fieldError input("Title") "Enter a title between 1 and 120 characters."
  then focused input("Title")
  then no logs
}
```

The detail screen completes the list-to-detail-to-editor flow and keeps storage failure distinct from a missing task:

```facet
screen TaskDetail(id: Id<Task>) {
  route "/tasks/detail"
  state errorMessage: Text? = none
  data task = GetTask(id: id) reload on enter

  action remove {
    set errorMessage = none
    try RemoveTask(id: id)
    navigate TaskList()

    failure StorageFailed {
      log error "task.deleteFailed"
      set errorMessage = "Could not delete task."
    }
  }

  ui {
    states task {
      loading { Text("Loading task...", role: status) }
      error TaskNotFound { Text("This task no longer exists.", role: alert) }
      error StorageFailed { Text("Could not load task.", role: alert) }
      content {
        Panel(title: task.title.text) {
          content {
            Text(task.completed ? "Completed" : "Active")
            Button("Edit", action: navigate(TaskEditor(id: id)))
            Button("Delete", action: remove, disabled: remove.pending)
          }
        }
      }
    }
  }
}
```

The host uses declared design tokens and themes:

```facet
token color.background: Color = #FFFFFF
token color.foreground: Color = #111827
token space.medium: Length = 16dp
token font.heading: Font = system(size: 24sp, weight: semibold)

style heading {
  font: font.heading
  foreground: color.foreground
}

theme Light {}
theme Dark {
  color.background = #111827
  color.foreground = #F9FAFB
}
```

The shared host supplies one entry point. Adapters declare capabilities, while target declarations choose native implementations:

```facet
adapter LocalTasks implements Tasks {
  requires storage.local
}

adapter RandomTaskIds implements TaskIds {
  requires identity.random
}

host {
  start TaskList()
  themes Light, Dark
  appearance system
  background color.background
}

target ios {
  profile swiftui
  logging local
  bind Tasks = LocalTasks using sqlite.apple
  bind TaskIds = RandomTaskIds using uuid.apple
}

target android {
  profile compose
  logging local
  bind Tasks = LocalTasks using sqlite.android
  bind TaskIds = RandomTaskIds using uuid.android
}

target web {
  profile browser.dom
  logging local
  bind Tasks = LocalTasks using indexeddb.browser
  bind TaskIds = RandomTaskIds using uuid.browser
}
```

The screen scenario selects the presentation test driver and checks a semantic result:

```facet
test "task load failure is visible" {
  given {
    FindTasks(filter: TaskFilter.All) fails StorageFailed
  }
  when open TaskList()
  then visible alert("Could not load tasks.")
  then no logs
}
```

The profile and implementation keys name entries in a versioned built-in capability catalog, not code that exists merely because it is referenced. Native compilation and tests must verify those implementations.

#### Provisional language syntax reference

This table summarizes the proposed braced surface used in the examples. The manifest is TOML; catalog types, widgets, properties, and test selectors are built-ins, not declaration keywords. `Name`, `Type`, and `...` are explanatory placeholders. The grammar, token spellings, and semantics remain provisional until the FR-011 comparison selects an edition 0.1 syntax.

| Area | Representative syntax | Purpose |
| --- | --- | --- |
| Project manifest | `edition = "0.1"` and `sources = ["src"]` in `facet.toml` | Selects the language edition and source roots; TOML keys are not Facet keywords. |
| Shared entry point | `host { start TaskList() ... }` | Declares the single unnamed host and shared application start. |
| Host settings | `themes Light, Dark`<br>`appearance system`<br>`background color.background` | Selects themes, appearance behavior, and a typed background token. |
| Target | `target ios { profile swiftui ... }` | Declares one platform composition and a versioned capability profile. |
| Port binding | `bind Tasks = LocalTasks using sqlite.apple` | Connects an inner port to an adapter and compatible native implementation. |
| Entity | `entity Task { ... }` | Declares an immutable domain object with identity. |
| Value | `value TaskTitle { ... }` | Declares an immutable value and constructor invariant. |
| Enum | `enum TaskFilter { All, Active, Completed }` | Declares a closed set of variants. |
| Error | `error TaskNotFound(id: Id<Task>)` | Declares a typed failure, optionally with payload. |
| Port | `port Tasks { ... }` | Declares an inner-owned contract implemented by a target adapter. |
| Port signature | `get(id: Id<Task>) -> Task? errors StorageFailed` | Declares parameters, success type, and external failures. |
| Use case | `usecase CreateTask(title: Text) -> Task { ... }` | Defines executable application behavior; dependencies and failures are inferred. |
| Adapter | `adapter LocalTasks implements Tasks { ... }`<br>`requires storage.local` | Declares full-port implementation and required native capability. |
| Scalar types | `Bool`, `Int`, `Text`, `Unit` | Names boolean, portable integer, text, and no-value types. |
| Composite types | `Id<Task>`, `[Task]`, `Task?` | Names typed identity, ordered list, and optional values. |
| Local binding | `let task = ...` | Binds an immutable local with a uniquely inferred type. |
| State update | `set errorMessage = none` | Updates existing presentation state without mutating domain values. |
| Conditional | `if condition { ... } else { ... }` | Selects a statement or UI branch, including responsive conditions. |
| Optional narrowing | `if let task = optionalTask { ... }` | Makes the present value nonoptional within the branch. |
| Value construction | `Task(id: identity, title: name, completed: false)` | Constructs a typed value and checks its invariants. |
| Collection filtering | `tasks where task => !task.completed` | Applies a bounded, pure predicate. |
| Collection ordering | `tasks order task => task.title.text asc, task.id asc` | Orders by typed keys with an explicit tie-breaker. |
| Success return | `return task` | Returns a successful use-case result. |
| Typed failure | `fail TaskNotFound(id: id)` | Fails with a declared error and contributes to inferred failures. |
| Failure propagation | `try Tasks.get(id)` | Propagates a typed failure from a fallible expression. |
| Optional operations | `task?.title.text`; `value ?? fallback` | Accesses an optional value or supplies a fallback for absence, not failure. |
| Component | `component Panel(title: Text) { ... }` | Declares reusable UI with typed parameters. |
| Slot | `slot content`; `Panel(title: "Tasks") { content { ... } }` | Declares and supplies named component content. |
| Design token | `token space.medium: Length = 16dp` | Declares a typed design value. |
| Theme | `theme Dark { color.background = #111827 }` | Overrides base token values for a theme. |
| Style | `style heading { font: font.heading }` | Groups typed presentation properties. |
| Screen | `screen TaskDetail(id: Id<Task>) { ... }` | Declares a navigable surface with typed arguments. |
| Route | `route "/tasks/detail"` | Associates a screen with a route. |
| Screen state | `state errorMessage: Text? = none` | Declares local mutable state with an explicit initial value. |
| Query resource | `data tasks = FindTasks(filter: filter)` | Declares a use-case-backed resource with inferred value type. |
| Reload trigger | `reload on enter, change(filter)` | Specifies when a query executes; rendering alone does not reload it. |
| Form | `form draft { title: Text = task?.title.text ?? "" }` | Declares typed local fields initialized after query dependencies succeed. |
| Presentation action | `action save { ... }` | Declares event-driven presentation behavior. |
| Action failure handler | `failure StorageFailed { ... }` | Handles an inferred failure after success statements; the action does not resume. |
| UI tree | `ui { ... }` | Declares pure semantic rendering with no effects. |
| Resource states | `states tasks { ... }` with `loading`, `error`, `empty`, and `content` branches | Defines loading, typed failure, empty-collection, and content branches. |
| Navigation | `navigate TaskDetail(id: id)` | Navigates imperatively from an action or through a deferred control binding. |
| Query refresh | `refresh tasks` | Explicitly requests a query rerun. |
| Widgets | `Text(...)`, `Input(...)`, `Button(...)`, `Toggle(...)`, `List(...)`, `Row { ... }`, `Column { ... }`, `Stack { ... }`, `Scroll { ... }`, `Image(...)` | Uses the bounded, profile-checked widget catalog. |
| Accessibility | `Text(message, role: alert)`; `Input(bind: draft.title, label: "Title")` | Declares semantic feedback and accessible control names. |
| Deferred event binding | `onChange: complete(id: task.id, completed: event.value)` | Supplies a typed event value when the user acts, not during rendering. |
| Structured log | `log error "task.saveFailed"` | Emits a fixed local record in an action or its failure handler. |
| Test | `test "invalid title is rejected" { ... }` | Colocates a domain or screen scenario with its contract. |
| Test setup/stub | `given { Tasks.get(id) fails StorageFailed }` | Defines test fixtures and controlled port/use-case outcomes. |
| Domain test | `when CreateTask(title: "Read")` | Invokes a use case through the domain driver. |
| Screen test | `when open TaskList()` | Opens a typed screen through the UI-state driver. |
| Interaction/assertion | `when tap button("Save")`; `then visible alert("Could not save.")` | Drives accessible controls and checks semantic UI results. |
| Call/log assertions | `then no calls Tasks`; `then logs exactly [record]` | Checks effects and the complete ordered application log records. |
| Comment | `// explanation` | Adds a line comment preserved by formatting. |

### UI and logging model

The shared UI model includes typed screens and inputs, use-case-backed queries, local state and forms, loading/empty/content/error branches, reusable components with typed parameters and slots, and explicit navigation actions. Query types are inferred, but reload triggers remain explicit; rendering does not perform effects. A query renders loading until it resolves, then either a typed error state or content. `empty` is for an empty collection; an optional `none` is successful content, not an error or loading state. `refresh` reruns a query only when explicitly requested.

Form fields initialize after their query dependencies first succeed, including an optional `none`. They preserve edits across query refreshes and are reinitialized only by an explicit reset or a new screen instance. Each action attempt clears its previous error before execution. Action pending state prevents re-entry; screen disposal cancels outstanding work and late results cannot mutate the disposed screen. Cancellation does not undo a storage operation that already committed.

Typed route parameters must be present and valid. Native profiles map them to native navigation state; the browser profile maps them to URL query parameters. Missing or malformed required arguments produce navigation diagnostics, not guessed identities. The checker derives navigation edges from actions, permits navigation cycles, rejects cycles in declaration initialization, component expansion, and composition dependencies, and requires every screen to be reachable from `host.start`. List row keys must be unique. Interactive controls require accessible names and roles; informative images require alternatives. Alerts provide visible accessible feedback without automatically moving focus or re-announcing unchanged content. Field validation feedback and explicit focus remain separate behaviors.

Layout uses logical units and bounded responsive conditions. Typed design tokens, styles, light/dark themes, labels, accessible roles, focus, and informative image alternatives are checked against versioned profile catalogs. The initial catalog contains text, input, button, toggle, list, row, column, stack, scroll, and image, with alignment, spacing, padding, sizing, color, typography, radius, and motion properties. Each selected target must support every used widget/property/capability combination; unsupported combinations fail before briefs are emitted.

Presentation actions and their failure handlers may emit `log <level> "<event>"`, where level is `debug`, `info`, `warn`, or `error`, and the event is a nonempty literal dot-separated identifier. Each record has exactly `level`, `source`, `event`, and `error`. The checker derives `source` from the enclosing action and `error` from its active failure handler, or `none` outside a handler. Do not log task titles, entity identifiers, raw exceptions, stack traces, or other dynamic application data.

Logging occurs only when an action statement executes; rerendering does not emit a record. Query/render failures do not log automatically. Each target explicitly chooses a local sink: Apple unified logging, Android logging, or the browser console. The evaluator uses an in-memory sink. Sink failure cannot alter a business result or cause recursive logging; it is reported through a separate local diagnostic channel where available. Any buffering is bounded and has documented overflow behavior. Traces, metrics, network exporters, and durable audit delivery are deferred.

### CLI and verification contracts

The proposed native `facet` CLI exposes:

| Command | Contract |
| --- | --- |
| `facet check` | Parse, resolve names and types, enforce layer rules, and validate UI, capabilities, and target bindings. |
| `facet lint` | Report naming, unused declarations, scenario gaps, and token-budget findings; support warnings-as-errors in CI. |
| `facet format` | Preserve comments and semantics, use a canonical format, and provide non-writing `--check`. |
| `facet run` | Run bounded behavior and UI-state tests against controlled fakes. |
| `facet schema` | Export versioned language, catalog, capability, fixture, and diagnostic schemas. |
| `facet brief --target T --task ID --budget N` | Emit deterministic, dependency-complete, token-counted implementation context; fail if it exceeds the budget. |
| `facet verify --target T` | Check artifact mappings and hashes and compare native test results with the complete expected case set. Target process execution requires explicit trusted configuration and opt-in. |
| `facet stats` | Report measured specification and brief token counts with pinned tokenizer metadata. |

JSON output uses a versioned schema and deterministic ordering. Each diagnostic carries a stable code, severity, project-relative path, source span, message, remediation help, and related locations. Proposed span coordinates use UTF-8 byte offsets; line and column are one-based, columns count Unicode scalar values, and CRLF is one line break. Exit codes are 0 for success, 1 for validation or conformance findings, 2 for invocation or configuration errors, and 3 for operational or internal failures. Human diagnostics go to stderr; JSON stdout contains no terminal decoration or progress text.

`format --check` fails when canonical output differs. Formatting preserves comments and semantics, does not reorder effectful constructs or rename symbols, refuses malformed input, validates all selected files before writing, and uses atomic replacement per file. A multi-file write is not claimed to be transactional. Static commands do not execute project processes. `verify` must not evaluate shell strings from Facet source; it uses reviewed executable and argument arrays, constrains paths to project roots, checks symlink escape, bounds time and output, and cleans up owned children. Target process execution requires an explicit opt-in and trusted configuration. Local process execution is not a sandbox.

A brief includes language/schema versions, target profile, qualified declarations and tests, semantic specification hash, task boundary, expected files, dependency closure, errors and invariants, UI contracts, acceptance fixtures, bindings, and unsupported operations. Count the complete serialized pack, including instructions and metadata, using a pinned offline tokenizer and vocabulary. Never substitute character estimates, silently truncate required content, or emit a success-shaped brief from an invalid specification. Hashes detect drift but do not prove behavior or identity. Renames require explicit manifest migration where history is retained.

The implementation manifest maps qualified Facet declarations to project-relative native artifacts and records hashes and toolchain/test evidence. Formatting-only changes do not stale semantic hashes; semantic changes invalidate the affected dependency closure. Renames require explicit manifest migration when history is retained; content similarity is not identity. `verify` checks missing, duplicate, stale, and unsupported mappings and compares expected and reported test case sets.

The CLI derives versioned JSON inputs, expected outcomes, errors, and observable effects from colocated `test` declarations. It materializes fixtures in an owned temporary directory for native runners and removes them after success or failure. Do not maintain a second conformance-test tree or duplicate acceptance requirements in native projects. Each native harness invokes the actual application use cases through fake ports and reports outcomes keyed by qualified test name. Native test reports are evidence, not a second source of requirements. Review harness connections and inject deliberate faults to demonstrate test sensitivity. Report static checks, native conformance, source-boundary inspection, and manual UI review separately.

`check` establishes syntax, resolved names, types, declared dependencies, binding completeness, and supported UI/capability combinations. It cannot establish that prose requirements are correct, that native code uses the prescribed SDKs, or that the native implementation conforms. The evaluator establishes only outcomes for the scenarios executed within its bounded semantics; finite passing tests do not prove equivalence for all inputs. Hashes and traceability identifiers detect drift, not behavior or model correctness.

### Reference application

The proof is a task manager with an opaque task identity, a nonblank bounded title, and completion state. The provisional `TaskTitle` invariant accepts 1 through 120 Unicode scalar values after trimming the candidate ASCII whitespace set described above; invalid titles fail before identity or storage calls. Use cases create, retrieve, rename, complete or uncomplete, remove, and filter tasks. Listing supports All, Active, and Completed filters with deterministic title ordering and identity tie-breaks. Repeated completion with the current value succeeds; deleting an absent task succeeds, so repeated deletion is idempotent. Missing tasks and storage failures are distinct typed outcomes.

Port operations expose typed storage and identity failures. `get` returns `none` only when a task is absent, never when storage failed. `rename` changes only the title; `setCompleted` changes only completion; each mutation is atomic and preserves unrelated fields. Adapters translate expected platform failures into declared port errors. Tests cover invalid and valid titles, prove invalid construction performs no identity or storage calls, and inject identity and storage failures.

Persist data locally across restarts. Use platform-local persistence through declared ports: SQLite-backed storage on iOS and Android and IndexedDB on Web are proposed starting adapters. Serialize mutations to avoid lost updates. Never convert failed loads into empty results. Cloud sync, authentication, collaboration, arbitrary network access, and cross-platform rendering are excluded. Application scenarios also cover create/edit, complete/uncomplete, filtering, missing identities, delete, restart persistence, loading/error/empty states, navigation, themes, responsive layouts, keyboard operation, and accessibility.

The iOS application is an Xcode application target with simulator UI tests; a Swift package may be used for isolated domain tests but is not the application-delivery plan by itself. Android uses a Gradle application module, lifecycle-safe state, and emulator/device tests. Web uses strict TypeScript, browser-native DOM controls, IndexedDB, and browser tests, with no React or shared rendering runtime. Each target requires native build, domain conformance, real persistence integration, and UI/accessibility checks. Desktop application profiles report planned or unsupported until separate specifications and evidence support them.

### Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| A bounded language still grows into a general-purpose language. | Version the language, reject unbounded constructs, and add features only to meet an exercised product contract. |
| Facet contracts and native behavior drift. | Share derived fixtures, compare complete case sets, inject faults, and review harnesses. |
| Three platform implementations exceed proof capacity. | Complete a thin vertical slice on all targets before expanding flows; report unavailable hardware or SDKs rather than substituting evidence. |
| Token or readability claims are overstated. | Compare semantically equivalent full input packs with a pinned tokenizer and controlled editing tasks. Publish measurements and limitations. |
| UI-state tests miss platform presentation defects. | Add target UI, accessibility, responsive, screenshot, and human native-idiom review. |
| LLM-authored tests or reports appear complete without exercising the application. | Inspect harness connections, require every expected case, and prove sensitivity with deliberate faults. |
| Resource or process limits hide failures. | Return explicit diagnostics and operational errors; bound parser, evaluator, output, time, and child processes. |

## Adoption

This RFC proposes the following ordered gates. A passing gate permits the next dependent work; it does not mark that work complete.

| Gate | Deliverable | Exit criteria |
| --- | --- | --- |
| A. Product contract and syntax | Complete Facet examples, equivalent brace and indentation candidates, grammar and semantics proposal | Record pinned-tokenizer baselines, readability/edit results, parser feasibility, rejected syntax cases, and select syntax before full parser development. Define resource limits and edition policy. |
| B. Rust core and CLI | Workspace, project loader, parser, typed model, diagnostics, architecture and target checks, formatter, `check`, `lint`, and `format` | Positive/negative fixture corpus covers types, naming, architecture, bindings, host cardinality, optional values, errors, malformed inputs, deterministic diagnostics, and formatter round-trip/idempotence. |
| C. Evaluator and UI contracts | Bounded evaluator, `run`, versioned fixtures, UI-state model, capabilities, structured logs | Tests cover success, failure, forbidden effects, bounds, UI states, handler behavior, accessibility, routes, and deterministic export. Invalid or incomplete fixtures fail explicitly. |
| D. Briefs and first vertical slice | `schema`, `brief`, `stats`, manifest, `verify`, and create/list/persist flow on all three targets | Briefs are deterministic and budget-enforced. Native behavior matches shared fixtures, survives restart, rejects deliberate faults, and has approved device/browser performance thresholds. Record device, OS, browser, and toolchain versions; measure release-build cold launch, list interaction/rendering, persistence latency, peak memory, and Web bundle size at fixed small and 1,000-task fixture sizes. Measure CLI startup/check time and peak memory on a fixed synthetic corpus. Keep token counts separate from runtime benchmarks. |
| E. Complete reference applications | Full task manager on iOS, Android, and Web | iOS passes Xcode build, domain conformance, SQLite integration, simulator UI tests, and named-device gates. Android passes Gradle build, domain conformance, storage integration, emulator UI tests, and named-device gates. Web passes strict type checks, production build, domain conformance, IndexedDB integration, browser UI tests, and browser performance gates. Human review confirms accessibility and platform idiom. |
| F. Release hardening | Fuzz/resource tests, process/path safety tests, release smoke tests, guides, measurement and evidence report | Reproducible CLI host packages work on macOS, Linux, and Windows. Report actual support separately from planned desktop profiles and publish no unsupported performance or token claims. |

The syntax choice, edition compatibility policy, evaluator limits, UI catalog, language wire semantics, tokenizer, and target-specific performance gates must be settled or explicitly versioned at the gate that depends on them. A positive/negative fixture corpus covers each architecture rule, type/error condition, unsupported UI combination, and invalid input family. Property tests cover parse/format semantic equivalence, comment preservation, idempotence, and deterministic exports; fuzz tests cover malformed syntax and resource boundaries. CLI timing and memory should be measured on a fixed synthetic corpus. Application performance should be measured on named devices and browser/toolchain versions using the fixed small and 1,000-task fixtures; simulator correctness does not substitute for physical-device performance. Token reduction must compare full equivalent specifications and briefs, including schema and instruction overhead.

Because this is a new specification and no Facet application format is established here, no application-data migration is specified. Future incompatible language changes require explicit editions and compatibility diagnostics. A failed rollout can revert the RFC and implementation changes without altering the preserved source plan. Public name/package review and CLI distribution channels must be resolved before public release.

## Open questions

- Which syntax candidate performs best in equivalent-content editing and repair tests? Resolve at Gate A before freezing grammar.
- What exact input, collection, recursion-in-data, step, process-time, and output limits are appropriate? Establish explicit defaults and boundary tests before evaluator and process verification ship.
- Which language edition and diagnostic-code stability policy should the first release use? Resolve before accepting external specifications.
- Should the provisional integer range, ASCII trim set, Unicode-scalar length and ordering rules become edition 0.1 semantics? Specify equality, nullability, text/wire encodings, and numeric serialization in the same decision; add fixtures before native conformance work.
- Which pinned tokenizer and complete-pack budget should `stats`, `lint`, and `brief` use? Select and publish the tokenizer and vocabulary before making size claims.
- What device, OS, browser, and latency thresholds define acceptable performance on each target? Approve reproducible gates at the first vertical slice before declaring a target complete.
- Is Facet available as a public package/trademark name, and which binary distribution channels should be supported? Check before public release.

## References

- [ADR-0001: Use Makespec to manage decisions and specifications](../decisions/0001-use-makespec-to-manage-decisions-and-specifications.md) defines the authoring and lifecycle process for this proposed RFC.
- [RFC-0001: Prism cross-platform product specification framework](0001-prism-cross-platform-product-specification-framework.md) is a related proposed alternative with overlapping language and verification goals, but different application source ownership, target coverage, and reference product.
- [Facet implementation plan](../../plan-gpt-6-astra.md) is the source handoff migrated into this proposal. It remains unchanged and is not an accepted specification.