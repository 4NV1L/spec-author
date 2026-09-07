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
<grader>/bin/spec-grade ./SPEC.md --model <model>
```

When the user does opt in, locate the grader in this order:

1. `./tools/spec-grader/bin/spec-grade`
2. `../spec-grader/bin/spec-grade`
3. `spec-grader/bin/spec-grade` elsewhere in the workspace

If none exists, say so and stop. Never download or install a grader.

Then run exactly one revision cycle: grade, apply only fixes that clear a
readiness gate or close a gap using evidence already gathered, regrade once,
stop. Never pass `--improve` to the grader — you apply the fixes yourself,
using evidence already gathered; the grader's own improver is a stateless
single-shot revision with no access to your research. A finding that requires
a judgment call is never auto-applied — record it as an open decision. There
is no third pass and no score that triggers further revision. Report the
score, the residual gaps, and the open decisions, and lead with the gaps
rather than the number.

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
