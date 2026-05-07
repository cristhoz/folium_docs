# Tareas de Implementación — Capa BFF Segura

**Versión:** 1.2 | **Fecha:** Mayo 2026 | **Proyecto:** Folium — foliumhq.co | **Uso interno**

Documento de trabajo derivado de `BFF-arquitectura-front.md`. Cubre todas las tareas necesarias para implementar la capa BFF en Folium bajo estándares MinTIC/OWASP.

---

## Fase 0 — Configuración Base del Servidor BFF

| # | Tarea | Prioridad |
|---|-------|-----------|
| ~~0.1~~ | ~~Inicializar proyecto Node.js + TypeScript en `folium_bff/`~~ | ✅ |
| ~~0.2~~ | ~~Instalar dependencias base: `express`, `typescript`, `ts-node`, `@types/express`~~ | ✅ |
| ~~0.3~~ | ~~Integrar Vite como middleware de Express para desarrollo~~ | ✅ |
| ~~0.4~~ | ~~Configurar modo producción: Express sirve el build estático de Vite (`dist/`)~~ | ✅ |
| ~~0.5~~ | ~~Añadir servicio `folium_bff` y `redis` al `docker-compose.yml` del proyecto~~ | ✅ |
| ~~0.6~~ | ~~Definir variables de entorno del BFF~~ | ✅ |

---

## SSR + Router (Completado)

- [x] Implementar `app/entry-server.tsx` (renderToString + TanStack Router + createMemoryHistory)
- [x] Implementar `app/entry-client.tsx` (hidratación en el cliente)
- [x] Middleware `api/middlewares/vite-dev-resolve.ts` (dev: Vite HMR vía `ssrLoadModule`)
- [x] Middleware `api/middlewares/vite-prod-resolve.ts` (prod: bundles desde `dist/client` + `dist/server`)
- [x] Configurar `@tanstack/router-vite-plugin` (file-based routing → `routeTree.gen.ts`)
- [x] Rutas iniciales: `__root.tsx`, `_auth.tsx`, `index.tsx`, `login.tsx`
- [x] `scripts/build-server.mjs` (esbuild bundle del servidor → `dist-server/`)

---

## CI Local (Completado)

- [x] Lefthook: hook `pre-commit` con lint + format-check
- [x] Commitlint: Conventional Commits configurado

---

## Backlog Item A — Gestión de Sesiones Token-per-Session

> **Prioridad: Crítica** — Requisito para que el JWT nunca sea expuesto al navegador.
> **Bloqueado por:** endpoint `POST /api/auth/login` del backend Go (aún pendiente).

- [x] **A.1** Instalar: `express-session`, `connect-redis`, `ioredis`
- [x] **A.2** Configurar Redis como store de sesiones en Express (`connect-redis` + `express-session`)
- [ ] **A.3** Implementar `POST /auth/login`:
  - Recibir credenciales del frontend
  - Hacer proxy al backend Go (`POST /api/auth/login`)
  - Extraer el JWT de la respuesta del backend
  - Guardar el JWT en Redis vinculado al `SessionID` (TTL igual al `exp` del JWT)
  - Devolver al navegador solo una cookie con flags: `HttpOnly`, `Secure`, `SameSite: Strict`
  - Devolver al frontend el perfil del usuario (nombre, rol, tenantId) — **sin JWT**
- [ ] **A.4** Implementar `POST /auth/logout`:
  - Destruir la sesión en Redis (`req.session.destroy()`)
  - Limpiar la cookie en el navegador
- [ ] **A.5** Implementar middleware de autenticación BFF:
  - Leer `SessionID` de la cookie `sid`
  - Resolver el JWT en Redis
  - Adjuntar `Authorization: Bearer <token>` + `X-Service-Token` a la petición hacia el backend Go
  - Devolver `401` si la sesión no existe o expiró
- [ ] **A.6** Implementar `GET /auth/me`: devolver el perfil del usuario de la sesión activa (sin JWT)

**Criterio de aceptación:** El JWT no es visible en el Application Tab (LocalStorage / Cookies) del navegador en ningún momento.

---

## Backlog Item B — Blindaje de Headers (MinTIC / OWASP)

> **Prioridad: Alta**

- [x] **B.1** Instalar: `helmet`
- [x] **B.2** Configurar `helmet()` con Content Security Policy (CSP) estricta:
  - `default-src 'self'`
  - `script-src 'self'` (bloquear scripts externos e inline)
  - `style-src 'self' 'unsafe-inline'` (solo si necesario para la UI)
  - `img-src 'self' data:`
  - `connect-src 'self'` (solo al mismo origen BFF)
  - `frame-ancestors 'none'` (anti-clickjacking)
- [x] **B.3** Configurar `Referrer-Policy: no-referrer`
- [x] **B.4** Configurar HSTS: `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- [ ] **B.5** Verificar headers con [securityheaders.com](https://securityheaders.com) contra el entorno de staging

**Criterio de aceptación:** El scanner de headers reporta calificación A o superior.

---

## Backlog Item C — Protección Anti-CSRF (Nivel Gubernamental)

> **Prioridad: Alta**

- [x] **C.1** Instalar: `@dr.pogodin/csurf`
- [x] **C.2** Configurar el middleware CSRF con el patrón Double Submit Cookie
- [x] **C.3** Implementar `GET /api/csrf-token`: devolver el token CSRF para que el frontend lo cachee en memoria al montar la app
- [x] **C.4** Aplicar el middleware CSRF a todas las rutas mutables: `POST`, `PUT`, `PATCH`, `DELETE`
- [x] **C.5** El BFF debe devolver `403` a cualquier petición mutable sin `X-CSRF-Token` válido, aunque la cookie de sesión esté presente
- [ ] **C.6** Actualizar `app/services/client.ts` en el frontend para:
  - Obtener el token CSRF de `GET /api/csrf-token` al iniciar la app
  - Adjuntar `X-CSRF-Token: <token>` en cada `POST`, `PUT`, `PATCH`, `DELETE`
  - Manejar `419` (token expirado): refrescar el token y reintentar la petición una sola vez

**Criterio de aceptación:** El BFF rechaza peticiones mutables sin token CSRF, incluso con sesión activa.

---

## Backlog Item D — Gestión Documental Segura

> **Prioridad: Alta** — Requerimiento para entidades gubernamentales.

- [ ] **D.1** Implementar middleware de validación de MIME-type real en subidas de archivos:
  - Usar `file-type` (inspección de magic bytes) — no confiar en `Content-Type` del cliente
  - Rechazar tipos no permitidos con `415 Unsupported Media Type`
- [ ] **D.2** Implementar validación de tamaño máximo de archivo (configurable por variable de entorno, ej. `MAX_FILE_SIZE_MB=20`)
- [ ] **D.3** Implementar middleware de auditoría para todas las peticiones al BFF:
  - Campos: `userId`, `action` (método + ruta), `timestamp` (ISO 8601), `ip` (con soporte `X-Forwarded-For` detrás de proxy)
  - Destino inicial: log estructurado JSON a stdout — Pino ya redacta `Authorization` y `Cookie`; compatible con Loki / CloudWatch
- [ ] **D.4** Implementar sanitización de metadatos en campos de entrada antes de proxear al backend:
  - Escapar caracteres HTML en campos de texto libre
  - Rechazar patrones de script injection en campos de metadatos de documentos

**Criterio de aceptación:** Existe un log de auditoría por cada petición; archivos con MIME-type inválido son rechazados antes de llegar al backend Go.

---

## Tareas de Ajuste — Backend Go

- [ ] **BE.1** Restringir CORS del backend Go exclusivamente al origen del BFF (variable de entorno `BFF_ORIGIN`); eliminar acceso desde el navegador
- [ ] **BE.2** Implementar middleware de validación del header `X-Service-Token` en todas las rutas protegidas:
  - Rechazar con `403` si el header está ausente o no coincide con el secreto configurado
- [ ] **BE.3** Documentar el contrato interno BFF → Backend en `folium_backend/docs/bff-contract.md`:
  - Headers esperados
  - Endpoints del backend que el BFF consume
  - Formato de respuesta de login (`/api/auth/login`)

---

## Tareas de Ajuste — Frontend

- [x] **FE.1** Actualizar `useAuthStore`: sin campos JWT; guarda solo `{ userId, nombre, rol, dependencia, tenantId }`
- [ ] **FE.2** Actualizar `app/services/client.ts`:
  - Agregar `withCredentials: true` (Axios)
  - Integrar lógica de CSRF (tarea C.6)
- [ ] **FE.3** Auditar y eliminar cualquier uso de `localStorage` o `sessionStorage` para datos de autenticación
- [ ] **FE.4** Actualizar el flujo de login: tras `POST /auth/login` exitoso, guardar el perfil de usuario en `useAuthStore` (no el JWT)
- [ ] **FE.5** Actualizar `.env.example`: confirmar que `VITE_API_URL` apunta al BFF y no al backend Go directamente

---

## Criterios de Aceptación Globales

- [ ] El JWT no es visible en el Application Tab (LocalStorage/Cookies) del navegador
- [ ] Las cookies de sesión tienen los flags `HttpOnly`, `Secure` y `SameSite: Strict`
- [ ] El servidor rechaza peticiones mutables (POST, PUT, DELETE) sin un token CSRF válido
- [ ] Los archivos con MIME-type no permitido son rechazados antes de llegar al backend Go
- [ ] Existe un log de auditoría estructurado para cada petición al BFF
- [ ] El backend Go no responde a peticiones sin el header `X-Service-Token` válido

---

## Hoja de Ruta de Evolución

| Fase | Descripción | Estado |
|---|---|---|
| **Fase 1** | BFF como proxy de seguridad + SSR híbrido | ✅ Implementado |
| **Fase 2** | React Islands (hidratación parcial) — carga JS solo en módulos críticos de gestión documental | 🔜 Largo plazo |

---

## Changelog

| Versión | Fecha      | Cambio |
|---------|------------|--------|
| 1.2     | 2026-05-07 | A.1-A.2 marcadas completas; FE.1 marcada completa; nueva sección SSR+Router (completado); nueva sección CI Local; roadmap actualizado (SSR implementado) |
| 1.1     | 2026-05-07 | Alineación con stack decidido (Vite + React + TypeScript + Zustand) |
| 1.0     | 2026-04-XX | Versión inicial — backlog de implementación BFF completo |
