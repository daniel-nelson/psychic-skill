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
for d in "${CLAUDE_SKILL_DIR:-}" "${CODEX_SKILL_DIR:-}" "$HOME/.agents/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" "$HOME/.codex/skills/psychic-skill" ".agents/skills/psychic-skill" ".claude/skills/psychic-skill" ".codex/skills/psychic-skill"; do
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

### Step 2: Detect install type

```bash
if [ -d "$HOME/.agents/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.agents/skills/psychic-skill"
elif [ -d "$HOME/.claude/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.claude/skills/psychic-skill"
elif [ -d "$HOME/.codex/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.codex/skills/psychic-skill"
elif [ -d ".agents/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".agents/skills/psychic-skill"
elif [ -d ".claude/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".claude/skills/psychic-skill"
elif [ -d ".codex/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".codex/skills/psychic-skill"
elif [ -d "$HOME/.agents/skills/psychic-skill" ]; then
  INSTALL_TYPE="vendored-global"
  INSTALL_DIR="$HOME/.agents/skills/psychic-skill"
elif [ -d ".agents/skills/psychic-skill" ]; then
  INSTALL_TYPE="vendored"
  INSTALL_DIR=".agents/skills/psychic-skill"
elif [ -d ".claude/skills/psychic-skill" ]; then
  INSTALL_TYPE="vendored"
  INSTALL_DIR=".claude/skills/psychic-skill"
elif [ -d "$HOME/.claude/skills/psychic-skill" ]; then
  INSTALL_TYPE="vendored-global"
  INSTALL_DIR="$HOME/.claude/skills/psychic-skill"
elif [ -d ".codex/skills/psychic-skill" ]; then
  INSTALL_TYPE="vendored"
  INSTALL_DIR=".codex/skills/psychic-skill"
elif [ -d "$HOME/.codex/skills/psychic-skill" ]; then
  INSTALL_TYPE="vendored-global"
  INSTALL_DIR="$HOME/.codex/skills/psychic-skill"
else
  echo "ERROR: psychic-skill not found"
  exit 1
fi
echo "Install type: $INSTALL_TYPE at $INSTALL_DIR"
```

The install type and directory path printed above will be used in all subsequent steps.

### Step 3: Save old version

Use the install directory from Step 2's output:

```bash
OLD_VERSION=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
```

### Step 4: Upgrade

Use the install type and directory detected in Step 2:

**For git installs** (global-git, local-git):
```bash
cd "$INSTALL_DIR"
STASH_OUTPUT=$(git stash 2>&1)
git fetch origin
git reset --hard origin/main
./setup
```
If `$STASH_OUTPUT` contains "Saved working directory", warn the user: "Note: local changes were stashed. Run `git stash pop` in the skill directory to restore them."

**For vendored installs** (vendored, vendored-global):
```bash
PARENT=$(dirname "$INSTALL_DIR")
TMP_DIR=$(mktemp -d)
git clone --depth 1 https://github.com/daniel-nelson/psychic-skill.git "$TMP_DIR/psychic-skill"
mv "$INSTALL_DIR" "$INSTALL_DIR.bak"
mv "$TMP_DIR/psychic-skill" "$INSTALL_DIR"
cd "$INSTALL_DIR" && ./setup
rm -rf "$INSTALL_DIR.bak" "$TMP_DIR"
```

### Step 4.5: Sync local vendored copies

Use the install directory from Step 2. Check if there are also local vendored copies that need updating:

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || true)
LOCAL_COPIES=""
for _SUBDIR in .agents/skills/psychic-skill .claude/skills/psychic-skill .codex/skills/psychic-skill; do
  if [ -n "$_ROOT" ] && [ -d "$_ROOT/$_SUBDIR" ]; then
    _RESOLVED_LOCAL=$(cd "$_ROOT/$_SUBDIR" && pwd -P)
    _RESOLVED_PRIMARY=$(cd "$INSTALL_DIR" && pwd -P)
    if [ "$_RESOLVED_LOCAL" != "$_RESOLVED_PRIMARY" ]; then
      LOCAL_COPIES="${LOCAL_COPIES}${LOCAL_COPIES:+
}$_ROOT/$_SUBDIR"
    fi
  fi
done
printf 'LOCAL_COPIES=%s\n' "$LOCAL_COPIES"
```

For each path in `LOCAL_COPIES`, compare versions first. Only sync copies whose version differs from the primary:
```bash
PRIMARY_VER=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
printf '%s\n' "$LOCAL_COPIES" | while IFS= read -r LOCAL_COPY; do
  [ -n "$LOCAL_COPY" ] || continue
  LOCAL_VER=$(cat "$LOCAL_COPY/VERSION" 2>/dev/null || echo "unknown")
  if [ "$LOCAL_VER" = "$PRIMARY_VER" ]; then
    echo "LOCAL_COPY_CURRENT $LOCAL_COPY $LOCAL_VER"
    continue
  fi

  rm -rf "$LOCAL_COPY.bak"
  mv "$LOCAL_COPY" "$LOCAL_COPY.bak"
  cp -Rf "$INSTALL_DIR" "$LOCAL_COPY"
  rm -rf "$LOCAL_COPY/.git"
  (cd "$LOCAL_COPY" && ./setup)
  rm -rf "$LOCAL_COPY.bak"
  echo "LOCAL_COPY_UPDATED $LOCAL_COPY $LOCAL_VER $PRIMARY_VER"
done
```
Tell user which vendored copies were updated and which were already current. If any were updated, add: "Commit the vendored copy when you're ready."

If `./setup` fails, restore from backup and warn the user:
```bash
rm -rf "$LOCAL_COPY"
mv "$LOCAL_COPY.bak" "$LOCAL_COPY"
```
Tell user: "Sync failed — restored previous version at `$LOCAL_COPY`. Run `/psychic-update-skill` manually to retry."

### Step 5: Write marker + clear cache

```bash
mkdir -p ~/.psychic-skill
echo "$OLD_VERSION" > ~/.psychic-skill/just-upgraded-from
rm -f ~/.psychic-skill/last-update-check
rm -f ~/.psychic-skill/update-snoozed
```

### Step 6: Show What's New

Read `$INSTALL_DIR/CHANGELOG.md`. Find all version entries between the old version and the new version. Summarize as 5-7 bullets grouped by theme. Focus on user-facing changes. Skip internal refactors unless significant.

Format:
```
psychic-skill v{new} — upgraded from v{old}!

What's new:
- [bullet 1]
- [bullet 2]
- ...
```

### Step 7: Continue

After showing What's New, continue with whatever skill the user originally invoked. The upgrade is done — no further action needed.

---

## Standalone usage

When invoked directly as `/psychic-update-skill` (not from a preamble):

1. Detect the install directory first (run the Step 2 bash block). Then attempt a forced update check. Avoid using shell variable names like `status` in zsh; `status` is read-only there.
```bash
UPDATE_CHECK_OUTPUT=""
UPDATE_CHECK_OK=false
for d in "${CLAUDE_SKILL_DIR:-}" "${CODEX_SKILL_DIR:-}" "$HOME/.agents/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" "$HOME/.codex/skills/psychic-skill" ".agents/skills/psychic-skill" ".claude/skills/psychic-skill" ".codex/skills/psychic-skill"; do
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

2. If `UPDATE_CHECK_OUTPUT` contains `UPGRADE_AVAILABLE <old> <new>`: follow Steps 2-6 above.

3. If `UPDATE_CHECK_OK=true` and no `UPGRADE_AVAILABLE` in output: do **not** assume the install is current yet. The helper may have emitted `JUST_UPGRADED`, `UP_TO_DATE`, or no output from cached/local state. Confirm by comparing the installed version to the remote version using the Step 4 fallback block. If the remote differs, treat it as `UPGRADE_AVAILABLE $INSTALLED_VERSION $REMOTE_VERSION` and follow Steps 2-6. If they match, continue to step 5.

4. **If `UPDATE_CHECK_OK=false`** (script failed or was not found — e.g. sandbox filesystem restrictions): fall back to directly checking the remote version:
```bash
INSTALLED_VERSION=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
REMOTE_VERSION=$(curl -fsSL https://raw.githubusercontent.com/daniel-nelson/psychic-skill/main/VERSION 2>/dev/null | tr -d '[:space:]')
echo "INSTALLED=$INSTALLED_VERSION REMOTE=$REMOTE_VERSION"
```
- If `REMOTE_VERSION` is empty (no network): tell the user "Could not reach GitHub to check for updates. Verify network access and try again." Stop.
- If `REMOTE_VERSION` differs from `INSTALLED_VERSION`: treat this as `UPGRADE_AVAILABLE $INSTALLED_VERSION $REMOTE_VERSION` and follow Steps 2-6.
- If they match: the install is current. Continue to step 5.

**Never conclude "up to date" solely because the update-check script produced no output.** No output can mean the script failed. Always confirm by comparing installed vs. remote version before telling the user they are current.

5. Check for stale local vendored copies. Run the Step 2 bash block to confirm `INSTALL_TYPE` and `INSTALL_DIR`, then run the Step 4.5 detection bash block to check for `LOCAL_COPIES`.

**If `LOCAL_COPIES` is empty** (no local vendored copies): tell the user "You're already on the latest version (v{version})."

**If `LOCAL_COPIES` is non-empty**, compare versions for every copy:
```bash
PRIMARY_VER=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
printf '%s\n' "$LOCAL_COPIES" | while IFS= read -r LOCAL_COPY; do
  [ -n "$LOCAL_COPY" ] || continue
  LOCAL_VER=$(cat "$LOCAL_COPY/VERSION" 2>/dev/null || echo "unknown")
  echo "PRIMARY=$PRIMARY_VER LOCAL_COPY=$LOCAL_COPY LOCAL=$LOCAL_VER"
done
```

**If any versions differ:** follow the Step 4.5 sync bash block above to update stale local copies from the primary. Tell user: "Global v{PRIMARY_VER} is up to date. Updated stale local vendored copies to v{PRIMARY_VER}. Commit the vendored copies when you're ready."

**If all versions match:** tell the user "You're on the latest version (v{PRIMARY_VER}). Global and local vendored copies are all up to date."
