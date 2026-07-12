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
    """Params: pydantic model. run(): domain logic. Declares upstream
    artifact dependencies for DAG construction."""

class Flow(metaclass=FlowMeta):
    """A named, versioned collection of Tasks with an explicit or inferred
    dependency graph."""

class Artifact(metaclass=ArtifactMeta):
    """A typed, versioned unit of data I/O — backed by Iceberg for tabular
    data, fsspec-addressable paths otherwise."""
```

Key open design question: should task dependencies be declared explicitly
(constructor references to upstream tasks) or inferred from artifact
type-matching between producers and consumers? Both are viable; this has
real implications for the graph layer and deserves a dedicated design
discussion before locking in.

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

## Open Questions

Track unresolved design decisions here; remove once settled and reflected
in the sections above.

1. **Graph library choice.** Candidates include `rustworkx` (fast,
   minimal, Rust-backed, no hard networkx dependency) vs. `networkx`
   (ubiquitous, larger surface area, pure Python, slower at scale). Given
   the "minimal but performant" requirement, `rustworkx` is the leading
   candidate — confirm before adding as a dependency.
2. **Dependency inference model.** Explicit task wiring vs. artifact
   type-matching inference (see Domain Model Sketch above).
3. **Execution engine scope.** Is a distributed/async scheduler in scope
   for v1, or does v1 target single-process sequential/threaded execution
   with the engine designed to be swapped later?
4. **Versioning strategy for flows and artifacts.** Needed for
   reproducibility guarantees against Iceberg's native table versioning/
   time-travel — decide how much the orchestrator reimplements vs. defers
   to Iceberg.
5. **Package name.** Placeholder `<pkg_name>` used throughout — needs a
   real name before scaffolding begins in earnest.
