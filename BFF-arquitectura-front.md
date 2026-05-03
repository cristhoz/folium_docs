Este documento detalla la arquitectura de seguridad y el plan de implementación de la capa **BFF (Backend-for-Frontend)** para Folium. El objetivo es blindar la aplicación bajo estándares gubernamentales y preparar el terreno para una futura migración a **SSR** y **React Islands**.

# Iniciativa Arquitectónica: Implementación de Capa BFF Segura

## 1. Resumen del Objetivo

Transformar la arquitectura actual de la SPA (Vite + React) hacia un modelo de **Servidor Personalizado (Custom Server)** que actúe como un BFF. Esta capa centralizará la lógica de autenticación, la gestión de sesiones sensibles y la protección contra ataques dirigidos, asegurando que los tokens de acceso (JWT) nunca sean expuestos al navegador.

## 2. Requisitos Técnicos de Seguridad (Backlog Items)

### A. Gestión de Sesiones "Token-per-Session"

 * **Implementación:** Configurar un servidor Express que envuelva a Vite.
 * **Almacenamiento:** Utilizar **Redis** para persistir las sesiones en el servidor.
 * **Mecanismo:**
   * El BFF recibe el JWT del Backend real.
   * El JWT se guarda en Redis vinculado a un SessionID.
   * El navegador solo recibe una cookie HttpOnly, Secure y SameSite: Strict con el ID de sesión.
 * **Prioridad:** Crítica.

### B. Blindaje de Headers (Estándar MinTIC/OWASP)

 * **Herramienta:** **Helmet.js**.
 * **Acción:** Configurar una **Content Security Policy (CSP)** estricta que:
   * Bloquee la ejecución de scripts no autorizados (Anti-XSS).
   * Deshabilite el rastreo de Referrer-Policy.
   * Implemente HSTS (Strict-Transport-Security) para forzar HTTPS.
 * **Prioridad:** Alta.

### C. Protección Anti-CSRF (Nivel Gubernamental)

 * **Herramienta:** @dr.pogodin/csurf (Double Submit Cookie).
 * **Mecanismo:** Implementar el patrón de doble envío de token para todas las operaciones mutables (POST, PUT, DELETE).
 * **Validación:** El BFF rechazará cualquier petición que no incluya el header X-CSRF-Token válido, incluso si la cookie de sesión está presente.
 * **Prioridad:** Alta.

## 3. Especificaciones para Gestión Documental

Para cumplir con los requisitos de entidades del gobierno, el BFF deberá incluir:
 1. **Validación de Integridad de Archivos:** El BFF debe interceptar las subidas de documentos para validar el **MIME-type** real y el tamaño antes de enviarlos al almacenamiento.
 2. **Trazabilidad y Auditoría:** Implementar un middleware que registre:
   * ID de usuario.
   * Acción realizada.
   * Timestamp y dirección IP origen (detrás de proxy si aplica).
 3. **Sanitización de Metadatos:** Limpiar cualquier rastro de scripts maliciosos en los campos de entrada de datos antes de procesar el documento.
 
## 4. Hoja de Ruta de Evolución (Iteraciones Futuras)

> [!IMPORTANT]
> Esta arquitectura de BFF con servidor Node.js es el requisito previo para las siguientes fases del proyecto:
> 

 * **Fase 1 (Actual):** BFF como Proxy de Seguridad y entrega de SPA.
 * **Fase 2 (Medio Plazo):** Implementación de **SSR (Server-Side Rendering)** usando el servidor del BFF para mejorar el SEO y la velocidad de carga inicial.
 * **Fase 3 (Largo Plazo):** Adopción de **React Islands** (hidratación parcial) para optimizar el rendimiento en módulos críticos de la gestión documental, cargando JS solo donde sea necesario.
 
## 5. Stack Tecnológico
 * **Frontend:** React + Vite + TypeScript.
 * **BFF Server:** Express.js.
 * **Seguridad:** Helmet, @dr.pogodin/csurf.
 * **Persistencia de Sesión:** Redis + connect-redis.
 * **Comunicación:** Axios (BFF a Backend real).
 
**Criterios de Aceptación:**
 * [ ] El JWT no es visible en el Application Tab (LocalStorage/Cookies) del navegador.
 * [ ] Las cookies de sesión tienen los flags HttpOnly, secure y SameSite: Strict.
 * [ ] El servidor rechaza peticiones mutables (POST, PUT, DELETE) sin un token CSRF válido.
 * [ ] Existe un log de auditoría básico para cada petición al BFF.
