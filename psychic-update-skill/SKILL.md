---
name: psychic-update-skill
description: Update the psychic-skill to the latest version from GitHub
argument-hint: ''
---

# Upgrade psychic-skill

Run the following commands to upgrade psychic-skill to the latest version:

## Claude Code personal install (`~/.claude/skills/psychic-skill`)

```bash
cd ~/.claude/skills/psychic-skill && git fetch origin && git reset --hard origin/main && ./setup
```

## Claude Code project install (`.claude/skills/psychic-skill`)

If the current project also has psychic-skill installed at `.claude/skills/psychic-skill`, update it too:

```bash
rm -rf .claude/skills/psychic-skill && cp -Rf ~/.claude/skills/psychic-skill .claude/skills/psychic-skill && rm -rf .claude/skills/psychic-skill/.git && cd .claude/skills/psychic-skill && ./setup
```

## Codex personal install (`${CODEX_HOME:-~/.codex}/skills/psychic-skill`)

```bash
cd "${CODEX_HOME:-$HOME/.codex}/skills/psychic-skill" && git fetch origin && git reset --hard origin/main && ./setup
```

## Codex project install (`.codex/skills/psychic-skill`)

If the current project also has psychic-skill installed at `.codex/skills/psychic-skill`, update it too:

```bash
rm -rf .codex/skills/psychic-skill && cp -Rf "${CODEX_HOME:-$HOME/.codex}/skills/psychic-skill" .codex/skills/psychic-skill && rm -rf .codex/skills/psychic-skill/.git && cd .codex/skills/psychic-skill && ./setup
```

After upgrading, tell the user what version was installed by running the first command that succeeds:

```bash
cat ~/.claude/skills/psychic-skill/VERSION 2>/dev/null || cat "${CODEX_HOME:-$HOME/.codex}/skills/psychic-skill/VERSION"
```
