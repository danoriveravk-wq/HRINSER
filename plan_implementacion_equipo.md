# 🛠️ Guía de Implementación y Despliegue Técnico: Ecosistema DMS & CRM HRINSER
### Manual de Ejecución Paso a Paso para Desarrolladores (Cumplimiento Ley 21.719 - Chile)
**Código del Documento:** HR-DEV-2026-001  
**Preparado por:** Lead DevOps & Systems Architect  
**Destinatarios:** Equipo de Desarrollo, DevOps e Ingenieros de Software  
**Fecha:** 13 de Julio de 2026  

---

## 1. Distribución de Infraestructura y Resumen Arquitectónico

Para evitar confusiones en el desarrollo y despliegue, se establece explícitamente la distribución física de cada componente del ecosistema digital de **HRINSER** (Gestión Comercial SpA):

| Componente | Tipo de Aplicación | Proveedor de Hosting / Servidor | Razón Técnica / Operativa |
| :--- | :--- | :--- | :--- |
| **Landing Page** | Frontend estático (HTML/CSS/JS) | **Hostinger Shared Web Hosting** (Plan Unlimited) | Consumo mínimo de recursos, bajo costo, despliegue simple. |
| **EspoCRM** | Aplicación PHP Nativa + DB MariaDB | **Hostinger Shared Web Hosting** (Plan Unlimited) | Instalación directa sin contenedores, aprovechando la base de datos MySQL y el soporte PHP nativo contratado para ahorrar costos. |
| **DMS (Nextcloud)** | Stack Docker (App, Postgres, Redis, Cron) | **Elestio Multi-VPS** (Un VPS de 4GB por Cliente) | Requisitos de CPU para procesamiento de OCR (Tesseract) en PDF e inmutabilidad de almacenamiento bajo la **Ley 21.719**. |
| **DNS Manager** | Gestión y resolución de nombres | **Hostinger DNS Zone Editor** (Incluido en el Hosting) | Centraliza la resolución de nombres del dominio `hrinser.cl` y subdominios en Hostinger sin requerir proveedores externos adicionales. |

```
[Cliente Web] ---> [DNS Hostinger (Gestor de Nombres)]
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

> [!IMPORTANT]
> **Certificados SSL en Hostinger y Elestio:**
> Al eliminar proveedores proxy intermediarios, la seguridad de las conexiones SSL/TLS se gestiona nativamente en el punto final. Hostinger provee e instala certificados SSL gratuitos de por vida (Let's Encrypt / ZeroSSL) para la Landing Page y el subdominio del CRM. En el caso de los VPS de Elestio, la propia plataforma de Elestio aprovisiona automáticamente el certificado SSL mediante su proxy reverso interno integrado al apuntar los registros DNS.

---

## 2. Fase 1: Configuración de DNS, Landing Page y Despliegue de EspoCRM en Hostinger (Shared Hosting)

### Paso 2.1: Gestión de DNS en Hostinger
1. Iniciar sesión en el hPanel de Hostinger y acceder a **Avanzado -> Editor de Zona DNS** para el dominio `hrinser.cl`.
2. Configurar la tabla de registros DNS para direccionar el tráfico web de la siguiente manera:
   * **Registro A (raíz):** Apuntar `@` a la dirección IP del servidor web de Hostinger Shared Hosting (ej: `IP_HOSTINGER_SERVER`).
   * **Registro A (CRM):** Crear `crm` apuntando a `IP_HOSTINGER_SERVER`.
   * **Registro A (Cliente Alfa DMS):** Crear `alfa` apuntando a la IP pública del VPS aprovisionado en Elestio (ej: `IP_VPS_ELESTIO_ALFA`).
   * **Registro A (Cliente Beta DMS):** Crear `beta` apuntando a la IP pública del segundo VPS de Elestio (ej: `IP_VPS_ELESTIO_BETA`).
3. Activar el SSL gratuito para el dominio principal y el subdominio `crm.hrinser.cl` desde la sección **Seguridad -> SSL** en el hPanel de Hostinger.

### Paso 2.2: Despliegue de la Landing Page en Hostinger y Ley 21.719
1. **CI/CD Pipeline (GitHub Actions):** Si la landing page requiere una compilación previa (como empaquetadores de Tailwind o Vite), utilizar la plantilla de GitHub Actions `.github/workflows/deploy.yml` para compilar y despliegue vía SFTP de manera automática:

```yaml
name: Deploy Landing Page to Hostinger

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Build Project
        run: npm run build

      - name: Deploy to Hostinger via SFTP
        uses: SamKirkland/FTP-Deploy-Action@v4.3.4
        with:
          server: ${{ secrets.SFTP_SERVER }}
          username: ${{ secrets.SFTP_USERNAME }}
          password: ${{ secrets.SFTP_PASSWORD }}
          local-dir: ./dist/ # O la carpeta de build correspondiente
          server-dir: /domains/hrinser.cl/public_html/
```
*(Nota: Configurar `SFTP_SERVER`, `SFTP_USERNAME` y `SFTP_PASSWORD` en los Secretos del repositorio GitHub).*

2. **Diseño de Formulario con Consentimiento Explícito (Ley 21.719):**
   * El formulario de la Landing Page debe incluir un **checkbox obligatorio (no pre-marcado)** de aceptación de tratamiento de datos personales:
     ```html
     <input type="checkbox" id="consentimiento" name="consentimiento" required>
     <label for="consentimiento">Acepto el tratamiento de mis datos personales según la <a href="/politica-privacidad.html">Política de Privacidad</a> de HRINSER.</label>
     ```
   * Enlazar el documento `/politica-privacidad.html` indicando expresamente los derechos ARCO de los usuarios y que la información recopilada se utilizará exclusivamente para el proceso de onboarding comercial.

3. **Optimización de i18n (Multi-Idioma ES/EN/ZH):**
   * **Diseño Líquido:** Diseñar componentes (cajas de texto, botones, contenedores) con medidas elásticas (`min-width`, `height: auto` en Flexbox y CSS Grid) evitando tamaños fijos horizontales y verticales. El español es un 20% más expansivo que el inglés, y el chino es extremadamente corto pero con mayor densidad vertical.
   * **Fuentes del Sistema para Chino:** Para evitar la descarga de fuentes de chino de varios megabytes, utilizar el stack de fuentes del sistema operativo:
     ```css
     font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Arial, "Microsoft YaHei", "PingFang SC", sans-serif;
     ```
   * **Actualizar Atributo Lang:** Al alternar de idioma vía JavaScript, inyectar dinámicamente `document.documentElement.lang = selectedLangCode;`. Para SEO internacional óptimo, preferir carpetas físicas (`/en/index.html` y `/zh/index.html`) y configurar reglas de redirección en el archivo `.htaccess`.
4. **Optimización de Modo Claro/Oscuro (Theme Toggle):**
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
   * **Nota de Advertencia:** En Hostinger Shared Hosting, el host de la base de datos a menudo **no es localhost**, sino una dirección específica del servidor MySQL (ej: `sql123.main-hosting.eu`). Copiar la dirección exacta del host desde hPanel.
   * Configurar el conjunto de caracteres en **`utf8mb4`** y la colación en **`utf8mb4_unicode_ci`** para evitar fallos de ejecución con caracteres especiales en los formularios:
     ```sql
     ALTER DATABASE uXXXXX_espocrm CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
     ```
2. **Configuración de PHP y Permisos:**
   * En hPanel, ir a **Avanzado -> Configuración de PHP** y activar **PHP 8.2** con extensiones requeridas (`pdo_mysql`, `gd`, `openssl`, `zip`, `mbstring`, `curl`, `exif`, `opcache`).
   * Configurar `memory_limit` a un mínimo de `256M`.
   * En el entorno de hosting compartido de Hostinger (SuExec), configurar directorios con permisos **`755`** y archivos con **`644`**. Está estrictamente prohibido usar permisos `777`.

#### B. Subida de Archivos y Tarea Cron
1. Crear el subdominio `crm.hrinser.cl`. En Hostinger, esto crea la carpeta física en `/home/uXXXXX/domains/hrinser.cl/public_html/crm/` o en `/home/uXXXXX/crm.hrinser.cl/`.
2. Subir y descomprimir el paquete de EspoCRM oficial en el directorio asignado al subdominio.
3. Completar el asistente web en `https://crm.hrinser.cl` vinculando la base de datos configurada.
4. **Programación del Cron en hPanel:**
   * Ir a **Avanzado -> Tareas Cron**.
   * Agregar la ejecución del cron cada un minuto, utilizando la ruta absoluta correcta de Hostinger y redirigiendo errores a un archivo de bitácora:
     ```text
     /usr/bin/php8.2 /home/uXXXXX/domains/hrinser.cl/public_html/crm/cron.php >> /home/uXXXXX/domains/hrinser.cl/public_html/crm/data/logs/cron_errors.log 2>&1
     ```

#### C. Envío de Correos (SMTP Corporativo)
* **Requisito:** Para evitar que las notificaciones de leads del CRM y los emails de bienvenida caigan en SPAM o sean bloqueados por el hosting, ir a **Administración -> Outbound Emails** en EspoCRM y configurar un servidor **SMTP Externo** utilizando la casilla de correos corporativas creada en Hostinger (ej: `contacto@hrinser.cl`) con autenticación SSL/TLS y puerto `465` o `587`.

#### D. Seguridad de Formulario (Validación RUT y reCAPTCHA v3)
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
3. **Configuración Externa de Credenciales (`config.php`):**
   Para evitar subir las claves de API a repositorios públicos, crear el archivo `config.php` en la carpeta de la landing page para guardar los secretos del backend:
   ```php
   <?php
   // config.php - Secretos Protegidos en el Servidor
   define('RECAPTCHA_SECRET_KEY', 'TU_RECAPTCHA_SECRET_KEY_AQUI');
   define('ESPOCRM_ACCESS_KEY', 'TU_ACCESS_KEY_DE_ESPOCRM');
   ?>
   ```
4. **Crear script Proxy Seguro PHP (`submit_lead.php`):**
   Este archivo incluye control de tasa local (**Rate Limiting** de 5 peticiones por minuto por IP) almacenando los logs en un directorio privado seguro dentro de la cuenta (evitando el directorio compartido `/tmp` de Hostinger) e implementa bloqueos de archivos exclusivas (`flock`) contra condiciones de carrera, además de duplicar la validación matemática del RUT en el backend:

```php
<?php
// submit_lead.php - Handler Seguro para Captura de Leads en Hostinger Shared Hosting
header('Content-Type: application/json');
ini_set('display_errors', 0);
error_reporting(E_ALL);

require_once 'config.php';

// Validar RUT en Backend
function validarRutBackend($rut) {
    $rut = preg_replace('/[^kK0-9]/', '', $rut);
    if (strlen($rut) < 2) return false;
    $dv = substr($rut, -1);
    $numero = substr($rut, 0, -1);
    $suma = 0; $factor = 2;
    for ($i = strlen($numero) - 1; $i >= 0; $i--) {
        $suma += $numero[$i] * $factor;
        $factor = $factor === 7 ? 2 : $factor + 1;
    }
    $dvEsperado = 11 - ($suma % 11);
    $dvCalculado = $dvEsperado === 11 ? '0' : ($dvEsperado === 10 ? 'k' : (string)$dvEsperado);
    return strtolower($dv) === $dvCalculado;
}

// A. Control de Tasa (Rate Limiting) con bloqueo flock en directorio privado
$ip = $_SERVER['REMOTE_ADDR'] ?? 'unknown';
$limitDir = '/home/uXXXXX/rate_limits'; // Cambiar a la ruta absoluta correcta en Hostinger
if (!file_exists($limitDir)) {
    mkdir($limitDir, 0700, true);
}
$limitFile = $limitDir . '/rate_limit_' . md5($ip) . '.json';
$currentTime = time();
$window = 60; // 1 minuto
$limit = 5;

// Abrir o crear archivo con bloqueo flock para evitar condiciones de carrera
$fp = fopen($limitFile, 'c+');
if ($fp) {
    flock($fp, LOCK_EX);
    $fileSize = filesize($limitFile);
    $content = $fileSize > 0 ? fread($fp, $fileSize) : '';
    $data = json_decode($content, true) ?: ['requests' => []];

    // Filtrar solicitudes antiguas
    $data['requests'] = array_filter($data['requests'], function($ts) use ($currentTime, $window) {
        return $ts > ($currentTime - $window);
    });

    if (count($data['requests']) >= $limit) {
        flock($fp, LOCK_UN);
        fclose($fp);
        http_response_code(429);
        header('Retry-After: ' . $window);
        echo json_encode(['error' => 'Demasiadas solicitudes. Por favor intente más tarde.']);
        exit;
    }

    $data['requests'][] = $currentTime;
    ftruncate($fp, 0);
    rewind($fp);
    fwrite($fp, json_encode($data));
    fflush($fp);
    flock($fp, LOCK_UN);
    fclose($fp);
}

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    echo json_encode(['error' => 'Method Not Allowed']);
    exit;
}

// B. Validar RUT
$rut = $_POST['rutEmpresa'] ?? '';
if (!validarRutBackend($rut)) {
    http_response_code(400);
    echo json_encode(['error' => 'El RUT de empresa ingresado es inválido.']);
    exit;
}

// C. Validar Google reCAPTCHA v3 en el Backend con Timeout
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
    'secret' => RECAPTCHA_SECRET_KEY,
    'response' => $recaptchaResponse
]));
curl_setopt($chVerify, CURLOPT_TIMEOUT, 5); 
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

// D. Sanitizar datos restantes
$rutSanitized = filter_var($rut, FILTER_SANITIZE_SPECIAL_CHARS);
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

// E. Reenviar Payload vía cURL a EspoCRM con Timeout de 5s
$crmUrl = 'https://crm.hrinser.cl/api/v1/LeadCapture/' . ESPOCRM_ACCESS_KEY;
$payload = json_encode([
    'firstName' => $firstName,
    'lastName' => $lastName,
    'emailAddress' => $email,
    'rutEmpresa' => $rutSanitized,
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

## 3. Fase 2: Aprovisionamiento de Servidores y Orquestación Docker DMS (Elestio)

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

Configurar el archivo `/opt/hrinser-dms/nextcloud.env` para mantener las contraseñas e identificadores aislados del control de versiones:
```ini
POSTGRES_DB=nextcloud_db
POSTGRES_USER=nextcloud_usr
POSTGRES_PASSWORD=SecurePass_Postgres_32Char! # MODIFICAR EN PRODUCCIÓN
REDIS_PASSWORD=SecurePass_Redis_32Char!       # MODIFICAR EN PRODUCCIÓN
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
Este archivo define el stack con límites de memoria de un total de **3.0 GB** de RAM, reservando 1 GB para el sistema operativo Host del VPS. Utiliza variables de entorno cargadas dinámicamente de `nextcloud.env` para evitar credenciales hardcoded.

```yaml
version: '3.8'

services:
  # ----------------------------------------------------------------------------
  # BASE DE DATOS: PostgreSQL dedicada con tuning de memoria y variables .env
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
    env_file:
      - nextcloud.env
    environment:
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
  # CACHÉ Y LOCKING: Redis configurado con directivas Maxmemory y variables .env
  # ----------------------------------------------------------------------------
  redis:
    image: redis:7-alpine
    container_name: hrinser-nextcloud-redis
    restart: always
    networks:
      - hrinser-network
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 180mb --maxmemory-policy allkeys-lru
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
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
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
    env_file:
      - nextcloud.env
    environment:
      - POSTGRES_HOST=db
      - REDIS_HOST=redis
      - REDIS_HOST_PASSWORD=${REDIS_PASSWORD}
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
    env_file:
      - nextcloud.env
    environment:
      - TZ=America/Santiago
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 256M
        reservations:
          memory: 64M

networks:
  hrinser-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

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

### Paso 4.2: Enforzar Doble Factor de Autenticación (2FA), Dominios de Confianza y ARCO
1. Habilitar la app de autenticación de dos factores TOTP:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ app:enable twofactor_totp
   ```
2. Forzar que todos los usuarios deban configurar obligatoriamente el 2FA en su primer inicio de sesión:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ twofactorauth:enforce --all
   ```
3. **Registrar subdominio del cliente en Trusted Domains:**
   Nextcloud rechaza conexiones que no estén en su lista blanca. Para evitar el bloqueo del subdominio del cliente, registrarlo de forma explícita:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ config:system:set trusted_domains 2 --value="alfa.hrinser.cl"
   ```
4. **Procedimientos para el Ejercicio de Derechos ARCO (Ley N° 21.719):**
   * Los administradores de Nextcloud deben canalizar las solicitudes de Acceso, Rectificación, Cancelación y Oposición del contratista a través de un ticket de soporte.
   * El borrado definitivo de datos personales y sensibles (Derecho de Cancelación) se debe ejecutar de forma segura mediante los comandos `files:cleanup` y el purgado de la papelera del usuario respectivo:
     ```bash
     docker exec -u www-data hrinser-nextcloud-app php occ trashbin:cleanup --all-users
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

### Paso 4.4: Envío de Correos (SMTP de Nextcloud)
* **Requisito:** Configurar las notificaciones transaccionales y de recuperación del DMS utilizando un servidor SMTP externo. Ejecutar el asistente de configuración en la GUI en **Ajustes de Administración -> Ajustes Básicos -> Servidor de Correo** o inyectando los parámetros en `config.php`:
  ```bash
  docker exec -u www-data hrinser-nextcloud-app php occ config:system:set mail_smtpmode --value="smtp"
  docker exec -u www-data hrinser-nextcloud-app php occ config:system:set mail_smtphost --value="smtp.hostinger.com"
  docker exec -u www-data hrinser-nextcloud-app php occ config:system:set mail_smtpport --value="465"
  docker exec -u www-data hrinser-nextcloud-app php occ config:system:set mail_smtpsecure --value="ssl"
  docker exec -u www-data hrinser-nextcloud-app php occ config:system:set mail_smtpauth --value=true --type=boolean
  docker exec -u www-data hrinser-nextcloud-app php occ config:system:set mail_smtpname --value="notificaciones@hrinser.cl"
  docker exec -u www-data hrinser-nextcloud-app php occ config:system:set mail_smtppassword --value="TU_PASSWORD_SMTP"
  ```

### Paso 4.5: Estructura de Carpetas Estandarizada (Skeleton) y Previsualización
1. Acceder al directorio de la plantilla base en el contenedor: `/var/www/html/core/skeleton/`.
2. Crear los siguientes directorios por defecto eliminando los archivos demostrativos de Nextcloud:
   * `/01_Contratos_y_Anexos/`
   * `/02_Liquidaciones_y_Cotizaciones/`
   * `/03_Examenes_y_Certificados_Salud/`
3. **Previsualizador de PDF e Imágenes:** Nextcloud previsualiza nativamente imágenes y archivos PDF. Para visualizar o editar documentos Office (.docx, .xlsx, .pptx) sin necesidad de descargarlos, los desarrolladores deberán instalar la app **ONLYOFFICE** en Nextcloud y configurar el conector a un servidor de documentos ONLYOFFICE independiente en la Fase 3 del Hito 3.

---

## 5. Fase 4: Ingesta de Datos, OCR y Tareas en Segundo Plano

### Paso 5.1: Ingesta Masiva de Documentos Históricos (Bypass de Cifrado Resuelto)
> [!IMPORTANT]
> **Advertencia de Seguridad:** Si los archivos históricos se copian directamente en el sistema de archivos del host mediante SFTP, Nextcloud los indexará pero **quedarán en texto plano (sin cifrar)** en el almacenamiento físico, violando el control TC-02. Para forzar su encriptación, los desarrolladores deben utilizar obligatoriamente una de las dos siguientes rutas de ingesta:

*   **Ruta Recomendada (WebDAV/API):** Sincronizar los archivos del cliente utilizando un cliente automatizado WebDAV (por ejemplo, configurando `rclone` local con las credenciales del cliente contratista y apuntando a `https://cliente.hrinser.cl/remote.php/dav/files/USERNAME/`). Esto obliga a Nextcloud a cifrar los archivos en caliente en el momento de la ingesta.
*   **Ruta de Contingencia (Copia en Caliente + Cifrado Masivo):** 
    1. Transferir archivos históricos vía SFTP seguro al volumen Docker en el host.
    2. Corregir permisos en el volumen físico:
       ```bash
       chown -R www-data:www-data /var/lib/docker/volumes/hrinser-dms_nextcloud_data/_data/data/USER_CLIENTE/files/
       ```
    3. Forzar escaneo de la base de datos:
       ```bash
       docker exec -u www-data hrinser-nextcloud-app php occ files:scan --all
       ```
    4. **EJECUTAR CIFRADO EN BLOQUE:** Correr el proceso de encriptación masiva de archivos huérfanos:
       ```bash
       docker exec -u www-data hrinser-nextcloud-app php occ encryption:encrypt-all
       ```
       *(Nota: Este comando es intensivo en CPU/Disco y debe ejecutarse antes de que los usuarios comiencen a operar en el portal).*

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

### 6.1. Copias de Seguridad de EspoCRM (Hostinger Shared Hosting)
Para contar con copias locales e independientes adicionales a las de Hostinger, los desarrolladores deben configurar un Cron Job semanal de tipo "Custom Command" en el hPanel de Hostinger que exporte la base de datos MySQL y la comprima en una carpeta segura no accesible mediante peticiones HTTP web:

```text
mysqldump -u uXXXXX_espo_usr -p'CONTRASEÑA_BD' uXXXXX_espocrm | gzip > /home/uXXXXX/backups/espocrm_$(date +\%F).sql.gz
```

### 6.2. Copias de Seguridad de Nextcloud DMS (Elestio Multi-VPS)
Crear un archivo script en el Host VPS en `/opt/hrinser-dms/scripts/backup_nextcloud.sh`. El script realiza el dump PostgreSQL de forma no interactiva importando las credenciales desde `nextcloud.env`, realiza el **cifrado asimétrico GPG por llave pública**, **segrega lógicamente las llaves** y añade un paso de **transmisión externa** mandatorio a una bóveda fuera del VPS local, borrando la llave en caliente del servidor host para cumplir con el control **TC-08**:

```bash
#!/bin/bash
# ==============================================================================
# Backup Nextcloud DMS - Elestio VPS (Cifrado Asimétrico GPG + Segregación Física)
# Código de Control: HR-SEC-BACKUP-v3
# ==============================================================================
set -e

# Cargar variables de entorno del archivo .env de Docker para evitar contraseñas hardcoded
if [ -f "/opt/hrinser-dms/nextcloud.env" ]; then
    export $(grep -v '^#' /opt/hrinser-dms/nextcloud.env | xargs)
fi

PROJECT_NAME="hrinser-dms"
BACKUP_DIR="/var/backups/nextcloud"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/nextcloud_backup_$DATE"
KEYS_BACKUP_PATH="$BACKUP_DIR/keys_backup_$DATE"

APP_CONTAINER="hrinser-nextcloud-app"
DB_CONTAINER="hrinser-nextcloud-db"
DB_PASS="${POSTGRES_PASSWORD}"
GPG_RECIPIENT="backup-key@hrinser.cl"  # Llave pública GPG del administrador importada

# Rutas de almacenamiento
VOLUME_DATA_DIR="/var/lib/docker/volumes/${PROJECT_NAME}_nextcloud_data/_data"

# Validar llave pública GPG en el host
if ! gpg --list-keys "$GPG_RECIPIENT" >/dev/null 2>&1; then
    echo "ERROR: La llave pública GPG para $GPG_RECIPIENT no está instalada en el Host." >&2
    exit 1
fi

mkdir -p "$BACKUP_DIR"
echo "=== [${DATE}] Iniciando Respaldo DMS Segurizado ==="

# 1. Activar Modo Mantenimiento
docker exec -u www-data "$APP_CONTAINER" php occ maintenance:mode --on

# 2. Dump de BD sin prompt interactivo
docker exec -e PGPASSWORD="$DB_PASS" "$DB_CONTAINER" pg_dump -U nextcloud_usr -d nextcloud_db > "$BACKUP_PATH.sql"

# 3. Comprimir Datos del Sistema (EXCLUYENDO llaves de cifrado)
tar -czf "$BACKUP_PATH.tar.gz" \
    --exclude="./data/files_encryption" \
    --exclude="./data/*/files_encryption" \
    -C "$VOLUME_DATA_DIR" . \
    -C "$BACKUP_DIR" "nextcloud_backup_$DATE.sql"

# 4. Comprimir Llaves de Cifrado por separado (Aborta si hay error)
tar -czf "$KEYS_BACKUP_PATH.tar.gz" \
    -C "$VOLUME_DATA_DIR" \
    "./data/files_encryption" \
    $(find "$VOLUME_DATA_DIR/data/" -maxdepth 2 -type d -name "files_encryption" -not -path '*/appdata_*' | sed "s|$VOLUME_DATA_DIR/||g")

# 5. Desactivar Modo Mantenimiento
docker exec -u www-data "$APP_CONTAINER" php occ maintenance:mode --off
rm "$BACKUP_PATH.sql"

# 6. Cifrar asimétricamente los respaldos con la clave pública GPG
gpg --encrypt --recipient "$GPG_RECIPIENT" --trust-model always "$BACKUP_PATH.tar.gz"
gpg --encrypt --recipient "$GPG_RECIPIENT" --trust-model always "$KEYS_BACKUP_PATH.tar.gz"
rm "$BACKUP_PATH.tar.gz"
rm "$KEYS_BACKUP_PATH.tar.gz"

# 7. MANDATORIO: Transmitir llave cifrada externamente y borrarla localmente (TC-08)
# Ejemplo usando SCP a un servidor frío de auditoría externo
scp -i /home/dano/.ssh/id_rsa_backup "$KEYS_BACKUP_PATH.tar.gz.gpg" auditor@seguro.hrinser.cl:/boveda/keys/
# Eliminar copia de llave local de forma permanente tras la copia externa exitosa
rm "$KEYS_BACKUP_PATH.tar.gz.gpg"

# 8. Rotación automática local de volumen de datos a 14 días
find "$BACKUP_DIR" -name "nextcloud_backup_*.tar.gz.gpg" -mtime +14 -delete

echo "=== Respaldo DMS Finalizado Exitosamente ==="
```

Programar en el crontab del Host:
```text
0 4 * * * /bin/bash /opt/hrinser-dms/scripts/backup_nextcloud.sh >> /var/log/backup_nextcloud.log 2>&1
```

---

## 7. Procedimiento de Restauración ante Desastres (Disaster Recovery)

Para cumplir con el tiempo objetivo de recuperación (**RTO < 20 minutos**) establecido en el test **TC-07**:

1. **Detener Contenedores en Ejecución:**
   ```bash
   cd /opt/hrinser-dms/
   docker compose down -v
   ```
2. **Desencriptar Respaldos con GPG:**
   *(Requiere ingresar la llave privada del administrador mediante agente GPG seguro)*:
   ```bash
   gpg --decrypt /var/backups/nextcloud/nextcloud_backup_DATE.tar.gz.gpg > nextcloud_backup_DATE.tar.gz
   # Descargar la llave criptográfica desde la bóveda fría externa de auditoría
   scp auditor@seguro.hrinser.cl:/boveda/keys/keys_backup_DATE.tar.gz.gpg /var/backups/nextcloud/
   gpg --decrypt /var/backups/nextcloud/keys_backup_DATE.tar.gz.gpg > keys_backup_DATE.tar.gz
   ```
3. **Restaurar el Volumen de Datos y Llaves:**
   ```bash
   # Crear volumen vacío montándolo de nuevo
   docker compose up db redis -d
   # Extraer los datos en el volumen físico correspondiente
   tar -xzf nextcloud_backup_DATE.tar.gz -C /var/lib/docker/volumes/hrinser-dms_nextcloud_data/_data/
   # Extraer las llaves criptográficas segregadas en su ubicación original
   tar -xzf keys_backup_DATE.tar.gz -C /var/lib/docker/volumes/hrinser-dms_nextcloud_data/_data/
   ```
4. **Restaurar la Base de Datos PostgreSQL:**
   * Cargar la variable de base de datos e importar el dump SQL de forma no interactiva:
   ```bash
   docker exec -i -e PGPASSWORD="SecurePass_Postgres_32Char!" hrinser-nextcloud-db psql -U nextcloud_usr -d nextcloud_db < /var/lib/docker/volumes/hrinser-dms_nextcloud_data/_data/nextcloud_backup_DATE.sql
   # Eliminar el archivo SQL residual
   rm /var/lib/docker/volumes/hrinser-dms_nextcloud_data/_data/nextcloud_backup_DATE.sql
   ```
5. **Corregir Permisos y Levantar la Aplicación:**
   ```bash
   docker compose up -d
   # Forzar corrección de permisos del volumen en caliente a www-data
   docker exec -u root hrinser-nextcloud-app chown -R www-data:www-data /var/www/html
   docker exec -u www-data hrinser-nextcloud-app php occ maintenance:mode --off
   # Limpiar archivos .tar.gz residuales del host
   rm nextcloud_backup_DATE.tar.gz keys_backup_DATE.tar.gz
   ```

### 7.1. Protocolo de Notificación de Brechas de Seguridad (Obligatoriedad Ley 21.719)
En caso de que durante el desastre o restauración se detecte una intrusión, robo de datos o filtración de datos sensibles de peritajes:
1. **Reporte Interno Inmediato:** El DevOps a cargo debe levantar una alerta roja a Vanessa Galdames.
2. **Notificación APDP:** HRINSER deberá notificar formalmente a la **Agencia de Protección de Datos Personales (APDP)** de Chile dentro de un plazo máximo de **72 horas** contadas desde que se tomó conocimiento del incidente, detallando la naturaleza del incidente, categorías de datos afectados y medidas correctoras en curso.
3. **Notificación a Afectados:** Si el incidente representa un riesgo para los derechos de los titulares (trabajadores de contratistas), notificar individualmente mediante correo transaccional seguro sobre la brecha.

---

## 8. Flujo de Onboarding Técnico de un Nuevo Cliente Contratista

Cuando una oportunidad comercial se cierra como ganada y el cliente pasa a estado **"Cliente Activo"** en EspoCRM, se ejecuta el siguiente flujo técnico de aprovisionamiento:

```
[Cambio de Estado en EspoCRM]
              │
              ▼
1. Aprovisionamiento Cloud (Desarrollador DevOps)
   - Levantar VPS Dedicado de 2 vCPUs y 4 GB de RAM en Elestio.
   - Apuntar subdominio en el DNS Zone Editor de Hostinger (ej: cliente.hrinser.cl) a la IP.
   - Activar cifrado en reposo LUKS a nivel de disco del host.
              │
              ▼
2. Configuración y Endurecimiento DMS (Desarrollador DevOps)
   - Ejecutar docker-compose up -d.
   - Activar Cifrado Server-Side y 2FA obligatorio vía occ CLI.
   - Registrar el subdominio en Trusted Domains del Nextcloud.
   - Estructurar el skeleton de carpetas básicas de control documental.
              │
              ▼
3. Ingesta de Datos Históricos (Desarrollador DevOps)
   - Cargar los documentos previos del contratista vía WebDAV.
   - Sincronizar e invocar cifrado en bloque mediante encrypt-all.
              │
              ▼
4. Entrega de Credenciales e Inducción (Vanessa Galdames)
   - Generar el usuario final de acceso en Nextcloud.
   - Enrolar 2FA del cliente y realizar capacitación de 30 minutos.
```

---

## 9. Plan de Verificación y Pruebas (UAT) para Desarrolladores

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
