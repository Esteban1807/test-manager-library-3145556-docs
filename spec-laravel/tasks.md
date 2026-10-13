# Tasks — Manager Library (Laravel)

Each task is finished when its **evidence** exists and can be **executed with a command or an automated test** (Article XII).

The tasks follow the order of [`spec-python/tasks.md`](../spec-python/tasks.md). The inconsistencies found in `docs` are corrected here directly, without touching Python. The corrections are listed at the end.

| # | Task | Depends on | Executable evidence |
|---|---|---|---|
| T-01 | Laravel skeleton and `GET /api/v1/health` | — | `php artisan test --filter=HealthTest` passes: answers `200` with `{"status":"ok"}` |
| T-02 | Initial migration: six tables, FKs, unique indexes, `version` on `book`, five seeded categories | T-01 | `php artisan test --filter=SchemaTest` queries `information_schema` on PostgreSQL and checks the tables of `data-model.md` §3, with no `DEFAULT` |
| T-03 | Domain: `Isbn`, `Quantity`, `DocumentNumber`, `DateRange` (value objects) | T-01 | `php artisan test --testsuite=Unit` passes: 13-digit ISBN, quantity > 0, end ≥ start, document not empty |
| T-04 | Domain: `Book` (`withdraw`, `restock`, logical deletion, `version`) | T-03 | Unit tests: available never < 0 nor > total |
| T-05 | Domain: `Loan` (`addItem`, `returnItem`, calculated status, 14 days, max. 3) | T-04 | Tests with a fixed "now": `open`, `overdue`, `closed` |
| T-06 | Domain: `Member` and `User` (name normalization, valid role) | T-03 | Tests: `"Ana "` becomes `ana`; role `admin` rejected through the API |
| T-07 | Ports and use cases for **authentication** (`LoginService`, `CreateLibrarianService`) with doubles | T-06 | HU-01 and HU-02 pass with `php artisan test --filter=AuthUseCaseTest` |
| T-08 | Ports and use cases for **books** (create, search, edit, retire, cover) with doubles | T-04 | HU-03 and HU-04 pass with `php artisan test --filter=BookUseCaseTest` |
| T-09 | **Lend** use case (`LendBooksService`) with doubles | T-05, T-08 | HU-06 complete: limits, overdue, no copies |
| T-10 | **Return** and **overdue** use cases | T-09 | HU-07 and HU-09; returning twice answers `ITEM_ALREADY_RETURNED` |
| T-11 | **Most-borrowed report** use case and its read port | T-09 | HU-10; the old title is kept after renaming (ADR-004) |
| T-12 | Persistence adapter: book, member and user repositories (Eloquent) | T-02, T-08 | `php artisan test --testsuite=Integration` passes against a real PostgreSQL |
| T-13 | Persistence adapter: loan repository + `version` token | T-12, T-09 | `php artisan test --filter=ConcurrencyTest`: two loans of the last copy; the second is refused (RNF-2) |
| T-14 | Security adapter: Argon2id `Hash` and JWT | T-07 | `php artisan test --filter=SecretsTest`: the password appears in no response and in no log line (RNF-3) |
| T-15 | Cover adapter (disk) | T-08 | `php artisan test --filter=CoverHttpTest`: a 2 MB PNG is stored; 3 MB answers `422`; without a cover, `cover_key` is `null` |
| T-16 | HTTP adapter: routes of `api-contract.md`, single error body, authorization by role | T-10, T-11, T-14, T-15 | `php artisan test --testsuite=Contract` walks every protected route and the error codes |
| T-17 | Composition and startup: command `library:bootstrap-admin` from the environment | T-16 | `php artisan test --filter=AdminLoginTest` passes |
| T-18 | Move the **domain-only** invariants into the database: `ck_book_total_positive`, `ck_user_role` | T-12 | `php artisan test --filter=DomainOnlyChecksTest`: inserting a total of 0 or an invalid role throws `QueryException` |
| T-19 | Architecture tests (Articles I, II and III) with Deptrac | T-17 | `vendor/bin/deptrac analyse` ends with code 0; a forbidden import added on purpose makes it fail |
| T-20 | `infra`: `docker compose up` starts an empty engine, the API and the frontend | T-17 | `docker compose up -d db` and then `\dt library.*` in `psql` lists no tables until the API runs the migrations |
| T-21 | `tool`: demo seeder **through the API** (≥ 40 books, 10 members, 15 loans, 3 overdue) | T-17 | After `php seed.php`, `GET /api/v1/loans/overdue` returns 3 lines and the report has rows |
| T-22 | `app`: login, catalog, members, lend, return, overdue, report | T-16 | `npm run test:e2e` (Playwright) walks every flow and passes |
| T-23 | `page`: static public presentation site | — | `npm test` (Playwright) without the API: loads at 375 px and 1280 px, with no horizontal scroll |
| T-24 | Search performance (RNF-1) | T-21 | `php seed.php --books=5000` and `php bench.php`: p95 under 300 ms over 20 searches |

## Definition of done for the test

Whoever reaches T-17 with evidence has the **core** done. T-18 to T-24 are what separates good work from outstanding work. Performance with SDD is evaluated, not the number of tasks: one task with executable evidence is worth more than three half-done.

## Corrections with respect to `spec-python`

These corrections live only in Laravel. `spec-python/` is not modified.

| Inconsistency in `docs` | Correction in `spec-laravel` |
|---|---|
| ADR-002 requires a version token; `data-model.md` does not have one | `version` on `book` (T-02) and concurrency test (T-13). See D-04 in `plan.md` |
| RNF-1 with no task and no evidence | New task T-24 |
| Manual evidence ("walkthrough at the fair", "correct on mobile and desktop") | Replaced by Playwright tests (T-22, T-23) |
| `DocumentNumber` in the plan but not in the tasks | Included in T-03 and T-06 |
| `GET /books/{id}/cover` mentioned but not listed | Included in the contract test of T-16 |
| No error code for a missing or expired token | T-16 uses `UNAUTHENTICATED` (401). See [`api-desviaciones.md`](api-desviaciones.md) D-API-01 |

## Status and executed evidence

Verified on 13 October 2026. Each command was run and finished successfully.

| Tasks | Repository | Command | Result |
|---|---|---|---|
| T-01 to T-18 | `api` | `php artisan test` | 116 tests, 206 assertions |
| T-19 | `api` | `vendor/bin/deptrac analyse` | 0 violations |
| T-20 | `infra` | `docker compose up --build` | The database starts with no tables; once the API starts there are 6 tables; health, frontend and login answer 200; the seeder runs against the containers |
| T-21 | `tool` | `php seed.php` | 40 books, 10 members, 15 loans, 3 overdue lines, 12 rows in the report |
| T-22 | `app` | `npm run test:e2e` | 3 tests |
| T-23 | `page` | `npm test` | 2 tests (375 px and 1280 px) |
| T-24 | `tool` | `php seed.php --books=5000` and `php bench.php` | p95 of 203 ms with 5,000 books |
