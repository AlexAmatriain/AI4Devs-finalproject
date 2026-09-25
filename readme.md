# Plataforma de intervenciones · MVP

Documentación previa al desarrollo del MVP: descripción del producto, arquitectura, modelo de datos, especificación de la API, historias de usuario y tickets de trabajo.

## Índice

1. [Descripción general del producto](#descripción-general-del-producto)
2. [Arquitectura del sistema](#arquitectura-del-sistema)
3. [Modelo de datos](#modelo-de-datos)
4. [Especificación de la API](#especificación-de-la-api)
5. [Historias de usuario](#historias-de-usuario)
6. [Tickets de trabajo](#tickets-de-trabajo)

---

## Descripción general del producto

### Objetivo

Construir un producto mínimo viable que recoja la información cruda que publican las plataformas de los fabricantes de equipos de plantas solares, elimine el ruido y, aplicando el conocimiento del equipo técnico, genere una lista de **intervenciones** a realizar en las plantas, visible y editable desde una tabla web.

#### Problema

- Las alarmas llegan desde varias plataformas distintas, cada una con su propio formato.
- Muchas alarmas se repiten o no requieren acción.
- No existe una lista única de trabajo pendiente ni trazabilidad de lo realizado.

### Alcance

#### Incluido

- Ingesta de datos de dos plataformas externas: una de inversores (Plataforma A) y otra de contadores (Plataforma B).
- Almacenamiento de los datos crudos en una base de datos relacional y en una base de datos de series temporales.
- Generación de alarmas propias a partir de las alarmas crudas.
- Generación de intervenciones a partir de las alarmas propias.
- Tabla web para consultar y editar intervenciones, y registro de visitas (actuaciones).

#### No incluido

- Roles y permisos diferenciados.
- Notificaciones automáticas.
- Cuadros de mando analíticos.

### Usuarios

| Perfil | Uso |
|---|---|
| Responsable de mantenimiento | Revisa y prioriza las intervenciones. |
| Planificador | Asigna fechas y equipos; crea preventivos. |
| Técnico | Registra las visitas realizadas. |
| Administrador | Gestiona usuarios y supervisa los procesos. |

### Reglas de negocio principales

#### Alarmas propias

- Un catálogo define cada tipo de alarma propia (descripción, severidad, tipo de intervención y periodicidad).
- Una tabla de mapeo indica qué alarmas crudas alimentan cada alarma propia.
- Una alarma cruda repetida en varias consultas genera una sola alarma propia.
- La alarma propia se cierra cuando el origen informa del cierre o cuando deja de reportarse durante más tiempo que su periodicidad.

#### Intervenciones

- Una intervención por alarma propia; volver a ejecutar el proceso no genera duplicados.
- Tipos: correctivo inmediato, correctivo planificado y preventivo.
- Severidades: urgente, mayor y menor.
- Estados: planificada, abierta, cerrada y cancelada.

| Severidad (correctivos) | Fecha plan |
|---|---|
| Urgente | Fecha actual |
| Mayor | Activación + 15 días |
| Menor | Activación + 30 días |

| Fechas con valor | Estado |
|---|---|
| Solo fecha plan | Planificada |
| Fecha plan y activación | Abierta |
| Fecha plan, activación y desactivación | Cerrada |

Cancelada es un estado manual. Los preventivos se crean manualmente.

#### Actuaciones

- Cada visita a planta se registra como actuación de una intervención: fechas, descripción, técnicos, repuestos y fotos.
- Las actuaciones deben estar dentro del periodo de la intervención y no solaparse entre sí.
- Se pueden anular (no se borran) y guardan histórico de cambios.

### Riesgos

- Límites de peticiones de las plataformas externas.
- Algunas plataformas no informan del cierre de las alarmas.
- Relación entre contadores y plantas no siempre evidente.

### Planificación orientativa

| Fase | Contenido |
|---|---|
| 1 | Base técnica, modelo de datos y acceso |
| 2 | Sincronización de activos e ingesta |
| 3 | Alarmas propias e intervenciones |
| 4 | Aplicación web |
| 5 | Actuaciones, móvil y programación de procesos |

---

## Arquitectura del sistema

### Visión general

El sistema se compone de un backend que recoge y procesa los datos, dos bases de datos y una aplicación web. El núcleo del diseño es la **intervención**: el resto de componentes existen para alimentarla o para mostrarla.

```
Plataforma A ─┐                                         ┌─▶ Base relacional
              ├─▶ Backend (procesos programados + API) ─┤
Plataforma B ─┘                 ▲                       └─▶ Series temporales
                                │ API REST
                          Aplicación web
```

### Flujo de datos

1. **Ingesta**: se leen de la base de datos las plantas y dispositivos y se consultan las plataformas externas.
2. **Almacenamiento**: datos de eventos en base relacional; medidas periódicas en series temporales.
3. **Procesamiento**: tareas independientes, una por tipo de alarma propia.
4. **Alarmas propias**: resultado del procesamiento, sin duplicados.
5. **Intervenciones**: se crean o actualizan a partir de las alarmas propias.
6. **Exposición**: API REST consumida por la aplicación web.

### Componentes

| Elemento | Propuesta |
|---|---|
| Backend | .NET con arquitectura hexagonal (dominio, aplicación, infraestructura, API) |
| Base relacional | PostgreSQL con Entity Framework (código primero) |
| Series temporales | InfluxDB |
| Frontend | Angular |
| Seguridad | Usuario y contraseña con token |
| Despliegue | Contenedores en la nube con procesos programados |

### Capas del backend

El backend sigue una arquitectura hexagonal: las dependencias apuntan siempre hacia el dominio.

| Capa | Responsabilidad |
|---|---|
| Dominio | Entidades (planta, dispositivo, alarma, intervención, actuación) y reglas de negocio. No depende de ninguna otra capa. |
| Aplicación | Casos de uso: sincronizar activos, recoger alarmas y medidas, generar alarmas propias, generar y gestionar intervenciones, registrar actuaciones. |
| Infraestructura | Acceso a las bases de datos y clientes de las plataformas externas. |
| API | Endpoints REST, autenticación y arranque de los procesos programados. |

### Procesos programados

Cada paso del flujo es un proceso independiente que se puede ejecutar de forma programada o manualmente desde la API. Todos devuelven un resumen (creados, actualizados, errores) y son repetibles: volver a ejecutarlos no duplica datos.

| Proceso | Frecuencia orientativa |
|---|---|
| Sincronizar activos (plantas y dispositivos) | Diaria |
| Recoger alarmas de las plataformas | Cada pocos minutos |
| Recoger medidas periódicas | Diaria |
| Generar alarmas propias | Tras cada recogida de alarmas |
| Generar intervenciones | Tras generar alarmas propias |

Si el backend se ejecuta en varias instancias, los procesos se lanzan desde un planificador externo para que cada uno se ejecute una sola vez.

### Seguridad y configuración

- Acceso con usuario y contraseña; el login devuelve un token que se envía en el resto de peticiones.
- Todos los usuarios tienen el mismo nivel de acceso en el MVP.
- Las contraseñas se guardan cifradas.
- Cadenas de conexión y credenciales de las plataformas externas parametrizadas por entorno (local, producción), nunca en el repositorio.

---

## Modelo de datos

### Introducción

Primera propuesta del modelo de datos del MVP. Es un modelo de partida que se irá ajustando durante el desarrollo.

- Base de datos relacional: activos, alarmas, intervenciones y actuaciones.
- Base de datos de series temporales: medidas periódicas de los dispositivos.

Convenciones: `PK` clave primaria, `FK` clave foránea, `UK` valor único.

### Tablas

| Tabla | Contenido |
|---|---|
| USUARIOS | Usuarios de la aplicación |
| EQUIPOS | Equipos técnicos |
| PLANTAS | Plantas solares |
| DISPOSITIVOS | Inversores y contadores de cada planta |
| ALARMAS_CRUDAS | Alarmas tal como llegan de las plataformas |
| TIPOS_ALARMA | Alarmas propias definidas por el equipo técnico |
| REGLAS_ALARMA | Qué alarmas crudas corresponden a cada tipo de alarma propia |
| ALARMAS | Alarmas propias generadas |
| INTERVENCIONES | Trabajo a realizar en las plantas |
| ACTUACIONES | Visitas realizadas |
| FOTOS | Imágenes de las actuaciones |

### Detalle de tablas

#### USUARIOS

Personas con acceso a la aplicación.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| usuario | texto | UK | Nombre de acceso |
| contrasena | texto |   | Contraseña cifrada |

#### EQUIPOS

Equipos técnicos que atienden las plantas.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| nombre | texto | UK | Nombre del equipo |

#### PLANTAS

Plantas solares gestionadas.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| codigo | texto | UK | Código en la plataforma de origen |
| nombre | texto |   | Nombre de la planta |
| id_equipo | entero | FK | Equipo que la atiende |

#### DISPOSITIVOS

Inversores y contadores. Un contador puede no tener planta hasta que se vincule.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| id_planta | entero | FK | Planta a la que pertenece |
| tipo | texto |   | Inversor o contador |
| codigo | texto | UK | Número de serie o código en la plataforma |
| plataforma | texto |   | Plataforma A o Plataforma B |

#### ALARMAS_CRUDAS

Alarmas recibidas de las plataformas. Se guarda una fila por cada recogida.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| id_dispositivo | entero | FK | Dispositivo que la genera |
| plataforma | texto |   | Plataforma de origen |
| codigo_alarma | texto |   | Código de la alarma en la plataforma |
| fecha_inicio | fecha-hora |   | Inicio de la alarma |
| fecha_fin | fecha-hora |   | Fin, si la plataforma lo informa |
| fecha_recogida | fecha-hora |   | Momento en que se recogió |

#### TIPOS_ALARMA

Catálogo de alarmas propias.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| nombre | texto | UK | Nombre de la alarma propia |
| severidad | texto |   | Urgente, mayor o menor |
| tipo_intervencion | texto |   | Correctivo inmediato o planificado |
| minutos_cierre | entero |   | Minutos sin recibir la alarma para darla por cerrada |

#### REGLAS_ALARMA

Asocia códigos de alarma cruda con un tipo de alarma propia.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| id_tipo_alarma | entero | FK | Alarma propia |
| plataforma | texto |   | Plataforma de origen |
| codigo_alarma | texto |   | Código de la alarma cruda |

#### ALARMAS

Alarmas propias, sin duplicados.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| id_tipo_alarma | entero | FK | Tipo de alarma propia |
| id_dispositivo | entero | FK | Dispositivo afectado |
| fecha_inicio | fecha-hora |   | Inicio |
| fecha_fin | fecha-hora |   | Cierre (vacío si sigue abierta) |

#### INTERVENCIONES

Trabajo a realizar. Se crea desde una alarma o a mano (preventivos).

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| id_alarma | entero | FK | Alarma de origen (vacío en preventivos) |
| id_planta | entero | FK | Planta |
| tipo | texto |   | Correctivo inmediato, correctivo planificado o preventivo |
| severidad | texto |   | Urgente, mayor o menor |
| estado | texto |   | Planificada, abierta, cerrada o cancelada |
| fecha_activacion | fecha |   | Inicio |
| fecha_plan | fecha |   | Fecha planificada |
| fecha_cierre | fecha |   | Cierre |
| id_equipo | entero | FK | Equipo asignado |
| sistema | texto |   | Sistema afectado |
| descripcion | texto |   | Descripción libre |

#### ACTUACIONES

Visitas realizadas en una intervención.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| id_intervencion | entero | FK | Intervención |
| fecha_inicio | fecha-hora |   | Inicio de la visita |
| fecha_fin | fecha-hora |   | Fin de la visita |
| descripcion | texto |   | Trabajo realizado |
| tecnicos | texto |   | Técnicos que asistieron |
| repuestos | texto |   | Repuestos utilizados |
| anulada | sí/no |   | Marca de anulación |
| id_usuario | entero | FK | Usuario que la registra |

#### FOTOS

Imágenes adjuntas a las actuaciones.

| Campo | Tipo | Clave | Descripción |
|---|---|---|---|
| id | entero | PK |   |
| id_actuacion | entero | FK | Actuación |
| archivo | binario |   | Imagen |

### Series temporales

| Medida | Etiquetas | Valores |
|---|---|---|
| medidas_dispositivo | código de dispositivo, código de planta | Valores numéricos recogidos (potencia, energía, etc.) |

### Relaciones

| Relación | Cardinalidad |
|---|---|
| EQUIPOS → PLANTAS | 1 : N |
| PLANTAS → DISPOSITIVOS | 1 : N |
| DISPOSITIVOS → ALARMAS_CRUDAS | 1 : N |
| TIPOS_ALARMA → REGLAS_ALARMA | 1 : N |
| TIPOS_ALARMA → ALARMAS | 1 : N |
| ALARMAS → INTERVENCIONES | 1 : 0..1 |
| PLANTAS → INTERVENCIONES | 1 : N |
| INTERVENCIONES → ACTUACIONES | 1 : N |
| ACTUACIONES → FOTOS | 1 : N |

### Pendiente de definir

- Histórico de cambios de las actuaciones.
- Si tipos, severidades, estados y sistemas pasan a ser tablas de catálogo.
- Qué dispositivos afecta cada intervención cuando son varios.
- Qué ocurre con los registros relacionados al borrar una planta o una intervención.

---

## Especificación de la API

### Convenciones

- API REST con cuerpos en JSON y nombres de campo en camelCase.
- Todas las rutas empiezan por `/api` y requieren el token de login en la cabecera `Authorization: Bearer <token>`, salvo el login.
- Fechas en formato ISO 8601 y en UTC.
- Los listados se paginan con `page` y `pageSize`.

| Código | Significado |
|---|---|
| 200 / 201 | Operación correcta / recurso creado |
| 400 | Datos no válidos (el cuerpo indica el motivo) |
| 401 | Sin token o token no válido |
| 404 | Recurso inexistente |
| 409 | Operación no permitida en el estado actual (p. ej. editar una actuación anulada) |

### Autenticación

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/api/auth/login` | Recibe usuario y contraseña; devuelve el token |
| POST | `/api/usuarios` | Da de alta un usuario (solo administrador) |

```json
POST /api/auth/login
{ "usuario": "tecnico1", "contrasena": "••••••" }

200 OK
{ "token": "eyJhbGciOi...", "caduca": "2026-10-01T20:00:00Z" }
```

### Catálogos

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/plantas` | Plantas con su equipo asignado |
| GET | `/api/equipos` | Equipos técnicos |
| GET | `/api/plantas/{id}/dispositivos` | Dispositivos de una planta |

### Procesos

Permiten lanzar manualmente cada paso del flujo. Los que consultan las plataformas aceptan opcionalmente `desde` y `hasta`.

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/api/procesos/sincronizar-activos` | Sincroniza plantas y dispositivos |
| POST | `/api/procesos/recoger-alarmas?desde=&hasta=` | Recoge alarmas de las plataformas |
| POST | `/api/procesos/recoger-medidas?desde=&hasta=` | Recoge medidas periódicas |
| POST | `/api/procesos/generar-alarmas` | Genera alarmas propias |
| POST | `/api/procesos/generar-intervenciones` | Genera y actualiza intervenciones |

```json
200 OK
{ "creados": 12, "actualizados": 3, "errores": [] }
```

### Intervenciones

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/intervenciones` | Listado con filtros |
| GET | `/api/intervenciones/{id}` | Detalle |
| POST | `/api/intervenciones` | Crea un preventivo |
| PATCH | `/api/intervenciones/{id}` | Modifica solo los campos enviados |

Filtros del listado: `estado`, `severidad`, `tipo`, `idPlanta`, `idEquipo`, `texto` (planta o alarma), `page`, `pageSize`.

```json
GET /api/intervenciones/42

200 OK
{
  "id": 42,
  "planta": { "id": 7, "nombre": "Planta Norte" },
  "alarma": { "id": 130, "nombre": "Parada de producción" },
  "tipo": "CORRECTIVO_INMEDIATO",
  "severidad": "URGENTE",
  "estado": "ABIERTA",
  "fechaActivacion": "2026-09-20",
  "fechaPlan": "2026-09-20",
  "fechaCierre": null,
  "equipo": { "id": 2, "nombre": "Equipo Sur" },
  "sistema": "Inversor",
  "descripcion": null
}
```

Creación de un preventivo: `idPlanta` y `fechaPlan` son obligatorios; `fechaActivacion`, `fechaCierre`, `idEquipo` y `descripcion` son opcionales. El tipo (preventivo) y el estado (calculado a partir de las fechas) los asigna el sistema.

```json
POST /api/intervenciones
{ "idPlanta": 7, "fechaPlan": "2026-10-15", "idEquipo": 2, "descripcion": "Limpieza de paneles" }

201 Created   (cuerpo con el mismo formato que el detalle)
```

Edición: se pueden enviar `estado`, `severidad`, `fechaPlan`, `fechaCierre`, `idEquipo`, `sistema` y `descripcion`; `fechaActivacion` solo en preventivos. Un campo con `null` vacía su valor y un campo que no se envía no cambia. Si el estado no es coherente con las fechas, responde `400`.

```json
PATCH /api/intervenciones/42
{ "estado": "CERRADA", "fechaCierre": "2026-09-22" }
```

### Actuaciones

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/intervenciones/{id}/actuaciones` | Actuaciones de una intervención |
| POST | `/api/intervenciones/{id}/actuaciones` | Registra una actuación |
| PUT | `/api/actuaciones/{id}` | Modifica una actuación |
| POST | `/api/actuaciones/{id}/anular` | Anula una actuación (no se borra) |
| POST | `/api/actuaciones/{id}/fotos` | Adjunta fotos (`multipart/form-data`) |
| GET | `/api/fotos/{id}` | Descarga una foto |

```json
POST /api/intervenciones/42/actuaciones
{
  "fechaInicio": "2026-09-21T08:30:00Z",
  "fechaFin": "2026-09-21T11:00:00Z",
  "descripcion": "Sustitución de fusible en inversor 3",
  "tecnicos": "Técnico 1, Técnico 2",
  "repuestos": "Fusible 15 A"
}
```

Se responde `400` si la actuación queda fuera del periodo de la intervención, si el fin es anterior al inicio o si se solapa con otra actuación no anulada.

---

## Historias de usuario

### Introducción

Historias de usuario del MVP en formato "Como… quiero… para…", con un criterio de aceptación básico cada una.

### Historias

| ID | Historia | Criterio de aceptación |
|---|---|---|
| HU-01 | **Como** usuario, **quiero** iniciar sesión con usuario y contraseña, **para** acceder de forma segura. | Con credenciales válidas accedo al panel; con credenciales erróneas no. |
| HU-02 | **Como** administrador, **quiero** dar de alta usuarios, **para** controlar quién accede. | El usuario creado puede iniciar sesión. |
| HU-03 | **Como** administrador, **quiero** que plantas, inversores y contadores se sincronicen solos, **para** no darlos de alta a mano. | Los activos nuevos aparecen tras la sincronización. |
| HU-04 | **Como** responsable de mantenimiento, **quiero** que las alarmas de los fabricantes se recojan automáticamente, **para** no revisar cada plataforma. | Las alarmas quedan guardadas tras cada ejecución. |
| HU-05 | **Como** responsable de mantenimiento, **quiero** definir alarmas propias a partir de alarmas de los fabricantes, **para** aplicar el criterio del equipo. | Una nueva alarma propia se configura solo con datos. |
| HU-06 | **Como** responsable de mantenimiento, **quiero** que una alarma repetida aparezca una sola vez, **para** evitar ruido. | Varias recogidas de la misma alarma generan una sola alarma propia. |
| HU-07 | **Como** responsable de mantenimiento, **quiero** que las alarmas propias se cierren solas, **para** saber qué se ha resuelto. | Cuando el origen deja de reportar la alarma, se cierra. |
| HU-08 | **Como** responsable de mantenimiento, **quiero** que cada alarma propia genere una intervención, **para** tener una lista de trabajo priorizada. | La intervención tiene tipo, severidad y fechas. |
| HU-09 | **Como** planificador, **quiero** que se calcule una fecha plan según la urgencia, **para** cumplir plazos. | Urgente hoy, mayor +15 días, menor +30 días. |
| HU-10 | **Como** responsable de mantenimiento, **quiero** ver todas las intervenciones en una tabla con indicadores, **para** conocer el trabajo pendiente. | Veo la tabla y los totales por estado. |
| HU-11 | **Como** responsable de mantenimiento, **quiero** filtrar y buscar intervenciones, **para** centrarme en lo relevante. | Los filtros se combinan entre sí. |
| HU-12 | **Como** planificador, **quiero** editar fechas, equipo, sistema y estado de una intervención, **para** organizar el trabajo. | No se permite guardar un estado incoherente con las fechas. |
| HU-13 | **Como** planificador, **quiero** crear intervenciones preventivas, **para** planificar el mantenimiento. | La intervención aparece en la tabla como planificada. |
| HU-14 | **Como** técnico, **quiero** registrar cada visita con fotos, **para** dejar constancia del trabajo. | La visita queda asociada a la intervención con sus fotos. |
| HU-15 | **Como** técnico, **quiero** usar la aplicación desde el móvil, **para** trabajar en planta. | Puedo consultar y registrar sin hacer zoom. |
| HU-16 | **Como** administrador, **quiero** que todo el proceso se ejecute de forma programada, **para** mantener los datos al día. | Los procesos se ejecutan solos y pueden relanzarse a mano. |

---

## Tickets de trabajo

### Introducción

Listado inicial de tickets de trabajo para construir el MVP, agrupados por bloque. Las estimaciones se harán en la sesión de planificación.

### Base técnica

| ID | Tarea | Criterio de aceptación |
|---|---|---|
| T-01 | Crear la solución backend con arquitectura hexagonal | La solución compila y respeta la dirección de dependencias. |
| T-02 | Parametrizar la configuración por entorno | Conexiones y credenciales sobrescribibles por variables de entorno. |
| T-03 | Entorno local con contenedores de bases de datos | Las dos bases de datos arrancan con un solo comando. |
| T-04 | Crear el proyecto frontend | La aplicación arranca en local y apunta a la API configurada. |
| T-05 | Pipeline de integración y despliegue | Cada cambio en la rama principal se despliega automáticamente. |

### Datos y acceso

| ID | Tarea | Criterio de aceptación |
|---|---|---|
| T-06 | Modelo de datos y migración inicial | Todas las tablas y relaciones se crean desde código. |
| T-07 | Datos iniciales de catálogos | Tipos, severidades, estados y alarmas propias iniciales cargados. |
| T-08 | Acceso a series temporales | Se pueden escribir medidas con etiquetas. |
| T-09 | Usuarios, login y protección de la API | Sin token válido la API responde 401. |

### Activos e ingesta

| ID | Tarea | Criterio de aceptación |
|---|---|---|
| T-10 | Cliente de la Plataforma A | Autenticación y reintentos ante límites de peticiones. |
| T-11 | Cliente de la Plataforma B | Autenticación y control de ritmo de peticiones. |
| T-12 | Sincronizar plantas e inversores | Crea nuevos y actualiza existentes sin borrar. |
| T-13 | Sincronizar contadores y vincularlos a plantas | Informa de los contadores no vinculados. |
| T-14 | Ingesta de alarmas de ambas plataformas | Se guardan con instante de ingesta; un fallo parcial no detiene el proceso. |
| T-15 | Ingesta de medidas a series temporales | KPIs de inversores y curvas de contadores guardados. |

### Alarmas propias e intervenciones

| ID | Tarea | Criterio de aceptación |
|---|---|---|
| T-16 | Generar alarmas propias | Según catálogo y mapeo; sin duplicados. |
| T-17 | Cierre automático de alarmas propias | Se cierran según el criterio de cada origen. |
| T-18 | Generar y actualizar intervenciones | Una por alarma propia; proceso repetible sin duplicados. |
| T-19 | Reglas de fecha plan y estado | Se cumplen las tablas de reglas del documento funcional. |
| T-20 | API de consulta, edición y creación de intervenciones | Listado, detalle, edición parcial y alta de preventivos. |

### Aplicación web

| ID | Tarea | Criterio de aceptación |
|---|---|---|
| T-21 | Pantalla de login | Sin sesión se muestra el login. |
| T-22 | Tabla de intervenciones con filtros e indicadores | Filtros combinables y estados de carga, error y vacío. |
| T-23 | Detalle y edición de intervención | Solo se envían los campos modificados; se valida estado frente a fechas. |
| T-24 | Creación de preventivos | Planta y fecha plan obligatorias. |

### Actuaciones y operación

| ID | Tarea | Criterio de aceptación |
|---|---|---|
| T-25 | Actuaciones con fotos e histórico | Validación de fechas y sin solapes; anulación sin borrado. |
| T-26 | Adaptación a móvil | Uso completo sin scroll horizontal en el móvil. |
| T-27 | Ejecución programada de procesos | Cada proceso se ejecuta una sola vez según su frecuencia. |
