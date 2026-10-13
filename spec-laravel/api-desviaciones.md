# Desviaciones y aclaraciones del contrato de la API — Laravel

El contrato HTTP se rige por [`spec-python/api-contract.md`](../spec-python/api-contract.md), que sigue siendo la fuente compartida. Este archivo registra lo que **Laravel hace distinto o aclara**, sin modificar el documento compartido.

## D-API-01 · Código de error para token ausente, inválido o vencido

- **Problema:** `api-contract.md` §2 define `INVALID_CREDENTIALS` (401), pero solo para el login. No hay código para un 401 por token.
- **Decisión:** agregar `UNAUTHENTICATED` con estado **401**. Se usa cuando falta el token, el token es inválido o está vencido.
- **Formato:** el mismo cuerpo de error único de §1, con `code: "UNAUTHENTICATED"` y `message` en español.
- **Dónde se verifica:** T-16 (prueba de contrato).

## D-API-02 · Endpoint de lectura de la portada

- **Problema:** `api-contract.md` §3.4 menciona `GET /books/{id}/cover` en el texto de la portada, pero no aparece en la tabla de endpoints.
- **Decisión:** se añade a la tabla de endpoints de libros con rol **autenticado**. Devuelve el binario de la portada o `404`.
- **Dónde se verifica:** T-16 (prueba de contrato recorre todas las rutas).

## D-API-03 · Prefijo de la ruta de salud

- **Problema:** `api-contract.md` §3.1 escribe `GET /health` sin prefijo, pero §1 dice que la base de todas las rutas es `/api/v1`.
- **Decisión:** la ruta real es **`GET /api/v1/health`**. Aplica la regla general de la base.
- **Dónde se verifica:** T-01.

## Regla de implementación · Orden de rutas

Las rutas literales deben registrarse **antes** que las que tienen parámetro. Si no, `GET /api/v1/loans/overdue` puede capturarse como `GET /api/v1/loans/{id}` y responder `404` con un id que no es UUID.

- **Dónde se verifica:** T-16 (prueba de contrato con `/loans/overdue`).

## D-API-04 · Reloj de demostración

- **Problema:** T-21 pide que el sembrador cree 3 préstamos vencidos **por la API**. El vencimiento siempre es `loaned_at + 14 días` y `loaned_at` lo pone el reloj del servidor, así que por la API no se puede crear un préstamo que ya esté vencido.
- **Decisión:** el adaptador `SystemClock` acepta la cabecera `X-Demo-Now` (instante ISO 8601) como «ahora» de esa petición, **solo** si la variable de entorno `LIBRARY_DEMO_CLOCK` vale `true`. Por defecto vale `false` y la cabecera se ignora.
- **Alcance:** vive en el adaptador del puerto `Clock`. El dominio y la aplicación no cambian. No debe activarse en producción.
- **Dónde se verifica:** `php artisan test --filter=DemoClockTest` y la ejecución de `php seed.php` en el repositorio `tool`.

## D-API-05 · Un identificador que no es UUID

- **Problema:** el contrato dice que los identificadores son UUID, pero no dice qué responder si la ruta trae otra cosa (`/books/abc`). Sin tratarlo, el motor rechaza el valor y la API respondía 500.
- **Decisión:** los parámetros de ruta solo aceptan UUID. Cualquier otro valor es una ruta que no existe: **404 `NOT_FOUND`**. En el cuerpo de una petición, un identificador mal formado o una categoría inexistente es **422 `VALIDATION_ERROR`**.
- **Dónde se verifica:** `php artisan test --testsuite=Contract`.
