# Technical plan — Manager Library (Laravel)

How to build what [`spec-python/spec.md`](../spec-python/spec.md), [`spec-python/api-contract.md`](../spec-python/api-contract.md) and [`spec-python/data-model.md`](../spec-python/data-model.md) say, applying [`constitution.md`](constitution.md). This plan replaces the decisions of `spec-python/plan.md` with their Laravel equivalents.

Rows marked **⚠ deviation** change something in the shared document. That correction lives **only in Laravel**; `spec-python/` is not touched.

## 1. Technical decisions

| # | Decision | Choice (Laravel) | Why |
|---|---|---|---|
| D-01 | Language and runtime | PHP 8.3 or higher | Minimum version that supports the `readonly` classes and native enums of the domain |
| D-02 | HTTP framework | Laravel 13. Routes in `routes/api.php`, controllers in `app/Adapters/Http/`, validation in the controllers | Contract in `api-contract.md` |
| D-03 | Persistence | Eloquent in `app/Adapters/Persistence/` on PostgreSQL 16. Laravel migrations in `database/migrations/` | The schema belongs to the migrations (Article V) |
| D-04 | Concurrency | **⚠ deviation.** Version token: column `version integer not null` on `book`, incremented on every write, plus the `CHECK` `ck_book_available_range` | ADR-002 requires a version token, but `data-model.md` §3 does not have one. See §4 |
| D-05 | Dates | The domain receives `DateTimeImmutable`. The clock is the `Clock` port | Test due dates without waiting 14 days |
| D-06 | Report | One aggregate query in a read adapter, behind a read port | [ADR-004](adr/adr-004-reporte-agregado-y-congelado.md): stable and cheap |
| D-07 | Value objects | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange`: `readonly` domain classes, with no table of their own | No identity. **⚠ deviation:** includes `DocumentNumber`, which the Python plan listed and `tasks.md` did not cover |
| D-08 | Covers | Laravel `Storage` behind the `CoverStorage` port; local disk in development, S3 in production. The database keeps only `cover_key` | Opaque key, never a path |
| D-09 | Passwords | `Hash::driver('argon2id')` behind the `PasswordHasher` port; the domain never sees the clear password | Article IX |
| D-10 | Startup | Artisan command `library:bootstrap-admin`, which creates the admin from `ADMIN_USERNAME` / `ADMIN_PASSWORD`. It runs when the container starts | The database cannot produce the hash |
| D-11 | Authentication | JWT HS256, 8 hours, key in `JWT_SECRET`, behind the `TokenIssuer` port. Library: `firebase/php-jwt` | Simple and stateless |
| D-12 | Tests | PHPUnit (included in Laravel). Domain and application tests without the framework; integration against a real PostgreSQL | Test pyramid |
| D-13 | Layer rules | **Deptrac**, with `deptrac.yaml` at the root of the `api` repo | Replaces import-linter (Python) and NetArchTest (.NET) |

## 2. Configuration by environment

| Variable | Required | Example |
|---|---|---|
| `DB_CONNECTION` | yes | `pgsql` |
| `DB_HOST` / `DB_PORT` / `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD` | yes | `db` / `5432` / `library` / `library` / *(in `.env`)* |
| `JWT_SECRET` | yes | long random string |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | yes | defined in `.env`, never in the repo |
| `CORS_ALLOWED_ORIGINS` | yes | `http://localhost:5173` |
| `LIBRARY_DEMO_CLOCK` | no | `false` (default). See D-API-04 |

## 3. Test strategy

| Level | What it tests | Against what | Tool |
|---|---|---|---|
| Domain | Invariants: maximum of 3, due date at 14 days, available copies in range | Nothing: pure objects | PHPUnit |
| Application | Use cases: lend, return, report | Doubles of the ports | PHPUnit |
| Integration | Schema, constraints and concurrency | Real PostgreSQL | PHPUnit |
| Contract | Single error body and status codes | The API | PHPUnit (feature tests) |
| Architecture | Articles I, II and III | `deptrac.yaml` | `vendor/bin/deptrac analyse` |
| Concurrency | Two loans of the last copy | Real PostgreSQL | PHPUnit |

## 4. Correction of the version token (D-04)

ADR-002 and the Python plan require a version token. The shared `data-model.md` does not have one. In Laravel it is added like this:

- `book.version`: `integer`, not null, **without** a default value in the migration (the domain sets it to `0` when the book is created, like every other value).
- Every write on the book uses `UPDATE ... WHERE id = ? AND version = ?` and stores the new version. If no row is affected, the operation answers `409 BOOK_NOT_AVAILABLE`.
- This column is **not** listed in `data-model.md`. It is recorded here as a deviation.

## 5. Build order

Skeleton and health → initial migration with seed → domain → use cases with doubles → adapters → API → contract and concurrency tests.
The detail, with executable evidence, is in [`tasks.md`](tasks.md).
