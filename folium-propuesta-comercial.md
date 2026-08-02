# Propuesta Comercial — Folium
## Sistema de Gestión Documental SaaS para Entidades Públicas Colombianas

**Versión:** 2.0  
**Fecha:** Mayo 2026  
**Confidencial — Solo para presentación al cliente**

---

## 1. La Oportunidad

El mercado de gestión documental en Colombia para entidades públicas tiene un problema sin resolver:

| Sistema actual | Problema |
|---|---|
| **Orfeo GPL** | Tecnología PHP de los años 2000, sin mantenimiento, riesgos de seguridad críticos |
| **Orfeo NG** | Código cerrado, no disponible para nuevos clientes |
| **Alfresco** | Requiere meses de adaptación costosa y no cumple normativa AGN de forma nativa |

**Ninguna solución SaaS colombiana** existe hoy que cumpla con la normativa del Archivo General de la Nación de forma nativa. Folium es esa solución.

---

## 2. La Solución

**Folium** es una plataforma SaaS de gestión documental construida en Go, diseñada desde el primer día para cumplir con la Ley 594/2000, el Decreto 1080/2015 y los acuerdos del AGN.

- **Tecnología moderna** — Go (el mismo lenguaje que usa Google, Cloudflare y Uber), React 19, PostgreSQL
- **Cumplimiento normativo nativo** — AGN, Ley 1581 de privacidad, firma electrónica
- **Modelo SaaS** — el cliente no instala nada, accede desde el navegador y paga una suscripción mensual
- **Multi-tenant seguro** — los datos de cada entidad están completamente aislados
- **Código propietario** — propiedad intelectual propia, sin dependencias de terceros con licencias problemáticas

---

## 3. Módulos del Sistema

### Módulos incluidos en la entrega inicial (MVP)
- Autenticación, usuarios y roles por dependencia
- Radicación de entrada, salida e interna
- Numeración automática con formato oficial colombiano
- Sticker y etiqueta de radicado imprimible (PDF)
- Enrutamiento de documentos entre dependencias
- Historial de trazabilidad completo por radicado
- Control de vencimientos y alertas automáticas
- Almacenamiento y visualización de documentos digitales
- Búsqueda por número, remitente, asunto, código de barras
- Reportes básicos por período y dependencia (Excel / PDF)
- Panel de administración del sistema

### Módulos incluidos en la segunda entrega (Cumplimiento AGN)
- Tablas de Retención Documental (TRD) completas
- Series, subseries y disposición final
- Transferencias documentales primarias y secundarias
- Actas de transferencia e Inventario Documental Único (FUID)
- Reportes normativos para el AGN
- Portal PQRSD básico (atención al ciudadano)
- Agente local para impresoras de etiquetas y escáneres

---

## 4. Fases del Proyecto

El proyecto se estructura en **cinco fases**, cada una con un entregable concreto y fecha comprometida.

---

### FASE 0 — Demo y cierre comercial
**Duración:** Semanas 1-3  
**Fecha estimada de entrega:** Junio 6, 2026

Una demostración funcional del sistema en un entorno real, desplegado en la nube con dominio real (`app.foliumhq.co`).

**Lo que se demuestra:**
- Radicación de entrada con número colombiano oficial
- Bandeja de radicados por dependencia
- Sticker de radicado imprimible
- Búsqueda por número y código de barras
- Acceso por roles (recepcionista vs. jefe de dependencia)

**Objetivo:** Firmar el contrato. El cliente ve el sistema funcionando, no una presentación.

**Compromiso de fecha:** Demo disponible el **6 de junio de 2026**.

---

### FASE 1 — Levantamiento y configuración de la entidad
**Duración:** 3 semanas  
**Período:** Junio 23 — Julio 11, 2026  
**Prerequisito:** Contrato firmado

Esta fase no es desarrollo — es la configuración del sistema con los datos reales de la entidad.

**Actividades:**
- Reunión de levantamiento de requisitos (mínimo 2 sesiones con el equipo de la entidad)
- Carga del organigrama: dependencias, secciones, cargos
- Creación de usuarios y asignación de roles
- Configuración de tipos de documento y medios de recepción
- Configuración de tiempos de respuesta por tipo de radicado
- Configuración inicial de series y subseries (esqueleto de TRD)
- Revisión del flujo de trabajo con el equipo de ventanilla
- Entorno de pruebas disponible para la entidad

**Entregable:** Entorno de pruebas configurado con los datos reales de la entidad. Usuarios creados. Lista para capacitación.

**Lo que necesitamos del cliente:**
- Organigrama actualizado de la entidad
- Lista de usuarios con cargos y dependencias
- Tipos de documento que maneja la entidad
- Tiempos de respuesta reglamentarios por tipo de radicado
- Persona de contacto técnico y funcional durante esta fase

---

### FASE 2 — Capacitación
**Duración:** 2 semanas  
**Período:** Julio 14 — julio 25, 2026

Capacitación en el entorno de pruebas, sobre el sistema ya configurado con los datos reales de la entidad.

**Sesiones incluidas:**
| Sesión | Público | Duración | Contenido |
|--------|---------|----------|-----------|
| Administradores del sistema | 2-3 personas | 4 horas | Gestión de usuarios, configuración, reportes |
| Operadores de ventanilla | Hasta 10 personas | 3 horas | Radicación, búsqueda, sticker, enrutamiento |
| Jefes de dependencia | Hasta 10 personas | 2 horas | Bandeja, asignación, trazabilidad, alertas |
| Archivo / TRD | 1-2 personas | 3 horas | TRD, transferencias (preview Fase 2) |

**Formato:** Presencial o videoconferencia. Grabaciones disponibles para el cliente.

**Entregable:** Manual de usuario en PDF para cada perfil.

---

### FASE 3 — Puesta en marcha (Go-Live)
**Duración:** 2 semanas de acompañamiento  
**Período:** Agosto 1 — agosto 15, 2026

El sistema pasa a producción con datos y usuarios reales.

**Actividades:**
- Migración de datos históricos (si aplica — radicados del año en curso)
- Activación del entorno de producción (`{entidad}.app.foliumhq.co`)
- Acompañamiento diario durante la primera semana (canal directo de soporte)
- Resolución prioritaria de issues críticos (SLA: 4 horas en horario hábil)
- Ajustes de configuración según retroalimentación real

**Criterio de éxito:** La entidad radica documentos reales en producción sin soporte activo.

**Compromiso de fecha:** Sistema en producción a más tardar el **15 de agosto de 2026**.

---

### FASE 4 — Estabilización y entrega del MVP completo
**Duración:** Meses 3-4 post go-live  
**Período:** Agosto 16 — Noviembre 30, 2026

Durante esta fase se completan los módulos restantes del MVP y se estabiliza el sistema en producción.

**Entregables adicionales:**
- Radicación de salida vinculada al de entrada
- Radicación interna entre dependencias
- Reasignación y devolución de radicados con observaciones
- Reportes avanzados (tiempos de respuesta, vencidos, por período)
- Exportación a Excel y PDF
- Alertas automáticas por email (próximos a vencer, vencidos)
- Backups automáticos configurados

**Compromisos de entrega parciales:**
| Entregable | Fecha límite |
|---|---|
| Radicación de salida + interna | Septiembre 30, 2026 |
| Reasignación, devolución, trazabilidad completa | Octubre 31, 2026 |
| Reportes, exportación, alertas email | Noviembre 30, 2026 |

---

### FASE 5 — Cumplimiento AGN completo
**Duración:** 6 meses  
**Período:** Enero — Junio 2027

Esta fase entrega el cumplimiento normativo completo exigido por el Archivo General de la Nación.

**Entregables:**
- TRD completa con editor en el panel de administración
- Clasificación TRD al momento de radicar
- Transferencias documentales primarias (gestión → central)
- Transferencias secundarias (central → histórico)
- Actas de transferencia
- Inventario Documental Único (FUID) exportable
- Reportes normativos para el AGN
- Portal PQRSD básico (radicación web por el ciudadano)
- Agente local (impresoras de etiquetas Zebra, escáner por carpeta)

**Compromiso de fecha:** Módulos AGN completos y en producción a más tardar el **30 de junio de 2027**.

---

## 5. Cronograma Consolidado

```
2026
─────────────────────────────────────────────────────────────────────
Jun 06   │  Demo funcional disponible para el cliente
Jun 23   │  ► INICIO (firma del contrato)
Jun 23   │  Fase 1 — Levantamiento y configuración
Jul 14   │  Fase 2 — Capacitación
Ago 01   │  Fase 3 — Go-Live en producción
Ago 15   │  ● HITO: Sistema en producción con usuarios reales
Ago 16   │  Fase 4 — Estabilización + MVP completo
Sep 30   │  ● Entrega parcial: radicación de salida + interna
Oct 31   │  ● Entrega parcial: reasignación, trazabilidad completa
Nov 30   │  ● HITO: MVP completo en producción
─────────────────────────────────────────────────────────────────────
2027
─────────────────────────────────────────────────────────────────────
Ene 01   │  Fase 5 — Cumplimiento AGN
Jun 30   │  ● HITO: AGN completo (TRD, transferencias, FUID, PQRSD)
─────────────────────────────────────────────────────────────────────
```

**Hitos con fecha comprometida:**
| # | Hito | Fecha límite |
|---|------|-------------|
| 1 | Sistema en producción (go-live) | **15 agosto 2026** |
| 2 | MVP completo | **30 noviembre 2026** |
| 3 | Cumplimiento AGN completo | **30 junio 2027** |

---

## 6. Inversión

### 6.1 Tarifa de la suscripción mensual

| Plan | Perfil | Usuarios | Almacenamiento | Precio/mes |
|---|---|---|---|---|
| **Básico** | Entidad pequeña | Hasta 20 | 10 GB | $800.000 — $1.200.000 COP |
| **Profesional** | Municipio, secretaría | Hasta 100 | 50 GB | $2.000.000 — $3.500.000 COP |
| **Institucional** | Gobernación, entidad grande | Ilimitados | 200 GB | $5.000.000 — $8.000.000 COP |

La suscripción incluye: actualizaciones del sistema, correcciones de seguridad, soporte técnico, backups automáticos y acceso a todas las funcionalidades del plan.

---

### 6.2 Fee de implementación y configuración (pago único)

El fee cubre las Fases 1, 2 y 3: levantamiento, configuración, capacitación y acompañamiento en el go-live.

| Complejidad | Perfil | Fee |
|---|---|---|
| Básica | Hasta 50 usuarios, estructura organizacional sencilla | $8.000.000 COP |
| Estándar | Hasta 150 usuarios, múltiples dependencias | $15.000.000 COP |
| Compleja | Más de 150 usuarios, migración de datos históricos | $25.000.000 — $40.000.000 COP |

---

### 6.3 Condiciones del cliente piloto

El primer cliente tiene condiciones preferenciales a cambio de ser co-constructor del producto:

| Concepto | Valor piloto | Valor comercial (año 2+) |
|---|---|---|
| Fee de implementación | 50% de descuento | Precio estándar |
| Suscripción mensual año 1 | 40% de descuento | Precio de lista |
| Influencia sobre el roadmap | Alta (requisitos directos) | Estándar |
| Soporte durante go-live | Acompañamiento diario semana 1 | SLA estándar |

**A cambio, el cliente piloto:**
- Acepta trabajar con una versión en maduración activa
- Proporciona retroalimentación constante durante las primeras 12 semanas
- Autoriza ser referenciado como caso de éxito (con su aprobación por comunicado)

---

### 6.4 Costo total estimado primer año (cliente piloto)

Ejemplo para plan Profesional, complejidad estándar, con descuentos piloto:

| Concepto | Valor |
|---|---|
| Fee de implementación (descuento 50%) | $7.500.000 COP |
| Suscripción mensual piloto (12 meses × $1.500.000) | $18.000.000 COP |
| **Total primer año** | **$25.500.000 COP** |

A partir del segundo año: suscripción a precio de lista (~$2.500.000/mes).

---

## 7. Estimación del Costo de Desarrollo

*Esta sección responde cuánto cuesta construir Folium desde cero. Referencia para evaluar la propuesta en perspectiva.*

### Alcance del desarrollo

| Fase | Duración | Horas estimadas | Descripción |
|---|---|---|---|
| POC | 4 semanas | 160 horas | Demo funcional, autenticación, radicación básica, sticker |
| MVP | 5 meses | 800 horas | Sistema completo para primer cliente en producción |
| Cumplimiento AGN | 6 meses | 840 horas | TRD, transferencias, FUID, PQRSD |
| **Total fases 0-2** | **~12 meses** | **~1.800 horas** | |

### Tarifa de referencia (desarrollador senior full-stack, Colombia 2026)

Un desarrollador senior con perfil Go + React + infraestructura SaaS factura entre **$80.000 y $110.000 COP por hora** en el mercado colombiano actual.

| Escenario | Tarifa/hora | Total fases 0-2 |
|---|---|---|
| Conservador | $80.000 COP | $144.000.000 COP |
| Estándar | $95.000 COP | $171.000.000 COP |
| Premium | $110.000 COP | $198.000.000 COP |

### Equivalente mensual

| Modalidad | Costo mensual aprox. |
|---|---|
| Desarrollador full-time (empleado) | $12.000.000 — $18.000.000 COP |
| Contratista senior (tiempo completo) | $15.000.000 — $22.000.000 COP |

### Conclusión

Un proyecto de esta envergadura — SaaS multi-tenant, stack moderno, cumplimiento normativo colombiano completo — representa una **inversión de desarrollo de $144M a $200M COP** para las primeras dos fases (12 meses).

El modelo SaaS recupera esa inversión mediante suscripciones recurrentes. Con 10 clientes en plan Profesional a $2.500.000/mes, el ingreso mensual es **$25.000.000 COP**, lo que amortiza el desarrollo en menos de 8 meses de escala.

---

## 8. Normativa Cumplida

| Norma | Descripción | Módulo que lo cumple |
|---|---|---|
| Ley 594 de 2000 | Ley General de Archivos | Todo el sistema |
| Decreto 1080 de 2015 | Marco reglamentario de archivos | TRD, transferencias |
| Acuerdo 003 AGN 2015 | Sistemas de gestión de documentos | Radicación, almacenamiento |
| Acuerdo 006 AGN 2014 | Digitalización y documentos electrónicos | Escaneo, metadatos |
| Ley 1581 de 2012 | Protección de datos personales | Consentimiento, anonimización |
| Decreto 2364 de 2012 | Firma electrónica | Integración Certicámara (Fase 5) |

---

## 9. Ventajas Competitivas

| Criterio | Orfeo GPL | Alfresco | **Folium** |
|---|---|---|---|
| Cumplimiento AGN nativo | ✅ | ⚠️ Adaptación | ✅ |
| Mantenimiento activo | ❌ | ✅ | ✅ |
| Soporte con SLA | ❌ | Solo Enterprise | ✅ Incluido |
| Modelo SaaS | ❌ | ⚠️ Parcial | ✅ |
| Precio accesible | ✅ (gratuito) | ❌ Caro | ✅ |
| Implementación rápida | ❌ Semanas | ❌ Meses | ✅ Días |
| Soporte local colombiano | ⚠️ Comunidad | ❌ | ✅ Directo |
| Tecnología moderna | ❌ PHP | ✅ Java | ✅ Go |

---

## 10. Garantías y SLA

| Concepto | Compromiso |
|---|---|
| Disponibilidad del sistema | 99,5% mensual en horario hábil |
| Tiempo de respuesta soporte crítico (producción caída) | 2 horas hábiles |
| Tiempo de respuesta soporte estándar | 24 horas hábiles |
| Correcciones de seguridad | 48 horas tras detección confirmada |
| Actualizaciones del sistema | Incluidas en suscripción, sin costo adicional |
| Backups diarios de datos | Incluidos, retención 30 días |
| Notificación previa a mantenimientos programados | 48 horas |

---

## 11. Lo que el Cliente No Necesita Contratar

Folium es SaaS. El cliente **no necesita**:
- Comprar servidores ni infraestructura
- Contratar hosting o nube propios
- Pagar licencias de base de datos
- Mantener un área TI para el sistema
- Gestionar actualizaciones o parches de seguridad
- Configurar backups

---

## 12. Siguiente Paso

Para iniciar el proceso:

1. **Agendar demo** — el cliente ve el sistema funcionando, no una presentación
2. **Reunión de levantamiento** — 2 horas para entender el contexto específico de la entidad
3. **Ajustar propuesta** — confirmar plan, módulos y condiciones del piloto
4. **Firma del contrato** — inicio oficial del proyecto

**Contacto:**  
Cristián de la Hoz  
cristhoz@gmail.com  
foliumhq.co

---

*Documento confidencial — Propuesta comercial Folium. Versión 2.0 — Mayo 2026.*

---

## Changelog

| Versión | Fecha | Cambio |
|---|---|---|
| 2.0 | 2026-05-14 | Reescritura completa: fases con fechas comprometidas, onboarding, pricing detallado, estimación de desarrollo |
| 1.0 | 2026-04-XX | Versión inicial — propuesta comercial y técnica |
