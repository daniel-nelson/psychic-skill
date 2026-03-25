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
for d in "${CLAUDE_SKILL_DIR:-}" "${CODEX_SKILL_DIR:-}" "$HOME/.claude/skills/psychic-skill" "$HOME/.codex/skills/psychic-skill" ".claude/skills/psychic-skill" ".codex/skills/psychic-skill"; do
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
if [ -d "$HOME/.claude/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.claude/skills/psychic-skill"
elif [ -d "$HOME/.codex/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.codex/skills/psychic-skill"
elif [ -d ".claude/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".claude/skills/psychic-skill"
elif [ -d ".codex/skills/psychic-skill/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".codex/skills/psychic-skill"
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

### Step 4.5: Sync local vendored copy

Use the install directory from Step 2. Check if there's also a local vendored copy that needs updating:

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || true)
LOCAL_COPY=""
for _SUBDIR in .claude/skills/psychic-skill .codex/skills/psychic-skill; do
  if [ -n "$_ROOT" ] && [ -d "$_ROOT/$_SUBDIR" ]; then
    _RESOLVED_LOCAL=$(cd "$_ROOT/$_SUBDIR" && pwd -P)
    _RESOLVED_PRIMARY=$(cd "$INSTALL_DIR" && pwd -P)
    if [ "$_RESOLVED_LOCAL" != "$_RESOLVED_PRIMARY" ]; then
      LOCAL_COPY="$_ROOT/$_SUBDIR"
      break
    fi
  fi
done
echo "LOCAL_COPY=$LOCAL_COPY"
```

If `LOCAL_COPY` is non-empty, update it by copying from the freshly-upgraded primary install:
```bash
mv "$LOCAL_COPY" "$LOCAL_COPY.bak"
cp -Rf "$INSTALL_DIR" "$LOCAL_COPY"
rm -rf "$LOCAL_COPY/.git"
cd "$LOCAL_COPY" && ./setup
rm -rf "$LOCAL_COPY.bak"
```
Tell user: "Also updated vendored copy at `$LOCAL_COPY` — commit it when you're ready."

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

1. Force a fresh update check (bypass cache):
```bash
for d in "${CLAUDE_SKILL_DIR:-}" "${CODEX_SKILL_DIR:-}" "$HOME/.claude/skills/psychic-skill" "$HOME/.codex/skills/psychic-skill" ".claude/skills/psychic-skill" ".codex/skills/psychic-skill"; do
  [ -n "$d" ] && [ -x "$d/bin/psychic-skill-update-check" ] && "$d/bin/psychic-skill-update-check" --force 2>/dev/null && break
done
```
Use the output to determine if an upgrade is available.

2. If `UPGRADE_AVAILABLE <old> <new>`: follow Steps 2-6 above.

3. If no output (primary is up to date): check for a stale local vendored copy.

Run the Step 2 bash block above to detect the primary install type and directory (`INSTALL_TYPE` and `INSTALL_DIR`). Then run the Step 4.5 detection bash block above to check for a local vendored copy (`LOCAL_COPY`).

**If `LOCAL_COPY` is empty** (no local vendored copy): tell the user "You're already on the latest version (v{version})."

**If `LOCAL_COPY` is non-empty**, compare versions:
```bash
PRIMARY_VER=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
LOCAL_VER=$(cat "$LOCAL_COPY/VERSION" 2>/dev/null || echo "unknown")
echo "PRIMARY=$PRIMARY_VER LOCAL=$LOCAL_VER"
```

**If versions differ:** follow the Step 4.5 sync bash block above to update the local copy from the primary. Tell user: "Global v{PRIMARY_VER} is up to date. Updated local vendored copy from v{LOCAL_VER} → v{PRIMARY_VER}. Commit the vendored copy when you're ready."

**If versions match:** tell the user "You're on the latest version (v{PRIMARY_VER}). Global and local vendored copy are both up to date."
