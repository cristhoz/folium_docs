# Tareas de Implementación — Capa BFF Segura

Documento de trabajo derivado de `BFF-arquitectura-front.md`. Cubre todas las tareas necesarias para implementar la capa BFF en Folium bajo estándares MinTIC/OWASP.

---

## Fase 0 — Configuración Base del Servidor BFF

| # | Tarea | Prioridad |
|---|-------|-----------|
| 0.1 | Inicializar proyecto Node.js + TypeScript en `folium_bff/` (o como workspace dentro del monorepo frontend) | Crítica |
| 0.2 | Instalar dependencias base: `express`, `typescript`, `ts-node`, `@types/express` | Crítica |
| 0.3 | Integrar Vite como middleware de Express para desarrollo (`vite.createServer` + `server.middlewares`) | Crítica |
| 0.4 | Configurar modo producción: Express sirve el build estático de Vite (`dist/`) | Alta |
| 0.5 | Añadir servicio `folium_bff` y `redis` al `docker-compose.yml` del proyecto | Crítica |
| 0.6 | Definir variables de entorno del BFF: `REDIS_URL`, `SESSION_SECRET`, `X_SERVICE_TOKEN`, `BACKEND_URL`, `PORT` | Crítica |

---

## Backlog Item A — Gestión de Sesiones Token-per-Session

> **Prioridad: Crítica** — Requisito para que el JWT nunca sea expuesto al navegador.

- [ ] **A.1** Instalar: `express-session`, `connect-redis`, `ioredis`
- [ ] **A.2** Configurar Redis como store de sesiones en Express (`connect-redis` + `express-session`)
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
  - Leer `SessionID` de la cookie
  - Resolver el JWT en Redis
  - Adjuntar `Authorization: Bearer <token>` + `X-Service-Token` a la petición hacia el backend Go
  - Devolver `401` si la sesión no existe o expiró
- [ ] **A.6** Implementar `GET /auth/me`: devolver el perfil del usuario de la sesión activa (sin JWT)

**Criterio de aceptación:** El JWT no es visible en el Application Tab (LocalStorage / Cookies) del navegador en ningún momento.

---

## Backlog Item B — Blindaje de Headers (MinTIC / OWASP)

> **Prioridad: Alta**

- [ ] **B.1** Instalar: `helmet`
- [ ] **B.2** Configurar `helmet()` con Content Security Policy (CSP) estricta:
  - `default-src 'self'`
  - `script-src 'self'` (bloquear scripts externos e inline)
  - `style-src 'self' 'unsafe-inline'` (solo si necesario para la UI)
  - `img-src 'self' data:`
  - `connect-src 'self'` (solo al mismo origen BFF)
  - `frame-ancestors 'none'` (anti-clickjacking)
- [ ] **B.3** Configurar `Referrer-Policy: no-referrer`
- [ ] **B.4** Configurar HSTS: `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- [ ] **B.5** Verificar headers con [securityheaders.com](https://securityheaders.com) contra el entorno de staging

**Criterio de aceptación:** El scanner de headers reporta calificación A o superior.

---

## Backlog Item C — Protección Anti-CSRF (Nivel Gubernamental)

> **Prioridad: Alta**

- [ ] **C.1** Instalar: `@dr.pogodin/csurf`
- [ ] **C.2** Configurar el middleware CSRF con el patrón Double Submit Cookie
- [ ] **C.3** Implementar `GET /csrf-token`: devolver el token CSRF para que el frontend lo cachee en memoria al montar la app
- [ ] **C.4** Aplicar el middleware CSRF a todas las rutas mutables: `POST`, `PUT`, `PATCH`, `DELETE`
- [ ] **C.5** El BFF debe devolver `403` (o `419`) a cualquier petición mutable sin `X-CSRF-Token` válido, aunque la cookie de sesión esté presente
- [ ] **C.6** Actualizar `app/api/client.ts` en el frontend para:
  - Obtener el token CSRF de `GET /csrf-token` al iniciar la app
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
  - Destino inicial: log estructurado JSON a stdout (compatible con agregadores tipo Loki / CloudWatch)
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

- [ ] **FE.1** Actualizar `useAuthStore`: eliminar cualquier campo relacionado con JWT; guardar solo `{ userId, nombre, rol, dependencia, tenantId }`
- [ ] **FE.2** Actualizar `app/api/client.ts`:
  - URL base: `VITE_BFF_URL` (eliminar `VITE_API_URL`)
  - Agregar `credentials: 'include'` (Axios: `withCredentials: true`)
  - Integrar lógica de CSRF (tarea C.6)
- [ ] **FE.3** Auditar y eliminar cualquier uso de `localStorage` o `sessionStorage` para datos de autenticación
- [ ] **FE.4** Actualizar el flujo de login: tras `POST /auth/login` exitoso, guardar el perfil de usuario en `useAuthStore` (no el JWT)
- [ ] **FE.5** Actualizar `.env.example`: reemplazar `VITE_API_URL` por `VITE_BFF_URL=http://localhost:3000`

---

## Hoja de Ruta de Evolución (Para Futuras Fases)

| Fase | Descripción | Prerrequisito |
|------|-------------|---------------|
| **Fase 1 (Actual)** | BFF como proxy de seguridad + entrega de SPA | Backlog A, B, C completos |
| **Fase 2 (Medio Plazo)** | SSR con el servidor Express del BFF — mejora SEO y tiempo de carga inicial | Fase 1 completada |
| **Fase 3 (Largo Plazo)** | React Islands (hidratación parcial) — carga JS solo en módulos críticos de gestión documental | Fase 2 completada |

---

## Criterios de Aceptación Globales

- [ ] El JWT no es visible en el Application Tab (LocalStorage/Cookies) del navegador
- [ ] Las cookies de sesión tienen los flags `HttpOnly`, `Secure` y `SameSite: Strict`
- [ ] El servidor rechaza peticiones mutables (POST, PUT, DELETE) sin un token CSRF válido
- [ ] Los archivos con MIME-type no permitido son rechazados antes de llegar al backend Go
- [ ] Existe un log de auditoría estructurado para cada petición al BFF
- [ ] El backend Go no responde a peticiones sin el header `X-Service-Token` válido
