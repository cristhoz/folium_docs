# Folium — Schema de Base de Datos
## Documentación técnica del modelo de datos

**Versión:** 1.1
**Fecha:** Mayo 2026
**Proyecto:** Folium — foliumhq.co
**Confidencial — Uso interno**

---

## 1. Estrategia multi-tenant

Cada entidad cliente tiene su propio **schema de PostgreSQL** (`tenant_{slug}`). El schema `public` contiene únicamente las tablas de la plataforma SaaS global.

Esta estrategia garantiza:
- Aislamiento total de datos entre clientes a nivel de base de datos
- Posibilidad de backup/restore por tenant de forma independiente
- Row-Level Security (RLS) como capa de defensa adicional
- Escalabilidad horizontal moviendo schemas a instancias separadas si se requiere

El almacenamiento de archivos usa **Garage** (S3-compatible). Cada tenant tiene su propio bucket con la convención `folium-{tenant_slug}`.

---

## 2. Schema `public` — Plataforma SaaS

Contiene las tablas que pertenecen a la operación global de Folium, no a ningún tenant específico.

### 2.1 `tenants`

Representa a cada entidad cliente contratante del servicio.

```sql
CREATE TABLE public.tenants (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    slug          TEXT        NOT NULL UNIQUE,   -- "gobernacion-bolivar" → subdominio en app.foliumhq.co
    name          TEXT        NOT NULL,           -- "Gobernación de Bolívar"
    nit           TEXT        NOT NULL UNIQUE,
    logo_url      TEXT,
    plan          TEXT        NOT NULL DEFAULT 'basic',
                              -- basic | professional | institutional | on_premise
    status        TEXT        NOT NULL DEFAULT 'trial',
                              -- trial | active | suspended | cancelled
    max_users     INT         NOT NULL DEFAULT 20,
    max_storage_gb INT        NOT NULL DEFAULT 10,
    schema_name   TEXT        NOT NULL UNIQUE,   -- "tenant_gobernacion_bolivar"
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

> El campo `slug` define el subdominio del tenant: `{slug}.app.foliumhq.co`

---

### 2.2 `subscriptions`

Historial de suscripciones y facturación por tenant.

```sql
CREATE TABLE public.subscriptions (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL REFERENCES public.tenants(id),
    plan            TEXT        NOT NULL,
    price_cop       NUMERIC(12,2) NOT NULL,
    started_at      DATE        NOT NULL,
    ends_at         DATE,
    status          TEXT        NOT NULL DEFAULT 'active',
                                -- active | expired | cancelled
    payment_ref     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### 2.3 `platform_admins`

Administradores de la plataforma Folium (superadmins internos).

```sql
CREATE TABLE public.platform_admins (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    email         TEXT        NOT NULL UNIQUE,
    password_hash TEXT        NOT NULL,
    name          TEXT        NOT NULL,
    active        BOOLEAN     NOT NULL DEFAULT true,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 3. Schema por tenant

Cada tenant tiene su propio schema. El nombre sigue la convención `tenant_{slug_normalizado}` (guiones reemplazados por guiones bajos). Todas las tablas de este schema incluyen `tenant_id` para RLS como defensa en profundidad.

---

### 3.1 Seguridad y acceso

#### `users`

```sql
CREATE TABLE users (
    id             UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id      UUID        NOT NULL,   -- RLS
    email          TEXT        NOT NULL UNIQUE,
    password_hash  TEXT        NOT NULL,
    first_name     TEXT        NOT NULL,
    last_name      TEXT        NOT NULL,
    department_id  UUID        REFERENCES departments(id),
    active         BOOLEAN     NOT NULL DEFAULT true,
    last_login_at  TIMESTAMPTZ,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### `roles`

Roles configurables por tenant. Los permisos se almacenan como mapa JSON para flexibilidad sin migraciones.

```sql
CREATE TABLE roles (
    id          UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID    NOT NULL,
    name        TEXT    NOT NULL,           -- "receptionist", "department_head", "administrator"
    description TEXT,
    permissions JSONB   NOT NULL DEFAULT '{}',
                        -- { "records.create": true, "records.redirect": true, "trd.manage": false }
    UNIQUE (tenant_id, name)
);
```

#### `user_roles`

```sql
CREATE TABLE user_roles (
    user_id   UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id   UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
```

#### `sessions`

Representa una sesión activa de usuario. El JWT de acceso es stateless, pero el `jti` almacenado aquí permite su revocación inmediata (logout, cambio de contraseña, bloqueo de cuenta). El refresh token nunca se almacena en texto plano.

```sql
CREATE TABLE sessions (
    id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID        NOT NULL,
    user_id      UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash   TEXT        NOT NULL UNIQUE,
                 -- SHA-256 del refresh token — nunca en texto plano
    jti          UUID        NOT NULL UNIQUE,
                 -- JWT ID del access token asociado; permite revocación inmediata antes de su expiración
    user_agent   TEXT,
    ip_address   INET,
    expires_at   TIMESTAMPTZ NOT NULL,
    revoked      BOOLEAN     NOT NULL DEFAULT false,
    revoked_at   TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_user_id    ON sessions (user_id);
CREATE INDEX idx_sessions_token_hash ON sessions (token_hash);
CREATE INDEX idx_sessions_jti        ON sessions (jti);
```

---

#### `audit_logs`

Registro inmutable de toda acción relevante en el sistema. No se borran registros de esta tabla.

```sql
CREATE TABLE audit_logs (
    id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id  UUID        NOT NULL,
    user_id    UUID        REFERENCES users(id),
    action     TEXT        NOT NULL,   -- "user.login", "record.create", "record.redirect"
    entity     TEXT,                   -- "record", "user", "department"
    entity_id  UUID,
    ip_address INET,
    user_agent TEXT,
    metadata   JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### 3.2 Estructura organizacional

#### `departments`

Representa las dependencias del organigrama de la entidad. Soporta jerarquía mediante autorreferencia.

```sql
CREATE TABLE departments (
    id         UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id  UUID    NOT NULL,
    name       TEXT    NOT NULL,
    code       TEXT    NOT NULL,   -- "DAF", "DESPACHO", "RRHH"
    parent_id  UUID    REFERENCES departments(id),   -- NULL = raíz del organigrama
    active     BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (tenant_id, code)
);
```

---

### 3.3 Configuración documental (TRD)

Implementa las Tablas de Retención Documental exigidas por el AGN (Acuerdo 004 de 2019).

#### `document_series`

```sql
CREATE TABLE document_series (
    id          UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID    NOT NULL,
    code        TEXT    NOT NULL,   -- "1", "2", "3"
    name        TEXT    NOT NULL,   -- "Actas", "Contratos"
    description TEXT,
    active      BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (tenant_id, code)
);
```

#### `document_subseries`

Cada subserie define los tiempos de retención y la disposición final, conforme a la TRD aprobada por la entidad.

```sql
CREATE TABLE document_subseries (
    id                      UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID    NOT NULL,
    series_id               UUID    NOT NULL REFERENCES document_series(id),
    code                    TEXT    NOT NULL,   -- "1.1", "1.2"
    name                    TEXT    NOT NULL,   -- "Actas de Comité"
    description             TEXT,
    retention_management_years  INT,            -- años en archivo de gestión
    retention_central_years     INT,            -- años en archivo central
    final_disposition       TEXT    NOT NULL DEFAULT 'preserve',
                            -- preserve | destroy | microfilm | select
    active                  BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (tenant_id, code)
);
```

#### `document_types`

```sql
CREATE TABLE document_types (
    id              UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID    NOT NULL,
    name            TEXT    NOT NULL,   -- "Derecho de Petición", "Circular", "Contrato"
    code            TEXT    NOT NULL,
    response_days   INT,                -- plazo legal de respuesta en días hábiles; NULL = sin plazo
    active          BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (tenant_id, code)
);
```

#### `reception_channels`

```sql
CREATE TABLE reception_channels (
    id         UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id  UUID    NOT NULL,
    name       TEXT    NOT NULL,   -- "Ventanilla", "Email", "Fax", "Web", "Mensajería"
    active     BOOLEAN NOT NULL DEFAULT true
);
```

---

### 3.4 Radicados

Entidad central del sistema. Representa cualquier comunicación oficial recibida, generada o tramitada internamente por la entidad.

#### `record_counters`

Maneja la numeración anual por tipo. Se usa para generar el número de radicado con formato `YYYYMMDD-[E/S/I]-NNNNNN` y se reinicia cada año.

```sql
CREATE TABLE record_counters (
    tenant_id   UUID NOT NULL,
    year        INT  NOT NULL,
    record_type TEXT NOT NULL,   -- incoming | outgoing | internal
    last_seq    INT  NOT NULL DEFAULT 0,
    PRIMARY KEY (tenant_id, year, record_type)
);
```

#### `records`

```sql
CREATE TABLE records (
    id                    UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id             UUID        NOT NULL,

    -- Identificación
    record_number         TEXT        NOT NULL UNIQUE,
                          -- formato: "20260502-E-001234"
    record_type           TEXT        NOT NULL,
                          -- incoming | outgoing | internal
    filed_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    document_date         DATE,                   -- fecha del documento original
    subject               TEXT        NOT NULL,

    -- Clasificación documental
    document_type_id      UUID        REFERENCES document_types(id),
    subseries_id          UUID        REFERENCES document_subseries(id),
    reception_channel_id  UUID        REFERENCES reception_channels(id),

    -- Remitente (campos PII — ver sección 3.7)
    sender_type           TEXT        NOT NULL,
                          -- individual | organization | employee
    sender_name           TEXT        NOT NULL,
    sender_id_hash        TEXT,       -- HMAC-SHA256(cedula, tenant_key) — permite búsqueda sin exponer el valor
    sender_email_hash     TEXT,       -- HMAC-SHA256(email, tenant_key)
    sender_phone_enc      TEXT,       -- AES-256-GCM
    sender_address_enc    TEXT,       -- AES-256-GCM
    sender_city           TEXT,
    sender_state          TEXT,       -- departamento
    sender_country        TEXT        NOT NULL DEFAULT 'Colombia',

    -- Enrutamiento
    target_department_id  UUID        REFERENCES departments(id),
    filed_by_id           UUID        NOT NULL REFERENCES users(id),
    current_user_id       UUID        REFERENCES users(id),
    internal_reference    TEXT,       -- referencia interna de la entidad

    -- Vínculos
    parent_record_id      UUID        REFERENCES records(id),
                          -- si es respuesta a otro radicado o radicado derivado

    -- Físicos
    attachment_type       TEXT        NOT NULL DEFAULT 'none',
                          -- none | physical | digital
    physical_pages        INT,        -- folios físicos

    -- Control
    due_date              DATE,       -- calculado: filed_at + document_type.response_days
    status                TEXT        NOT NULL DEFAULT 'active',
                          -- active | assigned | in_progress | replied | archived | overdue | cancelled
    anonymized            BOOLEAN     NOT NULL DEFAULT false,
                          -- true cuando se aplicó supresión por Ley 1581 (no borrado físico)

    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Índices de búsqueda frecuente
CREATE INDEX idx_records_number        ON records (record_number);
CREATE INDEX idx_records_filed_at      ON records (filed_at);
CREATE INDEX idx_records_department    ON records (target_department_id);
CREATE INDEX idx_records_status        ON records (status);
CREATE INDEX idx_records_due_date      ON records (due_date);
CREATE INDEX idx_records_sender_id     ON records (sender_id_hash);    -- búsqueda PII segura
CREATE INDEX idx_records_sender_email  ON records (sender_email_hash);
CREATE INDEX idx_records_parent        ON records (parent_record_id);
```

---

### 3.5 Documentos adjuntos

Archivos almacenados en Garage. La `storage_key` es la ruta dentro del bucket `folium-{tenant_slug}`.

```sql
CREATE TABLE record_attachments (
    id             UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id      UUID        NOT NULL,
    record_id      UUID        NOT NULL REFERENCES records(id) ON DELETE CASCADE,
    original_name  TEXT        NOT NULL,
    mime_type      TEXT        NOT NULL,
    size_bytes     BIGINT      NOT NULL,
    storage_key    TEXT        NOT NULL,
                   -- ruta en Garage: "records/{record_id}/{uuid}.pdf"
    sha256_hash    TEXT        NOT NULL,   -- integridad del archivo
    version        INT         NOT NULL DEFAULT 1,
    active         BOOLEAN     NOT NULL DEFAULT true,
    uploaded_by_id UUID        NOT NULL REFERENCES users(id),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### 3.6 Workflow — Máquina de estados

Permite a cada entidad configurar sus propios flujos de tramitación sin escribir código. Ver `folium-workflows-y-privacidad.md` para la lógica de transición en Go.

#### `workflow_templates`

Define el flujo para un tipo de radicado.

```sql
CREATE TABLE workflow_templates (
    id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID        NOT NULL,
    name         TEXT        NOT NULL,      -- "Correspondencia Entrada - Estándar"
    record_type  TEXT        NOT NULL,      -- incoming | outgoing | internal | citizen_request
    active       BOOLEAN     NOT NULL DEFAULT true,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);
```

#### `workflow_states`

```sql
CREATE TABLE workflow_states (
    id              UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID    NOT NULL,
    template_id     UUID    NOT NULL REFERENCES workflow_templates(id) ON DELETE CASCADE,
    name            TEXT    NOT NULL,
                    -- "filed", "assigned", "in_progress", "replied", "archived"
    description     TEXT,
    is_initial      BOOLEAN NOT NULL DEFAULT false,
    is_final        BOOLEAN NOT NULL DEFAULT false,
    sla_working_days INT             -- NULL = sin límite de tiempo
);
```

#### `workflow_transitions`

```sql
CREATE TABLE workflow_transitions (
    id                  UUID      PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID      NOT NULL,
    template_id         UUID      NOT NULL REFERENCES workflow_templates(id) ON DELETE CASCADE,
    from_state_id       UUID      NOT NULL REFERENCES workflow_states(id),
    to_state_id         UUID      NOT NULL REFERENCES workflow_states(id),
    action_name         TEXT      NOT NULL,
                        -- "assign", "start", "reply", "return", "archive"
    allowed_roles       TEXT[]    NOT NULL DEFAULT '{}',
                        -- ["receptionist", "department_head"]
    requires_comment    BOOLEAN   NOT NULL DEFAULT false,
    requires_attachment BOOLEAN   NOT NULL DEFAULT false
);
```

#### `record_states`

Estado actual de cada radicado. Una sola fila por radicado, se actualiza en cada transición.

```sql
CREATE TABLE record_states (
    record_id           UUID        PRIMARY KEY REFERENCES records(id),
    tenant_id           UUID        NOT NULL,
    state_id            UUID        NOT NULL REFERENCES workflow_states(id),
    department_id       UUID        REFERENCES departments(id),
    assigned_user_id    UUID        REFERENCES users(id),
    assigned_at         TIMESTAMPTZ,
    due_at              TIMESTAMPTZ -- calculado: assigned_at + state.sla_working_days
);
```

#### `record_history`

Historial inmutable de movimientos. Cumple el requisito de trazabilidad completa del AGN. **No se borran registros de esta tabla.**

```sql
CREATE TABLE record_history (
    id                   UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID        NOT NULL,
    record_id            UUID        NOT NULL REFERENCES records(id),
    from_state_id        UUID        REFERENCES workflow_states(id),   -- NULL en la transición inicial
    to_state_id          UUID        NOT NULL REFERENCES workflow_states(id),
    from_department_id   UUID        REFERENCES departments(id),
    to_department_id     UUID        REFERENCES departments(id),
    action_by_id         UUID        NOT NULL REFERENCES users(id),
    comment              TEXT,
    attachment_id        UUID        REFERENCES record_attachments(id),
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_record_history_record  ON record_history (record_id);
CREATE INDEX idx_record_history_date    ON record_history (created_at);
```

---

### 3.7 Privacidad — Ley 1581 de 2012

#### `data_consents`

Registra el consentimiento del ciudadano para el tratamiento de sus datos. El número de identidad nunca se almacena en texto plano.

```sql
CREATE TABLE data_consents (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID        NOT NULL,
    id_number_hash  TEXT        NOT NULL,
                    -- HMAC-SHA256(cedula, tenant_key) — nunca el valor real
    purpose         TEXT        NOT NULL,
                    -- "document_management_{tenant_slug}"
    channel         TEXT        NOT NULL,
                    -- in_person | web_portal | email
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked         BOOLEAN     NOT NULL DEFAULT false,
    revoked_at      TIMESTAMPTZ
);
```

#### `data_rights_requests`

Gestiona las solicitudes de ejercicio de derechos del titular (conocer, actualizar, suprimir, revocar, acceder).

```sql
CREATE TABLE data_rights_requests (
    id               UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID        NOT NULL,
    request_type     TEXT        NOT NULL,
                     -- access | update | delete | revoke | export
    id_number_hash   TEXT        NOT NULL,
    status           TEXT        NOT NULL DEFAULT 'pending',
                     -- pending | in_progress | completed | rejected
    resolved_by_id   UUID        REFERENCES users(id),
    resolved_at      TIMESTAMPTZ,
    notes            TEXT,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

### 3.8 Radicados de salida (extensión)

Campos adicionales específicos de los radicados de tipo `outgoing`. Se relaciona 1:1 con `records`.

```sql
CREATE TABLE outgoing_record_details (
    record_id            UUID        PRIMARY KEY REFERENCES records(id) ON DELETE CASCADE,
    tenant_id            UUID        NOT NULL,
    origin_record_id     UUID        REFERENCES records(id),   -- radicado de entrada que origina la respuesta
    recipient_name       TEXT        NOT NULL,
    recipient_entity     TEXT,
    recipient_address    TEXT,
    send_channel         TEXT        NOT NULL,
                         -- physical | email | fax | courier
    tracking_number      TEXT,       -- número de guía o consecutivo de envío
    sent_at              TIMESTAMPTZ
);
```

---

### 3.9 PQRSD — Portal ciudadano (Fase 2)

Extiende los radicados de tipo `incoming` con atributos propios de solicitudes ciudadanas.

```sql
CREATE TABLE citizen_requests (
    id                       UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id                UUID    NOT NULL,
    record_id                UUID    NOT NULL UNIQUE REFERENCES records(id),
    request_type             TEXT    NOT NULL,
                             -- petition | complaint | claim | suggestion | report
    intake_channel           TEXT    NOT NULL,
                             -- in_person | web | email
    lookup_token             TEXT    NOT NULL UNIQUE,
                             -- token público para que el ciudadano consulte el estado sin login
    notify_email             TEXT,   -- email del ciudadano para notificaciones
    notifications_log        JSONB   NOT NULL DEFAULT '[]'
);
```

---

## 4. Row-Level Security

Defensa adicional ante bugs de cross-tenant. La aplicación establece `app.tenant_id` en cada conexión antes de ejecutar queries.

```sql
-- Habilitar RLS en tablas con datos sensibles
ALTER TABLE records          ENABLE ROW LEVEL SECURITY;
ALTER TABLE record_history   ENABLE ROW LEVEL SECURITY;
ALTER TABLE record_attachments ENABLE ROW LEVEL SECURITY;
ALTER TABLE users            ENABLE ROW LEVEL SECURITY;

-- Política: un tenant solo ve sus propios datos
CREATE POLICY tenant_isolation ON records
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation ON record_history
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

> La aplicación en Go establece `SET app.tenant_id = '{uuid}'` al inicio de cada request tras validar el JWT del tenant.

---

## 5. Convenciones de nombres

| Regla | Ejemplo |
|---|---|
| Tablas en plural, snake_case | `records`, `document_types`, `record_attachments` |
| Claves primarias siempre `id UUID` | `id UUID PRIMARY KEY DEFAULT gen_random_uuid()` |
| Claves foráneas: `{tabla_singular}_id` | `department_id`, `record_id`, `user_id` |
| Timestamps: `created_at`, `updated_at` | siempre `TIMESTAMPTZ`, nunca `TIMESTAMP` |
| Booleanos en positivo | `active`, `anonymized`, `is_initial` (no `is_not_deleted`) |
| Campos PII buscables: sufijo `_hash` | `sender_id_hash`, `sender_email_hash` |
| Campos PII cifrados: sufijo `_enc` | `sender_phone_enc`, `sender_address_enc` |
| Campos de almacenamiento (Garage): `storage_key` | ruta relativa dentro del bucket del tenant |
| Enums como `TEXT` con constraint o check | mantiene flexibilidad sin migraciones de tipo |

---

## 6. Resumen de decisiones de diseño

| Decisión | Detalle |
|---|---|
| Multi-tenant por schema | Schema `tenant_{slug}` por cliente + RLS como defensa en profundidad |
| Almacenamiento de archivos | Garage S3-compatible — bucket `folium-{tenant_slug}` por cliente |
| Numeración de radicados | `record_counters` con reinicio anual — formato `YYYYMMDD-[E/S/I]-NNNNNN` |
| PII buscable | HMAC-SHA256 determinístico por tenant (`sender_id_hash`, `sender_email_hash`) |
| PII no buscable | AES-256-GCM (`sender_phone_enc`, `sender_address_enc`) |
| Supresión Ley 1581 | Campo `anonymized = true` + borrado de valores PII — **no borrado físico** del radicado |
| Historial | `record_history` inmutable — nunca se borran filas, cumple trazabilidad AGN |
| Workflows | Máquina de estados configurable por tenant (`workflow_templates` → `workflow_states` → `workflow_transitions`) |
| BPMN | Diferido a Fase 3 vía Temporal.io |
| Consentimiento | Tabla `data_consents` desde la POC — `id_number_hash` nunca en texto plano |
| TRD | `document_series` → `document_subseries` con tiempos de retención y disposición final |
| PQRSD | Tabla `citizen_requests` como extensión 1:1 de `records` — Fase 2 |

---

## 7. Diagrama de entidades principales

```
tenants (public)
    │
    └── schema: tenant_{slug}
            │
            ├── departments ◄──────────────────────────┐
            │       └── users ──── user_roles ── roles  │
            │               └── sessions                │
            │                                           │
            ├── document_types                          │
            ├── reception_channels                      │
            ├── document_series                         │
            │       └── document_subseries              │
            │                                           │
            ├── record_counters                         │
            │                                           │
            └── records ◄── parent_record_id (self)     │
                    │           │                       │
                    │           └── outgoing_record_details
                    │           └── citizen_requests (Fase 2)
                    │
                    ├── record_attachments (→ Garage)
                    │
                    ├── record_states ──── workflow_states
                    │                           │
                    └── record_history    workflow_templates
                                               │
                    data_consents         workflow_transitions
                    data_rights_requests
                    audit_logs
```

---

*Documento técnico — Proyecto Folium. Versión 1.0 — Mayo 2026.*
