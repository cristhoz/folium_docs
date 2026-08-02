# Folium — Roadmap de Tareas

**Versión:** 1.6 | **Fecha:** 2026-08-01 | **Proyecto:** Folium — foliumhq.co | **Uso interno**

> Documento vivo. Fuente de verdad del estado de ejecución del proyecto.  
> Los detalles técnicos de cada tarea viven en sus documentos fuente (ver referencias al pie).

---

## Estado Rápido

| Área | Progreso | Próxima acción |
|------|----------|----------------|
| **v1 Single-tenant (repo nuevo)** | 🟢 Prompt #2 de QA ejecutado y validado, 0 Críticos/Altos abiertos | Resolver pendientes de QA (DOCX/XLSX, riesgos no cubiertos) → desplegar en VPS |
| Backend Go — Auth | ⏸️ Pausado (track SaaS) | — |
| Backend Go — Seguridad | ⏸️ Pausado (track SaaS) | BE.3 documentar contrato BFF→Backend |
| Backend Go — POC | ⏸️ Pausado (track SaaS) | Multi-tenant, CRUD, radicación |
| BFF — Config base | ⏸️ Pausado (track SaaS) | — |
| BFF — SSR híbrido | ⏸️ Pausado (track SaaS) | — |
| BFF — Sesiones (A) | ⏸️ Pausado (track SaaS) | A.3–A.6 listos para implementar |
| BFF — Headers (B) | ⏸️ Pausado (track SaaS) | B.5 verificar securityheaders.com |
| BFF — CSRF (C) | ⏸️ Pausado (track SaaS) | C.6 integrar CSRF en client.ts |
| BFF — Docs (D) | ⏸️ Pausado (track SaaS) | D.1 validación MIME-type |
| Frontend | ⏸️ Pausado (track SaaS) | C.6 integrar CSRF en client.ts |

---

## 🎯 Pivote — v1 Single-tenant (decidido 2026-08-01)

La primera versión ejecutable de Folium **no es SaaS**: se construye single-tenant para una sola entidad, con el alcance funcional de la Fase 1 completa, para cumplir una entrega y generar valor. La idea de negocio y la arquitectura SaaS (multi-tenant, BFF, Garage, Valkey) siguen vigentes como objetivo posterior; el trabajo existente del track SaaS queda **en pausa**, no descartado.

El desarrollo de la v1 se itera vía LLM en un **repositorio nuevo** (`folium`), a partir de dos prompts de un solo tiro:

| Prompt | Archivo | Propósito |
|--------|---------|-----------|
| #1 Construcción | `folium-prompt-fase1-v1.md` | Construye el sistema completo desde cero |
| #2 Verificación/QA | `folium-prompt-fase1-qa.md` | Audita contra la spec, informa hallazgos y corrige Críticos/Altos |

### Decisiones cerradas para la v1

| Decisión | v1 Single-tenant | Diseño SaaS (v2+) |
|----------|------------------|-------------------|
| Tenancy | Un solo schema `public`, sin `tenant_id` ni RLS; tabla `entity_settings` | Schema por tenant + RLS |
| Frontend | SPA embebida en el binario Go (`embed.FS`), mismo origen | BFF Express + SSR |
| Autenticación | Sesiones server-side: cookie HttpOnly + tabla `sessions` (sin JWT) | JWT + refresh + blocklist JTI |
| Adjuntos | Disco local vía interfaz `Storage` | Garage, bucket por tenant |
| Caché/colas | Ninguna — scheduler in-process (goroutine) | Valkey |
| Workflow | Estados fijos en código (`filed → assigned → in_progress → replied → archived`) | Máquina de estados configurable |
| PII (Ley 1581) | HMAC-SHA256 + AES-256-GCM desde el día uno | Igual |
| Roles | 4 fijos: `admin`, `ventanilla`, `jefe_dependencia`, `funcionario` | Configurables (JSONB) |
| Migraciones | golang-migrate | — (decisión heredada) |
| PDF / barcode | gofpdf + boombuler/barcode | — (decisión heredada) |

### Tareas v1

- [x] Iterar plan y cerrar decisiones técnicas de la v1
- [x] Generar prompt #1 de construcción (`folium-prompt-fase1-v1.md`)
- [x] Generar prompt #2 de verificación/QA (`folium-prompt-fase1-qa.md`)
- [x] Ejecutar prompt #1 en repo nuevo `folium` (sesión LLM aparte) — ver `DECISIONES.md` del repo `folium` y riesgos abiertos abajo
- [x] Ejecutar prompt #2 de QA sobre el resultado — ver `QA-INFORME.md` en el repo `folium` (commits `2385417`…`0e26575`, informe en `163f6a1`); verificado independientemente contra el repo real (build/vet/lint/tests en verde, los 5 commits de corrección hacen lo que dicen)
- [x] Revisar `QA-INFORME.md` y decidir si amerita prompt #3 de pulido — **sí amerita**: 0 Críticos/Altos quedaron abiertos, pero el QA no cubrió el riesgo de seguridad de DOCX/XLSX ya señalado abajo (violación de spec no detectada) ni los demás riesgos de integridad de datos; ver "Pendientes tras el QA" abajo
- [ ] Resolver pendientes tras el QA (ver sección nueva abajo) — candidatos a un prompt #3 acotado
- [ ] Desplegar en VPS para la entrega

### ⚠️ Riesgos abiertos de la ejecución del prompt #1 (estado tras el QA del prompt #2)

Decisiones que el LLM tomó durante la construcción y que no estaban explícitas en el prompt. El QA (prompt #2) corrigió 2 de estos (marcados ✅) pero **no cubrió ni mencionó el resto** — siguen abiertos y deben resolverse antes de producción:

**Seguridad**
- ⚠️ **Aún abierto y NO cubierto por el QA** — Validación de DOCX/XLSX solo confirma que el archivo es un ZIP válido (`internal/storage/mime.go:27-29`, comentario propio: "docx/xlsx are both zip containers; caller disambiguates via extension") y confía en la extensión — no inspecciona `[Content_Types].xml`. Permite subir cualquier ZIP renombrado como `.docx`/`.xlsx`. Esto **viola directamente** el requisito explícito de la spec del prompt #2 ("validación MIME por magic bytes") y el caso de prueba 1.4 del QA solo probó un `.exe` renombrado a `.pdf` (que sí es rechazado), nunca un ZIP genérico renombrado a `.docx`/`.xlsx` (que pasaría). Verificado directamente en el código el 2026-08-01: el problema sigue presente en HEAD (`61d0d52`) y el `QA-INFORME.md` no lo menciona en ningún hallazgo.
- ✅ Resuelto por el QA — Cookie `folium_sid` con `Secure` rompía el login en despliegues por IP de LAN sin TLS. Corregido como `QA-001` (commit `2385417`): ahora configurable vía `COOKIE_SECURE`, con `true` como default seguro.
- `PII_HMAC_KEY` acepta mínimo 16 bytes; el prompt y `.env.example` sugieren 32 para ambas llaves. Con HMAC-SHA256 lo recomendado es llave ≥ 32 bytes para margen de seguridad completo — sigue sin homologar, el QA no lo tocó.

**Integridad de datos / consistencia** *(ninguno cubierto por el QA — fuera del alcance de su spec de prueba)*
- La acción "Responder" no es atómica: si falla la creación del radicado de salida tras marcar el de entrada como `replied`, queda huérfano sin mecanismo de detección. Falta un chequeo periódico de "`replied` sin salida vinculada".
- Redundancia entre `records.parent_record_id` y `outgoing_record_details.origin_record_id` (ambos enlazan salida↔entrada) — aclarar cuál es la fuente de verdad antes de construir reportes sobre esto.
- El estado `filed` se reutiliza tanto para "radicado nuevo" como para "devuelto/redirigido pendiente de reasignación" — distinguible solo vía `record_history.action`. Cualquier filtro de bandeja/reporte que use solo `status` puede mezclar ambos casos.
- El semáforo de vencimiento en frontend solo cuenta fines de semana (no festivos colombianos), mientras el cálculo autoritativo del backend sí los considera — cerca de un festivo el color puede no coincidir con la urgencia real de un plazo legal.

**Build/despliegue**
- Lockfile doble (pnpm en dev, `npm ci` en Docker/Makefile) — persiste, ahora documentado formalmente como `QA-007` (Bajo, no corregido por decisión explícita del QA). Confirmar que `package-lock.json` está commiteado y sincronizado con `pnpm-lock.yaml`, o eliminar uno de los dos gestores.
- Migraciones automáticas al arrancar el binario son correctas para una sola instancia; si más adelante se corre más de una réplica, pueden generar condiciones de carrera — revisar antes de escalar horizontalmente. El QA no probó múltiples réplicas (fuera de alcance de single-tenant v1).

### 📋 Pendientes tras el QA (prompt #2) — validado 2026-08-01

El `QA-INFORME.md` fue verificado de forma independiente contra el repo real `folium_single_tenant` (commits `9e1b747`…`61d0d52`): build, `go vet`, `golangci-lint`, tests, `tsc`/ESLint/build de `web/` en verde; los 5 commits de corrección (`QA-001`…`QA-005`) efectivamente implementan lo que documentan, con sus tests de regresión presentes y en verde. El informe es confiable en lo que afirma. Sin embargo, quedan pendientes que el QA no cubrió:

1. **Prioridad alta — DOCX/XLSX magic bytes** (ver arriba): es una violación de spec explícita, no un hallazgo Bajo. Candidato directo a un prompt #3 acotado: exigir inspección real de `[Content_Types].xml` dentro del ZIP antes de aceptar `.docx`/`.xlsx`, con test que sube un ZIP arbitrario renombrado y espera 415.
2. **QA-006 (Bajo, no corregido):** `pdftotext` no extrae el número completo del sticker PDF (tabla `ToUnicode` incompleta en la fuente subconjunto de gofpdf). Sin pérdida de datos funcionales; pendiente de investigación en la generación de fuentes si se prioriza accesibilidad del PDF.
3. **QA-007 (Bajo, no corregido):** mezcla de artefactos npm/pnpm en `web/` — decidir un único gestor de paquetes.
4. **Riesgos de integridad de datos pre-QA sin tocar:** atomicidad de "Responder", redundancia `parent_record_id`/`origin_record_id`, ambigüedad del estado `filed` reutilizado, semáforo de frontend sin festivos colombianos — ninguno estaba en el alcance de prueba del prompt #2 (que se ciñó a la spec original), pero siguen siendo riesgo real antes de producción.
5. Homologar `PII_HMAC_KEY` a mínimo 32 bytes (actualmente acepta 16).

---

## ⏸️ Track SaaS — BFF + POC Backend *(en pausa desde 2026-08-01 — ver Pivote v1)*

### BFF — Backlog Item A: Gestión de Sesiones *(Crítica — bloquea todo)*
- [x] **A.1** Instalar `express-session`, `connect-redis`, `ioredis`
- [x] **A.2** Configurar Redis como store de sesiones (`connect-redis` + `express-session`)
- [ ] **A.3** Implementar `POST /auth/login` — proxy al backend Go, guardar JWT en Redis, devolver solo cookie `sid` HttpOnly
- [ ] **A.4** Implementar `POST /auth/logout` — destruir sesión en Redis, limpiar cookie
- [ ] **A.5** Implementar middleware de autenticación BFF — resolver JWT desde Redis, adjuntar `Authorization` + `X-Service-Token`
- [ ] **A.6** Implementar `GET /auth/me` — devolver perfil de sesión activa (sin JWT)

### BFF — Backlog Item B: Headers de Seguridad *(casi listo)*
- [x] B.1 Instalar `helmet`
- [x] B.2 CSP estricta (`default-src 'self'`, anti-XSS, anti-clickjacking)
- [x] B.3 `Referrer-Policy: no-referrer`
- [x] B.4 HSTS `max-age=31536000; includeSubDomains`
- [ ] **B.5** Verificar calificación en securityheaders.com contra staging

### BFF — Backlog Item C: Anti-CSRF *(casi listo — falta integración frontend)*
- [x] C.1 Instalar `@dr.pogodin/csurf`
- [x] C.2 Middleware CSRF con Double Submit Cookie
- [x] C.3 Endpoint `GET /api/csrf-token`
- [x] C.4 Middleware aplicado a `POST`, `PUT`, `PATCH`, `DELETE`
- [x] C.5 BFF rechaza mutaciones sin `X-CSRF-Token` válido (`403`)
- [ ] **C.6** Actualizar `app/services/client.ts`: obtener token CSRF al iniciar, adjuntarlo en mutaciones, manejar `419` con reintento

### Backend Go — Auth *(completado)*
- [x] Autenticación JWT con roles básicos (`pkg/crypto/jwt.go` — HS256, `FoliumClaims`)
- [x] Blocklist de JTIs en Redis — fail-closed (`pkg/cache/redis_blocklist.go`)
- [x] Tipos de dominio de autenticación (`internal/domain/auth.go`)
- [x] Servicio de autenticación — login, refresh con token rotation, logout (`internal/service/auth_service.go`)
- [x] Repositorio de sesiones en PostgreSQL (`internal/repository/session_repo.go`)
- [x] Middleware `RequireAuth` — validación JWT + blocklist + tenant (`internal/middleware/auth.go`)
- [x] Handlers HTTP — login, refresh, logout, me (`internal/handler/auth.go`)
- [x] Endpoints: `POST /v1/auth/login`, `POST /v1/auth/refresh`, `POST /v1/auth/logout`, `GET /v1/auth/me`

### Backend Go — POC
- [x] Estructura base del proyecto en Go
- [ ] Multi-tenant básico (2 tenants de demo)
- [ ] CRUD de dependencias y usuarios
- [ ] Endpoint radicación de entrada (integración con Garage)

### Frontend — Ajustes post-BFF
- [x] **FE.1** `useAuthStore` (Zustand 5): sin JWT, guarda `{ userId, nombre, rol, dependencia, tenantId }`
- [ ] **FE.2** `app/services/client.ts`: `withCredentials: true`, integrar CSRF (C.6)
- [ ] **FE.3** Auditar y eliminar `localStorage`/`sessionStorage` para datos de autenticación
- [ ] **FE.4** Flujo de login: guardar perfil en Zustand (no JWT) tras `POST /auth/login` exitoso
- [ ] **FE.5** `.env.example`: confirmar que `VITE_API_URL` apunta al BFF y no al backend Go

---

## 🔜 Siguiente Sprint — BFF completo + POC funcional

### BFF — Backlog Item D: Gestión Documental Segura
- [ ] **D.1** Middleware de validación MIME-type real con `file-type` (magic bytes) — rechazar con `415`
- [ ] **D.2** Validación de tamaño máximo de archivo (`MAX_FILE_SIZE_MB` por env var)
- [ ] **D.3** Middleware de auditoría: `userId`, `action`, `timestamp`, `ip` → JSON a stdout (compatible Loki)
- [ ] **D.4** Sanitización de metadatos: escapar HTML, rechazar script injection en campos de documentos

### Backend Go — Ajustes de Seguridad
- [x] **BE.1** Restringir CORS del backend Go solo al origen BFF (var `BFF_ORIGIN`)
- [x] **BE.2** Middleware de validación de header `X-Service-Token` en todas las rutas protegidas
- [ ] **BE.3** Documentar contrato BFF → Backend en `folium_backend/docs/bff-contract.md`

### Backend Go — POC (continuación)
- [ ] Radicación de entrada funcional
- [ ] Número de radicado con formato colombiano (`AAAAMMDD-E-NNNNNN`)
- [ ] Subida de documentos a Garage (presigned URLs, TTL 60s vista / 5min descarga)
- [ ] Bandeja de radicados por dependencia

---

## 📋 POC Completa — Criterio: Demo al cliente

- [ ] Sticker de radicado en PDF imprimible
- [ ] Búsqueda básica por número de radicado y remitente
- [ ] Lector de código de barras funcional (HID nativo)
- [ ] UI presentable
- [ ] Desplegado en VPS con dominio real (`app.foliumhq.co`) para demo

**Criterio de éxito:** El cliente puede radicar, ver en bandeja, imprimir sticker y buscar por código de barras. Sin errores visibles.

---

## 🗓️ Fase 1 — MVP (Meses 1-6)

> **2026-08-01:** Este alcance se entrega mediante la **v1 single-tenant** (ver sección Pivote), no sobre el track SaaS.

*Desbloquear: primer cliente en producción pagando.*

- [ ] Radicación de salida vinculada al radicado de entrada
- [ ] Radicación interna entre dependencias
- [ ] Flujo de enrutamiento completo (asignación, reasignación, devolución)
- [ ] Historial de trazabilidad completo por radicado
- [ ] Vencimientos: cálculo automático de fecha límite + alertas por email
- [ ] Reportes básicos (por período, por dependencia) + exportación Excel/PDF
- [ ] Panel de administración completo (tipos de documento, medios de recepción)
- [ ] Seguridad robusta: rate limiting, auditoría completa, logs estructurados
- [ ] Backups automáticos de PostgreSQL y Garage
- [ ] Documentación de usuario

**Hito:** Mes 6 — cliente piloto en producción con contrato firmado.

---

## 🗓️ Fase 2 — Cumplimiento AGN (Meses 7-12)

- [ ] TRD completa: series, subseries, disposición final
- [ ] Clasificación TRD al momento de radicar
- [ ] Transferencias documentales primarias y secundarias
- [ ] Actas de transferencia + Inventario Documental Único (FUID)
- [ ] Reportes normativos para el AGN
- [ ] PQRSD básico (portal ciudadano)
- [ ] Agente local Go: impresoras de etiquetas Zebra, escáner por carpeta
- [ ] Integración firma electrónica (Certicámara)
- [ ] LDAP / Active Directory (opcional por entidad)

**Hito:** Mes 10 — segundo cliente en producción.

---

## 🗓️ Fase 3 — SaaS Escalable (Meses 13-18)

- [ ] Onboarding self-service (cliente se registra solo)
- [ ] Gestión de planes y suscripciones
- [ ] Integración pasarela de pago (PSE / Wompi) + facturación DIAN
- [ ] Panel de administración global multi-tenant
- [ ] Portal PQRSD avanzado con notificaciones al ciudadano
- [ ] App móvil básica (consulta y aprobaciones)
- [ ] API pública documentada para integraciones
- [ ] Kubernetes para alta disponibilidad + SLA por plan
- [ ] React Islands (hidratación parcial en módulos críticos de gestión documental)

**Hito:** Mes 18 — 10+ clientes activos.

---

## ⚙️ Decisiones Técnicas Pendientes

| # | Decisión | Opciones | Urgencia |
|---|----------|----------|----------|
| 1 | ~~Migraciones DB~~ | ✅ **golang-migrate** (cerrada en v1, 2026-08-01) | — |
| 2 | ~~Generación de PDFs~~ | ✅ **gofpdf** + boombuler/barcode (cerrada en v1, 2026-08-01) | — |
| 3 | OCR para PDFs escaneados | Tesseract / AWS Textract / Google Document AI | Fase 1 |
| 4 | IA extracción de metadatos de cartas | LLM local vs API externa (Claude, OpenAI) | Fase 1 |
| 5 | Proveedor de nube SaaS | AWS / DigitalOcean / Hetzner | Antes de Fase 1 |
| 6 | Estrategia de backups | Por definir | Fase 1 |
| 7 | Política de retención de logs | Por definir | Fase 1 |

---

## ✅ Completado

| Tarea | Fecha aprox. |
|-------|-------------|
| Stack decidido: Vite 6 + React 19 + TypeScript 5.8 + Zustand 5 | 2026-05-07 |
| Router decidido: TanStack Router v1 (file-based) | 2026-05-07 |
| Framework Go decidido: Chi | 2026-05-07 |
| Almacenamiento: Garage (MinIO descartado) | 2026-04-XX |
| Cache/colas: Valkey (Redis descartado en producción) | 2026-04-XX |
| Schema de BD: multi-tenant por schema + RLS | 2026-05-XX |
| Estructura base del proyecto en Go | 2026-05-XX |
| Backend Go — Autenticación JWT completa: login, refresh, logout, me (`/v1/auth/*`) | 2026-05-07 |
| Backend Go — Middleware `RequireAuth`: verificación JWT + blocklist Redis (fail-closed) + validación cruzada tenant | 2026-05-07 |
| Backend Go — Token rotation en refresh + hash SHA-256 del refresh token en DB | 2026-05-07 |
| Backend Go — Blocklist JTI con TTL exacto en Redis (`pkg/cache/redis_blocklist.go`) | 2026-05-07 |
| Backend Go — Repositorio de sesiones en PostgreSQL (tabla `sessions`) | 2026-05-07 |
| BFF — Config base del servidor (0.1–0.6) | 2026-05-XX |
| BFF — Headers de seguridad (B.1–B.4) | 2026-05-07 |
| BFF — Protección Anti-CSRF (C.1–C.5) | 2026-05-07 |
| BFF — A.1-A.2: infraestructura de sesiones (express-session, connect-redis, ioredis) | 2026-05-07 |
| SSR híbrido: entry-server.tsx + entry-client.tsx + middlewares vite-dev/prod-resolve | 2026-05-07 |
| TanStack Router v1 file-based configurado (routeTree.gen.ts generado automáticamente) | 2026-05-07 |
| useAuthStore (Zustand 5) — perfil sin JWT; useRadicadoStore — esqueleto | 2026-05-07 |
| services/client.ts (Axios 1 + interceptores 401/403/5xx); auth.ts, radicados.ts | 2026-05-07 |
| types/: AuthUser, Radicado (espejo de structs Go del backend) | 2026-05-07 |
| CI local: Lefthook (pre-commit lint + format-check) + Commitlint (Conventional Commits) | 2026-05-07 |
| CSS: SCSS + Stylelint (recess-order) | 2026-05-07 |
| Logging BFF: Pino 10 + pino-http (redacción automática de Authorization + Cookie) | 2026-05-07 |
| Backend Go — BE.1: CORS restringido al origen BFF (`BFF_ORIGIN`), fail-fast si no está definido | 2026-05-07 |
| Backend Go — BE.2: Middleware `RequireServiceToken` con `subtle.ConstantTimeCompare` en rutas protegidas | 2026-05-07 |
| Pivote v1 single-tenant: decisiones cerradas (SPA embebida, disco local, PII cifrada, estados fijos, sesiones sin JWT) | 2026-08-01 |
| Prompt #1 de construcción v1 (`folium-prompt-fase1-v1.md`) y prompt #2 de QA (`folium-prompt-fase1-qa.md`) | 2026-08-01 |
| Prompt #2 de QA ejecutado sobre `folium`: 2 hallazgos Crítico/Alto y 3 Medio corregidos (`QA-001`…`QA-005`), 2 Bajo documentados sin corregir; verificado independientemente contra el repo real | 2026-08-01 |

---

## Referencias

| Documento | Contenido |
|-----------|-----------|
| `folium-plan-proyecto.md` | Plan maestro, fases, stack, modelo de negocio |
| `BFF-tareas-implementacion.md` | Backlog detallado de implementación BFF |
| `BFF-arquitectura-front.md` | Arquitectura de seguridad y renderizado del BFF |
| `folium-schema-base-datos.md` | Schema de BD y decisiones de modelo de datos |
| `folium-workflows-y-privacidad.md` | Flujos de trabajo, privacidad (Ley 1581), trazabilidad |
| `comparativa-orfeo-alfresco.md` | Análisis competitivo para propuesta comercial |
| `folium-prompt-fase1-v1.md` | Prompt #1 — construcción de la v1 single-tenant desde cero |
| `folium-prompt-fase1-qa.md` | Prompt #2 — verificación, QA y corrección de la v1 |

---

## Changelog

| Versión | Fecha      | Cambio |
|---------|------------|--------|
| 1.6     | 2026-08-01 | Prompt #2 de QA validado contra el repo real `folium_single_tenant` (commits `9e1b747`…`61d0d52`): build/vet/lint/tests en verde, los 5 commits de corrección hacen lo que documentan. Riesgos abiertos actualizados (DOCX/XLSX marcado como violación de spec no cubierta por el QA); nueva sección "Pendientes tras el QA" con candidatos a prompt #3 |
| 1.5     | 2026-08-01 | Prompt #1 ejecutado en el repo `folium`; marcado en Estado Rápido y Tareas v1; nueva sección "Riesgos abiertos de la ejecución del prompt #1" con hallazgos de seguridad, consistencia de datos y build a resolver antes del QA |
| 1.4     | 2026-08-01 | Pivote v1 single-tenant: nueva sección con decisiones cerradas y tareas; track SaaS/BFF marcado en pausa; decisiones 1 y 2 cerradas (golang-migrate, gofpdf); prompts #1 y #2 en referencias |
| 1.3     | 2026-05-07 | BE.1 y BE.2 marcadas ✅ completas; nueva fila Backend Go — Seguridad en Estado Rápido; 2 nuevas entradas en ✅ Completado |
| 1.2     | 2026-05-07 | Backend Go — Auth marcado ✅ completo; BFF Sesiones desbloqueado; sección Auth completada en Sprint Actual; 5 nuevas entradas en ✅ Completado |
| 1.1     | 2026-05-07 | A.1-A.2 y FE.1 marcadas completas; nueva sección SSR en estado rápido; 9 nuevas entradas en ✅ Completado; React Islands movido a Fase 3 |
| 1.0     | 2026-05-07 | Creación del roadmap consolidado; estados iniciales según progreso real |
