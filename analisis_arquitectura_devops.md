# Reporte de Auditoría de Infraestructura: Viabilidad de Arquitectura Multi-Tenant Nextcloud para HRINSER
**Proyecto:** DMS de Documentos Laborales y de Salud (Ley 21.719 - Chile)  
**Autor:** Senior Cloud Infrastructure & DevOps Architect  
**Fecha:** 4 de Julio de 2026  

---

## 1. Resumen de la Arquitectura Propuesta

La propuesta tecnológica para HRINSER contempla el despliegue de cuatro (4) portales independientes basados en **Nextcloud (PHP-Apache)** bajo una arquitectura multi-instancia física a nivel de contenedores. 

### Componentes de la Arquitectura:
*   **Orquestación y Enrutamiento:** Un (1) contenedor central de **Nginx Proxy Manager (NPM)** que actúa como proxy reverso y terminación SSL/TLS, distribuyendo el tráfico basado en nombres de host (`clienteX.hrinser.cl`) a través de redes Docker internas hacia las instancias correspondientes.
*   **Segregación de Clientes (Tenants):** Cuatro (4) stacks de Docker Compose independientes, cada uno compuesto por:
    *   Un (1) contenedor de aplicación Nextcloud (imagen oficial `nextcloud:apache`).
    *   Un (1) contenedor de base de datos relacional dedicada (**PostgreSQL** o **MariaDB**).
    *   Volúmenes locales del host mapeados directamente al sistema de archivos para persistencia de datos de usuario y bases de datos.
*   **Hardware Proyectado:** Un único VPS en la plataforma Elestio con un costo de $20-$30 USD/mes, cuyas especificaciones típicas son **4 vCPUs (compartidas), 8 GB de RAM y 160 GB de almacenamiento SSD/NVMe**.
*   **Métricas de Carga Estimadas:** 
    *   10 usuarios concurrentes globales distribuidos entre las 4 empresas.
    *   100 GB de almacenamiento histórico consolidado.
    *   Procesamiento de OCR masivo (vía Tesseract/Nextcloud OCR) al inicio del proyecto.

---

## 2. Análisis de Viabilidad de Hardware y Recursos (CPU, RAM, Disco)

El dimensionamiento de recursos para esta arquitectura es altamente crítico debido a la naturaleza síncrona y pesada de la pila LAMP/LEMP tradicional sobre la que corre Nextcloud.

### A. Consumo de Memoria RAM (El Factor Limitante Principal)

Nextcloud en su distribución basada en Apache y PHP-FPM / `mod_php` consume cantidades considerables de memoria RAM bajo carga mínima.

*   **Nginx Proxy Manager:** ~150 MB a 250 MB RAM (incluyendo procesos de OpenResty y base de datos interna SQLite/MariaDB para gestión de certificados).
*   **Bases de Datos (PostgreSQL/MariaDB):** Cada instancia de PostgreSQL requiere configuraciones de `shared_buffers` y `work_mem`. Una configuración mínima para producción estable requiere un piso de:
    $$\text{RAM por DB} \approx 512\text{ MB} \implies 4 \times 512\text{ MB} = 2.0\text{ GB RAM}$$
    *Si se utiliza MariaDB con un `innodb_buffer_pool_size` restrictivo de 384 MB, el consumo mínimo por instancia con buffers adicionales y threads es de ~500 MB.*
*   **Contenedores de Aplicación Nextcloud (PHP-Apache):**
    *   Cada proceso PHP de Nextcloud consume entre 128 MB y 512 MB de RAM (limitado por `memory_limit` en `php.ini`, típicamente configurado en 512 MB por Nextcloud para operaciones de carga y manipulación de archivos).
    *   Con un número bajo de workers por contenedor (ej. 4 workers Apache concurrentes por portal), cada contenedor consumirá un mínimo de 512 MB (idle/bajo tráfico) hasta 2.0 GB (bajo carga de sincronización o renderizado de vistas previas).
    $$\text{RAM Mínima (Apps)} = 4 \times 512\text{ MB} = 2.0\text{ GB RAM (Teórico Idle)}$$
    $$\text{RAM Pico (Apps)} = 4 \times 1.5\text{ GB} = 6.0\text{ GB RAM (Sincronización/Uso de WebDAV)}$$
*   **Procesamiento de OCR (Tesseract):**
    *   Tesseract OCR no corre de forma nativa en la GPU dentro de este tipo de VPS y consume memoria RAM dinámica de manera intensiva por cada página procesada (aproximadamente 200 MB - 500 MB por hilo de ejecución).
*   **Sistema Operativo y Overhead de Docker Daemon:** ~500 MB - 800 MB.

| Componente | RAM Mínima (Idle) | RAM Requerida en Pico de Carga |
| :--- | :--- | :--- |
| OS + Docker Daemon | 500 MB | 800 MB |
| Nginx Proxy Manager | 150 MB | 250 MB |
| 4x PostgreSQL Instances | 1.5 GB | 2.5 GB (Optimizadas) |
| 4x Nextcloud Apache | 2.0 GB | 6.0 GB (Peak WebDAV / Cron) |
| OCR (Tesseract / Redis cache) | 0 MB (Inactivo) | 1.5 GB (Procesamiento concurrente) |
| **Total Requerido** | **4.15 GB** | **11.05 GB** |

> [!WARNING]
> **Riesgo Crítico de OOM Killer:** Con 8 GB de RAM física en el VPS, en un escenario donde se activen tareas de fondo (`cron.php`) concurrentes en los 4 portales o se carguen archivos pesados de salud/contratos, el sistema superará los 8 GB físicos. Esto forzará al kernel de Linux a invocar el **Out-Of-Memory (OOM) Killer**, dando de baja aleatoriamente contenedores de base de datos o de Nextcloud, generando corrupción de datos o downtime no planificado.

### B. Rendimiento y Compartición de CPU (vCPUs)

El procesador de un VPS de $20-$30 USD suele ser de hilos compartidos (*shared vCPUs*). 
*   **Procesamiento de OCR:** Tesseract es una biblioteca C++ monohilo por imagen, pero si procesa documentos PDF de muchas páginas en paralelo, intentará consumir el 100% de los núcleos de CPU asignados.
*   **PHP-Apache:** Al no usar PHP-FPM optimizado con Nginx en el contenedor de aplicación (la imagen oficial `nextcloud:apache` levanta un servidor Apache `mpm_prefork`), cada petición de carga o descarga bloquea un hilo de ejecución.
*   **CPU Steal Time:** Al estar en un VPS de recursos compartidos, un pico de uso de CPU del 100% sostenido por el OCR provocará que el proveedor limite el procesamiento (*throttling*), elevando el *CPU Steal Time* por encima del 15-20%, lo que degradará la latencia del API de Nextcloud a segundos por petición, congelando la interfaz para todos los usuarios.

---

## 3. Impacto del I/O de Disco (I/O Bottleneck / I/O Wait)

El talón de Aquiles de correr múltiples bases de datos relacionales en almacenamiento virtualizado no dedicado es el **I/O Wait**.

```mermaid
graph TD
    subgraph Host VPS (SSD/NVMe Compartido)
        HostOS[Kernel de Linux - Sistema de Archivos Host]
    end
    subgraph Red Docker
        NPM[Nginx Proxy Manager]
    end
    subgraph Stacks de Clientes
        NC1[Nextcloud App 1] --> DB1[(PostgreSQL 1)]
        NC2[Nextcloud App 2] --> DB2[(PostgreSQL 2)]
        NC3[Nextcloud App 3] --> DB3[(PostgreSQL 3)]
        NC4[Nextcloud App 4] --> DB4[(PostgreSQL 4)]
    end

    DB1 & DB2 & DB3 & DB4 -->|Lecturas/Escrituras WAL masivas| HostOS
    HostOS -->|Cuello de botella de IOPS limitadas| Disco[NVMe Físico Compartido]
```

### A. El problema de la concurrencia en escritura de BDs y Archivos
Nextcloud realiza un volumen masivo de consultas SQL por cada acción simple del usuario (listar un directorio genera decenas de llamadas para validar metadatos, permisos, locks de archivos en `oc_file_locks` y almacenamiento).
*   **Locks y transacciones concurrentes:** Con 4 bases de datos escribiendo en sus respectivos Write-Ahead Logs (WAL) sobre el mismo sistema de archivos físico:
    *   Las escrituras a disco no son secuenciales, se dispersan por los diferentes directorios mapeados en el host (`/var/lib/docker/volumes/...`).
    *   Aunque el disco sea NVMe, los proveedores de VPS económicos limitan los **IOPS** (operaciones de entrada/salida por segundo) a valores entre **500 y 3000 IOPS** ráfaga.
*   **Picos de OCR:** Durante la indexación inicial de PDFs escaneados, el proceso de OCR lee archivos grandes del disco, genera temporales en `/tmp` y escribe datos de texto extraídos masivamente en la base de datos para habilitar la búsqueda indexada.
*   **Consecuencia Práctica:** El valor de **`iowait`** (medido mediante `iostat` o `top`) superará el **15% - 25%**. Cuando el kernel pasa este porcentaje de tiempo esperando que el disco complete operaciones de escritura/lectura, el procesador se bloquea en modo de espera no interrumpible (D-state). La consecuencia inmediata es la lentitud extrema de todo el sistema y la pérdida de conexión por *timeout* en Nginx Proxy Manager (Error 504 Gateway Timeout).

---

## 4. Análisis de Seguridad y Segregación de Datos a nivel de Docker/Kernel

Dado que HRINSER maneja información sensible bajo la **Ley 21.719 en Chile** (que regula el teletrabajo, registro de asistencia y la protección de datos personales y sensibles de salud), la segregación de datos es un requerimiento legal, no solo técnico.

### A. El riesgo del Kernel Compartido
Los contenedores Docker comparten el **mismo Kernel del sistema operativo host**. No proveen una barrera de hardware real como las máquinas virtuales (KVM/Hyper-V).
*   **Vulnerabilidades de Docker Escape:** Si un atacante logra comprometer la instancia Nextcloud de un cliente (por ejemplo, explotando una vulnerabilidad RCE en PHP o en una biblioteca del sistema como ImageMagick), obtendrá ejecución de código como el usuario `www-data` (o `root` dentro del contenedor si no se ha configurado la directiva `userns-remap` o ejecutado Rootless Docker).
*   **Escalada de privilegios vía System Calls:** Al compartir el kernel, fallos de seguridad a nivel de llamadas al sistema (como *Dirty Pipe*, *Dirty COW* o fallas en el subsistema `io_uring`) permiten a un contenedor comprometido escapar al host principal.
*   **Compromiso Total:** Una vez que el atacante accede al host como `root`, tiene acceso inmediato a los sockets de Docker de los otros 3 clientes y a sus volúmenes en disco. **Esto destruye la confidencialidad exigida por la legislación chilena y expone a la empresa a graves multas por fuga de datos de salud y laborales.**

### B. Aislamiento de Redes en Docker
Por defecto, si todos los Docker Compose se configuran sin una arquitectura de red explita y aislada:
1.  Si comparten la red por defecto de Docker (`bridge`), un contenedor comprometido en el Stack 1 puede realizar escaneos de puertos y ataques de suplantación ARP/DNS contra las bases de datos de los Stacks 2, 3 y 4.
2.  Para evitar esto, cada Stack de cliente debe correr en su propia red interna aislada y NPM debe comunicarse con cada portal mediante redes creadas específicamente para el proxy, limitando el acceso cruzado.

### C. Almacenamiento no Cifrado
En un VPS básico de Elestio, los volúmenes están mapeados directamente en directorios locales `/var/lib/...` en texto plano. Si el proveedor del VPS sufre un incidente de seguridad, o si las credenciales de la consola de Elestio son comprometidas, cualquier atacante puede clonar los discos y leer la información confidencial de salud sin restricciones.

---

## 5. Puntos de Falla Críticos Identificados

1.  **Agotamiento de RAM (OOM Killer):** 8 GB de RAM son insuficientes para sostener 9 contenedores en ejecución (4 portales Nextcloud + 4 BDs + 1 NPM) bajo operaciones concurrentes de sincronización de archivos y OCR.
2.  **Cuello de Botella de IOPS en Disco:** El uso de 4 instancias de base de datos ejecutando de forma independiente sobre un único disco virtualizado sin límites de I/O provocará un congelamiento del sistema en cargas de trabajo de OCR o subidas masivas de documentos.
3.  **Segregación de Datos Débil (No Compliant con Ley de Datos Personales):** Compartir el kernel en una única máquina virtual de bajo costo significa que un ataque exitoso a un solo cliente compromete el 100% de la base de datos de salud y contratos de los demás clientes.
4.  **Carencia de Redis para Gestión de Bloqueos:** Nextcloud requiere de forma obligatoria un sistema de almacenamiento en caché en memoria (Redis) para manejar los bloqueos de archivos (`file locking`). Si no se implementa, Nextcloud delega esto en la base de datos, multiplicando por 5 las operaciones de escritura en disco y acelerando el colapso del I/O. La propuesta actual no contempla instancias Redis, lo que causará degradación de rendimiento inmediata.

---

## 6. Recomendaciones de Mitigación y Ajuste Técnico

Para viabilizar este modelo y garantizar la seguridad legal y operativa del DMS de HRINSER, se deben aplicar las siguientes medidas de re-diseño:

### A. Optimización de Arquitectura y Recursos

1.  **Escalar el Hardware:** Incrementar la capacidad a un VPS de mínimo **16 GB de RAM** y **8 vCPUs dedicadas** (no compartidas) si se mantiene el esquema de 4 instancias separadas.
2.  **Unificar la Base de Datos (Opcional pero Recomendado para Ahorro de Recursos):**
    *   En lugar de correr 4 contenedores PostgreSQL independientes, se puede desplegar **un solo contenedor PostgreSQL centralizado** y bien optimizado (ej: configurado con `pgtune`), creando 4 bases de datos y 4 usuarios con permisos estrictamente segregados.
    *   *Ahorro estimado:* ~1.2 GB a 1.5 GB de RAM libre en el sistema y menor cantidad de conexiones concurrentes y buffers en disco compitiendo entre sí.
3.  **Migrar a Nextcloud con PHP-FPM y Nginx:**
    *   Reemplazar la imagen `nextcloud:apache` por `nextcloud:fpm` combinada con un contenedor Nginx ligero por cliente. PHP-FPM consume sustancialmente menos memoria que Apache `mpm_prefork` y permite limitar de forma estricta los procesos hijo máximos (`pm.max_children = 5`).
4.  **Implementar Redis Centralizado:**
    *   Desplegar un único contenedor Redis centralizado con bases de datos lógicas aisladas (`database 0` para cliente 1, `database 1` para cliente 2, etc.) para delegar el caché de metadatos y el bloqueo de archivos de Nextcloud. Esto reducirá el I/O en disco en un 60%.

### B. Endurecimiento de Seguridad (Hardening)

1.  **Cifrado en Reposo (Encryption at Rest):**
    *   Habilitar el cifrado del lado del servidor (*Server-Side Encryption*) dentro de cada Nextcloud o utilizar volumenes cifrados en el host mediante LUKS.
2.  **Segregación de Redes Estricta:**
    *   Definir redes Docker independientes para cada cliente. El contenedor de Nginx Proxy Manager debe ser el único miembro externo conectado a las subredes de frontend de cada Nextcloud.
3.  **Configurar Limites de Recursos en Docker Compose:**
    *   Limitar por código la RAM y CPU máxima que puede consumir cada contenedor para evitar que un proceso de OCR de un portal tire abajo el servidor completo:
    ```yaml
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 1.5G
        reservations:
          memory: 512M
    ```
4.  **Ejecutar OCR fuera de Horas Laborales:**
    *   Configurar los procesos de Tesseract de Nextcloud para que se ejecuten mediante tareas programadas (`cron` del sistema host) durante la noche, limitando su ejecución en horario laboral para evitar picos de uso del CPU.

---

## 7. Dictamen Final de Viabilidad de Infraestructura

> [!CAUTION]
> **DICTAMEN: INVIABLE EN SU ESTADO ACTUAL (CRÍTICO)**
> 
> La arquitectura propuesta sobre un único VPS de $20-$30 USD (4 vCPUs / 8 GB RAM) **no es viable para producción segura y estable** bajo las condiciones de HRINSER. 
> 
> Si bien la carga de 10 usuarios concurrentes es baja, la sobrecarga (*overhead*) de correr 9 contenedores simultáneos (4 Nextcloud + 4 BDs + 1 Proxy), sumado al procesamiento pesado de OCR y a la falta de Redis, causará el colapso del servidor por agotamiento de memoria RAM (OOM) e I/O Wait en el disco durante las cargas iniciales de datos.
> 
> Asimismo, **no cumple con las garantías óptimas de confidencialidad y segregación real de datos exigibles para información sensible de salud (Ley 21.719)** debido a que el aislamiento a nivel de contenedores Docker estándar comparte kernel, haciendo vulnerable a toda la infraestructura si una sola de las instancias de los clientes es compromised.

### Alternativa de Infraestructura Recomendada:
Para mantener costos bajos y cumplir con la legislación, se recomienda migrar a un esquema de **Nextcloud Multi-Tenant nativo** (una única instancia grande de Nextcloud con control estricto de grupos y almacenamiento cifrado por cliente) montada sobre un VPS con discos NVMe dedicados, o bien, distribuir los 4 portales independientes en 4 servidores VPS pequeños dedicados de 2 GB de RAM y 2 vCPUs cada uno en lugar de centralizarlos en un solo nodo compartiendo recursos críticos de manera ineficiente.
