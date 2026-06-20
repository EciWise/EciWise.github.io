---
layout: default
title: Tutoring Service
---

# Microservicio de Tutorías Académicas

El microservicio `tutoring` gestiona la oferta y reserva de tutorías académicas de ECIWise. Permite que los tutores publiquen disponibilidad recurrente, materializa esa disponibilidad en slots concretos y ofrece a los estudiantes búsqueda, reserva, cancelación y reprogramación con control transaccional de cupos.

> Esta página describe el comportamiento confirmado en el código de `EciWise/tutoring`. El esquema Prisma se considera soporte de datos; una tabla sin controlador ni caso de uso no se presenta como funcionalidad disponible.

## Capacidades actuales

- Catálogos de materias, salas, franjas horarias y asignaciones tutor–materia.
- Plantillas recurrentes de disponibilidad por tutor, materia, modalidad, cupos y vigencia.
- Materialización automática e idempotente de tutorías dentro de una ventana móvil.
- Búsqueda de tutorías programadas con cupo por materia, modalidad, fecha y tutor.
- Reserva, cancelación y reprogramación atómica, incluida la cancelación de la tutoría por tutor o administrador.
- Espejo local de usuarios alimentado de forma perezosa con los claims del JWT.
- Eventos de dominio dentro del proceso y documentación OpenAPI en `/api/docs`.

## Stack y estructura

| Área | Implementación |
|---|---|
| Runtime | Node.js 20+, TypeScript estricto |
| API | NestJS 11, REST, Swagger/OpenAPI |
| Persistencia | Prisma 7, `@prisma/adapter-pg`, PostgreSQL/Neon |
| Seguridad | Passport JWT, HS256, guards de autenticación y roles |
| Procesos programados | `@nestjs/schedule` |
| Eventos | `@nestjs/event-emitter`, publisher in-memory |
| Pruebas | Jest, dominio puro y casos de uso con repositorios in-memory |

El código sigue **arquitectura hexagonal, vertical slicing y DDD**. Cada capacidad contiene dominio, aplicación y adaptadores; las dependencias apuntan hacia los puertos y el dominio, no hacia Prisma o NestJS.

```text
src/
├── auth/                 JWT, guards, roles y decorators
├── config/               validación de entorno
├── shared/               kernel de dominio e infraestructura común
└── modules/
    ├── identidad/        espejo local de usuarios
    ├── catalogos/        materias, salas, franjas y tutor–materia
    ├── disponibilidad/   plantillas y materialización
    ├── tutorias/         búsqueda y detalle de slots
    └── reservas/         reservar, cancelar y reprogramar
```

## Arquitectura C4

Los diagramas se publican como SVG porque el sitio de GitHub Pages usa Jekyll/Kramdown y no procesa Mermaid en el navegador. Se puede abrir cada imagen para verla a tamaño completo.

### Nivel 1 — Contexto

[![C4 contexto del microservicio de Tutorías](tutoring/c4-contexto.svg)](tutoring/c4-contexto.svg)

El servicio `auth` emite el JWT, pero `tutoring` no lo consulta por red durante cada request: valida firma, expiración y claims localmente con el secreto HS256 compartido. PostgreSQL/Neon es la única dependencia de datos activa.

### Nivel 2 — Contenedores

[![C4 contenedores del microservicio de Tutorías](tutoring/c4-contenedores.svg)](tutoring/c4-contenedores.svg)

La API, el scheduler y el bus de eventos viven en el mismo proceso Node.js. `EventEmitter2` no es un broker externo y RabbitMQ todavía no tiene un adaptador activo. Prisma conecta la aplicación con PostgreSQL mediante `adapter-pg`.

### Nivel 3 — Componentes

[![C4 componentes del microservicio de Tutorías](tutoring/c4-componentes.svg)](tutoring/c4-componentes.svg)

Los controladores y el job son adaptadores de entrada. Los casos de uso coordinan entidades y puertos; los repositorios Prisma y el publisher in-memory son adaptadores de salida. Esto mantiene el dominio TypeScript independiente de NestJS y Prisma.

## Flujos críticos

### Materialización de disponibilidad

1. El tutor publica una `DisponibilidadTutor` activa sobre una franja institucional.
2. El cron diario de las **01:00** o el endpoint administrativo ejecuta `MaterializarVentana`.
3. El caso de uso calcula las fechas compatibles con día de semana, vigencia y ventana configurada.
4. Por cada fecha crea una `Tutoria` en estado `PROGRAMADA`.
5. La restricción única `(tutorUserId, franjaId, fecha)` evita duplicados; los conflictos se cuentan como omitidos.
6. Cada creación emite `tutoria.materializada` dentro del proceso.

El proceso es idempotente. No existe lock distribuido: varias réplicas pueden repetir trabajo, pero la unicidad de base de datos evita duplicar tutorías.

### Reserva y concurrencia

1. Un estudiante autenticado solicita `POST /reservas`.
2. El caso de uso comprueba existencia, estado, traslapes y reserva previa.
3. PostgreSQL incrementa `cupos_ocupados` solo si la tutoría sigue programada y tiene capacidad.
4. Dentro de la misma transacción se crea o reactiva el `Participante`.
5. Si ninguna fila puede actualizarse, la API responde con conflicto `409`.

La reprogramación ocupa primero el destino y cancela el origen dentro de una única transacción; cualquier fallo revierte ambas operaciones.

## Modelo de datos

Fuente de verdad: `prisma/schema.prisma`.

[![Modelo entidad–relación de Tutorías](tutoring/modelo-er.svg)](tutoring/modelo-er.svg)

El esquema contiene 12 modelos. `UsuarioLocal`, `ReputacionTutor` y `ReputacionEstudiante` usan identificadores provenientes del sistema de identidad y no tienen claves foráneas hacia los demás modelos. Las evaluaciones y reputaciones existen en el esquema, pero todavía no cuentan con API ni casos de uso activos.

| Núcleo | Responsabilidad |
|---|---|
| `Materia`, `Sala`, `FranjaHoraria` | Catálogos institucionales |
| `TutorMateria` | Autorización de un tutor para una materia |
| `DisponibilidadTutor` | Plantilla recurrente y vigente |
| `Tutoria` | Slot materializado con modalidad, estado y capacidad |
| `Participante` | Reserva y estado del estudiante |
| `UsuarioLocal` | Últimos datos observados en el JWT |
| Evaluaciones y reputaciones | Soporte de datos para una fase posterior |

## Autenticación y autorización

Todas las rutas funcionales, salvo `GET /`, requieren `Authorization: Bearer <token>`.

Claims consumidos:

```json
{
  "sub": "uuid-del-usuario",
  "email": "usuario@escuelaing.edu.co",
  "nombre": "Ada",
  "apellido": "Lovelace",
  "rol": "estudiante"
}
```

Los roles reconocidos son `estudiante`, `tutor` y `admin`. `JwtAuthGuard` autentica y `RolesGuard` aplica los roles declarados por cada handler. El interceptor de identidad actualiza `usuario_local` en segundo plano con el último JWT observado; un fallo de esa captura no bloquea la respuesta principal.

## API

La especificación ejecutable está disponible en `http://localhost:<PORT>/api/docs`.

| Módulo | Método y ruta | Acceso | Propósito |
|---|---|---|---|
| Salud | `GET /` | Público | Respuesta básica del servicio |
| Identidad | `GET /identidad/me` | Autenticado | Claims normalizados |
| Identidad | `GET /identidad/usuarios/:userId` | Autenticado | Usuario del espejo local |
| Materias | `POST /catalogos/materias` | Admin | Crear materia |
| Materias | `GET /catalogos/materias` | Autenticado | Listar materias |
| Materias | `PATCH /catalogos/materias/:id/activar` | Admin | Activar materia |
| Materias | `PATCH /catalogos/materias/:id/desactivar` | Admin | Desactivar materia |
| Salas | `POST /catalogos/salas` | Admin | Crear sala |
| Salas | `GET /catalogos/salas` | Autenticado | Listar salas |
| Franjas | `POST /catalogos/franjas` | Admin | Crear franja |
| Franjas | `GET /catalogos/franjas` | Autenticado | Listar por día opcional |
| Tutor–materia | `POST /catalogos/tutor-materias` | Admin | Asignar materia |
| Tutor–materia | `GET /catalogos/tutor-materias` | Autenticado | Listar asignaciones |
| Tutor–materia | `PATCH .../:id/autorizar` | Admin | Autorizar asignación |
| Tutor–materia | `PATCH .../:id/desautorizar` | Admin | Desautorizar asignación |
| Disponibilidad | `POST /disponibilidad` | Tutor, admin | Publicar plantilla recurrente |
| Disponibilidad | `GET /disponibilidad` | Tutor, admin | Listar disponibilidades |
| Disponibilidad | `PATCH /disponibilidad/:id` | Propietario, admin | Editar plantilla |
| Disponibilidad | `PATCH /disponibilidad/:id/desactivar` | Propietario, admin | Desactivar plantilla |
| Materialización | `POST /disponibilidad/materializacion` | Admin | Ejecutar ventana manualmente |
| Tutorías | `GET /tutorias` | Autenticado | Buscar slots con filtros |
| Tutorías | `GET /tutorias/:id` | Autenticado | Consultar detalle |
| Reservas | `POST /reservas` | Estudiante | Reservar cupo |
| Reservas | `POST /reservas/:tutoriaId/cancelar` | Estudiante | Cancelar reserva |
| Reservas | `POST /reservas/reprogramar` | Estudiante | Cambiar de slot atómicamente |
| Reservas | `POST /reservas/cancelacion-tutoria` | Tutor, admin | Cancelar tutoría y liberar reservas |

Filtros de búsqueda de tutorías: `materiaId`, `modalidad`, `fecha` y `tutorUserId`. Los errores de dominio se traducen principalmente a `400` (validación), `404` (no encontrado), `409` (conflicto) y `422` (regla de negocio), además de `401/403` para seguridad.

## Reglas de negocio principales

| Regla | Estado | Garantía actual |
|---|---|---|
| RN-01: el estudiante no debe traslapar tutorías | Implementada | Consulta previa al reservar o reprogramar |
| RN-02: una disponibilidad por tutor y franja | Implementada | Restricción única en PostgreSQL |
| RN-03: materia registrada y asignada | Implementada | FK y verificación de autorización |
| RN-04: solo estudiantes pueden reservar | Implementada mediante JWT | Rol `estudiante`; no existe estado local de usuario |
| RN-05: solo tutores autorizados publican | Implementada | Validación de `TutorMateria` autorizada |
| RN-06: evaluar solo tutorías realizadas | Pendiente | No hay caso de uso de evaluación |
| RN-07: una evaluación por participante | Parcial | Unicidad en Prisma, sin API |
| RN-08: cancelación con motivo | Implementada | DTO y validación de dominio |
| RN-09: no exceder cupos | Implementada | Update condicional transaccional |

RN-01 se comprueba mediante lectura previa y no con aislamiento serializable; una doble reserva concurrente del mismo estudiante sigue siendo un escenario que merece una prueba de integración específica.

## Configuración y ejecución local

Requisitos: Node.js 20+, npm, PostgreSQL 14+ y un JWT compatible con el servicio `auth`.

```bash
npm install
npx prisma generate
npx prisma migrate deploy
npm run seed
npm run start:dev
```

Variables relevantes:

| Variable | Uso |
|---|---|
| `NODE_ENV` | Entorno de ejecución |
| `PORT` | Puerto HTTP |
| `DATABASE_URL` | Conexión PostgreSQL de runtime y Prisma CLI actual |
| `DIRECT_URL` | Variable requerida por la validación de arranque |
| `JWT_SECRET` | Secreto HS256 compartido, mínimo 16 caracteres |
| `JWT_EXPIRATION` | TTL informativo |
| `MATERIALIZACION_VENTANA_SEMANAS` | Horizonte de materialización |

Comandos de verificación:

```bash
npm run build
npm test
npm run test:cov
npm run test:e2e
```

## Estado y límites conocidos

- No hay endpoints para asistencia, observaciones, evaluaciones, reputación o historial dedicado.
- El enlace virtual y la sala se leen, pero no existe un endpoint para “iniciar” una tutoría.
- Los eventos se publican en memoria; RabbitMQ y notificaciones externas son trabajo posterior.
- No existen `Dockerfile` ni `docker-compose.yml` en el servicio.
- No hay definición de despliegue de la aplicación dentro del repositorio.
- La zona horaria del cron no está configurada explícitamente.
- Editar una disponibilidad no modifica las tutorías ya materializadas.

## Fuentes técnicas

- Repositorio: [EciWise/tutoring](https://github.com/EciWise/tutoring)
- Modelo relacional: `prisma/schema.prisma`
- Composición de módulos: `src/app.module.ts` y `src/modules/*`
- Contrato HTTP: controladores y DTOs bajo `src/modules/*/infrastructure/http`
- Reglas y transacciones: casos de uso, entidades y repositorios Prisma

