# Casos mínimos de prueba de contrato por HU

Fuente: `bookflow.postman_collection.json` (esquemas de respuesta extraídos de `openapi.yaml`).

**Precondiciones:** API levantada en `http://localhost:8080/api/v1` con un juego de datos de prueba: 2 negocios activos (Peluquería Demo y Barbería Otra), 1 negocio suspendido y usuarios por rol; los ids se configuran en las variables de la colección. Usuarios de prueba con contraseña `Demo1234!`: `superadmin@bookflow.test`, `admin@demo.test`, `worker@demo.test`, `admin@otra.test`, `admin@suspendido.test`.

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
| 1 | Health → 200 | `GET /salud` | HTTP 200; Content-Type JSON |

## US-01 · Registro de negocio

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Registro válido → 201 | `POST /negocios/registro` | HTTP 201; Content-Type JSON; Cumple esquema Negocio; Genera tenant_id y slug; Location |
| 2 | Email duplicado → 409 | `POST /negocios/registro` | HTTP 409; Content-Type JSON; Error EMAIL_YA_REGISTRADO con formato {code,message,details[]} |
| 3 | Campos vacíos → 422 | `POST /negocios/registro` | HTTP 422; Content-Type JSON; Cumple esquema Error; Error ERROR_VALIDACION con formato {code,message,details[]}; Lista todos los campos erróneos |
| 4 | Campo no permitido (password) → 422 | `POST /negocios/registro` | HTTP 422; Content-Type JSON; Error ERROR_VALIDACION con formato {code,message,details[]} |

## US-02 · Login por roles ★

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Login admin → 200 | `POST /auth/iniciar-sesion` | HTTP 200; Content-Type JSON; Cumple esquema TokensSesion; Rol correcto |
| 2 | Login worker → 200 | `POST /auth/iniciar-sesion` | HTTP 200; Content-Type JSON; Cumple esquema TokensSesion; Rol correcto |
| 3 | Login superadmin → 200 | `POST /auth/iniciar-sesion` | HTTP 200; Content-Type JSON; Cumple esquema TokensSesion; Rol correcto |
| 4 | Login admin_b → 200 | `POST /auth/iniciar-sesion` | HTTP 200; Content-Type JSON; Cumple esquema TokensSesion; Rol correcto |
| 5 | Contraseña incorrecta → 401 | `POST /auth/iniciar-sesion` | HTTP 401; Content-Type JSON; Error CREDENCIALES_INVALIDAS con formato {code,message,details[]} |
| 6 | Email inexistente → 401 (mismo código) | `POST /auth/iniciar-sesion` | HTTP 401; Content-Type JSON; Error CREDENCIALES_INVALIDAS con formato {code,message,details[]} |
| 7 | Negocio suspendido → 403 | `POST /auth/iniciar-sesion` | HTTP 403; Content-Type JSON; Error NEGOCIO_SUSPENDIDO con formato {code,message,details[]} |
| 8 | /auth/me sin token → 401 | `GET /auth/usuario-actual` | HTTP 401; Content-Type JSON; Error NO_AUTENTICADO con formato {code,message,details[]} |
| 9 | /auth/me admin → 200 | `GET /auth/usuario-actual` | HTTP 200; Content-Type JSON; Cumple esquema Usuario; tenant y rol del token |
| 10 | SuperAdmin lista negocios → 200 | `GET /negocios?page=1&limit=10` | HTTP 200; Content-Type JSON; Paginado |
| 11 | BusinessAdmin lista negocios → 403 | `GET /negocios` | HTTP 403; Content-Type JSON; Error PROHIBIDO con formato {code,message,details[]} |

## US-03 · Catálogo de servicios

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Crear servicio → 201 | `POST /servicios` | HTTP 201; Content-Type JSON; Cumple esquema Servicio |
| 2 | Nombre duplicado → 409 | `POST /servicios` | HTTP 409; Content-Type JSON; Error NOMBRE_SERVICIO_DUPLICADO con formato {code,message,details[]} |
| 3 | Precio negativo y duración 0 → 422 | `POST /servicios` | HTTP 422; Content-Type JSON; Error ERROR_VALIDACION con formato {code,message,details[]}; details por campo |
| 4 | Trabajador no puede crear → 403 | `POST /servicios` | HTTP 403; Content-Type JSON; Error PROHIBIDO con formato {code,message,details[]} |
| 5 | Listar paginado → 200 | `GET /servicios?page=1&limit=2&orden=nombre` | HTTP 200; Content-Type JSON; Cumple esquema ServicioLista; limit respetado |
| 6 | Editar servicio → 200 | `PATCH /servicios/{{servicio_nuevo_id}}` | HTTP 200; Content-Type JSON; Cumple esquema Servicio |
| 7 | Aislamiento: servicio de otro negocio → 404 | `GET /servicios/{{servicio_b_id}}` | HTTP 404; Content-Type JSON; Error NO_ENCONTRADO con formato {code,message,details[]} |
| 8 | Eliminar servicio sin citas → 204 | `DELETE /servicios/{{servicio_nuevo_id}}` | HTTP 204 |

## US-04 · Personal del local

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Alta trabajador con servicios → 201 | `POST /trabajadores` | HTTP 201; Content-Type JSON; Cumple esquema Trabajador; Servicio asignado |
| 2 | Asignar servicio de otro negocio → 422 | `PUT /trabajadores/{{trabajador_nuevo_id}}/servicios` | HTTP 422; Content-Type JSON; Error ERROR_VALIDACION con formato {code,message,details[]} |
| 3 | Reemplazar servicios → 200 | `PUT /trabajadores/{{trabajador_nuevo_id}}/servicios` | HTTP 200; Content-Type JSON; Cumple esquema Trabajador |

## US-05 · Horario y pausas

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Horario del local → 200 | `PUT /horario-local` | HTTP 200; Content-Type JSON; Cumple esquema FranjasSemanales |
| 2 | Franjas solapadas → 422 | `PUT /horario-local` | HTTP 422; Content-Type JSON; Error HORARIO_SOLAPADO con formato {code,message,details[]} |
| 3 | Fin antes que inicio → 422 | `PUT /trabajadores/{{trabajador_nuevo_id}}/turnos` | HTTP 422; Content-Type JSON; Error ERROR_VALIDACION con formato {code,message,details[]} |
| 4 | Bloqueo pausa comida 1 día → 201 | `POST /bloqueos-horario` | HTTP 201; Content-Type JSON |

## US-06 · Huecos libres ★

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Disponibilidad trabajador → 200 | `GET /publico/negocios/{{slug_a}}/disponibilidad?servicio_id={{servicio_a_id}}&fecha={{fecha_prueba}}&trabajador_id={{trabajador_a_id}}` | HTTP 200; Content-Type JSON; Cumple esquema Disponibilidad; Hay huecos; Respeta bloqueo 13:00-14:00; Duración del servicio; Tiempo < 300 ms (RNF-03) |
| 2 | Opción 'Cualquiera' → 200 | `GET /publico/negocios/{{slug_a}}/disponibilidad?servicio_id={{servicio_a_id}}&fecha={{fecha_prueba}}` | HTTP 200; Content-Type JSON; Cumple esquema Disponibilidad |
| 3 | Fecha pasada → 422 | `GET /publico/negocios/{{slug_a}}/disponibilidad?servicio_id={{servicio_a_id}}&fecha=2020-01-01` | HTTP 422; Content-Type JSON; Error RESERVA_EN_PASADO con formato {code,message,details[]} |
| 4 | Slug inexistente → 404 | `GET /publico/negocios/no-existe/disponibilidad?servicio_id={{servicio_a_id}}&fecha={{fecha_prueba}}` | HTTP 404; Content-Type JSON; Error NO_ENCONTRADO con formato {code,message,details[]} |

## US-07 · Reserva ★

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Reservar hueco 1 → 201 | `POST /publico/negocios/{{slug_a}}/citas` | HTTP 201; Content-Type JSON; Cumple esquema ConfirmacionCita; Estado pendiente y precio congelado |
| 2 | Mismo hueco otra vez → 409 (sin doble reserva) | `POST /publico/negocios/{{slug_a}}/citas` | HTTP 409; Content-Type JSON; Error HUECO_NO_DISPONIBLE con formato {code,message,details[]} |
| 3 | Fuera de horario (03:00) → 422 | `POST /publico/negocios/{{slug_a}}/citas` | HTTP 422; Content-Type JSON; Error FUERA_DE_HORARIO con formato {code,message,details[]} |
| 4 | Reservar hueco 2 → 201 | `POST /publico/negocios/{{slug_a}}/citas` | HTTP 201; Content-Type JSON; Cumple esquema ConfirmacionCita |

## US-08 · Agenda diaria

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Agenda del día (admin) → 200 | `GET /citas?fecha_desde={{fecha_prueba}}&fecha_hasta={{fecha_prueba}}` | HTTP 200; Content-Type JSON; Cumple esquema CitaLista; Incluye cliente, teléfono, servicio, hora y trabajador |
| 2 | Agenda del trabajador solo propia → 200 | `GET /citas?fecha_desde={{fecha_prueba}}` | HTTP 200; Content-Type JSON; Cumple esquema CitaLista; Solo sus citas |
| 3 | Marcar atendida → 200 | `PATCH /citas/{{cita2_id}}/atender` | HTTP 200; Content-Type JSON; Cumple esquema Cita; Estado atendida |
| 4 | Cancelar una atendida → 409 | `PATCH /citas/{{cita2_id}}/cancelar` | HTTP 409; Content-Type JSON; Error TRANSICION_ESTADO_INVALIDA con formato {code,message,details[]} |
| 5 | Aislamiento: admin B no ve cita de A → 404 | `GET /citas/{{cita1_id}}` | HTTP 404; Content-Type JSON; Error NO_ENCONTRADO con formato {code,message,details[]} |

## US-09 · Cancelación ★

| # | Caso | Petición | Verifica |
|---|---|---|---|
| 1 | Token de cancelación erróneo → 403 | `PATCH /publico/citas/{{cita1_id}}/cancelar` | HTTP 403; Content-Type JSON; Error TOKEN_CANCELACION_INVALIDO con formato {code,message,details[]} |
| 2 | Cliente cancela → 200 | `PATCH /publico/citas/{{cita1_id}}/cancelar` | HTTP 200; Content-Type JSON; Cumple esquema Cita; Cancelada por el cliente |
| 3 | Hueco liberado vuelve a estar disponible → 200 | `GET /publico/negocios/{{slug_a}}/disponibilidad?servicio_id={{servicio_a_id}}&fecha={{fecha_prueba}}&trabajador_id={{trabajador_a_id}}` | HTTP 200; Content-Type JSON; hueco1 libre de nuevo |
| 4 | Re-reserva del hueco liberado → 201 | `POST /publico/negocios/{{slug_a}}/citas` | HTTP 201; Content-Type JSON; Cumple esquema ConfirmacionCita |
| 5 | Local cancela → 200 | `PATCH /citas/{{cita3_id}}/cancelar` | HTTP 200; Content-Type JSON; Cumple esquema Cita |
| 6 | Cancelar dos veces → 409 | `PATCH /citas/{{cita3_id}}/cancelar` | HTTP 409; Content-Type JSON; Error TRANSICION_ESTADO_INVALIDA con formato {code,message,details[]} |
| 7 | Limpieza: borrar bloqueo → 204 | `DELETE /bloqueos-horario/{{bloqueo_id}}` | HTTP 204 |

## Prueba de concurrencia (fuera de Postman)

El caso «Mismo hueco otra vez → 409» es secuencial. La carrera real (RNF-04) se cubre con un test de integración del backend que lanza **20 reservas simultáneas** al mismo `trabajador_id` + `fecha_inicio` y comprueba: exactamente 1 respuesta `201`, 19 respuestas `409 HUECO_NO_DISPONIBLE` y 1 sola fila no cancelada en `citas`.
