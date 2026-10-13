# ADR-001 · El esquema es propiedad de las migraciones de la aplicación

- **Estado:** aceptada (adaptada de [`spec-python`](../../spec-python/adr/adr-001-propiedad-del-esquema.md))
- **Contexto:** el repositorio `infra` levanta un motor de base de datos. Alguien podría poner el DDL en un script de arranque del
  contenedor y otro en las migraciones de Laravel; con dos dueños, nadie sabe qué esquema corre en producción.
- **Decisión:** **todo el DDL vive en las migraciones de Laravel** (`database/migrations/`): tablas, índices, restricciones y semilla de
  categorías. `infra` solo crea el motor vacío, el usuario y el volumen.
- **Alternativas descartadas:** scripts SQL en `infra/db/init/` (segunda fuente de verdad); `Schema::create` ejecutado al arrancar la
  aplicación (no deja historial ni reversa).
- **Consecuencias:** una extensión de Postgres o un índice nuevo se crea en la **misma** migración que lo usa. Un esquema cambiado a
  mano en el motor es un defecto, no una variante.
- **Cómo se verifica:** `infra` no contiene `CREATE TABLE`; `php artisan test --filter=SchemaTest` (T-02) comprueba el esquema contra `data-model.md` §3.
