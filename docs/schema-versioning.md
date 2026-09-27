# Schema Versioning

The database schema is versioned through Prisma migrations, tracked as files
under `prisma/migrations/`. This document explains how those versions relate
to application releases and what to do when two migrations conflict.

## How migrations are versioned

Each migration is a directory named `<timestamp>_<description>` (for example
`prisma/migrations/20260828120000_indexed_events`), containing a single
`migration.sql` file with the SQL to apply. The timestamp prefix is the
version: migrations are applied in ascending timestamp order, and Prisma
records each one it has applied — by name and a checksum of its SQL — in a
`_prisma_migrations` table in the target database. There is no separate,
hand-maintained schema version number; the ordered set of migration
directories **is** the version history, and `prisma/schema.prisma` describes
the schema that results from applying all of them in order.

`prisma/migrations/migration_lock.toml` pins the database provider
(`postgresql`) so a migration written for one provider can't be silently
applied against another.

## How schema versions relate to releases

Migrations are applied with `prisma migrate deploy`, not `prisma migrate dev`
(that command is for local development only — see `npm run prisma:migrate` in
`package.json`). `migrate deploy` applies whatever migrations exist in the
`prisma/migrations/` directory that the target database hasn't recorded yet,
in order, and does nothing if there are none pending.

This ties the schema version directly to what's checked out or deployed:

- **In the Docker image**, the `prisma/` directory — migrations included — is
  copied into the runtime image at build time (see `Dockerfile`). The image
  therefore carries a fixed, immutable snapshot of the schema history it
  expects.
- **On container start**, `docker-compose.yml` runs `npx prisma migrate
  deploy && node dist/index.js`: every deployment applies any migrations not
  yet recorded against that database before the service starts. A release is
  never "ahead" of its own schema, because it brings its migrations with it
  and applies them itself.
- **Locally**, `scripts/dev.sh` does the same thing before starting the dev
  server.

In practice this means: **a migration ships in the same commit/PR as the code
that depends on it**, not as a separate, independently-versioned artifact.
Checking out any given commit and running `prisma migrate deploy` against a
correctly-versioned database always produces the schema that commit's code
expects. `CHANGELOG.md` and the `package.json` version follow [Semantic
Versioning](https://semver.org/) for human-facing release notes, but they are
not programmatically linked to migration names — a release that changes the
schema should call that out in its changelog entry, but the actual source of
truth for what schema a given release needs is its `prisma/migrations/`
directory at that commit.

## What to do when two migrations conflict

Two contributors branching from the same `main` and each adding a migration
on their own branch is the common case. Because filenames are timestamped,
merging both is safe as long as neither migration assumes the schema state the
other one introduces — Prisma applies them in filename order, so whichever
sorts later runs second.

**If `prisma migrate dev` reports drift after a merge** (the local dev
database's history doesn't match the migrations folder), reset the local
database rather than editing migration files:

```bash
npx prisma migrate reset
```

This drops the local database and replays every migration from scratch in
order, which is safe for a local dev database but must never be run against a
shared or production database.

**If two merged migrations conflict outright** — for example, both add a
column with the same name to the same table — the later one fails with a SQL
error when applied. Fix this with a new, corrective migration that adjusts the
result; do not edit a migration file that has already been merged into `main`.
Once a migration has shipped, its SQL and its recorded checksum in
`_prisma_migrations` must match forever, in every environment that has already
applied it — editing it after the fact causes `migrate deploy` to refuse to
run there.

**If a shared environment's migration history and actual schema fall out of
sync** — most often because someone applied a change by hand, or a `migrate
deploy` run failed partway through — use `prisma migrate resolve` to tell
Prisma what actually happened before retrying:

```bash
# The migration's SQL was already applied (e.g. by hand); mark it as such
# without Prisma re-running it.
npx prisma migrate resolve --applied <migration-name>

# The migration was not applied and should be treated as if it never ran.
npx prisma migrate resolve --rolled-back <migration-name>
```

After resolving, `prisma migrate deploy` can run normally again.

## Related documentation

- [Database Schema Overview](database-schema.md) — the models and fields the current migration history produces.
- [CONTRIBUTING.md](../CONTRIBUTING.md) — required checks and commit conventions for a pull request that adds a migration.
