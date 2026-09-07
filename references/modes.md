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
