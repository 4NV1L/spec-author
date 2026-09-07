# Spec Author Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `spec-author`, a prompt-only agent skill that turns an idea, feature request, or implementation objective into an evidence-backed, rubric-aligned Markdown specification.

**Architecture:** The skill is Markdown instructions only — the host agent is the engine. `SKILL.md` drives a five-phase workflow (classify → frame → gather evidence → ask blocking questions → draft) and delegates detail to three reference files. Grading is opt-in and shells out to the sibling `spec-grader` CLI; it is never reimplemented here.

**Tech Stack:** Markdown. No runtime, no dependencies, no package manifest, no API keys. Verification uses `grep`/shell assertions run inline.

**Spec:** `docs/superpowers/specs/2026-09-06-spec-author-design.md`

## Global Constraints

Copied verbatim from the spec. Every task's requirements implicitly include these.

- **No bundled runtime.** No Python engine, provider abstraction, scripts directory, dependency manifest, or API key of its own. The skill is Markdown instructions and reference files only.
- **No model pinned anywhere in the skill.** Model guidance is documentation only: Opus or an equivalent high-reasoning configuration for greenfield architecture and unfamiliar systems, Sonnet for scoped feature specs, `opusplan` noted as the natural Claude Code fit.
- **The six claim classes, spelled exactly:** `author-fact`, `repo-observed`, `external-source`, `reasoned-proposal`, `assumption`, `open-decision`.
- **Only `assumption` and `open-decision` carry inline markers**, written `[A3]` and `[D2]`. Marking every sentence is prohibited.
- **Inapplicable sections read** `Not applicable — <reason>` — never generated prose. Note the em dash.
- **Grader discovery order:** `./tools/spec-grader/bin/spec-grade`, then `../spec-grader/bin/spec-grade`, then `spec-grader/bin/spec-grade` elsewhere in the workspace. Never download or install a grader.
- **Bounded fix cycle:** when grading is opted into, `spec-grade` is invoked at most twice and at most one revision pass is performed. No third pass, no score threshold that triggers further revision.
- **Rubric v1.0.0**, 16 categories, mapped by the 15 content sections plus `ambiguity_testability` earned cross-cutting.
- **Sidecar thresholds:** `SPEC.research.md` at roughly 5+ external findings; `SPEC.decisions.md` at roughly 8+ combined open decisions and assumptions.
- **Out of scope (deferred, open decision D3):** a non-interactive / CI mode. FR-5 assumes a human is present to answer blocking questions.

**Working directory for every task:** `/Users/mavrick/workbench/spec-author`

---

## File Structure

| File | Responsibility |
|---|---|
| `references/evidence.md` | The six claim classes, citation requirements, marker syntax, research-finding format. The skill's core discipline; everything else references it. |
| `references/spec-template.md` | The `SPEC.md` section skeleton, rubric mapping, requirement-numbering and `Not applicable` conventions. |
| `SKILL.md` | Frontmatter, the five-phase workflow, opt-in grading procedure, required invariants. The entry point. |
| `references/modes.md` | Per-mode evidence-gathering procedures for greenfield, codebase-informed, and expand. |
| `integrations/cursor/spec-author.md` | Cursor projection of the workflow. |
| `README.md` | Installation, model guidance, relationship to `spec-grader`. |

Order matters: `evidence.md` and `spec-template.md` define the vocabulary and structure that `SKILL.md` references, so they land first.

---

### Task 1: Evidence and provenance reference

**Files:**
- Create: `references/evidence.md`

**Interfaces:**
- Consumes: nothing (first task).
- Produces: the six class names exactly as listed in Global Constraints; the marker syntax `[A#]` / `[D#]`; the four research-finding field labels `Finding:`, `Source:`, `Design consequence:`, `Status:`; and the section headings `## The six classes`, `## Marking rule`, `## Citation requirements`, `## Research findings`, `## Never claim`. Tasks 2–5 reference this file by path and reuse these exact terms.

- [ ] **Step 1: Write the failing check**

Save this as a shell one-liner you will re-run after writing the file. It asserts all six class names appear in backticks, both marker forms are documented, and all four research labels exist.

```bash
check_evidence() {
  local f=references/evidence.md fail=0
  for c in author-fact repo-observed external-source reasoned-proposal assumption open-decision; do
    grep -q "\`$c\`" "$f" 2>/dev/null && echo "OK   class $c" || { echo "FAIL class $c"; fail=1; }
  done
  for m in '\[A#\]' '\[D#\]'; do
    grep -q "$m" "$f" 2>/dev/null && echo "OK   marker $m" || { echo "FAIL marker $m"; fail=1; }
  done
  for l in 'Finding:' 'Source:' 'Design consequence:' 'Status:'; do
    grep -q "$l" "$f" 2>/dev/null && echo "OK   label $l" || { echo "FAIL label $l"; fail=1; }
  done
  return $fail
}
check_evidence
```

- [ ] **Step 2: Run it to verify it fails**

Run: `check_evidence; echo "exit=$?"`
Expected: twelve `FAIL` lines and `exit=1` — the file does not exist yet.

- [ ] **Step 3: Write the file**

Create `references/evidence.md` with exactly this content:

````markdown
# Evidence and provenance

Every substantive statement in a specification belongs to exactly one class.
The class decides what the statement must carry before it may appear in the
document. This is the skill's core discipline: a specification is trustworthy
because its claims are traceable, not because it reads confidently.

## The six classes

| Class | Meaning | Must carry |
|---|---|---|
| `author-fact` | The user asserted it | Nothing; they are the source |
| `repo-observed` | Read in this repository | A `path:line` or symbol reference |
| `external-source` | Found outside the repository | A URL and the date accessed |
| `reasoned-proposal` | Design that follows from the facts above | Must be derivable from stated facts |
| `assumption` | A guess that could be wrong | Inline `[A#]` and a row in `## Assumptions` |
| `open-decision` | Not yours to make | Inline `[D#]` and a row in `## Open decisions` |

Classify silently. The classes are a discipline for the author, not vocabulary
for the reader — do not print class names into the specification.

## Marking rule

Only `assumption` and `open-decision` carry inline markers. Facts cite
themselves through backticked paths and linked sources. A `reasoned-proposal`
is the default voice of a specification and carries no marker.

```
Retention is 30 days [A3].
Alert thresholds are unset [D2].
Grades are computed locally in `scripts/spec_grade.py:191`.
The provider caps requests at 10 MB ([docs](https://example.com), accessed 2026-09-06).
```

Marking every sentence is prohibited: it makes the document unreadable and
buries the claims that genuinely need scrutiny. Numbering is sequential in
order of first appearance, and every marker resolves to exactly one table row.

## Citation requirements

- **`repo-observed`:** cite `path:line` for a specific statement, or
  `path` plus a symbol name for a general one. If you have not opened the
  file, you have not observed it — either read it or reclassify the claim as
  an `assumption`.
- **`external-source`:** cite the URL and the date you accessed it. Recalled
  knowledge is not an external source. If you cannot reach the source, record
  the gap as an `open-decision` rather than citing from memory.
- **`reasoned-proposal`:** the facts it follows from must already appear in the
  document. A proposal resting on an unstated premise is an `assumption`.

## Research findings

External research is recorded in this shape, so the leap from fact to
requirement stays visible:

```
Finding:            Provider imposes a 10 MB request limit.
Source:             https://example.com/docs, accessed 2026-09-06
Design consequence: Uploads are rejected or split before submission.
Status:             Confirmed external constraint
```

A finding never becomes a requirement implicitly. `Design consequence` is
written as a separate line precisely so a reader can disagree with the leap
while accepting the fact.

## Delegation and conflict

- **"You decide."** When the user delegates a blocking decision, record the
  outcome as a `reasoned-proposal` plus an `[A#]` assumption noting it was
  delegated. It stays visible and reversible rather than hardening into a
  fact.
- **Conflicting evidence.** When the repository contradicts an author claim,
  record both and raise an `open-decision`. Never resolve the conflict
  silently in favour of either one.

## Never claim

- Invented user research, business metrics, or adoption numbers.
- Security, retention, authentication, or access policy chosen without the
  author's direction — always `open-decision`.
- An assumption restated as an existing requirement.
- Codebase behaviour you have not read. Cite `path:line` or do not say it.
- A dependency justified by popularity rather than by the stated constraints.
````

- [ ] **Step 4: Run the check to verify it passes**

Run: `check_evidence; echo "exit=$?"`
Expected: twelve `OK` lines and `exit=0`.

- [ ] **Step 5: Verify the marking rule is stated as prohibition, not preference**

Run: `grep -n "prohibited" references/evidence.md`
Expected: one match, in the "Marking rule" section.

- [ ] **Step 6: Commit**

```bash
git add references/evidence.md
git commit -m "Add evidence reference: six claim classes and citation rules"
```

---

### Task 2: Spec template reference

**Files:**
- Create: `references/spec-template.md`

**Interfaces:**
- Consumes: `references/evidence.md` — the marker syntax `[A#]` / `[D#]` and the `## Open decisions` / `## Assumptions` / `## Sources` section names.
- Produces: the canonical ordered list of 15 content section headings plus 3 provenance section headings; the requirement-numbering scheme `FR-#` and `AC-#`; and the exact string `Not applicable — <reason>`. Tasks 3–5 reference these.

- [ ] **Step 1: Write the failing check**

Asserts all 18 section headings appear in the correct order, and that the numbering conventions and `Not applicable` string are documented.

```bash
check_template() {
  local f=references/spec-template.md fail=0
  local want="Problem|Goals / Non-goals|Users and flows|Functional requirements|Non-functional requirements|Architecture|Interfaces and contracts|Security and privacy|Failure modes|Edge cases|Acceptance criteria|Dependencies and constraints|Observability|Deployment and rollback|Implementation notes|Open decisions|Assumptions|Sources"
  local got
  got=$(grep -oE '^\| `## [^`]+`' "$f" 2>/dev/null | sed 's/^| `## //; s/`$//' | paste -sd'|' -)
  [ "$got" = "$want" ] && echo "OK   18 sections in order" || { echo "FAIL sections"; echo "  want: $want"; echo "  got:  $got"; fail=1; }
  for s in 'FR-' 'AC-' 'Not applicable — '; do
    grep -q "$s" "$f" 2>/dev/null && echo "OK   convention $s" || { echo "FAIL convention $s"; fail=1; }
  done
  return $fail
}
check_template
```

- [ ] **Step 2: Run it to verify it fails**

Run: `check_template; echo "exit=$?"`
Expected: `FAIL sections` with an empty `got:` line, three `FAIL convention` lines, and `exit=1`.

- [ ] **Step 3: Write the file**

Create `references/spec-template.md` with exactly this content:

````markdown
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
````

- [ ] **Step 4: Run the check to verify it passes**

Run: `check_template; echo "exit=$?"`
Expected: `OK   18 sections in order`, three `OK convention` lines, `exit=0`.

- [ ] **Step 5: Commit**

```bash
git add references/spec-template.md
git commit -m "Add spec template: section contract and rubric mapping"
```

---

### Task 3: SKILL.md

**Files:**
- Create: `SKILL.md`

**Interfaces:**
- Consumes: `references/evidence.md` (six classes, markers) and `references/spec-template.md` (section contract, numbering, sidecar rules), both by relative path.
- Produces: the skill's `name: spec-author` frontmatter and description; the five phase names `Classify`, `Frame`, `Gather evidence`, `Ask blocking questions`, `Draft`; the three mode names `greenfield`, `codebase-informed`, `expand`; and the `## Required invariants` block. Tasks 4–5 reference the phase and mode names.

- [ ] **Step 1: Write the failing check**

Asserts valid frontmatter, all three modes, all five phases, the grader search order, the one-cycle cap, and that no model id is pinned.

```bash
check_skill() {
  local f=SKILL.md fail=0
  head -1 "$f" 2>/dev/null | grep -q '^---$' && echo "OK   frontmatter opens" || { echo "FAIL frontmatter"; fail=1; }
  grep -q '^name: spec-author$' "$f" 2>/dev/null && echo "OK   name" || { echo "FAIL name"; fail=1; }
  grep -q '^description: ' "$f" 2>/dev/null && echo "OK   description" || { echo "FAIL description"; fail=1; }
  for m in greenfield codebase-informed expand; do
    grep -q "$m" "$f" 2>/dev/null && echo "OK   mode $m" || { echo "FAIL mode $m"; fail=1; }
  done
  for p in 'Classify' 'Frame' 'Gather evidence' 'Ask blocking questions' 'Draft'; do
    grep -q "$p" "$f" 2>/dev/null && echo "OK   phase $p" || { echo "FAIL phase $p"; fail=1; }
  done
  grep -q 'tools/spec-grader/bin/spec-grade' "$f" 2>/dev/null && echo "OK   grader path 1" || { echo "FAIL grader path 1"; fail=1; }
  grep -q '\.\./spec-grader/bin/spec-grade' "$f" 2>/dev/null && echo "OK   grader path 2" || { echo "FAIL grader path 2"; fail=1; }
  grep -q 'Required invariants' "$f" 2>/dev/null && echo "OK   invariants" || { echo "FAIL invariants"; fail=1; }
  grep -qE 'claude-(opus|sonnet|haiku)-[0-9]' "$f" 2>/dev/null && { echo "FAIL model pinned"; fail=1; } || echo "OK   no model pinned"
  return $fail
}
check_skill
```

- [ ] **Step 2: Run it to verify it fails**

Run: `check_skill; echo "exit=$?"`
Expected: `FAIL` for frontmatter, name, description, all three modes, all five phases, both grader paths and invariants; `OK   no model pinned` (vacuously true on a missing file); `exit=1`.

- [ ] **Step 3: Write the file**

Create `SKILL.md` with exactly this content:

````markdown
---
name: spec-author
description: Turn an idea, feature request, or implementation objective into an evidence-backed Markdown specification, separating observed facts from proposals, assumptions, and decisions the author must make. Use when starting a new project or feature and a specification is needed before implementation; it drafts and researches but never grades its own output.
---

# Spec Author

Turn an idea, feature request, or implementation objective into a
specification someone can implement from without first auditing it for
fabrication.

The discipline is provenance. Read [references/evidence.md](references/evidence.md)
before drafting: every substantive claim is one of six classes, and the class
decides what the claim must carry. Read
[references/spec-template.md](references/spec-template.md) for the section
contract and numbering. Read [references/modes.md](references/modes.md) for
the per-mode evidence procedure.

## Workflow

1. **Classify** the invocation as `greenfield`, `codebase-informed`, or
   `expand`, and say which out loud. An argument that resolves to an existing
   file is `expand`; prose issued in a repository holding code relevant to the
   request is `codebase-informed`; otherwise `greenfield`. The user may
   override.
2. **Frame** the objective and intended outcome in two or three sentences and
   confirm that framing before spending any research effort. A misread
   objective invalidates everything downstream.
3. **Gather evidence** per the mode. Inspect the repository before proposing
   anything that touches it. Research externally only when a design decision
   depends on a fact you cannot observe locally — third-party API behaviour,
   protocol or standard requirements, published security guidance, platform
   limits. A scoped internal change needs no external research. State what you
   inspected and what you did not.
4. **Ask blocking questions** — only those whose wrong answer would invalidate
   the design, typically two to five, one at a time. Everything else becomes a
   labelled assumption the user can correct in review. When a blocking
   decision surfaces mid-draft, stop and ask rather than guessing forward.
5. **Draft** `SPEC.md` against the section contract, then present it with its
   assumptions and open decisions. A default run ends here.

## Grading is opt-in

Do not grade your own draft unless the user asks. End a default run by naming
the command that would:

```sh
<grader>/bin/spec-grade ./SPEC.md --provider anthropic --model <model>
```

When the user does opt in, locate the grader in this order:

1. `./tools/spec-grader/bin/spec-grade`
2. `../spec-grader/bin/spec-grade`
3. `spec-grader/bin/spec-grade` elsewhere in the workspace

If none exists, say so and stop. Never download or install a grader.

Then run exactly one revision cycle: grade, apply only fixes that clear a
readiness gate or close a gap using evidence already gathered, regrade once,
stop. A finding that requires a judgment call is never auto-applied — record it
as an open decision. There is no third pass and no score that triggers further
revision. Report the score, the residual gaps, and the open decisions, and lead
with the gaps rather than the number.

## Required invariants

- Classify every substantive statement, and carry what the class requires.
- Mark only assumptions `[A#]` and open decisions `[D#]` inline; never annotate
  every sentence.
- Never claim codebase behaviour without having read the file. Cite `path:line`
  or do not say it.
- Never invent user research, business metrics, or adoption numbers.
- Never author security, retention, authentication, or access policy without
  the author's direction; it is always an open decision.
- Never present an assumption as an existing requirement.
- Never add a dependency for popularity; justify it against stated constraints.
- Never redesign parts of the system the objective does not touch.
- Never pad a section to improve a score. An inapplicable section reads
  `Not applicable — <reason>`.
- Never put verbatim spec text, internal product names, or confidential
  constraints into an external search query. If a question cannot be asked
  without disclosing them, do not ask it — record an open decision instead.
- Never read secrets into the spec. Exclude `.env` files, credentials, and key
  material from inspection; reference such values by name only.
- Never grade your own draft unprompted, and never score it yourself.

## Degradation

Missing web access, no repository, or no grader is not a failure. Continue, and
state plainly which evidence class was unavailable, rather than substituting
speculation for it.

## Model guidance

The skill pins no model. Authoring is planning, so give it a strong reasoning
configuration when the work is architectural: Opus or equivalent high reasoning
for greenfield design and unfamiliar systems, Sonnet for scoped feature specs.
In Claude Code, `opusplan` fits naturally.
````

- [ ] **Step 4: Run the check to verify it passes**

Run: `check_skill; echo "exit=$?"`
Expected: all `OK` lines including `OK   no model pinned`, `exit=0`.

- [ ] **Step 5: Verify the referenced files resolve**

```bash
for l in $(grep -oE '\(references/[a-z-]+\.md\)' SKILL.md | tr -d '()'); do
  [ -f "$l" ] && echo "OK   $l" || echo "PENDING $l (created in Task 4)"
done
```
Expected: `OK` for `references/evidence.md` and `references/spec-template.md`; `PENDING` for `references/modes.md`, which Task 4 creates.

- [ ] **Step 6: Commit**

```bash
git add SKILL.md
git commit -m "Add SKILL.md: workflow, opt-in grading, required invariants"
```

---

### Task 4: Modes reference

**Files:**
- Create: `references/modes.md`

**Interfaces:**
- Consumes: the three mode names and five phase names from `SKILL.md`; the class names from `references/evidence.md`.
- Produces: per-mode procedures under the headings `## Greenfield`, `## Codebase-informed`, `## Expand`, plus `## Scope too large`. Task 5 refers to modes but not to these headings.

- [ ] **Step 1: Write the failing check**

Asserts the three mode headings, the decomposition guidance, and that codebase-informed mode demands citations.

```bash
check_modes() {
  local f=references/modes.md fail=0
  for h in '## Greenfield' '## Codebase-informed' '## Expand' '## Scope too large'; do
    grep -q "^$h$" "$f" 2>/dev/null && echo "OK   heading $h" || { echo "FAIL heading $h"; fail=1; }
  done
  grep -q 'path:line' "$f" 2>/dev/null && echo "OK   citation rule" || { echo "FAIL citation rule"; fail=1; }
  return $fail
}
check_modes
```

- [ ] **Step 2: Run it to verify it fails**

Run: `check_modes; echo "exit=$?"`
Expected: four `FAIL heading` lines, one `FAIL citation rule`, `exit=1`.

- [ ] **Step 3: Write the file**

Create `references/modes.md` with exactly this content:

````markdown
# Modes

Mode changes only how evidence is gathered in phase 3. The rest of the
workflow is identical.

## Greenfield

No implementation exists. Most evidence comes from the author and from
external research.

1. Establish the problem, the user, and what success looks like before any
   solution talk.
2. Research externally only what the design depends on: platform limits,
   protocol or standard requirements, published security guidance, the
   behaviour of third-party services you intend to rely on. Record each in the
   finding format.
3. Propose an architecture in terms of units with clear boundaries. Prefer the
   smallest structure that satisfies the requirements.
4. Expect more assumptions than in other modes. That is honest, not sloppy —
   label them.

## Codebase-informed

The change lands in a system that already exists. Inspect before proposing.

1. Read before writing anything: the architecture and conventions in use, the
   modules the change touches, data models and interfaces, existing tests, the
   deployment model, and the constraints the current code imposes.
2. Name the files and components the change will likely affect.
3. Every statement about current behaviour carries a `path:line` or symbol
   reference. If you have not opened the file, you have not observed it.
4. Follow the conventions already in the repository rather than importing your
   own. Where existing code genuinely obstructs the objective, propose a
   targeted improvement; do not propose unrelated refactoring.
5. State your inspection scope. Areas you did not examine become assumptions,
   never silent claims.

## Expand

A partially formed plan already exists on disk.

1. Read the source document fully before proposing changes to it.
2. Preserve the author's intent and their existing decisions. Converting
   informal notes into structure is the job; replacing their choices is not.
3. If the input already satisfies the section contract, say so and propose
   targeted additions only. Do not rewrite for the sake of rewriting.
4. Where a note is ambiguous, it becomes a blocking question or a labelled
   assumption — never a silent interpretation.
5. Gather repository or external evidence afterwards, as the content warrants.

## Scope too large

When the objective spans multiple independent subsystems, stop before
drafting. Say so, propose a decomposition into sub-projects, name their
relationships and a build order, and author the first one. One unimplementable
document helps nobody.
````

- [ ] **Step 4: Run the check to verify it passes**

Run: `check_modes; echo "exit=$?"`
Expected: five `OK` lines, `exit=0`.

- [ ] **Step 5: Re-run the Task 3 link check**

Run the link loop from Task 3 Step 5 again.
Expected: `OK` for all three reference files; no `PENDING`.

- [ ] **Step 6: Commit**

```bash
git add references/modes.md
git commit -m "Add modes reference: per-mode evidence procedures"
```

---

### Task 5: Cursor integration and README

**Files:**
- Create: `integrations/cursor/spec-author.md`
- Create: `README.md`

**Interfaces:**
- Consumes: mode names, phase names, invariants, and the grader search order from `SKILL.md`; the section contract from `references/spec-template.md`.
- Produces: the outward-facing surfaces. Nothing consumes these.

- [ ] **Step 1: Write the failing check**

Asserts both files exist, the Cursor file states the opt-in grading rule and the no-install rule, and the README documents installation, model guidance, and the grader relationship without pinning a model.

```bash
check_dist() {
  local c=integrations/cursor/spec-author.md r=README.md fail=0
  for f in "$c" "$r"; do
    [ -f "$f" ] && echo "OK   exists $f" || { echo "FAIL exists $f"; fail=1; }
  done
  grep -qi 'do not download or install' "$c" 2>/dev/null && echo "OK   cursor no-install" || { echo "FAIL cursor no-install"; fail=1; }
  grep -qi 'only when the user asks\|opt-in\|does not grade' "$c" 2>/dev/null && echo "OK   cursor opt-in" || { echo "FAIL cursor opt-in"; fail=1; }
  grep -q '\.claude/skills/spec-author' "$r" 2>/dev/null && echo "OK   readme install" || { echo "FAIL readme install"; fail=1; }
  grep -qi 'spec-grader' "$r" 2>/dev/null && echo "OK   readme grader link" || { echo "FAIL readme grader link"; fail=1; }
  grep -qE 'claude-(opus|sonnet|haiku)-[0-9]' "$r" 2>/dev/null && { echo "FAIL model pinned in readme"; fail=1; } || echo "OK   no model pinned"
  return $fail
}
check_dist
```

- [ ] **Step 2: Run it to verify it fails**

Run: `check_dist; echo "exit=$?"`
Expected: two `FAIL exists` lines and four further `FAIL` lines, `exit=1`.

- [ ] **Step 3: Write the Cursor integration**

Create `integrations/cursor/spec-author.md` with exactly this content:

````markdown
# Author a specification

Turn the user's idea, feature request, or implementation objective into an
evidence-backed Markdown specification. Ask for the objective if it is not
clear from the request or current context.

Classify the request as `greenfield` (no implementation exists),
`codebase-informed` (it changes code in this workspace), or `expand` (the user
pointed at an existing idea file), and say which. Confirm a two-sentence
restatement of the objective before doing research.

In `codebase-informed` mode, inspect the relevant modules, interfaces, tests,
and conventions before proposing anything, and cite `path:line` for every
statement about current behaviour. Research externally only when the design
depends on a fact you cannot observe locally.

Ask only the questions whose wrong answer would invalidate the design —
typically two to five. Everything else becomes a labelled assumption.

Write `SPEC.md` with these sections in order: Problem; Goals / Non-goals; Users
and flows; Functional requirements; Non-functional requirements; Architecture;
Interfaces and contracts; Security and privacy; Failure modes; Edge cases;
Acceptance criteria; Dependencies and constraints; Observability; Deployment
and rollback; Implementation notes; Open decisions; Assumptions; Sources.
Number requirements `FR-1`, `FR-2`, … and acceptance criteria `AC-1`, `AC-2`, …
with each criterion naming the requirement it verifies. Mark assumptions `[A#]`
and open decisions `[D#]` inline and resolve every marker to a table row. Write
an inapplicable section as `Not applicable — <reason>`; never pad one.

Do not grade the draft. Grading happens only when the user asks. If they do,
use the bundled grader at `./tools/spec-grader/bin/spec-grade`, or
`spec-grader/bin/spec-grade` in the workspace; do not download or install a
grader. Run at most one revision cycle: grade, apply only evidence-backed and
gate-clearing fixes, regrade once, stop. Record anything needing a judgment
call as an open decision instead of applying it.

Never invent user research or metrics, never author security or retention
policy without the user's direction, never claim codebase behaviour you have
not read, and never put confidential product details into an external search
query.
````

- [ ] **Step 4: Write the README**

Create `README.md` with exactly this content:

````markdown
# spec-author

Turn an idea, feature request, or implementation objective into an
evidence-backed Markdown specification.

The problem it solves is provenance, not prose. A model asked to "write a spec"
will emit user research it never did, metrics nobody measured, and codebase
claims it never verified — and the result reads authoritative enough to be
acted on. `spec-author` makes the distinction structural: observed facts carry
citations, guesses are marked `[A#]`, and decisions that are the author's to
make are marked `[D#]` and handed back rather than invented.

## What it produces

Always `SPEC.md`, containing the 15 content sections plus `Open decisions`,
`Assumptions`, and `Sources`. Provenance is promoted to `SPEC.research.md` or
`SPEC.decisions.md` only when it outgrows the spec; `SPEC.md` then keeps a
one-line summary and a link, so it stays implementable on its own.

## Modes

| Invocation | Mode |
|---|---|
| an existing file path | `expand` — restructure notes, preserve intent |
| prose, in a repo with relevant code | `codebase-informed` — inspect, then design |
| prose, nothing relevant exists | `greenfield` |

Detected automatically, stated out loud, overridable.

## Installation

Copy or link this folder as `.claude/skills/spec-author` in a project, or into
your user skills directory. For Cursor, use
`integrations/cursor/spec-author.md`. There is nothing to install and no
runtime: the skill is Markdown, and the host agent is the engine. Removing the
folder is a complete uninstall.

## Relationship to spec-grader

`spec-grader` evaluates a finished spec against a weighted rubric. It cannot
gather evidence by design — it sees only the spec text. `spec-author` is the
other half: it gathers the evidence, then hands the result over for
independent critique.

The two stay separate on purpose. A tool that writes and grades its own work
tends to justify its own choices.

Grading is opt-in. A normal run ends with the spec and the command you would
use to grade it. When you do opt in, the skill runs exactly one revision
cycle — grade, apply only evidence-backed fixes, regrade once, stop — so it
cannot drift into padding sections to raise a score.

## Model guidance

No model is pinned. Authoring is planning, so match the model to the work:
Opus or an equivalent high-reasoning configuration for greenfield architecture
and unfamiliar systems, Sonnet for scoped feature specs. In Claude Code,
`opusplan` fits naturally.

## Compatibility

Targets `spec-grader` rubric v1.0.0 (16 categories). The section contract maps
onto those category IDs, which the grader commits to keeping stable.
````

- [ ] **Step 5: Run the check to verify it passes**

Run: `check_dist; echo "exit=$?"`
Expected: six `OK` lines, `exit=0`.

- [ ] **Step 6: Commit**

```bash
git add integrations/cursor/spec-author.md README.md
git commit -m "Add Cursor integration and README"
```

---

### Task 6: Dogfood validation against AC-1 through AC-11

**Files:**
- Create: `/private/tmp/claude-501/-Users-mavrick-workbench/6debc57f-0a48-42f5-ac0e-f33ebd038ae5/scratchpad/dogfood/SPEC.md` (scratch, not committed)
- Modify: none — this task changes the skill only if a criterion fails.

**Interfaces:**
- Consumes: every file built in Tasks 1–5.
- Produces: a pass/fail record for AC-1 through AC-11 and any resulting fixes.

The objective to author against, chosen because it exercises `codebase-informed`
mode with a real repository sitting next door and touches no private code:

> Add a `--diff` flag to `spec-grade` that compares two existing grade reports and reports per-category movement.

- [ ] **Step 1: Verify the whole skill is structurally sound before use**

```bash
check_evidence && check_template && check_skill && check_modes && check_dist && echo "ALL STRUCTURAL CHECKS PASS"
```
Expected: `ALL STRUCTURAL CHECKS PASS`.

- [ ] **Step 2: Run the skill on the objective**

Invoke `spec-author` with the objective above, from `/Users/mavrick/workbench/spec-grader`, writing output to the scratch dogfood directory. Do not ask for grading — this run tests the default path.

Observe and record, as it runs:
- **AC-1** — mode is announced before any other action. Since the argument is prose issued in a repo holding relevant code, expected mode is `codebase-informed`, not `expand`.
- **AC-2** — no research or drafting happens before the objective restatement is confirmed.
- **AC-3** — this is a scoped internal change, so the run completes with zero external research calls.
- **AC-8** — the run terminates without invoking `spec-grade`, and its final message names the command that would.

- [ ] **Step 3: Check the produced spec structurally**

```bash
cd /private/tmp/claude-501/-Users-mavrick-workbench/6debc57f-0a48-42f5-ac0e-f33ebd038ae5/scratchpad/dogfood

# AC-6: all 18 sections, in template order
grep -oE '^## .+' SPEC.md

# AC-4: every inline marker resolves to a table row, and no stray classes are marked
echo "used:"; grep -oE '\[[AD][0-9]+\]' SPEC.md | sort -u | tr '\n' ' '; echo
echo "defined:"; grep -oE '^\| (A|D)[0-9]+ ' SPEC.md | tr -d '| ' | sort -u | tr '\n' ' '; echo
echo "stray class names in body:"; grep -cE '\b(author-fact|repo-observed|external-source|reasoned-proposal)\b' SPEC.md

# AC-5: claims about existing code carry references
grep -oE '`[a-zA-Z0-9_/.-]+\.(py|md|json|sh):[0-9]+`' SPEC.md | sort -u

# AC-7: inapplicable sections use the exact form
grep -n 'Not applicable' SPEC.md
```

Expected: 18 headings in template order; every used marker appears in `defined`; stray class count `0`; at least one `path:line` citation into `spec_grade.py` or `rubric.json`; any `Not applicable` line matching `Not applicable — <reason>`.

- [ ] **Step 4: Verify the grader-absent path (AC-10)**

```bash
cd "$(mktemp -d)" && mkdir -p isolated && cd isolated
ls ../../spec-grader/bin/spec-grade 2>/dev/null || echo "no grader on search path here"
```

Then invoke `spec-author` in this isolated directory on a trivial objective **with** grading requested. Expected: it emits `SPEC.md`, reports the grader as unavailable, and does not attempt to download or install anything.

- [ ] **Step 5: Verify the bounded cycle (AC-9)**

Re-run the Step 2 objective from `/Users/mavrick/workbench/spec-grader`, this time asking for grading. Count invocations:

```bash
# after the run, inspect what was produced
ls -1 *.grade.json *.grade.md 2>/dev/null
```
Expected: `spec-grade` invoked at most twice, exactly one revision pass, and any judgment-call finding recorded as an open decision rather than applied. Requires a configured provider; if no API key is present, record AC-9 as **deferred** rather than passed — do not substitute `--provider mock`, which returns fixture data and proves nothing.

- [ ] **Step 6: Verify the research-privacy rule (AC-11)**

Review the transcript of the Step 2 and Step 5 runs for any external search. Expected: none occurred (AC-3). If any did, confirm no query contained verbatim spec text or internal product names.

- [ ] **Step 7: Record results and fix what failed**

Write the pass/fail/deferred result for AC-1 through AC-11 into the commit message. For any failure, fix the responsible skill file, re-run that file's structural check, and re-run only the affected criterion. Do not weaken a criterion to make it pass.

- [ ] **Step 8: Commit**

```bash
cd /Users/mavrick/workbench/spec-author
git add -A
git commit -m "Validate spec-author against AC-1..AC-11 via dogfood run

$(printf 'Record each criterion as pass, fail-and-fixed, or deferred.')"
```

---

## Self-Review

**Spec coverage.** FR-1 → Task 3 (classification rules) and Task 6 Step 2 (AC-1). FR-2 → Task 3 phase 2, AC-2. FR-3 → Task 4 per-mode procedures. FR-4 → Task 3 phase 3 and Task 4, AC-3. FR-5 → Task 3 phase 4. FR-6 → Task 1 class table. FR-7 → Task 1 marking rule, AC-4. FR-8 → Task 2 template and numbering, AC-6. FR-9 → Task 2 empty sections, Task 3 invariants, AC-7. FR-10 → Task 2 provenance sections and sidecar thresholds. FR-11 → Task 2 promotion rule. FR-12 → Task 3 opt-in section, AC-8. FR-13 → Task 3 bounded cycle, AC-9. FR-14 → Task 3 search order, AC-10. Security section → Task 3 invariants (query hygiene, secret exclusion), AC-11. NFRs → Task 3 degradation and model guidance, Task 5 README; enforced by the "no model pinned" assertions in Tasks 3 and 5. Observability → Task 3 phases 1 and 3 state mode and inspection scope out loud. Deployment/rollback → Task 5 README installation section. No spec requirement is unassigned.

**Placeholder scan.** Every file's full content is written inline; no task says "similar to Task N" or defers detail. The only intentionally open item is AC-9's provider dependency, which the plan resolves explicitly by recording the criterion as deferred rather than faking it with `--provider mock`.

**Type consistency.** The six class names, the `[A#]`/`[D#]` marker forms, the 18 section headings, the `FR-#`/`AC-#` numbering, the three grader search paths, and the exact string `Not applicable — <reason>` are identical in Global Constraints and in Tasks 1, 2, 3, 5 and their assertions. The template's heading list in Task 2 and the Cursor file's inline heading list in Task 5 were written from the same source order and match. Shell helper functions `check_evidence`, `check_template`, `check_skill`, `check_modes`, `check_dist` are defined once each and reused by name in Task 6 Step 1.
