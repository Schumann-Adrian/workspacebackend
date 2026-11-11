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
| Rutas (paths) | `kebab-case`, sustantivos en plural | `/trabajadores`, `/horario-local`, `/bloqueos-horario` |
| Parámetros de ruta y query | `snake_case` | `{trabajador_id}`, `?fecha_desde=` |
| Campos JSON | `snake_case` (igual que la BD, mapping directo) | `duracion_minutos`, `fecha_inicio` |
| Tablas y columnas BD | `snake_case`, tablas en plural | `trabajadores.nombre_visible` |
| Restricciones BD | prefijo de tipo: `pk_`, `fk_`, `uq_`, `ck_`, `ix_`, `ex_` | `ex_citas_sin_solape` |
| Valores enum | `snake_case` en minúscula | `admin_negocio`, `pendiente` |
| Códigos de error | `UPPER_SNAKE_CASE` | `HUECO_NO_DISPONIBLE` |
| `operationId` | `camelCase` verbo + recurso | `crearServicio`, `listarCitas` |
| Cabeceras propias | `Title-Case` sin prefijo `X-` salvo `X-Request-Id` | `Idempotency-Key` |

Acciones que no son CRUD se expresan como **sub-recurso verbo** con `PATCH` sobre la entidad: `/citas/{id}/cancelar`, `/citas/{id}/atender`.

## 3. Comandos y consultas (CQS)

* **Consultas** (`GET`): nunca modifican estado; filtrables y paginadas. Ej.: `GET /citas?fecha_desde=…&trabajador_id=…&status=pendiente`.
* **Comandos**: `POST` crea, `PATCH` modifica parcialmente o ejecuta una transición de estado, `PUT` reemplaza un conjunto completo (idempotente: `/horario-local`, `/trabajadores/{id}/servicios`), `DELETE` elimina.
* La consulta de disponibilidad (`/disponibilidad`) es una *read model* calculada: no se persiste; se optimiza con índices `ix_citas_trabajador`, `ix_turnos_trabajador_busqueda` e `ix_bloqueos_horario_rango`.

## 4. Envoltorios y metadatos

**Recurso único**
```json
{ "data": { "id": "…", "nombre": "Corte de pelo", "precio": 15.0, "duracion_minutos": 30 } }
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
* Orden de campos en las respuestas: `id` → identificadores/FK → atributos de negocio → estado → objetos anidados → timestamps (`creado_en`, `actualizado_en`).
* Los campos opcionales sin valor se devuelven como `null` (no se omiten), para que el contrato sea estable.

## 5. Paginación, filtros y orden

* `page` (≥ 1, por defecto 1) y `limit` (1–100, por defecto 20). Valores fuera de rango → `422 ERROR_VALIDACION`.
* Orden con `orden=campo` (ascendente) o `orden=-campo` (descendente); solo los campos enumerados en el contrato.
* Búsqueda libre con `q` (contiene, sin distinguir mayúsculas).
* Rangos de fechas con `fecha_desde` / `fecha_hasta` (inclusive, `YYYY-MM-DD`, zona horaria del negocio).
* Filtros multivalor separados por coma: `status=pendiente,atendida`.

## 6. Fechas, dinero e identificadores

* Instantes en **ISO 8601 con offset** (`2026-12-01T10:00:00+01:00`); en BD `timestamptz` (UTC).
* Fechas de calendario en `YYYY-MM-DD`, interpretadas en `negocios.zona_horaria` (por defecto `Europe/Madrid`).
* Horas de horario en `HH:mm`; `dia_semana` ISO (1 = lunes … 7 = domingo).
* Importes como número con 2 decimales en euros (`numeric(10,2)` en BD). El precio se **congela** en la cita.
* Identificadores `uuid` v4 generados por la BD. El `slug` es el identificador público del negocio.

## 7. Seguridad

* `Authorization: Bearer <JWT>`. Access token 15 min, refresh token 7 días con **rotación** (cada uso invalida el anterior; reutilizar uno revocado revoca toda la sesión).
* Claims: `sub` (id del usuario), `tid` (tenant_id), `rol`, `exp`, `iat`, `jti`.
* El `tenant_id` se obtiene **solo del token** en endpoints privados; nunca de la URL o el cuerpo (RNF-01). En endpoints públicos se resuelve por `slug`.
* Matriz de roles: cada operación declara `x-roles` en el contrato. Resumen:

| Recurso | superadmin | admin_negocio | trabajador | publico |
|---|---|---|---|---|
| `/negocios` (global) | CRUD estado | – | – | registro |
| `/perfil-negocio` | – | leer/editar | leer | – |
| `/servicios`, `/trabajadores`, horarios | – | CRUD | leer | – |
| `/citas` | – | todas las del local | solo las suyas | – |
| `/publico/**` | – | – | – | leer, reservar, cancelar con token |

* Contraseñas con bcrypt (coste 12). Rate limiting en login, registro y endpoints públicos.
* CORS restringido a los orígenes del frontend. HTTPS obligatorio fuera de local.

## 8. Versionado

* Versión mayor en la ruta: `/api/v1`. Cambios **compatibles** (nuevo endpoint, nuevo campo opcional en respuesta, nuevo valor de filtro) se publican en `v1` incrementando `info.version` (semver minor/patch).
* Cambios **incompatibles** (renombrar/eliminar campo, cambiar tipo, nuevo campo obligatorio en petición, cambiar códigos de estado) → nueva ruta `/api/v2`. `v1` se mantiene al menos 6 meses con cabeceras `Deprecation: true` y `Sunset: <fecha>`.
* Los clientes deben **ignorar campos desconocidos** en las respuestas.

## 9. Concurrencia e idempotencia

* Reserva: la BD impide solapes con `EXCLUDE USING gist (trabajador_id WITH =, tstzrange(fecha_inicio, fecha_fin) WITH &&) WHERE estado <> 'cancelada'`. La API traduce la violación a `409 HUECO_NO_DISPONIBLE`.
* `Idempotency-Key` (uuid) en `POST …/citas`: reenvíos en 24 h devuelven la respuesta original.
* `PUT` y las transiciones `PATCH …/cancelar` son idempotentes a nivel de contrato en el sentido de que un segundo intento devuelve `409 TRANSICION_ESTADO_INVALIDA` sin efectos secundarios.

## 10. Pruebas de contrato

* Herramienta: **Postman** (colección `docs/api/contract-tests/bookflow.postman_collection.json`), ejecutable en CI con `newman`.
* Cada petición valida: código de estado, cabecera `Content-Type`, esquema JSON del cuerpo (extraído de `openapi.yaml`) y reglas de negocio clave.
* Casos mínimos por HU crítica en `docs/api/contract-tests/casos-minimos.md`.

## 11. Correspondencia con las rutas del documento de requisitos

| PDF (Sprint 1) | Contrato v1 |
|---|---|
| `POST /api/tenants/register` | `POST /api/v1/negocios/registro` |
| `POST /api/auth/login` | `POST /api/v1/auth/iniciar-sesion` |
| CRUD de servicios por `tenant_id` | `/api/v1/servicios` (el `tenant_id` sale del token) |
| `POST /api/appointments` | `POST /api/v1/publico/negocios/{slug}/citas` (reserva del cliente) |
| `PATCH /api/appointments/{id}/cancel` | `PATCH /api/v1/citas/{cita_id}/cancelar` (local) y `PATCH /api/v1/publico/citas/{cita_id}/cancelar` (cliente) |
| Tabla `Schedules` | Tablas `horario_local`, `turnos_trabajador` y `bloqueos_horario` |
