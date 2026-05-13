# Changelog

## 0.34.0 — 2026-05-12

### Changed

- **`controllers.md`** — rewrote the "Generator output" subsection to reflect that `psy g:resource` (and related generators) now emit a shared `paramSafeColumns` const at the top of the controller file referenced from every `create` / `update` action. Admin and non-admin scaffolds emit the same shape. Replaces the prior "materialize the full safe-list into the array at each call site, delete what doesn't belong" framing.
- **`deploying.md`** — Postgres TLS section updated for the framework's narrowed `SingleDbCredential.ssl` type (`TlsConnectionOptions | false`; bare `ssl: true` is no longer accepted) and the new `MissingDbSslDirective` throw at `app.set('db', ...)` time when neither `ssl` nor `useSsl` is set. Added the boilerplate `DB_NO_SSL` default, a managed-provider matrix (Supabase / Neon / Render / Azure Flexible Server work with the verified-TLS default; AWS RDS and GCP Cloud SQL need a CA bundle; Heroku Hobby is the `rejectUnauthorized: false` path), and reframed the `useSsl` paragraph as a migration note for apps scaffolded before this change rather than as new-app guidance.

### Added

- **`migrations.md`** — guardrail on the JSON column section: reaching for `json` or `jsonb` should be a stop-and-reconsider moment because Dream's column-type inference and Psychic's auto-derived OpenAPI shapes short-circuit on schemaless blobs. Calls out the relational alternatives (HasMany / BelongsTo for repeated keyed data, real columns for bounded attributes), narrows the legitimate use cases (verbatim third-party payloads, opaque per-tenant config, audit snapshots), and asks authors to document in the migration why the relational alternative was rejected.
