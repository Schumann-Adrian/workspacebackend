# Guía de errores · BookFlow API v1

Todas las respuestas de error (4xx/5xx) usan **el mismo cuerpo**, definido en `openapi.yaml` como `components.schemas.Error`.

```json
{
  "code": "VALIDATION_ERROR",
  "message": "La petición contiene campos no válidos.",
  "details": [
    { "field": "duration_minutes", "issue": "must_be_between", "message": "Debe estar entre 5 y 480." },
    { "field": "customer.email",   "issue": "invalid_format",  "message": "Email no válido." }
  ],
  "trace_id": "01JC8Z6R4K2M3N4P5Q6R7S8T9V"
}
```

| Campo | Obligatorio | Descripción |
|---|---|---|
| `code` | sí | Código **estable** en `UPPER_SNAKE_CASE`. El frontend decide por `code`, nunca por `message`. |
| `message` | sí | Texto en español apto para mostrar al usuario. Puede cambiar sin previo aviso. |
| `details[]` | sí (puede ser `[]`) | Lista de problemas concretos. `field` usa notación de puntos e índices (`slots[2].end_time`); `issue` en `snake_case`. |
| `trace_id` | no | Id de correlación; coincide con la cabecera `X-Request-Id` y con los logs. |

## Reglas

1. **Nunca** se devuelven trazas de pila, SQL ni nombres de tablas/restricciones al cliente. Se registran en el log con el `trace_id`.
2. **Aislamiento multi-tenant:** acceder a un recurso de otro negocio devuelve `404 NOT_FOUND` (no `403`) para no revelar que existe.
3. **Login:** email inexistente y contraseña errónea devuelven el mismo `401 INVALID_CREDENTIALS` (evita enumerar usuarios).
4. `400` = la petición no se puede interpretar (JSON roto, tipo incorrecto en query). `422` = se entiende pero viola reglas de validación o de negocio.
5. `409` = conflicto con el **estado actual** de los datos (duplicados, hueco ocupado, transición no permitida). Reintentar la misma petición no lo arreglará.
6. Todos los errores de validación se acumulan en `details[]` en una sola respuesta (no se para en el primero).
7. Errores de la BD se traducen en la capa de servicio: violación de `ex_appt_no_overlap` → `SLOT_UNAVAILABLE`; violación de unicidad → código `*_DUPLICATED` / `*_ALREADY_REGISTERED`.

## Catálogo de códigos

| HTTP | `code` | Cuándo | Endpoints típicos |
|---|---|---|---|
| 400 | `MALFORMED_REQUEST` | Cuerpo no es JSON válido o tipo erróneo en parámetro | Todos |
| 401 | `UNAUTHENTICATED` | Falta cabecera `Authorization` o token inválido/firmado mal | Privados |
| 401 | `TOKEN_EXPIRED` | Access token caducado → el cliente llama a `/auth/refresh` | Privados |
| 401 | `INVALID_CREDENTIALS` | Email o contraseña incorrectos | `POST /auth/login` |
| 401 | `REFRESH_TOKEN_INVALID` | Refresh token caducado, revocado o reutilizado | `POST /auth/refresh` |
| 403 | `FORBIDDEN` | El rol no puede ejecutar la operación (p. ej. `worker` crea servicio) | Privados |
| 403 | `ACCOUNT_INACTIVE` | Usuario desactivado o pendiente de activación | `POST /auth/login` |
| 403 | `TENANT_SUSPENDED` | El negocio está suspendido por el SuperAdmin | Login y privados |
| 403 | `CANCEL_TOKEN_INVALID` | `cancel_token` no corresponde a la cita | `PATCH /public/appointments/{id}/cancel` |
| 404 | `NOT_FOUND` | Recurso inexistente **o de otro tenant**; slug inexistente | Todos con `{id}`/`{slug}` |
| 409 | `EMAIL_ALREADY_REGISTERED` | Email ya usado por otro negocio/usuario | `POST /tenants/register`, `POST /staff-members` |
| 409 | `SLUG_ALREADY_TAKEN` | Slug público ya en uso | `PATCH /business-profile` |
| 409 | `SERVICE_NAME_DUPLICATED` | Ya existe un servicio con ese nombre en el local | `POST/PATCH /services` |
| 409 | `RESOURCE_IN_USE` | Borrar servicio/trabajador con citas pendientes futuras | `DELETE /services/{id}`, `DELETE /staff-members/{id}` |
| 409 | `SLOT_UNAVAILABLE` | El hueco se ocupó entre la consulta y la reserva (carrera) | `POST /public/businesses/{slug}/appointments` |
| 409 | `INVALID_STATUS_TRANSITION` | Transición no permitida (`completed → cancelled`, `cancelled → completed`…) | `PATCH …/complete`, `PATCH …/cancel` |
| 409 | `BLOCK_OVERLAPS_APPOINTMENTS` | Bloqueo solapa citas pendientes (ids en `details`) | `POST /time-blocks` |
| 409 | `IDEMPOTENCY_KEY_REUSED` | Misma `Idempotency-Key` con cuerpo distinto | `POST …/appointments` |
| 410 | `ACTIVATION_TOKEN_INVALID` | Token de activación caducado o ya usado | `POST /auth/activate` |
| 422 | `VALIDATION_ERROR` | Campos obligatorios, formatos, rangos, `additionalProperties` | Todos con cuerpo/query |
| 422 | `STAFF_NOT_QUALIFIED` | El trabajador no realiza ese servicio | `POST …/appointments`, `GET …/availability` |
| 422 | `OUTSIDE_WORKING_HOURS` | `starts_at` fuera de horario/turno o en un bloqueo | `POST …/appointments` |
| 422 | `BOOKING_IN_PAST` | Fecha/hora pasada o a más de 90 días | `GET …/availability`, `POST …/appointments` |
| 422 | `SLOT_MISALIGNED` | `starts_at` no encaja en la rejilla `slot_step_minutes` | `POST …/appointments` |
| 422 | `SCHEDULE_OVERLAP` | Franjas semanales que se solapan entre sí | `PUT /business-hours`, `PUT …/schedules` |
| 422 | `INVALID_DATE_RANGE` | `date_to < date_from` o rango > 31 días | `GET /appointments`, `GET /time-blocks` |
| 429 | `RATE_LIMITED` | Demasiadas peticiones (login: 5/min por IP+email; público: 60/min por IP) | Login, registro, públicos |
| 500 | `INTERNAL_ERROR` | Error no controlado | Todos |
| 503 | `SERVICE_UNAVAILABLE` | BD u otra dependencia caída | Todos, `/health` |

### Valores habituales de `details[].issue`

`required`, `invalid_format`, `invalid_type`, `too_short`, `too_long`, `must_be_between`, `not_allowed` (campo extra), `duplicated`, `slot_taken`, `invalid_transition`, `in_past`, `not_found` (id referenciado inexistente), `overlap`.

## Mapeo HU → errores críticos

| HU | Casos de error que deben tener prueba de contrato |
|---|---|
| US-01 | 422 campos vacíos/formato, 409 `EMAIL_ALREADY_REGISTERED` |
| US-02 | 401 `INVALID_CREDENTIALS`, 403 `ACCOUNT_INACTIVE`, 403 `TENANT_SUSPENDED`, 401 `TOKEN_EXPIRED`, 403 `FORBIDDEN` por rol |
| US-03 | 422 precio negativo/duración fuera de rango, 409 `SERVICE_NAME_DUPLICATED`, 404 servicio de otro tenant |
| US-04 | 422 `service_ids` de otro tenant (`details.issue = not_found`), 409 `RESOURCE_IN_USE` |
| US-05 | 422 `SCHEDULE_OVERLAP`, 422 `end_time <= start_time`, 409 `BLOCK_OVERLAPS_APPOINTMENTS` |
| US-06 | 422 `BOOKING_IN_PAST`, 422 `STAFF_NOT_QUALIFIED`, 404 slug inexistente |
| US-07 | 409 `SLOT_UNAVAILABLE` (doble reserva concurrente), 422 `OUTSIDE_WORKING_HOURS` |
| US-08 | 403 `worker` filtrando por otro `staff_id`, 409 `INVALID_STATUS_TRANSITION` |
| US-09 | 403 `CANCEL_TOKEN_INVALID`, 409 cancelar cita ya atendida/cancelada |
