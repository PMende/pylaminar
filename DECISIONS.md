# Decision Log

This is an append-only historical record of material decisions made on this
project — the *why* behind choices, not the current rules themselves. For
current, prescriptive guidance (what Claude should actually do), see
`CLAUDE.md`. See CLAUDE.md's "Working With Claude" section for the rule on
when to append here vs. when to amend CLAUDE.md.

Entries are never rewritten after the fact — if a decision changes, add a
new entry that supersedes the old one rather than editing history.

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
  edge is traced from the binding itself. See Domain Model Sketch (in
  CLAUDE.md) for the `FieldRef` shape. Open follow-up: `FieldRef`
  resolution under fan-out/mapped task execution, tracked in Open
  Questions.
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
  - See Domain Model Sketch (in CLAUDE.md) for the `MappedTask` shape.
- 2026-07-18: Split the Decision Log out of `CLAUDE.md` into this file.
  CLAUDE.md is meant to be a snapshot of current, prescriptive guidance;
  the log is a growing historical record, and conflating the two made
  CLAUDE.md harder to skim. CLAUDE.md retains pointers to this file and
  now states explicit rules for when to append here vs. amend CLAUDE.md
  directly (see its "Working With Claude" section).
