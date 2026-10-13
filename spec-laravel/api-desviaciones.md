# Deviations and clarifications of the API contract — Laravel

The HTTP contract is governed by [`spec-python/api-contract.md`](../spec-python/api-contract.md), which remains the shared source. This file records what **Laravel does differently or clarifies**, without modifying the shared document.

## D-API-01 · Error code for a missing, invalid or expired token

- **Problem:** `api-contract.md` §2 defines `INVALID_CREDENTIALS` (401), but only for login. There is no code for a 401 caused by the token.
- **Decision:** add `UNAUTHENTICATED` with status **401**. It is used when the token is missing, invalid or expired.
- **Format:** the same single error body of §1, with `code: "UNAUTHENTICATED"` and `message` in Spanish.
- **Where it is verified:** T-16 (contract test).

## D-API-02 · Endpoint to read the cover

- **Problem:** `api-contract.md` §3.4 mentions `GET /books/{id}/cover` in the text about covers, but it does not appear in the endpoint table.
- **Decision:** it is added to the book endpoints for any **authenticated** role. It returns the cover binary or `404`.
- **Where it is verified:** T-16 (the contract test walks every route).

## D-API-03 · Prefix of the health route

- **Problem:** `api-contract.md` §3.1 writes `GET /health` without a prefix, but §1 says the base of every route is `/api/v1`.
- **Decision:** the real route is **`GET /api/v1/health`**. The general rule of the base applies.
- **Where it is verified:** T-01.

## Implementation rule · Route order

Literal routes must be registered **before** the ones with a parameter. Otherwise `GET /api/v1/loans/overdue` could be captured as `GET /api/v1/loans/{id}`.

- **Where it is verified:** T-16 (contract test with `/loans/overdue`).

## D-API-04 · Demo clock

- **Problem:** T-21 asks the seeder to create 3 overdue loans **through the API**. The due date is always `loaned_at + 14 days` and `loaned_at` is set by the server clock, so a loan that is already overdue cannot be created through the API.
- **Decision:** the `SystemClock` adapter accepts the `X-Demo-Now` header (ISO 8601 instant) as the "now" of that request, **only** if the environment variable `LIBRARY_DEMO_CLOCK` is `true`. It is `false` by default and the header is ignored.
- **Scope:** it lives in the adapter of the `Clock` port. The domain and the application do not change. It must not be enabled in production.
- **Where it is verified:** `php artisan test --filter=DemoClockTest` and running `php seed.php` in the `tool` repository.

## D-API-05 · An identifier that is not a UUID

- **Problem:** the contract says identifiers are UUIDs, but it does not say what to answer if the route carries something else (`/books/abc`). Left untreated, the engine rejects the value and the API answered 500.
- **Decision:** route parameters only accept UUIDs. Any other value is a route that does not exist: **404 `NOT_FOUND`**. In a request body, a malformed identifier or a category that does not exist is **422 `VALIDATION_ERROR`**.
- **Where it is verified:** `php artisan test --testsuite=Contract`.
