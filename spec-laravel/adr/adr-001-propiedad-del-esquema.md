# ADR-001 · The schema belongs to the application's migrations

- **Status:** accepted (adapted from [`spec-python`](../../spec-python/adr/adr-001-propiedad-del-esquema.md))
- **Context:** the `infra` repository starts a database engine. Someone could put the DDL in a container startup script and someone else in the Laravel migrations; with two owners, nobody knows which schema runs in production.
- **Decision:** **all DDL lives in the Laravel migrations** (`database/migrations/`): tables, indexes, constraints and the category seed. `infra` only creates the empty engine, the user and the volume.
- **Discarded alternatives:** SQL scripts in `infra/db/init/` (a second source of truth); `Schema::create` executed when the application starts (leaves no history and no rollback).
- **Consequences:** a Postgres extension or a new index is created in the **same** migration that uses it. A schema changed by hand in the engine is a defect, not a variant.
- **How it is verified:** `infra` contains no `CREATE TABLE`; `php artisan test --filter=SchemaTest` (T-02) checks the schema against `data-model.md` §3.
