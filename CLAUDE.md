# CLAUDE.md — psychic-skill

This repo IS the `psychic-skill` itself. Never invoke `psychic-skill` against this repo — there is no Dream/Psychic application here to reason about.

## CLI command style: write `pnpm psy`, not bare `psy`

Runnable command examples in the skill always use the `pnpm psy ...` form (e.g. `pnpm psy sync`, `pnpm psy g:resource`, `pnpm psy g:encryption-key`). `SKILL.md` carries the single disclaimer (around line 21) that examples use `pnpm` but a reader should substitute their project's actual package manager (`yarn psy ...`, `npm run psy ...`) — that note is what makes the `pnpm` prefix stand for "your package manager," so individual examples do **not** drop the prefix. Bare `psy ...` is acceptable **only** in inline prose that refers to a command by name (e.g. "`psy console` sessions are exempt"), never in a runnable code block or a step a reader is meant to copy. When adding or editing any command example, write `pnpm psy`.

The agent-facing counterpart is Critical Rule #2 in `SKILL.md`, which tells a reader to detect the project's real package manager (from `package.json`'s `"packageManager"` field or the lockfile) and substitute — so writing `pnpm` here doesn't mislead a yarn/npm/bun project. Keep that rule and this note consistent; if either changes, update the other.

## Example domain: BearBnB

Every example in the skill — models, controllers, serializers, generators, migrations, routing — uses one shared domain: BearBnB, a bear-themed short-term-rental app (Airbnb for bears). The core nouns are `Place`, `Room` (an STI base with children `Bathroom`/`Bedroom`/`Den`), `Host`, `Guest`, `Booking`, `City`, and `LocalizedText`. Reusing these nouns keeps examples mutually consistent — an association in one file lines up with the model and serializer shown in another — so when adding or editing any example, draw from this domain rather than introducing a new one, and extend the noun set only when no existing noun fits.

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

## Delete stale guidance cleanly

When correcting outdated or incorrect skill guidance, remove the stale pattern without adding explanatory prose whose only purpose is to contrast with, justify, or memorialize the removed mistake. The skill should teach the current generated/currently-correct shape directly. Add an explanation only when it helps an agent make a future implementation decision that remains relevant after the old guidance is gone.

The same rule applies to the framework itself, not just to the skill's own prior wording: **do not document past or changed behaviors in the Dream/Psychic ecosystem.** The skill always teaches the current (or imminent) state, never narrates what the framework used to do or that a behavior changed. State the behavior as it is now; never write "an older version did X; current tooling does Y," "this used to…," or "X disappears on regen — that's a correction." A reader on an older package version is served by the "Ecosystem versions & staleness policy" baseline ("upgrade if reality disagrees"), not by historical narration in the skill. The one exception is a breaking change that genuinely needs migration steps — that belongs in a dedicated `upgrade.md` (or similar) with explicit version-to-version guidance, not woven into the feature docs.

## Does this earn its place? Evaluating a candidate addition

Being true is not sufficient. Every added word costs context and dilutes what surrounds it, so the test for any candidate addition is **what an agent building a Psychic app would have to go through to learn this without the skill.** Guidance an agent would discover in seconds on their own is a net negative: it spends context and buries the guidance that isn't discoverable.

Run this before adding anything — a processed learning, a new section, an expansion of an existing one.

### The rubric

1. **Failure mode.** If an agent doesn't know this, what happens?
   - Silent wrong behavior, data corruption, or a production 500 that specs pass over — highest value.
   - A long confusing debugging loop — high.
   - A runtime error whose own message states the rule — low. Dream's error messages are unusually explicit; several name the whole rule (`Can only pass BelongsTo associated models as params`).
   - An immediate TypeScript error at the call site — very low, and lower still because `pnpm build:spec` typechecks `spec/` too.
2. **Discoverability elsewhere.** Would the agent hit this first in the TSDoc on the method or decorator they are already calling, in the error text, in the guides at `~/work/psychic-guides`, or in `pnpm psy <cmd> --help`? Go read those and quote what you find. A `@returns` line on the exact method being documented means the skill does not need to restate it.
3. **Duplication.** Grep every `*.md` in this repo. Guidance stated twice competes with itself, and the second statement is usually the one that drifts.
4. **Frequency.** Does a typical Psychic app hit this, or does it need an uncommon column type, a rare refactor, or a horizontally-scaled fleet?
5. **Word cost.** Count the words. Would the same value survive at a third of the length? Usually yes.

A candidate that clears (1) and (2) earns its place. One that fails (2) does not, no matter how true it is.

### How to run the evaluation

Spawn one adversarial reviewer per file being changed, in parallel, and tell each one the facts are already verified so it spends its whole budget on the value question. Require of each verdict:

- `KEEP AS IS` / `TRIM` / `CUT` / `REFRAME`, with word counts now and proposed.
- Evidence bullets that **quote something concrete** — the error string, the TSDoc line, the guides passage, the existing skill line that already says it. A verdict without quotes is an opinion.
- The exact replacement markdown when trimming, in this repo's conventions.
- The strongest argument against its own verdict.

Reviewers are advisory. Nothing gets edited until a human has seen the verdicts, and a reviewer that keeps everything has told you nothing.

### Prescriptions get a second test

When a candidate tells an agent what to *do* rather than what the framework *does*, also check: does the framework's own scaffold or the guides prescribe something different, is the claimed benefit real or a micro-optimization, and what does the recommendation cost in exchange? State the honest claim. If the real content is "you must do this at all" rather than "do it here instead of there", write that instead — the comparison is usually the weaker half.

## What this skill deliberately does not document

### `@rvoh/dream-plugin-json-snapshot`

This plugin was considered for inclusion and explicitly declined. The rationale, for future reference:

The skill tells agents what to reach for when building Psychic apps. `@Snapshotable` comes with a sharp constraint — internal retention use only, wrong for user-facing data export — which means an agent reading the skill would need to apply real judgment about when to suggest it. That is a higher bar than most of what the skill documents, where the guidance is "here is how to do the thing."

There is also a category difference: workers, soft-delete, websockets are optional features but part of the core framework install. `@rvoh/dream-plugin-json-snapshot` is a separate package with a separate install and a narrow use case that most apps will never need. Adding specialized plugins that come with "do not misuse this" warnings risks producing worse agent behavior — an agent that knows about `@Snapshotable` but misreads context might suggest it whenever someone mentions compliance or data export, which is exactly the wrong outcome.

The plugin is fully documented in the psychic-guides site (`psychicframework.com/docs/plugins/snapshot`), which is the right place for developer discovery. If the use case becomes common enough that agents should suggest it by default, add a short paragraph near the workers section — not a full file — pointing to the docs and stating the one-liner constraint.

## When asked to "Put up a PR" / "open a PR" / "ship" / "publish"

Treat this as the complete release workflow for the current change:

1. If currently on `main`, create a new branch with a short descriptive name.
2. Verify the working branch contains the intended tracked changes and no unrelated user changes are staged.
3. Bump `VERSION` for the change scope.
4. Add the matching `CHANGELOG.md` entry under a new `## <version> — <YYYY-MM-DD>` heading.
5. Commit the intended files, push the branch, and open the PR.

Do not open the PR until the branch includes the version bump and changelog entry. Do not rely on a post-merge fixup.
