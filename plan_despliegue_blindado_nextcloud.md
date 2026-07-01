# 🗺️ Plan de Despliegue Blindado para Nextcloud (Elestio)
### Enfoque Multi-Instancia y Cumplimiento Normativo (Ley 21.719)

Este plan de despliegue propone una arquitectura de **aislamiento físico total** levantando una instancia independiente de Nextcloud (con su propia base de datos y volumen de almacenamiento) para cada cliente de HRINSER. Esto reduce a cero el riesgo de fugas de información cruzada y garantiza el cumplimiento del principio de confidencialidad de la Ley 21.719.

---

## 📦 Paso 1: Configurar la Infraestructura y el Servidor en Elestio

### 1. Dimensionamiento del Servidor
*   **Capacidad del Host:** Para servir a ~5-10 empresas de forma inicial con instancias independientes de Nextcloud, seleccionamos un VPS con:
    *   **CPU:** 4 vCPUs.
    *   **RAM:** 8 GB o 16 GB de RAM (cada Nextcloud en Docker consume ~500 MB a 1 GB en reposo).
    *   **Almacenamiento:** 250 GB SSD/NVMe con **cifrado en reposo activo** a nivel de volumen del VPS.
*   **Acceso al Servidor:** Activar 2FA obligatorio en el panel de administración de Elestio para los administradores de HRINSER.

### 2. Configuración del Orquestador (Nginx Proxy Manager)
*   Desplegar un contenedor de **Nginx Proxy Manager (NPM)** en el servidor. NPM actuará como la puerta de entrada única del servidor, recibiendo el tráfico web y enrutándolo a la instancia correcta de Nextcloud según el subdominio.
*   NPM administrará y renovará automáticamente los certificados SSL/TLS para cada subdominio de cliente.

---

## 🌐 Paso 2: Estructura de Subdominios y Aislamiento por Cliente

### 1. Mapeo de Subdominios
Por cada cliente contratista de HRINSER se registrará un subdominio específico apuntando a la IP del VPS:
*   `clientealfa.hrinser.cl`
*   `clientebeta.hrinser.cl`

### 2. Despliegue de Instancias Docker por Cliente
Cada cliente tendrá su propio archivo `docker-compose.yml` que levantará:
1.  **Contenedor Nextcloud App:** Aislado en su propia red Docker interna.
2.  **Contenedor de Base de Datos (PostgreSQL o MariaDB):** Exclusivo para la instancia de ese cliente.
3.  **Volumen de Datos Cifrado:** Carpeta montada en el VPS donde se guardan los archivos físicos de ese cliente en específico.

```
[Tráfico de Red]
       │
       ▼
┌──────────────────────────────────────────────┐
│        Nginx Proxy Manager (SSL)             │
└──────────────────────┬───────────────────────┘
                       │
       ┌───────────────┴───────────────┐
       ▼                               ▼
┌──────────────┐                ┌──────────────┐
│ Nextcloud    │                │ Nextcloud    │
│ Cliente Alfa │                │ Cliente Beta │
├──────────────┤                ├──────────────┤
│ DB Alfa      │                │ DB Beta      │
├──────────────┤                ├──────────────┤
│ Vol. Alfa    │                │ Vol. Beta    │
└──────────────┘                └──────────────┘
```

---

## 👥 Paso 3: Configuración Interna de Nextcloud por Instancia

Una vez levantada la instancia de un cliente específico (ej: `clientealfa.hrinser.cl`):

### 1. Usuarios y Control de Acceso
*   **Usuarios Internos de HRINSER:** Crear las cuentas para Vanessa y Daniel con rol de administración de la instancia.
*   **Usuarios del Cliente:** Crear cuentas individuales para los encargados de RRHH o administración del cliente.
*   **Doble Factor de Autenticación (2FA) Obligatorio:** Instalar la aplicación oficial de Two-Factor Authentication en Nextcloud y forzar su uso obligatorio para todos los usuarios.

### 2. Endurecimiento de Seguridad en Nextcloud
*   Desactivar la opción de crear enlaces públicos compartidos sin contraseña o sin fecha de expiración.
*   Configurar la aplicación oficial de **Server-side Encryption** de Nextcloud para cifrar los archivos directamente en el almacenamiento antes de escribirlos a disco.

---

## 📂 Paso 4: Ingesta Masiva y Operación Diaria de los 100 GB

### 1. Ingesta Inicial de Históricos
*   Utilizar la herramienta de línea de comando `occ` de Nextcloud para importar archivos grandes directamente desde el servidor local (SSH) al volumen del cliente respectivo.
*   Comando para escanear y sincronizar los archivos:
    ```bash
    docker exec --user www-data nextcloud-app-alfa php occ files:scan --all
    ```
*   Esto evita la subida manual de 100 GB por navegador web, reduciendo la carga y eliminando errores de red.

### 2. OCR en Nextcloud (Lectura de PDFs)
*   Instalar la aplicación **"OCRMyPDF"** o similar en Nextcloud.
*   Para evitar picos de uso del servidor, configurar la tarea cron de Nextcloud para que ejecute el indexado OCR únicamente en horas de la noche (madrugada).

---

## 📊 Paso 5: Gestión de Backups y Trazabilidad (Cumplimiento Ley 21.719)

### 1. Registro de Actividades Inmutable (Audit Trail)
*   Habilitar y configurar la aplicación oficial **"Activity"** e **"Auditing / Logging"** en cada instancia de Nextcloud.
*   Configurar los logs para registrar obligatoriamente:
    *   Logins de usuarios (exitosos y fallidos con IP de origen).
    *   Visualizaciones y descargas de cualquier documento.
    *   Creación, edición o eliminación de archivos.
*   Enviar los logs a un archivo syslog seguro en el VPS para evitar alteraciones.

### 2. Backups Independientes por Cliente
*   Configurar copias de seguridad diarias en Elestio.
*   Al ser instancias separadas, es posible respaldar y restaurar la base de datos y archivos de la **Empresa A** de forma independiente sin interferir en los datos ni la operación de la **Empresa B**.
*   Los respaldos son enviados de forma cifrada a un bucket S3 externo.
