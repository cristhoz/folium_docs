# Folium — Flujos de Trabajo y Privacidad de Datos
## Base de conocimiento técnica y normativa

**Fecha:** Abril 2026  
**Proyecto:** Folium  
**Confidencial — Uso interno**

---

## 1. Flujos de Trabajo — Máquina de Estados

### 1.1 Por qué no BPMN (todavía)

BPMN (Business Process Model and Notation) es el estándar internacional para modelar procesos. Sin embargo, para el contexto de Folium en su etapa actual **no es la opción correcta** por tres razones:

1. **Complejidad operacional** — implementar un motor BPMN propio en Go son meses de trabajo; los motores externos (Camunda, Zeebe) añaden dependencias críticas.
2. **Perfil del usuario** — los administradores de entidades públicas colombianas no están capacitados para modelar procesos con diagramas BPMN.
3. **Sobrediseño prematuro** — el 95% de los flujos documentales se resuelve con una máquina de estados configurable, que es órdenes de magnitud más simple.

**Decisión:** Máquina de estados configurable en MVP y Fase 2. BPMN/Temporal en Fase 3 si el mercado lo exige.

---

### 1.2 Concepto: Máquina de Estados

Una **máquina de estados** define:
- Un conjunto finito de **estados** en los que puede estar un radicado
- Las **transiciones** permitidas entre estados
- Las **condiciones** para que una transición sea válida (rol, plazo, documento adjunto)

El radicado siempre está en exactamente un estado. Para moverse al siguiente, debe ocurrir una acción válida ejecutada por un usuario con el rol correcto.

**Flujo típico de correspondencia de entrada:**

```
[radicado] ──asignar──► [asignado] ──iniciar──► [en_tramite]
                             │                        │
                          devolver                responder
                             │                        │
                         [radicado]            [respondido] ──archivar──► [archivado]
```

Cada entidad puede configurar su propio flujo. Una alcaldía pequeña puede tener 3 estados; una gobernación puede tener 7 con validaciones intermedias.

---

### 1.3 Modelo de Datos

```sql
-- Plantilla de flujo: define un tipo de proceso para un tipo de radicado
CREATE TABLE workflow_templates (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     UUID NOT NULL,
    nombre        TEXT NOT NULL,      -- "Correspondencia Entrada - Estándar"
    tipo_radicado TEXT NOT NULL,      -- entrada / salida / interna / pqrsd
    activo        BOOLEAN DEFAULT true,
    creado_en     TIMESTAMPTZ DEFAULT now()
);

-- Estados posibles dentro de un flujo
CREATE TABLE workflow_states (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id    UUID NOT NULL REFERENCES workflow_templates(id),
    nombre         TEXT NOT NULL,     -- radicado, asignado, en_tramite, respondido, archivado
    descripcion    TEXT,
    es_inicial     BOOLEAN DEFAULT false,
    es_final       BOOLEAN DEFAULT false,
    plazo_dias_hab INT               -- SLA: NULL = sin límite de tiempo
);

-- Transiciones permitidas entre estados
CREATE TABLE workflow_transitions (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id        UUID NOT NULL REFERENCES workflow_templates(id),
    estado_origen_id   UUID NOT NULL REFERENCES workflow_states(id),
    estado_destino_id  UUID NOT NULL REFERENCES workflow_states(id),
    nombre_accion      TEXT NOT NULL,   -- "asignar", "responder", "devolver", "archivar"
    roles_permitidos   TEXT[],          -- {"recepcionista", "jefe_dependencia"}
    requiere_comentario BOOLEAN DEFAULT false,
    requiere_documento  BOOLEAN DEFAULT false
);

-- Estado actual de cada radicado (una fila por radicado)
CREATE TABLE radicado_estado (
    radicado_id          UUID PRIMARY KEY,
    estado_id            UUID NOT NULL REFERENCES workflow_states(id),
    dependencia_id       UUID,
    usuario_asignado_id  UUID,
    fecha_asignacion     TIMESTAMPTZ,
    fecha_vencimiento    TIMESTAMPTZ     -- calculado: fecha_asignacion + plazo_dias_hab
);

-- Historial completo de movimientos (auditoría AGN)
CREATE TABLE radicado_historial (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    radicado_id        UUID NOT NULL,
    estado_origen_id   UUID,            -- NULL en la transición inicial
    estado_destino_id  UUID NOT NULL,
    usuario_id         UUID NOT NULL,
    fecha              TIMESTAMPTZ DEFAULT now(),
    comentario         TEXT,
    documento_id       UUID             -- documento adjunto en la transición
);
```

---

### 1.4 Lógica de Transición en Go

```go
// ErrTransicionNoPermitida se retorna cuando la acción no existe desde el estado actual
var (
    ErrTransicionNoPermitida = errors.New("transición no permitida desde el estado actual")
    ErrSinPermiso            = errors.New("el usuario no tiene el rol requerido para esta acción")
    ErrComentarioRequerido   = errors.New("esta transición requiere un comentario")
    ErrDocumentoRequerido    = errors.New("esta transición requiere un documento adjunto")
)

type TransicionRequest struct {
    RadicadoID  uuid.UUID
    Accion      string      // nombre de la acción: "asignar", "responder", etc.
    Usuario     Usuario
    Comentario  string
    DocumentoID *uuid.UUID
}

func (s *WorkflowService) Transicionar(ctx context.Context, req TransicionRequest) error {
    estadoActual, err := s.repo.GetEstadoActual(ctx, req.RadicadoID)
    if err != nil {
        return err
    }

    transicion, err := s.repo.GetTransicion(ctx, estadoActual.EstadoID, req.Accion)
    if err != nil || transicion == nil {
        return ErrTransicionNoPermitida
    }

    if !tieneRol(req.Usuario.Roles, transicion.RolesPermitidos) {
        return ErrSinPermiso
    }

    if transicion.RequiereComentario && req.Comentario == "" {
        return ErrComentarioRequerido
    }

    if transicion.RequiereDocumento && req.DocumentoID == nil {
        return ErrDocumentoRequerido
    }

    return s.repo.EjecutarTransicion(ctx, EjecutarTransicionParams{
        RadicadoID:      req.RadicadoID,
        EstadoDestinoID: transicion.EstadoDestinoID,
        UsuarioID:       req.Usuario.ID,
        Comentario:      req.Comentario,
        DocumentoID:     req.DocumentoID,
    })
    // EjecutarTransicion actualiza radicado_estado y escribe en radicado_historial
    // en una sola transacción de base de datos
}

func tieneRol(rolesUsuario, rolesRequeridos []string) bool {
    for _, requerido := range rolesRequeridos {
        for _, rol := range rolesUsuario {
            if rol == requerido {
                return true
            }
        }
    }
    return false
}
```

---

### 1.5 Beneficios de este enfoque

| Beneficio | Detalle |
|-----------|---------|
| Configurable sin código | Cada entidad define sus flujos desde el panel de administración |
| Auditoría automática | `radicado_historial` cumple con la trazabilidad exigida por el AGN |
| Extensible | El modelo de datos es compatible con una capa BPMN futura |
| SLA nativo | `plazo_dias_hab` + `fecha_vencimiento` permiten alertas y escalaciones |
| Flexible por tipo | Radicado de entrada, salida, interna y PQRSD pueden tener flujos distintos |

---

### 1.6 Hoja de ruta de workflows

| Fase | Funcionalidad |
|------|--------------|
| POC | Estados fijos en código (sin configuración por UI) |
| MVP | Editor de flujos en panel de administración, notificaciones por vencimiento |
| Fase 2 | Escalaciones automáticas, reportes de tiempos de respuesta por AGN |
| Fase 3 | Integración con Temporal.io para flujos complejos multi-entidad |

---

## 2. Privacidad de Datos — Ley 1581 de 2012

### 2.1 ¿Qué es la Ley 1581?

La **Ley Estatutaria 1581 de 2012** es la ley colombiana de protección de datos personales. Es el equivalente colombiano al GDPR europeo. Está reglamentada por el **Decreto 1377 de 2013** y su cumplimiento es supervisado por la **Superintendencia de Industria y Comercio (SIC)**.

Aplica a cualquier organización que recolecte, almacene, use o transfiera datos personales de ciudadanos colombianos — lo que incluye directamente a Folium, ya que cada radicado contiene datos personales del remitente.

---

### 2.2 Conceptos clave

| Término | Definición en contexto de Folium |
|---------|--------------------------------------|
| **Titular** | El ciudadano o funcionario cuyos datos se registran en el radicado |
| **Responsable** | La entidad pública cliente (la que contrata el sistema) |
| **Encargado** | Folium como plataforma (procesa datos en nombre del responsable) |
| **Dato personal** | Nombre, cédula, teléfono, dirección, email del remitente |
| **Dato sensible** | Datos de salud, origen étnico, orientación sexual (no aplica en el núcleo) |
| **Tratamiento** | Toda operación sobre el dato: recolectar, almacenar, usar, transferir, eliminar |
| **Finalidad** | El propósito declarado por el que se recolectan los datos |

---

### 2.3 Derechos del titular y su impacto en el sistema

| Derecho | Descripción | Implementación en el sistema |
|---------|-------------|------------------------------|
| **Conocer** | Saber qué datos se tienen sobre él | Consulta por número de identidad |
| **Actualizar** | Corregir datos inexactos | Endpoint de corrección con auditoría |
| **Suprimir** | Solicitar eliminación de sus datos | Anonimización (no borrado físico — ver sección 2.6) |
| **Revocar** | Retirar la autorización de tratamiento | Tabla de consentimientos con estado |
| **Acceder** | Obtener copia de sus datos | Exportación en formato estándar |
| **Presentar quejas** | Ante la SIC por violaciones | Proceso interno de gestión de reclamos |

---

### 2.4 Principios que debe cumplir el sistema

- **Legalidad** — Solo tratar datos con base en autorización o contrato
- **Finalidad** — Solo para el propósito declarado (gestión documental de la entidad)
- **Libertad** — No tratar datos sin autorización del titular
- **Veracidad** — Mantener los datos exactos y actualizados
- **Transparencia** — El titular debe saber quién trata sus datos y para qué
- **Acceso restringido** — Solo personal autorizado accede a los datos PII
- **Seguridad** — Medidas técnicas y administrativas para proteger los datos
- **Confidencialidad** — Obligación de reserva sobre los datos tratados

---

### 2.5 Modelo de consentimiento

```go
// ConsentimientoTitular registra la autorización del ciudadano
// para que sus datos sean tratados por la entidad
type ConsentimientoTitular struct {
    ID                  uuid.UUID  `db:"id"`
    TenantID            uuid.UUID  `db:"tenant_id"`
    NumeroIdentidadHash string     `db:"numero_identidad_hash"` // HMAC-SHA256, nunca en texto plano
    Finalidad           string     `db:"finalidad"`             // "gestión_documental_entidad_X"
    FechaOtorgado       time.Time  `db:"fecha_otorgado"`
    Canal               string     `db:"canal"`                 // "radicacion_presencial" / "portal_web"
    Revocado            bool       `db:"revocado"`
    FechaRevocado       *time.Time `db:"fecha_revocado"`
}
```

```sql
CREATE TABLE consentimientos_titulares (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id             UUID NOT NULL,
    numero_identidad_hash TEXT NOT NULL,    -- HMAC del número real con clave de tenant
    finalidad             TEXT NOT NULL,
    fecha_otorgado        TIMESTAMPTZ NOT NULL DEFAULT now(),
    canal                 TEXT NOT NULL,    -- radicacion_presencial / portal_web / email
    revocado              BOOLEAN DEFAULT false,
    fecha_revocado        TIMESTAMPTZ
);
```

---

### 2.6 El conflicto entre Ley 1581 y normativa archivística AGN

Este es el punto más importante de entender:

- La **Ley 1581** dice: *"El titular tiene derecho a que sus datos sean eliminados cuando los solicite."*
- La **normativa AGN** (Ley 594/2000 + TRD) dice: *"Los documentos deben conservarse según los plazos de retención definidos. No se pueden destruir sin cumplir el proceso."*

**Ambas leyes son de cumplimiento obligatorio.** La solución es la **anonimización**:

```
Solicitud de supresión recibida
            │
            ▼
¿El radicado está dentro del plazo de retención TRD?
            │
     SÍ ───┤─── NO
            │         └──► Eliminar físicamente el documento
            ▼               y los datos PII del registro
  Anonimizar los campos PII en el registro del radicado:
  - nombre → "[DATO SUPRIMIDO]"
  - numero_identidad → "[DATO SUPRIMIDO]"
  - email, telefono, direccion → "[DATO SUPRIMIDO]"
  
  El radicado, su número, fechas e historial de gestión
  se conservan (son parte del archivo institucional)
```

Esto cumple ambas leyes: el documento institucional se preserva, los datos personales del titular son eliminados.

---

### 2.7 Estrategia de seguridad de datos PII — por capas

#### Capa 1 — Aislamiento (desde el POC)
- Un schema de PostgreSQL por tenant
- Un bucket de Garage por tenant
- Un bug de código no puede filtrar datos entre clientes

#### Capa 2 — Cifrado selectivo de campos

| Campo | Estrategia | Razón |
|-------|-----------|-------|
| `numero_identidad` | HMAC-SHA256 determinístico | Permite búsqueda exacta sin exponer el valor |
| `email` | HMAC-SHA256 determinístico | Permite búsqueda exacta |
| `telefono`, `direccion` | AES-256-GCM | Raramente se busca por estos campos |
| `nombre`, `apellidos` | Sin cifrar en MVP | La búsqueda por nombre es funcionalidad core; se añade tokenización en Fase 2 |
| Archivos (Garage) | SSE-S3 con clave por tenant | Cifrado nativo de Garage |

**Por qué HMAC determinístico para campos buscables:**

```go
func HashIdentidad(numeroIdentidad, clavetenant string) string {
    mac := hmac.New(sha256.New, []byte(claveTenant))
    mac.Write([]byte(numeroIdentidad))
    return hex.EncodeToString(mac.Sum(nil))
}

// Al buscar: hasheas el término de búsqueda con la misma clave
// y buscas el hash en la DB — sin exponer el valor real nunca
```

#### Capa 3 — Gestión de claves por fase

| Fase | Mecanismo | Complejidad operacional |
|------|-----------|------------------------|
| POC | Variables de entorno / Docker secrets | Baja |
| MVP | Doppler o AWS Secrets Manager | Media |
| Fase 2+ | HashiCorp Vault Transit Engine | Alta — pero clave rotation automática y audit log completo |

**Vault Transit en producción:** tu aplicación nunca tiene la clave de cifrado en memoria. Envía el texto plano a Vault, recibe el ciphertext. En lectura: envía el ciphertext, recibe el texto plano. La clave nunca sale de Vault.

#### Capa 4 — Row-Level Security en PostgreSQL

```sql
-- Defensa en profundidad: incluso si hay un bug de cross-tenant,
-- la base de datos misma rechaza el acceso
ALTER TABLE radicados ENABLE ROW LEVEL SECURITY;

CREATE POLICY radicados_tenant_isolation ON radicados
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

#### Capa 5 — Tránsito cifrado

- TLS terminado en el reverse proxy (Caddy o Traefik)
- Caddy emite y renueva certificados SSL automáticamente
- HSTS habilitado
- Comunicación interna entre servicios también cifrada (mTLS en Fase 3)

---

### 2.8 Hoja de ruta de cumplimiento Ley 1581

| Fase | Qué implementar |
|------|----------------|
| **POC** | Tabla de consentimientos en el modelo de datos. Aviso de privacidad en formulario de radicación. TLS en tránsito. Schema isolation. |
| **MVP** | HMAC para campos buscables PII. AES-256-GCM para teléfono/dirección. Row-Level Security en PostgreSQL. Proceso básico de solicitud de supresión (manual). |
| **Fase 2** | HashiCorp Vault o KMS en la nube. Portal de derechos del titular. Flujo automatizado de anonimización. Tokenización de nombre/apellidos con índice de búsqueda separado. Registro de actividades de tratamiento. |
| **Fase 3** | Reportes ante la SIC. Auditoría externa de cumplimiento. Data Processing Agreements (DPA) con cada cliente. |

---

## 3. Resumen de Decisiones Arquitectónicas

| Tema | Decisión | Revisión |
|------|---------|----------|
| Workflows | Máquina de estados configurable | POC → MVP |
| BPMN | Diferido a Fase 3 vía Temporal.io | Fase 3 |
| Cifrado PII buscable | HMAC-SHA256 determinístico por tenant | Desde MVP |
| Cifrado PII no buscable | AES-256-GCM | Desde MVP |
| Gestión de claves | Env vars → Secrets Manager → Vault | Progresivo |
| Aislamiento base | PostgreSQL schema por tenant + RLS | Desde POC |
| Supresión de datos | Anonimización (no borrado) por conflicto AGN/1581 | Desde MVP |
| Consentimiento | Tabla de consentimientos desde el modelo inicial | Desde POC |

---

## 4. Referencias Normativas

| Norma | Descripción | Relevancia |
|-------|-------------|-----------|
| Ley 1581 de 2012 | Protección de datos personales | PII de ciudadanos en radicados |
| Decreto 1377 de 2013 | Reglamentación de Ley 1581 | Aviso de privacidad, autorización |
| Ley 594 de 2000 | Ley General de Archivos | Retención documental, TRD |
| Decreto 1080 de 2015 | Decreto Único Reglamentario del sector cultura | Marco archivístico |
| Acuerdo 003 AGN 2015 | Sistemas de gestión de documentos | Funcionalidades requeridas |
| Acuerdo 006 AGN 2014 | Digitalización y comunicaciones oficiales | Radicación electrónica |

---

*Documento de base de conocimiento — Proyecto Folium. Versión 1.0 — Abril 2026.*
