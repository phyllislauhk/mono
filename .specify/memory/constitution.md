<!--
Sync Impact Report
- Version change: 0.0.0 (template placeholders) → 1.0.0
- Modified principles: none (initial adoption)
- Added sections:
  - Core Principles (10 decision-ready principles across distributed systems, software design,
    data engineering, DevOps, and observability)
  - Engineering Standards (cross-cutting constraints)
  - Development Workflow (review, delivery, and compliance gates)
  - Governance
- Removed sections: template placeholder slots ([PRINCIPLE_1_NAME] … [PRINCIPLE_5_NAME],
  [SECTION_2_NAME], [SECTION_3_NAME], [GOVERNANCE_RULES])
- Deferred TODOs: none
-->

# mono Constitution

## Core Principles

### I. Do Not Distribute by Default

**Domain:** Distributed Systems

**Rationale:** Coordination is a cost measured in round trips, not milliseconds. Every additional
process, service, or queue introduces failure modes, deployment coupling, and operational surface
area. Vertical scaling and a single deployable minimize that cost until a concrete constraint
forces distribution.

**Rule:** The starting architecture MUST be a single deployable unit scaled vertically. A new
process, service, message queue, or separate datastore MUST NOT be introduced without a written
justification tied to exactly one of: (1) working-set overflow that cannot be resolved by scaling
the single deployable, (2) genuinely independent compute that cannot share a runtime, (3)
geographic latency requirements that cannot be met from a single region, or (4) organizational
independence requiring separate ownership and release cadence. Coordination overhead MUST be
estimated in round trips and documented alongside the justification.

**How To Apply:**
- Reject PRs that add a new service, worker, or queue without a linked justification document
  naming the qualifying constraint.
- Prefer in-process calls, shared memory, or a single database over network hops.
- When reviewing a diff, count new network boundaries; each one without justification violates
  this principle.
- Decompose only after profiling or capacity evidence shows the single deployable cannot meet
  requirements.

### II. Optimize for Deletion, Not Extension

**Domain:** Software Design

**Rationale:** Code that cannot be removed safely accumulates risk. Small modules with low
coupling reduce the blast radius of change and make rewrites feasible. Speculative abstractions
front-load complexity and ossify wrong assumptions.

**Rule:** Every module MUST be small enough that one engineer can delete and rewrite it in one
working day. Speculative abstractions (interfaces, base classes, plugin systems, generic
frameworks) MUST be rejected unless three or more concrete call sites already exist. Code MUST be
inlined until duplication causes measurable maintenance pain; extraction is permitted only after
the third occurrence of the same pattern. Duplication below three occurrences is preferred over
the wrong abstraction.

**How To Apply:**
- Block PRs introducing abstraction layers with fewer than three proven use cases.
- Measure module size by lines of code plus external dependencies; flag modules exceeding one-day
  rewrite scope for splitting or deletion.
- Prefer copy-paste over premature extraction; require evidence of repeated change before DRY
  refactors.
- In review, ask: "Can one engineer delete this module tomorrow without breaking unrelated
  features?" If no, the diff violates this principle.

### III. Make Dependencies Explicit

**Domain:** Software Design

**Rationale:** Hidden coupling and implicit global state make behavior unpredictable and testing
brittle. A reader must be able to trace every dependency without executing the program or reading
the entire codebase.

**Rule:** Hidden coupling, implicit global state, and import-time side effects MUST NOT be
introduced. Every dependency of a function MUST be visible in its signature or declared at the top
of its file. Singletons MUST NOT be used for injectable dependencies; dependency injection MUST
be used instead. Module-level mutable state MUST NOT be modified during import.

**How To Apply:**
- Reject PRs that access global registries, environment variables, or singleton getters inside
  business logic without passing them as parameters.
- Require constructor or function parameters for databases, HTTP clients, clocks, and
  configuration.
- Flag `import` statements that trigger network calls, file I/O, or configuration loading at
  module load time.
- In review, enumerate each function's dependencies from its signature and file header only; any
  undeclared dependency is a violation.

### IV. Contract at the Boundary, Not in the Middle

**Domain:** Data Engineering

**Rationale:** Shared mutable schemas create silent coupling between producers and consumers.
Semantic translation belongs at the boundary where both contexts are understood, not in shared
internal models that drift independently.

**Rule:** Every producer/consumer boundary (HTTP endpoint, message queue, file format, database
table or view) MUST have an explicit schema with a version identifier. Semantic reconciliation
MUST occur at the boundary and MUST be owned by the side that understands both contexts. Shared
mutable schemas between services or modules MUST NOT be used. Internal domain models MUST NOT leak
across boundaries without an explicit mapping layer.

**How To Apply:**
- Require OpenAPI/Protobuf/JSON Schema/DDL version fields for every external interface in the
  diff.
- Reject PRs that import another service's internal types or ORM models across a network
  boundary.
- Place mapping logic in the adapter layer (controller, consumer handler, ETL step), not in
  shared utility packages used by both sides.
- In review, trace data from boundary ingress to internal model; if the boundary lacks a versioned
  schema or mapping is mid-stack, the diff violates this principle.

### V. Test the Transformation, Not the Plumbing

**Domain:** Software Design

**Rationale:** Unit tests on framework wiring provide false confidence. Tests must prove business
logic correctness and boundary behavior, not that mocks return what mocks return.

**Rule:** Unit tests MUST cover pure transformation logic only. Integration tests MUST cover
producer/consumer boundaries. Code owned by this repository MUST NOT be mocked in tests; external
systems not owned by this repository MAY be mocked or stubbed. A bug fix MUST include a failing
test that reproduces the bug before the fix; a green CI run without such a test for a bug fix is
not acceptable.

**How To Apply:**
- Reject bug-fix PRs that do not add a test failing on the prior commit and passing after the
  fix.
- Move tests that mock internal modules to integration tests using real implementations.
- Classify each new test as unit (pure function, no I/O) or integration (boundary crossing); mixed
  tests must be split.
- In review, identify mocked types; any mock of a class defined in this repo violates this
  principle.

### VI. Emit Structured Events, Derive Everything Else

**Domain:** Observability

**Rationale:** Logs, metrics, and traces are projections of the same underlying fact: something
happened. A single structured event primitive eliminates inconsistent signals and enables
high-cardinality debugging.

**Rule:** All observability output MUST originate from structured events. Unstructured log lines
MUST NOT be added in new code. Every structured event MUST include high-cardinality fields:
`request_id`, and when applicable `user_id`, `tenant_id`, and `feature_flag_state`. Metrics and
traces MUST be derived from events or share the same correlation identifiers. Free-text log
messages without a machine-parseable schema violate this principle.

**How To Apply:**
- Reject PRs adding `console.log`, `print`, or unstructured logger calls in application code.
- Require event schemas (field names, types) for new instrumentation; correlate with
  `request_id` across services.
- Convert existing unstructured logging to structured events when touching a file.
- In review, search the diff for string-only log statements; each one is a violation.

### VII. Recovery Over Prevention

**Domain:** DevOps

**Rationale:** All preventive measures fail eventually. The ability to revert quickly limits blast
radius more reliably than pre-deploy confidence alone.

**Rule:** Every change MUST be revertible in under five minutes without a code change. Risky code
paths MUST be gated behind feature flags. Database migrations MUST follow expand-then-contract:
add new schema compatibly, migrate data, then remove old schema in a later release. Rollback MUST
be tested as part of the deploy procedure, not assumed. A deploy without a documented and tested
rollback path violates this principle.

**How To Apply:**
- Require feature flags for behavior changes affecting more than 1% of traffic or critical paths.
- Split destructive migrations across at least two releases (expand, then contract).
- Document rollback steps in the PR or deploy runbook; execute rollback in staging before
  production deploy.
- In review, ask: "Can we revert this in under five minutes by toggling a flag or rolling back
  the deploy?" If no, block merge.

### VIII. Attention Is Finite

**Domain:** Observability

**Rationale:** Unactionable alerts cause fatigue and hide real incidents. Every signal must earn
the on-call engineer's attention by mapping to user-visible impact and a known response.

**Rule:** Every alert MUST correspond to a user-visible symptom and MUST link to a runbook with
remediation steps. Dashboards MUST be saved queries answering specific operational questions, not
decorative charts. Signals (alerts, dashboards, SLOs) that have not resulted in a useful page or
actionable investigation within 90 days MUST be deleted.

**How To Apply:**
- Reject new alerts without a `runbook_url` or inline remediation steps and a named user-visible
  symptom.
- Audit alert and dashboard inventory quarterly; delete stale signals.
- Name dashboards after the question they answer (e.g., "Which tenants exceed p99 latency?"), not
  after the chart type.
- In review, verify each new alert has symptom + runbook; decorative dashboards violate this
  principle.

### IX. Value Is Realized at the User, Not at Merge

**Domain:** DevOps

**Rationale:** Merged code that is not deployed delivers zero value and accumulates integration
risk. "Done" means users are affected, the change is observable, and it can be reverted.

**Rule:** A pull request is NOT complete until the change is deployed to users, instrumented with
structured events, and monitored with alerts or dashboards tied to the change. The definition of
"shipped" is: deployed, instrumented, and monitored—not merged. PRs that merge without a deploy
plan, instrumentation, and monitoring for the changed behavior violate this principle.

**How To Apply:**
- Require deploy plan, observability additions, and monitoring verification in the PR checklist.
- Block "merge and deploy later" without a tracked follow-up with an owner and deadline.
- Verify post-deploy that new structured events appear and relevant dashboards/alerts are active.
- In review, confirm the PR description states deployment status; merged-but-undeployed work is
  not done.

### X. Commands Are Discoverable; Local Dev Matches CI

**Domain:** DevOps

**Rationale:** Undocumented commands create "works on my machine" gaps and slow onboarding. If a
new contributor cannot enumerate every repeatable action in 30 seconds, the development interface
is broken.

**Rule:** Every repeatable action (build, test, lint, migrate, deploy, seed) MUST be a single
named command listed in one canonical location (e.g., `README.md` or `Makefile` with a `help`
target). Commands MUST run with no hidden required arguments or environment setup not documented
in that list. The command a developer runs locally MUST be identical to the command CI runs. CI-only
shell steps, undocumented Makefile targets, and scripts not listed in the canonical command index
MUST NOT be introduced.

**How To Apply:**
- Maintain a single command index; update it in the same PR that adds or changes any script.
- CI workflow files MUST invoke commands from the index, not inline shell logic.
- Reject PRs adding scripts without listing them in the canonical index.
- In review, diff CI config against local docs; any CI step without a documented local equivalent
  violates this principle.

## Engineering Standards

Cross-cutting constraints that apply to all principles above.

- **Decision-ready review:** A reviewer MUST be able to point at a diff and cite the violated
  principle by name without interpretation. Principles use MUST/MUST NOT language intentionally.
- **Justification on complexity:** Any deviation from these principles requires a written
  exception in the PR linking the principle, the constraint, and the accepted trade-off.
- **First-principles grounding:** New practices MUST trace to one of the five domains (distributed
  systems, software design, data engineering, DevOps, observability) and MUST NOT be adopted by
  analogy or convention alone.

## Development Workflow

### Pull Request Requirements

Every PR MUST:

1. State which principles govern the change and confirm compliance or document exceptions.
2. Include tests per Principle V (unit for transformations, integration for boundaries).
3. Include structured events per Principle VI for new or changed behavior.
4. Confirm rollback path per Principle VII and deploy/monitor plan per Principle IX.
5. Update the canonical command index per Principle X if commands change.

### Review Gates

- **Architecture:** Principles I, II, III, IV for structural changes.
- **Quality:** Principle V for all code changes.
- **Operations:** Principles VI, VII, VIII, IX, X for deployable changes.

Non-compliance is a blocking review finding unless a documented exception is approved.

## Governance

This constitution supersedes ad-hoc conventions, team habits, and undocumented practices. When
conflict arises, this document governs.

**Amendment procedure:**
1. Propose changes via PR modifying `.specify/memory/constitution.md`.
2. Include a Sync Impact Report comment documenting version bump rationale.
3. Require approval from at least one engineering owner.
4. Update dependent templates only when explicitly required; templates read this file at runtime.

**Versioning policy:**
- MAJOR: Backward-incompatible removal or redefinition of a principle.
- MINOR: New principle or materially expanded guidance.
- PATCH: Clarifications, wording, typo fixes without semantic change.

**Compliance review:**
- Reviewers MUST cite principle numbers when requesting changes.
- Quarterly audit: verify command index completeness, alert runbook coverage, and stale signal
  deletion per Principle VIII.

**Version**: 1.0.0 | **Ratified**: 2026-08-12 | **Last Amended**: 2026-08-12
