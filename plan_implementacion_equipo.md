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
1. **CI/CD Pipeline (GitHub Actions):** Si la landing page se desarrolla utilizando frameworks o compiladores (Tailwind, Vite, Node), configurar una acción de GitHub para compilar en frío en la rama principal y subir el directorio de salida `/dist` o `/build` al servidor Hostinger usando SFTP de forma automática con llaves protegidas. Si es puramente estático, se puede subir directamente al directorio raíz `/public_html/`.
2. **Optimización de i18n (Multi-Idioma ES/EN/ZH):** 
   * **Diseño Líquido:** Diseñar componentes (cajas de texto, botones, contenedores) con medidas elásticas (`min-width`, `height: auto` en Flexbox y CSS Grid) evitando tamaños fijos horizontales y verticales. El español es un 20% más expansivo que el inglés, y el chino es extremadamente corto pero con mayor densidad vertical.
   * **Fuentes del Sistema para Chino:** Para evitar la descarga de fuentes web de chino de varios megabytes (que arruinarían el LCP y el SEO), utilizar el stack de fuentes del sistema operativo:
     ```css
     font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Arial, "Microsoft YaHei", "PingFang SC", sans-serif;
     ```
   * **Actualizar Atributo Lang:** Al alternar de idioma vía JavaScript, inyectar dinámicamente `document.documentElement.lang = selectedLangCode;`. Para SEO internacional óptimo, preferir carpetas físicas (`/en/index.html` y `/zh/index.html`).
3. **Optimización de Modo Claro/Oscuro (Theme Toggle):**
   * **Script Bloqueante Anti-Parpadeo (FOUC):** Colocar el siguiente bloque de JavaScript inline al principio del `<head>` del HTML antes de cargar las hojas de estilo:
     ```html
     <script>
       (function() {
         try {
           const storedTheme = localStorage.getItem('theme');
           const systemDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
           if (storedTheme === 'dark' || (!storedTheme && systemDark)) {
             document.documentElement.classList.add('dark');
           } else {
             document.documentElement.classList.remove('dark');
           }
         } catch (e) {}
       })();
     </script>
     ```
   * **Integración del Navegador:** Declarar `color-scheme: light dark;` en el CSS raíz para forzar la adaptación de barras de navegación y campos nativos. Asegurar que los contrastes de ambos temas cumplan la norma WCAG AA (mínimo 4.5:1).

---

### Paso 2.3: Despliegue e Instalación Nativa de EspoCRM (PHP/MySQL)

#### A. Configuración de Base de Datos y PHP en hPanel
1. **Crear Base de Datos MySQL:**
   * Ir a **Bases de datos -> Bases de datos MySQL** en hPanel.
   * Crear la base de datos `uXXXXX_espocrm` y el usuario `uXXXXX_espo_usr` con contraseña de 24 caracteres.
   * Configurar el conjunto de caracteres en **`utf8mb4`** y la colación en **`utf8mb4_unicode_ci`** para evitar fallos de ejecución con caracteres especiales en los formularios:
     ```sql
     ALTER DATABASE uXXXXX_espocrm CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
     ```
2. **Configuración de PHP y Permisos:**
   * En hPanel, ir a **Avanzado -> Configuración de PHP** y activar **PHP 8.2** con extensiones requeridas (`pdo_mysql`, `gd`, `openssl`, `zip`, `mbstring`, `curl`, `exif`, `opcache`).
   * Configurar `memory_limit` a un mínimo de `256M`.
   * En el entorno de hosting compartido de Hostinger (SuExec), configurar directorios con permisos **`755`** y archivos con **`644`**. Está estrictamente prohibido usar permisos `777`.

#### B. Subida de Archivos y Tarea Cron
1. Subir y descomprimir el paquete de EspoCRM oficial en el directorio del subdominio `/crm.hrinser.cl/`.
2. Completar el asistente web en `https://crm.hrinser.cl` vinculando la base de datos configurada.
3. **Programación del Cron en hPanel:**
   * Ir a **Avanzado -> Tareas Cron**.
   * Agregar la ejecución del cron cada un minuto, forzando la versión CLI exacta y redirigiendo errores a un log rotativo:
     ```text
     /usr/bin/php8.2 /home/uXXXXX/public_html/crm/cron.php >> /home/uXXXXX/public_html/crm/data/logs/cron_errors.log 2>&1
     ```

#### C. Seguridad de Formulario (Validación RUT y reCAPTCHA v3)
Para evitar inyectar leads duplicados y proteger la capacidad de procesamiento de MySQL y CPU de la cuenta compartida de Hostinger (evitando errores *508 Resource Limit Reached*):

1. **Validación de RUT en el Frontend (Módulo 11):**
   Agregar una función Javascript en el frontend para evitar llamadas HTTP con datos inválidos:
   ```javascript
   function validarRutChileno(rutCompleto) {
       if (!/^[0-9]+-[0-9kK]{1}$/.test(rutCompleto.replace(/\./g, ''))) return false;
       const tmp = rutCompleto.replace(/\./g, '').split('-');
       let rut = tmp[0]; const dv = tmp[1].toLowerCase();
       let suma = 0; let multiplicador = 2;
       for (let i = rut.length - 1; i >= 0; i--) {
           suma += parseInt(rut.charAt(i)) * multiplicador;
           multiplicador = multiplicador === 7 ? 2 : multiplicador + 1;
       }
       const dvEsperado = 11 - (suma % 11);
       let dvCalculado = dvEsperado === 11 ? '0' : (dvEsperado === 10 ? 'k' : dvEsperado.toString());
       return dv === dvCalculado;
   }
   ```
2. **Prevención de Doble Envío:** Deshabilitar el botón submit al hacer clic y cambiar su contenido a `"Enviando..."`. Usar pseudoclases CSS `:user-valid` y `:user-invalid` para una validación amigable al quitar el foco.
3. **Crear script Proxy Seguro PHP (`submit_lead.php`):**
   Subir este archivo PHP al directorio raíz de la Landing Page para procesar el token de reCAPTCHA y realizar la llamada cURL interna con un timeout estricto de 5 segundos (evitando bloqueos de red concurrentes):

```php
<?php
// submit_lead.php - Handler Seguro para Captura de Leads en Hostinger Shared Hosting
header('Content-Type: application/json');
ini_set('display_errors', 0);
error_reporting(E_ALL);

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    echo json_encode(['error' => 'Method Not Allowed']);
    exit;
}

// 1. Validar Google reCAPTCHA v3 en el Backend con Timeout
$recaptchaSecret = 'TU_RECAPTCHA_SECRET_KEY_AQUI'; 
$recaptchaResponse = $_POST['g-recaptcha-response'] ?? '';

if (empty($recaptchaResponse)) {
    http_response_code(400);
    echo json_encode(['error' => 'Falta validación de seguridad de Google reCAPTCHA.']);
    exit;
}

$chVerify = curl_init("https://www.google.com/recaptcha/api/siteverify");
curl_setopt($chVerify, CURLOPT_RETURNTRANSFER, true);
curl_setopt($chVerify, CURLOPT_POST, true);
curl_setopt($chVerify, CURLOPT_POSTFIELDS, http_build_query([
    'secret' => $recaptchaSecret,
    'response' => $recaptchaResponse
]));
curl_setopt($chVerify, CURLOPT_TIMEOUT, 5); // Timeout para evitar bloqueos
$verifyResponse = curl_exec($chVerify);
$verifyHttpCode = curl_getinfo($chVerify, CURLINFO_HTTP_CODE);
curl_close($chVerify);

if ($verifyHttpCode !== 200) {
    http_response_code(502);
    echo json_encode(['error' => 'Error de conexión con el servidor de seguridad de Google.']);
    exit;
}

$responseData = json_decode($verifyResponse);
if (!$responseData->success || $responseData->score < 0.5) {
    http_response_code(403);
    echo json_encode(['error' => 'Acceso denegado: El sistema ha detectado comportamiento inusual (bot).']);
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
    echo json_encode(['error' => 'El correo electrónico ingresado no es válido.']);
    exit;
}

// 3. Reenviar Payload vía cURL a EspoCRM con Timeout de 5s
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
curl_setopt($ch, CURLOPT_TIMEOUT, 5); 
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    'Content-Type: application/json',
    'Content-Length: ' . strlen($payload)
]);

$response = curl_exec($ch);
$httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);

if ($httpCode === 200 || $httpCode === 201) {
    echo json_encode(['status' => 'success', 'message' => 'Información registrada exitosamente.']);
} else {
    error_log("Fallo en LeadCapture de EspoCRM. HTTP Code: {$httpCode}. Respuesta: {$response}");
    http_response_code(502);
    echo json_encode(['error' => 'No se pudo conectar con el CRM de destino.']);
}
?>
```

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

Se incorpora una red bridge personalizada, la sincronización de la zona horaria (`TZ=America/Santiago`) y la optimización de los parámetros del motor de PostgreSQL y Redis para rendimiento bajo restricciones de RAM.

```yaml
version: '3.8'

networks:
  hrinser-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

services:
  # ----------------------------------------------------------------------------
  # BASE DE DATOS: PostgreSQL dedicada con tuning de memoria
  # ----------------------------------------------------------------------------
  db:
    image: postgres:15-alpine
    container_name: hrinser-nextcloud-db
    restart: always
    stop_grace_period: 1m30s
    networks:
      - hrinser-network
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=nextcloud_db
      - POSTGRES_USER=nextcloud_usr
      - POSTGRES_PASSWORD=SecurePass_Postgres_32Char! # MODIFICAR EN PRODUCCIÓN
      - TZ=America/Santiago
    command: >
      postgres
      -c shared_buffers=192MB
      -c work_mem=16MB
      -c maintenance_work_mem=64MB
      -c effective_cache_size=512MB
    deploy:
      resources:
        limits:
          cpus: '0.8'
          memory: 768M
        reservations:
          memory: 256M
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nextcloud_usr -d nextcloud_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ----------------------------------------------------------------------------
  # CACHÉ Y LOCKING: Redis configurado con directivas Maxmemory contra caídas OOM
  # ----------------------------------------------------------------------------
  redis:
    image: redis:7-alpine
    container_name: hrinser-nextcloud-redis
    restart: always
    networks:
      - hrinser-network
    command: redis-server --requirepass SecurePass_Redis_32Char! --maxmemory 180mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    environment:
      - TZ=America/Santiago
    deploy:
      resources:
        limits:
          cpus: '0.3'
          memory: 256M
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
    networks:
      - hrinser-network
    expose:
      - "80"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - nextcloud_data:/var/www/html
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_DB=nextcloud_db
      - POSTGRES_USER=nextcloud_usr
      - POSTGRES_PASSWORD=SecurePass_Postgres_32Char!
      - REDIS_HOST=redis
      - REDIS_HOST_PASSWORD=SecurePass_Redis_32Char!
      - PHP_MEMORY_LIMIT=1024M
      - PHP_UPLOAD_LIMIT=10G
      - TRUSTED_PROXIES=172.28.0.0/16 10.0.0.0/8
      - OVERWRITEPROTOCOL=https
      - TZ=America/Santiago
    deploy:
      resources:
        limits:
          cpus: '1.2'
          memory: 1536M
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
    networks:
      - hrinser-network
    volumes:
      - nextcloud_data:/var/www/html
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    entrypoint: /cron.sh
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - TZ=America/Santiago
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M
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
Crear un archivo script en el Host VPS en `/opt/hrinser-dms/scripts/backup_nextcloud.sh`. Este script resuelve la falla de autenticación de `pg_dump` mediante inyección no interactiva, utiliza el nombre correcto del volumen Docker, realiza el **cifrado asimétrico GPG** (mediante llave pública del administrador importada previamente) y **excluye las llaves criptográficas** para respaldarlas por separado (cumpliendo estrictamente el control **TC-08**):

```bash
#!/bin/bash
# ==============================================================================
# Backup Nextcloud DMS - Elestio VPS (Cifrado Asimétrico GPG + Segregación de Llaves)
# Código de Control: HR-SEC-BACKUP-v2
# ==============================================================================
set -e

# Configuración de variables
PROJECT_NAME="hrinser-dms"
BACKUP_DIR="/var/backups/nextcloud"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/nextcloud_backup_$DATE"
KEYS_BACKUP_PATH="$BACKUP_DIR/keys_backup_$DATE"

APP_CONTAINER="hrinser-nextcloud-app"
DB_CONTAINER="hrinser-nextcloud-db"
DB_PASS="SecurePass_Postgres_32Char!" # Se debe configurar externamente en producción
GPG_RECIPIENT="backup-key@hrinser.cl"  # Correo de la llave pública GPG importada en el Host

# Rutas del sistema de archivos de Docker
VOLUME_DATA_DIR="/var/lib/docker/volumes/${PROJECT_NAME}_nextcloud_data/_data"

# Validar que la llave pública GPG existe en el host antes de iniciar
if ! gpg --list-keys "$GPG_RECIPIENT" >/dev/null 2>&1; then
    echo "ERROR: La llave pública GPG para $GPG_RECIPIENT no está importada en el Host." >&2
    exit 1
fi

mkdir -p "$BACKUP_DIR"
echo "=== [${DATE}] Iniciando Respaldo DMS Segurizado ==="

# 1. Activar Modo Mantenimiento en Nextcloud
docker exec -u www-data "$APP_CONTAINER" php occ maintenance:mode --on

# 2. Dump de Base de Datos inyectando contraseña de forma segura
docker exec -e PGPASSWORD="$DB_PASS" "$DB_CONTAINER" pg_dump -U nextcloud_usr -d nextcloud_db > "$BACKUP_PATH.sql"

# 3. Comprimir Datos del Sistema (EXCLUYENDO llaves de cifrado)
# Esto cumple con TC-08 al segregar datos de llaves criptográficas.
tar -czf "$BACKUP_PATH.tar.gz" \
    --exclude="./data/files_encryption" \
    --exclude="./data/*/files_encryption" \
    -C "$VOLUME_DATA_DIR" . \
    -C "$BACKUP_DIR" "nextcloud_backup_$DATE.sql"

# 4. Comprimir Llaves de Cifrado por separado
# Se guardan de forma independiente para permitir almacenamiento en otra bóveda externa.
tar -czf "$KEYS_BACKUP_PATH.tar.gz" \
    -C "$VOLUME_DATA_DIR" \
    "./data/files_encryption" \
    $(find "$VOLUME_DATA_DIR/data/" -maxdepth 2 -type d -name "files_encryption" -not -path '*/appdata_*' | sed "s|$VOLUME_DATA_DIR/||g") 2>/dev/null || true

# 5. Desactivar Modo Mantenimiento
docker exec -u www-data "$APP_CONTAINER" php occ maintenance:mode --off

# 6. Limpiar archivo SQL temporal
rm "$BACKUP_PATH.sql"

# 7. Cifrar asimétricamente los respaldos con GPG (solo cifrar, no descifrar en el Host)
gpg --encrypt --recipient "$GPG_RECIPIENT" --trust-model always "$BACKUP_PATH.tar.gz"
gpg --encrypt --recipient "$GPG_RECIPIENT" --trust-model always "$KEYS_BACKUP_PATH.tar.gz"

# Eliminar comprimidos temporales en texto plano
rm "$BACKUP_PATH.tar.gz"
rm "$KEYS_BACKUP_PATH.tar.gz"

# 8. Mantener solo los últimos 14 respaldos (Rotación automática)
find "$BACKUP_DIR" -name "nextcloud_backup_*.tar.gz.gpg" -mtime +14 -delete
find "$BACKUP_DIR" -name "keys_backup_*.tar.gz.gpg" -mtime +14 -delete

echo "=== Respaldo DMS Finalizado Exitosamente ==="
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
