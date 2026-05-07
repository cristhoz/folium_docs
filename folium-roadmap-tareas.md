# Folium — Roadmap de Tareas

**Versión:** 1.1 | **Fecha:** 2026-05-07 | **Proyecto:** Folium — foliumhq.co | **Uso interno**

> Documento vivo. Fuente de verdad del estado de ejecución del proyecto.  
> Los detalles técnicos de cada tarea viven en sus documentos fuente (ver referencias al pie).

---

## Estado Rápido

| Área | Progreso | Próxima acción |
|------|----------|----------------|
| Backend Go — POC | 🟡 En curso | Autenticación JWT |
| BFF — Config base | ✅ Completo | — |
| BFF — SSR híbrido | ✅ Completo | — |
| BFF — Sesiones (A) | 🟡 En curso | A.3 POST /auth/login (bloqueado por backend Go) |
| BFF — Headers (B) | 🟡 Casi listo | B.5 verificar securityheaders.com |
| BFF — CSRF (C) | 🟡 Casi listo | C.6 integrar CSRF en client.ts |
| BFF — Docs (D) | 🔴 Pendiente | D.1 validación MIME-type |
| Frontend | 🟡 En curso | C.6 integrar CSRF en client.ts |

---

## 🚧 Sprint Actual — BFF + POC Backend

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

### Backend Go — POC
- [x] Estructura base del proyecto en Go
- [ ] **Autenticación JWT con roles básicos**
- [ ] Multi-tenant básico (2 tenants de demo)
- [ ] CRUD de dependencias y usuarios
- [ ] Endpoint `POST /api/auth/login` con respuesta JWT (contrato con BFF)

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
- [ ] **BE.1** Restringir CORS del backend Go solo al origen BFF (var `BFF_ORIGIN`)
- [ ] **BE.2** Middleware de validación de header `X-Service-Token` en todas las rutas protegidas
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
| 1 | Migraciones DB | golang-migrate vs atlas | Antes de POC |
| 2 | Generación de PDFs | gofpdf vs chromedp | Durante POC |
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

---

## Changelog

| Versión | Fecha      | Cambio |
|---------|------------|--------|
| 1.1     | 2026-05-07 | A.1-A.2 y FE.1 marcadas completas; nueva sección SSR en estado rápido; 9 nuevas entradas en ✅ Completado; React Islands movido a Fase 3 |
| 1.0     | 2026-05-07 | Creación del roadmap consolidado; estados iniciales según progreso real |
