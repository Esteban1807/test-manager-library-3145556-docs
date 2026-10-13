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
