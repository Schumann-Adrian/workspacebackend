# Casos mínimos de prueba de contrato por HU

Fuente: `bookflow.postman_collection.json` (esquemas de respuesta extraídos de `openapi.yaml`).

**Precondiciones:** Requiere la API levantada con un juego de datos de prueba con 2 negocios activos (Peluquería Demo y Barbería Otra), 1 negocio suspendido y los usuarios indicados abajo; los ids se configuran en las variables de la colección. API en `http://localhost:8080/api/v1`. Usuarios de prueba con contraseña `Demo1234!`: `superadmin@bookflow.test`, `admin@demo.test`, `worker@demo.test`, `admin@otra.test`, `admin@suspendido.test`.

**Ejecución:**

```bash
npx newman run docs/api/contract-tests/bookflow.postman_collection.json \
  --env-var base_url=http://localhost:8080/api/v1 --reporters cli,junit
```

Cada petición comprueba: código HTTP, `Content-Type`, esquema JSON del contrato (respuestas 2xx) o `code` + formato `{code,message,details[]}` (errores), y la regla de negocio indicada.

Las HU críticas (★) son US-02, US-06, US-07 y US-09: concentran seguridad, concurrencia y la experiencia del cliente.

## 00 · Sistema

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Health → 200 | `GET /health` | HTTP 200; Content-Type JSON |

## US-01 · Registro de negocio

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Registro válido → 201 | `POST /tenants/register` | HTTP 201; Content-Type JSON; Cumple esquema Tenant; Genera tenant_id y slug; Location |
| 2 | Email duplicado → 409 | `POST /tenants/register` | HTTP 409; Content-Type JSON; Error EMAIL_ALREADY_REGISTERED con formato {code,message,details[]} |
| 3 | Campos vacíos → 422 | `POST /tenants/register` | HTTP 422; Content-Type JSON; Cumple esquema Error; Error VALIDATION_ERROR con formato {code,message,details[]}; Lista todos los campos erróneos |
| 4 | Campo no permitido (password) → 422 | `POST /tenants/register` | HTTP 422; Content-Type JSON; Error VALIDATION_ERROR con formato {code,message,details[]} |

## US-02 · Login por roles ★

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Login admin → 200 | `POST /auth/login` | HTTP 200; Content-Type JSON; Cumple esquema AuthTokens; Rol correcto |
| 2 | Login worker → 200 | `POST /auth/login` | HTTP 200; Content-Type JSON; Cumple esquema AuthTokens; Rol correcto |
| 3 | Login superadmin → 200 | `POST /auth/login` | HTTP 200; Content-Type JSON; Cumple esquema AuthTokens; Rol correcto |
| 4 | Login admin_b → 200 | `POST /auth/login` | HTTP 200; Content-Type JSON; Cumple esquema AuthTokens; Rol correcto |
| 5 | Contraseña incorrecta → 401 | `POST /auth/login` | HTTP 401; Content-Type JSON; Error INVALID_CREDENTIALS con formato {code,message,details[]} |
| 6 | Email inexistente → 401 (mismo código) | `POST /auth/login` | HTTP 401; Content-Type JSON; Error INVALID_CREDENTIALS con formato {code,message,details[]} |
| 7 | Negocio suspendido → 403 | `POST /auth/login` | HTTP 403; Content-Type JSON; Error TENANT_SUSPENDED con formato {code,message,details[]} |
| 8 | /auth/me sin token → 401 | `GET /auth/me` | HTTP 401; Content-Type JSON; Error UNAUTHENTICATED con formato {code,message,details[]} |
| 9 | /auth/me admin → 200 | `GET /auth/me` | HTTP 200; Content-Type JSON; Cumple esquema User; tenant y rol del token |
| 10 | SuperAdmin lista negocios → 200 | `GET /tenants?page=1&limit=10` | HTTP 200; Content-Type JSON; Paginado |
| 11 | BusinessAdmin lista negocios → 403 | `GET /tenants` | HTTP 403; Content-Type JSON; Error FORBIDDEN con formato {code,message,details[]} |

## US-03 · Catálogo de servicios

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Crear servicio → 201 | `POST /services` | HTTP 201; Content-Type JSON; Cumple esquema Service |
| 2 | Nombre duplicado → 409 | `POST /services` | HTTP 409; Content-Type JSON; Error SERVICE_NAME_DUPLICATED con formato {code,message,details[]} |
| 3 | Precio negativo y duración 0 → 422 | `POST /services` | HTTP 422; Content-Type JSON; Error VALIDATION_ERROR con formato {code,message,details[]}; details por campo |
| 4 | Trabajador no puede crear → 403 | `POST /services` | HTTP 403; Content-Type JSON; Error FORBIDDEN con formato {code,message,details[]} |
| 5 | Listar paginado → 200 | `GET /services?page=1&limit=2&sort=name` | HTTP 200; Content-Type JSON; Cumple esquema ServiceList; limit respetado |
| 6 | Editar servicio → 200 | `PATCH /services/{{service_new_id}}` | HTTP 200; Content-Type JSON; Cumple esquema Service |
| 7 | Aislamiento: servicio de otro negocio → 404 | `GET /services/{{service_b_id}}` | HTTP 404; Content-Type JSON; Error NOT_FOUND con formato {code,message,details[]} |
| 8 | Eliminar servicio sin citas → 204 | `DELETE /services/{{service_new_id}}` | HTTP 204 |

## US-04 · Personal del local

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Alta trabajador con servicios → 201 | `POST /staff-members` | HTTP 201; Content-Type JSON; Cumple esquema StaffMember; Servicio asignado |
| 2 | Asignar servicio de otro negocio → 422 | `PUT /staff-members/{{staff_new_id}}/services` | HTTP 422; Content-Type JSON; Error VALIDATION_ERROR con formato {code,message,details[]} |
| 3 | Reemplazar servicios → 200 | `PUT /staff-members/{{staff_new_id}}/services` | HTTP 200; Content-Type JSON; Cumple esquema StaffMember |

## US-05 · Horario y pausas

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Horario del local → 200 | `PUT /business-hours` | HTTP 200; Content-Type JSON; Cumple esquema WeeklySlots |
| 2 | Franjas solapadas → 422 | `PUT /business-hours` | HTTP 422; Content-Type JSON; Error SCHEDULE_OVERLAP con formato {code,message,details[]} |
| 3 | Fin antes que inicio → 422 | `PUT /staff-members/{{staff_new_id}}/schedules` | HTTP 422; Content-Type JSON; Error VALIDATION_ERROR con formato {code,message,details[]} |
| 4 | Bloqueo pausa comida 1 día → 201 | `POST /time-blocks` | HTTP 201; Content-Type JSON |

## US-06 · Huecos libres ★

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Disponibilidad trabajador → 200 | `GET /public/businesses/{{slug_a}}/availability?service_id={{service_a_id}}&date={{test_date}}&staff_id={{staff_a_id}}` | HTTP 200; Content-Type JSON; Cumple esquema Availability; Hay huecos; Respeta bloqueo 13:00-14:00; Duración del servicio; Tiempo < 300 ms (RNF-03) |
| 2 | Opción 'Cualquiera' → 200 | `GET /public/businesses/{{slug_a}}/availability?service_id={{service_a_id}}&date={{test_date}}` | HTTP 200; Content-Type JSON; Cumple esquema Availability |
| 3 | Fecha pasada → 422 | `GET /public/businesses/{{slug_a}}/availability?service_id={{service_a_id}}&date=2020-01-01` | HTTP 422; Content-Type JSON; Error BOOKING_IN_PAST con formato {code,message,details[]} |
| 4 | Slug inexistente → 404 | `GET /public/businesses/no-existe/availability?service_id={{service_a_id}}&date={{test_date}}` | HTTP 404; Content-Type JSON; Error NOT_FOUND con formato {code,message,details[]} |

## US-07 · Reserva ★

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Reservar hueco 1 → 201 | `POST /public/businesses/{{slug_a}}/appointments` | HTTP 201; Content-Type JSON; Cumple esquema Confirmation; Estado pending y precio congelado |
| 2 | Mismo hueco otra vez → 409 (sin doble reserva) | `POST /public/businesses/{{slug_a}}/appointments` | HTTP 409; Content-Type JSON; Error SLOT_UNAVAILABLE con formato {code,message,details[]} |
| 3 | Fuera de horario (03:00) → 422 | `POST /public/businesses/{{slug_a}}/appointments` | HTTP 422; Content-Type JSON; Error OUTSIDE_WORKING_HOURS con formato {code,message,details[]} |
| 4 | Reservar hueco 2 → 201 | `POST /public/businesses/{{slug_a}}/appointments` | HTTP 201; Content-Type JSON; Cumple esquema Confirmation |

## US-08 · Agenda diaria

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Agenda del día (admin) → 200 | `GET /appointments?date_from={{test_date}}&date_to={{test_date}}` | HTTP 200; Content-Type JSON; Cumple esquema AppointmentList; Incluye cliente, teléfono, servicio, hora y trabajador |
| 2 | Agenda del trabajador solo propia → 200 | `GET /appointments?date_from={{test_date}}` | HTTP 200; Content-Type JSON; Cumple esquema AppointmentList; Solo sus citas |
| 3 | Marcar atendida → 200 | `PATCH /appointments/{{appt2_id}}/complete` | HTTP 200; Content-Type JSON; Cumple esquema Appointment; completed |
| 4 | Cancelar una atendida → 409 | `PATCH /appointments/{{appt2_id}}/cancel` | HTTP 409; Content-Type JSON; Error INVALID_STATUS_TRANSITION con formato {code,message,details[]} |
| 5 | Aislamiento: admin B no ve cita de A → 404 | `GET /appointments/{{appt1_id}}` | HTTP 404; Content-Type JSON; Error NOT_FOUND con formato {code,message,details[]} |

## US-09 · Cancelación ★

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Token de cancelación erróneo → 403 | `PATCH /public/appointments/{{appt1_id}}/cancel` | HTTP 403; Content-Type JSON; Error CANCEL_TOKEN_INVALID con formato {code,message,details[]} |
| 2 | Cliente cancela → 200 | `PATCH /public/appointments/{{appt1_id}}/cancel` | HTTP 200; Content-Type JSON; Cumple esquema Appointment; cancelled por cliente |
| 3 | Hueco liberado vuelve a estar disponible → 200 | `GET /public/businesses/{{slug_a}}/availability?service_id={{service_a_id}}&date={{test_date}}&staff_id={{staff_a_id}}` | HTTP 200; Content-Type JSON; slot1 libre de nuevo |
| 4 | Re-reserva del hueco liberado → 201 | `POST /public/businesses/{{slug_a}}/appointments` | HTTP 201; Content-Type JSON; Cumple esquema Confirmation |
| 5 | Local cancela → 200 | `PATCH /appointments/{{appt3_id}}/cancel` | HTTP 200; Content-Type JSON; Cumple esquema Appointment |
| 6 | Cancelar dos veces → 409 | `PATCH /appointments/{{appt3_id}}/cancel` | HTTP 409; Content-Type JSON; Error INVALID_STATUS_TRANSITION con formato {code,message,details[]} |
| 7 | Limpieza: borrar bloqueo → 204 | `DELETE /time-blocks/{{block_id}}` | HTTP 204 |

## Prueba de concurrencia (fuera de Postman)

El caso «Mismo hueco otra vez → 409» es secuencial. La carrera real (RNF-04) se cubre con un test de integración del backend que lanza **20 reservas simultáneas** al mismo `staff_id` + `starts_at` y comprueba: exactamente 1 respuesta `201`, 19 respuestas `409 SLOT_UNAVAILABLE` y 1 sola fila no cancelada en `appointments`.
