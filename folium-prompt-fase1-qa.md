# Prompt #2 — Verificación y QA de Folium v1 (Fase 1, single-tenant)

> Uso: pegar todo el contenido debajo de la línea como primer prompt en una sesión de Claude Code
> abierta en la **raíz del repositorio `folium`** ya construido con el prompt #1.
> Es de un solo tiro: audita, reporta y corrige sin pedir más contexto.

---

Actúa como auditor técnico y de QA de **Folium v1**, un sistema de gestión documental para una entidad pública colombiana que acaba de ser construido en este repositorio. Tu trabajo tiene tres partes, en este orden: **(1)** verificar contra la especificación de referencia de abajo, **(2)** producir un informe de hallazgos, **(3)** corregir todo hallazgo Crítico y Alto, y re-verificar. No asumas que algo funciona porque el código "se ve bien": ejecútalo y compruébalo. No amplíes el alcance ni agregues funcionalidades nuevas.

## Especificación de referencia

Contrato que el sistema debe cumplir (resumen de la spec original; ante conflicto entre esto y `DECISIONES.md` del repo, esto manda salvo que la decisión esté justificada y no viole seguridad ni normativa):

- **Arquitectura:** binario Go único (Chi + pgx + PostgreSQL) que sirve la SPA (Vite + React 19 + TanStack Router + Zustand) vía `embed.FS`, API bajo `/api/v1`. Single-tenant: sin `tenant_id`, sin BFF, sin Redis/Valkey, sin JWT. `docker-compose.yml` con `postgres` + `app`.
- **Auth:** sesiones server-side — cookie `HttpOnly` + `SameSite`, tabla `sessions` con `token_hash` (SHA-256, nunca el token en claro), logout revoca, rate limit en login. Passwords con bcrypt. Roles: `admin`, `ventanilla`, `jefe_dependencia`, `funcionario`.
- **Radicados:** tipos `incoming|outgoing|internal`; número `YYYYMMDD-{E|S|I}-NNNNNN` generado atómicamente desde `record_counters`, secuencia por tipo con reinicio anual, sin duplicados bajo concurrencia.
- **Workflow (estados fijos):** `filed → assigned → in_progress → replied → archived` + `cancelled`. Asignar: jefe_dependencia/admin. Iniciar: solo el funcionario asignado. Responder crea radicado de salida vinculado (`outgoing_record_details.origin_record_id`). Devolver exige comentario. Toda transición escribe en `record_history` **en la misma transacción**; transición inválida → 422.
- **Vencimientos:** `due_date` en **días hábiles colombianos** (L-V menos festivos, Ley Emiliani con trasladables y fechas dependientes de Pascua). Scheduler diario 6:00 America/Bogota: alerta email "por vencer" (3 días hábiles antes) y "vencido", sin duplicar envíos.
- **PII (Ley 1581):** cédula/email del remitente solo como HMAC-SHA256 (búsqueda por hash); teléfono/dirección con AES-256-GCM; supresión (solo admin) que anonimiza sin borrar el radicado ni su historial; consentimiento en `data_consents`.
- **Adjuntos:** interfaz `Storage` con disco local; validación MIME por magic bytes; límite de tamaño por env; sha256 por archivo; descarga/visualización SOLO vía endpoint autenticado que registra en `document_access_logs` antes de servir — nunca estáticos.
- **Inmutabilidad:** `record_history`, `document_access_logs` y `audit_logs` nunca se actualizan ni borran desde la aplicación.
- **Reportes:** por período, por dependencia, tiempos de respuesta; export Excel y PDF. Sticker PDF con Code128 del número de radicado.
- **Pantallas (en español):** login, bandeja con semáforo de vencimientos, radicar entrada/salida/interna, detalle con historial + visor de adjuntos + acciones por rol, búsqueda, reportes, administración.
- **Seeds:** entidad demo, 4 dependencias, 6 usuarios (roles variados), tipos de documento con plazos reales (Derecho de petición 15, Tutela 3, Solicitud de información 10), ~20 radicados de ejemplo.

## Parte 1 — Verificación

Ejecuta cada bloque y registra el resultado (pasa / falla / parcial, con evidencia). Si el arranque mismo falla, corrígelo primero: sin sistema corriendo no hay QA.

### 1.1 Arranque limpio

Desde cero: `docker compose down -v`, luego `docker compose up --build -d`. Verifica que migraciones y seed dejan el sistema usable siguiendo solo el README (si el README omite un paso necesario, es hallazgo). Revisa logs de arranque: sin errores ni warnings sospechosos. La SPA carga en la raíz y las rutas profundas del router (ej. `/radicados/xyz`) devuelven el `index.html` (fallback embed).

### 1.2 Suites automáticas

`make test` (o equivalente), `golangci-lint run`, `go vet ./...`, y en `web/`: `tsc --noEmit` + ESLint + build de producción. Todo debe pasar. Revisa que los tests críticos EXISTEN y prueban lo que dicen: numeración concurrente (goroutines reales, no secuencial), transiciones inválidas por rol, festivos colombianos 2026-2027, HMAC/AES round-trip, supresión.

### 1.3 Flujo funcional end-to-end (vía API con curl, usando los usuarios del seed)

Guarda las cookies por rol (`curl -c/-b`) y ejecuta la cadena completa:

1. Login como **ventanilla** → radicar entrada (Derecho de petición, remitente persona natural con cédula/email/teléfono/dirección, con consentimiento) subiendo un PDF real → verifica: número con formato y fecha de hoy, `due_date` correcto en días hábiles, respuesta sin PII en claro.
2. Descargar `sticker.pdf` → es un PDF válido y contiene el número.
3. Login como **jefe** de la dependencia destino → bandeja lo muestra → asignar a un **funcionario** con comentario.
4. Login como **funcionario** → iniciar trámite → responder adjuntando PDF → verifica que se creó el radicado de salida `S` vinculado al de entrada.
5. Login como **jefe** → archivar. Consultar el detalle: el historial contiene TODOS los pasos anteriores en orden, con autores correctos.
6. Búsqueda: por número exacto, por cédula del remitente (el término en claro debe encontrar el radicado vía hash), por asunto parcial, y filtros combinados con paginación.
7. Reportes del período: los números cuadran con lo creado; descargar Excel y PDF y validar que abren.
8. Descargar el adjunto → confirma en BD que quedó fila en `document_access_logs`.
9. Como **admin**: suprimir los datos del remitente de un radicado seed → detalle anonimizado, historial intacto, radicado sigue existiendo.

### 1.4 Casos negativos y seguridad (obligatorios)

- Sin cookie: todo endpoint de datos → 401. Cookie de sesión revocada (post-logout) → 401.
- **Escalación por rol:** funcionario intenta asignar/archivar → 403. Ventanilla intenta endpoints de admin (crear usuario, suprimir PII) → 403. Jefe de la dependencia A intenta asignar un radicado de la dependencia B → 403.
- **Transiciones ilegales:** responder un radicado en `filed`, archivar uno en `in_progress`, iniciar como un funcionario NO asignado → 422/403, y `record_history` NO registra nada.
- **Adjuntos:** subir un `.exe` renombrado a `.pdf` → 415 (magic bytes, no extensión); archivo sobre el límite → 413; intentar acceder al archivo por ruta directa (`STORAGE_PATH` expuesto, path traversal `../../` en storage_key o en el download) → imposible.
- **Inyección:** SQL injection en búsqueda (`' OR 1=1 --`), XSS almacenado en `subject`/`sender_name` (verifica que el frontend lo renderiza escapado).
- **PII en reposo:** conéctate a Postgres y confirma que en `records` no hay cédulas/emails/teléfonos/direcciones legibles; que `sessions.token_hash` no es el token; que los logs de la app no imprimen PII ni cookies.
- Login: 6 intentos fallidos seguidos → rate limit responde 429 (o equivalente).
- Cookie con `HttpOnly` y `SameSite`; sin secretos hardcodeados (grep de claves, passwords y tokens en el código); `.env` fuera de git; config falla al arrancar si faltan claves críticas.

### 1.5 Reglas de negocio finas

- Días hábiles: radica un Derecho de petición (15 días) y valida `due_date` a mano contra el calendario colombiano vigente (cuenta festivos y fines de semana del rango real). Prueba una Tutela (3 días) radicada un viernes.
- Numeración: radica entrada, salida e interna → prefijos E/S/I con secuencias independientes.
- Inmutabilidad: intenta (vía código/SQL de la app, no como superuser) actualizar `record_history` — la app no debe exponer ninguna vía; señala si existen endpoints o repos con UPDATE/DELETE sobre las tablas inmutables.
- Zona horaria: el número de radicado y `filed_at` usan America/Bogota (un radicado a las 23:30 Bogotá no debe tomar la fecha UTC del día siguiente).

## Parte 2 — Informe

Escribe `QA-INFORME.md` en la raíz: resumen ejecutivo (¿está listo para entregar? sí/no y por qué), tabla de resultados por sección 1.1–1.5, y hallazgos numerados `QA-001…` con severidad **Crítico** (pérdida/exposición de datos, bypass de auth/roles, caída, numeración duplicada) / **Alto** (regla de negocio incorrecta: días hábiles mal, workflow permisivo, historial incompleto) / **Medio** (UX rota, validación débil, test faltante) / **Bajo** (estilo, docs). Cada hallazgo: evidencia reproducible (comando + salida), archivo:línea, y corrección propuesta.

## Parte 3 — Corrección

Corrige **todos** los Críticos y Altos (los Medios si la corrección es local y de bajo riesgo; los demás quedan documentados). Cada corrección: su propio commit (Conventional Commits, referenciando `QA-001` etc.), con test de regresión cuando aplique. Al terminar: re-ejecuta 1.1, 1.2 y los pasos de 1.3/1.4 afectados, y actualiza `QA-INFORME.md` marcando cada hallazgo como Corregido (con hash del commit) o Pendiente (con justificación).

**Criterio de terminado:** `docker compose up` limpio desde cero + suites en verde + flujo E2E de 1.3 completo sin errores + cero hallazgos Críticos/Altos abiertos + `QA-INFORME.md` actualizado y commiteado.
