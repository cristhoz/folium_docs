# Comparativa de Sistemas de Gestión Documental
## Orfeo vs Alfresco — Guía de Evaluación para Propuesta Comercial

**Fecha:** Abril 2026  
**Elaborado por:** Cristián Hozman  
**Propósito:** Apoyo técnico para cotización y selección de plataforma DMS

---

## 1. Resumen Ejecutivo

| Característica | Orfeo | Alfresco |
|---|---|---|
| Origen | Colombia (Gobernación de Nariño) | Internacional (ahora parte de Hyland) |
| Licencia | GPL (open source) | Community (GPL) / Enterprise (comercial) |
| Tecnología | PHP + PostgreSQL | Java (Spring) + PostgreSQL/MySQL/Oracle |
| Orientación | Entidades públicas colombianas | Empresas medianas y grandes |
| Curva de aprendizaje | Baja | Alta |
| Costo base | Gratuito | Gratuito (Community) / Alto (Enterprise) |
| Escalabilidad | Media | Alta |

---

## 2. Descripción de los Sistemas

### 2.1 Orfeo

Orfeo es un sistema de gestión documental de código abierto desarrollado en Colombia, orientado principalmente a entidades del sector público. Fue creado por la Gobernación de Nariño y ha sido adoptado por múltiples instituciones gubernamentales colombianas.

**Características principales:**
- Radicación de correspondencia (entrada, salida e interna)
- Control de tiempos de respuesta y vencimientos
- Enrutamiento y flujos de trabajo básicos
- Digitalización y almacenamiento de documentos
- Tablas de Retención Documental (TRD)
- Reportes y auditoría de trazabilidad

**Stack tecnológico:**
- Lenguaje: PHP
- Base de datos: PostgreSQL
- Interfaz: Web (navegador)
- Despliegue: Servidor Linux tradicional o Docker

---

### 2.2 Alfresco

Alfresco es una plataforma de gestión de contenido empresarial (ECM) de clase mundial, con presencia en grandes organizaciones a nivel global. Ofrece dos ediciones: Community (open source) y Enterprise (comercial con soporte oficial).

**Módulos principales:**
- **ACS** — Alfresco Content Services (repositorio central)
- **APS** — Alfresco Process Services (flujos BPMN)
- **AGS** — Alfresco Governance Services (cumplimiento normativo)
- **Digital Workspace** — Interfaz moderna de usuario

**Stack tecnológico:**
- Lenguaje: Java (Spring Framework)
- Base de datos: PostgreSQL, MySQL, MS SQL Server, Oracle
- Motor de búsqueda: Apache Solr
- Almacenamiento: Local, AWS S3, Azure Blob
- Despliegue: Docker, Kubernetes, nube

---

## 3. Comparativa Detallada por Criterios

### 3.1 Funcionalidades

| Funcionalidad | Orfeo | Alfresco |
|---|---|---|
| Radicación de documentos | ✅ Nativo | ✅ Configurable |
| Control de correspondencia | ✅ Nativo | ✅ Con configuración |
| Flujos de trabajo simples | ✅ | ✅ |
| Flujos BPMN complejos | ❌ | ✅ |
| Gestión de versiones | ⚠️ Básica | ✅ Completa |
| Búsqueda avanzada (full-text) | ⚠️ Básica | ✅ Solr integrado |
| Tablas de Retención Documental | ✅ Nativo | ⚠️ Requiere configuración |
| Digitalización | ✅ | ✅ |
| Firma electrónica | ⚠️ Limitada | ✅ Integrable |
| Portal ciudadano / externo | ❌ | ✅ |

---

### 3.2 Integraciones

| Integración | Orfeo | Alfresco |
|---|---|---|
| Office 365 / OneDrive | ❌ | ✅ |
| Google Workspace | ❌ | ✅ |
| SAP / ERP | ❌ | ✅ |
| Sistemas gubernamentales CO | ✅ | ⚠️ |
| API REST | ⚠️ Limitada | ✅ Madura y documentada |
| CMIS (estándar ISO) | ❌ | ✅ |
| Directorio LDAP / Active Directory | ⚠️ | ✅ |
| SSO / SAML / OAuth | ❌ | ✅ |

---

### 3.3 Normativa y Cumplimiento

| Norma / Estándar | Orfeo | Alfresco |
|---|---|---|
| Normativa AGN Colombia | ✅ Nativo | ⚠️ Requiere adaptación |
| Ley General de Archivos (594/2000) | ✅ | ⚠️ |
| TRD y TVD colombianas | ✅ | ⚠️ |
| ISO 15489 (Gestión de registros) | ⚠️ Parcial | ✅ |
| DoD 5015.2 (Registros gubernamentales) | ❌ | ✅ |
| GDPR (protección de datos EU) | ❌ | ✅ |

---

### 3.4 Infraestructura y Despliegue

| Aspecto | Orfeo | Alfresco |
|---|---|---|
| Requisitos de hardware | Bajos | Medios-Altos |
| RAM mínima recomendada | 4 GB | 8–16 GB |
| Soporte Docker | ✅ | ✅ |
| Soporte Kubernetes | ❌ | ✅ |
| Despliegue en nube | ⚠️ Básico | ✅ AWS, Azure, GCP |
| Alta disponibilidad (HA) | ❌ | ✅ |
| Backup y recuperación | Manual | ✅ Automatizable |

---

### 3.5 Costos

| Concepto | Orfeo | Alfresco Community | Alfresco Enterprise |
|---|---|---|---|
| Licencia de software | Gratis | Gratis | Desde ~$20,000 USD/año |
| Implementación | Bajo | Medio | Alto |
| Infraestructura | Baja | Media | Media-Alta |
| Mantenimiento anual | Bajo | Medio | Alto (incluye soporte) |
| Capacitación | Baja | Media | Alta |
| Personalización | Media | Alta | Alta |

> **Nota:** Los costos de Alfresco Enterprise varían según número de usuarios, módulos y proveedor. Se recomienda solicitar cotización oficial.

---

### 3.6 Soporte y Comunidad

| Aspecto | Orfeo | Alfresco Community | Alfresco Enterprise |
|---|---|---|---|
| Soporte oficial | ❌ | ❌ | ✅ SLA garantizado |
| Comunidad activa | ⚠️ Colombia | ✅ Global | ✅ Global |
| Documentación | ⚠️ Escasa | ✅ Completa | ✅ Completa |
| Actualizaciones | Lentas | Regulares | Continuas |
| Proveedores locales CO | ✅ Varios | ⚠️ Pocos | ⚠️ Muy pocos |

---

## 4. Criterios de Selección

### 4.1 Árbol de Decisión

```
¿Es una entidad pública colombiana?
│
├── SÍ
│   ├── ¿Tienen presupuesto > $15,000 USD para software?
│   │   ├── SÍ → Evaluar Alfresco Enterprise con módulo de gestión pública
│   │   └── NO → Orfeo (cumple normativa AGN de forma nativa)
│   │
│   └── ¿Requieren integraciones complejas o más de 200 usuarios?
│       ├── SÍ → Considerar Alfresco Community con personalización
│       └── NO → Orfeo es suficiente
│
└── NO (empresa privada)
    ├── ¿Más de 100 usuarios o flujos de trabajo complejos?
    │   ├── SÍ → Alfresco Community o Enterprise
    │   └── NO → Orfeo o Alfresco Community
    │
    └── ¿Requieren integraciones con Office 365, SAP o ERP?
        ├── SÍ → Alfresco (Community mínimo)
        └── NO → Orfeo puede ser suficiente
```

---

### 4.2 Tabla de Criterios Ponderados

| Criterio | Peso | Orfeo | Alfresco Community | Alfresco Enterprise |
|---|---|---|---|---|
| Cumplimiento normativo CO | 20% | 10 | 5 | 7 |
| Costo total de propiedad | 20% | 10 | 7 | 3 |
| Facilidad de implementación | 15% | 8 | 5 | 4 |
| Escalabilidad | 15% | 4 | 7 | 10 |
| Integraciones | 10% | 3 | 8 | 10 |
| Soporte disponible | 10% | 5 | 5 | 9 |
| Funcionalidades avanzadas | 10% | 4 | 7 | 10 |

> **Instrucción:** Asigne el peso de cada criterio según la realidad del cliente y calcule el puntaje total ponderado para obtener la recomendación objetiva.

---

## 5. Escenarios de Recomendación

### Escenario A — Entidad Pública con Presupuesto Limitado
**Perfil:** Alcaldía, secretaría, entidad descentralizada. Hasta 80 usuarios. Presupuesto bajo.  
**Recomendación:** ✅ **Orfeo**  
**Justificación:** Cumple normativa AGN de forma nativa, costo de implementación bajo, comunidad de soporte local en Colombia.

---

### Escenario B — Empresa Privada Mediana
**Perfil:** Empresa de 100–300 empleados, requiere digitalizar procesos internos, sin integración con ERP.  
**Recomendación:** ✅ **Alfresco Community**  
**Justificación:** Gratuito, escalable, buena documentación, suficiente para flujos internos sin costo de licencia.

---

### Escenario C — Empresa Grande o con Integraciones Complejas
**Perfil:** Empresa de 500+ usuarios, requiere integración con Office 365, SAP u otros sistemas, alta disponibilidad.  
**Recomendación:** ✅ **Alfresco Enterprise**  
**Justificación:** Soporte oficial, SLA garantizado, módulos avanzados, escalabilidad real.

---

### Escenario D — Entidad Pública con Presupuesto Medio-Alto
**Perfil:** Ministerio, gobernación o entidad grande. Más de 200 usuarios. Requieren portal ciudadano.  
**Recomendación:** ✅ **Alfresco Community o Enterprise** con adaptación normativa colombiana  
**Justificación:** Escala mejor, permite portal externo, aunque requiere inversión en personalización para cumplir normativa AGN.

---

## 6. Riesgos por Sistema

### Riesgos de Orfeo
- Documentación escasa y desactualizada en algunos forks
- Sin soporte oficial — dependencia de la comunidad local
- Escalabilidad limitada para grandes volúmenes
- Actualizaciones lentas o inexistentes en algunas versiones

### Riesgos de Alfresco
- Alta complejidad de implementación y mantenimiento
- Costo elevado en versión Enterprise
- Pocos proveedores certificados en Colombia
- La Community Edition queda rezagada respecto a la Enterprise

---

## 7. Checklist de Preguntas para el Cliente

Antes de definir la plataforma, responder:

- [ ] ¿Es entidad pública o empresa privada?
- [ ] ¿Cuántos usuarios usarán el sistema?
- [ ] ¿Cuál es el volumen de documentos por mes?
- [ ] ¿Tienen servidores propios o prefieren nube?
- [ ] ¿Qué sistemas tienen actualmente (ERP, Office, etc.)?
- [ ] ¿Requieren cumplir normativa AGN o TRD?
- [ ] ¿Cuál es el presupuesto disponible (implementación + licencias)?
- [ ] ¿Tienen área de TI para el mantenimiento?
- [ ] ¿Necesitan portal para usuarios externos o ciudadanos?
- [ ] ¿Requieren firma electrónica certificada?

---

## 8. Conclusión

No existe una respuesta universal. La selección debe basarse en los criterios del cliente evaluados objetivamente. Como regla general:

- **Orfeo** es la opción natural para entidades públicas colombianas con presupuesto limitado que necesitan cumplir normativa AGN sin grandes inversiones.
- **Alfresco Community** es ideal para empresas privadas que necesitan escalar y tener integraciones modernas sin pagar licencias.
- **Alfresco Enterprise** está justificado solo cuando el cliente tiene presupuesto, volumen alto, y necesita soporte oficial garantizado.

---

*Documento elaborado como apoyo técnico para proceso de cotización. Versión 1.0 — Abril 2026.*
