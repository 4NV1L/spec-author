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
use the bundled grader at `./tools/spec-grader/bin/spec-grade`, then
`../spec-grader/bin/spec-grade`, then `spec-grader/bin/spec-grade` elsewhere in
the workspace; do not download or install a grader. Run at most one revision cycle: grade, apply only evidence-backed and
gate-clearing fixes, regrade once, stop. Record anything needing a judgment
call as an open decision instead of applying it.

Never invent user research or metrics, never author security or retention
policy without the user's direction, never claim codebase behaviour you have
not read, and never put confidential product details into an external search
query.
