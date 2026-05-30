# kplane-dev/enhancements

Kplane Enhancement Proposals (**KPEPs**) — design documents for cross-repo, architectural, or strategic changes to the kplane platform.

Modeled after [kubernetes/enhancements](https://github.com/kubernetes/enhancements) (the upstream KEP process), trimmed to fit kplane's scale.

## When to write a KPEP

Write one when a change:

- **Affects multiple repos** (e.g. a new pattern that touches the apiserver, storage helpers, and a backend repo)
- **Establishes a precedent** future contributors will follow (interface contracts, registration patterns, deployment models)
- **Has architectural tradeoffs worth recording** for future engineers debugging "why did we do it this way"
- **May eventually be upstreamed** as a Kubernetes KEP — writing it in this format gets us most of the way there

**Don't write one** for routine bugfixes, single-repo feature work that doesn't change a public contract, or implementation details that can be reviewed at PR time.

## Process

1. **Draft locally** until the shape is roughly right.
2. **Open a PR** adding a new directory under `kpeps/` with the next KPEP number (e.g. `kpeps/0007-pluggable-storage-backends/`).
3. **Discuss in the PR** — comments, alternatives, scope adjustments. The PR is the design review surface, not Slack/DMs.
4. **Land in `provisional` status** once the high-level shape has consensus. The KPEP is now the canonical reference for the work.
5. **Update the status** as the work progresses: `provisional` → `implementable` → `implemented` → (optional) `withdrawn` / `replaced`.
6. **Link from work PRs.** Every PR that implements part of a KPEP cites it in the description.

## Status values

| Status | Meaning |
|---|---|
| `provisional` | Shape agreed, not yet committed to a release. Discussion can still re-scope. |
| `implementable` | Design is firm; work can be parceled into PRs against the relevant repos. |
| `implemented` | All the work landed. KPEP becomes historical reference. |
| `withdrawn` | We started but decided not to ship it. KPEP stays as the "why we didn't" record. |
| `replaced` | A later KPEP supersedes this one. Both stay; the older points forward. |

## Layout

```
kpeps/
  0001-<slug>/
    README.md         ← the KPEP itself
    images/           ← optional diagrams
  0002-<slug>/
    ...

templates/
  README.md           ← the KPEP template (start here when drafting)
```

KPEP numbers are sequential and never reused — even withdrawn or replaced ones keep their number.

## Relationship to issues

- **Issues** track concrete bugs and unscoped work in the implementation repos (e.g., `kplane-dev/apiserver/issues/...`, `kplane-dev/infrastructure/issues/...`).
- **KPEPs** track design decisions that need a paper trail.

Cross-link freely. A KPEP may spawn many issues; an issue may surface the need for a KPEP.

## Relationship to upstream Kubernetes KEPs

When a KPEP captures a pattern that could plausibly land upstream (e.g., apiserver extension points), the KPEP itself is structured so it can be lifted into the upstream `kubernetes/enhancements` repo with minimal reshaping. The template's section headings mirror the upstream KEP template intentionally.

## License

Apache-2.0. See [`LICENSE`](./LICENSE).
