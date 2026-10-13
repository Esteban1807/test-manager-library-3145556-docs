# Constitución — Manager Library (Laravel)

Principios innegociables del proyecto en **PHP 8.x con Laravel** y **React**. Son los mismos 12 artículos de [`spec-python/constitution.md`](../spec-python/constitution.md), con su verificación traducida a las herramientas de este stack. Un principio que nadie puede comprobar es una intención: cada artículo trae **cómo se verifica**. Un cambio que viole un artículo se rechaza sin discutir el resto.

Lo que define el negocio (historias, reglas, API y modelo de datos) no vive aquí: vive en [`spec-python/spec.md`](../spec-python/spec.md), [`spec-python/api-contract.md`](../spec-python/api-contract.md) y [`spec-python/data-model.md`](../spec-python/data-model.md), que son la fuente compartida de los tres stacks.

---

## Artículo I — El dominio no depende de nada

`app/Domain/` no importa ningún otro namespace del proyecto ni del framework: nada de `Illuminate\*`, Eloquent, facades, helpers de Laravel ni librerías de terceros. Solo PHP puro: clases `final` y `readonly`, enums nativos, excepciones propias.

**Por qué:** es lo único que permite probar las invariantes sin base de datos, sin servidor y sin contenedor, y cambiar de ORM o de framework sin tocar reglas.
**Verificación:** una regla de Deptrac que prohíbe que `Domain` dependa de `Application`, `Adapters`, `Bootstrap` o `Illuminate`. El análisis falla en el CI si se introduce la dependencia.

## Artículo II — El núcleo declara sus puertos; el mundo los implementa

La capa de aplicación solo conoce el dominio. Todo lo que necesita del exterior (repositorios, reloj, hash de claves, almacenamiento de portadas, generación de tokens) lo expresa como una **interfaz que ella misma define** en `app/Application/Ports/`. Ningún tipo de Eloquent, de Laravel o de librería aparece en la firma de un puerto.

**Verificación:** una regla de Deptrac que impide que `Application` importe `Adapters` o `Illuminate\Database\*`. Los tests de aplicación corren con dobles de los puertos y sin referenciar `Adapters`.

## Artículo III — Un único punto de composición

Un solo `ServiceProvider` (`app/Bootstrap/Composition/PortBindings.php`) es donde un puerto se encuentra con su adaptador. Fuera de él no se instancia infraestructura: no se usa `new` sobre clases de `Adapters` y no se resuelven adaptadores con `app()` en el resto del código.

**Verificación:** un test de arquitectura que falla si una clase de `Adapters` se instancia fuera de `Bootstrap/Composition/`.

## Artículo IV — SOLID no es decorativo

| Principio | Aplicación concreta | Verificación |
|---|---|---|
| **SRP** | Un caso de uso = una clase en `app/Application/UseCases/`. `LendBooksService` presta y no hace nada más | Una clase con dos razones para cambiar se parte |
| **OCP** | Cambiar el almacenamiento de portadas (disco → S3) no toca dominio ni aplicación | El adaptador nuevo solo añade una línea en `PortBindings` |
| **LSP** | Los dobles de test sustituyen al adaptador real sin cambiar expectativas | El mismo test pasa contra el doble y contra el real |
| **ISP** | Puertos pequeños: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | Ningún implementador deja métodos sin usar |
| **DIP** | El puerto lo define quien lo consume | Los puertos viven en `app/Application/Ports/` |

## Artículo V — El esquema es propiedad de las migraciones

El repositorio `infra` levanta un motor **vacío**. Todo el DDL (tablas, índices, restricciones, extensiones y semilla) vive en las **migraciones de Laravel** (`database/migrations/`). Ver [ADR-001](adr/adr-001-propiedad-del-esquema.md).

**Verificación:** `infra` no contiene `CREATE TABLE`.

## Artículo VI — Las invariantes que la base puede imponer, la base las impone

Toda invariante expresable en el motor (`CHECK`, `UNIQUE`, FK) se declara en las migraciones, con `->unique()`, `->foreign()` o `DB::statement()` para los `CHECK`. Lo que no está aplicado se marca **pendiente** en `spec-python/data-model.md`; no se promete.

**Verificación:** las consultas de `data-model.md` §10 coinciden con lo que declaran las migraciones.

## Artículo VII — Lo derivado se calcula, no se almacena

El estado de un préstamo (`open`, `overdue`, `closed`) y los días de retraso se calculan en el dominio. No hay columna para ellos.

**Verificación:** ninguna migración contiene columnas `status` ni `days_late`.

## Artículo VIII — Los valores congelados no siguen al catálogo

La línea de préstamo guarda título, ISBN y categoría del instante del préstamo. Ningún reporte los lee del catálogo vivo. Ver [ADR-004](adr/adr-004-reporte-agregado-y-congelado.md).

**Verificación:** una prueba que renombra un libro después de prestarlo y comprueba que el reporte conserva el título viejo.

## Artículo IX — Ningún secreto entra al repositorio ni a un log

Ni contraseñas, ni hashes precalculados, ni la clave del token. Todo secreto viene del entorno (`.env`). El `.env` real está en `.gitignore`; solo se versiona `.env.example`.

**Verificación:** `gitleaks` en el CI, y un test que comprueba que ninguna respuesta HTTP incluye `password` ni `password_hash`.

## Artículo X — Gana el motor

Si el documento de datos y la base discrepan, el documento está roto y se corrige el documento.

## Artículo XI — Idioma

Código, columnas, rutas y namespaces en inglés. Documentación y mensajes al usuario en español (`lang/es/`).

## Artículo XII — Nada se da por hecho sin evidencia

Una tarea de [`tasks.md`](tasks.md) está terminada cuando su **evidencia existe y se puede ejecutar** con un comando o una prueba automatizada, no cuando «ya está programada» ni cuando alguien la revisó a mano.

**Verificación:** cada tarea de `tasks.md` tiene un comando o una prueba concreta que la demuestra.
