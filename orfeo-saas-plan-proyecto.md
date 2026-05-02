# Plan de Proyecto — Orfeo SaaS
## Sistema de Gestión Documental para Entidades Públicas Colombianas

**Versión:** 1.0  
**Fecha:** Abril 2026  
**Estado:** En planificación  
**Confidencial**

---

## 1. Visión del Producto

Plataforma SaaS de gestión documental construida desde cero en Go, diseñada nativamente para cumplir con la normativa colombiana del AGN (Archivo General de la Nación). Reemplaza a Orfeo GPL (obsoleto e inseguro) con una solución moderna, segura, escalable y comercialmente sostenible.

### Principios de diseño
- **Normativa primero** — cada módulo se diseña con la regulación colombiana como requisito base
- **SaaS multi-tenant** — un solo despliegue sirve a múltiples entidades con datos completamente aislados
- **API-first** — todo el backend expone API REST; el frontend es un cliente más
- **Código propietario** — reescritura limpia (clean room), sin copiar código GPL
- **Simplicidad operacional** — binario único en Go, fácil de desplegar y mantener

---

## 2. Stack Tecnológico

### Backend
| Componente | Tecnología | Justificación |
|---|---|---|
| Lenguaje | Go | Rendimiento, seguridad, binario único, ideal para APIs |
| Framework web | Gin o Echo | Maduros, rápidos, buen ecosistema |
| Base de datos | PostgreSQL | Multi-tenant por schemas, robusto, open source |
| Almacenamiento | MinIO | S3-compatible, auto-hospedable, escalable |
| Cola de trabajos | Asynq (Redis) | Emails, notificaciones, procesamiento async |
| Autenticación | JWT + Refresh tokens | Stateless, compatible con SaaS multi-tenant |
| PDF | gofpdf / go-wkhtmltopdf | Generación de stickers y reportes |

### Frontend
| Componente | Tecnología |
|---|---|
| Framework | Next.js / SvelteKit / Vue (a definir) |
| Estilos | Tailwind CSS |
| Componentes | Librería UI a definir (shadcn, PrimeVue, etc.) |
| Estado | React Query / Pinia / TanStack |

### Infraestructura
| Componente | Tecnología |
|---|---|
| Contenedores | Docker + Docker Compose |
| Orquestación | Kubernetes (fases avanzadas) |
| CI/CD | GitHub Actions |
| Monitoreo | Prometheus + Grafana |
| Logs | Loki o similar |
| Nube | AWS / DigitalOcean / Hetzner |

---

## 3. Arquitectura General

```
┌──────────────────────────────────────────────────────┐
│                    CLIENTES                          │
│         Navegador web / App móvil (futuro)           │
└─────────────────────┬────────────────────────────────┘
                      │ HTTPS
┌─────────────────────▼────────────────────────────────┐
│                  API GATEWAY                         │
│              Nginx / Reverse Proxy                   │
│         tenant-a.orfeo.co / tenant-b.orfeo.co        │
└─────────────────────┬────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────┐
│               BACKEND EN GO                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐  │
│  │  API REST  │  │  Workers   │  │  Scheduler     │  │
│  │  (núcleo)  │  │  (async)   │  │  (alertas)     │  │
│  └────────────┘  └────────────┘  └────────────────┘  │
└──────┬───────────────────┬───────────────────────────┘
       │                   │
┌──────▼──────┐     ┌──────▼──────┐     ┌────────────┐
│ PostgreSQL  │     │    MinIO    │     │   Redis    │
│ (por schema │     │ (archivos   │     │  (colas y  │
│  por tenant)│     │  por bucket)│     │   caché)   │
└─────────────┘     └─────────────┘     └────────────┘

┌──────────────────────────────────────────────────────┐
│            AGENTE LOCAL (opcional)                   │
│     Go binary — corre en PC del funcionario          │
│     Maneja: impresoras, escáneres, código de barras  │
└──────────────────────────────────────────────────────┘
```

### Multi-tenant
- Cada cliente (entidad) tiene su propio **schema en PostgreSQL**
- Cada cliente tiene su propio **bucket en MinIO**
- El routing por subdominio identifica el tenant en cada request
- Datos completamente aislados entre clientes

---

## 4. Módulos del Sistema

### 4.1 Módulos Core

#### Seguridad y Acceso
- Autenticación (usuario/contraseña, futuro SSO)
- Gestión de usuarios
- Roles y permisos por dependencia
- Sesiones y auditoría de acceso
- Recuperación de contraseña

#### Administración
- Configuración de la entidad (nombre, NIT, logo)
- Gestión de dependencias / organigrama
- Parámetros generales del sistema
- Tipos de documento
- Medios de recepción
- Series y subseries documentales (TRD)

#### Radicación de Entrada
- Formulario de radicación (persona natural, jurídica, funcionario)
- Generación automática de número de radicado (formato colombiano)
- Clasificación TRD al momento de radicar
- Tipo de anexo (físico, digital, ninguno)
- Sticker / etiqueta de radicado (PDF imprimible)
- Vinculación con radicados previos (respuestas, derivados)
- Subida de documento digital

#### Radicación de Salida
- Generación de radicado de salida vinculado al de entrada
- Numeración propia para salidas
- Destinatario externo
- Medio de envío (físico, email, fax)
- Sticker de salida

#### Radicación Interna
- Comunicaciones entre dependencias de la misma entidad
- Numeración diferenciada

#### Bandeja de Radicados
- Vista por dependencia y por usuario
- Filtros: fecha, estado, tipo, vencimiento
- Indicadores de alerta (próximos a vencer, vencidos)
- Acciones: ver, redirigir, responder, archivar

#### Flujo y Enrutamiento
- Asignación de radicado a dependencia / usuario
- Historial de trazabilidad completa
- Reasignación y devolución
- Observaciones por movimiento

#### Gestión Documental
- Visualización de documentos (PDF, imágenes)
- Versiones de documento
- Descarga de archivos
- Control de acceso por documento

#### Vencimientos y Alertas
- Configuración de tiempos de respuesta por tipo de documento
- Cálculo automático de fecha límite
- Alertas por email (próximo a vencer, vencido)
- Dashboard de semáforo de vencimientos

#### Búsqueda
- Búsqueda por número de radicado
- Búsqueda por remitente, asunto, fecha, dependencia
- Búsqueda por código de barras (lector)
- Filtros combinados

#### Reportes
- Radicados por período
- Radicados por dependencia
- Tiempos de respuesta
- Inventario documental (FUID)
- Exportación a Excel / PDF

### 4.2 Módulos Normativos (Fase 2)

#### TRD — Tablas de Retención Documental
- Gestión de series documentales
- Gestión de subseries
- Tiempos de retención (gestión / central)
- Disposición final (conservar, eliminar, microfilmar)
- Reportes normativos para el AGN

#### Transferencias Documentales
- Transferencias primarias (gestión → central)
- Transferencias secundarias (central → histórico)
- Actas de transferencia
- Inventario de transferencia

#### PQRSD (Fase 2+)
- Portal ciudadano externo
- Radicación web por el ciudadano
- Consulta de estado por número de radicado
- Notificaciones automáticas al ciudadano

### 4.3 Módulos SaaS (Fase 3)

#### Gestión de Tenants
- Onboarding de nuevos clientes
- Configuración por tenant
- Límites de usuarios y almacenamiento por plan

#### Facturación y Suscripciones
- Planes y precios
- Integración con pasarela de pago colombiana (PSE, Wompi)
- Facturación electrónica (DIAN)
- Gestión de vencimientos de suscripción

#### Panel de Administración Global
- Vista de todos los tenants
- Métricas de uso
- Gestión de suscripciones
- Soporte técnico

---

## 5. Periféricos Soportados

| Periférico | Mecanismo | Fase |
|---|---|---|
| Lector código de barras | HID nativo (funciona como teclado) | POC |
| Impresión PDF / sticker | Generación Go + print navegador | POC |
| Impresora etiquetas (Zebra) | Agente local Go + ZPL | Fase 2 |
| Escáner (carpeta watcher) | Agente local Go detecta archivos | Fase 2 |
| Escáner TWAIN completo | Agente local Go + librería TWAIN | Fase 3 |

### Agente Local
Binario Go pequeño que corre en la PC del funcionario. Instalación única.
- Se conecta al servidor por WebSocket seguro
- Maneja impresión local y detección de documentos escaneados
- Compatible con Windows y Linux
- Actualizaciones automáticas

---

## 6. Modelo de Datos Principal

### Entidades core

```
Tenant (entidad cliente)
├── Dependencias
│   └── Usuarios (con roles)
├── TRD
│   ├── Series
│   │   └── Subseries
│   │       └── Tipos de documento
├── Radicados
│   ├── Remitente
│   ├── Documentos adjuntos
│   ├── Movimientos (trazabilidad)
│   └── Vencimientos
└── Configuración general
```

### Radicado — Campos principales

```
Radicado
├── numero_radicado     (14 dígitos, formato: AAAAMMDD + secuencial)
├── fecha_radicado
├── fecha_documento     (fecha del documento original)
├── tipo_radicado       (entrada / salida / interna)
├── tipo_documento
├── medio_recepcion     (físico / email / fax / web / ventanilla)
├── asunto
├── tipo_remitente      (persona natural / jurídica / funcionario)
├── nombre_remitente
├── identificacion
├── telefono / email / direccion
├── municipio / departamento / pais
├── dependencia_destino
├── usuario_radica
├── usuario_actual
├── referencia_interna
├── clasificacion_trd   (serie → subserie)
├── tipo_anexo          (ninguno / físico / digital)
├── radicado_padre      (si es respuesta o derivado)
├── fecha_vencimiento
└── estado              (activo / archivado / vencido)
```

---

## 7. Fases del Proyecto

---

### FASE 0 — POC (Prueba de Concepto)
**Duración:** 3-4 semanas  
**Objetivo:** Demostrar el concepto al cliente y cerrar el primer contrato

#### Entregables
- [ ] Estructura base del proyecto en Go
- [ ] Autenticación JWT con roles básicos
- [ ] Multi-tenant básico (2 tenants de demo)
- [ ] CRUD de dependencias y usuarios
- [ ] Radicación de entrada funcional
- [ ] Número de radicado con formato colombiano
- [ ] Sticker de radicado en PDF imprimible
- [ ] Bandeja de radicados por dependencia
- [ ] Subida y visualización de documentos
- [ ] Búsqueda básica por número y remitente
- [ ] Lector de código de barras funcional
- [ ] UI presentable (no necesita ser perfecta)
- [ ] Desplegado en VPS con dominio real para demo

#### Semana a semana
```
Semana 1  →  Proyecto Go base, DB, autenticación, dependencias
Semana 2  →  Radicación entrada, numeración, subida de archivos
Semana 3  →  Bandeja, flujo básico, búsqueda, sticker PDF
Semana 4  →  UI pulida, multi-tenant demo, despliegue, ensayo demo
```

#### Criterio de éxito
El cliente puede radicar un documento, verlo en la bandeja, imprimirlo y buscarlo por código de barras. Sin errores visibles en la demo.

---

### FASE 1 — MVP (Producto mínimo vendible)
**Duración:** Meses 1-6 (desde inicio del proyecto)  
**Objetivo:** Primer cliente en producción pagando

#### Entregables adicionales a la POC
- [ ] Radicación de salida vinculada
- [ ] Radicación interna
- [ ] Flujo de enrutamiento completo entre dependencias
- [ ] Historial de trazabilidad por radicado
- [ ] Vencimientos y alertas por email
- [ ] Reasignación y devolución de radicados
- [ ] Reportes básicos (por período, por dependencia)
- [ ] Exportación a Excel y PDF
- [ ] Panel de administración completo
- [ ] Gestión de tipos de documento y medios de recepción
- [ ] Seguridad robusta (rate limiting, auditoría, logs)
- [ ] Backups automáticos
- [ ] Documentación de usuario

#### Hito clave
**Mes 6:** Cliente piloto en producción con contrato firmado.

---

### FASE 2 — Cumplimiento AGN completo
**Duración:** Meses 7-12  
**Objetivo:** Producto completo vendible a cualquier entidad pública colombiana

#### Entregables
- [ ] TRD completa (series, subseries, disposición final)
- [ ] Clasificación TRD en radicación
- [ ] Transferencias documentales primarias y secundarias
- [ ] Actas de transferencia
- [ ] Inventario Documental Único (FUID)
- [ ] Reportes normativos para el AGN
- [ ] PQRSD básico (portal ciudadano)
- [ ] Agente local (impresoras de etiquetas, escáner por carpeta)
- [ ] Integración firma electrónica (Certicámara)
- [ ] LDAP / Active Directory (para entidades que lo requieran)

#### Hito clave
**Mes 10:** Segundo cliente en producción. Producto referenciable.

---

### FASE 3 — Plataforma SaaS escalable
**Duración:** Meses 13-18  
**Objetivo:** Producto SaaS con onboarding automatizado y facturación

#### Entregables
- [ ] Onboarding self-service (el cliente se registra solo)
- [ ] Gestión de planes y suscripciones
- [ ] Integración pasarela de pago (PSE / Wompi)
- [ ] Facturación electrónica DIAN
- [ ] Panel de administración global (multi-tenant management)
- [ ] Métricas de uso por tenant
- [ ] Portal PQRSD avanzado con notificaciones al ciudadano
- [ ] App móvil básica (consulta y aprobaciones)
- [ ] API pública documentada para integraciones
- [ ] Kubernetes para alta disponibilidad
- [ ] Monitoreo y alertas de infraestructura
- [ ] SLA documentado por plan

#### Hito clave
**Mes 18:** 10+ clientes activos. Ingresos recurrentes estables.

---

## 8. Normativa de Referencia

| Norma | Descripción | Impacto en el sistema |
|---|---|---|
| Ley 594 de 2000 | Ley General de Archivos | Base de toda la gestión documental |
| Decreto 1080 de 2015 | Único Reglamentario cultura/archivos | TRD, transferencias, disposición |
| Acuerdo 003 AGN 2015 | Digitalización de documentos | Requisitos de calidad de escaneo |
| Acuerdo 006 AGN 2014 | Documentos electrónicos | Metadatos, autenticidad, integridad |
| Ley 527 de 1999 | Comercio electrónico y firma digital | Validez de documentos digitales |
| Decreto 2364 de 2012 | Firma electrónica | Integración con Certicámara |

---

## 9. Modelo de Negocio

### Planes de suscripción

| Plan | Usuarios | Almacenamiento | Precio / mes |
|---|---|---|---|
| Básico | Hasta 20 | 10 GB | $800.000 — $1.200.000 COP |
| Profesional | Hasta 100 | 50 GB | $2.000.000 — $3.500.000 COP |
| Institucional | Ilimitados | 200 GB | $5.000.000 — $8.000.000 COP |
| On-premise | Ilimitados | Propio | Licencia anual negociable |

### Servicios adicionales
- Implementación y configuración inicial
- Migración de datos desde Orfeo existente
- Capacitación a usuarios
- Soporte prioritario
- Desarrollo de integraciones a la medida

### Proyección de ingresos

| Clientes | Plan promedio | Ingreso mensual recurrente |
|---|---|---|
| 5 | Profesional | ~$12.500.000 COP |
| 15 | Profesional | ~$37.500.000 COP |
| 30 | Mixto | ~$65.000.000 COP |

---

## 10. Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| Cambios en normativa AGN | Baja | Alto | Arquitectura modular — fácil de actualizar |
| Cliente piloto abandona | Media | Alto | Contrato con hito de pago al inicio |
| Competencia (Orfeo NG) | Media | Medio | Diferenciador en precio y soporte local |
| Deuda técnica temprana | Alta | Medio | Code review disciplinado desde la POC |
| Brechas de seguridad | Baja | Muy alto | Auditoría de seguridad antes de cada release |
| Escalabilidad en SaaS | Baja (corto plazo) | Alto | PostgreSQL schemas + MinIO escalan bien |

---

## 11. Decisiones Técnicas Pendientes

- [ ] Framework frontend: Next.js vs SvelteKit vs Vue
- [ ] Framework backend Go: Gin vs Echo vs Chi
- [ ] Estrategia de migraciones DB: golang-migrate vs atlas
- [ ] Generación de PDFs: gofpdf vs chromedp (headless Chrome)
- [ ] Proveedor de nube principal para el SaaS
- [ ] Estrategia de backups automatizados
- [ ] Política de retención de logs

---

## 12. Referencias Técnicas

### Repositorios de referencia (lógica de dominio, no código)
- `github.com/themonki/metrocali-orfeogpl` — adaptación Metro Cali, referencia de campos
- `github.com/c-m-a/orfeo` — Orfeo Plus v4.0, mejor mapa de módulos disponible

### Estructura de módulos extraída de Orfeo
```
radicacion/     →  campos y flujo de radicados de entrada
radsalida/      →  radicados de salida
expediente/     →  expedientes y carpetas
flujo/          →  enrutamiento entre dependencias
trd/            →  series, subseries, retención
alertas/        →  vencimientos y notificaciones
estadisticas/   →  reportes
seguridad/      →  usuarios, roles, permisos
```

---

## 13. Próximos Pasos Inmediatos

1. **Definir framework frontend** — según preferencia del desarrollador
2. **Crear repositorio del proyecto** — estructura base de carpetas Go
3. **Diseñar esquema de base de datos** — tablas iniciales para la POC
4. **Levantar entorno de desarrollo** — Docker Compose con Go + PostgreSQL + MinIO
5. **Iniciar Semana 1 de la POC** — autenticación y dependencias

---

*Documento vivo — se actualiza en cada iteración del proyecto.*  
*Versión 1.0 — Abril 2026*
