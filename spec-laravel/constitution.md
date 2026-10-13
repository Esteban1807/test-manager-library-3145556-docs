# Constitution — Manager Library (Laravel)

Non-negotiable principles of the project in **PHP 8.x with Laravel** and **React**. They are the same 12 articles as [`spec-python/constitution.md`](../spec-python/constitution.md), with their verification translated to the tools of this stack. A principle nobody can check is only an intention: every article states **how it is verified**. A change that violates an article is rejected without discussing the rest.

What defines the business (stories, rules, API and data model) does not live here: it lives in [`spec-python/spec.md`](../spec-python/spec.md), [`spec-python/api-contract.md`](../spec-python/api-contract.md) and [`spec-python/data-model.md`](../spec-python/data-model.md), which are the source shared by the three stacks.

---

## Article I — The domain depends on nothing

`app/Domain/` imports no other namespace of the project or of the framework: no `Illuminate\*`, no Eloquent, no facades, no Laravel helpers, no third-party libraries. Only plain PHP: `final` and `readonly` classes, native enums, its own exceptions.

**Why:** it is the only thing that lets the invariants be tested without a database, a server or a container, and lets the ORM or the framework change without touching the rules.
**Verification:** a Deptrac rule forbids `Domain` from depending on `Application`, `Adapters`, `Bootstrap` or `Illuminate`. The analysis fails in CI if the dependency is introduced.

## Article II — The core declares its ports; the world implements them

The application layer only knows the domain. Everything it needs from outside (repositories, clock, password hashing, cover storage, token generation) is expressed as an **interface that it defines itself** in `app/Application/Ports/`. No Eloquent, Laravel or library type appears in the signature of a port.

**Verification:** a Deptrac rule prevents `Application` from importing `Adapters` or `Illuminate\Database\*`. Application tests run with test doubles of the ports and never reference `Adapters`.

## Article III — A single composition point

A single `ServiceProvider` (`app/Bootstrap/Composition/PortBindings.php`) is where a port meets its adapter. Outside of it no infrastructure is instantiated: `new` is not used on `Adapters` classes and adapters are not resolved with `app()` in the rest of the code.

**Verification:** an architecture test fails if an `Adapters` class is instantiated outside `Bootstrap/Composition/`.

## Article IV — SOLID is not decoration

| Principle | Concrete application | Verification |
|---|---|---|
| **SRP** | One use case = one class in `app/Application/UseCases/`. `LendBooksService` lends and does nothing else | A class with two reasons to change is split |
| **OCP** | Changing cover storage (disk → S3) touches neither domain nor application | The new adapter only adds one line in `PortBindings` |
| **LSP** | Test doubles replace the real adapter without changing expectations | The same test passes against the double and the real one |
| **ISP** | Small ports: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | No implementer leaves methods unused |
| **DIP** | The port is defined by whoever consumes it | Ports live in `app/Application/Ports/` |

## Article V — The schema belongs to the migrations

The `infra` repository starts an **empty** engine. All DDL (tables, indexes, constraints, extensions and seed) lives in the **Laravel migrations** (`database/migrations/`). See [ADR-001](adr/adr-001-propiedad-del-esquema.md).

**Verification:** `infra` contains no `CREATE TABLE`.

## Article VI — Invariants the database can enforce, the database enforces

Every invariant expressible in the engine (`CHECK`, `UNIQUE`, FK) is declared in the migrations, with `->unique()`, `->foreign()` or `DB::statement()` for `CHECK` constraints. What is not applied is marked **pending** in `spec-python/data-model.md`; it is not promised.

**Verification:** the queries of `data-model.md` §10 match what the migrations declare.

## Article VII — Derived values are calculated, not stored

The status of a loan (`open`, `overdue`, `closed`) and the days late are calculated in the domain. There is no column for them.

**Verification:** no migration contains a `status` or `days_late` column.

## Article VIII — Frozen values do not follow the catalog

A loan line keeps the title, ISBN and category from the moment of the loan. No report reads them from the live catalog. See [ADR-004](adr/adr-004-reporte-agregado-y-congelado.md).

**Verification:** a test that renames a book after lending it and checks that the report keeps the old title.

## Article IX — No secret enters the repository or a log

No passwords, no precomputed hashes, no token key. Every secret comes from the environment (`.env`). The real `.env` is in `.gitignore`; only `.env.example` is versioned.

**Verification:** `gitleaks` in CI, and a test that checks that no HTTP response includes `password` or `password_hash`.

## Article X — The engine wins

If the data document and the database disagree, the document is broken and the document is corrected.

## Article XI — Language

Code, columns, routes and namespaces in English. Messages shown to the user in Spanish (`lang/es/`).

> **Deviation:** `spec-python` also requires the documentation to be in Spanish. The documentation of `spec-laravel/` is written in English by decision of the project owner. Messages to the user stay in Spanish.

## Article XII — Nothing is taken as done without evidence

A task in [`tasks.md`](tasks.md) is finished when its **evidence exists and can be executed** with a command or an automated test, not when it is "already programmed" nor when someone reviewed it by hand.

**Verification:** every task in `tasks.md` has a concrete command or test that proves it.
