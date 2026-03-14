---
name: psychic-upgrade
description: Upgrade the psychic-skill to the latest version from GitHub
argument-hint: ''
---

# Upgrade psychic-skill

Run the following commands to upgrade psychic-skill to the latest version:

## Personal install (~/.claude/skills/psychic-skill)

```bash
cd ~/.claude/skills/psychic-skill && git fetch origin && git reset --hard origin/main && ./setup
```

## Project install (.claude/skills/psychic-skill)

If the current project also has psychic-skill installed at `.claude/skills/psychic-skill`, update it too:

```bash
rm -rf .claude/skills/psychic-skill && cp -Rf ~/.claude/skills/psychic-skill .claude/skills/psychic-skill && rm -rf .claude/skills/psychic-skill/.git && cd .claude/skills/psychic-skill && ./setup
```

After upgrading, tell the user what version was installed by running:

```bash
cd ~/.claude/skills/psychic-skill && git log --oneline -1
```
