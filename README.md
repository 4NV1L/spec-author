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
