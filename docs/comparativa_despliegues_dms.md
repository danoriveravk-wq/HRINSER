# ⚖️ Comparativa de Soluciones DMS: Mayan EDMS vs. Nextcloud Multi-Instancia
### Evaluación Estratégica para HRINSER / Gestión Comercial SpA

Este documento analiza comparativamente las dos opciones de despliegue blindadas propuestas para la gestión documental de HRINSER, evaluándolas bajo la Ley 21.719 de Protección de Datos Personales, viabilidad técnica de almacenamiento (100 GB) y facilidad operativa.

---

## 📊 Tabla Comparativa Resumida

| Criterio | Mayan EDMS (Instancia Única) | Nextcloud (Multi-Instancia) | Ganador |
|---|---|---|---|
| **Aislamiento de Datos** | ⚠️ Lógico (Riesgo de error de ACL) | ✅ Físico (100% aislado en Docker/BD) | **Nextcloud** |
| **Experiencia de Usuario (UI)** | 🔴 Compleja y técnica (Mala UX) | 🟢 Familiar e intuitiva (Tipo Google Drive) | **Nextcloud** |
| **Cumplimiento Ley 21.719** | ✅ Completo (Si está bien configurado) | ✅ Nativo y fácil de comprobar | **Nextcloud** |
| **Logs de Auditoría** | ✅ Inmutables y nativos | ✅ Inmutables (vía apps de auditoría) | Empate |
| **Complejidad de Setup** | 🔴 Muy alta (Celery, Redis, Django, Postgres) | 🟡 Media (Nginx Proxy Manager + Docker) | **Nextcloud** |
| **Consumo de Recursos** | 🟡 Moderado (1 instancia robusta) | 🔴 Alto (Múltiples BDs y apps corriendo) | **Mayan EDMS** |
| **Procesamiento de OCR** | ✅ Nativo y altamente automatizado | ⚠️ Requiere configuraciones y apps extra | **Mayan EDMS** |
| **Mantenimiento Técnico** | 🔴 Alto (Monitoreo de colas Celery/OCR) | 🟡 Medio (Parches individuales simples) | **Nextcloud** |

---

## 🅰️ Mayan EDMS: Pros y Contras

### 👍 Ventajas (Pros)
1.  **Automatización Inteligente:** Indexación automática basada en OCR y extracción de metadatos. Puede leer el RUT de un PDF y clasificarlo solo.
2.  **Eficiencia en un solo Servidor:** Al ser una sola aplicación ejecutándose, consume menos recursos de RAM en reposo en comparación con levantar 5 bases de datos distintas.
3.  **Logs de Auditoría Internos Estrictos:** Registro inmutable del ciclo de vida del documento diseñado para corporaciones.

### 👎 Desventajas (Contras)
1.  **Riesgo Crítico de Error de ACL:** Si Vanessa comete un error configurando una regla de acceso (ACL), el usuario de la Empresa A podría ver documentos confidenciales de la Empresa B. Al compartir la base de datos, el riesgo de filtración cruzada es alto.
2.  **Interfaz Compleja (UX Hostil):** Alta probabilidad de que tus clientes no quieran usar la plataforma directamente debido a la dificultad de navegación, forzando a que HRINSER deba operar manualmente por ellos.
3.  **Setup muy frágil:** Si el proceso de OCR consume toda la memoria, la base de datos principal se congela y detiene el servicio para todos tus clientes al mismo tiempo.

---

## 🅱️ Nextcloud Multi-Instancia: Pros y Contras

### 👍 Ventajas (Pros)
1.  **Seguridad Física Absoluta (Zero Leak Risk):** Cada empresa cliente tiene su propia base de datos PostgreSQL/MariaDB y su propia carpeta de almacenamiento encriptada. Es físicamente imposible que una empresa acceda a los datos de otra.
2.  **UI Familiar e Intuitiva (Google Drive Open Source):** Los clientes finales de HRINSER saben usarlo de forma nativa. Vanessa puede arrastrar y soltar archivos sin requerir entrenamiento técnico.
3.  **Fácil Soporte y Resiliencia:** Si la instancia de la Empresa A sufre una desconfiguración o se llena de espacio, las instancias de la Empresa B y C siguen operando con normalidad. Los backups son individuales y fáciles de restaurar por cliente.
4.  **Cumplimiento Demostrable:** Ante una auditoría de la Agencia de Datos, es infinitamente más fácil demostrar cumplimiento legal mostrando contenedores físicamente separados por RUT de cliente que mostrar un árbol complejo de ACLs lógicas de Mayan.

### 👎 Desventajas (Contras)
1.  **Mayor Consumo de RAM en Reposo:** Cada instancia requiere levantar su propio motor de base de datos y servidor web PHP. Esto obliga a contratar un VPS con un poco más de RAM a medida que crezca el número de clientes (ej: pasar de 8 GB a 16 GB).
2.  **OCR menos robusto nativamente:** Nextcloud no está diseñado principalmente como un archivador inteligente OCR. La lectura de PDFs escaneados requiere plugins y el procesamiento consume recursos del servidor que hay que programar por las noches.

---

## 💰 Comparativa de Costos Estimados (Elestio VPS)

*   **Opción A: Mayan EDMS (1 Instancia)**
    *   Servidor recomendado: 4 vCPU, 8 GB RAM, 160 GB SSD.
    *   Costo mensual estimado: **~$25 - $40 USD / mes**.
*   **Opción B: Nextcloud Multi-Instancia (Hasta 6 clientes en el mismo host)**
    *   Servidor recomendado: 4 vCPU, 12 GB - 16 GB RAM, 200 GB SSD.
    *   Costo mensual estimado: **~$45 - $60 USD / mes**.

---

## 🏆 Veredicto y Recomendación Final

> [!IMPORTANT]  
> Para el negocio real de **HRINSER** hoy, la **Opción B: Nextcloud Multi-Instancia es la ganadora indiscutible.**

### Justificación de Negocio y Legal:
1.  **Protección Legal Absoluta:** La Ley 21.719 es sumamente severa con la mezcla de datos sensibles. El aislamiento físico de Nextcloud en Docker elimina de raíz el riesgo de que tus clientes vean liquidaciones o licencias médicas ajenas por un error de configuración de roles en Mayan.
2.  **Adopción Comercial:** Un DMS no sirve de nada si tus clientes se quejan de la interfaz o Vanessa pasa horas dando soporte de cómo usarlo. Nextcloud se siente moderno y familiar.
3.  **Facilidad Técnica en Antigravity 2.0:** Yo puedo escribir la configuración completa de Nginx Proxy Manager y la plantilla de Nextcloud en Docker. Esto reduce tu tiempo de desarrollo e implementación a solo unas horas de setup inicial.
