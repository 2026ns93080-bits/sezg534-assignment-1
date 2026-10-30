<!--
Sync Impact Report
Version change: unestablished → 1.0.0
Modified principles: scaffold placeholders → Framework-Independent Core;
  Deterministic Time and Behavior; Layered Architecture; Explicit HTTP Contracts;
  Simplicity and Secret-Free Source
Added sections: System and HTTP Constraints; Quality, Security, and Change Boundaries
Removed sections: none
Follow-up TODO: none.
-->
# Tiered Rate Limiter Microservice Constitution

## Core Principles

### I. Framework-Independent Core
The rate-limiting algorithm MUST reside in a core module that has no dependency on
FastAPI or other HTTP framework code. HTTP concerns MUST NOT leak into algorithm
decisions. This separation keeps the algorithm independently testable and reusable.

### II. Deterministic Time and Behavior
All time used by application behavior MUST come from an injectable clock. Code MUST
NOT read the system clock outside the default clock implementation. The service MUST 
use no randomness, MUST NOT depend on wall-clock time except through the injected 
clock, and MUST NOT make external network or database calls. These constraints make 
behavior repeatable and exactly testable.

### III. Layered Architecture
The application MUST separate the core algorithm, tier configuration, storage
adapter interface, its in-memory implementation, and the HTTP layer. The only
storage implementation in scope is in-memory; adding another storage backend
requires an approved specification change.

### IV. Explicit HTTP Contracts
The HTTP layer MUST use FastAPI. Allowed requests MUST return HTTP 200. Rejected
requests MUST return HTTP 429 and include `Retry-After`, `X-RateLimit-Limit`, and
`X-RateLimit-Remaining` headers. Invalid input MUST return an appropriate 4xx status
with a JSON error body. The service MUST validate input and explicitly handle
unknown tiers and missing client identifiers; neither case may be silently accepted.

### V. Simplicity and Secret-Free Source
Implementations MUST prefer the simplest maintainable design that satisfies this
constitution and the approved specification. Source code MUST NOT expose secrets.
Credentials, API keys, and `.env` files MUST NOT be committed.

## System and HTTP Constraints

The service MUST use Python 3.12 or later. Tier configuration MUST remain separate
from the core algorithm and HTTP layer. Storage access MUST use an adapter interface
with only an in-memory implementation. No external service, network, or database
dependency may be introduced without an approved specification change.

## Quality, Security, and Change Boundaries

The independent harness MUST enforce all of the following acceptance standards:

- Ruff reports zero errors.
- Bandit reports zero medium- or high-severity findings.
- All tests pass.
- Line coverage of `src/` is at least 90%.
- Every acceptance criterion in the specification maps to at least one automated test.

The `tests/` and `.harness/` directories are owned by the human engineer and are outside 
the scope of Spec Kit implementation tasks: generated task lists MUST NOT include tasks 
that create or modify them, although a task MAY reference them for traceability. During 
implementation and repair, the coding agent MUST implement application code in `src/` only. 
During implementation and repair it MUST NOT modify the specification, `tests/`, or `.harness/`; 
delete or weaken tests or assertions; lower quality thresholds; disable lint or security 
checks; or bypass mandatory harness gates. Recorded harness results MUST NOT be edited manually. 
Specification artifacts MAY be changed only through the Spec Kit commands 
(specify, clarify, plan, tasks), and every legitimate specification change MUST be recorded 
with its reason.

## Governance

This constitution governs project decisions and implementation. Every proposed
change to this constitution MUST be reviewed for its effect on existing principles
and quality gates, and MUST be recorded through the Spec Kit artifacts with a reason.
Changes to application requirements MUST be made through the specification rather
than by bypassing this governance.

Constitution versions use semantic versioning: MAJOR for backward-incompatible
governance changes or principle removals/redefinitions, MINOR for new principles or
materially expanded requirements, and PATCH for clarifications and non-semantic
refinements. Compliance MUST be reviewed during implementation and change review.
The independent harness MUST remain the authority for its measurable quality gates.

**Version**: 1.0.0 | **Ratified**: 2026-10-30 | **Last Amended**: 2026-10-30
