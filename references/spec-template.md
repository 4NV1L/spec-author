# Spec template

`SPEC.md` uses these sections, in this order. The mapping column names the
`spec-grader` rubric category (v1.0.0) each section serves and its weight — the
structure exists so a good-faith draft cannot structurally fail a readiness
gate. It is not a scoring target.

| Section | Rubric category | Weight |
|---|---|---|
| `## Problem` | `problem_definition` | 7 |
| `## Goals / Non-goals` | `goals` | 5 |
| `## Users and flows` | `user_stories` | 5 |
| `## Functional requirements` | `functional_requirements` | 10 |
| `## Non-functional requirements` | `non_functional_requirements` | 7 |
| `## Architecture` | `architecture` | 6 |
| `## Interfaces and contracts` | `interfaces_contracts` | 7 |
| `## Security and privacy` | `security_privacy` | 7 |
| `## Failure modes` | `failure_modes` | 5 |
| `## Edge cases` | `edge_cases` | 5 |
| `## Acceptance criteria` | `acceptance_criteria` | 9 |
| `## Dependencies and constraints` | `dependencies` | 4 |
| `## Observability` | `observability` | 4 |
| `## Deployment and rollback` | `deployment_rollback` | 4 |
| `## Implementation notes` | `implementation_readiness` | 8 |
| `## Open decisions` | provenance | — |
| `## Assumptions` | provenance | — |
| `## Sources` | provenance | — |

`ambiguity_testability` (weight 7) has no section of its own. It is earned
cross-cutting, through numbered requirements and acceptance criteria that cite
them.

## Numbering

- Functional requirements are numbered `FR-1`, `FR-2`, … and each states one
  testable behaviour.
- Acceptance criteria are numbered `AC-1`, `AC-2`, … and each names the
  requirement it verifies: `**AC-3** (FR-2) …`.

That traceability is what makes the spec implementable and is also what the
downstream implementation plan consumes.

## Empty sections

A section with no genuine content is written as `Not applicable — <reason>`.
Never fill a section with generated prose. Raising a rubric score is not a
reason to add content; a specification that admits a section does not apply is
more useful than one that pads it.

## Provenance sections

```markdown
## Open decisions

| # | Decision | Why it blocks | Owner |
|---|---|---|---|
| D1 | … | … | Author |

## Assumptions

| # | Assumption | Status |
|---|---|---|
| A1 | … | Needs confirmation |

## Sources

- `path/to/file.py:42` — what it establishes
- [Title](https://example.com), accessed 2026-09-06 — what it establishes
```

Every `[A#]` and `[D#]` used in the body resolves to a row here. A row may
exist without an inline marker when it concerns the project rather than a
specific claim.

## Sidecar promotion

Keep everything in `SPEC.md` until it outgrows the document:

- Promote to `SPEC.research.md` at roughly five or more external findings, or
  when any finding needs the full Finding / Source / Design consequence /
  Status treatment.
- Promote to `SPEC.decisions.md` at roughly eight or more combined open
  decisions and assumptions, or when decisions need recorded rationale and
  rejected alternatives.

On promotion, `SPEC.md` keeps a one-line summary of each item plus a link. The
grader reads only `SPEC.md`, and an implementer must never need the sidecars to
proceed.
