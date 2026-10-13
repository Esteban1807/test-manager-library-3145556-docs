# Tareas — Manager Library (Laravel)

Cada tarea termina cuando su **evidencia** existe y se puede **ejecutar con un comando o una prueba automatizada** (artículo XII). Estado inicial de todas: **pendiente**.

Las tareas siguen el orden de [`spec-python/tasks.md`](../spec-python/tasks.md) y se corrigen las incongruencias detectadas en `docs` directamente aquí, sin tocar Python. Las correcciones están listadas al final.

| # | Tarea | Depende de | Evidencia ejecutable |
|---|---|---|---|
| T-01 | Esqueleto de Laravel y `GET /api/v1/health` | — | `php artisan test --filter=HealthTest` pasa: responde `200` con `{"status":"ok"}` |
| T-02 | Migración inicial: seis tablas, FK, índices únicos, `version` en `book`, cinco categorías sembradas | T-01 | `php artisan test --filter=SchemaTest` consulta `information_schema` sobre PostgreSQL y comprueba las columnas de `data-model.md` §3, sin ningún `DEFAULT` |
| T-03 | Dominio: `Isbn`, `Quantity`, `DocumentNumber`, `DateRange` (objetos de valor) | T-01 | `php artisan test --testsuite=Unit` pasa: ISBN de 13 dígitos, cantidad > 0, fin ≥ inicio, documento no vacío |
| T-04 | Dominio: `Book` (`withdraw`, `restock`, baja lógica, `version`) | T-03 | Pruebas unitarias: disponibles nunca < 0 ni > total |
| T-05 | Dominio: `Loan` (`addItem`, `returnItem`, estado calculado, 14 días, máx. 3) | T-04 | Pruebas con un `Clock` falso: `open`, `overdue`, `closed` |
| T-06 | Dominio: `Member` y `User` (normalización de nombre, rol válido) | T-03 | Pruebas: `"Ana "` queda `ana`; rol `admin` rechazado en la API |
| T-07 | Puertos y casos de uso de **autenticación** (`LoginService`, `CreateLibrarianService`) con dobles | T-06 | HU-01 y HU-02 pasan con `php artisan test --filter=AuthUseCaseTest` |
| T-08 | Puertos y casos de uso de **libros** (crear, buscar, editar, baja, portada) con dobles | T-04 | HU-03 y HU-04 pasan con `php artisan test --filter=BookUseCaseTest` |
| T-09 | Caso de uso **prestar** (`LendBooksService`) con dobles | T-05, T-08 | HU-06 completa: límites, vencidos, sin ejemplares |
| T-10 | Caso de uso **devolver** y **vencidos** | T-09 | HU-07 y HU-09; devolver dos veces responde `ITEM_ALREADY_RETURNED` |
| T-11 | Caso de uso **reporte de más prestados** y su puerto de lectura | T-09 | HU-10; el título viejo se conserva tras renombrar (ADR-004) |
| T-12 | Adaptador de persistencia: repositorios de libro, socio, usuario (Eloquent) | T-02, T-08 | `php artisan test --testsuite=Integration` pasa contra PostgreSQL real |
| T-13 | Adaptador de persistencia: repositorio de préstamos + testigo de `version` | T-12, T-09 | Prueba concurrente: dos peticiones sobre el último ejemplar; una responde `201` y la otra `409` (RNF-2) |
| T-14 | Adaptador de seguridad: `Hash` Argon2id y JWT | T-07 | Prueba que captura el log y comprueba que la clave no aparece en ninguna línea ni en la respuesta (RNF-3) |
| T-15 | Adaptador de portadas (disco) | T-08 | Prueba: PNG de 2 MB se guarda; 3 MB responde `422`; sin portada, `cover_key` es `null` |
| T-16 | Adaptador HTTP: rutas de `api-contract.md`, cuerpo de error único, autorización por rol | T-10, T-11, T-14, T-15 | `php artisan test --testsuite=Contract` recorre todas las rutas de §3 y todos los códigos de §2 |
| T-17 | Composición y arranque: comando `library:bootstrap-admin` desde el entorno | T-16 | `php artisan migrate:fresh && php artisan library:bootstrap-admin && php artisan test --filter=AdminLoginTest` pasa |
| T-18 | Bajar a la base las invariantes **solo dominio**: `ck_book_total_positive`, `ck_user_role` | T-12 | Prueba: `DB::table('book')->insert(...)` con total 0 lanza `QueryException`; mismo caso para `role` inválido |
| T-19 | Pruebas de arquitectura (artículos I, II y III) con Deptrac | T-17 | `vendor/bin/deptrac analyse` termina con código 0; una importación prohibida de prueba hace que falle |
| T-20 | `infra`: `docker compose up` levanta motor vacío, API y frontend | T-17 | `docker compose up -d` y luego `\dt library.*` en `psql` no lista tablas hasta correr las migraciones |
| T-21 | `tool`: sembrador de demostración **por la API** (≥ 40 libros, 10 socios, 15 préstamos, 3 vencidos) | T-17 | Tras correr el sembrador, `GET /api/v1/loans/overdue` devuelve 3 líneas y `GET /api/v1/reports/most-borrowed` devuelve filas |
| T-22 | `app`: login, catálogo, socios, prestar, devolver, vencidos, reporte | T-16 | Prueba end-to-end con Playwright en el repo `app` recorre cada flujo y pasa |
| T-23 | `page`: sitio público estático de presentación | — | Prueba con Playwright sin API: carga en 375 px y en 1280 px, sin scroll horizontal |
| T-24 | Rendimiento de búsqueda (RNF-1) | T-21 | Sembrador con 5 000 libros; `GET /api/v1/books?q=...` tiene p95 < 300 ms en 20 ejecuciones, con un script que imprime el resultado |

## Definición de terminado de la prueba

Quien llegue a T-17 con evidencia tiene el **núcleo** hecho. T-18 a T-24 son lo que separa un trabajo bueno de uno sobresaliente. Se evalúa el desempeño con SDD, no la cantidad de tareas: una tarea con evidencia ejecutable vale más que tres a medias.

## Correcciones respecto a `spec-python`

Estas correcciones viven solo en Laravel. `spec-python/` no se modifica.

| Incongruencia en `docs` | Corrección en `spec-laravel` |
|---|---|
| ADR-002 exige testigo de versión; `data-model.md` no lo tiene | `version` en `book` (T-02) y prueba de concurrencia (T-13). Ver D-04 en `plan.md` |
| RNF-1 sin tarea ni evidencia | Nueva tarea T-24 |
| Evidencias manuales (`recorrido en la feria`, `correcto en móvil y escritorio`) | Reemplazadas por pruebas de Playwright (T-22, T-23) |
| `DocumentNumber` en el plan pero no en las tareas | Incluido en T-03 y T-06 |
| `GET /books/{id}/cover` mencionado pero no listado | Incluido en la prueba de contrato de T-16 |
| Sin código de error para token ausente o vencido | T-16 usa `UNAUTHENTICATED` (401). Ver [`api-desviaciones.md`](api-desviaciones.md) D-API-01 |

## Estado y evidencia ejecutada

Verificado el 13 de octubre de 2026. Cada comando se ejecutó y terminó con éxito.

| Tareas | Repositorio | Comando | Resultado |
|---|---|---|---|
| T-01 a T-18 | `api` | `php artisan test` | 116 pruebas, 206 aserciones |
| T-19 | `api` | `vendor/bin/deptrac analyse` | 0 violaciones |
| T-21 | `tool` | `php seed.php` | 40 libros, 10 socios, 15 préstamos, 3 líneas vencidas, 12 filas en el reporte |
| T-22 | `app` | `npm run test:e2e` | 3 pruebas |
| T-23 | `page` | `npm test` | 2 pruebas (375 px y 1280 px) |
| T-24 | `tool` | `php seed.php --books=5000` y `php bench.php` | p95 de 203 ms con 5 000 libros |

**T-20 queda parcial.** `docker compose config` valida el archivo y el repositorio `infra` no contiene DDL, pero `docker compose up --build` **no se ejecutó**: construir las imágenes exige descargar varios cientos de megabytes y la prueba se hizo con datos móviles.
