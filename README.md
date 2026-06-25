# psychic-skill

**A comprehensive AI coding assistant skill for developing applications with [Dream ORM](https://github.com/rVOHealth/dream) and [Psychic web framework](https://github.com/rVOHealth/psychic).**

This skill gives Claude Code and Codex deep knowledge of Dream and Psychic patterns, conventions, and best practices. It auto-loads when working on Dream/Psychic projects and covers the full development lifecycle.

## What it covers

| Topic | Coverage |
|-------|---------|
| **Models** | Associations (BelongsTo, HasMany, HasOne, through, polymorphic), hooks, validations, scopes, query operators, soft deletes, encrypted columns, virtual attributes, dirty tracking, transactions |
| **Controllers** | Controller hierarchy, authentication patterns, nested resource bases, CRUD patterns, `@OpenAPI` decorator, `@BeforeAction`, parameter handling (`castParam`, `extractParams`), response methods |
| **Serializers** | `DreamSerializer` / `ObjectSerializer`, composition pattern, `.attribute` / `.customAttribute` / `.delegatedAttribute` / `.rendersOne` / `.rendersMany`, passthrough context, `preloadFor` integration |
| **STI** | Full lifecycle: generating parent with `--sti-base-serializer`, generating children with `g:sti-child`, generic base serializers, type discrimination, controller switch pattern, migrations with check constraints |
| **Migrations** | Kysely migrations, all column types, enums, enum arrays, foreign keys, indexes, `DreamMigrationHelpers`, extensions (citext, pg_trgm) |
| **Routing** | `r.resources`, `r.resource`, `r.namespace`, `r.scope`, nested resources, collection routes |
| **Testing** | Factories, model specs, controller specs with `OpenapiSpecRequest`, authentication helpers, BDD conventions |
| **Workers** | Background jobs (BullMQ), priorities, scheduled/cron jobs, `AfterCommit` hook patterns |
| **Websockets** | `Ws` emitter, `Cable` server, Socket.IO, Redis pub/sub, user registration |
| **Code Generation** | `g:model`, `g:resource`, `g:controller`, `g:migration`, `g:sti-child` with full flag reference |

## Install

### Install via your agent

Requires [Git](https://git-scm.com/).

Open your agent and paste the install prompt. Then optionally add to your project so teammates get the skill too.

#### Claude Code

**1. Install on your machine** — open Claude Code and paste:

```
Install psychic-skill: run `git clone https://github.com/daniel-nelson/psychic-skill.git ~/.claude/skills/psychic-skill && cd ~/.claude/skills/psychic-skill && ./setup`
```

**2. Add to your project (optional):**

```
Add psychic-skill to this project: run `cp -Rf ~/.claude/skills/psychic-skill .claude/skills/psychic-skill && rm -rf .claude/skills/psychic-skill/.git && cd .claude/skills/psychic-skill && ./setup`
```

Real files get committed to your repo (not a submodule), so `git clone` just works. Teammates just need to run `cd .claude/skills/psychic-skill && ./setup` once.

#### Codex

**1. Install on your machine** — open Codex and paste:

```
Install psychic-skill: run `git clone https://github.com/daniel-nelson/psychic-skill.git ~/.agents/skills/psychic-skill && cd ~/.agents/skills/psychic-skill && ./setup`
```

**2. Add to your project (optional):**

```
Add psychic-skill to this project: run `cp -Rf ~/.agents/skills/psychic-skill .agents/skills/psychic-skill && rm -rf .agents/skills/psychic-skill/.git && cd .agents/skills/psychic-skill && ./setup`
```

Real files get committed to your repo (not a submodule), so teammates just need to run `cd .agents/skills/psychic-skill && ./setup` once.

### Install via `npx skills`

```bash
npx skills add daniel-nelson/psychic-skill
```

It will prompt you for which agent to install to. For more details, see [vercel-labs/skills](https://github.com/vercel-labs/skills).

### For agents that don't auto-load skills

Cursor, Aider, Cline, and similar tools don't auto-load skills, but they can still use psychic-skill as **prescribed reading**. Clone the repo somewhere your agent reads files:

```bash
git clone https://github.com/daniel-nelson/psychic-skill.git .ai/psychic-skill
```

Then instruct your agent (via its rules file — `.cursorrules`, `.aider.conf.yml`, `AGENTS.md`, etc.) to read `SKILL.md` and the linked topic files (`models.md`, `generators.md`, `controllers.md`, `serializers.md`, `sti.md`, `migrations.md`, `querying.md`, `testing.md`, `workers.md`, `websockets.md`) as required reading before making changes. The files are plain markdown — they work as documentation for any agent that can read files.

To stay current, run `cd .ai/psychic-skill && git fetch origin && git reset --hard origin/main && ./setup` periodically. This is what `/psychic-update-skill` does under the hood for Claude Code and Codex.

### What gets installed

- Skill files (Markdown) in `~/.claude/skills/psychic-skill/` (Claude Code) or `~/.agents/skills/psychic-skill/` (Codex), with optional project installs at `.claude/skills/psychic-skill/` or `.agents/skills/psychic-skill/`
- `psychic-skill` skill auto-loads when working with Dream/Psychic projects
- `/psychic-update-skill` slash command (symlinked) for easy upgrades

Everything lives inside your assistant's skills directory. Nothing touches your PATH or runs in the background.

## How it works

The `psychic-skill` skill has `user-invocable: false` and auto-load triggers. Claude Code and Codex should automatically load it when they detect Dream/Psychic work (imports from `@rvoh/dream`, `@rvoh/psychic`, `pnpm psy` commands, etc.). You don't need to invoke it manually.

Skill metadata note: keep `SKILL.md` front matter in the same minimal format used by the working installed skills. Do not add one-off fields such as `argument-hint`, and do not switch `allowed-tools` to a multiline YAML list unless the loader is known to accept it. The safer pattern here is simple scalar fields only.

The skill enforces critical conventions:
- Always run `pnpm psy <command> --help` before using generators
- Never use JavaScript `Date` — use `DateTime`, `CalendarDate`, `ClockTime`, or `ClockTimeTz` from Dream
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
cd ~/.agents/skills/psychic-skill && git fetch origin && git reset --hard origin/main && ./setup

# Codex project install (optional)
rm -rf .agents/skills/psychic-skill
cp -Rf ~/.agents/skills/psychic-skill .agents/skills/psychic-skill
rm -rf .agents/skills/psychic-skill/.git
cd .agents/skills/psychic-skill && ./setup
```

## Uninstalling

```bash
# Claude Code
rm -f ~/.claude/skills/psychic-update-skill
rm -rf ~/.claude/skills/psychic-skill
rm -f .claude/skills/psychic-update-skill
rm -rf .claude/skills/psychic-skill

# Codex
rm -f ~/.agents/skills/psychic-update-skill
rm -rf ~/.agents/skills/psychic-skill
rm -f .agents/skills/psychic-update-skill
rm -rf .agents/skills/psychic-skill
```

## Skill files

| File | Content |
|------|---------|
| `SKILL.md` | Main skill — always-on critical rules, project structure, key commands, and a decision map pointing to the topic files below for each task |
| `models.md` | Dream models, organization/namespacing, associations, hooks, validations, scopes, operators, decorators |
| `generators.md` | Scaffolding generators (`g:resource`/`g:model`/`g:sti-child`/`g:migration`) — decision tree, arguments, post-gen workflow, adding properties |
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
