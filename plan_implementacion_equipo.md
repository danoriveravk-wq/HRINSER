# 🛠️ Guía de Implementación y Despliegue Técnico: Ecosistema DMS & CRM HRINSER
### Manual de Ejecución Paso a Paso para Desarrolladores (Cumplimiento Ley 21.719 - Chile)
**Código del Documento:** HR-DEV-2026-001  
**Preparado por:** Lead DevOps & Systems Architect  
**Destinatarios:** Equipo de Desarrollo, DevOps e Ingenieros de Software  
**Fecha:** 13 de Julio de 2026  

---

## 1. Resumen de la Arquitectura del Sistema

Esta guía detalla las tareas técnicas y de infraestructura necesarias para implementar y desplegar el ecosistema digital de **HRINSER** (Gestión Comercial SpA). 

Dado que se cuenta contratado el plan **Hostinger Shared Web Hosting (Plan Unlimited)**, la arquitectura se optimiza para no requerir un VPS adicional para el CRM, reduciendo costos operacionales. La Landing Page y el CRM se desplegarán de forma nativa (sin contenedores) en Hostinger. La plataforma de almacenamiento documental (DMS), por sus requisitos de OCR pesado, volumen e inmutabilidad, se mantiene sobre servidores dedicados en Elestio:

1. **Landing Page Corporativa:** Frontend estático desplegado en **Hostinger Web Hosting (Plan Unlimited)**.
2. **Backoffice CRM de Código Abierto (EspoCRM):** Instalación nativa basada en la pila **PHP / MySQL** dentro del mismo plan **Hostinger Shared Web Hosting**. Se configuran las bases de datos y la ejecución del script cron mediante el hPanel, parametrizando el pipeline de ventas de 10 etapas en la entidad Oportunidades y campos personalizados mediante el Entity Manager.
3. **Plataforma de Gestión Documental (DMS Nextcloud):** Arquitectura **Multi-VPS en Elestio**. Cada cliente contratista opera en una máquina virtual dedicada con su propio stack aislado en Docker (Nextcloud Core + PostgreSQL + Redis + Cron), cumpliendo de manera estricta las exigencias de privacidad de la **Ley 21.719**.

```
[Cliente Web] ---> [DNS Cloudflare / Hostinger]
                          │
       ┌──────────────────┴────────────────────────────────────────────────┐ (HTTPS / TLS 1.3)
       ▼                                   ▼                               ▼
┌──────────────┐                    ┌──────────────┐               ┌──────────────┐
│ Hostinger    │                    │ VPS Cliente1 │               │ VPS Cliente2 │
│ (Shared Host)│                    │ (Elestio NC) │               │ (Elestio NC) │
├──────────────┤                    ├──────────────┤               ├──────────────┤
│ Landing Page │                    │ NC App       │               │ NC App       │
│ EspoCRM PHP  │                    │ Postgres DB  │               │ Postgres DB  │
│ MySQL DB     │                    │ Redis Cache  │               │ Redis Cache  │
│              │                    │ Vol. Cifrado │               │ Vol. Cifrado │
└──────────────┘                    └──────────────┘               └──────────────┘
```

---

## 2. Fase 1: Configuración de DNS, Landing Page y Despliegue de EspoCRM en Hostinger (Shared Hosting)

### Paso 2.1: Gestión de DNS en Hostinger o Cloudflare
1. Configurar la zona DNS del dominio `hrinser.cl`.
2. Mapear los registros DNS para que apunten a los servidores web del hosting compartido y del DMS en Elestio:
   * `@` (raíz para la landing page) -> `IP_HOSTINGER_SHARED_HOSTING`
   * `crm.hrinser.cl` (subdominio para el CRM) -> `IP_HOSTINGER_SHARED_HOSTING`
   * `alfa.hrinser.cl` -> `IP_VPS_ELESTIO_ALFA`
   * `beta.hrinser.cl` -> `IP_VPS_ELESTIO_BETA`
3. En Cloudflare o Hostinger, habilitar el certificado SSL gratuito para el dominio principal y el subdominio `crm.hrinser.cl`, forzando la redirección automática a HTTPS.

### Paso 2.2: Despliegue de la Landing Page en Hostinger
1. Subir los archivos HTML/CSS/JS de la Landing Page al directorio raíz del dominio principal (típicamente `/public_html/`) a través del **Administrador de Archivos** de Hostinger o vinculando el repositorio git en la sección **Git** de hPanel.
2. Comprobar que el selector de idiomas (ES/EN/ZH) y el modo oscuro funcionen correctamente mediante peticiones HTTP seguras.

### Paso 2.3: Despliegue e Instalación Nativa de EspoCRM (PHP/MySQL)
Al usar el plan Hosting Compartido Unlimited de Hostinger, EspoCRM se instala de forma nativa en una carpeta/subdominio dedicado.

#### A. Configuración de Base de Datos y PHP en hPanel
1. **Crear Base de Datos MySQL:**
   * Ir a **Bases de datos -> Bases de datos MySQL** en hPanel.
   * Crear una nueva base de datos llamada `uXXXXX_espocrm` y un usuario `uXXXXX_espo_usr` con una contraseña segura de 24 caracteres. Anotar estas credenciales.
2. **Configuración de Versión PHP:**
   * Ir a **Avanzado -> Configuración de PHP**.
   * Seleccionar **PHP 8.2** (o la versión recomendada estable por EspoCRM) y verificar que las extensiones requeridas estén habilitadas: `pdo_mysql`, `gd`, `openssl`, `zip`, `mbstring`, `curl`, `exif`.
   * En la pestaña **Opciones de PHP**, subir el límite de memoria `memory_limit` a mínimo `256M` o `512M` (según permita el plan).

#### B. Subida e Instalación de Archivos
1. Crear el subdominio `crm.hrinser.cl` en la sección **Dominios -> Subdominios** de hPanel. Esto creará una carpeta en el servidor llamada `/public_html/crm/` o `/crm.hrinser.cl/`.
2. Descargar el paquete oficial de instalación de EspoCRM (.zip) desde su sitio web oficial.
3. Subir el archivo `.zip` utilizando el **Administrador de Archivos** a la carpeta del subdominio y descomprimirlo.
4. Asegurar que los permisos de los archivos y carpetas del subdominio tengan la propiedad correcta (`755` para directorios y `644` para archivos).
5. Acceder en el navegador a `https://crm.hrinser.cl/` para iniciar el instalador web:
   * Aceptar la licencia de software.
   * Ingresar los datos de conexión MySQL creados en el paso anterior (Host: `localhost` o la dirección provista por Hostinger, Base de datos: `uXXXXX_espocrm`, Usuario: `uXXXXX_espo_usr`, Contraseña).
   * Crear la cuenta del Administrador General de HRINSER (Vanessa Galdames).
   * Completar la instalación guiada.

#### C. Programación del Cron Job en Hostinger (hPanel)
EspoCRM requiere la ejecución de un cron cada 1 minuto para procesar notificaciones, correos entrantes y alertas automáticas de acreditación.
1. Ir a **Avanzado -> Tareas Cron** en el panel de Hostinger (hPanel).
2. Crear una nueva tarea cron con la siguiente configuración:
   * **Tipo de Cron:** PHP script (o comando personalizado).
   * **Comando:** `/usr/bin/php8.2 /home/uXXXXX/public_html/crm/cron.php > /dev/null 2>&1` (reemplazar `/home/uXXXXX/public_html/crm/` por la ruta absoluta de instalación y forzar el uso del binario `/usr/bin/php8.2` para evitar discrepancias de versión en la CLI).
   * **Frecuencia:** Cada minuto (`* * * * *`).
3. Guardar la tarea cron y verificar en el registro de EspoCRM (en **Administración -> Jobs**) que el cron se esté ejecutando exitosamente.

#### D. Ocultar la API de Lead Capture y Evitar Ataques DoS/Spam
Para evitar exponer la API Key y el endpoint de EspoCRM en el código JavaScript del frontend (lo que dejaría el hosting vulnerable a inundaciones de spam y caídas por CPU/MySQL), los desarrolladores deben implementar un handler proxy en PHP.

1. **Crear el script `submit_lead.php` en Hostinger:**
   Subir este archivo PHP al directorio raíz de la Landing Page:

```php
<?php
// submit_lead.php - Handler Seguro para Captura de Leads en Hostinger Shared Hosting
header('Content-Type: application/json');

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    echo json_encode(['error' => 'Method Not Allowed']);
    exit;
}

// 1. Validar Google reCAPTCHA v3 en el Backend
$recaptchaSecret = 'TU_RECAPTCHA_SECRET_KEY_AQUI'; // Reemplazar con clave secreta
$recaptchaResponse = $_POST['g-recaptcha-response'] ?? '';

if (empty($recaptchaResponse)) {
    http_response_code(400);
    echo json_encode(['error' => 'Falta validación de seguridad recaptcha.']);
    exit;
}

$verify = file_get_contents("https://www.google.com/recaptcha/api/siteverify?secret={$recaptchaSecret}&response={$recaptchaResponse}");
$responseData = json_decode($verify);

if (!$responseData->success || $responseData->score < 0.5) {
    http_response_code(403);
    echo json_encode(['error' => 'Fallo en la validación anti-bot.']);
    exit;
}

// 2. Sanitizar datos recibidos
$rut = filter_var($_POST['rutEmpresa'] ?? '', FILTER_SANITIZE_SPECIAL_CHARS);
$trabajadores = filter_var($_POST['numeroTrabajadores'] ?? 0, FILTER_VALIDATE_INT);
$mandante = filter_var($_POST['mandantePrincipal'] ?? '', FILTER_SANITIZE_SPECIAL_CHARS);
$plataforma = filter_var($_POST['plataformaAcreditacion'] ?? '', FILTER_SANITIZE_SPECIAL_CHARS);
$firstName = filter_var($_POST['firstName'] ?? '', FILTER_SANITIZE_SPECIAL_CHARS);
$lastName = filter_var($_POST['lastName'] ?? '', FILTER_SANITIZE_SPECIAL_CHARS);
$email = filter_var($_POST['email'] ?? '', FILTER_VALIDATE_EMAIL);

if (!$email) {
    http_response_code(400);
    echo json_encode(['error' => 'Email no válido.']);
    exit;
}

// 3. Reenviar Payload vía cURL seguro a la API interna de EspoCRM
$crmUrl = 'https://crm.hrinser.cl/api/v1/LeadCapture/TU_ACCESS_KEY_DE_ESPOCRM';
$payload = json_encode([
    'firstName' => $firstName,
    'lastName' => $lastName,
    'emailAddress' => $email,
    'rutEmpresa' => $rut,
    'numeroTrabajadores' => $trabajadores,
    'mandantePrincipal' => $mandante,
    'plataformaAcreditacion' => $plataforma
]);

$ch = curl_init($crmUrl);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $payload);
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    'Content-Type: application/json',
    'Content-Length: ' . strlen($payload)
]);

$response = curl_exec($ch);
$httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);

if ($httpCode === 200 || $httpCode === 201) {
    echo json_encode(['status' => 'success', 'message' => 'Lead registrado correctamente.']);
} else {
    error_log("Fallo en LeadCapture de EspoCRM: " . $response);
    http_response_code(502);
    echo json_encode(['error' => 'No se pudo conectar con el servidor CRM de destino.']);
}
?>
```

2. **Configuración de Oportunidades y Pipeline:**
   Ingresar a **Administración -> Entity Manager -> Opportunity** y configurar los campos personalizados (`rutEmpresa`, `numeroTrabajadores`, `mandantePrincipal`, `plataformaAcreditacion`) y las 10 etapas del pipeline comercial como se describió anteriormente.

---

## 3. Fase 2: Aprovisionamiento de Servidores y Orquestación Docker DMS

Por cada cliente contratista de HRINSER se debe desplegar una máquina virtual VPS dedicada en **Elestio**.

### Paso 3.1: Securización del Sistema Operativo Host (Hardening)
1. **Aprovisionamiento:** Seleccionar un VPS con mínimo 2 vCPUs dedicadas, 4 GB de RAM y almacenamiento cifrado en reposo (LUKS) mediante la consola de Elestio.
2. **Configuración de Firewall de Red:**
   * Permitir tráfico entrante solo en puertos `80/tcp` (HTTP) y `443/tcp` (HTTPS).
   * Restringir el puerto `22/tcp` (SSH) a nivel de firewall para que solo acepte conexiones desde llaves públicas SSH autorizadas de los desarrolladores.
3. **Instalación de Docker y Compose:** Asegurar que Docker Engine (v24+) y Docker Compose (v2.20+) estén actualizados en el host.

### Paso 3.2: Estructura del Proyecto en el VPS
Crear la siguiente estructura de directorios en `/opt/hrinser-dms/`:
```bash
/opt/hrinser-dms/
├── Dockerfile
├── docker-compose.yml
└── nextcloud.env
```

### Paso 3.3: Construcción de la Imagen Docker (Dockerfile)
Crear el `Dockerfile` personalizado para instalar las librerías necesarias de Tesseract OCR y soporte del idioma Español:

```dockerfile
FROM nextcloud:28-apache

# Actualizar repositorios e instalar Tesseract OCR con el paquete de idioma en español
RUN apt-get update && apt-get install -y \
    tesseract-ocr \
    tesseract-ocr-spa \
    && rm -rf /var/lib/apt/lists/*

# Asegurar permisos correctos para el usuario Apache de Nextcloud
RUN chown -R www-data:www-data /var/www/html
```

### Paso 3.4: Archivo de Orquestación Docker Compose (`docker-compose.yml`)
Este archivo define el stack con límites estrictos de CPU y memoria RAM para evitar que los picos de OCR consuman toda la memoria física del host y activen el kernel **OOM Killer** del sistema operativo. Los límites se reajustan a un total de **3.0 GB** de RAM, reservando 1 GB para el sistema operativo Host del VPS.

```yaml
version: '3.8'

services:
  # ----------------------------------------------------------------------------
  # BASE DE DATOS: PostgreSQL dedicada para la instancia del cliente
  # ----------------------------------------------------------------------------
  db:
    image: postgres:15-alpine
    container_name: hrinser-nextcloud-db
    restart: always
    stop_grace_period: 1m30s
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=nextcloud_db
      - POSTGRES_USER=nextcloud_usr
      - POSTGRES_PASSWORD=SecurePass_Postgres_32Char! # MODIFICAR EN PRODUCCIÓN
    deploy:
      resources:
        limits:
          cpus: '0.8'
          memory: 768M      # Reducido de 1024M para evitar OOM
        reservations:
          memory: 256M
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nextcloud_usr -d nextcloud_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ----------------------------------------------------------------------------
  # CACHÉ Y LOCKING: Redis para caché distribuida y file locking en memoria
  # Reduce el impacto de I/O de disco del log transaccional (WAL) en más de un 60%
  # ----------------------------------------------------------------------------
  redis:
    image: redis:7-alpine
    container_name: hrinser-nextcloud-redis
    restart: always
    command: redis-server --requirepass SecurePass_Redis_32Char! # MODIFICAR EN PRODUCCIÓN
    volumes:
      - redis_data:/data
    deploy:
      resources:
        limits:
          cpus: '0.3'
          memory: 256M      # Reducido de 512M para optimización
        reservations:
          memory: 64M
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "SecurePass_Redis_32Char!", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ----------------------------------------------------------------------------
  # APLICACIÓN: Contenedor Nextcloud App construido con Tesseract OCR
  # ----------------------------------------------------------------------------
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: hrinser-nextcloud-app
    restart: always
    expose:
      - "80"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - nextcloud_data:/var/www/html
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_DB=nextcloud_db
      - POSTGRES_USER=nextcloud_usr
      - POSTGRES_PASSWORD=SecurePass_Postgres_32Char!
      - REDIS_HOST=redis
      - REDIS_HOST_PASSWORD=SecurePass_Redis_32Char!
      - PHP_MEMORY_LIMIT=1024M
      - PHP_UPLOAD_LIMIT=10G
      - TRUSTED_PROXIES=172.18.0.0/16 172.19.0.0/16 10.0.0.0/8
      - OVERWRITEPROTOCOL=https
    deploy:
      resources:
        limits:
          cpus: '1.2'
          memory: 1536M     # Reducido de 2048M para evitar OOM
        reservations:
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/cron.php"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ----------------------------------------------------------------------------
  # CRON SERVICE: Ejecución de tareas de fondo usando el script oficial cron.sh
  # ----------------------------------------------------------------------------
  cron:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: hrinser-nextcloud-cron
    restart: always
    volumes:
      - nextcloud_data:/var/www/html
    entrypoint: /cron.sh   # Reemplaza bucles personalizados para evitar fallas de inicio
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M      # Ajustado para ejecución secuencial
        reservations:
          memory: 64M

volumes:
  db_data:
    driver: local
  redis_data:
    driver: local
  nextcloud_data:
    driver: local
```

---

## 4. Fase 3: Hardening de Nextcloud, Configuración de Cifrado y Logs de Auditoría

Una vez que los contenedores estén corriendo (`docker compose up -d`), se debe ingresar por SSH para ejecutar las siguientes tareas de configuración criptográfica y de seguridad mediante la línea de comandos `occ`.

### Paso 4.1: Activación de Cifrado del Lado del Servidor (Server-Side Encryption)
Para garantizar la confidencialidad de los exámenes de salud y liquidaciones de sueldo bajo la **Ley 21.719**:
1. Habilitar la app oficial de encriptación interna:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ app:enable encryption
   ```
2. Forzar la activación del cifrado global en el almacenamiento:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ encryption:enable
   ```
3. Activar el uso de la Master Key para posibilitar recuperaciones centralizadas ante olvido de contraseñas de usuarios:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ encryption:enable-master-key
   ```

### Paso 4.2: Enforzar Doble Factor de Autenticación (2FA) Obligatorio
Para impedir el uso de contraseñas débiles o accesos no autorizados:
1. Habilitar la app de autenticación de dos factores TOTP:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ app:enable twofactor_totp
   ```
2. Forzar que todos los usuarios deban configurar obligatoriamente el 2FA en su primer inicio de sesión:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ twofactorauth:enforce --all
   ```

### Paso 4.3: Registro de Actividades Inmutable y Redirección a Standard Output
Para evitar que un intruso con privilegios de administrador pueda borrar el log de auditoría local, se re-direcciona el flujo de logs de Nextcloud hacia el motor de logs del Host (stderr/systemd) y se configura el loglevel necesario para el módulo `admin_audit`:
1. Habilitar el módulo de auditoría administrativa:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ app:enable admin_audit
   ```
2. Inyectar los parámetros de log en el archivo `config.php`:
   ```bash
   # Configurar log_type a errorlog o systemd para enviar las trazas fuera del volumen de Docker
   docker exec -u www-data hrinser-nextcloud-app php occ config:system:set log_type --value="errorlog"
   # Ajustar nivel de logs a 1 (Info) para capturar lecturas y escrituras de archivos
   docker exec -u www-data hrinser-nextcloud-app php occ config:system:set loglevel --value=1 --type=int
   ```

### Paso 4.4: Estructura de Carpetas Estandarizada (Skeleton)
Configurar la plantilla base de Nextcloud para que, al crear la cuenta del cliente contratista, se genere la estructura obligatoria de acreditación laboral de forma automática.
1. Acceder al directorio de la plantilla base en el contenedor: `/var/www/html/core/skeleton/`.
2. Crear los siguientes directorios por defecto eliminando los archivos demostrativos de Nextcloud:
   * `/01_Contratos_y_Anexos/`
   * `/02_Liquidaciones_y_Cotizaciones/`
   * `/03_Examenes_y_Certificados_Salud/`

---

## 5. Fase 4: Ingesta de Datos, OCR y Tareas en Segundo Plano

### Paso 5.1: Ingesta Masiva de Documentos Históricos (Evitar Timeouts)
Para transferir los 100 GB históricos sin cortes por carga de navegador:
1. Transferir el bloque de archivos históricos al VPS mediante un cliente SFTP seguro usando la llave SSH.
2. Mapear o mover los archivos directamente a la ruta de datos del usuario correspondiente en el host (ej: `/var/lib/docker/volumes/hrinser-dms_nextcloud_data/_data/data/USER_CLIENTE/files/`).
3. Corregir los permisos del sistema de archivos para asegurar la propiedad de Apache:
   ```bash
   chown -R www-data:www-data /var/lib/docker/volumes/hrinser-dms_nextcloud_data/_data/data/USER_CLIENTE/files/
   ```
4. Forzar el escaneo de base de datos de Nextcloud para indexar los archivos importados:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ files:scan --all
   ```

### Paso 5.2: Configuración del Procesamiento OCR Fuera de Horario Laboral
Tesseract OCR consume dinámicamente un hilo completo de CPU al digitalizar imágenes y PDFs. Para evitar cuellos de botella durante horario de oficina:
1. En el panel de administración de Nextcloud, ir a **Ajustes de Administración -> Ajustes Básicos -> Tareas en Segundo Plano** y seleccionar la opción **Cron**.
2. Instalar y habilitar la aplicación **OCR** de la tienda de aplicaciones de Nextcloud.
3. Desactivar el procesamiento OCR automático al cargar archivos, obligándolo a encolarse:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ config:app:set files_ocr ocr_process_on_upload --value="false"
   docker exec -u www-data hrinser-nextcloud-app php occ config:app:set files_ocr ocr_process_mode --value="cron"
   ```
4. Sincronizar la zona horaria del host VPS a la hora oficial de Chile para evitar desfases con UTC:
   ```bash
   sudo timedatectl set-timezone America/Santiago
   ```
5. Configurar el programador del host VPS para ejecutar la cola de OCR en horario no hábil (madrugada):
   ```bash
   sudo crontab -e
   ```
6. Añadir la siguiente regla cron programada diariamente a las **03:00 AM (hora local chilena)**:
   ```text
   0 3 * * * docker exec -u www-data hrinser-nextcloud-app php occ ocr:process
   ```

---

## 6. Automatización de Copias de Seguridad (Backups)

El cumplimiento normativo de la Ley 21.719 exige salvaguardas de disponibilidad y cifrado para los respaldos del sistema.

### 6.1. Copias de Seguridad de EspoCRM (Hostinger Shared Hosting)
Se debe programar una rutina semanal para archivar y respaldar la base de datos de EspoCRM. Hostinger realiza copias diarias automatizadas en su plan Unlimited. Para contar con un respaldo local adicional seguro, se puede estructurar un script de exportación MySQL y descargas mediante SSH.

### 6.2. Copias de Seguridad de Nextcloud DMS (Elestio Multi-VPS)
Crear un archivo script en el Host VPS en `/opt/hrinser-dms/scripts/backup_nextcloud.sh`:

```bash
#!/bin/bash
# Backup Nextcloud DMS - Elestio VPS (Cifrado GPG + Modo Mantenimiento)
set -e

BACKUP_DIR="/var/backups/nextcloud"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/nextcloud_backup_$DATE"
APP_CONTAINER="hrinser-nextcloud-app"
DB_CONTAINER="hrinser-nextcloud-db"

mkdir -p $BACKUP_DIR

echo "=== Iniciando Respaldo DMS ==="

# 1. Forzar Modo Mantenimiento en Nextcloud
docker exec -u www-data $APP_CONTAINER php occ maintenance:mode --on

# 2. Dump de la Base de Datos PostgreSQL
docker exec $DB_CONTAINER pg_dump -U nextcloud_usr -d nextcloud_db > "$BACKUP_PATH.sql"

# 3. Comprimir dump y volumen de datos
tar -czf "$BACKUP_PATH.tar.gz" \
    -C /var/lib/docker/volumes/opt_nextcloud_data/_data . \
    -C "$BACKUP_DIR" "nextcloud_backup_$DATE.sql"

# 4. Limpiar dump temporal
rm "$BACKUP_PATH.sql"

# 5. Apagar Modo Mantenimiento
docker exec -u www-data $APP_CONTAINER php occ maintenance:mode --off

# 6. Cifrar el archivo de respaldo con clave simétrica AES-256
# IMPORTANTE: Definir la clave y aislar el respaldo de /var/www/html/data/files_encryption/ en otra bóveda externa
echo "ClaveGPG_NextcloudDMS_2026" | gpg --batch --yes --passphrase-fd 0 -c "$BACKUP_PATH.tar.gz"
rm "$BACKUP_PATH.tar.gz"

# 7. Mantener solo los últimos 14 respaldos (Rotación automática)
find $BACKUP_DIR -name "nextcloud_backup_*.tar.gz.gpg" -mtime +14 -delete

echo "=== Respaldo DMS Finalizado: $BACKUP_PATH.tar.gz.gpg ==="
```

Programar la ejecución diaria del backup a las **04:00 AM** en el crontab del Host VPS:
```text
0 4 * * * /bin/bash /opt/hrinser-dms/scripts/backup_nextcloud.sh >> /var/log/backup_nextcloud.log 2>&1
```

---

## 7. Flujo de Onboarding Técnico de un Nuevo Cliente Contratista

Cuando una oportunidad comercial se cierra como ganada y el cliente pasa a estado **"Cliente Activo"** en EspoCRM, se ejecuta el siguiente flujo técnico de aprovisionamiento:

```
[Cambio de Estado en EspoCRM]
              │
              ▼
1. Aprovisionamiento Cloud (Desarrollador DevOps)
   - Levantar VPS Dedicado de 2 vCPUs y 4 GB de RAM en Elestio.
   - Apuntar subdominio de Cloudflare (ej: cliente.hrinser.cl) a la IP.
   - Activar cifrado en reposo LUKS a nivel de disco del host.
              │
              ▼
2. Configuración y Endurecimiento DMS (Desarrollador DevOps)
   - Ejecutar docker-compose up -d.
   - Activar Cifrado Server-Side y 2FA obligatorio vía occ CLI.
   - Estructurar el skeleton de carpetas básicas de control documental.
              │
              ▼
3. Ingesta de Datos Históricos (Desarrollador DevOps)
   - Cargar los documentos previos del contratista vía SFTP.
   - Sincronizar indexación de base de datos con occ files:scan.
              │
              ▼
4. Entrega de Credenciales e Inducción (Vanessa Galdames)
   - Generar el usuario final de acceso en Nextcloud.
   - Enrolar 2FA del cliente y realizar capacitación de 30 minutos.
```

---

## 8. Plan de Verificación y Pruebas (UAT) para Desarrolladores

Antes de entregar los portales de clientes y el CRM a producción, los desarrolladores deben validar las siguientes pruebas de aceptación:

| Test Case | Objetivo | Procedimiento de Validación | Resultado Esperado |
| :--- | :--- | :--- | :--- |
| **TC-01** | Validar aislamiento de red y firewall de puertos | Desde una máquina externa, ejecutar un escaneo de puertos: `nmap -p 5432,6379,80,443 IP_VPS_CLIENTE`. | Solo los puertos 80 y 443 deben figurar como abiertos. 5432 (Postgres) y 6379 (Redis) deben estar bloqueados. |
| **TC-02** | Validar cifrado en reposo | Acceder por terminal al volumen físico del host y ejecutar `cat` en un archivo PDF subido en la carpeta del cliente. | El texto devuelto debe ser ilegible / binario encriptado (Server-Side Encryption activo). |
| **TC-03** | Validar enforzamiento de 2FA | Crear un usuario de prueba en Nextcloud e intentar iniciar sesión sin configurar 2FA. | El sistema debe bloquear el dashboard y exigir obligatoriamente el enrolamiento de 2FA. |
| **TC-04** | Validar inmutabilidad de logs | Descargar un archivo con el usuario de prueba, acceder al logs del Host (syslog/systemd) y buscar eventos de `admin_audit`. | Debe existir una entrada JSON conteniendo la IP origen real, el usuario, la acción `file_download` y el archivo exacto. |
| **TC-05** | Validar OCR en horario no hábil y zona horaria | 1. Ejecutar `timedatectl` en el host para confirmar la zona horaria `America/Santiago`. <br>2. Verificar en el log de OCR que las tareas de digitalización se encolaron y se ejecutaron a las 03:00 AM. | El texto de la imagen del PDF es seleccionable/buscable y la tarea se ejecutó en la madrugada. |
| **TC-06** | Validar Lead Capture en EspoCRM | Completar el formulario de la Landing Page y verificar en la consola de administración de EspoCRM que se cree el Lead y los datos de RUT y trabajadores se guarden. | El contacto se crea automáticamente en la base de datos de EspoCRM con los campos correspondientes. |
| **TC-07** | Validar Recuperación ante Desastres (DR) | 1. Ejecutar el script `backup_nextcloud.sh`. <br>2. Destruir la instancia y volúmenes (`docker compose down -v`). <br>3. Restaurar base de datos y archivos. | El sistema web vuelve a estar operativo con la data recuperada en menos de 20 minutos. |
| **TC-08** | Aislamiento de llaves de cifrado | Comprobar que el script de backups no almacene el directorio `/var/www/html/data/files_encryption/` en el mismo destino físico que el backup del volumen de datos principal. | Las llaves de cifrado se respaldan de manera independiente en una bóveda o servidor separado. |
| **TC-09** | Validar Rate Limiting y Google reCAPTCHA v3 | Utilizar herramientas de testeo de carga (ej: `ab` o script en bucle) para enviar 50 peticiones consecutivas al script `submit_lead.php`. | El script intermediario de Hostinger rechaza las peticiones por falta de token de reCAPTCHA o devuelve HTTP `429 Too Many Requests`. |

---
*(Fin del Documento)*
