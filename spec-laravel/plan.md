# Plan técnico — Manager Library (Laravel)

Cómo se construye lo que dicen [`spec-python/spec.md`](../spec-python/spec.md), [`spec-python/api-contract.md`](../spec-python/api-contract.md) y [`spec-python/data-model.md`](../spec-python/data-model.md), aplicando [`constitution.md`](constitution.md). Este plan reemplaza las decisiones de `spec-python/plan.md` por sus equivalentes en Laravel.

Las filas marcadas con **⚠ desviación** cambian algo del documento compartido. Esa corrección vive **solo en Laravel**; `spec-python/` no se toca.

## 1. Decisiones técnicas

| # | Decisión | Elección (Laravel) | Por qué |
|---|---|---|---|
| D-01 | Lenguaje y runtime | PHP 8.3 o superior | Versión mínima que soporta `readonly` y enums nativos del dominio |
| D-02 | Framework HTTP | Laravel (última versión LTS al iniciar T-01). Rutas en `routes/api.php`, controladores en `app/Adapters/Http/`, validación con `FormRequest` | Contrato en `api-contract.md` |
| D-03 | Persistencia | Eloquent en `app/Adapters/Persistence/` sobre PostgreSQL 16. Migraciones de Laravel en `database/migrations/` | El esquema es de las migraciones (Artículo V) |
| D-04 | Concurrencia | **⚠ desviación.** Testigo de versión: columna `version integer not null` en `book`, incrementada en cada escritura, más el `CHECK` `ck_book_available_range` | ADR-002 exige testigo de versión, pero `data-model.md` §3 no lo tiene. Ver §4 |
| D-05 | Fechas | `CarbonImmutable` solo en adaptadores; el dominio recibe `DateTimeImmutable`. El reloj es el puerto `Clock` | Probar vencimientos sin esperar 14 días |
| D-06 | Reporte | Una consulta agregada en un adaptador de lectura, detrás de un puerto de lectura | [ADR-004](adr/adr-004-reporte-agregado-y-congelado.md): estable y barato |
| D-07 | Objetos de valor | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange`: clases `readonly` del dominio, sin tabla propia | Sin identidad. **⚠ desviación:** incluye `DocumentNumber`, que el plan de Python listaba y `tasks.md` no cubría |
| D-08 | Portadas | `Storage` de Laravel detrás del puerto `CoverStorage`; disco local en desarrollo, S3 en producción. La base guarda solo `cover_key` | Clave opaca, nunca ruta |
| D-09 | Contraseñas | `Hash::driver('argon2id')` detrás del puerto `PasswordHasher`; el dominio nunca ve la clave en claro | Artículo IX |
| D-10 | Arranque | Comando Artisan `library:bootstrap-admin` que crea el admin desde `ADMIN_USERNAME` / `ADMIN_PASSWORD`. Se ejecuta en el arranque del contenedor | La base no puede producir el hash |
| D-11 | Autenticación | JWT HS256, 8 horas, clave en `JWT_SECRET`, detrás del puerto `TokenIssuer`. **Librería pendiente de elegir en T-14** | Simple y sin estado |
| D-12 | Pruebas | PHPUnit (incluido en Laravel). Pruebas de dominio y aplicación sin framework; integración con PostgreSQL real | Pirámide de pruebas |
| D-13 | Reglas de capas | **Deptrac**, con el archivo `deptrac.yaml` en la raíz del repo `api`. Se ejecuta como prueba en el CI | Reemplaza a import-linter (Python) y NetArchTest (.NET) |

## 2. Configuración por entorno

| Variable | Obligatoria | Ejemplo |
|---|---|---|
| `DB_CONNECTION` | sí | `pgsql` |
| `DB_HOST` / `DB_PORT` / `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD` | sí | `db` / `5432` / `library` / `library` / *(en `.env`)* |
| `JWT_SECRET` | sí | cadena larga aleatoria |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | sí | definidos en `.env`, nunca en el repo |
| `COVERS_DISK` | no | `local` (por defecto) |
| `CORS_ALLOWED_ORIGINS` | sí | `http://localhost:5173` |

## 3. Estrategia de pruebas

| Nivel | Qué prueba | Contra qué | Herramienta |
|---|---|---|---|
| Dominio | Invariantes: máximo 3, vencimiento a 14 días, disponibles en rango | Nada: objetos puros | PHPUnit |
| Aplicación | Casos de uso: prestar, devolver, reporte | Dobles de los puertos | PHPUnit |
| Integración | Repositorios y consulta del reporte | PostgreSQL real | PHPUnit + `RefreshDatabase` |
| Contrato | Cuerpo de error único y códigos de estado | La API levantada | PHPUnit (tests de feature) |
| Arquitectura | Artículos I, II y III | `deptrac.yaml` | `vendor/bin/deptrac analyse` |
| Concurrencia | Dos préstamos del último ejemplar | PostgreSQL real | PHPUnit con dos procesos |

## 4. Corrección del testigo de versión (D-04)

ADR-002 y el plan de Python exigen un testigo de versión. El `data-model.md` compartido no lo tiene. En Laravel lo agregamos así:

- `book.version`: `integer`, no nulo, **sin** valor por defecto en la migración (el dominio lo pone en `0` al crear el libro, igual que el resto de valores).
- Cada escritura sobre el libro usa `UPDATE ... WHERE id = ? AND version = ?` y suma 1 a `version`. Si no afecta ninguna fila, la operación responde `409 BOOK_NOT_AVAILABLE`.
- Esta columna **no** se lista en `data-model.md`. Queda registrada aquí como desviación y se decide en `docs` más adelante si se incorpora al documento compartido.

## 5. Orden de construcción

Esqueleto y salud → migración inicial con semilla → dominio → casos de uso con dobles → adaptadores → API → pruebas de contrato y concurrencia.
El detalle, con evidencia ejecutable, está en [`tasks.md`](tasks.md) (se escribe en el paso siguiente).
