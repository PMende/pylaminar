# CLAUDE.md

Guidance for Claude Code when working in this repository. This file is a living
document — update it as architectural decisions are made, subagents are added,
or conventions change. Treat disagreements with this file as a prompt to
discuss and revise it, not to silently deviate.

## Project Overview

This repository implements a Python workflow orchestrator, `pylaminar`: a 
framework for defining, validating, executing, and observing data/ML pipelines
("flows" composed of "tasks").

Core design bets:

- **pydantic** is the schema layer for task/flow parameters, I/O artifact
  contracts, and config SerDe. If it can be a pydantic model, it should be.
- **Metaclasses** back `Task`, `Flow`, and `Artifact` base classes so that
  subclassing is the primary extension mechanism for downstream users
  (data scientists / ML engineers), not composition-heavy boilerplate.
- **pixi** is the project-level task runner and environment manager. It
  drives meta-operations on the *project itself* — scaffolding a new task
  from a template, listing registered flows, running the test suite — as
  distinct from flow/task execution at runtime. `conda-forge` is the
  primary channel for both Python packages and OS-level dependencies;
  PyPI (resolved via pixi's `uv` integration) is a fallback for packages
  unavailable or lagging on `conda-forge`, not a default.
- **Apache Iceberg** is the standard table format for structured data I/O.
  All tabular reads/writes in the artifact layer should target Iceberg
  tables unless there's a documented reason to fall back to raw files.
- **fsspec** abstracts blob storage. AWS (S3) is the first-class target;
  the abstraction should not assume AWS-only, but AWS is what gets built
  and tested first.
- A **lightweight graph library** models the DAG(s) of tasks within a flow
  for validation, topological execution ordering, and static analysis
  (cycle detection, impact analysis, visualization export). Favor a
  minimal, fast dependency over a heavyweight one — see Open Questions.

## Architecture Principles

1. **Schema-first.** Every task's inputs and outputs are pydantic models.
   Flow parameters are pydantic models. Serialization (to/from JSON,
   YAML, or Iceberg-backed config tables) goes through pydantic, not
   ad-hoc dict handling.
2. **Metaclass-driven registration.** `TaskMeta`, `FlowMeta`, and
   `ArtifactMeta` should handle: automatic registration into a project-wide
   registry (so pixi tasks like `list-flows` can discover them), interface
   validation at class-definition time (not first-call time), and
   injection of any boilerplate (logging, tracing hooks) so subclass
   authors write only domain logic.
3. **Separation of orchestration-time and runtime concerns.** pixi tasks
   operate on the *project* (scaffolding, discovery, dev-loop ergonomics).
   The orchestrator engine operates on *flow execution* (DAG resolution,
   scheduling, retries). Don't let these layers blur.
4. **Storage-agnostic by interface, AWS-first by implementation.** Define
   the fsspec-based I/O interfaces generically; implement and test against
   S3 first. Every cloud-specific implementation lives behind the same
   interface so a GCS/Azure backend is additive, not a rewrite.
5. **Fail at definition time, not execution time.** Favor validation in
   metaclasses/`__init_subclass__` and DAG construction over discovering
   problems mid-run. A malformed flow should fail to import cleanly.
6. **Small, composable core; templates for the rest.** The framework
   itself should stay minimal. Task/flow/artifact templates (scaffolded
   via pixi) are how most users interact with extensibility, not by
   reading the core source.

## Proposed Package Layout

This is a starting proposal — confirm or revise with me before treating it
as fixed.

```
.
├── pixi.toml                  # project tasks, env/deps, pixi-native tooling
├── pyproject.toml             # package metadata, build backend
├── src/
│   └── pylaminar/
│       ├── core/
│       │   ├── meta.py        # TaskMeta, FlowMeta, ArtifactMeta
│       │   ├── task.py        # Task base class
│       │   ├── flow.py        # Flow base class
│       │   ├── artifact.py    # Artifact base class
│       │   └── registry.py    # project-wide task/flow/artifact registry
│       ├── graph/
│       │   ├── dag.py         # DAG construction/validation from a Flow
│       │   └── analysis.py    # cycle detection, topo sort, impact analysis
│       ├── io/
│       │   ├── iceberg.py     # Iceberg table read/write layer
│       │   ├── fsspec_backend.py
│       │   └── aws.py         # S3-specific fsspec/credential wiring
│       ├── engine/
│       │   ├── scheduler.py   # DAG execution, retries, concurrency
│       │   └── context.py     # run-scoped execution context
│       ├── cli/                # pixi-invoked project tooling entry points
│       └── templates/          # scaffolding templates for new tasks/flows
├── tests/
├── examples/                   # example flows demonstrating the framework
└── .claude/
    ├── agents/                 # subagent definitions, see below
    └── commands/                # optional slash-commands for common loops
```

## Domain Model Sketch

Discuss and refine before implementing — this is a starting hypothesis.

```python
class TaskMeta(type):
    """Registers subclasses, validates input/output schemas at class
    creation, injects shared instrumentation."""

class Task(metaclass=TaskMeta):
    """Params: pydantic model. run(): domain logic. Upstream dependencies
    are declared at the parameter level: a Params field is bound to a
    FieldRef pointing at a specific field of an upstream Task's declared
    output Artifact, rather than to the whole upstream Task or Artifact."""

class Flow(metaclass=FlowMeta):
    """A named, versioned collection of Tasks. The dependency graph is
    derived by tracing FieldRef bindings across all Tasks' Params at
    flow-construction time — not declared separately, and not inferred by
    artifact type-matching."""

class Artifact(metaclass=ArtifactMeta):
    """A typed, versioned unit of data I/O — backed by Iceberg for tabular
    data, fsspec-addressable paths otherwise. Exposes its declared fields
    as symbolic FieldRefs at flow-definition time (before the producing
    Task has run), and as concrete values at execution time."""

class FieldRef:
    """A definition-time-only symbolic reference to a single field of an
    upstream Task's declared output Artifact — e.g. `TaskA.output.task_date`,
    used when wiring `TaskB.Params(upstream_task_date=TaskA.output.task_date)`.
    Captures (producing_task, artifact_type, field_name, field_type) so
    FlowMeta can validate field-type compatibility against the receiving
    Params field at class/flow-construction time (Principle 5), then
    resolve the reference into a DAG edge and, at execution time, the
    field's concrete runtime value."""

class MappedTask(Task, metaclass=TaskMeta):
    """A Task that fans out into one instance of `inner` per entry of a
    keyed collection (`items`). `items` binds to a FieldRef whose declared
    type is `Mapping[K, T]` — or `Sequence[T]`, accepted as sugar and
    auto-keyed `0..N-1` — so each fan-out instance has a stable identity
    across runs. `.output` is the aggregate Artifact: each field is the
    per-instance output field gathered across all keys, so a downstream
    Task binds to it with an ordinary FieldRef (many:1 / gather — e.g.
    `Summarize.Params(all_counts=DailyLoad.output.row_count)`).

    `.keys` is a second, definition-time-only symbolic handle (distinct
    from `.output`) usable only as another MappedTask's `items=` binding.
    Binding `items` to `SomeMappedTask.keys` makes the new MappedTask a
    *co-map* of `SomeMappedTask`: one `inner` instance per shared key
    (many:many), each parameterized element-wise off the producing
    MappedTask's per-key output rather than the gathered aggregate.
    Whether a FieldRef into a MappedTask's `.output` resolves as the full
    gather or as one key's value is determined structurally — by whether
    the referencing Task is itself co-mapping that same producing
    MappedTask — not by new FieldRef syntax.

    `MappedTaskMeta` validates only what's knowable at definition time:
    element-type compatibility, and, for co-mapping, that `items` is
    bound to literally the same producing MappedTask's `.keys` handle
    (not just a matching key type). Actual cardinality and key values are
    a runtime fact — resolved by the scheduler once the producing task
    materializes `items` — and fan-out is lockstep only for v1: a
    co-mapped MappedTask cannot begin expanding until every instance of
    the MappedTask it co-maps has completed. See Decision Log and Open
    Questions."""
```

**Resolved:** task dependencies are declared at the parameter level via
`FieldRef` bindings, not via whole-task explicit wiring or artifact
type-matching inference. This avoids the ambiguity that whole-artifact
type-matching would hit (multiple tasks emitting the same `Artifact`
subtype) since each reference names one concrete upstream task + field,
while staying terser than whole-task wiring since no separate
`depends_on=[...]` declaration is needed — the graph is traced from the
`FieldRef` bindings themselves. See Decision Log.

**Resolved:** fan-out/mapped task execution — the 1:many, many:1, and
many:many cardinalities (1:1 is the base `FieldRef` case above) — is
handled by a dedicated `MappedTask` subclass of `Task`, not by allowing
a plain `Task` to be invoked multiple times implicitly. `FieldRef`
syntax itself doesn't change between the aggregate (many:1) and
per-instance (many:many) cases; resolution mode is structural, per
`MappedTask`'s docstring above. See Decision Log. The scheduler-side
mechanics of runtime DAG expansion remain open — see Open Questions.

## Development Conventions

- Python 3.13+. *Never* use a Python minor version whose first public
  release is more than 3 years old. Only allow exceptions when core
  dependencies trigger an unsolvable environment when bumping the minimum
  minor version of Python.
- Full type hints everywhere; the codebase should type-check cleanly
  under `mypy`. Chosen over `pyright` for its plugin API — needed to
  teach the checker about the metaclass-driven registration pattern
  (`TaskMeta`/`FlowMeta`/`ArtifactMeta`) and to use pydantic's own mypy
  plugin — and because it's pure Python, fitting the conda-forge-first
  dependency policy (pyright bundles a compiled Node.js binary).
- `ruff` for lint + format.
- Tests via `pytest`. Every core primitive (metaclasses, DAG construction,
  Iceberg I/O, fsspec backends) needs unit tests; example flows should
  have at least one integration test each.
- All pixi tasks (`pixi task list`) should be self-documenting — a short
  description per task, discoverable without reading `pixi.toml`.
- When adding a dependency, check `conda-forge` first (both for Python
  packages and any OS-level/native libraries, e.g. Iceberg/Arrow native
  bits). Only add a package as a PyPI dependency (via pixi's `uv`-backed
  resolution) when it's absent from `conda-forge` or meaningfully
  outdated there — and note the reason in `pixi.toml`/PR description when
  it's not obvious.
- Prefer explicit exceptions with actionable messages over silent
  fallbacks, especially in schema validation and DAG construction.
- `pixi.toml` and `pyproject.toml` are kept as two separate files (see
  Proposed Package Layout) rather than consolidated via `[tool.pixi.*]`
  tables in `pyproject.toml`, to preserve the orchestration-time/
  project-metadata split from Architecture Principle 3. This creates a
  drift risk on fields duplicated across both files' `[project]` tables
  (package name, Python version floor, and `version` if hardcoded).
  Mitigate it by: preferring dynamic versioning (`hatch-vcs` or
  `setuptools-scm`, derived from git tags) so `version` isn't hardcoded
  in either file; and a small consistency-check script (stdlib
  `tomllib`, no new dependency) that asserts `name` and the Python floor
  agree between the two files, wired in as a self-documented pixi task
  (e.g. `pixi run check-consistency`), a pre-commit hook, and a required
  CI check.
- Docstrings on all (including internal) classes/methods. Internal engine
  code can be leaner than code that is end-user facing (i.e., task/flow/
  artifact authors) but should still explain *why*, not restate *what*.
- Internal code should not be considered "hidden" from end-users. If there
  is a convention or tool used internally within the codebase, ensure that
  it could be used by an end-user in the case where they absolutely require
  some type of custom behavior.

## Working With Claude

This project is meant to be developed collaboratively and iteratively.
Default posture:

- Prefer proposing a plan and confirming before large structural changes
  (new core abstractions, dependency additions, directory restructuring).
- Small, incremental changes (bug fixes, test additions, docstrings) can
  proceed without a check-in.
- When a design decision is genuinely open (see Open Questions below),
  surface the tradeoff explicitly rather than picking silently.
- Keep this file in sync with reality. If an architectural decision is
  made in conversation, update the relevant section of CLAUDE.md as part
  of that change, not as an afterthought.
- Maintain a lightweight decision log (see below) so future sessions have
  context without re-litigating settled questions.

### Subagents

Claude Code subagents live under `.claude/agents/` as individual markdown
files with YAML frontmatter (`name`, `description`, optional `tools`).
Use subagents to scope context and reduce cross-contamination between
unrelated concerns. Proposed initial set — create these on request or as
the corresponding work begins:

| Subagent | Scope |
|---|---|
| `schema-designer` | pydantic model design for task/flow/artifact interfaces; SerDe correctness |
| `dag-analyst` | graph library integration, DAG validation, cycle/impact analysis |
| `iceberg-io` | Iceberg table read/write layer, schema evolution, partitioning strategy |
| `cloud-backend` | fsspec + AWS integration, credential handling, S3 I/O |
| `engine-runtime` | `engine/` (scheduler.py, context.py) — DAG execution ordering, retries, concurrency, run-scoped execution context |
| `pixi-tooling` | pixi.toml task definitions, project scaffolding templates, CLI ergonomics, pixi.toml/pyproject.toml consistency checks |
| `test-engineer` | test coverage for core primitives, fixtures, integration test scaffolding for example flows |

Each subagent's markdown file should state its scope narrowly and defer
to this file for shared project context rather than duplicating it. When
a new recurring concern emerges that doesn't fit an existing subagent,
propose a new one rather than overloading an existing scope.

### Decision Log

Append short entries here as material decisions are made, so future
sessions (and future me) have the "why," not just the "what."

- 2026-07-12: Standardized on `mypy` over `pyright` for type-checking.
  Deciding factor is its plugin API, needed to teach the checker about
  the metaclass-driven registration pattern and to use pydantic's own
  mypy plugin; also pure Python, fitting the conda-forge-first policy.
- 2026-07-12: Added the `engine-runtime` subagent, scoped to `engine/`
  (scheduler.py, context.py), closing the one core architectural layer
  that previously had no dedicated subagent.
- 2026-07-12: Kept `pixi.toml` and `pyproject.toml` as separate files
  rather than consolidating pixi config into `pyproject.toml` via
  `[tool.pixi.*]`, to preserve the orchestration-time/project-metadata
  split from Architecture Principle 3. Drift risk on duplicated fields
  (name, Python floor, version) is mitigated by preferring dynamic
  versioning and a scripted consistency check run as a pixi task,
  pre-commit hook, and CI check (see Development Conventions).
- 2026-07-12: Confirmed the "Python 3.13+" floor language as-is; the
  "no more than 3 years old" rule is upkept manually as part of normal
  CLAUDE.md maintenance rather than reworded to be version-agnostic.
- 2026-07-12: Package name settled as `pylaminar` (matches the existing
  repo name).
- 2026-07-12: Graph library settled as `rustworkx` over `networkx`,
  per the "minimal but performant" requirement in Core design bets.
- 2026-07-12: Dependency inference model settled as parameter-level
  `FieldRef` bindings (a `Task.Params` field bound to a specific field of
  an upstream `Task`'s output `Artifact`, e.g.
  `TaskB.Params(upstream_task_date=TaskA.output.task_date)`), rather than
  whole-task explicit wiring or whole-artifact type-matching inference.
  Precedent: Flyte's `Promise` objects and Kubeflow Pipelines v2's DSL use
  the same symbolic-reference-at-definition-time pattern. Rationale:
  avoids the ambiguity whole-artifact type-matching would hit when
  multiple tasks emit the same `Artifact` subtype, without the
  boilerplate of a separate `depends_on=[...]` declaration — the DAG
  edge is traced from the binding itself. See Domain Model Sketch for the
  `FieldRef` shape. Open follow-up: `FieldRef` resolution under
  fan-out/mapped task execution, tracked in Open Questions.
- 2026-07-12: Settled the fan-out/mapped task execution model (closing
  the `FieldRef`-under-fan-out follow-up above), covering all cardinality
  cases beyond the base 1:1 `FieldRef` binding:
  - A dedicated `MappedTask` subclass of `Task` (metaclass-backed like
    every other core primitive, per Principle 2) represents fan-out,
    rather than allowing a plain `Task` to be invoked multiple times
    implicitly.
  - **Naming:** `MappedTask`, not `ContainerTask` (collides with the
    OS/Docker-container reading, which is the wrong first association in
    an AWS/S3-flavored codebase) or `TaskCollection` (reads as a
    collection of Task objects rather than as a Task itself, which is a
    mismatch since it must subclass `Task` and participate in the DAG
    like any other Task). `MappedTask` matches the term used by
    comparable systems (Flyte's `map_task`, AWS Step Functions' `Map`
    state, Prefect's `.map()`).
  - **Fan-out source:** `items` is canonically a keyed `Mapping[K, T]`;
    `Sequence[T]` is accepted as sugar, auto-keyed `0..N-1`. Stable keys
    are required for many:many correspondence (below) to survive
    retries/re-runs without an index-only fan-out silently breaking under
    upstream reordering.
  - **many:1 (gather):** a downstream `Task` binds to a `MappedTask`'s
    `.output` field with an ordinary `FieldRef`; it resolves to the
    aggregate collection across all keys. No new syntax.
  - **many:many (co-map):** a `MappedTask` can bind `items` to another
    `MappedTask`'s `.keys` — a second, definition-time-only symbolic
    handle distinct from `.output` — making it a co-map: one `inner`
    instance per shared key, parameterized element-wise. Whether a
    `FieldRef` into a `MappedTask`'s `.output` resolves as the full
    gather or as one key's value is determined structurally (is the
    referencing Task itself co-mapping that same producing `MappedTask`?
    ), not by new `FieldRef` syntax — kept deliberately symmetric with
    the many:1 case above.
  - **Scheduling:** lockstep only for v1 — a co-mapped `MappedTask`
    cannot begin expanding until every instance of the `MappedTask` it
    co-maps has completed. No per-key pipelining/streaming expansion.
    Chosen over pipelining to avoid key-level dependency tracking in the
    scheduler for v1; revisit only if a concrete workload needs it.
  - **Validation scoping:** `MappedTaskMeta` validates element-type
    compatibility and (for co-mapping) keyspace identity — the same
    producing `MappedTask`, not just a matching key type — at definition
    time. Actual cardinality and key values are irreducibly a runtime
    fact (no orchestrator can know them earlier); the scheduler-side
    mechanics of expanding the runtime DAG once `items` materializes are
    deferred, tracked in Open Questions under Execution engine scope.
  - See Domain Model Sketch for the `MappedTask` shape.

## Open Questions

Track unresolved design decisions here; remove once settled and reflected
in the sections above.

1. **Execution engine scope.** Is a distributed/async scheduler in scope
   for v1, or does v1 target single-process sequential/threaded execution
   with the engine designed to be swapped later? Now also gates the
   `MappedTask` runtime-expansion mechanics: the static, class-level
   graph `FlowMeta` builds at import time can't contain concrete fan-out
   node instances (their count isn't known until a `MappedTask`'s
   `items` materializes at runtime — see Decision Log), so the scheduler
   needs some form of runtime DAG mutation. Open: does this live as a
   second, run-scoped graph that gets nodes appended as each
   `MappedTask` expands, or as an expansion-in-place of the static graph
   (placeholder node → N concrete nodes)? Deferred until engine scope
   itself is settled, since the answer likely depends on whether
   execution is sequential or concurrent.
2. **Versioning strategy for flows and artifacts.** Needed for
   reproducibility guarantees against Iceberg's native table versioning/
   time-travel — decide how much the orchestrator reimplements vs. defers
   to Iceberg.
