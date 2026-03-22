# psychic-skill

**A comprehensive AI coding assistant skill for developing applications with [Dream ORM](https://github.com/rVOHealth/dream) and [Psychic web framework](https://github.com/rVOHealth/psychic).**

This skill gives Claude Code and Codex deep knowledge of Dream and Psychic patterns, conventions, and best practices. It auto-loads when working on Dream/Psychic projects and covers the full development lifecycle.

## What it covers

| Topic | Coverage |
|-------|---------|
| **Models** | Associations (BelongsTo, HasMany, HasOne, through, polymorphic), hooks, validations, scopes, query operators, soft deletes, encrypted columns, virtual attributes, dirty tracking, transactions |
| **Controllers** | Controller hierarchy, authentication patterns, nested resource bases, CRUD patterns, `@OpenAPI` decorator, `@BeforeAction`, parameter handling (`castParam`, `paramsFor`), response methods |
| **Serializers** | `DreamSerializer` / `ObjectSerializer`, composition pattern, `.attribute` / `.customAttribute` / `.delegatedAttribute` / `.rendersOne` / `.rendersMany`, passthrough context, `preloadFor` integration |
| **STI** | Full lifecycle: generating parent with `--sti-base-serializer`, generating children with `g:sti-child`, generic base serializers, type discrimination, controller switch pattern, migrations with check constraints |
| **Migrations** | Kysely migrations, all column types, enums, enum arrays, foreign keys, indexes, `DreamMigrationHelpers`, extensions (citext, pg_trgm) |
| **Routing** | `r.resources`, `r.resource`, `r.namespace`, `r.scope`, nested resources, collection routes |
| **Testing** | Factories, model specs, controller specs with `OpenapiSpecRequest`, authentication helpers, BDD conventions |
| **Workers** | Background jobs (BullMQ), priorities, scheduled/cron jobs, `AfterCommit` hook patterns |
| **Websockets** | `Ws` emitter, `Cable` server, Socket.IO, Redis pub/sub, user registration |
| **Code Generation** | `g:model`, `g:resource`, `g:controller`, `g:migration`, `g:sti-child` with full flag reference |

## Install

**Requirements:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or Codex, plus [Git](https://git-scm.com/)

### Claude Code: install on your machine

Open Claude Code and paste this:

> Install psychic-skill: run `git clone https://github.com/daniel-nelson/psychic-skill.git ~/.claude/skills/psychic-skill && cd ~/.claude/skills/psychic-skill && ./setup`

### Claude Code: add to your project (optional)

> Add psychic-skill to this project: run `cp -Rf ~/.claude/skills/psychic-skill .claude/skills/psychic-skill && rm -rf .claude/skills/psychic-skill/.git && cd .claude/skills/psychic-skill && ./setup`

Real files get committed to your repo (not a submodule), so `git clone` just works. Teammates just need to run `cd .claude/skills/psychic-skill && ./setup` once.

### Codex: install on your machine

Open Codex and paste this:

> Install psychic-skill: run `git clone https://github.com/daniel-nelson/psychic-skill.git ~/.codex/skills/psychic-skill && cd ~/.codex/skills/psychic-skill && ./setup`

### Codex: add to your project (optional)

> Add psychic-skill to this project: run `cp -Rf ~/.codex/skills/psychic-skill .codex/skills/psychic-skill && rm -rf .codex/skills/psychic-skill/.git && cd .codex/skills/psychic-skill && ./setup`

Real files get committed to your repo (not a submodule), so teammates just need to run `cd .codex/skills/psychic-skill && ./setup` once.

### Install with `npx skills`

You can also install `psychic-skill` with the open agent skills CLI:

```bash
npx skills add daniel-nelson/psychic-skill
```

It will prompt you for which agent to install to. For more details, see [vercel-labs/skills](https://github.com/vercel-labs/skills).

### What gets installed

- Skill files (Markdown) in `~/.claude/skills/psychic-skill/` or `~/.codex/skills/psychic-skill/` (and optional project installs at `.claude/skills/psychic-skill/` or `.codex/skills/psychic-skill/`)
- `dream-psychic` skill auto-loads when working with Dream/Psychic projects
- `/psychic-update-skill` slash command (symlinked) for easy upgrades

Everything lives inside your assistant's skills directory (`.claude` or `.codex`). Nothing touches your PATH or runs in the background.

## How it works

The `dream-psychic` skill has `user-invocable: false` and auto-load triggers. Claude Code and Codex should automatically load it when they detect Dream/Psychic work (imports from `@rvoh/dream`, `@rvoh/psychic`, `pnpm psy` commands, etc.). You don't need to invoke it manually.

The skill enforces critical conventions:
- Always run `pnpm psy <command> --help` before using generators
- Never use JavaScript `Date` — use `DateTime` / `CalendarDate` from Dream
- Never mock Dream internals in tests — use factories
- BDD approach: write failing spec first, then implement
- Always run `pnpm psy sync` after changing associations

## Upgrading

In either Claude Code or Codex, run:

> /psychic-update-skill

Or update manually:

```bash
# Claude Code personal install
cd ~/.claude/skills/psychic-skill && git fetch origin && git reset --hard origin/main && ./setup

# Claude Code project install (optional)
rm -rf .claude/skills/psychic-skill
cp -Rf ~/.claude/skills/psychic-skill .claude/skills/psychic-skill
rm -rf .claude/skills/psychic-skill/.git
cd .claude/skills/psychic-skill && ./setup

# Codex personal install
cd ~/.codex/skills/psychic-skill && git fetch origin && git reset --hard origin/main && ./setup

# Codex project install (optional)
rm -rf .codex/skills/psychic-skill
cp -Rf ~/.codex/skills/psychic-skill .codex/skills/psychic-skill
rm -rf .codex/skills/psychic-skill/.git
cd .codex/skills/psychic-skill && ./setup
```

## Uninstalling

```bash
# Claude Code
rm -f ~/.claude/skills/psychic-update-skill
rm -rf ~/.claude/skills/psychic-skill
rm -f .claude/skills/psychic-update-skill
rm -rf .claude/skills/psychic-skill

# Codex
rm -f ~/.codex/skills/psychic-update-skill
rm -rf ~/.codex/skills/psychic-skill
rm -f .codex/skills/psychic-update-skill
rm -rf .codex/skills/psychic-skill
```

## Skill files

| File | Content |
|------|---------|
| `SKILL.md` | Main skill — critical rules, project structure, quick references for all topics |
| `models.md` | Dream models, associations, hooks, validations, scopes, operators, decorators |
| `controllers.md` | Psychic controllers, auth, CRUD, OpenAPI, parameters |
| `querying.md` | Dream query patterns, `toKysely(...)`, and typed project `db()` usage |
| `serializers.md` | DreamSerializer / ObjectSerializer, composition, STI serializers, passthrough |
| `sti.md` | Complete STI guide — generation, models, serializers, migrations, controllers, testing |
| `migrations.md` | Kysely migrations, column types, enums, indexes, helpers |
| `testing.md` | Factories, model/controller specs, BDD patterns |
| `workers.md` | Background jobs (BullMQ), services, scheduling, testing |
| `websockets.md` | Real-time communication (Socket.IO, Redis pub/sub) |

## License

MIT
