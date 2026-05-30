# KPEP-NNNN: <Short, descriptive title>

<!--
Copy this file when starting a new KPEP:
  cp -r templates kpeps/NNNN-short-slug
Pick the next unused NNNN. Fill in every section. Delete the HTML
comments before opening the PR.

Sections are roughly the same shape as a Kubernetes KEP (so this can
lift upstream cleanly if it ever needs to), but lighter — kplane is
smaller than upstream Kubernetes and most of the heavyweight ceremony
(PRR, version-skew strategy, release signoff) isn't useful at our scale.
-->

| Field | Value |
|---|---|
| **Status** | `provisional` <!-- or implementable / implemented / withdrawn / replaced --> |
| **Authors** | @<github-handle>, @<github-handle> |
| **Created** | YYYY-MM-DD |
| **Last updated** | YYYY-MM-DD |
| **Tracking issue** | <kplane-dev/infrastructure#N or N/A> |
| **Affected repos** | `kplane-dev/apiserver`, `kplane-dev/storage`, ... |
| **Supersedes** | KPEP-NNNN <!-- if applicable --> |

## Summary

<!--
One paragraph. What changes, in plain language. If a senior engineer
who hasn't been following this work reads only this paragraph, they
should understand what we're committing to and why.
-->

## Motivation

<!--
What problem does this solve. What's the cost of not solving it.
Concrete examples beat abstractions — point at PR numbers, log lines,
operational pain.
-->

### Goals

<!--
Numbered list. Each goal is a specific, testable outcome. Goals are
what the KPEP commits to delivering.
-->

1.
2.

### Non-goals

<!--
What this KPEP is explicitly NOT taking on. Things people might ask
"will this also do X?" and the answer is "no, separate KPEP."
-->

1.
2.

## Proposal

<!--
The "what" of the design, at the level a reviewer can argue with.
Cover the shape of the change without yet committing to every detail.
Diagrams welcome (drop them in images/ and reference inline).
-->

### User stories

<!--
1-3 concrete walkthroughs of how this change shows up to someone
using kplane. Engineers writing a new backend, operators deploying
the platform, customers via the cloud-ui — whoever the affected
audience is. Write from their POV, not the implementer's.
-->

#### Story 1: <name>

#### Story 2: <name>

### Notes / constraints / caveats

<!--
Anything subtle that's easy to miss. Upstream contracts we have to
honor, legacy behaviors we can't break, performance constraints, etc.
-->

## Design details

<!--
Now the "how." This is where reviewers verify the proposal is
actually buildable. Be concrete: which packages, which files,
which function signatures.

Avoid pseudo-code that hides hard parts. If a section feels hand-wavy,
that's the section to deepen.
-->

### Cross-repo coordination

<!--
KPEPs frequently touch several kplane repos. Spell out which repo
gets which change, in what order, and what the merge sequence is.

Example:
  1. kplane-dev/storage   — add registry.go, helpers/, conformance/
  2. kplane-dev/spanner   — self-register via init(); refactor onto helpers
  3. kplane-dev/apiserver — replace hardcoded selection with registry lookup
-->

### Test plan

<!--
What proves the change works. Doesn't have to be exhaustive — aim for
"a reviewer can read this and feel confident the implementation will
be validated." For KPEPs that include a conformance suite, point at
where the suite lives.

For perf-shaped changes: mention the benchmark + the expected delta
direction.
-->

## Risks and mitigations

<!--
Each risk is a sentence + a mitigation. Don't list theoretical
concerns; list the ones that would actually trip up implementation
or operation.
-->

| Risk | Mitigation |
|---|---|
| | |

## Alternatives considered

<!--
The serious ones we evaluated and rejected, with the reason. This is
the section that earns its keep three years from now when someone
asks "why didn't we just do X?" — you can point at this paragraph.
-->

### Alternative A: <name>

### Alternative B: <name>

## Implementation plan

<!--
Sequenced. What lands in what order. Distinguish:
  - MVP: minimum that the design is real
  - v1: feature-complete for first users
  - Future: things this KPEP enables but doesn't ship
-->

### MVP

### v1

### Future / out of scope

## Open questions

<!--
Honest list of things still up in the air. Better to surface them
here than have a reviewer find them. Each question should have a
short framing of the tradeoffs.
-->

1.
2.

## Drawbacks

<!--
What's bad about this approach. Even if we ship it.

Every nontrivial design has drawbacks; if this section is empty,
you probably haven't thought hard enough.
-->

## Upstream KEP candidacy

<!--
Optional. If the pattern this KPEP introduces could plausibly land
in kubernetes/enhancements as a Kubernetes KEP, note what the
upstream framing would look like and what the political/technical
blockers are. If it can't ever be upstream, say so.
-->

## History

<!--
Rolling log of significant updates. Keep it short — git history is
the detailed record. This is "shape" history.
-->

- YYYY-MM-DD: Initial draft (@author)
