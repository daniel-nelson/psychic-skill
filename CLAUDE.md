# CLAUDE.md — psychic-skill

This repo IS the `psychic-skill` itself. Never invoke `/psychic-skill` (the `dream-psychic` skill) against this repo — there is no Dream/Psychic application here to reason about.

## Release process (required on every PR)

"Publishing" a version means merging into `main`. There is no separate publish step — whatever is on `main` is the released skill, and installs upgrade by pulling `main`.

Because of that, **every PR must include both of the following**, or it is not ready to open:

1. **A `VERSION` bump.**
   - **Minor bump** (`0.34.0` → `0.35.0`) — the default for almost everything: new guidance, new sections, reworked explanations, new rules, syncing the skill to a framework change.
   - **Patch bump** (`0.34.0` → `0.34.1`) — reserved for small corrections: typo fixes, broken-link/anchor fixes, factual corrections to existing prose, or an odd one-off fix that doesn't introduce new guidance.
   - When in doubt, bump minor.

2. **A matching `CHANGELOG.md` entry** under a new `## <version> — <YYYY-MM-DD>` heading, grouped into `### Added` / `### Changed` / `### Removed` as appropriate. Describe user-facing skill changes (what an agent reading the skill will now see differently), not internal churn.

### One unmerged branch = one version

Merging to `main` **is** the publishing event. A version that never merged was never released, so it does not get its own `CHANGELOG` section.

Consequently, while a branch/PR is still open, it carries **exactly one** version heading — the version it will publish as. If the scope of the open PR grows (more commits, a bigger change class), do not add a second `## <version>` section; instead bump the single heading to the new appropriate version and fold all the branch's changes under it. Multiple `## <version>` sections only ever exist for versions that have *actually merged* in separate PRs. Never append a later PR's changes into an already-merged version's section.

A PR that changes skill content without a `VERSION` bump + `CHANGELOG` entry is incomplete. Treat the bump and changelog as part of the change, not a follow-up.

## When asked to "open a PR" / "ship" / "publish"

Before opening the PR, verify the working branch already contains the `VERSION` bump and the `CHANGELOG` entry for this change. If either is missing, add it first, then open the PR. Do not rely on a post-merge fixup.
