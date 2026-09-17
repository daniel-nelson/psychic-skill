# Maintaining this skill

This file is about **skill authoring** — how to size, shape, cut and verify `SKILL.md` and its
reference files. `CLAUDE.md` holds the rules that bind every change to this repo; this file holds
the craft those rules assume, so that an agent maintaining the skill does not have to rediscover
the current standard before it can start.

It is a maintainer document, not skill content. It is deliberately **not** linked from `SKILL.md`,
no agent using the skill ever loads it, and it is exempt from the token budget below — but not
from `CLAUDE.md`'s "Does this earn its place?" rubric.

**This file is a digest, and reading it is not reading the standard.** Every quotation below was
scoped correctly when written — which is the trap: an accurate summary is indistinguishable from
the thing it summarises, with nothing inside it saying so. Saying so is what this label buys. It
does not make anyone read the sources; it removes the condition that produced the failure, so an
agent answering out of the digest can no longer believe it is answering out of the standard. Being
binary, the label is also the one part of this pair a check could assert. **The rule beside it: any
claim you are about to rely on gets checked in its own page and its own section first**, not merely
confirmed to exist somewhere. Every source below is named, and the published ones serve raw markdown.

**That rule does not enforce itself; do not mistake it for a check.** It asks you to judge whether
you checked well enough, and in all three instances this repo recorded, the actor believed they had.
Each was caught by a *second reader of the primary source* instead — a peer session with the page
already open, this repo's own review pass, and a peer catching the coordinator mid-sentence while
this rule was being drafted; the sibling repo that reported the failure mode falsified four of its
own assertions the same way. Judgment stays judgment, and a second reader on the primary source is
the cheap thing that has worked whenever a claim is about to change what ships. The residual hole,
stated rather than papered over: `CLAUDE.md`'s trigger fires on the first *edit* to a skill file, so
nothing makes an agent read these sources on a turn where no edit begins — the exact shape of the
failure this paragraph exists for. "…and before answering a question about skill authoring" would be
intent-classification, the shape that already failed, so the hole stays open rather than closed with
a rule that would not hold.

> **Last verified: 2026-09-17.** Checked against: the five `agentskills.io` pages, fetched as raw
> markdown; `code.claude.com/docs/en/context-window.md` and `code.claude.com/docs/en/skills.md`;
> the `@rvoh/*` `package.json` versions in `~/work/dream_and_psychic`; six of Anthropic's
> first-party skills on disk (`pdf`, `pptx`, `docx`, `xlsx`, `skill-creator`, `frontend-design`);
> `skills-ref@0.1.5` and `claude plugin validate` on Claude Code 2.1.274; and a fresh token and
> anchor measurement of this repo.

**The stamp is not stale-guidance narration, and must not be deleted as such.** `CLAUDE.md`'s
"Delete stale guidance cleanly" forbids narrating what the framework or the skill *used to do*; a
freshness marker records when these claims were last checked, not what anything used to be. Keep
**one stamp for the whole file and never per-claim dates**, the same design as the skill's own
ecosystem baseline — a file whose every paragraph carries a date rots into bookkeeping nobody
updates. What moves the date is governed at the end of this file, under "Refreshing this file".

---

## Where the standard lives

The authoritative specification is the **Agent Skills open standard at `agentskills.io`**. Five
pages matter for authoring:

| Page | URL |
|---|---|
| Specification | `https://agentskills.io/specification` |
| Best practices for skill creators | `https://agentskills.io/skill-creation/best-practices` |
| Optimizing skill descriptions | `https://agentskills.io/skill-creation/optimizing-descriptions` |
| Evaluating skill output quality | `https://agentskills.io/skill-creation/evaluating-skills` |
| Using scripts in skills | `https://agentskills.io/skill-creation/using-scripts` |

**Append `.md` to any of those URLs to get the raw markdown.** All five return `200` with
`content-type: text/markdown`. Read them that way. A rendering fetch summarizes, and the
qualifiers these pages hang their meaning on are exactly what a summary drops.

`anthropics/skills` redirects there — its `spec/agent-skills-spec.md` is now one sentence, "The spec
is now located at <https://agentskills.io/specification>" — and Claude Code's docs defer to it:
"Claude Code skills follow the [Agent Skills](https://agentskills.io) open standard."
`platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices` is the shorter
cross-product summary; where the two differ, the `agentskills.io` page is longer and more specific.

**This location has moved once and will move again.** A stale source list cost this skill's last
maintenance pass most of its budget — not a wrong fact, a right fact from a page that was no longer
the standard. Before trusting any source list, this one included, re-check that the pages still
resolve and still say what is quoted here.

## The two size limits, and why the token one binds

The spec pairs two numbers:

> The specification recommends keeping `SKILL.md` under 500 lines and 5,000 tokens — just the core
> instructions the agent needs on every run.
>
> — `best-practices.md` (its "specification" is a link to `/specification#progressive-disclosure`)

The 500-line figure is the one everyone quotes; the 5,000-token figure is the one that binds, and
it is mechanical rather than advisory. `code.claude.com/docs/en/context-window.md`, "What survives
compaction":

> | Invoked skill bodies | Re-injected, capped at 5,000 tokens per skill and 25,000 tokens total; oldest dropped first |

> Skill bodies are re-injected after compaction, but large skills are truncated to fit the
> per-skill cap, and the oldest invoked skills are dropped once the total budget is exceeded.
> **Truncation keeps the start of the file**, so put the most important instructions near the top
> of `SKILL.md`.

For a prose-dense file the token ceiling binds long before the line ceiling. This one cleared 500
lines comfortably at 443 while measuring **11,259 tokens** — 2.25× the cap — so everything from
line 134 down was dropped silently on re-injection: every pointer to every reference file, the
whole routing map. The rules survived and the map did not.

**Measure; do not estimate.** A chars/4 estimate happens to land close here, but it is not what the
runtime counts, and at tens of tokens of headroom the difference decides the answer. Self-contained
recipe:

```bash
D=$(mktemp -d) && cd "$D" && npm i -s @anthropic-ai/tokenizer
node -e '
const {countTokens} = require("@anthropic-ai/tokenizer");
const raw = require("fs").readFileSync(process.argv[1], "utf8");
const L = raw.split("\n").map((l, i, a) => i === a.length - 1 ? l : l + "\n");
const pfx = n => countTokens(L.slice(0, n).join(""));
let lo = 0, hi = L.length;
while (lo < hi) { const m = (lo + hi) >> 1; pfx(m) >= 5000 ? hi = m : lo = m + 1; }
console.log("tokens", countTokens(raw), "lines", raw.split("\n").length - 1, "5000-token line", lo);
' /path/to/SKILL.md
```

Binary-search the *growing prefix* of the real file, as above. Summing independently-tokenized
lines forces a token boundary at every newline and overstates the total.

`@anthropic-ai/tokenizer` is Anthropic's own package but self-describes as beta and ships a legacy
vocabulary; `tiktoken` cross-checks 5-6% lower. Treat the number as accurate to a few percent, and
keep real headroom rather than shaving the cap.

**Current state: `SKILL.md` is 4,931 tokens over 126 lines, with 69 tokens of headroom.** That
slack is tens of tokens, not hundreds. Any addition is measured with a real tokenizer *before* it
is written, and anything added has to say where the room comes from.

`SKILL.md` carries a sentinel line above its H1 so a truncated copy can notice it is truncated: it
names the file's last heading and says what to do if that heading is missing. If the last heading
is ever renamed, the sentinel must be updated with it. Check it with `grep -qxF`, never `grep -qF` —
a substring match passes against a heading that merely *starts* with the sentinel text (renaming
`## Troubleshooting Migrations` to `## Troubleshooting Migrations and Seeds` satisfies `-qF` and
correctly fails `-qxF`).

**Recovery is the agent's own `Read`, which is why the sentinel is worded as an instruction to
re-read the path.** "Claude Code does not re-read the skill file on later turns" (`skills.md:512`):
nothing in the harness will re-open `SKILL.md` for you, and re-injection hands back a truncated
body rather than the file. **Nor is recovery sticky.** Every compaction re-truncates, and the
harness's automatic re-read of recently-modified files does not restore an oversized one, so the
sentinel has to fire *again* each time — which is why its position is asserted inside the first
4,000 bytes rather than merely "near the top". A sentinel below the cut cannot report the cut.

**Reference files are not subject to the per-skill cap**, because they are read on demand rather
than re-injected as skill body. Eight of the seventeen here exceed 5,000 tokens on their own and
that is fine. One related fact is worth knowing, from the same docs page: after compaction Claude
Code re-reads at most five previously-read files, and "a file over 5,000 tokens comes back as a
path reference without its content." A large reference file survives as a path the agent can
re-open, not as content. That sentence sits under the "Files Claude read or edited" row and governs
the automatic re-read only; nothing caps an explicit `Read` call at 5,000 tokens.

## The cut test

> Ask yourself about each piece of content: "Would the agent get this wrong without this
> instruction?" If the answer is no, cut it. If you're unsure, test it. And if the agent already
> handles the entire task well without the skill, the skill may not be adding value.
>
> — `best-practices.md`, "Add what the agent lacks, omit what it knows"

The framing sentence above it is the reason the test is strict: "Every token in your skill competes
for the agent's attention with everything else in that window." And on the temptation to be
thorough:

> Overly comprehensive skills can hurt more than they help — the agent struggles to extract what's
> relevant and may pursue unproductive paths triggered by instructions that don't apply to the
> current task.

`CLAUDE.md`'s "Does this earn its place?" rubric is the same test with more steps, and it is the
binding one here: its point 1 (failure mode) and point 2 (discoverability elsewhere) are the two
halves of "would the agent get this wrong without this instruction." Run the rubric. The
`agentskills.io` sentence is its one-line form, not a second, looser standard.

The highest-value content, per the same page, is the opposite of general advice:

> The highest-value content in many skills is a list of gotchas — environment-specific facts that
> defy reasonable assumptions. These aren't general advice ("handle errors appropriately") but
> concrete corrections to mistakes the agent will make without being told otherwise.

## Calibrate control per rule, not per skill

> Not every part of a skill needs the same level of prescriptiveness. Match the specificity of your
> instructions to the fragility of the task.
>
> **Give the agent freedom** when multiple approaches are valid and the task tolerates variation.
> For flexible instructions, explaining *why* can be more effective than rigid directives — an
> agent that understands the purpose makes better context-dependent decisions.
>
> **Be prescriptive** when operations are fragile, consistency matters, or a specific sequence must
> be followed.
>
> Most skills have a mix. Calibrate each part independently.
>
> — `best-practices.md`, "Calibrating control"

So directive form is a **per-rule classification**, not one decision for the whole file. A rule
about a mechanically inevitable failure is the fragile branch, where the bare directive is endorsed
outright — the page's own prescriptive example carries no reasoning at all ("Run exactly this
sequence… Do not modify the command or add additional flags"). A rule that is really a style
preference is the flexible branch, where the *why* does the work.

The maintainer's own framing rule sharpens this: a non-negotiable rule rests on framework opinion
or mechanical inevitability, never on a capability argument an agent can invert. "The agent might
get this wrong" is invertible; "the framework does it this way" and "this fails by construction"
are not.

**Scope check on a line that reads like a general ban.** `/skill-creation/evaluating-skills`
contains "**Explain the why.** Reasoning-based instructions ('Do X because Y tends to cause Z')
work better than rigid directives ('ALWAYS do X, NEVER do Y')." That is a real quote and it is not
an authoring rule. It is on the evaluating page, not the best-practices page, under
`## Iterating on the skill`, inside the list introduced by "When prompting the LLM, include these
guidelines:" — advice for composing a *revision prompt* during eval iteration. Cited as a general
prohibition it produced a wrong decision in this repo's last maintenance pass.

**Capitalization is settled by no source, and three of them point three ways.** `agentskills.io`'s
own prescriptive example is ordinary-cased. `platform.claude.com` writes "ALWAYS use this exact
template structure:" in its own template and, inside a worked iteration example, has the reviewing
model suggest "stronger language such as 'MUST filter' instead of 'always filter.'" Anthropic's
`skill-creator` skill says the opposite: "If you find yourself writing ALWAYS or NEVER in all caps,
or using super rigid structures, that's a yellow flag — if possible, reframe and explain the
reasoning" — note the three conditionals in one sentence. The all-caps `ALWAYS`/`NEVER` in this
repo's Critical Rules is a house-style choice made here, on grounds of churn. Do not cite any of
these sources for it in either direction.

## Descriptions carry the whole burden of triggering

> At startup, [agents] load only the `name` and `description` of each available skill… This means
> the description carries the entire burden of triggering. If the description doesn't convey when
> the skill is useful, the agent won't know to reach for it.
>
> — `optimizing-descriptions.md`

Its four principles: use imperative phrasing ("Use this skill when…" rather than "This skill
does…"); focus on user intent, not implementation; err on the side of being pushy, listing contexts
including ones where the user doesn't name the domain; keep it concise. The 1024-character limit is
a hard one, enforced by the spec and by the frontmatter validators.

**Two caps, and they measure different things.** The spec caps `description` itself at 1,024
characters. Claude Code separately truncates the *skill listing* at 1,536 characters, and that
budget covers two fields: `when_to_use` is a real frontmatter field, "Appended to `description` in
the skill listing and counts toward the 1,536-character cap" (`skills.md:333`). Anyone budgeting a
description is budgeting both. This skill uses `description` alone, so the whole 1,536 is its own.

**The `name` field is validated and its rules are narrow** (`specification.md`, "The required `name`
field"): 1-64 characters; lowercase alphanumerics (`a-z`, `0-9`) and hyphens only; no leading,
trailing or consecutive hyphens; and **it must match the parent directory name** — so renaming the
skill is a directory rename too. `compatibility`, the spec field this repo does not use, "accepts a
string of up to 500 characters. Claude Code accepts the field but doesn't act on it"
(`skills.md:350`).

The description is the one place where spending tokens is unambiguously worth it: it sits inside the
window that survives compaction, and one that fails to trigger makes the rest of the file unreachable.

**The "third person" / "imperative" contradiction is resolvable, not a real conflict.**
`platform.claude.com`'s "Always write in third person" targets first- and second-person
*self-reference* — "I can help you…", "You can use this to…" — and does not forbid an imperative
use-clause; all of its own worked examples carry one. The shape that satisfies both, and the shape
this skill's description already uses: **third-person capability verbs, then `Covers …`, then
`Use this skill whenever …`.**

## What Anthropic ships versus what it prescribes

Measured on disk, 2026-09-17, across six first-party skills:

| Skill | `SKILL.md` lines | Reference `.md` | Table of contents |
|---|---:|---|---|
| `pdf` | 314 | `REFERENCE.md` 611, `FORMS.md` 294 | none |
| `skill-creator` | 485 | `references/schemas.md` 430 | none |
| `pptx` / `docx` / `xlsx` / `frontend-design` | 241 / 91 / 99 / 71 | none | — |

- **No table of contents anywhere**, including a 611-line reference file — while `skill-creator`'s
  own `SKILL.md` prescribes one: "For large reference files (>300 lines), include a table of
  contents." Navigation is done instead by pointer sentences in `SKILL.md` that say *when* to open
  each file.
- **Zero `ALWAYS` and two `NEVER`** across 1,650 lines of markdown in the four document skills.
- **Three of the six ship no reference markdown at all.** The heavy lifting in `docx`, `xlsx` and
  `pptx` is done by executable scripts and XSD schemas — content that never enters context.

Shipped practice is evidence, and when it contradicts a prescription the prescription is weak. This
is the main reason a table-of-contents pass was proposed for this repo and rejected: no current
source prescribes it, the source that does prescribe it does not follow it in its own shipped
skills, and `CLAUDE.md`'s duplication rubric counts a file's fourth restatement of its own contents
as a defect.

## Two rulings from the maintainer

Both reverse a plausible reading, so neither is re-litigated.

**The guides are not a discoverability source for the agent using the skill.** In his words:
"Agents won't know anything about the guides (or if they do, from training data, it is likely
outdated and incorrect)." `CLAUDE.md`'s rubric point 2 names `~/work/psychic-guides` as something
for the **skill author** to go read while evaluating a candidate, on a machine where that checkout
exists. It is not a claim that an agent running in someone's Psychic app can reach them, and
overlap with the guides is not by itself a reason to cut.

**Opinionated practice is not generic advice.** In his words: "The Psychic framework is
opinionated. This includes testing practices that I have seen many people and agents get 'wrong'
(from the point of view of Psychic). That's why these exist. I could be convinced to remove them,
but I don't want the quality of Psychic applications to suffer because agents fall to bad decisions
that are prevelant in the world of software development." The operative test when cutting for
concision: cut where a capable model left alone would do the same thing anyway; keep where the
prevalent industry default differs from Psychic's opinion. Trimming the generic exposition *around*
an opinionated rule is still in bounds.

## How this work goes wrong

**A decision repaired by review can be silently undone by the next prose that paraphrases it.**
Restatements — a dispatch brief, a summary, a commit message, a handoff — are drafted under time
pressure from the long form and lose its qualifications. It happened twice in one run, in the same
direction both times: the restatement dropped the qualifier that made the decision right. When a
restatement and its source disagree, the source wins; check the restatement against it rather than
working from the restatement.

## Verification, for a repo with no CI

There is no test suite, no lint config and no CI here. These are the checks, each with its expected
result, because a check with an undefined expected result is not verification.

- **`wc -l SKILL.md`** — under 500. Currently 126.
- **Token count of `SKILL.md`** — under 5,000, measured with the recipe above. Currently 4,931.
- **Every reference file still linked from `SKILL.md`**, all 17:
  `grep -o '](\([a-z0-9-]*\.md\)' SKILL.md | sed 's/](//' | sort -u`. The character class must
  include digits or `i18n.md` is missed.
- **Anchors: 0 broken.** Scope any count you record: `SKILL.md` plus the 17 reference files
  carry 119 anchored links, and `CHANGELOG.md` adds 4 more that are never maintained. Zero
  broken is the claim that must hold. No off-the-shelf command does this correctly. A hand-rolled
  slugifier must (a) strip fenced code before collecting headings, (b) **preserve `_`** — it is a
  `\w` character and GitHub keeps it, so stripping it as emphasis falsely breaks
  `migrations.md#aliased-belongsto-shorthand-modelaliasbelongs_to` and `console.md#node_env-defaults` —
  and (c) map **each** whitespace character to its own hyphen. Then: drop backticks, unwrap
  `[text](url)`, drop `*`, lowercase, delete `[^\w\s-]`, and suffix `-1`, `-2` for duplicate slugs.
  Getting any of these wrong produces false positives, not false negatives.
- **`npx -y skills-ref@0.1.5 validate .`** — expected to exit 1 with **exactly one** diagnostic,
  naming only `user-invocable`. A second diagnostic is a regression.
- **Reading the diff**, plus the adversarial-reviewer pass `CLAUDE.md` mandates.

## The validators, and the failures this repo accepts on purpose

Three exist, and all three fail here. None of them checks size, links, or body content — they are
frontmatter and packaging checks only, so passing them would say nothing about whether the skill is
any good.

| Validator | What it is | Verdict here |
|---|---|---|
| `claude plugin validate <dir>` | Claude Code's own, 2.1.233+ | Fails: "No manifest found in directory. Expected `.claude-plugin/marketplace.json` or `.claude-plugin/plugin.json`" |
| `npx skills-ref@0.1.5 validate <dir>` | Third-party, named in the spec's Validation section | Fails: `user-invocable` |
| `skill-creator/scripts/quick_validate.py` | Anthropic's, inside the skill-creator skill | Fails: `user-invocable`, and (newer copy) two `SKILL.md` files |

**Do not "fix" any of these three failures.**

- `claude plugin validate` wants a plugin manifest. This repo is a *plain skill*, not a plugin —
  `<skills-dir>/foo/SKILL.md` with no manifest is "a plain skill named `foo`" by the published
  disambiguation table — and the validator only enters component mode when pointed at a directory
  containing `skills/`, `agents/` or `commands/`. A layout mismatch, not a defect.
- `user-invocable: false` is deliberate, and the disagreement is documented rather than mysterious.
  `code.claude.com/docs/en/skills.md`: Claude Code accepts every field in its frontmatter table,
  while "claude.ai skill uploads, the Skills API, and packaging with `package_skill.py`" accept only
  the spec's six — `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools` —
  and reject anything else with a hard error of exactly the form both validators emit. This skill
  ships by git clone into a Claude Code skills directory, the path that accepts the field; it would
  have to be dropped only if the skill were uploaded to claude.ai or the Skills API.
- The second `SKILL.md` is `psychic-update-skill/`, an intentional sub-skill. Only the newer of the
  two `quick_validate.py` copies enforces one-`SKILL.md`-per-directory; the two copies differ, and
  neither is canonical. Prefer `skills-ref`, which enforces the same six keys.

## The skill is evaluable as it stands

`claude plugin eval` resolves a bare directory containing `SKILL.md` as a plugin (plugin name =
directory basename), loads the skill into a "with" arm and runs an automatic no-plugin baseline
arm. The only structure missing from this repo is an `evals/` directory; a minimum viable case is
`evals/<case>/prompt.md` plus `evals/<case>/graders/<name>.md`, and `claude plugin eval --help`
lists the grader types.

A measured one-case run against a scratch copy of this repo scored **with-skill 1.00, without-skill
0.00, Δ +1.00, for $0.22** (`--runs 1`, two arms, 30s). That is the whole feasibility question
answered: an eval suite is affordable and would be this repo's first objective check. It is filed as
a to-do (`psychic-skill-eval-suite`) rather than built, because it is a new durable surface with its
own maintenance story.

Caveat: the bare-directory resolution is undocumented. The published reference says a
skills-directory *plugin* needs `.claude-plugin/plugin.json`; the CLI resolves a plain skill
directory anyway. Measured behavior is the stronger evidence, but re-run the one-case suite before
relying on it rather than trusting this paragraph.

The eval docs also give the sharpest operational form of the cut test: "Remove or replace assertions
that always pass in both configurations… They inflate the with-skill pass rate without reflecting
actual skill value."

## Refreshing this file

Each claim above names the source that establishes it, so a single claim can be re-verified without
re-deriving the whole document. To move the stamp, run all of it:

1. **Fetch the five `agentskills.io` pages as raw markdown** (`curl -sS https://agentskills.io/<page>.md`)
   and confirm the quoted passages still exist, on the same page, under the same heading. Confirm
   the five-page list itself is still current against `https://agentskills.io/llms.txt`.
2. **Re-read `code.claude.com/docs/en/context-window.md`** for the compaction row and the
   truncation prose, and **`code.claude.com/docs/en/skills.md`** for the frontmatter field table
   and the six-field spec restriction.
3. **Re-sample Anthropic's first-party skills on disk** — `~/.claude/skills/synced/*/` and
   `~/.claude/plugins/marketplaces/*/plugins/*/skills/*/` — for line counts, all-caps usage and
   tables of contents. The synced path carries a UUID that changes; glob it rather than hard-coding
   it.
4. **Re-run the validators** and confirm the accepted failures are still exactly the accepted
   failures, with no additional diagnostic.
5. **Re-measure `SKILL.md`** — tokens, lines, headroom — re-run the anchor check, and update
   the figure in `CLAUDE.md`'s skill-authoring section to match.
6. **Re-verify the `@rvoh/*` baseline** in `SKILL.md` against `~/work/dream_and_psychic`, which
   `CLAUDE.md` requires before finalizing any skill change regardless.

**Then move the stamp, but only as far as the work actually went.** The date moves only if every
source in its "checked against" list was re-read in the same session that moves it. If you refreshed
part of the list, **narrow the list to what you re-read** and move the date with the narrowed list;
never carry the full list over a partial pass. A date attesting to work nobody did is worse than no
stamp at all, because it reads as evidence.

A date is the weakest form of this in any case: nobody downstream can check it. The stronger form is
a recorded content hash per source, with a test that fetches each source and compares — mechanical,
falsifiable, and no self-report anywhere in it. This repo has no test runner to hang that on, so it
is filed rather than built. Do not add a second stamp, do not date individual sections, and do not
record what this file said before.
