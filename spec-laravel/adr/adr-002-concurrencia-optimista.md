# ADR-002 · Optimistic concurrency on copies

- **Status:** accepted (adapted from [`spec-python`](../../spec-python/adr/adr-002-concurrencia-optimista.md))
- **Context:** two librarians can lend the **last copy** of the same book at the same time (RNF-2). If both read `available_copies = 1` and both write, the system lends a book it does not have.
- **Decision:** **optimistic** concurrency control. The book carries the `version` column (**deviation:** the shared `data-model.md` does not have it; see D-04 in [`plan.md`](../plan.md)). Each write runs `UPDATE library.book SET ... WHERE id = ? AND version = ?` and stores the new version. If no row is affected, the adapter throws `BookNotAvailable` and the server answers **409 `BOOK_NOT_AVAILABLE`**. The **last barrier** is the `CHECK` `ck_book_available_range` (`0 <= available_copies <= total_copies`).
- **Discarded alternatives:** pessimistic row lock (`lockForUpdate()`) on every loan: it serializes the whole library for a rare case; an in-memory counter: it does not survive more than one process.
- **Consequences:** if the `CHECK` fires, **something wrote outside the adapter**: it is a defect, not a business case. Returning the same line twice does not add twice, because `returned_at` is already set (CA-07.2).
- **How it is verified:** `php artisan test --filter=ConcurrencyTest` (T-13): two loans of a book with one copy, both read before either is saved; exactly one succeeds.
