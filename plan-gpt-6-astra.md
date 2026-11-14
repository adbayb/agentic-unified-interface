# Facet implementation plan

## Problem and approach

Build a human-readable, token-conscious declarative product specification language with clean architecture as checked semantics. An offline Rust CLI validates the specification, evaluates bounded behavior scenarios, and supplies external LLMs with exact implementation context. LLMs implement native applications from the same specification for iOS, Android, and Web.

Facet is the confirmed working name, not a verified public package or trademark. This plan is a handoff for future implementation. It does not authorize implementation now or change any existing project record.

## Inspected current state

Project root: `/Users/ayoubadib/Developer/poc-new-language`.

- No application source, Cargo manifest, package manifest, tests, or build pipeline were found. The workspace is not a Git repository.
- `docs/decisions/0001-use-makespec-to-manage-decisions-and-specifications.md` is accepted. It governs ADR/RFC authoring and lifecycle.
- `docs/specifications/0001-prism-cross-platform-product-specification-framework.md` is proposed, not accepted. It covers similar language goals but requires deterministic native code generation and an iOS/Web reference application. Its targets and performance assertions are proposal requirements, not measured results.
- `plan-deepseek.md`, `plan-glm.md`, and `plan-minimax.md` describe Weave, Blueprint, and Strata alternatives. They contain conflicting syntax, ownership, naming, and delivery choices. They are context, not requirements for Facet.
- The installed Makespec skill reports version 1.1.2; the available binary reports 0.4.2. No record validation or application tests were run for this planning task.

Preserve these files. During implementation, create a separate proposed Facet RFC using Makespec and its returned identifier; do not overwrite Prism or assume the next record number. Link Prism as an alternative and explain Facet's different implementation ownership and three-platform proof.

## Confirmed decisions and scope

| Area | Confirmed direction |
| --- | --- |
| Initial product | Language, verification CLI, and a small cross-platform proof of concept |
| CLI implementation | Rust; native host binaries |
| Application implementation | External, provider-independent LLMs; no model invocation inside the CLI |
| Code ownership | LLM/human-owned native application source; no native source-code generators |
| Behavior | Small executable rule/transition language, one `test` declaration for behavior and UI, and reference evaluator |
| Observability | Inline structured logs in presentation actions/handlers only; traces, metrics, and network exporters deferred |
| Architecture | Entities, use cases, adapters, and frameworks are language-level concepts |
| Shared entry point | Exactly one unnamed `host`; targets provide platform-specific composition |
| UI | Shared widget/layout tree, reusable components, design tokens, themes, responsive variants, navigation, and accessibility |
| Syntax | Infer redundant metadata; reuse `try`/`if`; compare braces and indentation before choosing |
| Naming | Kebab-case Facet files/folders, PascalCase type-like declarations, camelCase members; enforce through linting |
| Initial targets | Swift/SwiftUI on iOS; Kotlin/Jetpack Compose on Android; TypeScript and browser-native DOM APIs on Web |
| Reference product | Local-only task manager with list, editor, and detail screens |
| Features | Create, edit, complete, delete, and filter tasks; persistence across restarts |
| Performance | Native implementations without a shared application runtime; establish device-specific gates during the first vertical slice |
| Later targets | Windows, Linux, and macOS application implementations |

Exclude cloud sync, authentication, collaboration, native source-code generation, arbitrary embedded platform code in specifications, a hosted service, and a cross-platform rendering runtime from this proof. Defer editor extensions, LSP/MCP servers, plugin marketplaces, and general-purpose language features.

Distinguish CLI host support from application target support: release the CLI for macOS, Linux, and Windows; initially demonstrate applications only on iOS, Android, and Web. Desktop application profiles must report planned or unsupported, never verified.

## Verification claims and limits

`check` establishes syntax, resolved names, types, declared dependencies, binding completeness, and supported UI/capability combinations. It cannot establish that prose requirements are correct, that an LLM used the prescribed SDKs, or that native code implements the specification.

The evaluator establishes outcomes for the scenarios executed within its defined semantics. Native builds, shared conformance fixtures, adapter tests, UI tests, source-boundary checks, and human review supply separate implementation evidence. Passing finite tests does not prove equivalence for all inputs. Hashes and traceability identifiers detect drift; they do not prove behavior or prevent hallucinations.

Treat token reduction, readability, and native performance as measurements. Neither short keywords nor native SDK selection alone establishes those outcomes.

## Technical design

### Rust boundaries and proposed layout

Use a small Cargo workspace with two crates. Organize compiler capabilities into cohesive modules with a pure core, and keep filesystem, process, and terminal I/O in the CLI crate. Do not copy application-layer folders into every compiler module.

```text
Cargo.toml
Cargo.lock
rust-toolchain.toml
crates/
  facet-core/
    src/
      lib.rs
      syntax/
      checking/
      formatting/
      evaluation/
      ui/
      context/
      model/
      diagnostics/
    tests/
  facet-cli/
    src/
      main.rs
      check/
      lint/
      format/
      run/
      schema/
      brief/
      verify/
      stats/
      project/
    tests/
tests/fixtures/
  valid/
  invalid/
  formatting/
  specification-tests/
  protocol/
examples/tasks/
  facet.toml
  src/
    host.facet
    targets.facet
    tasks/
      task.facet
      ports.facet
      adapters.facet
      create-task.facet
      get-task.facet
      get-editor-task.facet
      find-tasks.facet
      rename-task.facet
      set-task-completed.facet
      remove-task.facet
      task-list.facet
      task-editor.facet
      task-detail.facet
      presentation.facet
  platforms/
    ios/
    android/
    web/
benches/
docs/
  decisions/
  specifications/
  language.md
  cli.md
  llm-protocol.md
  measurements.md
.github/workflows/
```

The `.facet` extension remains provisional; source filenames follow the naming conventions below. Modules remain private unless a real caller needs a contract. `facet-core` owns source spans, typed checked declarations, UI catalogs, scenario semantics, and diagnostics. `facet-cli` loads projects, invokes the core, renders output, and controls explicitly authorized target processes.

For Facet applications, organize contracts by feature under `src/<feature>/`. The task-manager proof has one `tasks` feature owning its domain, ports, adapters, use cases, screens, and presentation components. Keep each use case or screen's `test` statements in the same `.facet` file. Do not create global layer directories or a separate application acceptance-test tree. Shared bootstrap and target composition stay at `src/host.facet` and `src/targets.facet`.

Colocation changes file organization, not runtime dependencies: validate `test` declarations as test-driver roots, but exclude their stubs and assertions from production dependency inference and target binding requirements.

`platforms/` contains the independent native projects and their native test harnesses, not copies of Facet contracts. These paths are configurable; the language does not require that directory name. Native tests remain colocated according to each SDK's module/test conventions. The repository's `tests/fixtures/` directory above belongs to the compiler's own test corpus, not the reference application's authored acceptance tests.

Prefer Rust enums, exhaustive matching, `Result`, and functions over class-like abstractions. Start with `clap`, `serde`/`serde_json`, `toml`, and focused error-reporting dependencies as needed; evaluate parser and tokenizer libraries during the relevant milestone. Pin toolchains, dependency resolution, and measurement assets. Do not build a plugin system.

### Language and project model

One manifest selects local source roots, the edition, enabled target profiles, and implementation roots. Load `.facet` contracts and colocated tests recursively from the configured `src` root. Feature folders group ownership; they do not implicitly create namespaces or change declaration/test identities when files move. Each project declares exactly one unnamed `host` for shared bootstrap and presentation settings. Targets provide its platform-specific bindings and native startup implementation; `host` does not represent a running process. Reject missing, duplicate, or named host declarations. No network imports or arbitrary shell directives in language files. Begin with a project-wide declaration namespace, qualified member references, order-independent declarations, and duplicate-name errors; defer package imports.

The parser preserves comments and source spans. Syntax must provide explicit public types, readable keyword names, unambiguous blocks, and one canonical format. Allow local inference only where the checker derives a unique type. Reject unknown keywords and editions, undeclared names, implicit effects, and undocumented defaults.

The first type set covers booleans, portable integers, text, opaque typed identifiers, records, enums, lists, and optional values. Use immutable values and typed errors. Bound portable integers to the exact JSON/JavaScript integer range, detect overflow, and specify arithmetic and comparison semantics. Freeze text length, whitespace normalization, ordering, nullability, and numeric wire encodings before target conformance work. Keep UI lengths/colors as separate validated types; postpone floating-point domain arithmetic, dates, and time-zone behavior until needed.

The task manager does not require dates or ambient time. Identity comes through a declared port. Use-case bodies contain bindings, constructors, `if` branches, bounded collection operations, typed port calls, `return`, and `fail` directly. Infer required ports from resolved calls instead of repeating a dependency list. Port references still represent injected inner-owned contracts; target composition supplies their implementations, not global singletons. Exclude recursion, unbounded loops, dynamic evaluation, arbitrary native snippets, and user-defined higher-order functions.

Declare port failures rather than hiding I/O failure as absence or an empty list. Define cancellation and serial effect order. The reference evaluator uses scenario-controlled fakes, never real application storage or network calls. Bound evaluation steps, collection sizes, recursion in parsed structures, and input sizes with documented errors. Bounded evaluation does not imply real platform I/O always terminates.

#### Naming conventions

Use one naming convention in both syntax candidates and their catalogs:

| Scope | Convention | Examples |
| --- | --- | --- |
| Facet source filenames and feature folders | kebab-case; keep the extension separate | `create-task.facet`, `task-editor.facet`, `task-management/` |
| Entities, values, enums, errors, ports, use cases, adapters, screens, components, themes, and enum variants | PascalCase | `TaskTitle`, `CreateTask`, `LocalTasks`, `TaskFilter.Active` |
| Fields, parameters, locals, port operations, actions, state, resources, forms, slots, styles, and target names | camelCase | `errorMessage`, `getAll`, `openTask`, `draft`, `ios` |
| Token names and literal log-event names | Dot-separated camelCase segments | `color.background`, `space.extraSmall`, `task.saveFailed` |
| Catalog properties, built-ins, and test selectors/assertions | camelCase | `onChange`, `fieldError` |
| Quoted test descriptions and user-facing text | Natural language, without a case rule | `test "invalid input keeps the editor open"` |

Names are case-sensitive. Treat acronyms as words, as in `TaskId` and `uiState`. Hyphens separate filename words, not identifier words; do not make expression parsing depend on spaces around subtraction. Keywords and catalog constants retain their documented spellings, such as `host`, `none`, and `alert`. The unnamed `host` needs no identifier.

Apply these rules to authored Facet source paths and names. Preserve tool-mandated filenames such as `Cargo.toml` and each native SDK's naming conventions. TOML keys, routes, and external profile/capability/implementation keys follow their own contracts; the Facet linter does not rewrite them.

`facet lint` reports naming violations with the expected spelling. Use the existing warnings-as-errors CI mode to enforce them; valid but noncanonical user identifiers are lint findings rather than type errors. Catalog spellings such as `onChange` and `fieldError` remain exact, with no alternate-case or snake-case aliases. `facet format` preserves names and never renames symbols, files, test descriptions, or log events. Explicit symbol renames must update references and implementation-manifest identities together.

#### Typed failures and propagation

Declare each error type once. Ports have no executable bodies in the specification, so their signatures explicitly list failures: `-> T errors E1, E2`. Use cases declare their successful return type but omit error lists; the checker derives their failures from executable bodies. `return value` succeeds; `fail error` reports a known error type. Authors need no wrapper types or success/error constructors.

`try expression` evaluates to the successful value or exits the current execution path with its typed failure. In a use case, that failure propagates unchanged to the caller. The checker infers a deduplicated union of errors from `fail` and fallible expressions propagated through `try`, including constructors whose invariants declare failures. Combine all typed branches conservatively; do not guess failures from prose or assume a runtime branch is impossible. Reject unknown error types and unhandled fallible expressions.

Fallible value constructors use `try` in executable bodies just like fallible port operations; pure construction still performs no I/O. Test setup validates fixture constructors before execution and rejects invalid fixtures. Keep `errors` clauses on external port contracts only, rather than adding optional duplicate annotations to use cases.

Presentation actions also use `try`, but handle failures in trailing `failure ErrorType { ... }` clauses inside the action. Ordinary statements run in order; a failed `try` skips the remaining statements, runs the matching handler once, and ends the action. Successful calls continue on the ordinary path, without a success block. Handlers follow all ordinary statements, must cover the inferred failure set, and cannot contain `try` or direct fallible calls in v0.1. A handler can request `refresh`; that query reports failures through its own resource states. A handled failure does not resume the success path.

Query `error` states cover the query's inferred failure contract. Export inferred ports, query types, and failures in schemas and briefs. Changing an invariant or port contract recomputes dependent contracts and invalidates stale context or incomplete handlers. No control-flow construct implicitly logs or retries an operation.

Absence remains a successful optional value: `get(id: Id<Task>) -> Task? errors StorageFailed` returns a task or `none`, or reports `StorageFailed`. A value of type `T` can be passed where `T?` is expected; narrowing still requires `if let` or an explicit fallback. No optional wrapper constructor is needed. Declared and inferred error sets cover business and adapter failures, not every possible host failure. Cancellation remains a distinct execution outcome; evaluator resource-limit failures and native crashes must not masquerade as domain errors or successful results. Native adapters translate expected SDK failures into the port's declared error set; unexpected faults remain explicit operational failures.

Use versioned, deterministic checked-model and fixture schemas for machine consumers. This is data export, not native source-code generation. Fail exports on invalid specifications; never emit success-shaped partial models.

The [worked syntax example](#worked-syntax-example-provisional) makes these constructs concrete. It covers the proposed v0.1 surface with a connected task-manager specification, rather than leaving the syntax entirely to T2.

### Clean architecture rules

| Layer | Owned constructs | Allowed references |
| --- | --- | --- |
| Entities | Entities, values, enums, invariants, domain errors | Domain types and pure built-ins |
| Use cases | Inputs/outputs, rules/transitions, application errors, required ports | Entities and inner-owned port contracts |
| Adapters | Declarative implementations of ports, mappings, presenters | Inner contracts, entities, and declared adapter configuration |
| Frameworks | Shared `host`, screens, components, navigation, target composition and platform bindings | Inward contracts and adapters through explicit wiring |

Check declaration kinds and resolved references, not directory names alone. Ports belong to the inner application layer. Adapters declare supported port members and capabilities without embedding Swift, Kotlin, SQL, or JavaScript bodies.

Disallow entities referencing ports/frameworks, use cases referencing concrete adapters, and UI actions/data bindings bypassing use cases to access repositories. Inferring dependencies does not grant access to forbidden layers. Verify target-specific bindings, signature compatibility, and required capabilities. An adapter's `implements` clause requires the entire port contract, without an operation-name list. Native compilation and conformance tests must establish that its implementation supplies those operations. Separate the source dependency graph from runtime call direction: injected calls may reach an outer adapter without an outward source dependency.

Specify which graphs must be acyclic: declaration initialization, component expansion, and composition dependencies. Navigation may contain cycles such as returning from detail to list. Derive its edges from navigation actions, check typed arguments, and require every declared screen to be reachable from `host.start`. Separate named navigation groups are deferred.

Native applications use feature slices with inward dependency boundaries. Language checks cover the specification graph; platform module/import rules and focused source review cover actual native source. Keep that distinction visible in verification reports.

### Shared UI and platform capabilities

Implement a complete but bounded UI model, not prose-only screen descriptions:

- Typed screen inputs, queries, local form state, actions, and loading/empty/content/error states.
- Action feedback through optional screen state and conditional `Text(..., role: alert)` rendering, without a separate notification primitive.
- Reusable components with typed parameters and slots, with expansion-cycle detection.
- Initial primitives: text, input, button, toggle, list, row, column, stack, scroll, and image. Add a primitive only when the reference product or a focused fixture exercises its contract.
- Layout properties for alignment, spacing, padding, sizing, and bounded responsive breakpoints expressed in logical units.
- Typed color, typography, spacing, radius, and motion tokens; named styles; light/dark theme overrides.
- Navigation targets and arguments, selection behavior, keyboard focus, accessible names/roles, informative image alternatives, and validation feedback.

Define how state and bindings change after actions, how loading and failure states resolve, and how form errors map from typed domain errors. A testable UI-state model produces semantic widget/state/navigation observations without pretending to render native pixels.

Use explicit profile catalogs for SwiftUI, Compose, and browser DOM mappings. Check each declared primitive/property/capability combination for the selected targets; report unsupported combinations before creating briefs. Shared product declarations contain no native widget names. Profile mappings and adapter bindings own platform differences.

Native code implements the validated UI contract. Check semantic widget presence, labels, state transitions, and navigation with target UI tests; inspect screenshots per platform and breakpoint. Do not require pixel-identical rendering across different operating systems.

Initial real service capabilities are local storage and identity. Unknown or unimplemented capability requests fail. Model permission/failure contracts before adding later capabilities such as notifications, camera, or biometrics.

### Structured logging in v0.1

Use one statement: `log <level> "<event>"`, inside a presentation action or its failure handler. Levels are `debug`, `info`, `warn`, and `error`; the event is a nonempty literal identifier such as `"task.saved"`, not interpolated text. Entities, use cases, widget trees, and render-state branches cannot log.

The fixed application record contains `level`, `source`, `event`, and `error`. The compiler supplies the enclosing action's qualified name as `source` and the active failure handler's error kind as `error`, or `none` outside a failure handler. No field selectors, payload expressions, separate policy blocks, automatic lifecycle events, or cross-layer reporting rules are needed.

`try` propagates declared failures without logging. Presentation handlers decide whether to log an outcome and still provide user feedback. The same executed statement emits one record; rerendering does not execute actions. Query/render errors remain visible in the UI but produce no automatic logs. Query lifecycle instrumentation is deferred; a retry action can explicitly log the retry.

Each target explicitly selects its native local sink with `logging local`: Apple unified logging, Android logging, or the browser console. The evaluator uses an in-memory sink. Sink failures never change business results or trigger recursive logging; adapters report delivery failures through a separate local diagnostic channel where available, and tests exercise this failure path. Any buffering must be bounded with documented overflow behavior.

Check statement placement, severity, literal event names, and target sink availability. Test exact records and the absence of dynamic application data. Do not capture titles, entity IDs, raw exceptions, or stack dumps; review literal event names for secrets too. Defer traces, metrics, network exporters, and durable audit delivery. Application logs do not authorize CLI telemetry.

### CLI contract

All commands below are proposed interfaces, not existing executables.

| Command | Contract |
| --- | --- |
| `facet check` | Syntax, names, types, architecture, UI, capabilities, and target bindings |
| `facet lint` | Naming-convention findings, unused declarations, scenario gaps, and token-budget violations |
| `facet format` | Canonical comment-preserving formatting; `--check` performs no writes |
| `facet run` | Execute specification behavior and UI-state scenarios with bounded fakes |
| `facet schema` | Export versioned language/catalog/diagnostic contracts |
| `facet brief --target T --task ID --budget N` | Emit a verified, dependency-complete task context pack |
| `facet verify --target T` | Report traceability, build/test evidence, and conformance for one target |
| `facet stats` | Report measured specification and brief token counts with tokenizer metadata |

Defer `init`, graph visualization, code generation, and separate diagnostic-explanation commands unless the proof establishes a need. Ship human output and versioned JSON envelopes, with deterministic ordering, stable diagnostic codes, severity, relative paths, spans, related locations, and remediation help. Define span units and line/column conventions, including Unicode and CRLF cases.

Exit codes: 0 success, 1 validation/conformance findings, 2 invalid invocation or configuration, 3 operational/internal failure. Lint reports warnings by default and supports an explicit warnings-as-errors CI mode. Formatting differences fail `format --check`. Never mix terminal decoration or progress text into JSON stdout.

Formatting must preserve semantics and comments, be idempotent, and avoid reordering effectful constructs. Refuse writes for malformed files. Validate all selected files before writing and use atomic replacement per file; report any operational failure without claiming a project-wide transaction.

### LLM context and conformance

Use the deterministic checked model as the common source for briefs, schemas, scenario fixtures, and UI contracts. Share catalogs in code; conformance fixtures must expose grammar/reference drift. A handwritten parser alone does not keep documentation synchronized.

Each brief contains its schema/language versions, target profile, qualified declaration/test names, semantic spec hash, task boundary, expected files, dependency closure, relevant UI/contracts, acceptance fixtures, and unsupported operations. Include target bindings, errors, invariants, and global constraints relevant to the task, not just directly referenced declarations.

Pin a tokenizer implementation, version, and vocabulary assets. Count the complete serialized output, including instructions and metadata. If the required closure exceeds the budget, fail with its measured size and suggest a smaller task. Do not truncate required context or silently substitute character estimates. Support offline counting without telemetry or runtime model downloads.

An implementation manifest maps project-relative declaration names to target artifacts and records hashes and toolchain/test evidence. Use `CreateTask` for a top-level declaration and `TaskEditor.save` for a member; the unnamed host adds no namespace prefix or identity annotation. Test names must be unique within their owning scope and use that scope for qualification. Semantic hashes detect changes, not identity. Formatting-only specification changes should not stale semantic hashes; semantic changes invalidate the affected dependency closure. Renames require explicit manifest migrations when preserving history; never infer identity from matching content. Missing/duplicate mappings, stale evidence, and unsupported targets fail verification.

Native application source and native test harnesses remain human/LLM-owned. Conformance means checking that each native implementation satisfies the same authored contracts and tests; it does not require an authored directory. The CLI derives versioned JSON inputs, expected outcomes, errors, and observable effects from `test` statements colocated with contracts. Materialize fixtures in an owned temporary directory for the native runners and clean it up after verification. Do not maintain a separate `conformance/` tree or duplicate acceptance expectations in native projects.

Each native test harness invokes the actual application use cases through fake ports and reports results keyed by qualified test name. The verifier compares expected and observed case sets and results, rejecting missing cases, not just unsuccessful cases. Retain reports as derived evidence when requested, never as a second source of acceptance requirements. Run separate integration tests against real persistence adapters using the platform project's normal test locations.

Tests and reports are not trusted merely because an LLM authored them. Review harness connections and introduce deliberate behavioral faults to show that conformance tests fail. Report source-boundary coverage and manual UI checks separately from automatic fixture conformance.

By default, static commands do not execute project processes. Running target builds/tests requires an explicit execution option on `verify` and reviewed executable/argument arrays in trusted project configuration. Do not evaluate shell strings from specifications. Constrain output paths to project roots, check symlink escape, bound process time/output, and clean up owned child processes. Local execution is not a sandbox.

## Reference application

Use a small task model with identity, nonblank bounded title, and completion state. Define all/active/completed filtering and deterministic list ordering. Use cases cover creation, title editing, setting completion, deletion, queries, and not-found failures.

Implement durable local storage through ports: SQLite through platform APIs on iOS/Android and IndexedDB on Web are the proposed starting adapters. Serialize mutations through native transactional/atomic operations; avoid read-modify-write lost updates and define repeated completion/deletion behavior in fixtures. Report storage errors to the UI; do not replace failed loads with empty lists. Store application data only in platform-approved application locations.

Use an Xcode application target for the iOS app and simulator UI tests, with a Swift package only where useful for isolated domain tests. Swift Package Manager alone is not the iOS app-delivery plan. Use an Android Gradle application module with emulator/device tests. Use strict TypeScript, DOM APIs, a lightweight development/build tool, and browser tests for Web; no React or shared rendering runtime.

Scenario coverage includes invalid/blank titles, successful creation, edit, complete/uncomplete, filtering, unknown identity, deletion, persistence after restart, injected storage failures, repeated actions, empty/loading/error screens, navigation, light/dark themes, responsive layout, and keyboard/accessibility behavior. Keep backend-specific persistence tests separate from pure-domain fixtures.

## Ordered implementation todos and acceptance gates

These are implementation tasks, not work completed by this planning session. Dependencies are recorded in the session database.

### T1. Establishing the Facet product contract

Create a separate proposed Facet RFC through Makespec, retaining the existing alternatives. Record the confirmed scope, verification limits, ownership, host/target distinction, feature-driven `src/` layout with colocated tests, and measurement methodology. Confirm the local-only storage/error policy and approve the bounded widget catalog. Define runtime resource limits and the language-edition policy.

Acceptance: a coherent proposed record passes Makespec validation; no Prism or alternative-plan edits; blocking semantic choices have owners and gates before dependent work.

### T2. Comparing syntax and freezing executable semantics

Expand the provisional worked example below into equivalent braced and indentation-based task-manager samples, including nested UI, themes, responsive branches, errors, and scenarios. Measure complete input packs using the same pinned tokenizer and identical semantic content. Compare with equivalent structured JSON/YAML and prose specification baselines, not just native code volume.

Run controlled human readability/edit tasks and optional external-LLM repair experiments with recorded model/prompt settings. Count first-pass validity, diagnostics, repair rounds, and total input tokens. Run a small lexer/parser/formatter feasibility spike where necessary; discard the losing implementation.

Present the measurements for user selection before building the full parser. Freeze the grammar, types, effects, built-ins, UI-state semantics, error behavior, and wire schemas for the initial edition. Use one `test` declaration for both use-case and screen subjects; resolve allowed setup and assertions from the typed subject without a test-kind keyword or field. Prefer readability and repair reliability over an unsubstantiated token-saving target.

Keep the simplified surface in both candidates: infer port dependencies and query types, use direct use-case bodies, share `try` between use cases and actions, use `if` for responsive branches, and derive navigation from one unnamed host entry. Do not retain aliases for removed constructs.

Carry the [naming conventions](#naming-conventions) into both grammars, catalogs, and examples. Specify identifier boundaries independently of whitespace around operators.

Acceptance: one approved syntax with equivalent-content measurements, unambiguous worked examples, an initial invalid corpus, and documented semantic edge cases.

### T3. Bootstrapping Rust and landing the checked-language core

Create the two-crate workspace, pinned toolchain, lockfile, test fixtures, and initial CI. Build project loading, lexing/parsing, spans, name/type checking, layer checks, initial bindings, canonical formatting, diagnostics, and `check`, `lint`, `format`.

Add naming diagnostics for source paths and authored names. Cover PascalCase declarations, camelCase members and dotted event segments, kebab-case paths, acronym handling, and the documented exemptions. Verify warning versus CI-error behavior, case-sensitive resolution, exact catalog spellings, and formatter preservation of names.

Acceptance: a small entity/use-case/port/adapter specification passes; misspellings, wrong types, malformed input, layer shortcuts, missing bindings, and unknown editions fail with stable diagnostics. Infer failures and port dependencies from constructors, `fail`, branches, and port calls; reject redundant metadata and recompute dependent contracts when a port or invariant changes. Verify exactly one unnamed host per project, project-relative identities, optional widening without implicit narrowing, and full-port adapter contracts. Formatter round-trip and idempotence hold; JSON and exit-code integration tests pass.

### T4. Implementing behavior evaluation and scenario fixtures

Add the bounded evaluator and `run`, fake ports, explicit failures, effect-call records, and deterministic JSON fixture export derived from colocated tests. Cover the task use cases without native I/O or logging. Establish the typed checked-model representation used by later consumers.

Acceptance: scenarios test success and failure, state changes and forbidden effects; step/resource-limit failures are explicit; malformed or underspecified fixtures fail rather than inventing defaults.

### T5. Implementing the shared UI and capability model

Add typed components, layout, tokens, styles, light/dark themes, responsive variants, navigation, accessibility checks, UI-state scenarios, and three real target catalogs. Add inline `log` statements in presentation actions/handlers with inferred source/error fields and an in-memory sink. Reject logs in domain/render code; test records, sink failures, and no emission on rerender. Add unsupported desktop-profile status.

Acceptance: reject unknown widgets/properties/tokens, component recursion, invalid actions/routes, inaccessible controls, unsupported target combinations, and handlers missing an inferred failure. Verify `try` skips the remaining action body on failure, runs exactly one handler, and never resumes the success path. Infer query types without removing explicit reload triggers. Cover navigation reachability from the shared host entry, valid navigation loops, responsive `if` boundaries, loading/error transitions, optional-state alert rendering, and clearing stale action errors before retry. Verify alerts do not steal focus or repeat on unchanged rerenders.

### T6. Delivering model-independent briefs and verification contracts

Implement `schema`, `brief`, `stats`, manifest/hash rules, and `verify` orchestration. Derive scenario and UI expectations from colocated `test` statements and provide temporary data artifacts to native harnesses, cleaning up owned fixtures after success or failure. Document the external-LLM loop: author contracts and tests together, check/lint/format, run tests, obtain a brief, implement native code, run native verification, repair from diagnostics.

Acceptance: repeated exports are byte-identical; closures contain required context; over-budget output fails; invalid specs produce no valid brief; stale/missing mappings and missing test cases fail. Static commands execute no project processes.

### T7. Proving one native vertical slice and setting performance gates

Implement create/list/persist for one screen on all three platforms from the same checked specification. Connect native use cases, fake-port conformance harnesses, real storage, native local log sinks, and a minimal UI test. Verify inline logging without payload leakage and sink failures that leave application outcomes unchanged. Introduce a faulty implementation on each target to demonstrate test sensitivity.

Record device/OS/browser/toolchain versions and benchmark release builds with fixed fixtures at small and 1,000-task sizes. Measure cold launch, list interaction/rendering, persistence latency, peak memory, and Web bundle size. Separate simulator correctness from physical-device performance.

Measure CLI startup/check time and peak memory on a fixed synthetic corpus as well. Record token counts separately from runtime benchmarks.

Acceptance: the slice runs on iOS, Android, and Web, matches the reference fixtures, survives restart, rejects the deliberate faults, and has approved device-specific performance thresholds before expanding the apps. Do not declare a platform complete if required hardware or SDKs are unavailable.

### T8. Completing the iOS task manager

Complete edit/detail/navigation/filter/delete flows, native storage failures, full UI contract, accessibility, responsive behavior, and themes. Keep domain and use cases independent of SwiftUI/storage imports.

Acceptance: Xcode build, domain conformance, SQLite integration, simulator UI tests, and named-device performance gates pass; manually inspect native idiom and accessibility.

### T9. Completing the Android task manager

Complete the same product behavior in Kotlin/Compose, with lifecycle-safe state and platform-local persistence. Keep domain/use cases independent of Compose/database types.

Acceptance: Gradle build, domain conformance, storage integration, emulator UI tests, and named-device performance gates pass; manually inspect native idiom and accessibility.

### T10. Completing the Web task manager

Complete the same behavior in strict TypeScript with DOM controls and IndexedDB. Handle reload, storage failures, keyboard focus, responsive themes, and native browser semantics without a shared application runtime.

Acceptance: type checks, production build, domain conformance, IndexedDB integration, browser UI tests, and agreed browser performance gates pass.

### T11. Hardening the CLI and cross-platform delivery

Add malformed-input fuzz/property coverage, resource-boundary cases, path containment, process timeout/cancellation tests, compatibility fixtures, and release smoke tests on macOS/Linux/Windows. Run Rust formatting, Clippy, targeted tests, and the complete proof-of-concept matrix.

Publish local measurement artifacts covering syntax, whole-brief tokens, repair experiments when available, CLI performance, and native performance. Write a concise user guide and LLM protocol grounded in implemented behavior. Check public naming availability before publishing; do not claim it has been checked already.

Acceptance: reproducible CLI packages, all three reference applications passing the defined gates, an honest verification coverage report, and documentation that distinguishes supported behavior from planned desktop work.

## Dependency graph

```text
T1 -> T2 -> T3 -> T4
                 |
                 +-> T5
T4 + T5 -> T6 -> T7
T7 -> T8
T7 -> T9
T7 -> T10
T8 + T9 + T10 -> T11
```

T4 and T5 can progress independently after the core contract exists. T8, T9, and T10 are independent platform workstreams after the shared thin slice and harness contracts stabilize. Keep one owner for common language semantics and fixtures.

## Validation strategy

- Rust unit/integration tests for diagnostics, semantics, bindings, UI validation, process behavior, and command contracts.
- A positive/negative language corpus; one focused failing case for each architecture rule and supported error condition.
- Property checks for parse/format semantic equivalence, comment preservation, idempotence, and deterministic exports; fuzz malformed input and resource limits.
- Shared fixture comparisons between reference evaluation and real native use cases, including omitted-case detection and deliberate mutation failures.
- Platform-specific integration/UI/accessibility checks and per-platform screenshots, not unsupported claims of universal visual equivalence.
- Pinned-tokenizer comparisons on equivalent semantics; report full spec, schema overhead, briefs, and repair-loop input separately.
- Performance thresholds and device baselines approved at T7, then enforced per target. Native framework choice alone is not a performance result.

Candidate implementation commands include `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`, followed by each target's checked-in build/test commands. Final project paths and SDK destinations must come from the implemented projects, not guessed commands in this plan.

## Risks and follow-on work

| Risk | Mitigation |
| --- | --- |
| Full UI scope exceeds a small proof | Keep the requested modeling features, bound the widget catalog and demo, and prove one screen end to end before expansion |
| Three native implementations drift | Shared independent fixtures, real adapter tests, boundary checks, and reviewed test harnesses |
| DSL becomes another general-purpose language | Edition gate, bounded operations, no native escape hatches, and additions justified by product examples |
| Token savings hide lost detail | Compare semantically equivalent specifications and count the complete input pack |
| UI behavior passes while visual quality fails | Per-platform screenshots, accessibility testing, and native human review |
| LLM emits plausible but untested evidence | Execute reviewed harnesses, verify complete case sets, inject faults, and distinguish metadata from behavior |
| Performance criteria remain vague | Establish named-device thresholds in T7 and block completion on unresolved measurements |
| SDK/device availability blocks a target | Record the blocker; do not substitute a host build or emulator benchmark for the missing result |
| Future native targets differ in capability | Add a profile and conformance suite per target; permit explicit unsupported diagnostics and edition changes when needed |

After the proof, evaluate native macOS, Windows, and Linux profiles against the actual UI/capability contracts. Native frameworks such as SwiftUI, WinUI, and GTK are candidates, not decisions made by this plan. Do not promise that arbitrary future platform features require no specification changes.

Non-blocking release choices include public package naming, binary distribution channels, and exact desktop SDK choices. Resolve these before their respective release milestones. Syntax selection and performance thresholds are explicit implementation gates, not silently assumed decisions.

## Language tokens

Reference for the current proposed language surface. The table includes language primitives and the built-in/catalog forms used by the examples; catalog types, widgets, properties, and test selectors are not additional declaration keywords. `Name`, `Type`, `T`, and `...` are explanatory placeholders. Apply the [naming conventions](#naming-conventions) to names in these forms. Braces illustrate blocks; the indentation candidate has the same concepts. The normative grammar, parser, and token measurements remain T2 work.

| Primitive or form | Syntax | Description |
| --- | --- | --- |
| Shared entry point | `host { ... }` | Exactly one unnamed shared bootstrap declaration per project. |
| Initial screen | `start TaskList()` | Sets the host's navigation entry; every screen must be reachable from it. |
| Available themes | `themes Light, Dark` | Selects declared themes for the host. |
| Appearance | `appearance system` | Uses the platform's appearance preference to select the theme. |
| Host background | `background color.background` | Uses a declared color token for the application background. |
| Entity | `entity Task { ... }` | Declares an immutable domain object with identity. |
| Value object | `value TaskTitle { ... }` | Declares an immutable domain value without entity identity. |
| Enumeration | `enum TaskFilter { All, Active, Completed }` | Declares a closed set of named variants. |
| Error type | `error InvalidTitle` or `error TaskNotFound(id: Id<Task>)` | Defines a typed failure once, optionally with a typed payload. |
| Invariant | `invariant condition else InvalidTitle` | Requires a pure domain condition on construction; names the failure if it does not hold. |
| Port | `port Tasks { ... }` | Defines an inner-owned contract whose implementation is supplied by a target. |
| Operation signature | `get(id: Id<Task>) -> Task? errors StorageFailed` | Declares parameters, successful return type, and explicit external failures. |
| External failures | `errors E1, E2` | Lists failures on a port operation; use-case failures are inferred instead. |
| Use case | `usecase CreateTask(title: Text) -> Task { ... }` | Defines application behavior directly in its body; dependencies and failures are inferred. |
| Adapter | `adapter LocalTasks implements Tasks { ... }` | Declares a native implementation of an entire port contract. |
| Implemented contract | `implements Tasks` | Requires all operations and failure contracts of the referenced port. |
| Required capability | `requires storage.local` | Declares the adapter capability required from its native implementation. |
| Platform target | `target ios { ... }` | Supplies platform composition for the shared host. |
| Platform profile | `profile swiftui` | Selects a versioned catalog of native UI mappings and supported capabilities. |
| Port binding | `bind Tasks = LocalTasks using sqlite.apple` | Binds a port to a logical adapter and a compatible native implementation. |
| Local logging sink | `logging local` | Selects the target profile's local sink; required when presentation code logs. |
| Design token | `token space.small: Length = 8dp` | Defines a typed, reusable design value. |
| Theme | `theme Dark { color.background = #111827 }` | Overrides base token values; omitted tokens retain their base values. |
| Style | `style heading { font: font.heading }` | Groups typed presentation properties using declared tokens. |
| Component | `component Panel(title: Text) { ... }` | Defines reusable, typed UI. |
| Content slot | `slot content` | Declares a required component slot; the same form inside `ui` renders it. |
| Slot content | `Panel(title: "Tasks") { content { ... } }` | Supplies a named slot at a component call site. |
| Screen | `screen TaskDetail(id: Id<Task>) { ... }` | Defines a navigable presentation surface with typed arguments. |
| Route | `route "/tasks/detail"` | Associates a screen with a route; native profiles map typed arguments to navigation. |
| Screen state | `state errorMessage: Text? = none` | Declares mutable presentation state with an explicit initial value. |
| Query resource | `data tasks = FindTasks(filter: filter)` | Declares a use-case-backed resource whose value type is inferred. |
| Reload triggers | `reload on enter, change(filter)` | Explicitly reloads a resource on screen activation or a named state change; rendering alone does not reload it. |
| Form | `form draft { title: Text = task?.title.text ?? "" }` | Initializes form fields after their query dependencies succeed; preserves edits until reset or a new screen instance. |
| Presentation action | `action save { ... }` or `action complete(id: Id<Task>, completed: Bool) { ... }` | Executes presentation statements and handles inferred failures locally. |
| Action pending state | `save.pending` | Read-only execution state, usable to disable controls while an action runs. |
| Action failure handler | `failure StorageFailed { ... }` | A trailing action clause that handles a failed `try` and ends that action without resuming its success path. |
| UI tree | `ui { ... }` | Declares pure widget rendering; domain effects and logging are forbidden here. |
| Resource rendering | `states tasks { ... }` | Chooses a resource's loading, error, empty, or content rendering branch. |
| Loading branch | `loading { Text("Loading...") }` | Renders while a query is pending. |
| Resource error branch | `error StorageFailed { ... }` | Renders a typed query failure; branches must cover the inferred query failures. |
| Empty branch | `empty { Text("No tasks.") }` | Renders a successful empty collection, not a failed query. |
| Content branch | `content { ... }` | Makes a successfully resolved query value available for rendering. |
| Local binding | `let task = ...` | Binds an immutable value with an inferred type. |
| State update | `set errorMessage = none` | Updates existing presentation or form state; does not mutate domain values. |
| Conditional | `if condition { ... } else { ... }` | Selects statements or UI branches; also handles responsive conditions such as `width >= 600dp`. |
| Optional narrowing | `if let task = optionalTask { ... }` | Exposes a nonoptional value only in the present-value branch. |
| Value selection | `condition ? a : b` | Selects one of two compatible values. |
| Success return | `return task` | Returns a successful value from a use case. |
| Typed failure | `fail TaskNotFound(id: id)` | Exits a use case with a known error type and contributes to its inferred failures. |
| Failure propagation | `try Tasks.get(id)` | Evaluates to a value or exits the current path with a typed failure; use cases propagate it, actions run a matching local handler. |
| Inline log | `log error "task.saveFailed"` | Emits a fixed structured record in presentation code; source and active error kind are inferred. |
| Log levels | `debug`, `info`, `warn`, `error` | Allowed severities; event names must be literal identifiers without dynamic payloads. |
| Navigation | `navigate TaskList()` or `action: navigate(TaskList())` | Navigates imperatively in an action or supplies a deferred navigation binding to a control. |
| Query refresh | `refresh tasks` or `action: refresh(tasks)` | Requests another query execution; its result uses the resource's normal rendering states. |
| Field feedback | `field draft.title "Title is required."` | Associates a validation message with a form field and its accessible input. |
| Explicit focus | `focus draft.title` | Requests focus for the input bound to the named form field. |
| Form reset | `reset draft` | Explicitly reinitializes a form; this is distinct from refreshing its backing query. |
| Collection filter | `tasks where task => !task.completed` | Applies a bounded pure predicate; does not introduce first-class functions. |
| Collection ordering | `tasks order task => task.title.text asc, task.id asc` | Applies ordered ascending keys, including an explicit tie-breaker. |
| Field/parameter type | `title: Text` | Gives a field or parameter an explicit type. |
| Return type | `-> Task` | Declares a successful operation or use-case return type. |
| Scalar types | `Bool`, `Int`, `Text`, `Unit` | Boolean, portable exact integer, Unicode text, and no-value result. |
| Scalar literals | `42`, `"Read"`, `true`, `false` | Writes integer, text, or boolean values directly. |
| Identity type | `Id<Task>` | Represents an opaque identity belonging to one entity type. |
| List type and literal | `[Task]`, `[a, b]`, `[]` | Declares an ordered list type or constructs a list value. |
| Optional type and absence | `Task?`, `none` | Represents a present value or absence; `Task` can widen to `Task?` without a wrapper. |
| Value construction | `Task(id: identity, title: name, completed: false)` | Constructs a domain value using named fields and checks its invariants. |
| Member reference | `TaskEditor.save`, `task.title.text` | Names a project-relative declaration member or accesses a typed value field. |
| Operation call | `Tasks.get(id)` or `Tasks.get(id: identity)` | Uses positional arguments in declaration order or named arguments; arbitrary dynamic calls are not supported. |
| Optional access | `task?.title.text` | Accesses a field only when the optional receiver is present. |
| Optional fallback | `value ?? fallback` | Supplies an explicit alternative for absence, never for a failed operation. |
| Boolean literals and operators | `true`, `false`, `!`, `&&`, `\|\|` | Represents boolean values, negation, conjunction, and disjunction. |
| Comparison operators | `==`, `!=`, `<`, `<=`, `>`, `>=` | Compares compatible typed values. |
| Text concatenation | `"Open " + task.title.text` | Combines text values without interpolation syntax. |
| Text built-ins | `trim(text)`, `length(text)` | Uses the pinned whitespace and Unicode length semantics defined below. |
| Test identity literal | `id<Task>("t1")` | Creates a typed identity in a test fixture; production identities come through a port. |
| Unit value | `unit` | Represents the successful value of type `Unit`. |
| UI catalog types | `Role`, `Action`, `Action<Bool>`, `Color`, `Length`, `Font`, `Radius`, `Motion`, `ImageSource` | Typed presentation concepts; parameterized forms are fixed catalog types, not user-defined generics. |
| UI literals and font | `#FFFFFF`, `8dp`, `16sp`, `120ms`, `system(size: 16sp, weight: regular)` | Represents colors, logical lengths, scalable typography, motion duration, and a platform-neutral font. |
| Text widget | `Text(message, role: alert)` | Renders text; semantic roles include heading, status, and alert. |
| Input widget | `Input(bind: draft.title, label: "Title", required: true)` | Binds an accessible input to form state. |
| Button widget | `Button("Save", action: save, disabled: save.pending)` | Binds an accessible button to an action. |
| Toggle widget | `Toggle(value: task.completed, label: "Completed", onChange: complete(event.value))` | Renders a boolean control and forwards its typed change event. |
| List widget and row binding | `List(tasks, key: task.id) { task -> ... }` | Renders keyed rows; the row binding scopes both the key expression and row body. |
| Layout widgets | `Row { ... }`, `Column { ... }`, `Stack { ... }`, `Scroll { ... }` | Provides horizontal, vertical, overlaid, and scrollable content. |
| Image widget | `Image(source: source, alt: caption)` | Renders an image with an accessible alternative description. |
| Widget/style properties | `gap: space.small`, `padding: space.medium`, `align: center`, `width: 240dp`, `height: 160dp`, `style: panel` | Catalog-checked layout/style properties; appearance properties include font, foreground, background, radius, and transition. |
| Typed event binding | `complete: complete(id: task.id, completed: event.value)` | Defers an action until its typed callback supplies the event value; no effect runs during rendering. |
| Test declaration | `test "blank title is rejected" { ... }` | Colocated with its contract in the same source file; the first typed subject determines the domain or screen driver. |
| Test setup | `given { ... }` | Declares test-local fixtures and stubbed outcomes. |
| Successful stub | `Tasks.get(id) returns task` | Supplies a successful value for a matching call; unconfigured calls fail the test. |
| Failed stub | `Tasks.get(id) fails StorageFailed` | Supplies a typed failure for a matching call. |
| Stub sequence | `FindTasks(...) sequence {`<br>`  fails StorageFailed`<br>`  returns []`<br>`}` | Supplies outcomes in call order using separate lines. |
| Test viewport | `viewport width: 390dp` | Sets the logical viewport width in test setup. |
| Test appearance | `appearance Light` | Selects a declared theme in test setup. |
| Domain test invocation | `when CreateTask(title: "Read")` | Invokes a use case through the domain test driver. |
| Screen test entry | `when open TaskList()` | Opens a typed screen through the presentation test driver. |
| Input interactions | `when tap button("Save")`, `when fill input("Title") with "Read"` | Drives controls by accessible names. |
| Return assertion | `then returns expected` | Compares the successful use-case value with the expected value. |
| Failure assertion | `then fails InvalidTitle` | Checks the reported error kind and any expected payload. |
| Call assertions | `then calls Tasks.create(task) once`, `then no calls Tasks` | Checks observable port calls or their absence. |
| Navigation assertion | `then screen TaskEditor(id: none)` | Checks the active screen and typed navigation arguments. |
| Visibility assertion | `then visible alert("Could not save task.")` | Checks semantic UI presence using selectors such as `text`, `button`, and `alert`. |
| Field-error assertion | `then fieldError input("Title") "Title is required."` | Checks input-associated validation feedback. |
| Focus assertion | `then focused input("Title")` | Checks which input has focus. |
| Log assertions | `then logs exactly [record]`, `then no logs` | Checks complete ordered application records, or their absence. |
| Structured test record | `{ level: error, source: "TaskEditor.save", event: "task.saveFailed", error: "StorageFailed" }` | Describes the fixed application-log shape; `error` is `none` outside a failure handler. |
| Comment | `// explanation` | A line comment preserved by formatting. |
| Blocks and separators | `{ ... }`, `( ... )`, `,`, `:`, `=` | Delimits blocks/calls and separates arguments, typed fields/properties, or initializers; punctuation is not an extra declaration. |

The manifest's `edition`, `sources`, `targets`, and implementation paths are TOML configuration, not Facet keywords. The [worked example](#worked-syntax-example-provisional) defines the context and behavior behind these forms; the [indentation alternative](#indentation-alternative) shows the other block notation. Removed forms are listed separately below and are not aliases.

## Worked syntax example (provisional)

This is a proposed surface syntax, not a finalized grammar or compiler-validated program. The connected examples below cover the planned language constructs and the task-manager flow. Braces make this first draft readable without deciding the T2 comparison; an indentation alternative follows. No parser exists yet, and no token counts or syntax-validity results are claimed.

The specific file names and `.facet` extension are provisional; the naming convention applies to both syntax candidates. Read the blocks as one project with a single declaration namespace. Forward references resolve across files. `src/tasks/` owns the task feature across all layers; declaration kinds, not paths, determine architectural restrictions. Tests follow the contract they verify in the same file.

### Example conventions and built-ins

The simplification keeps architecture, behavior, UI, and verification as distinct concepts while removing repeated metadata:

| Removed form | Canonical replacement |
| --- | --- |
| `needs`, use-case `errors` | Dependencies and failures inferred from the checked body |
| `do` wrapper | Statements directly inside `usecase` |
| Adapter `members` | Full port contract required by `implements` |
| `invoke` and `success` blocks | `try`, ordinary statements, and trailing action `failure` handlers |
| `ensure` | `if` with `fail` |
| `some(value)` | Pass `T` where `T?` is expected |
| Responsive `when` | UI `if`; `when` remains test vocabulary |
| Typed `data ... from ...` | `data name = UseCase(...)` with inferred value type |
| `flow` and named app references | One unnamed `host` with `start`; navigation edges inferred from screen actions |

Keep `entity` versus `value` for identity versus value semantics; `port` versus `adapter` for inward contracts versus native implementations; and `form`, slots, themes, and `reload` for their distinct lifecycle or composition semantics. Public types, target bindings, and external failures stay explicit. These are deliberate boundaries, not candidates for merging solely to shorten the keyword list.

The [Language tokens](#language-tokens) table is the syntax reference; the notes here define the examples' remaining semantic conventions.

For this candidate, `trim` removes leading/trailing ASCII space, tab, carriage return, and newline; `length` counts Unicode scalar values. Title validation does not normalize Unicode. Text sorting compares Unicode scalar sequences, and identity breaks title ties. `id<Task>("t1")` creates a scenario-only typed identity literal; production identities come from a port. A failed constructor reports its declared invariant error.

The UI catalog supplies the widgets and typed property names used below. `dp` denotes logical layout units, `sp` denotes user-scalable typography units, and `ms` denotes motion duration. `system(...)` is the catalog's platform-independent font constructor. Catalog intrinsics also include `text(...)`, `navigate`, `refresh`, `focus`, and `reset`.

User-operation arguments are either positional in declaration order or named; reject missing, repeated, and unknown arguments. Catalog widgets document their positional content and named properties. Local bindings infer their types; fields, state initialized to `none`, and public parameters/results retain explicit types. `if` has the same boolean-condition meaning in use-case, action, and UI contexts; UI branches remain pure. `where` and `order` remain bounded collection clauses rather than a general function/lambda system.

These are candidate semantic definitions for review, not additional accepted requirements. T2 must retain or revise them explicitly and add the corresponding fixtures.

### Project manifest

`facet.toml` owns tooling paths and the edition. The unnamed host owns shared bootstrap, presentation settings, and navigation entry; it does not duplicate tooling configuration.

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

### Entities, values, enums, and errors

`src/tasks/task.facet`:

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
```

`TaskTitle` owns the title rule; use cases do not repeat it. The absence of entity mutation syntax is intentional. Ports return new immutable snapshots after atomic updates.

### Inner-owned ports and executable use cases

`src/tasks/ports.facet`:

```facet
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

`src/tasks/create-task.facet`:

```facet
usecase CreateTask(title: Text) -> Task {
  let name = try TaskTitle(text: trim(title))
  let identity = try TaskIds.next()
  let task = Task(id: identity, title: name, completed: false)
  return try Tasks.create(task)
}

test "create trims the title and persists an incomplete task" {
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

test "invalid title causes no identity or storage effects" {
  when CreateTask(title: "   ")
  then fails InvalidTitle
  then no calls TaskIds
  then no calls Tasks
  then no logs
}

test "identity failure skips storage" {
  given {
    TaskIds.next() fails IdentityFailed
  }
  when CreateTask(title: "Private task title")
  then fails IdentityFailed
  then no calls Tasks
}

test "storage failure propagates through try" {
  given {
    let identity = id<Task>("t1")
    let task = Task(
      id: identity,
      title: TaskTitle(text: "Read"),
      completed: false
    )
    TaskIds.next() returns identity
    Tasks.create(task) fails StorageFailed
  }
  when CreateTask(title: "Read")
  then fails StorageFailed
  then calls Tasks.create(task) once
}
```

`src/tasks/get-task.facet`:

```facet
usecase GetTask(id: Id<Task>) -> Task {
  let task = try Tasks.get(id)
  if let found = task {
    return found
  }
  fail TaskNotFound(id: id)
}

test "missing task reports a domain error" {
  given {
    let identity = id<Task>("missing")
    Tasks.get(identity) returns none
  }
  when GetTask(id: identity)
  then fails TaskNotFound(id: identity)
}
```

`src/tasks/get-editor-task.facet`:

```facet
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
```

`src/tasks/find-tasks.facet`:

```facet
usecase FindTasks(filter: TaskFilter) -> [Task] {
  let tasks = try Tasks.getAll()
  let matching = tasks where task =>
    filter == TaskFilter.All ||
    (filter == TaskFilter.Active && !task.completed) ||
    (filter == TaskFilter.Completed && task.completed)
  return matching order task => task.title.text asc, task.id asc
}

test "storage failure is not an empty task list" {
  given {
    Tasks.getAll() fails StorageFailed
  }
  when FindTasks(filter: TaskFilter.All)
  then fails StorageFailed
}

test "active filtering retains deterministic title order" {
  given {
    let a = Task(
      id: id<Task>("a"), title: TaskTitle(text: "Alpha"), completed: false
    )
    let b = Task(
      id: id<Task>("b"), title: TaskTitle(text: "Beta"), completed: false
    )
    let c = Task(
      id: id<Task>("c"), title: TaskTitle(text: "Closed"), completed: true
    )
    Tasks.getAll() returns [b, c, a]
  }
  when FindTasks(filter: TaskFilter.Active)
  then returns [a, b]
}
```

`src/tasks/rename-task.facet`:

```facet
usecase RenameTask(id: Id<Task>, title: Text) -> Task {
  let name = try TaskTitle(text: trim(title))
  return try Tasks.rename(id, name)
}
```

`src/tasks/set-task-completed.facet`:

```facet
usecase SetTaskCompleted(id: Id<Task>, completed: Bool) -> Task {
  return try Tasks.setCompleted(id, completed)
}
```

`src/tasks/remove-task.facet`:

```facet
usecase RemoveTask(id: Id<Task>) -> Unit {
  return try Tasks.remove(id)
}
```

For example, the checker infers that `RenameTask` requires `Tasks` and can fail with `InvalidTitle` from construction or `TaskNotFound`/`StorageFailed` from the port call. `GetTask` and `GetEditorTask` add `TaskNotFound` through `fail`. No use-case annotations repeat those dependencies or errors.

`Tasks.rename` changes only the title; `Tasks.setCompleted` changes only completion. Each operation is atomic and preserves unrelated fields. Setting the current completion value succeeds without changing it. Removing an absent task succeeds, so repeated deletion is idempotent. `get` distinguishes absence from storage failure. Port signatures alone cannot prove these guarantees: adapter conformance fixtures and real database tests must exercise them.

### Declarative adapters

`src/tasks/adapters.facet`:

```facet
adapter LocalTasks implements Tasks {
  requires storage.local
}

adapter RandomTaskIds implements TaskIds {
  requires identity.random
}
```

`implements` requires all operations and error contracts of the named port. `requires` declares an adapter capability that cannot be inferred from a body because adapter bodies are native code. Target bindings select compatible native implementations; native compilation and tests verify their actual signatures and behavior.

### Tokens, styles, themes, and reusable components

`src/tasks/presentation.facet`:

```facet
token color.background: Color = #FFFFFF
token color.surface: Color = #F1F5F9
token color.foreground: Color = #111827
token color.danger: Color = #B91C1C
token space.small: Length = 8dp
token space.medium: Length = 16dp
token radius.panel: Radius = 8dp
token font.body: Font = system(size: 16sp, weight: regular)
token font.heading: Font = system(size: 24sp, weight: semibold)
token motion.feedback: Motion = 120ms

theme Light {}

theme Dark {
  color.background = #111827
  color.surface = #1F2937
  color.foreground = #F9FAFB
  color.danger = #FCA5A5
}

style body {
  font: font.body
  foreground: color.foreground
}

style heading {
  font: font.heading
  foreground: color.foreground
}

style panel {
  background: color.surface
  foreground: color.foreground
  padding: space.medium
  radius: radius.panel
  transition: motion.feedback
}

component Panel(title: Text) {
  slot content

  ui {
    Column(style: panel, gap: space.small) {
      Text(title, style: heading, role: heading)
      slot content
    }
  }
}

component TaskRow(task: Task, open: Action, complete: Action<Bool>) {
  ui {
    Row(gap: space.small, align: center) {
      Toggle(
        value: task.completed,
        label: "Completed: " + task.title.text,
        onChange: complete(event.value)
      )
      Button(task.title.text, action: open, label: "Open " + task.title.text)
    }
  }
}
```

The first `slot content` declares a required slot; its occurrence inside `ui` renders the supplied content. `Action<Bool>` is a typed event contract, not an arbitrary effectful function. Its callback property supplies a boolean `event.value` to the deferred action binding, including the component binding at the call site; the value is unavailable to ordinary render expressions. Controls supply their native role; custom interactive elements would need an explicit role. Themes inherit base token values and override only differences, so `Light` needs no repeated values.

`TaskRow` delegates interactions to screen actions. The `openTask` action below logs inline before navigation, without putting effects in the component's widget tree.

### Screens, state, actions, and navigation

`src/tasks/task-list.facet`:

```facet
screen TaskList {
  route "/tasks"
  state filter: TaskFilter = TaskFilter.All
  state errorMessage: Text? = none
  data tasks = FindTasks(filter: filter)
    reload on enter, change(filter)

  action selectFilter(next: TaskFilter) {
    set filter = next
  }

  action openTask(id: Id<Task>) {
    log debug "task.openRequested"
    navigate TaskDetail(id: id)
  }

  action retry {
    set errorMessage = none
    log info "tasks.retryRequested"
    refresh tasks
  }

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
    Column(padding: space.medium, gap: space.medium) {
      if width >= 600dp {
        Row(gap: space.medium) {
          Text("Tasks", style: heading, role: heading)
          Button("New task", action: navigate(TaskEditor(id: none)))
        }
      } else {
        Column(gap: space.small) {
          Text("Tasks", style: heading, role: heading)
          Button("New task", action: navigate(TaskEditor(id: none)))
        }
      }

      Row(gap: space.small) {
        Button("All", action: selectFilter(TaskFilter.All))
        Button("Active", action: selectFilter(TaskFilter.Active))
        Button("Completed", action: selectFilter(TaskFilter.Completed))
      }

      if let message = errorMessage {
        Text(message, role: alert)
      }

      states tasks {
        loading { Text("Loading tasks...", role: status) }
        error StorageFailed {
          Text("Could not load tasks.", role: alert)
          Button("Retry", action: retry)
        }
        empty { Text("No tasks match this filter.") }
        content {
          List(tasks, key: task.id) { task ->
            TaskRow(
              task: task,
              open: openTask(id: task.id),
              complete: complete(id: task.id, completed: event.value)
            )
          }
        }
      }
    }
  }
}

test "an empty list still offers task creation" {
  given {
    viewport width: 390dp
    appearance Light
    FindTasks(filter: TaskFilter.All) returns []
    GetEditorTask(id: none) returns none
  }
  when open TaskList()
  then visible text("No tasks match this filter.")
  then visible button("New task")
  when tap button("New task")
  then screen TaskEditor(id: none)
}

test "loading failure can be retried" {
  given {
    FindTasks(filter: TaskFilter.All) sequence {
      fails StorageFailed
      returns []
    }
  }
  when open TaskList()
  then visible alert("Could not load tasks.")
  then no logs
  when tap button("Retry")
  then visible text("No tasks match this filter.")
  then logs exactly [
    {
      level: info,
      source: "TaskList.retry",
      event: "tasks.retryRequested",
      error: none
    }
  ]
}

test "opening a task logs the presentation action without its title" {
  given {
    let task = Task(
      id: id<Task>("private"),
      title: TaskTitle(text: "Private task title"),
      completed: false
    )
    FindTasks(filter: TaskFilter.All) returns [task]
    GetTask(id: task.id) returns task
  }
  when open TaskList()
  when tap button("Open Private task title")
  then screen TaskDetail(id: task.id)
  then logs exactly [
    {
      level: debug,
      source: "TaskList.openTask",
      event: "task.openRequested",
      error: none
    }
  ]
}
```

`src/tasks/task-editor.facet`:

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
    log info "task.saved"
    navigate TaskList()

    failure InvalidTitle {
      field draft.title "Enter a title between 1 and 120 characters."
      focus draft.title
    }
    failure TaskNotFound { set errorMessage = "This task no longer exists." }
    failure DuplicateTask { set errorMessage = "Could not create task. Try again." }
    failure IdentityFailed {
      log error "task.identityFailed"
      set errorMessage = "Could not create task. Try again."
    }
    failure StorageFailed {
      log error "task.saveFailed"
      set errorMessage = "Could not save task. Try again."
    }
  }

  ui {
    states task {
      loading { Text("Loading editor...", role: status) }
      error TaskNotFound {
        Text("This task no longer exists.", role: alert)
        Button("Tasks", action: navigate(TaskList()))
      }
      error StorageFailed {
        Text("Could not load task.", role: alert)
        Button("Retry", action: refresh(task))
      }
      content {
        Scroll {
          Panel(title: id == none ? "New task" : "Edit task") {
            content {
              Input(bind: draft.title, label: "Title", required: true)
              if let message = errorMessage {
                Text(message, role: alert)
              }
              Button("Save", action: save, disabled: save.pending)
              Button("Cancel", action: navigate(TaskList()))
            }
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

test "save failure logs its kind without the form value" {
  given {
    GetEditorTask(id: none) returns none
    CreateTask(title: "Private task title") fails StorageFailed
  }
  when open TaskEditor(id: none)
  when fill input("Title") with "Private task title"
  when tap button("Save")
  then screen TaskEditor(id: none)
  then visible alert("Could not save task. Try again.")
  then logs exactly [
    {
      level: error,
      source: "TaskEditor.save",
      event: "task.saveFailed",
      error: "StorageFailed"
    }
  ]
}

test "successful save continues to logging and navigation" {
  given {
    let task = Task(
      id: id<Task>("t1"),
      title: TaskTitle(text: "Read"),
      completed: false
    )
    GetEditorTask(id: none) returns none
    CreateTask(title: "Read") returns task
    FindTasks(filter: TaskFilter.All) returns [task]
  }
  when open TaskEditor(id: none)
  when fill input("Title") with "Read"
  when tap button("Save")
  then screen TaskList()
  then logs exactly [
    {
      level: info,
      source: "TaskEditor.save",
      event: "task.saved",
      error: none
    }
  ]
}
```

`src/tasks/task-detail.facet`:

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
      error TaskNotFound {
        Text("This task no longer exists.", role: alert)
        Button("Tasks", action: navigate(TaskList()))
      }
      error StorageFailed {
        Text("Could not load task.", role: alert)
        Button("Retry", action: refresh(task))
      }
      content {
        Panel(title: task.title.text) {
          content {
            Text(task.completed ? "Completed" : "Active", style: body)
            if let message = errorMessage {
              Text(message, role: alert)
            }
            Button("Edit", action: navigate(TaskEditor(id: id)))
            Button("Delete", action: remove, disabled: remove.pending)
            Button("Tasks", action: navigate(TaskList()))
          }
        }
      }
    }
  }
}

```

`src/host.facet`:

```facet
host {
  start TaskList()
  themes Light, Dark
  appearance system
  background color.background
}
```

Here, `states` exposes the typed query value only in its `content` branch. Successful `none` is content for an optional query, not a loading state; `empty` applies to collections. Form initialization is deferred until its query dependencies first succeed, including `none`, and reruns only through an explicit `reset` or new screen instance. A retry does not silently overwrite an edited form. In `List(tasks, key: task.id) { task -> ... }`, the declared row binding scopes both the key expression and row body; keys must be unique in the list.

`data` infers its value type from the referenced use case, while `reload` keeps execution triggers explicit; `refresh` explicitly requests another query execution. Renders never reload by themselves. Each `try` contributes its use-case failures to the action's trailing handler set, including calls inside branches. After a failed call, ordinary statements such as logging `"task.saved"` or navigating are skipped. Action execution exposes a `pending` boolean; re-entry while pending is rejected. Actions awaiting I/O use the screen lifetime for cancellation, and late responses cannot mutate a disposed screen. Cancellation does not undo a storage operation that already committed.

Action failures update `errorMessage`; `if let` narrows its optional type and renders the alert only when a message exists. Each new attempt clears the previous action error before invoking the use case. Native profiles map `role: alert` to visible accessible feedback and platform-appropriate announcements without automatically moving focus; rerendering unchanged text must not repeatedly announce it. `field` still binds input-validation feedback to its input, and explicit `focus` remains separate. Query-level failures use their existing `error` states. No failure becomes an empty list.

Named screen parameters form typed route arguments. The profile maps them to native navigation or browser query parameters; missing or malformed required route arguments must produce a navigation diagnostic rather than a guessed identity.

The logging flow stays local: `CreateTask` uses `return try Tasks.create(task)` to return the stored task or propagate failure. The editor's `try CreateTask(...)` transfers a storage failure to its trailing handler, which logs `"task.saveFailed"` and updates the message state. The record contains source `TaskEditor.save` and error `StorageFailed`, without the title or error payload. Logging does not handle the failure or undo committed effects.

### Target profiles and composition

`src/targets.facet` contains platform-specific composition. The feature contracts and shared host remain platform-neutral.

```facet
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

`swiftui`, `compose`, `browser.dom`, and the six implementation keys are proposed entries in a versioned built-in profile catalog. They are not native snippets or downloaded plugins. The catalog declares supported UI properties and the `storage.local`/`identity.random` capabilities supplied by each implementation. No platform implementation exists merely because its catalog key appears here.

`logging local` selects the profile's native local sink. Applications using `log` require this explicit target setting; the checker rejects missing sinks. Remote exporters, trace correlation, and metric collection are not part of v0.1.

### One test declaration

Use `test` for both application behavior and screen interactions. Keep `given`, `when`, and `then` inside the block. Do not introduce a separate UI-prefixed declaration, a `kind` field, or a legacy declaration alias.

The checker resolves the first `when` subject from the same declaration table as the rest of the specification:

- `when CreateTask(...)` invokes a typed use case. Its test may stub required ports and assert results, declared failures, and calls. Domain execution emits no application logs.
- `when open TaskList()` activates a typed screen. Its test may stub use-case outcomes and assert UI state, navigation, focus, and logs.

Resolve the subject before validating `given`, even though setup appears first in the source. Reject missing subjects, incompatible stubs, unavailable fields, or assertions that do not belong to the resolved context. Do not infer test context from English descriptions. Once a screen is open, subsequent interaction steps operate on its active navigation state; a test cannot switch into direct domain invocation midway.

This keeps a single declaration form while preserving the different test boundaries. Both forms use `facet run` and the same result envelope. Native harnesses implement the corresponding domain or UI driver; a reference UI-state test does not replace a native rendering test.

Bindings declared in `given` remain visible to the test's invocation and assertions. Unconfigured calls fail the test instead of returning defaults. `logs exactly` compares the in-memory application records, including their complete field sets, so additional payload fields fail the assertion. Sink-specific metadata such as native timestamps is outside that application-record comparison. These cases demonstrate test syntax; the implementation milestone still adds boundary, mutation, cancellation, sink-failure, and real-storage cases from the validation strategy.

Tests opening a screen substitute typed use-case outcomes at the presentation boundary. Tests invoking use cases above substitute ports instead. This separation tests UI state without allowing production screens to bypass use cases.

UI `button(...)` selectors match the accessible name, including an explicit `label` when present. Log assertions compare the cumulative ordered application records for the test. Use-case stubs supply outcomes; the real presentation handler executes its inline statements. Native integration tests must verify the same behavior with actual use cases and log sinks.

### Remaining widget forms

The following isolated catalog fixture covers `Stack` and `Image`, which the task-manager screens do not require. It introduces no external asset or runtime download. `ImageSource` is a catalog type; `source` is a typed fixture parameter rather than a filesystem path.

```facet
component ImagePreview(source: ImageSource, caption: Text) {
  ui {
    Stack(align: bottom) {
      Image(source: source, alt: caption, width: 240dp, height: 160dp)
      Text(caption, style: body)
    }
  }
}
```

### Indentation alternative

The comparison candidate keeps the same names, types, clauses, and semantics, replaces block braces with indentation, and rejects tabs. This excerpt is equivalent to the corresponding braced declarations; it is not the full comparison corpus or a second accepted language.

```facet
value TaskTitle
  text: Text
  invariant text == trim(text) else InvalidTitle
  invariant length(text) >= 1 && length(text) <= 120 else InvalidTitle

usecase CreateTask(title: Text) -> Task
  let name = try TaskTitle(text: trim(title))
  let identity = try TaskIds.next()
  let task = Task(id: identity, title: name, completed: false)
  return try Tasks.create(task)

component Panel(title: Text)
  slot content
  ui
    Column(style: panel, gap: space.small)
      Text(title, style: heading, role: heading)
      slot content

host
  start TaskList()
  themes Light, Dark
  appearance system
  background color.background

target web
  profile browser.dom
  logging local
  bind Tasks = LocalTasks using indexeddb.browser
  bind TaskIds = RandomTaskIds using uuid.browser

test "invalid title causes no identity or storage effects"
  when CreateTask(title: "   ")
  then fails InvalidTitle
  then no calls TaskIds
  then no calls Tasks
  then no logs
```

T2 must translate the entire example, including multiline expressions and nested UI states, before comparing tokens or repair reliability. The shorter excerpt cannot establish a token-efficiency winner.

### Coverage and rejection examples

| Planned construct | Where exercised |
| --- | --- |
| Editions, local files, implementation roots | Project manifest |
| Entities, value objects, invariants, enums, errors, qualified names | Domain and test declarations |
| Text, booleans, typed IDs, lists, optional values, unit | Domain, port signatures, use cases, and editor state |
| Integer expressions | `length` comparisons; integer overflow needs dedicated T2 fixtures |
| Port effects, typed failures, guards, conditions, collection operations | Application declarations |
| Adapter contracts and capabilities | `LocalTasks`, `RandomTaskIds`, and target bindings |
| Full initial widget catalog | Screens, components, and `ImagePreview` fixture |
| Slots and typed component actions | `Panel` and `TaskRow` |
| Tokens, styles, themes, responsive layout | Presentation declarations and list screen |
| Form/error state, optional rendering, pending actions, accessibility | Editor and screen-state handlers |
| Shared entry point, typed navigation, and permitted navigation cycles | Screen actions and `host.start` |
| Native target differences without shared-code leakage | Three target declarations |
| Results, failures, effects, and UI expectations | One `test` declaration with typed use-case or screen subjects |
| Inline structured logs, native local sinks, and record assertions | Presentation actions/handlers, targets, and screen tests |

The table covers the planned construct families, not every lexical production or operator combination. A normative grammar and a complete positive/negative fixture corpus remain T2 deliverables.

Examples the future checker must reject:

```facet
// Entity invariant performs an outward effect.
entity InvalidEntity {
  invariant Tasks.getAll() != []
}

// UI accesses storage without an application use case.
screen InvalidScreen {
  data tasks = Tasks.getAll() reload on enter
}

// The selected adapter implements a different port.
target incompatible {
  profile browser.dom
  logging local
  bind Tasks = RandomTaskIds using uuid.browser
  bind TaskIds = RandomTaskIds using uuid.browser
}

// Unknown UI property and missing accessible input label.
component InvalidInput {
  ui { Input(magicSize: 24dp) }
}

// Rendering must not emit logs.
component InvalidLog {
  ui { log info "rendered" }
}

// Required port binding is absent.
target incomplete {
  profile browser.dom
  logging local
  bind TaskIds = RandomTaskIds using uuid.browser
}
```

These are intentionally invalid fragments, not part of the runnable example project. Document diagnostic categories now; allocate stable codes when the diagnostic catalog is implemented rather than implying codes already exist.

## Planning outcome

The plan is saved in `plan-gpt-6-astra.md` and the session plan file. It preserves all four architecture layers, the full application example, and milestones. Feature-driven `src/` files colocate contracts and tests; native projects live under configurable `platforms/` paths, and conformance fixtures are derived temporarily. Query types and use-case contracts are inferred; one unnamed host owns navigation entry and declarations use project-relative identities. Braces-versus-indentation selection, parser validation, and token measurements remain pending. Existing alternative plans, ADR/RFC status, and application files remain unchanged. No application implementation has started.
