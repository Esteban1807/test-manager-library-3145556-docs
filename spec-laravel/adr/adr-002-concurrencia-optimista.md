# ADR-002 · Concurrencia optimista sobre los ejemplares

- **Estado:** aceptada (adaptada de [`spec-python`](../../spec-python/adr/adr-002-concurrencia-optimista.md))
- **Contexto:** dos bibliotecarios pueden prestar el **último ejemplar** del mismo libro al mismo tiempo (RNF-2). Si ambos leen
  `available_copies = 1` y ambos escriben, el sistema presta un libro que no tiene.
- **Decisión:** control de concurrencia **optimista**. El libro lleva la columna `version` (**desviación:** `data-model.md` compartido no
  la tiene; ver D-04 en [`plan.md`](../plan.md)). Cada escritura hace `UPDATE library.book SET ..., version = version + 1 WHERE id = ? AND version = ?`.
  Si no afecta ninguna fila, el adaptador lanza `BookNotAvailable` y el servidor responde **409 `BOOK_NOT_AVAILABLE`**.
  La **última barrera** es el `CHECK` `ck_book_available_range` (`0 <= available_copies <= total_copies`).
- **Alternativas descartadas:** bloqueo pesimista de la fila (`lockForUpdate()`) en cada préstamo: serializa toda la biblioteca por
  un caso raro; contador en memoria: no sobrevive a más de un proceso.
- **Consecuencias:** si el `CHECK` salta, **algo escribió fuera del adaptador**: es un defecto, no un caso de negocio. Devolver dos veces
  la misma línea no suma dos veces porque `returned_at` ya está lleno (CA-07.2).
- **Cómo se verifica:** `php artisan test` con dos peticiones simultáneas sobre un libro con un ejemplar (T-13): exactamente una recibe `201`.
