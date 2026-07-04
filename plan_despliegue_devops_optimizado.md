# 🗺️ Plan de Despliegue DevOps Optimizado: Nextcloud Multi-VPS
### Arquitectura Segura, Escalable y Blindada bajo la Ley 21.719

Este documento describe el plan de implementación final para el sistema de gestión documental (DMS) de **HRINSER**. Corrige los fallos críticos de rendimiento y seguridad de la propuesta anterior mediante el uso de una arquitectura de **Servidores Virtuales (VPS) Dedicados e Independientes** por cada cliente.

---

## 🛑 1. Diagnóstico: ¿Por qué fallaba el plan anterior?

La auditoría técnica identificó tres cuellos de botella críticos al meter 4 portales independientes compartiendo un único VPS de 8 GB de RAM:
1.  **Downtime por OOM (Out of Memory):** El consumo en reposo de 9 contenedores Docker (4 de Nextcloud, 4 de bases de datos y 1 de proxy reverso) sumaba ~4.7 GB de RAM. Al iniciar la ingesta masiva de 100 GB y ejecutarse los procesos de OCR (Tesseract), la RAM colapsaba, activando el OOM Killer del sistema operativo y apagando la base de datos PostgreSQL de forma aleatoria.
2.  **Cuello de botella de disco (I/O Wait):** Las 4 bases de datos compitiendo por operaciones de escritura (IOPS) en un único disco SSD/NVMe virtualizado no dedicado ralentizaban drásticamente la velocidad de respuesta de las aplicaciones.
3.  **Riesgo de Seguridad e Incumplimiento Legal:** Al compartir el mismo Kernel de Linux y la misma red interna de Docker, un exploit o fallo de seguridad en el portal de un cliente comprometía de inmediato la privacidad de los datos de salud y liquidaciones de todos los demás clientes (Violación directa de la Ley 21.719).

---

## 🛠️ 2. Las Mejoras Aplicadas (El Nuevo Enfoque)

| Criterio | Plan Anterior | Plan Optimizado (Multi-VPS) | Impacto |
|---|---|---|---|
| **Aislamiento** | Lógico (Contenedores Docker compartiendo Kernel) | **Físico (Servidores VPS independientes)** | Seguridad total, cumple al 100% la Ley 21.719 |
| **Bases de Datos** | 4 bases de datos compitiendo en el mismo disco | **1 base de datos dedicada por VPS independiente** | Cero cuellos de botella de I/O de disco |
| **Resiliencia** | Si el servidor principal falla, todos los portales caen | **Si un VPS falla, los demás siguen funcionando** | Alta disponibilidad y estabilidad operativa |
| **Gestión de OCR** | OCR automático descontrolado e inmediato | **OCR controlado vía Cronjob en la madrugada** | Prevención de saturación de memoria RAM |
| **Costo por Cliente** | Fijo en VPS grande (~$30 USD/mes) | **Escalable (~$8-$10 USD/mes por cliente activo)** | Mejor escalabilidad comercial y financiera |

---

## 📋 3. Paso a Paso del Despliegue e Implementación

### Paso 1: Configurar la Cuenta e Infraestructura en Elestio
1.  Crear la cuenta corporativa en Elestio a nombre de **Gestión Comercial SpA** con un correo centralizado (ej: `seguridad@hrinser.cl`).
2.  Activar obligatoriamente el **Doble Factor de Autenticación (2FA)** en el acceso de administración del panel de Elestio.
3.  Configurar las reglas de firewall globales para bloquear puertos innecesarios, dejando abiertos únicamente el puerto `80` (HTTP), `443` (HTTPS) y restringiendo el puerto `22` (SSH) solo a llaves autorizadas.

### Paso 2: Creación del Primer Nodo de Cliente (Servidor Dedicado)
Por cada cliente nuevo de HRINSER:
1.  Lanzar un servicio de **Nextcloud** en Elestio.
2.  **Especificaciones de hardware recomendadas por cliente:**
    *   1 vCPU
    *   2 GB de RAM
    *   40 GB a 80 GB SSD (según el volumen inicial del cliente contratista)
3.  Habilitar la opción **Cifrado en Reposo (Encryption at Rest)** en la configuración de disco que ofrece Elestio antes del despliegue.

### Paso 3: Vinculación de Dominio y SSL/TLS
1.  Mapear el subdominio del cliente (ej: `alfa.hrinser.cl`) en tu proveedor de DNS (GoDaddy, Cloudflare, etc.) apuntando a la dirección IP pública del VPS recién creado.
2.  Configurar el dominio en el panel del servicio de Elestio. El proxy SSL integrado generará y renovará automáticamente el certificado **SSL/TLS de Let's Encrypt** para garantizar HTTPS.

### Paso 4: Optimización del Rendimiento (OCR & Cache)
1.  **Instalación del motor OCR:** Instalar la aplicación oficial de OCR en la instancia del cliente.
2.  **Programación del Cron de Nextcloud (Crítico):**
    *   Cambiar la ejecución de tareas en segundo plano de AJAX a **Cron de Sistema (System Cron)**.
    *   Configurar un cronjob en el host para ejecutar el indexado OCR únicamente a las **03:00 AM** diariamente:
        ```bash
        0 3 * * * docker exec -u www-data nextcloud-app php occ ocr:process
        ```
    *   Esto garantiza que el uso de RAM de Tesseract no afecte a los usuarios durante el horario de oficina.

### Paso 5: Estructura de Carpetas, Acreditación y 2FA
1.  Activar la aplicación de **2FA obligatorio** en la configuración interna de la instancia de Nextcloud para todos los usuarios clientes.
2.  Configurar la plantilla de inicio de carpetas (Skeleton) de Nextcloud para crear automáticamente la estructura básica al crear usuarios:
    *   `01_Contratos_y_Anexos`
    *   `02_Liquidaciones_y_Cotizaciones`
    *   `03_Examenes_y_Certificados_Salud`
3.  Configurar las reglas de **Logs de Auditoría (Activity)** para registrar de forma persistente y detallada quién descarga o visualiza archivos con datos personales y de salud.
