# Prompt de arranque — Folium v1 (Fase 1, single-tenant)

> Uso: copiar todo el contenido debajo de la línea y pegarlo como primer prompt en una sesión
> de Claude Code abierta en un **directorio vacío** (ej. `~/Documents/projects/folium`).

---

Construye desde cero **Folium v1**, un sistema de gestión documental (radicación de correspondencia) para una entidad pública colombiana. Implementa TODO lo descrito en este prompt de principio a fin: estructura del proyecto, migraciones, backend, frontend, seeds, tests y docker-compose funcionando. No me preguntes por decisiones que ya están tomadas aquí; si algo no está definido, elige la opción más simple y anótala en `DECISIONES.md`.

## 1. Contexto de negocio

Las entidades públicas colombianas están obligadas por la Ley 594 de 2000 (Ley General de Archivos) a radicar y dar trazabilidad a toda comunicación oficial. El software dominante (Orfeo GPL, PHP de los años 2000) está obsoleto e inseguro. Folium lo reemplaza con tecnología moderna.

Esta v1 **no es SaaS**: se despliega para UNA sola entidad (single-tenant). La versión SaaS multi-tenant vendrá después; por eso las convenciones de nombres y el modelo de datos deben quedar limpios, pero **sin** `tenant_id`, sin schemas por cliente y sin RLS.

Conceptos de dominio (usa estos términos en la UI en español; en código/BD usa los nombres en inglés indicados):

- **Radicado** (`record`): comunicación oficial registrada con número único. Tres tipos: **entrada** (`incoming`, llega a la entidad), **salida** (`outgoing`, la entidad responde o envía a un externo), **interna** (`internal`, entre dependencias de la misma entidad).
- **Dependencia** (`department`): unidad del organigrama (ej. "Despacho", "Secretaría de Hacienda"). Jerárquica (autorreferencia `parent_id`).
- **Número de radicado**: formato `YYYYMMDD-{E|S|I}-NNNNNN` (ej. `20260801-E-000042`). Secuencial por tipo, se reinicia cada año. Generación atómica (tabla de contadores con `SELECT ... FOR UPDATE` o `UPDATE ... RETURNING`).
- **Sticker**: etiqueta PDF imprimible del radicado con número, fecha, entidad y código de barras Code128 del número de radicado.
- **Vencimiento**: ciertos tipos de documento tienen plazo legal de respuesta en **días hábiles** (ej. derecho de petición: 15). El sistema calcula la fecha límite y alerta.

## 2. Stack (cerrado, no cambiar)

**Backend:** Go (última estable), router **Chi**, PostgreSQL 16+, `pgx/v5`, migraciones con **golang-migrate** (SQL puro en `migrations/`), PDF con **gofpdf**, código de barras con **boombuler/barcode**, Excel con **excelize**, contraseñas con **bcrypt**.

**Frontend:** Vite + React 19 + TypeScript + **TanStack Router v1** (file-based) + **Zustand 5** + SCSS. Sin librería de componentes pesada; CSS propio, limpio y sobrio (es software institucional: tablas densas, formularios claros, buen contraste).

**Arquitectura de despliegue:** un solo binario Go que expone la API REST bajo `/api/v1/*` y sirve la SPA compilada mediante `embed.FS` (fallback a `index.html` para rutas del router). Mismo origen: sin CORS. `docker-compose.yml` con dos servicios: `postgres` y `app` (build multi-stage: node para el front, go para el binario). Volúmenes para datos de Postgres y para archivos adjuntos.

**Sin**: BFF/Node en producción, Redis/Valkey, JWT, Kubernetes, S3. No los agregues.

## 3. Estructura del repositorio

Monorepo:

```
folium/
├── cmd/folium/main.go
├── internal/
│   ├── config/        # env vars, validación fail-fast al arrancar
│   ├── domain/        # structs y tipos de dominio, estados, errores
│   ├── repository/    # acceso a datos (pgx), una interfaz por agregado
│   ├── service/       # lógica de negocio (numeración, workflow, vencimientos, PII)
│   ├── handler/       # HTTP handlers + validación de entrada
│   ├── middleware/    # auth de sesión, require-role, audit, rate limit
│   ├── storage/       # interfaz Storage + implementación LocalDisk
│   ├── pdf/           # sticker y reportes PDF
│   ├── mailer/        # SMTP + plantillas de alerta
│   └── scheduler/     # goroutine de vencimientos
├── migrations/
├── web/               # frontend Vite (src/, package.json) — dist/ embebido en el binario
├── docker-compose.yml
├── Dockerfile
├── Makefile           # make dev, make test, make build, make migrate, make seed
├── .env.example
├── README.md
└── DECISIONES.md
```

## 4. Modelo de datos

Convenciones: tablas en plural snake_case; PK `id UUID DEFAULT gen_random_uuid()`; FKs `{tabla_singular}_id`; timestamps `TIMESTAMPTZ` (`created_at`, `updated_at`); enums como TEXT con CHECK; booleanos en positivo. PII buscable con sufijo `_hash`, PII cifrada con sufijo `_enc`.

Tablas (define columnas completas en las migraciones; aquí lo esencial):

- `entity_settings` — una sola fila: nombre de la entidad, NIT, dirección, logo (storage_key), zona horaria (`America/Bogota`).
- `users` — email único, `password_hash`, nombre, `department_id`, `role` TEXT CHECK (`admin | ventanilla | jefe_dependencia | funcionario`), `active`, `last_login_at`.
- `sessions` — `token_hash` (SHA-256 del token de sesión, nunca el valor), `user_id`, `user_agent`, `ip_address INET`, `expires_at`, `revoked`.
- `departments` — `name`, `code` único (ej. "DAF"), `parent_id` autorreferencia, `active`.
- `document_types` — `name`, `code` único, `response_days INT NULL` (días hábiles; NULL = sin plazo), `active`.
- `reception_channels` — Ventanilla, Email, Web, Mensajería… (`name`, `active`).
- `record_counters` — PK (`year`, `record_type`), `last_seq`. Numeración atómica.
- `records` — `record_number` único, `record_type` CHECK (`incoming|outgoing|internal`), `filed_at`, `document_date`, `subject`, `document_type_id`, `reception_channel_id`, remitente (ver PII abajo): `sender_type` (`individual|organization|employee`), `sender_name`, `sender_id_hash`, `sender_email_hash`, `sender_phone_enc`, `sender_address_enc`, `sender_city`, `sender_state`; `target_department_id`, `filed_by_id`, `assigned_user_id`, `internal_reference`, `parent_record_id` (respuesta/derivado), `attachment_type` (`none|physical|digital`), `physical_pages`, `due_date DATE`, `status` CHECK (`filed|assigned|in_progress|replied|archived|cancelled`), `anonymized BOOLEAN`. Índices en number, filed_at, department, status, due_date, sender_id_hash, parent.
- `outgoing_record_details` — 1:1 con records de salida: `origin_record_id`, `recipient_name`, `recipient_entity`, `recipient_address`, `send_channel` (`physical|email|courier`), `tracking_number`, `sent_at`.
- `record_attachments` — `record_id`, `original_name`, `mime_type`, `size_bytes`, `storage_key`, `sha256_hash`, `version`, `active`, `uploaded_by_id`.
- `record_history` — **inmutable, nunca se borra ni actualiza**: `record_id`, `from_status`, `to_status`, `from_department_id`, `to_department_id`, `from_user_id`, `to_user_id`, `action` (`file|assign|start|reply|return|redirect|archive|cancel|reopen`), `action_by_id`, `comment`, `attachment_id`, `created_at`.
- `document_access_logs` — inmutable: cada `view|download|print` de un adjunto con `user_id`, `ip_address`, `user_agent`.
- `audit_logs` — inmutable: `user_id`, `action` ("user.login", "record.create"…), `entity`, `entity_id`, `ip_address`, `metadata JSONB`.
- `data_consents` — Ley 1581: `id_number_hash`, `purpose`, `channel` (`in_person|web_portal|email`), `granted_at`, `revoked`.

## 5. Reglas de negocio clave

### Workflow (estados fijos en código, service dedicado)

`filed → assigned → in_progress → replied → archived`, más `cancelled` desde cualquier estado no final. Acciones y quién puede:

| Acción | Transición | Roles |
|---|---|---|
| Radicar | → `filed` | ventanilla, admin |
| Asignar a funcionario | `filed/assigned` → `assigned` | jefe_dependencia (de la dependencia destino), admin |
| Redirigir a otra dependencia | cualquier estado activo | jefe_dependencia, admin |
| Iniciar trámite | `assigned` → `in_progress` | funcionario asignado |
| Responder (crea radicado de salida vinculado) | `in_progress` → `replied` | funcionario asignado, jefe_dependencia |
| Devolver (a la dependencia anterior, comentario obligatorio) | estado activo | funcionario, jefe_dependencia |
| Archivar | `replied` → `archived` | jefe_dependencia, admin |

Toda transición escribe en `record_history` en la misma transacción. Transición inválida → 422 con mensaje claro.

### Numeración

Atómica y sin huecos dentro del año. Prefijo por tipo: E (incoming), S (outgoing), I (internal). Test obligatorio de concurrencia (N goroutines radicando a la vez → N números únicos consecutivos).

### Vencimientos y días hábiles

`due_date = filed_at + response_days` contando **solo días hábiles colombianos**: lunes-viernes menos festivos de Colombia (implementa la Ley Emiliani: festivos fijos + trasladables al lunes + los dependientes de Pascua; tabla o cálculo en `internal/service/workdays.go` con tests contra los festivos reales de 2026 y 2027). El scheduler (goroutine con ticker, corre a las 6:00 America/Bogota) envía por SMTP: alerta "por vencer" (3 días hábiles antes) al asignado y su jefe, y alerta "vencido" el día siguiente al vencimiento. Registra los envíos para no duplicar alertas.

### PII — Ley 1581 de 2012

- Cédula y email del remitente: **nunca en texto plano**. Guardar `HMAC-SHA256(valor_normalizado, PII_HMAC_KEY)`; la búsqueda por cédula/email hashea el término y compara.
- Teléfono y dirección: **AES-256-GCM** con `PII_AES_KEY` (nonce por valor, almacenado junto al ciphertext, base64). Se descifran solo al mostrar el detalle del radicado.
- Endpoint de supresión (solo admin): marca `anonymized = true`, borra los valores PII del radicado (nombre → "ANONIMIZADO", hashes y enc → NULL) pero **nunca borra el radicado ni su historial**.
- Al radicar presencialmente se registra el consentimiento en `data_consents`.

### Archivos adjuntos

Interfaz `Storage` (`Save`, `Open`, `Delete`, `Exists`) con implementación disco local bajo `STORAGE_PATH` (`records/{record_id}/{uuid}.{ext}`). Al subir: validar MIME real por magic bytes (pdf, jpg, png, tiff, docx, xlsx), límite `MAX_FILE_SIZE_MB`, calcular sha256. Descarga y visualización SIEMPRE vía endpoint autenticado que primero escribe en `document_access_logs` y luego hace stream del archivo; nunca rutas estáticas.

## 6. API y frontend

API REST bajo `/api/v1` con JSON; errores `{ "error": { "code", "message" } }`. Endpoints: auth (login/logout/me), CRUD de departments, users, document_types, reception_channels (solo admin), records (crear por tipo, detalle con historial y adjuntos, transiciones como sub-recursos `POST /records/{id}/assign|redirect|start|reply|return|archive`), attachments (upload/download/view), búsqueda (`GET /records?q=&type=&status=&department=&from=&to=&due=` con paginación), reportes, sticker (`GET /records/{id}/sticker.pdf`).

**Auth:** sesión server-side. Login valida bcrypt, crea fila en `sessions`, setea cookie `folium_sid` HttpOnly+Secure+SameSite=Lax. Middleware resuelve la sesión en cada request. Rate limit en login (5 intentos/minuto por IP). Logout revoca.

Pantallas del frontend (todas en español):

1. **Login**.
2. **Bandeja** (home): radicados de mi dependencia (jefe) o asignados a mí (funcionario); columnas número, fecha, asunto, remitente, estado, semáforo de vencimiento (verde >3 días hábiles, amarillo ≤3, rojo vencido); filtros por estado/tipo/fechas; paginación.
3. **Radicar entrada**: formulario completo (remitente, asunto, tipo de documento, medio de recepción, dependencia destino, anexos, consentimiento); al guardar muestra el número generado y botón "Imprimir sticker".
4. **Radicar salida** (desde un radicado de entrada o independiente) y **Radicar interna**.
5. **Detalle de radicado**: datos, adjuntos con visor de PDF/imagen inline, línea de tiempo del historial, botones de acción según rol/estado (asignar, redirigir, iniciar, responder, devolver, archivar) con modal de comentario.
6. **Búsqueda** global (número, remitente por cédula/email exacto, asunto, filtros combinados).
7. **Reportes**: radicados por período, por dependencia y tiempos de respuesta (promedio, a tiempo vs vencidos); vista en tabla + export **Excel** y **PDF**.
8. **Administración** (solo admin): entidad, dependencias (árbol), usuarios, tipos de documento (con plazo), medios de recepción.

Zustand solo para sesión/perfil y estado de UI; datos de servidor con fetch por ruta (loaders de TanStack Router). Manejo de 401 → redirigir a login.

## 7. Configuración y seeds

`.env.example` documentado: `DATABASE_URL`, `PORT`, `SESSION_TTL_HOURS`, `PII_HMAC_KEY`, `PII_AES_KEY` (32 bytes base64; el README explica cómo generarlas con openssl), `STORAGE_PATH`, `MAX_FILE_SIZE_MB`, `SMTP_HOST/PORT/USER/PASS/FROM`, `APP_BASE_URL`. Config falla al arrancar si falta algo crítico.

Seed (`make seed`): entidad "Alcaldía de Villanueva" (NIT ficticio), 4 dependencias (Despacho, Secretaría General, Hacienda, Planeación), 6 usuarios (1 admin, 1 ventanilla, 2 jefes, 2 funcionarios — password `Folium2026!` y anótalo en el README), tipos de documento colombianos reales (Derecho de petición 15 días, Tutela 3, Solicitud de información 10, Circular sin plazo, Factura sin plazo), medios de recepción, y ~20 radicados de ejemplo en distintos estados con historial coherente.

## 8. Calidad y entrega

- Tests de lo crítico: numeración concurrente, transiciones de workflow (válidas e inválidas por rol), días hábiles/festivos colombianos, HMAC/AES round-trip, endpoint de supresión. Handlers clave con httptest.
- `golangci-lint` limpio; frontend con `tsc --noEmit` y ESLint sin errores.
- Git desde el inicio con Conventional Commits, un commit por bloque funcional.
- README: qué es Folium, cómo levantar (`docker compose up` debe funcionar a la primera), usuarios de prueba, cómo correr tests, arquitectura en 10 líneas.
- `DECISIONES.md`: registra cada decisión que tomes que no esté explícita en este prompt.

**Orden de trabajo sugerido:** scaffolding + compose + migraciones → auth y sesiones → admin (dependencias, usuarios, catálogos) → radicación de entrada + numeración + adjuntos → sticker PDF → bandeja y detalle → workflow completo → salida e interna → vencimientos + mailer + scheduler → búsqueda → reportes y exportación → seeds → pulido de UI y README.

**Criterio de terminado:** con `docker compose up` y el seed cargado puedo — como ventanilla — radicar una entrada con PDF adjunto e imprimir el sticker; como jefe — verla en la bandeja, asignarla a un funcionario; como funcionario — iniciarla y responderla generando el radicado de salida vinculado; como admin — ver el reporte del período y exportarlo a Excel; y el historial del radicado muestra cada paso. Todo sin errores en consola ni en logs.
