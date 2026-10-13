# ADR-004 · The report is computed in the engine and groups by frozen values

- **Status:** accepted (adapted from [`spec-python`](../../spec-python/adr/adr-004-reporte-agregado-y-congelado.md))
- **Context:** the most-borrowed report (HU-10) joins `loan` and `loan_item` over a range. If it read the title or the category from the live catalog, renaming a book or a category would change the report of a period that is already closed.
- **Decision:** (1) the report **is not persisted**: it is computed with a single aggregate query in the read adapter `app/Adapters/Persistence/`, behind the `ReportReader` port. (2) It groups by `book_id, book_title, category_name` **as they were frozen on the line**. (3) `loan_item.category_name` **has no FK** on purpose. (4) If a book changed category within the range, the report shows **two rows**: it does not pick a winner.
- **Discarded alternatives:** joining with the live catalog (the report stops being stable); taking "the most recent value of the range" (a new loan would change what was already read); materializing a report table (a second source of truth).
- **Consequences:** CA-10.1 says "one row per book and frozen category name", not "one row per book". The report includes no members and no librarians (privacy).
- **How it is verified:** `php artisan test --filter=ReportTest` (T-11): lend a book, rename it, ask for the report of the same range; the title in the report is the old one.
