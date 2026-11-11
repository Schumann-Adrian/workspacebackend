# Guía de errores · BookFlow API v1

Todas las respuestas de error (4xx/5xx) usan **el mismo cuerpo**, definido en `openapi.yaml` como `components.schemas.Error`.

```json
{
  "code": "ERROR_VALIDACION",
  "message": "La petición contiene campos no válidos.",
  "details": [
    { "field": "duracion_minutos", "issue": "fuera_de_rango", "message": "Debe estar entre 5 y 480." },
    { "field": "cliente.email",    "issue": "formato_invalido", "message": "Email no válido." }
  ],
  "trace_id": "01JC8Z6R4K2M3N4P5Q6R7S8T9V"
}
```

| Campo | Obligatorio | Descripción |
|---|---|---|
| `code` | sí | Código **estable** en `UPPER_SNAKE_CASE`. El frontend decide por `code`, nunca por `message`. |
| `message` | sí | Texto en español apto para mostrar al usuario. Puede cambiar sin previo aviso. |
| `details[]` | sí (puede ser `[]`) | Lista de problemas concretos. `field` usa notación de puntos e índices (`franjas[2].hora_fin`); `issue` en `snake_case`. |
| `trace_id` | no | Id de correlación; coincide con la cabecera `X-Request-Id` y con los logs. |

## Reglas

1. **Nunca** se devuelven trazas de pila, SQL ni nombres de tablas/restricciones al cliente. Se registran en el log con el `trace_id`.
2. **Aislamiento multi-tenant:** acceder a un recurso de otro negocio devuelve `404 NO_ENCONTRADO` (no `403`) para no revelar que existe.
3. **Login:** email inexistente y contraseña errónea devuelven el mismo `401 CREDENCIALES_INVALIDAS` (evita enumerar usuarios).
4. `400` = la petición no se puede interpretar (JSON roto, tipo incorrecto en query). `422` = se entiende pero viola reglas de validación o de negocio.
5. `409` = conflicto con el **estado actual** de los datos (duplicados, hueco ocupado, transición no permitida). Reintentar la misma petición no lo arreglará.
6. Todos los errores de validación se acumulan en `details[]` en una sola respuesta (no se para en el primero).
7. Errores de la BD se traducen en la capa de servicio: violación de `ex_citas_sin_solape` → `HUECO_NO_DISPONIBLE`; violación de unicidad → código `*_DUPLICATED` / `*_ALREADY_REGISTERED`.

## Catálogo de códigos

| HTTP | `code` | Cuándo | Endpoints típicos |
|---|---|---|---|
| 400 | `PETICION_MAL_FORMADA` | Cuerpo no es JSON válido o tipo erróneo en parámetro | Todos |
| 401 | `NO_AUTENTICADO` | Falta cabecera `Authorization` o token inválido/firmado mal | Privados |
| 401 | `TOKEN_CADUCADO` | Access token caducado → el cliente llama a `/auth/renovar-token` | Privados |
| 401 | `CREDENCIALES_INVALIDAS` | Email o contraseña incorrectos | `POST /auth/iniciar-sesion` |
| 401 | `TOKEN_REFRESCO_INVALIDO` | Refresh token caducado, revocado o reutilizado | `POST /auth/renovar-token` |
| 403 | `PROHIBIDO` | El rol no puede ejecutar la operación (p. ej. `trabajador` crea servicio) | Privados |
| 403 | `CUENTA_INACTIVA` | Usuario desactivado o pendiente de activación | `POST /auth/iniciar-sesion` |
| 403 | `NEGOCIO_SUSPENDIDO` | El negocio está suspendido por el SuperAdmin | Login y privados |
| 403 | `TOKEN_CANCELACION_INVALIDO` | `token_cancelacion` no corresponde a la cita | `PATCH /publico/citas/{id}/cancelar` |
| 404 | `NO_ENCONTRADO` | Recurso inexistente **o de otro tenant**; slug inexistente | Todos con `{id}`/`{slug}` |
| 409 | `EMAIL_YA_REGISTRADO` | Email ya usado por otro negocio/usuario | `POST /negocios/registro`, `POST /trabajadores` |
| 409 | `SLUG_EN_USO` | Slug público ya en uso | `PATCH /perfil-negocio` |
| 409 | `NOMBRE_SERVICIO_DUPLICADO` | Ya existe un servicio con ese nombre en el local | `POST/PATCH /servicios` |
| 409 | `RECURSO_EN_USO` | Borrar servicio/trabajador con citas pendientes futuras | `DELETE /servicios/{id}`, `DELETE /trabajadores/{id}` |
| 409 | `HUECO_NO_DISPONIBLE` | El hueco se ocupó entre la consulta y la reserva (carrera) | `POST /publico/negocios/{slug}/citas` |
| 409 | `TRANSICION_ESTADO_INVALIDA` | Transición no permitida (`atendida → cancelada`, `cancelada → atendida`…) | `PATCH …/atender`, `PATCH …/cancelar` |
| 409 | `BLOQUEO_SOLAPA_CITAS` | Bloqueo solapa citas pendientes (ids en `details`) | `POST /bloqueos-horario` |
| 409 | `CLAVE_IDEMPOTENCIA_REUTILIZADA` | Misma `Idempotency-Key` con cuerpo distinto | `POST …/citas` |
| 410 | `TOKEN_ACTIVACION_INVALIDO` | Token de activación caducado o ya usado | `POST /auth/activar-cuenta` |
| 422 | `ERROR_VALIDACION` | Campos obligatorios, formatos, rangos, `additionalProperties` | Todos con cuerpo/query |
| 422 | `TRABAJADOR_NO_CAPACITADO` | El trabajador no realiza ese servicio | `POST …/citas`, `GET …/disponibilidad` |
| 422 | `FUERA_DE_HORARIO` | `fecha_inicio` fuera de horario/turno o en un bloqueo | `POST …/citas` |
| 422 | `RESERVA_EN_PASADO` | Fecha/hora pasada o a más de 90 días | `GET …/disponibilidad`, `POST …/citas` |
| 422 | `HUECO_DESALINEADO` | `fecha_inicio` no encaja en la rejilla `intervalo_minutos` | `POST …/citas` |
| 422 | `HORARIO_SOLAPADO` | Franjas semanales que se solapan entre sí | `PUT /horario-local`, `PUT …/turnos` |
| 422 | `RANGO_FECHAS_INVALIDO` | `fecha_hasta < fecha_desde` o rango > 31 días | `GET /citas`, `GET /bloqueos-horario` |
| 429 | `LIMITE_PETICIONES` | Demasiadas peticiones (login: 5/min por IP+email; público: 60/min por IP) | Login, registro, públicos |
| 500 | `ERROR_INTERNO` | Error no controlado | Todos |
| 503 | `SERVICIO_NO_DISPONIBLE` | BD u otra dependencia caída | Todos, `/salud` |

### Valores habituales de `details[].issue`

`obligatorio`, `formato_invalido`, `tipo_invalido`, `demasiado_corto`, `demasiado_largo`, `fuera_de_rango`, `no_permitido` (campo extra), `duplicado`, `hueco_ocupado`, `transicion_invalida`, `en_pasado`, `no_encontrado` (id referenciado inexistente), `solapado`.

## Mapeo HU → errores críticos

| HU | Casos de error que deben tener prueba de contrato |
|---|---|
| US-01 | 422 campos vacíos/formato, 409 `EMAIL_YA_REGISTRADO` |
| US-02 | 401 `CREDENCIALES_INVALIDAS`, 403 `CUENTA_INACTIVA`, 403 `NEGOCIO_SUSPENDIDO`, 401 `TOKEN_CADUCADO`, 403 `PROHIBIDO` por rol |
| US-03 | 422 precio negativo/duración fuera de rango, 409 `NOMBRE_SERVICIO_DUPLICADO`, 404 servicio de otro tenant |
| US-04 | 422 `servicio_ids` de otro tenant (`details.issue = no_encontrado`), 409 `RECURSO_EN_USO` |
| US-05 | 422 `HORARIO_SOLAPADO`, 422 `hora_fin <= hora_inicio`, 409 `BLOQUEO_SOLAPA_CITAS` |
| US-06 | 422 `RESERVA_EN_PASADO`, 422 `TRABAJADOR_NO_CAPACITADO`, 404 slug inexistente |
| US-07 | 409 `HUECO_NO_DISPONIBLE` (doble reserva concurrente), 422 `FUERA_DE_HORARIO` |
| US-08 | 403 `trabajador` filtrando por otro `trabajador_id`, 409 `TRANSICION_ESTADO_INVALIDA` |
| US-09 | 403 `TOKEN_CANCELACION_INVALIDO`, 409 cancelar cita ya atendida/cancelada |
