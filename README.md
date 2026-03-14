# psychic-skill

**A comprehensive Claude Code skill for developing applications with [Dream ORM](https://github.com/rVOHealth/dream) and [Psychic web framework](https://github.com/rVOHealth/psychic).**

This skill gives Claude deep knowledge of Dream and Psychic patterns, conventions, and best practices. It auto-loads when working on Dream/Psychic projects and covers the full development lifecycle.

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

**Requirements:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Git](https://git-scm.com/)

### Step 1: Install on your machine

Open Claude Code and paste this:

> Install psychic-skill: run `git clone https://github.com/daniel-nelson/psychic-skill.git ~/.claude/skills/psychic-skill && cd ~/.claude/skills/psychic-skill && ./setup`

### Step 2: Add to your project so teammates get it (optional)

> Add psychic-skill to this project: run `cp -Rf ~/.claude/skills/psychic-skill .claude/skills/psychic-skill && rm -rf .claude/skills/psychic-skill/.git && cd .claude/skills/psychic-skill && ./setup`

Real files get committed to your repo (not a submodule), so `git clone` just works. Teammates just need to run `cd .claude/skills/psychic-skill && ./setup` once.

### What gets installed

- Skill files (Markdown) in `~/.claude/skills/psychic-skill/` (or `.claude/skills/psychic-skill/` for project installs)
- `dream-psychic` skill auto-loads when working with Dream/Psychic projects
- `/psychic-update-skill` slash command (symlinked) for easy upgrades

Everything lives inside `.claude/`. Nothing touches your PATH or runs in the background.

## How it works

The `dream-psychic` skill has `user-invocable: false` — Claude automatically loads it when it detects you're working with Dream or Psychic (imports from `@rvoh/dream`, `@rvoh/psychic`, `pnpm psy` commands, etc.). You don't need to invoke it manually.

The skill enforces critical conventions:
- Always run `pnpm psy <command> --help` before using generators
- Never use JavaScript `Date` — use `DateTime` / `CalendarDate` from Dream
- Never mock Dream internals in tests — use factories
- BDD approach: write failing spec first, then implement
- Always run `pnpm psy sync` after changing associations

## Upgrading

Paste this into Claude Code:

> /psychic-update-skill

Or manually:

> Update psychic-skill: run `cd ~/.claude/skills/psychic-skill && git fetch origin && git reset --hard origin/main && ./setup`. If this project also has psychic-skill at `.claude/skills/psychic-skill`, update it too: run `rm -rf .claude/skills/psychic-skill && cp -Rf ~/.claude/skills/psychic-skill .claude/skills/psychic-skill && rm -rf .claude/skills/psychic-skill/.git && cd .claude/skills/psychic-skill && ./setup`

## Uninstalling

Paste this into Claude Code:

> Uninstall psychic-skill: run `rm -f ~/.claude/skills/psychic-update-skill && rm -rf ~/.claude/skills/psychic-skill`. If this project also has psychic-skill at `.claude/skills/psychic-skill`, remove it too: run `rm -f .claude/skills/psychic-update-skill && rm -rf .claude/skills/psychic-skill`

## Skill files

| File | Content |
|------|---------|
| `SKILL.md` | Main skill — critical rules, project structure, quick references for all topics |
| `models.md` | Dream models, associations, hooks, validations, scopes, operators, decorators |
| `controllers.md` | Psychic controllers, auth, CRUD, OpenAPI, parameters |
| `serializers.md` | DreamSerializer / ObjectSerializer, composition, STI serializers, passthrough |
| `sti.md` | Complete STI guide — generation, models, serializers, migrations, controllers, testing |
| `migrations.md` | Kysely migrations, column types, enums, indexes, helpers |
| `testing.md` | Factories, model/controller specs, BDD patterns |
| `workers-websockets.md` | Background jobs, real-time communication |

## License

MIT
