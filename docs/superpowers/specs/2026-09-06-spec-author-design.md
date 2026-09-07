# Spec Author — design

Status: draft for review
Date: 2026-09-06
Companion project: `spec-grader` (sibling repo)

This document is written in the format the skill itself produces, including
the `[A#]` / `[D#]` provenance markers it defines. It is deliberately
self-demonstrating.

## Problem

Turning an idea into an implementable specification is the step where most
agent-assisted work goes wrong. The failure is not prose quality — it is
provenance. A model asked to "write a spec" will confidently emit user
research it never did, metrics nobody measured, and codebase claims it never
verified. The resulting document reads as authoritative and is acted on as if
it were, because nothing in it distinguishes an observed fact from a guess.

`spec-grader` already evaluates a finished spec against a 16-category weighted
rubric and reports gaps. It cannot solve this problem, by construction: its v1
invariant is `Use only the input spec as evidence in v1; do not invent product
or repository context` (`spec-grader/SKILL.md:24`), and its `--improve` pass is
a single stateless API call that never sees the repository, the web, or the
author (`spec-grader/scripts/spec_grade.py:97-101`). It can say a requirement
is missing. It cannot go find out what the requirement should be.

Nothing currently occupies the authoring half of that loop: gathering evidence,
classifying it honestly, and returning consequential decisions to the human
instead of inventing them.

## Goals / Non-goals

**Goals**

- Turn an idea, feature request, or implementation objective into a
  rubric-aligned Markdown specification that is ready to implement from.
- Make provenance structural: every risky claim is marked and traceable to its
  origin, or it does not appear.
- Return consequential product, security, and architecture decisions to the
  human as an explicit, actionable queue.
- Work in any agent host that can read files and run a shell — Claude Code,
  Cursor, or another agent — without a bundled runtime.

**Non-goals**

- Not a grader. Evaluation stays in `spec-grader`; this skill never scores its
  own work.
- Not an implementation planner. It stops at the spec; producing a task plan
  is a separate step (`writing-plans` in Claude Code).
- Not a score maximizer. Raising a rubric score is never a reason to add prose.
- Not a research agent for its own sake. Research fires only when a design
  decision depends on a fact that cannot be observed locally.
- No bundled Python engine, provider abstraction, or API keys of its own.

## Users and flows

Primary user: an engineer who has an idea or an assigned objective and wants a
specification they can hand to an implementer (human or agent) without first
auditing it for fabrication.

Three entry flows, distinguished by what the user brings:

1. **Greenfield** — an idea, no code yet. "Design a companion app for CTF
   players." The skill develops problem, users, scope, architecture, contracts,
   risks, and acceptance criteria largely from dialogue and external research.
2. **Codebase-informed** — a change to a system that already exists. "Add
   collaborative notes to the CTF companion." The skill reads the repository
   before proposing anything, and every claim about current behavior cites a
   path.
3. **Expand** — a partially formed plan already on disk. `./IDEA.md`. The
   skill preserves the author's intent and converts informal notes into
   implementable structure, without quietly replacing their decisions.

## Functional requirements

**FR-1 — Mode detection.** The skill classifies the invocation into
`greenfield`, `codebase-informed`, or `expand`, states the classification in
its first response, and proceeds unless the user overrides it. Detection rules:
an argument resolving to an existing file is `expand`; otherwise, prose issued
inside a repository containing code relevant to the request is
`codebase-informed`; otherwise `greenfield`.

**FR-2 — Framing.** Before gathering evidence, the skill restates the objective
and intended outcome in two to three sentences and confirms that framing with
the user. A misread objective invalidates everything downstream, so this
confirmation precedes all research.

**FR-3 — Evidence gathering, scoped by mode.** In `codebase-informed` mode the
skill inspects the repository first: architecture and conventions, relevant
modules, data models and interfaces, tests, deployment model, and the files
likely affected. In `greenfield` mode it performs external and product research
as warranted. In `expand` mode it reads the source document first, then either
of the above as the content warrants.

**FR-4 — Conditional research.** External research is not automatic. It fires
only when a design decision depends on a fact the skill cannot observe locally
— third-party API behavior, protocol or standard requirements, published
security guidance, or platform limits. A scoped internal change performs no
external research.

**FR-5 — Blocking questions only.** Before drafting, the skill asks only
questions whose wrong answer would invalidate the design — typically two to
five, asked one at a time. Every other judgment call becomes a labeled
assumption in the draft rather than a question. Questions are asked when they
arise mid-draft too, rather than guessing forward.

**FR-6 — Claim classification.** Every substantive statement is internally
classified as exactly one of: `author-fact`, `repo-observed`,
`external-source`, `reasoned-proposal`, `assumption`, or `open-decision`.
Classification governs what the statement must carry:

| Class | Must carry |
|---|---|
| `author-fact` | nothing; the user asserted it |
| `repo-observed` | a `path:line` or symbol reference |
| `external-source` | a URL and the date accessed |
| `reasoned-proposal` | must follow from stated facts above it |
| `assumption` | an inline `[A#]` marker and confirmation status |
| `open-decision` | an inline `[D#]` marker and why it blocks |

**FR-7 — Selective inline marking.** Only `assumption` and `open-decision`
claims carry inline markers (`[A3]`, `[D2]`), resolving to tables at the end of
the document. Facts cite themselves naturally through backticked paths and
linked sources; reasoned proposals are the default voice of a specification and
carry no marker. Marking every sentence would render the document unreadable
and is prohibited.

**FR-8 — Rubric-aligned draft.** The draft follows the section template in
"Interfaces and contracts", which maps onto the 16 rubric categories in
`spec-grader/config/rubric.json`. Requirements are numbered `FR-#`; acceptance
criteria are numbered `AC-#` and each cites the requirement it verifies.

**FR-9 — No padding.** A section with no genuine content is written as
`Not applicable — <reason>`, never filled with generated prose. Improving a
rubric score is explicitly not a justification for adding content.

**FR-10 — Output package.** A run always produces `SPEC.md`, containing
`## Open decisions`, `## Assumptions`, and `## Sources` sections. Provenance is
promoted to a sidecar file only when it outgrows the spec:

- `SPEC.research.md` when external findings reach roughly five or more, or any
  finding needs full Finding / Source / Design consequence / Status treatment.
  [A1]
- `SPEC.decisions.md` when open decisions plus assumptions reach roughly eight
  or more, or when decisions need recorded rationale and rejected
  alternatives. [A1]

**FR-11 — Self-containment on promotion.** When content is promoted to a
sidecar, `SPEC.md` retains a one-line summary of each item plus a link. The
grader reads only `SPEC.md`, and an implementer must not need the sidecars to
proceed.

**FR-12 — Grading is opt-in.** A default run ends after the draft is presented.
The skill does not grade its own output unless the user asks, and it ends by
naming the exact command that would.

**FR-13 — Bounded fix cycle.** When the user opts in to grading, the skill
runs at most one revision cycle: grade, apply only fixes that clear a readiness
gate or close a gap using evidence already gathered, regrade once, stop. A
finding requiring a judgment call is never auto-applied; it is recorded as an
open decision. There is no third pass and no score threshold that triggers
further revision.

**FR-14 — Grader discovery.** When grading is requested, the skill locates the
grader in this order: `./tools/spec-grader/bin/spec-grade`, then
`../spec-grader/bin/spec-grade`, then `spec-grader/bin/spec-grade` elsewhere in
the workspace. If none is found it reports that and stops. It never downloads
or installs a grader. This mirrors the convention already established in
`spec-grader/integrations/cursor/spec-grade.md:5`.

## Non-functional requirements

- **Host-agnostic.** The skill is Markdown instructions and reference files
  only. It requires no runtime, no dependencies, and no API key of its own; it
  uses whatever tools the host agent provides. [A3]
- **Model-agnostic.** No model is pinned anywhere in the skill. Model guidance
  is documentation: Opus or an equivalent high-reasoning configuration for
  greenfield architecture and unfamiliar systems, Sonnet for scoped feature
  specs, `opusplan` noted as the natural Claude Code fit because authoring is
  planning.
- **Degradation over failure.** Absent web access, absent repository, or absent
  grader, the skill continues and states plainly which evidence class was
  unavailable, rather than silently substituting speculation.
- **Readability.** The spec body must read as a specification, not as annotated
  output. Provenance machinery is confined to markers and end sections.

## Architecture

The skill is a prompt, and the host agent is the engine. There is no code.

```
  invocation
      |
  [SKILL.md]  mode detection -> framing -> evidence -> questions -> draft
      |            |                                       |
      |            +-- references/modes.md                 +-- references/spec-template.md
      |            +-- references/evidence.md
      |
  SPEC.md  (+ SPEC.decisions.md / SPEC.research.md when earned)
      |
      +-- opt-in only --> shells out to spec-grader/bin/spec-grade
                              |
                          one bounded fix cycle --> STOP
```

Repository layout:

```
spec-author/
  SKILL.md
  README.md
  references/
    modes.md            per-mode evidence procedures
    evidence.md         the six classes, citation rules, research format
    spec-template.md    rubric-aligned skeleton
  integrations/cursor/spec-author.md
```

`spec-author` is a separate repository from `spec-grader`, not a folder inside
it. The two version and ship independently, and the grader remains useful to
people who want evaluation without an authoring workflow. The coupling between
them is one shell invocation and one file path convention.

## Interfaces and contracts

**Invocation.** `spec-author <objective-prose | path-to-idea-file>`, optionally
with a request to grade. Mode is inferred per FR-1; the user may override by
saying so.

**`SPEC.md` section contract**, with the rubric category and weight each
section serves:

| Section | Category | Weight |
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
| `## Open decisions` / `## Assumptions` / `## Sources` | provenance | — |

`ambiguity_testability` (weight 7) has no section of its own. It is earned
cross-cutting, through numbered requirements and acceptance criteria that cite
them.

**Research finding format**, required in `SPEC.research.md`:

```
Finding:            Provider imposes a 10 MB request limit.
Source:             <url>, accessed 2026-09-06
Design consequence: Uploads are rejected or split before submission.
Status:             Confirmed external constraint
```

A finding never becomes a requirement implicitly. The design consequence is
stated separately so the leap from fact to requirement stays visible.

**Grader contract.** The skill consumes `spec-grade`'s existing CLI and its
stable JSON schema (`spec-grader/schemas/report.schema.json`). It adds no
arguments to that tool and depends on category IDs remaining stable, which the
grader already commits to (`spec-grader/references/architecture.md:14`). [A2]

## Security and privacy

The material risk is exfiltration through research. A specification under
authorship frequently contains unreleased product plans, internal architecture,
or confidential constraints, and the research phase is the one point where the
skill talks to the outside world.

- External research queries must be composed from the general technical
  question only. Confidential product details, internal names, proprietary
  constraints, and verbatim spec text are never included in a search query or
  sent to an external service.
- When a research question cannot be asked without disclosing confidential
  context, the skill does not ask it. It records the gap as an open decision
  for the author to resolve privately.
- The skill never reads secrets into the spec. `.env` files, credentials, and
  key material are excluded from repository inspection, and any value that
  appears to be a secret is referenced by name only.
- Security and retention policy is never authored unilaterally. Thresholds,
  data retention periods, authentication models, and access rules are
  `open-decision` items requiring the author's explicit direction. [D2]

## Failure modes

| Failure | Handling |
|---|---|
| Objective misread at framing | FR-2 confirmation gate catches it before research cost is incurred |
| Repository too large to inspect exhaustively | Inspect the modules the change touches; state inspection scope explicitly; unexamined areas become assumptions, not silent claims |
| External source unreachable or paywalled | Record the gap as an open decision; never substitute recalled knowledge presented as a cited fact |
| Grader absent or unconfigured | Report it, emit the spec, print the command; the spec is the deliverable, grading is not |
| Grader findings conflict with author decisions | Author decisions win; the finding is recorded, not applied |
| Model drifts into padding | FR-9 plus the one-cycle cap in FR-13 remove both the mechanism and the incentive |

## Edge cases

- **Idea file that is already a good spec.** Expand mode must not rewrite for
  the sake of rewriting. If the input already satisfies the contract, the skill
  says so and proposes targeted additions only.
- **Request spanning multiple independent subsystems.** The skill flags that
  the scope needs decomposition and proposes the split before authoring, rather
  than producing one unimplementable document.
- **Repository present but irrelevant to the request.** Mode resolves to
  greenfield; the skill states why it is not treating the surrounding code as
  context.
- **User answers a blocking question with "you decide".** The decision becomes
  a `reasoned-proposal` with an explicit `[A#]` assumption recording that it
  was delegated, so it remains visible and reversible.
- **Conflicting evidence between repository and author claim.** Both are
  recorded; the conflict is raised as an open decision rather than silently
  resolved.

## Acceptance criteria

- **AC-1** (FR-1) Given an argument that is an existing file path, the skill
  announces `expand` mode before taking any other action.
- **AC-2** (FR-2) No research or drafting occurs before the objective restatement
  is confirmed.
- **AC-3** (FR-4) A run for a scoped internal change completes with zero
  external research calls.
- **AC-4** (FR-6, FR-7) Every `[A#]` and `[D#]` marker in `SPEC.md` resolves to
  an entry in the corresponding table, and no other claim classes carry inline
  markers.
- **AC-5** (FR-6) Every stated claim about existing code in a
  `codebase-informed` run carries a `path:line` or symbol reference.
- **AC-6** (FR-8) A produced `SPEC.md` contains all 15 content sections in the
  template order, with inapplicable sections marked per FR-9.
- **AC-7** (FR-9) No section contains generated prose whose only justification
  is rubric coverage; inapplicable sections read `Not applicable — <reason>`.
- **AC-8** (FR-12) A default run terminates without invoking `spec-grade`, and
  its final message names the command that would.
- **AC-9** (FR-13) An opted-in run invokes `spec-grade` at most twice and
  performs at most one revision pass.
- **AC-10** (FR-14) With no grader present anywhere in the search order, an
  opted-in run still emits `SPEC.md` and reports the grader as unavailable.
- **AC-11** (Security) No external research query contains verbatim spec text or
  identifiable internal product names.

## Dependencies and constraints

- **`spec-grader`**, optional and only for opted-in grading. Assumed to remain
  reachable at one of the FR-14 search paths, with rubric v1.0.0 category IDs
  stable. [A2]
- **Host agent capabilities:** file read, shell execution, and — for greenfield
  work — web access. Each degrades independently per the NFR above. [A3]
- **No package dependencies.** The skill is Markdown; there is nothing to
  install and nothing to pin.

## Observability

For a prompt-only skill, observability means the run is auditable after the
fact rather than instrumented during it:

- The mode classification is stated out loud (FR-1), so a wrong mode is visible
  immediately rather than inferred from bad output.
- Evidence-gathering scope is stated explicitly, so a reader knows what was and
  was not inspected.
- `[A#]` and `[D#]` tables are the durable audit surface: the count of
  unresolved assumptions is the honest signal of how much of the spec is not
  yet grounded.
- When grading is used, `spec-grade`'s existing JSON reports are retained as
  the record of the before/after pass.

## Deployment and rollback

Distribution follows the grader's convention: copy or link the folder as
`.claude/skills/spec-author` in a project, or the equivalent user skills
directory; Cursor uses `integrations/cursor/spec-author.md`. Rollback is
deleting or unlinking the folder — there is no installed state, no runtime, and
no migration. Version compatibility is documented as a rubric-version note in
the README rather than enforced in code.

## Implementation notes

Build order, smallest useful increment first:

1. `references/evidence.md` — the six classes, citation requirements, marker
   syntax, and research-finding format. This is the skill's core discipline and
   everything else references it.
2. `references/spec-template.md` — the section skeleton with the rubric mapping
   and the `Not applicable` convention.
3. `SKILL.md` — frontmatter (`name`, `description` written for accurate
   triggering), the workflow phases, and a "Required invariants" block mirroring
   `spec-grader/SKILL.md:22-30`.
4. `references/modes.md` — per-mode evidence procedures.
5. `integrations/cursor/spec-author.md` — the Cursor projection, modeled on the
   grader's equivalent.
6. `README.md` — installation, model guidance, relationship to `spec-grader`.

Validation is dogfooding: run the finished skill against a real objective, then
grade the output with `spec-grader` and confirm the acceptance criteria above
hold. This design document is itself a first dogfood of the template.

## Open decisions

| # | Decision | Why it blocks | Owner |
|---|---|---|---|
| D1 | Should `spec-author` be published to GitHub, and if so public or private? | Determines whether the repo is set up with a remote now or stays local | Author |
| D2 | Confirm that security/retention policy is always author-owned, never proposed with a default | Sets how firmly the skill refuses to author policy | Author |
| D3 | Is a non-interactive mode needed (CI or batch authoring, no blocking questions)? | Would add a whole execution path; FR-5 currently assumes a human is present | Author |

## Assumptions

| # | Assumption | Status |
|---|---|---|
| A1 | Sidecar thresholds of ~5 findings and ~8 decisions are reasonable defaults | Needs confirmation; tune after first real use |
| A2 | `spec-grader` stays at a sibling path with stable rubric category IDs | Needs confirmation; grader roadmap commits to stable IDs |
| A3 | Host agents provide file read, shell, and web access; each may be absent | Reasoned proposal, degradation handled per NFR |

## Sources

Repository evidence, all in the sibling `spec-grader` repo at rubric v1.0.0:

- `spec-grader/SKILL.md:22-34` — required invariants and output contract, the
  model for this skill's invariants block
- `spec-grader/scripts/spec_grade.py:97-101` — `--improve` prompt and its
  provenance vocabulary, establishing that improvement is stateless and
  spec-only
- `spec-grader/scripts/spec_grade.py:191` — deterministic local scoring; the
  grader owns arithmetic, this skill never scores
- `spec-grader/config/rubric.json` — the 16 categories and weights the template
  maps onto
- `spec-grader/references/architecture.md:14` — commitment to stable category
  IDs
- `spec-grader/integrations/cursor/spec-grade.md:5` — the grader-discovery
  convention reused in FR-14

No external sources were consulted; this design depends on no third-party
behavior.
