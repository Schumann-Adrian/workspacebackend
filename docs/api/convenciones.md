# Convenciones de la API · BookFlow v1

Nota de convenciones acordada en el Sprint 2. Aplica a `openapi.yaml` y a la implementación del backend.

## 1. Enfoque API-first

1. El contrato (`docs/api/openapi.yaml`) se modifica **antes** que el código, mediante PR revisada.
2. El backend valida peticiones y respuestas contra el contrato en los tests de contrato (ver `docs/api/contract-tests/`).
3. El frontend puede trabajar en paralelo con un mock generado del contrato (p. ej. `npx @stoplight/prism-cli mock docs/api/openapi.yaml`).
4. `npx @redocly/cli lint docs/api/openapi.yaml` debe pasar sin errores en CI.

## 2. Naming

| Ámbito | Convención | Ejemplo |
|---|---|---|
| Rutas (paths) | `kebab-case`, sustantivos en plural | `/staff-members`, `/business-hours`, `/time-blocks` |
| Parámetros de ruta y query | `snake_case` | `{staff_id}`, `?date_from=` |
| Campos JSON | `snake_case` (igual que la BD, mapping directo) | `duration_minutes`, `starts_at` |
| Tablas y columnas BD | `snake_case`, tablas en plural | `staff_members.display_name` |
| Restricciones BD | prefijo de tipo: `pk_`, `fk_`, `uq_`, `ck_`, `ix_`, `ex_` | `ex_appt_no_overlap` |
| Valores enum | `snake_case` en minúscula | `business_admin`, `pending` |
| Códigos de error | `UPPER_SNAKE_CASE` | `SLOT_UNAVAILABLE` |
| `operationId` | `camelCase` verbo + recurso | `createService`, `listAppointments` |
| Cabeceras propias | `Title-Case` sin prefijo `X-` salvo `X-Request-Id` | `Idempotency-Key` |

Acciones que no son CRUD se expresan como **sub-recurso verbo** con `PATCH` sobre la entidad: `/appointments/{id}/cancel`, `/appointments/{id}/complete`.

## 3. Comandos y consultas (CQS)

* **Consultas** (`GET`): nunca modifican estado; filtrables y paginadas. Ej.: `GET /appointments?date_from=…&staff_id=…&status=pending`.
* **Comandos**: `POST` crea, `PATCH` modifica parcialmente o ejecuta una transición de estado, `PUT` reemplaza un conjunto completo (idempotente: `/business-hours`, `/staff-members/{id}/services`), `DELETE` elimina.
* La consulta de disponibilidad (`/availability`) es una *read model* calculada: no se persiste; se optimiza con índices `ix_appt_staff`, `ix_staff_schedules_lookup` e `ix_time_blocks_range`.

## 4. Envoltorios y metadatos

**Recurso único**
```json
{ "data": { "id": "…", "name": "Corte de pelo", "price": 15.0, "duration_minutes": 30 } }
```

**Colección paginada**
```json
{
  "data": [ { … }, { … } ],
  "meta": { "total": 57, "page": 2, "limit": 20, "total_pages": 3 }
}
```

**Error** → ver `errores.md` (sin envoltorio `data`).

* `204 No Content` en `DELETE` y `logout` (sin cuerpo).
* `201 Created` + cabecera `Location` en creaciones.
* Orden de campos en las respuestas: `id` → identificadores/FK → atributos de negocio → estado → objetos anidados → timestamps (`created_at`, `updated_at`).
* Los campos opcionales sin valor se devuelven como `null` (no se omiten), para que el contrato sea estable.

## 5. Paginación, filtros y orden

* `page` (≥ 1, por defecto 1) y `limit` (1–100, por defecto 20). Valores fuera de rango → `422 VALIDATION_ERROR`.
* Orden con `sort=campo` (ascendente) o `sort=-campo` (descendente); solo los campos enumerados en el contrato.
* Búsqueda libre con `q` (contiene, sin distinguir mayúsculas).
* Rangos de fechas con `date_from` / `date_to` (inclusive, `YYYY-MM-DD`, zona horaria del negocio).
* Filtros multivalor separados por coma: `status=pending,completed`.

## 6. Fechas, dinero e identificadores

* Instantes en **ISO 8601 con offset** (`2026-12-01T10:00:00+01:00`); en BD `timestamptz` (UTC).
* Fechas de calendario en `YYYY-MM-DD`, interpretadas en `tenants.timezone` (por defecto `Europe/Madrid`).
* Horas de horario en `HH:mm`; `weekday` ISO (1 = lunes … 7 = domingo).
* Importes como número con 2 decimales en euros (`numeric(10,2)` en BD). El precio se **congela** en la cita.
* Identificadores `uuid` v4 generados por la BD. El `slug` es el identificador público del negocio.

## 7. Seguridad

* `Authorization: Bearer <JWT>`. Access token 15 min, refresh token 7 días con **rotación** (cada uso invalida el anterior; reutilizar uno revocado revoca toda la sesión).
* Claims: `sub` (user_id), `tid` (tenant_id), `role`, `exp`, `iat`, `jti`.
* El `tenant_id` se obtiene **solo del token** en endpoints privados; nunca de la URL o el cuerpo (RNF-01). En endpoints públicos se resuelve por `slug`.
* Matriz de roles: cada operación declara `x-roles` en el contrato. Resumen:

| Recurso | super_admin | business_admin | worker | public |
|---|---|---|---|---|
| `/tenants` (global) | CRUD estado | – | – | registro |
| `/business-profile` | – | leer/editar | leer | – |
| `/services`, `/staff-members`, horarios | – | CRUD | leer | – |
| `/appointments` | – | todas las del local | solo las suyas | – |
| `/public/**` | – | – | – | leer, reservar, cancelar con token |

* Contraseñas con bcrypt (coste 12). Rate limiting en login, registro y endpoints públicos.
* CORS restringido a los orígenes del frontend. HTTPS obligatorio fuera de local.

## 8. Versionado

* Versión mayor en la ruta: `/api/v1`. Cambios **compatibles** (nuevo endpoint, nuevo campo opcional en respuesta, nuevo valor de filtro) se publican en `v1` incrementando `info.version` (semver minor/patch).
* Cambios **incompatibles** (renombrar/eliminar campo, cambiar tipo, nuevo campo obligatorio en petición, cambiar códigos de estado) → nueva ruta `/api/v2`. `v1` se mantiene al menos 6 meses con cabeceras `Deprecation: true` y `Sunset: <fecha>`.
* Los clientes deben **ignorar campos desconocidos** en las respuestas.

## 9. Concurrencia e idempotencia

* Reserva: la BD impide solapes con `EXCLUDE USING gist (staff_id WITH =, tstzrange(starts_at, ends_at) WITH &&) WHERE status <> 'cancelled'`. La API traduce la violación a `409 SLOT_UNAVAILABLE`.
* `Idempotency-Key` (uuid) en `POST …/appointments`: reenvíos en 24 h devuelven la respuesta original.
* `PUT` y las transiciones `PATCH …/cancel` son idempotentes a nivel de contrato en el sentido de que un segundo intento devuelve `409 INVALID_STATUS_TRANSITION` sin efectos secundarios.

## 10. Pruebas de contrato

* Herramienta: **Postman** (colección `docs/api/contract-tests/bookflow.postman_collection.json`), ejecutable en CI con `newman`.
* Cada petición valida: código de estado, cabecera `Content-Type`, esquema JSON del cuerpo (extraído de `openapi.yaml`) y reglas de negocio clave.
* Casos mínimos por HU crítica en `docs/api/contract-tests/casos-minimos.md`.
