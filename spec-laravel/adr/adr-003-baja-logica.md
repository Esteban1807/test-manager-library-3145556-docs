# ADR-003 · Books are retired, not deleted

- **Status:** accepted (adapted from [`spec-python`](../../spec-python/adr/adr-003-baja-logica.md))
- **Context:** a loan line points to a book. If the book is physically deleted, the history and the report break.
- **Decision:** the book carries `deleted_at`. Retiring it sets that value. The catalog filters by `deleted_at IS NULL` in the book repository, which is the only place that queries the catalog. A book with open loans is **not** retired (409 `BOOK_HAS_OPEN_LOANS`). ISBN uniqueness is a **partial** index over active books, so a retired ISBN can be registered again.
- **Discarded alternatives:** physical deletion with cascade (destroys the history); marking `active = false` (does not say when).
- **Consequences:** `loan_item.book_id` carries a `RESTRICT` FK as the **last barrier**: a manual `DELETE` or a future code change fails loudly instead of corrupting the history. The cover binary is the only data that is physically deleted.
- **How it is verified:** querying a retired book returns `404`; a `DELETE` through `psql` on a book with lines fails because of the FK.
