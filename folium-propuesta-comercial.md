# Propuesta: Sistema de Gestión Documental SaaS
## Plataforma propia para entidades públicas colombianas

**Fecha:** Abril 2026  
**Confidencial — Solo para uso interno y presentación al cliente**

---

## 1. La Oportunidad

El mercado de gestión documental en Colombia para entidades públicas tiene un problema sin resolver:

- **Orfeo** (el estándar actual) está basado en tecnología obsoleta de los años 2000, sin mantenimiento activo y con riesgos de seguridad significativos
- **Orfeo NG** (la versión moderna) es código cerrado, no disponible libremente
- **Alfresco** requiere meses de adaptación costosa para cumplir la normativa colombiana
- **Ninguna solución SaaS colombiana** existe hoy que cumpla con la normativa del AGN de forma nativa

Esto representa una ventana de mercado clara para una plataforma nueva, moderna, segura y 100% adaptada a la realidad colombiana.

---

## 2. La Solución Propuesta

Desarrollar desde cero una plataforma de gestión documental:

- **Tecnología moderna** — construida en Go, una de las tecnologías más eficientes y seguras del mercado actual
- **Cumplimiento normativo nativo** — diseñada desde el inicio para cumplir con la Ley 594/2000, Decreto 1080/2015 y acuerdos del AGN
- **Modelo SaaS** — el cliente no instala nada, accede desde el navegador y paga una suscripción mensual
- **Código propietario** — propiedad intelectual 100% del desarrollador, no depende de terceros

---

## 3. ¿Por qué Go como tecnología base?

| Ventaja | Impacto práctico |
|---|---|
| Alto rendimiento | Maneja grandes volúmenes de documentos sin degradarse |
| Binario único | Despliegue simple, sin dependencias externas |
| Seguridad | Más robusto que PHP, menos superficie de ataque |
| Bajo consumo de recursos | Infraestructura más económica = mejor margen |
| Ideal para APIs | Base perfecta para modelo SaaS multi-cliente |
| Adoptado por Google, Cloudflare, Uber | Tecnología probada a escala global |

---

## 4. Arquitectura de la Plataforma

```
┌─────────────────────────────────────────┐
│              FRONTEND WEB               │
│      Interfaz moderna y responsiva      │
└──────────────────┬──────────────────────┘
                   │ API REST segura
┌──────────────────▼──────────────────────┐
│           BACKEND EN GO                  │
│  ┌─────────────┐  ┌─────────────────┐   │
│  │  API REST   │  │  Procesamiento  │   │
│  │  (núcleo)   │  │  en background  │   │
│  └─────────────┘  └─────────────────┘   │
└──────┬──────────────────┬───────────────┘
       │                  │
┌──────▼──────┐    ┌──────▼──────┐
│ PostgreSQL  │    │  Almacen    │
│ (datos)     │    │  Documentos │
└─────────────┘    └─────────────┘
```

**Multi-tenant:** cada entidad cliente tiene sus datos completamente aislados. Un cliente nunca ve los datos de otro.

---

## 5. Módulos del Sistema

### Fase 1 — MVP (Producto mínimo vendible)
- Autenticación, usuarios y roles por dependencia
- Radicación de correspondencia (entrada, salida, interna)
- Numeración automática con formato oficial colombiano
- Sticker y etiqueta de radicado imprimible
- Enrutamiento de documentos entre dependencias
- Control de vencimientos y alertas automáticas
- Almacenamiento y visualización de documentos digitales
- Búsqueda de radicados y documentos

### Fase 2 — Cumplimiento AGN completo
- Tablas de Retención Documental (TRD)
- Series y subseries documentales
- Transferencias documentales primarias y secundarias
- Reportes normativos para el AGN
- Inventario Documental Único (FUID)
- Histórico y trazabilidad completa

### Fase 3 — SaaS avanzado y diferenciadores
- Portal de atención al ciudadano (PQRSD)
- Integración con firma electrónica (Certicámara)
- Dashboard de indicadores y analítica
- Aplicación móvil básica
- API pública para integración con otros sistemas
- Facturación y gestión de suscripciones automatizada

---

## 6. Cronograma de Desarrollo

```
Mes 1-2    →  Arquitectura base, autenticación, estructura de datos
Mes 3-5    →  Radicación completa, gestión documental core
Mes 6      →  MVP funcional — primer demo al cliente piloto
Mes 7-9    →  TRD, reportes AGN, ajustes de UX
Mes 10     →  Primer cliente en producción (piloto pagado)
Mes 11-14  →  Multi-tenant, infraestructura SaaS, facturación
Mes 15-18  →  Plataforma SaaS estable y escalable comercialmente
```

**Hito clave:** A los 6 meses hay un producto demostrable y funcional para iniciar el proceso comercial con el primer cliente.

---

## 7. Modelo de Negocio SaaS

### Planes propuestos

| Plan | Perfil del cliente | Precio estimado / mes |
|---|---|---|
| **Básico** | Hasta 20 usuarios, 1 sede | $800.000 — $1.200.000 COP |
| **Profesional** | Hasta 100 usuarios, TRD completa | $2.000.000 — $3.500.000 COP |
| **Institucional** | Usuarios ilimitados, soporte prioritario | $5.000.000 — $8.000.000 COP |
| **On-premise** | Instalación en servidores del cliente | Licencia anual negociable |

### ¿Por qué SaaS es mejor para el cliente?
- No requiere comprar ni mantener infraestructura propia
- Actualizaciones automáticas — siempre en cumplimiento normativo
- Puede contratarse como servicio bajo modalidades de menor cuantía
- Soporte incluido en la suscripción
- Acceso desde cualquier lugar con internet

### Proyección de ingresos (conservadora)

| Clientes activos | Plan promedio | Ingreso mensual |
|---|---|---|
| 5 clientes | Profesional | ~$12.500.000 COP |
| 15 clientes | Profesional | ~$37.500.000 COP |
| 30 clientes | Mixto | ~$65.000.000 COP |

---

## 8. Ventajas Competitivas

### vs. Orfeo original
- Tecnología moderna y segura
- Mantenimiento activo y garantizado
- Soporte oficial con SLA
- Modelo SaaS — sin instalaciones complejas

### vs. Orfeo NG
- Disponible para nuevos clientes
- Precios competitivos
- Soporte local colombiano
- Evolución continua del producto

### vs. Alfresco
- Cumplimiento normativo colombiano desde el día 1
- Sin costos de licencia de terceros
- Implementación en días, no meses
- Precio accesible para entidades medianas y pequeñas

---

## 9. Propuesta para el Cliente Piloto

El primer cliente tiene una posición estratégica única:

- **Precio preferencial** durante los primeros 12 meses
- **Influencia directa** sobre las funcionalidades prioritarias
- **Soporte dedicado** durante la implementación
- **Co-construcción** — sus necesidades reales dan forma al producto

A cambio, el cliente piloto:
- Acepta trabajar con una versión en maduración
- Proporciona retroalimentación activa
- Puede convertirse en caso de éxito referenciable

---

## 10. Inversión y Siguiente Paso

### Lo que se requiere para arrancar
- Definir alcance del MVP con el cliente piloto
- Establecer contrato de desarrollo + suscripción piloto
- Iniciar desarrollo — primeros entregables visibles en 60 días

### Siguiente paso inmediato
Agendar reunión técnica-comercial para:
1. Levantar requisitos específicos del cliente
2. Confirmar módulos prioritarios para el MVP
3. Definir condiciones del piloto
4. Firmar acuerdo de inicio

---

## 11. Conclusión

El mercado existe, el problema es real y las soluciones actuales son insuficientes. Una plataforma moderna, segura y construida específicamente para la realidad colombiana tiene un espacio claro en el mercado.

La pregunta no es si el producto tendrá demanda — la pregunta es quién lo construye primero.

Esta es la oportunidad de ser ese primero.

---

*Documento confidencial — Propuesta comercial y técnica. Versión 1.0 — Abril 2026.*
