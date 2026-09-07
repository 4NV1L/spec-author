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
