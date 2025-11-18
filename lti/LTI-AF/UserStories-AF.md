# UserStories-AF

Documento: User Stories iniciales, backlog prioritizado y tickets técnicos para LTI-AF.

Fecha: 2025-11-17

---

## Resumen
Este documento recoge 10 user stories prioritarias derivadas del PRD (`LTI-AF.md`), el backlog ordenado por criticidad y los tickets técnicos detallados para la User Story priorizada: "Crear y publicar oferta".

Supuestos razonables:
- El sistema es multi-tenant. Todas las operaciones relevantes reciben `tenantId` en el contexto.
- Autenticación y autorización (IdP y RBAC) existen como servicios; los endpoints deberán validar permisos.
- Se usa PostgreSQL como BD principal y patrón Outbox para eventos.

---

## Plantilla usada para cada User Story
- ID: 
- Título:
- Rol (Quién):
- Objetivo (Qué):
- Beneficio (Por qué):
- Criterios de aceptación (AC):
- Prioridad (Critical / High / Medium / Low):
- Estimación: (Puntos Fibonacci / Horas aproximadas)
- Notas técnicas rápidas:

---

## 10 User Stories (detalladas)

### US-01
- ID: US-01
- Título: Crear y publicar una oferta de trabajo
- Rol: Recruiter (reclutador)
- Objetivo: Poder crear una oferta con campos obligatorios (título, descripción, ubicación, tipo contrato, criterios mínimos) y publicarla en los job boards seleccionados.
- Beneficio: Permite iniciar el flujo de adquisición de candidatos, exponiendo la vacante en canales externos.
- Criterios de aceptación:
  1. El recruiter autenticado con permiso `jobs:create` puede crear una oferta vía API o UI.
  2. Validaciones presentes: título no vacío, descripción >= 100 caracteres, ubicación, al menos 1 skill requerido.
  3. La oferta se guarda en la BD con estado `draft` o `published` según elección.
  4. Si se solicita publicación, se encola un evento `JobPublished` en outbox y se intenta publicar en adaptadores configurados; cualquier fallo parcial deja la oferta en `published_with_warnings` y registra el error en `EventLog`.
  5. Se devuelve ID de la oferta y ruta a la oferta en la plataforma.
- Prioridad: Critical
- Estimación: 8 puntos (Fibonacci) / 40 horas
- Notas técnicas rápidas: Endpoint POST `/api/v1/tenants/{tenantId}/jobs`; Domain event `JobCreated` + `JobPublished`; uso Outbox.

---

### US-02
- ID: US-02
- Título: Recibir aplicaciones y procesar CV automáticamente
- Rol: Sistema (ingesta de candidaturas)
- Objetivo: Cuando un candidato aplica, recoger CV, normalizar y extraer datos (nombre, email, skills, experiencia) mediante el motor de IA.
- Beneficio: Reduce el trabajo manual de extracción y permite pre-ranking automático.
- Criterios de aceptación:
  1. Aplicaciones pueden llegar vía formulario o webhook desde job boards.
  2. El documento de CV se sube a File Storage y se asocia a la aplicación.
  3. Un worker procesa el CV e inserta el resultado `ParsedResume` con score inicial.
  4. Si el CV no es parseable, la aplicación queda en `needs_manual_review` y se notifica al recruiter.
- Prioridad: Critical
- Estimación: 13 puntos / 65 horas
- Notas técnicas rápidas: Integración con IA engine (microservicio o Lambda); tasks idempotentes; retries; storage S3-compatible.

---

### US-03
- ID: US-03
- Título: Screening automático y ranking de candidatos
- Rol: Sistema (Screening engine) / Recruiter
- Objetivo: Ejecutar reglas y modelos ML para generar un score por candidato y ordenar la lista.
- Beneficio: Facilita selección rápida de candidatos más aptos.
- Criterios de aceptación:
  1. Existe un endpoint GET `/applications?jobId=&sort=score` que devuelve aplicaciones ordenadas.
  2. El score se recalcula al procesar el CV o cuando cambian criterios.
  3. Explicabilidad mínima: se muestran 3 factores que más impactaron el score.
- Prioridad: High
- Estimación: 8 puntos / 40 horas
- Notas técnicas rápidas: ElasticSearch o similar para búsqueda y ranking; modelo ML básico inicial con características estáticas.

---

### US-04
- ID: US-04
- Título: Dashboard de candidatos para una oferta
- Rol: Recruiter / Hiring Manager
- Objetivo: Ver la lista de candidatos, estados (applied, screening, interview, offer), filtros y acciones rápidas.
- Beneficio: Mejor ergonomía y rapidez en la toma de decisiones.
- Criterios de aceptación:
  1. UI lista candidatos con filtros por estado, score, fuente y etiquetas.
  2. Paginación y orden por fecha o score.
  3. Acciones: marcar para entrevista, rechazar, solicitar feedback.
- Prioridad: High
- Estimación: 5 puntos / 25 horas
- Notas técnicas rápidas: Frontend Next.js + llamadas a API paginadas; autorización RBAC.

---

### US-05
- ID: US-05
- Título: Programar entrevistas automáticamente con calendario integrado
- Role: Hiring Manager / Candidate
- Objetivo: Proponer y confirmar slots disponibles consultando calendarios externos (Google/Outlook) y actualizar la agenda.
- Beneficio: Reduce fricción en coordinación y acelera procesos.
- Criterios de aceptación:
  1. Seleccionar candidato(s), proponer slots y enviar invitación.
  2. Confirmación del candidato actualiza `Interview` y el calendario.
  3. Soporte para recordatorios y cancelaciones.
- Prioridad: High
- Estimación: 8 puntos / 40 horas
- Notas técnicas rápidas: Adapter para Google Calendar y Microsoft Graph; token per user; conflict policy.

---

### US-06
- ID: US-06
- Título: Capturar y consolidar feedback de entrevistas
- Rol: Interviewer / Hiring Manager
- Objetivo: Permitir a los entrevistadores enviar feedback estructurado (scores + comentarios) tras cada entrevista.
- Beneficio: Toma de decisiones basada en evidencia y trazabilidad.
- Criterios de aceptación:
  1. Existe formulario de feedback asociable a una `InterviewId`.
  2. El feedback se agrega al timeline de la aplicación y afecta a la decisión final.
  3. Notificaciones a hiring manager cuando todos los entrevistadores suben feedback.
- Prioridad: Medium
- Estimación: 3 puntos / 15 horas
- Notas técnicas rápidas: DB table `Feedback`, API POST `/interviews/{id}/feedback`.

---

### US-07
- ID: US-07
- Título: Generar oferta laboral y workflow de aprobaciones
- Rol: Recruiter / Hiring Manager / HR Admin
- Objetivo: Generar una oferta (salario, beneficios), enviar para aprobaciones y, al aceptar, firmar digitalmente.
- Beneficio: Cierra ciclo de contratación y automatiza aprobaciones.
- Criterios de aceptación:
  1. Crear oferta asociada a una `ApplicationId` con estado `pending_approvals`.
  2. Notificar a aprobadores según flujo configurado; permitir aprobar/rechazar con comentarios.
  3. Permitir firma electrónica (integración con proveedor o placeholder de demo) y marcar `hired`.
- Prioridad: High
- Estimación: 13 puntos / 65 horas
- Notas técnicas rápidas: Workflow de aprobaciones (simple en primera versión), outbox events `OfferCreated`, `OfferApproved`, `OfferSigned`.

---

### US-08
- ID: US-08
- Título: Notificaciones multi-canal y plantillas
- Rol: Sistema / Recruiter
- Objetivo: Enviar notificaciones por Email y Web (in-app) con plantillas reutilizables (confirmación de aplicación, invitación a entrevista, oferta).
- Beneficio: Comunicación consistente y trazable.
- Criterios de aceptación:
  1. Plantillas configurables por tenant.
  2. Se envía email cuando se crea una entrevista o se recibe una aplicación (según preferencia del tenant).
  3. Failures en envíos quedan en cola para reintentos y registran `EventLog`.
- Prioridad: Medium
- Estimación: 5 puntos / 25 horas
- Notas técnicas rápidas: Adapter SendGrid/SES; Notification service con retries.

---

### US-09
- ID: US-09
- Título: Control de acceso RBAC por tenant
- Rol: Admin / Sistema
- Objetivo: Definir roles (ADMIN, RECRUITER, INTERVIEWER) y permisos para endpoints y UI.
- Beneficio: Seguridad y separación de responsabilidades.
- Criterios de aceptación:
  1. CRUD de roles y asignación a usuarios por tenant.
  2. Middleware que valida permiso por endpoint.
  3. Al menos las rutas principales (`jobs:create`, `applications:read`, `offers:approve`) protegidas.
- Prioridad: Critical
- Estimación: 8 puntos / 40 horas
- Notas técnicas rápidas: Integración con IdP y cache de permisos (Redis).

---

### US-10
- ID: US-10
- Título: Auditoría y derecho al olvido (GDPR)
- Rol: Data Protection Officer / Candidate
- Objetivo: Registrar eventos auditables y permitir solicitudes de borrado (erasure) que obedezcan a reglas de retención.
- Beneficio: Cumplimiento legal y confianza.
- Criterios de aceptación:
  1. Todas las acciones críticas (create job, apply, update status, approve offer) generan `AuditEvent` persistente.
  2. Endpoint para solicitar borrado de candidato y procedimiento que anonimiza/borra datos dependiendo de retención.
  3. Reporte de auditoría accesible a admin con filtros por fecha y tipo evento.
- Prioridad: High
- Estimación: 8 puntos / 40 horas
- Notas técnicas rápidas: Event store + archiving; process de erasure asíncrono con estado.

---

## Backlog priorizado (urgencia por criticidad para el sistema)
Ordenado de mayor a menor prioridad (razón breve).

1. US-01 - Crear y publicar oferta (Critical) — pieza fundamental para iniciar el flujo.
2. US-09 - Control RBAC (Critical) — seguridad básica para exponer APIs.
3. US-02 - Recibir aplicaciones y procesar CV (Critical) — sin ingestión no hay candidatos.
4. US-03 - Screening y ranking (High) — reduce esfuerzo manual.
5. US-07 - Generar oferta y approvals (High) — completa ciclo de contratación.
6. US-10 - Auditoría y GDPR (High) — cumplimiento y trazabilidad.
7. US-05 - Programar entrevistas (High) — coordinación con calendarios.
8. US-04 - Dashboard de candidatos (High) — visibilidad operativa.
9. US-08 - Notificaciones multi-canal (Medium) — comunicación.
10. US-06 - Captura de feedback (Medium) — mejora decisiones, menos crítico al inicio.

Razonamiento: Las tres primeras (jobs creation, RBAC, ingest) son imprescindibles para que el flujo básico funcione y sean expuestas sin riesgo. Screening y oferta cierran ciclo de valor. Auditoría/GDPR es necesario por requisitos legales (alto). Calendarios y UI se priorizan tras core.

---

## Selección para tickets técnicos
He elegido la User Story prioritaria: **US-01 - Crear y publicar una oferta**. A continuación se detallan los tickets técnicos y las tareas que se discutirían en una planificación.

### Epic: Publicación de ofertas (EPIC-JOBS-01)
Descripción: Implementar el flujo de creación y publicación de ofertas, incluyendo API, dominio, persistencia, validaciones y publicación hacia job boards vía adaptadores.

Tickets:

1) TASK-JOBS-001 — API: Crear Oferta (Backend)
- Tipo: Backend / API
- Descripción: Implementar endpoint REST POST `/api/v1/tenants/{tenantId}/jobs` que crea una oferta.
- Subtareas:
  - Definir DTO `CreateJobDTO` (title, description, location, skills[], employmentType, remote, salaryRange?, status)
  - Validaciones: required fields, length checks.
  - Controller -> Application Service -> Domain Service -> JobRepository.
  - Persistir en tabla `jobs`.
  - Generar Domain Event `JobCreated` y persistir en Outbox.
- API response: 201 Created con payload { jobId, status }
- AC (aceptación): Las validaciones fallan con 400; usuario con permiso `jobs:create` puede crear; job aparece en DB con status `draft` o `published` según body.
- Dependencias: Auth middleware, JobRepository interface.
- Estimación: 3 puntos / 16 horas

2) TASK-JOBS-002 — DB: Esquema para `jobs`
- Tipo: Infraestructura / DB
- Descripción: Crear migración para tabla `jobs` con columnas: id(uuid PK), tenant_id, created_by, title, description, location, skills (jsonb), employment_type, remote(boolean), salary_min, salary_max, status(enum), published_at nullable, created_at, updated_at.
- AC: Migración aplicable, índice por tenant_id + status.
- Estimación: 1 punto / 4 horas

3) TASK-JOBS-003 — Domain: Entidad Job y reglas de negocio
- Tipo: Backend / Domain
- Descripción: Implementar entidad `Job` con invariantes (p. ej. no publicar sin título), métodos `publish()` que cambian estado y emiten `JobPublished`.
- AC: Unit tests que validan reglas (crear con título vacío lanza error, `publish()` cambia state y sets `published_at`).
- Estimación: 2 puntos / 8 horas

4) TASK-JOBS-004 — Outbox y EventPublishing
- Tipo: Infraestructura
- Descripción: Persistir eventos en tabla `outbox_events` dentro de la misma transacción; worker que lee outbox y publica a MQ o adapters.
- AC: Evento JobPublished aparece en outbox y luego transiciona a `processed` luego de publish attempt.
- Estimación: 3 puntos / 12 horas

5) TASK-JOBS-005 — Adapter: Publicación a Job Boards (Mock + Interface)
- Tipo: Integration
- Descripción: Definir interfaz `IJobBoardAdapter` y una implementación inicial `MockJobBoardAdapter` que simula publicación. Planificar `LinkedInAdapter`/`IndeedAdapter` como siguiente iteración.
- AC: Cuando se publica la oferta, el adapter es invocado asíncronamente por worker; el mock responde ok y se registra `published_at`.
- Estimación: 2 puntos / 8 horas

6) TASK-JOBS-006 — Frontend: Formulario Crear Oferta (Esqueleto)
- Tipo: Frontend
- Descripción: Implementar screen en Next.js para crear la oferta con validaciones del cliente y llamado al endpoint. Mostrar estados de envío.
- AC: Recruiter puede completar y enviar formulario; errores del backend se muestran.
- Estimación: 3 puntos / 12 horas

7) TASK-JOBS-007 — Tests: Integración y E2E básicos
- Tipo: Tests
- Descripción: Tests integrados para flujo crear -> outbox -> worker -> adapter (mock). E2E de alto nivel con API y DB.
- AC: Pipeline local puede ejecutar tests y pasan.
- Estimación: 3 puntos / 12 horas

8) TASK-JOBS-008 — Docs y Postman / OpenAPI
- Tipo: Docs
- Descripción: Documentar spec OpenAPI del POST /jobs y ejemplos; agregar a colección de Postman/Insomnia.
- AC: Endpoint documentado y probada con ejemplos.
- Estimación: 1 punto / 4 horas

9) TASK-JOBS-009 — Seguridad: autorización y validación tenant
- Tipo: Security
- Descripción: Asegurar middleware que valida `tenantId` y permisos; registrar auditoría `AuditEvent` para job create/publish.
- AC: Requests sin permiso devuelven 403; audit log entry created.
- Estimación: 2 puntos / 8 horas

10) TASK-JOBS-010 — Observabilidad y métricas
- Tipo: Infra / Ops
- Descripción: Añadir métricas básicas (jobs_created_total, jobs_published_total, job_publish_errors) y logs estructurados.
- AC: Métricas visibles en /metrics (Prometheus) y logs JSON con correlationId.
- Estimación: 1 punto / 4 horas

---

## Plan de ejecución (sprint hipotético de 2 semanas)
- Día 1: Setup entorno, DB migrations (TASK-JOBS-002), scaffolding domain (TASK-JOBS-003)
- Día 2-3: Endpoint API básico (TASK-JOBS-001) y middleware auth (TASK-JOBS-009)
- Día 4: Outbox (TASK-JOBS-004)
- Día 5: Adapter mock e integración (TASK-JOBS-005)
- Día 6-7: Frontend formulario (TASK-JOBS-006)
- Día 8-9: Tests e2e (TASK-JOBS-007)
- Día 10: Docs y métricas (TASK-JOBS-008, TASK-JOBS-010)

Dependencias críticas: Auth/IdP y acceso a storage/queue para worker.

---

## Estimaciones resumen (por ticket) — Fibonacci y horas
- TASK-JOBS-001: 3 pts / 16h
- TASK-JOBS-002: 1 pts / 4h
- TASK-JOBS-003: 2 pts / 8h
- TASK-JOBS-004: 3 pts / 12h
- TASK-JOBS-005: 2 pts / 8h
- TASK-JOBS-006: 3 pts / 12h
- TASK-JOBS-007: 3 pts / 12h
- TASK-JOBS-008: 1 pts / 4h
- TASK-JOBS-009: 2 pts / 8h
- TASK-JOBS-010: 1 pts / 4h

Total epic (sum estimado): 21 pts / ~88 horas (equivalente aprox. 2+ devs en 1 sprint)

---

## Criterios de Done (Definition of Done) para US-01
- Código revisado con PR y al menos 1 reviewer.
- Tests unitarios y de integración cubren la lógica principal.
- Migraciones aplicadas sin errores.
- Endpoint documentado en OpenAPI.
- Evento `JobPublished` persistido en outbox y procesado por worker en ambiente de desarrollo (mock adapter).
- Métricas mínimas y logs añadidos.

---

## Notas finales y siguientes pasos
- Siguiente User Story recomendada: US-09 (RBAC) paralela a US-01 para exponer el API con seguridad.
- Recomendación técnica: Empezar con adaptadores mocks y desplazar integraciones reales (LinkedIn/Indeed) a iteraciones posteriores.
- Recomendación de entrega: Hacer un spike rápido de 1 día para confirmar la estrategia de Outbox y worker (e.g., usar RabbitMQ o un worker simple que publique a HTTP).

---

Document created and ready in repository.
