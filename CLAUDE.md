# CLAUDE.md — psychic-skill

This repo IS the `psychic-skill` itself. Never invoke `psychic-skill` against this repo — there is no Dream/Psychic application here to reason about.

## CLI command style: write `pnpm psy`, not bare `psy`

Runnable command examples in the skill always use the `pnpm psy ...` form (e.g. `pnpm psy sync`, `pnpm psy g:resource`, `pnpm psy g:encryption-key`). `SKILL.md` carries the single disclaimer (around line 21) that examples use `pnpm` but a reader should substitute their project's actual package manager (`yarn psy ...`, `npm run psy ...`) — that note is what makes the `pnpm` prefix stand for "your package manager," so individual examples do **not** drop the prefix. Bare `psy ...` is acceptable **only** in inline prose that refers to a command by name (e.g. "`psy console` sessions are exempt"), never in a runnable code block or a step a reader is meant to copy. When adding or editing any command example, write `pnpm psy`.

The agent-facing counterpart is Critical Rule #2 in `SKILL.md`, which tells a reader to detect the project's real package manager (from `package.json`'s `"packageManager"` field or the lockfile) and substitute — so writing `pnpm` here doesn't mislead a yarn/npm/bun project. Keep that rule and this note consistent; if either changes, update the other.

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

## Keep the ecosystem version baseline current

`SKILL.md` carries an "Ecosystem versions & staleness policy" block listing the `@rvoh/*` package versions the skill is written against. **Before finalizing any skill change, verify that block against the actual current published versions** — check the `version` in each package's `package.json` in the monorepo at `~/work/dream_and_psychic` (`@rvoh/dream`, `@rvoh/psychic`, `@rvoh/psychic-workers`, `@rvoh/psychic-websockets`, `@rvoh/psychic-spec-helpers`), or `npm view <pkg> version`. If any have moved, update the baseline block in the same PR. The list is exactly the packages the skill documents — do not add packages the skill doesn't cover (`@rvoh/dream-plugin-json-snapshot`, for instance, is intentionally excluded — see below). Never annotate per-feature "available since" versions; the single baseline plus the "stay current / upgrade if reality disagrees" policy is the whole mechanism, by design.

## What this skill deliberately does not document

### `@rvoh/dream-plugin-json-snapshot`

This plugin was considered for inclusion and explicitly declined. The rationale, for future reference:

The skill tells agents what to reach for when building Psychic apps. `@Snapshotable` comes with a sharp constraint — internal retention use only, wrong for user-facing data export — which means an agent reading the skill would need to apply real judgment about when to suggest it. That is a higher bar than most of what the skill documents, where the guidance is "here is how to do the thing."

There is also a category difference: workers, soft-delete, websockets are optional features but part of the core framework install. `@rvoh/dream-plugin-json-snapshot` is a separate package with a separate install and a narrow use case that most apps will never need. Adding specialized plugins that come with "do not misuse this" warnings risks producing worse agent behavior — an agent that knows about `@Snapshotable` but misreads context might suggest it whenever someone mentions compliance or data export, which is exactly the wrong outcome.

The plugin is fully documented in the psychic-guides site (`psychicframework.com/docs/plugins/snapshot`), which is the right place for developer discovery. If the use case becomes common enough that agents should suggest it by default, add a short paragraph near the workers section — not a full file — pointing to the docs and stating the one-liner constraint.

## When asked to "open a PR" / "ship" / "publish"

Before opening the PR, verify the working branch already contains the `VERSION` bump and the `CHANGELOG` entry for this change. If either is missing, add it first, then open the PR. Do not rely on a post-merge fixup.
