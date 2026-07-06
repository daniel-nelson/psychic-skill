---
name: psychic-update-skill
description: Update the psychic-skill to the latest version from GitHub
allowed-tools: Bash, Read, Write
---

# /psychic-update-skill

Upgrade psychic-skill to the latest version and show what's new.

## Inline upgrade flow

This section is referenced by the main skill preamble when it detects `UPGRADE_AVAILABLE`.

### Step 1: Ask the user (or auto-upgrade)

First, check if auto-upgrade is enabled:
```bash
_SKILL_DIR=""
for d in "${CLAUDE_SKILL_DIR:-}" "$HOME/.agents/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" ".agents/skills/psychic-skill" ".claude/skills/psychic-skill"; do
  [ -n "$d" ] && [ -x "$d/bin/psychic-skill-config" ] && _SKILL_DIR="$d" && break
done
_AUTO=""
[ -n "$_SKILL_DIR" ] && _AUTO=$("$_SKILL_DIR/bin/psychic-skill-config" get auto_upgrade 2>/dev/null || true)
echo "AUTO_UPGRADE=$_AUTO SKILL_DIR=$_SKILL_DIR"
```

**If `AUTO_UPGRADE=true`:** Skip AskUserQuestion. Log "Auto-upgrading psychic-skill v{old} → v{new}..." and proceed directly to Step 2. If `./setup` fails during auto-upgrade, restore from backup (`.bak` directory) and warn the user: "Auto-upgrade failed — restored previous version. Run `/psychic-update-skill` manually to retry."

**Otherwise**, use AskUserQuestion:
- Question: "psychic-skill **v{new}** is available (you're on v{old}). Upgrade now?"
- Options: ["Yes, upgrade now", "Always keep me up to date", "Not now", "Never ask again"]

**If "Yes, upgrade now":** Proceed to Step 2.

**If "Always keep me up to date":**
```bash
"$_SKILL_DIR/bin/psychic-skill-config" set auto_upgrade true
```
Tell user: "Auto-upgrade enabled. Future updates will install automatically." Then proceed to Step 2.

**If "Not now":** Write snooze state with escalating backoff (first snooze = 24h, second = 48h, third+ = 1 week), then continue with the current skill. Do not mention the upgrade again.
```bash
_SNOOZE_FILE=~/.psychic-skill/update-snoozed
_REMOTE_VER="{new}"
_CUR_LEVEL=0
if [ -f "$_SNOOZE_FILE" ]; then
  _SNOOZED_VER=$(awk '{print $1}' "$_SNOOZE_FILE")
  if [ "$_SNOOZED_VER" = "$_REMOTE_VER" ]; then
    _CUR_LEVEL=$(awk '{print $2}' "$_SNOOZE_FILE")
    case "$_CUR_LEVEL" in *[!0-9]*) _CUR_LEVEL=0 ;; esac
  fi
fi
_NEW_LEVEL=$((_CUR_LEVEL + 1))
[ "$_NEW_LEVEL" -gt 3 ] && _NEW_LEVEL=3
mkdir -p ~/.psychic-skill
echo "$_REMOTE_VER $_NEW_LEVEL $(date +%s)" > "$_SNOOZE_FILE"
```
Note: `{new}` is the remote version from the `UPGRADE_AVAILABLE` output — substitute it from the update check result.

Tell user the snooze duration: "Next reminder in 24h" (or 48h or 1 week, depending on level). Tip: "Set `auto_upgrade: true` in `~/.psychic-skill/config.yaml` for automatic upgrades."

**If "Never ask again":**
```bash
"$_SKILL_DIR/bin/psychic-skill-config" set update_check false
```
Tell user: "Update checks disabled. Run `/psychic-update-skill` manually anytime, or re-enable with: `~/.psychic-skill/bin/psychic-skill-config set update_check true`"
Continue with the current skill.

### Step 2: Reconcile every installed copy

psychic-skill can be installed in more than one root at once (`~/.agents`,
`~/.claude`, plus project-local `.agents`/`.claude`). The
host may load a different copy than the one that sorts first, so the upgrade must
bring **every** copy to the remote version, not just the first one found.
`bin/psychic-skill-update-apply` does that in one pass — it upgrades git copies
via fetch + reset and vendored copies via a single re-clone, reconciling both
global and project-local installs. It replaces the old single-directory detect →
upgrade → local-copy-sync sequence.

Find a copy that ships the script (prefer the newest, since a stale copy may
predate it), then run it:

```bash
APPLY=""
for d in "${CLAUDE_SKILL_DIR:-}" "$HOME/.agents/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" ".agents/skills/psychic-skill" ".claude/skills/psychic-skill"; do
  [ -n "$d" ] && [ -x "$d/bin/psychic-skill-update-apply" ] && APPLY="$d/bin/psychic-skill-update-apply" && break
done
if [ -n "$APPLY" ]; then
  "$APPLY"
else
  echo "NO_APPLY_SCRIPT"   # every copy predates the reconcile-all updater; use the fallback below
fi
```

**Optional preview:** run `"$APPLY" --plan` first to show the user what will
change without mutating anything (lists each copy, its version, and whether it
would upgrade). Skip the preview during an auto-upgrade.

Read the script's output and relay it:

- `REMOTE <version>` — the target version.
- One `COPY <dir> <old> -> <new> <status>` line per copy. Statuses: `upgraded`,
  `unchanged` (already current), `stashed` (upgraded, but local git changes were
  stashed — tell the user to `git stash pop` in that dir), `dev-symlink-skipped`
  (a developer install; leave it, they `git pull` the source clone), `failed`.
- `SUMMARY <oldmin> -> <new> (<u> upgraded, <c> unchanged, <s> skipped, <f> failed)`.

If any copy is `failed`, tell the user which one and that they can re-run
`/psychic-update-skill`. The script writes the just-upgraded marker and clears
the update cache itself when at least one copy was upgraded. If any copy was
vendored (project-local, no `.git`), remind the user to commit it.

**Fallback (only if `NO_APPLY_SCRIPT`):** every installed copy predates this
updater, so reconcile the git copies inline:

```bash
for d in "$HOME/.agents/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" ".agents/skills/psychic-skill" ".claude/skills/psychic-skill"; do
  [ -d "$d/.git" ] || continue
  [ -L "$d" ] && { echo "$d: dev symlink, skipped"; continue; }
  OLD=$(cat "$d/VERSION" 2>/dev/null || echo unknown)
  ( cd "$d" && git stash >/dev/null 2>&1; git fetch origin >/dev/null 2>&1 && git reset --hard origin/main >/dev/null 2>&1 && ./setup >/dev/null 2>&1 )
  echo "$d: $OLD -> $(cat "$d/VERSION" 2>/dev/null || echo unknown)"
done
mkdir -p ~/.psychic-skill && rm -f ~/.psychic-skill/last-update-check ~/.psychic-skill/update-snoozed
```

### Step 2.5: Dedupe a dual project install (optional)

Older project installs can carry BOTH a real `.claude/skills/psychic-skill` copy
AND a real `.agents/skills/psychic-skill` copy in the same repo. The two are
independent full clones that drift apart. The canonical shape is one real tree at
`.agents` with `.claude` referencing it (a symlink for pnpm/yarn/bun, a real copy
for npm — npm's `ignore-scripts=true` plus Windows symlink breakage make the real
copy the safe choice there). A helper detects and fixes this on demand; it never
runs automatically.

Only offer this when the current project has both as real (non-symlink) copies.
Preview with `--plan`, ask the user, then run it. The helper lives next to the
apply script (`psychic-skill-dedupe` in the same `bin/`); reuse the `$APPLY` dir
found in Step 2, or locate it the same way.

```bash
DEDUPE=""
[ -n "$APPLY" ] && DEDUPE="$(dirname "$APPLY")/psychic-skill-dedupe"
if [ -z "$DEDUPE" ] || [ ! -x "$DEDUPE" ]; then
  for d in "${CLAUDE_SKILL_DIR:-}" "$HOME/.agents/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" ".agents/skills/psychic-skill" ".claude/skills/psychic-skill"; do
    [ -n "$d" ] && [ -x "$d/bin/psychic-skill-dedupe" ] && DEDUPE="$d/bin/psychic-skill-dedupe" && break
  done
fi
if [ -n "$DEDUPE" ] && [ -x "$DEDUPE" ]; then
  "$DEDUPE" --plan .
else
  echo "NO_DEDUPE_SCRIPT"   # this install predates the dedupe helper; skip this step
fi
```

Read the `RESULT` line from the `--plan` output:

- `no-agents-copy`, `no-claude-copy`, `not-dual`, or `already-linked` — nothing to
  do; skip silently and move on to Step 3.
- `left-npm-realcopy` — the project uses npm; the real `.claude` copy is correct
  and stays. No action; mention it only if the user asked.
- `left-unknown-pm` — package manager couldn't be determined; leave it as is.
- `plan:convert` — a dual real install on pnpm/yarn/bun. The plan lists the links
  that would be created. **Ask the user** whether to convert `.claude` into a
  symlink that references the canonical `.agents` tree (removing the duplicate
  copy). If yes, run `"$DEDUPE" .` and relay its `RESULT` (`converted` or
  `failed`). Remind the user to commit the changed `.claude` links.

The helper is idempotent and safe to re-run.

### Step 3: Show What's New

Read `CHANGELOG.md` from any upgraded copy (e.g. the `$APPLY` dir). Find all
version entries between `{old}` (the `SUMMARY` `oldmin`) and `{new}`. Summarize as
5-7 bullets grouped by theme. Focus on user-facing changes. Skip internal
refactors unless significant.

Format:
```
psychic-skill v{new} — upgraded from v{old}!

What's new:
- [bullet 1]
- [bullet 2]
- ...
```

### Step 4: Continue

After showing What's New, continue with whatever skill the user originally
invoked. The upgrade is done — no further action needed.

---

## Standalone usage

When invoked directly as `/psychic-update-skill` (not from a preamble):

1. Force a fresh check. The checker scans every installed copy and reports the lowest version, so this catches a stale copy in a root the host doesn't load first. Avoid using shell variable names like `status` in zsh; `status` is read-only there.
```bash
UPDATE_CHECK_OUTPUT=""
UPDATE_CHECK_OK=false
for d in "${CLAUDE_SKILL_DIR:-}" "$HOME/.agents/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" ".agents/skills/psychic-skill" ".claude/skills/psychic-skill"; do
  if [ -n "$d" ] && [ -x "$d/bin/psychic-skill-update-check" ]; then
    UPDATE_CHECK_OUTPUT=$("$d/bin/psychic-skill-update-check" --force 2>/dev/null)
    rc=$?
    [ $rc -eq 0 ] && UPDATE_CHECK_OK=true
    break
  fi
done
echo "UPDATE_CHECK_OK=$UPDATE_CHECK_OK"
echo "UPDATE_CHECK_OUTPUT=$UPDATE_CHECK_OUTPUT"
```

2. If `UPDATE_CHECK_OUTPUT` contains `UPGRADE_AVAILABLE <old> <new>`: run the inline flow (Step 2 reconcile onward). The `--plan` preview is a good idea here so the user sees which copies are behind before anything changes.

3. **Otherwise — do not trust silence.** No output can mean the check was cached, hit a disabled flag, or failed (sandbox filesystem restrictions); it does **not** prove the install is current. Just run the reconcile directly — `bin/psychic-skill-update-apply` fetches the remote version itself, is a no-op for copies already current, and reports per-copy status, so it is safe to run unconditionally. Locate it as in Step 2 and run `"$APPLY" --plan` then `"$APPLY"`. If no copy ships the script either, use the Step 2 fallback loop. If the reconcile reports every copy `unchanged`, tell the user "You're on the latest version (v{version}); all installed copies are up to date."
