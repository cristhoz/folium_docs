**Versión:** 1.2 | **Fecha:** Mayo 2026 | **Proyecto:** Folium — foliumhq.co | **Uso interno**

Este documento describe la arquitectura de seguridad y renderizado de la capa **BFF (Backend-for-Frontend)** de Folium, tal como está implementada en el repositorio actual.

# Arquitectura Frontend — Capa BFF Segura + SSR Híbrido

## 1. Resumen

La arquitectura frontend de Folium implementa un **servidor personalizado (BFF)** basado en Express 5 que cumple tres roles simultáneos:

1. **Proxy de seguridad**: centraliza autenticación, sesiones y protección CSRF; el JWT nunca llega al navegador.
2. **Servidor SSR**: renderiza cada ruta en el servidor antes de enviarla al navegador, mejorando el tiempo de carga inicial.
3. **Servidor de activos**: sirve los bundles del cliente (en producción) o integra Vite Dev Server con HMR (en desarrollo).

El navegador **nunca se comunica directamente con el backend Go**. Todo el tráfico pasa por el BFF:

```
Navegador → BFF (Express 5) → Backend Go (Chi)
```

---

## 2. Stack Tecnológico

| Capa | Tecnología |
|---|---|
| Bundler / Dev server | Vite 6 |
| Lenguaje | TypeScript 5.8 |
| Framework UI | React 19 |
| Router | TanStack Router v1 (file-based routing) |
| Estado global | Zustand 5 |
| BFF Server | Express 5 |
| Seguridad BFF | Helmet 8, @dr.pogodin/csurf, cookie-parser |
| Sesiones | express-session + connect-redis + ioredis 5 |
| HTTP Client | Axios 1 |
| Logging (BFF) | Pino 10 + pino-http |
| Configuración | YAML por entorno (`config-development.yaml`, `config-production.yaml`) |
| Testing | Jest 30 + ts-jest + Supertest |
| Linting | ESLint 9 (flat config) + Prettier + Stylelint |
| CSS | SCSS + Stylelint (recess-order) |
| Git hooks | Lefthook + Commitlint (Conventional Commits) |
| Infraestructura local | Docker Compose (Redis 7 Alpine) |
| Runtime scripts | tsx (watch) + esbuild (bundle servidor) |

---

## 3. Renderizado Híbrido SSR + CSR

El BFF implementa SSR per-ruta usando el propio servidor Express:

- **`app/entry-server.tsx`** — punto de entrada SSR: usa `renderToString` de React y TanStack Router con `createMemoryHistory`. Genera el HTML de cada ruta en el servidor.
- **`app/entry-client.tsx`** — punto de entrada cliente: hidrata el HTML recibido del servidor.
- El BFF inyecta el HTML renderizado y el estado inicial (`window.__INITIAL_STATE__`) en la plantilla HTML antes de enviarla al navegador.

### Modo desarrollo

El middleware **`api/middlewares/vite-dev-resolve.ts`** integra Vite Dev Server. Carga el módulo SSR vía `viteServer.ssrLoadModule` y mantiene HMR activo durante el desarrollo.

### Modo producción

El middleware **`api/middlewares/vite-prod-resolve.ts`** sirve los bundles pre-compilados desde `dist/client` (activos del navegador) y `dist/server` (módulo SSR). El bundle del servidor se genera con `scripts/build-server.mjs` (esbuild).

---

## 4. Routing (TanStack Router — file-based)

Las rutas se definen como archivos dentro de `app/pages/`. El plugin `@tanstack/router-vite-plugin` genera automáticamente el árbol de rutas en `app/routeTree.gen.ts`.

**Rutas actuales:**

| Archivo | Función |
|---|---|
| `app/pages/__root.tsx` | Layout raíz (aplica a todas las rutas) |
| `app/pages/_auth.tsx` | Layout para rutas protegidas (requiere sesión activa) |
| `app/pages/index.tsx` | Pantalla principal / bandeja |
| `app/pages/login.tsx` | Pantalla de autenticación |

---

## 5. Seguridad (BFF)

### 5.1 CORS

Solo el origin configurado (`allowedOrigin`) puede hacer requests con credenciales. Peticiones desde otros orígenes son rechazadas.

### 5.2 CSRF (Double Submit Cookie)

- Herramienta: `@dr.pogodin/csurf`.
- El cliente obtiene el token en `GET /api/csrf-token` al montar la aplicación.
- El token se envía en el header `X-CSRF-Token` en todas las mutaciones (`POST`, `PUT`, `PATCH`, `DELETE`).
- El BFF rechaza mutaciones sin token CSRF válido con `403`.
- El token CSRF viaja en una cookie firmada `_csrf`.

### 5.3 Sesión

- Almacenada en Redis con TTL configurable.
- El navegador recibe solo la cookie `sid` (`HttpOnly; Secure; SameSite=Strict`), firmada con `COOKIE_SECRET`.
- En cada request autenticado, el BFF resuelve la sesión en Redis, extrae el JWT y lo adjunta como `Authorization: Bearer` en la petición interna hacia Go.
- **El JWT nunca llega al navegador** ni existe en el estado de React/Zustand ni en `localStorage`.

### 5.4 Headers (Helmet)

Helmet 8 activo solo en producción. Configura: CSP estricta (`default-src 'self'`), anti-clickjacking (`frame-ancestors 'none'`), HSTS (`max-age=31536000; includeSubDomains`), `Referrer-Policy: no-referrer`.

### 5.5 Logging (Pino)

Pino 10 + pino-http registran cada petición en formato JSON. `Authorization` y `Cookie` son **redactados automáticamente** de los logs.

---

## 6. Configuración

El servidor lee configuración desde archivos YAML según la variable de entorno `ENVIRONMENT`:
- `config-development.yaml` — entorno local
- `config-production.yaml` — entorno de producción

Los valores que necesitan estar disponibles en el bundle del navegador usan el prefijo `VITE_` (Vite los inyecta en tiempo de compilación). Ejemplo: `VITE_API_URL` apunta al BFF, nunca directamente al backend Go.

---

## 7. Estructura de Carpetas

```
app/
├── pages/              # rutas (file-based, TanStack Router)
│   ├── __root.tsx
│   ├── _auth.tsx
│   ├── index.tsx
│   └── login.tsx
├── stores/             # stores Zustand por dominio
│   ├── useAuthStore.ts
│   └── useRadicadoStore.ts
├── services/           # clientes HTTP; client.ts apunta al BFF
│   ├── client.ts       # Axios + interceptores 401/403/5xx
│   ├── auth.ts
│   └── radicados.ts
├── types/              # tipos TS espejo de los modelos Go del backend
│   ├── AuthUser.ts
│   └── Radicado.ts
├── routeTree.gen.ts    # generado automáticamente por @tanstack/router-vite-plugin
├── createRouter.ts
├── entry-client.tsx    # hidratación del cliente
└── entry-server.tsx    # renderizado en servidor (SSR)

api/                    # capa BFF (Express 5)
├── clients/
│   └── redis.ts        # cliente ioredis
├── middlewares/
│   ├── security.ts     # Helmet, CORS, CSRF, cookie-parser
│   ├── session.ts      # express-session + connect-redis
│   ├── ssr-handler.ts  # orquesta SSR (entry-server + inyección HTML)
│   ├── vite-dev-resolve.ts   # desarrollo: Vite HMR + ssrLoadModule
│   └── vite-prod-resolve.ts  # producción: bundles desde dist/
├── routes/
│   ├── csrf.route.ts
│   └── health.route.ts
├── utils/
│   └── logger.ts       # instancia Pino
└── server.ts           # punto de entrada del BFF

config/                 # carga YAML por entorno + tipos
scripts/
└── build-server.mjs    # esbuild: bundle del servidor para dist-server/
```

---

## 8. Scripts Principales

| Comando | Descripción |
|---|---|
| `npm run dev` | BFF + Vite Dev Server en modo watch (tsx) |
| `npm run build` | Compila TS + bundle cliente + bundle SSR + bundle servidor |
| `npm run start` | Arranca el servidor de producción (`dist-server/server.js`) |
| `npm run test` | Jest (unit + integration con Supertest) |
| `npm run lint:all` | ESLint + Stylelint |

---

## 9. Criterios de Aceptación de Seguridad

- [ ] El JWT no es visible en el Application Tab (LocalStorage/Cookies) del navegador.
- [ ] Las cookies de sesión tienen los flags `HttpOnly`, `Secure` y `SameSite: Strict`.
- [ ] El servidor rechaza peticiones mutables (POST, PUT, DELETE) sin un token CSRF válido.
- [ ] Existe un log de auditoría básico para cada petición al BFF.

---

## 10. Hoja de Ruta de Evolución

| Fase | Descripción | Estado |
|---|---|---|
| **Fase 1** | BFF como proxy de seguridad + SSR híbrido | ✅ Implementado |
| **Fase 2** | React Islands (hidratación parcial) — carga JS solo en módulos críticos de gestión documental | 🔜 Largo plazo |

---

## Changelog

| Versión | Fecha      | Cambio |
|---------|------------|--------|
| 1.2     | 2026-05-07 | Reescritura mayor: stack con versiones exactas, SSR híbrido implementado, TanStack Router, estructura de carpetas, scripts, seguridad detallada, Pino, configuración YAML; roadmap actualizado |
| 1.1     | 2026-05-07 | Zustand añadido como gestor de estado; alineación con decisión de stack |
| 1.0     | 2026-04-XX | Versión inicial — arquitectura BFF con React + Vite + TypeScript |
