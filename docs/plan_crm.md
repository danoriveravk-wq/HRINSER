# 🛠️ Guía de Implementación y Despliegue Técnico: CRM & Landing Page HRINSER
### Manual de Ejecución Paso a Paso para Desarrolladores (Hostinger Shared Hosting)
**Código del Documento:** HR-DEV-CRM-2026-001  
**Preparado por:** Lead DevOps & Systems Architect  
**Destinatarios:** Desarrolladores Frontend y Backend  
**Fecha:** 13 de Julio de 2026  

---

## 1. Distribución de Infraestructura y Resumen Arquitectónico

Este documento detalla los pasos para implementar y desplegar el **CRM (EspoCRM)** y la **Landing Page Corporativa** de **HRINSER** (Gestión Comercial SpA) sobre el plan **Hostinger Shared Web Hosting (Plan Unlimited)** ya contratado.

| Componente | Tipo de Aplicación | Servidor | Razón Técnica |
| :--- | :--- | :--- | :--- |
| **Landing Page** | Frontend estático (HTML/CSS/JS) | **Hostinger Shared Hosting** | Consumo mínimo de recursos, bajo costo, despliegue simple. |
| **EspoCRM** | Aplicación PHP Nativa + DB MariaDB | **Hostinger Shared Hosting** | Instalación directa sin contenedores para optimizar costos de infraestructura. |
| **DNS Manager** | Editor de Zona DNS nativo | **Hostinger hPanel** | Centraliza la resolución de nombres de `hrinser.cl` de forma gratuita. |

---

## 2. Fase 1: Configuración de DNS y Despliegue de la Landing Page

### Paso 2.1: Gestión de DNS en Hostinger
1. Iniciar sesión en el hPanel de Hostinger y acceder a **Avanzado -> Editor de Zona DNS** para el dominio `hrinser.cl`.
2. Configurar la tabla de registros DNS para direccionar el tráfico web:
   * **Registro A (raíz):** Apuntar `@` a la dirección IP del servidor web de Hostinger (ej: `IP_HOSTINGER_SERVER`).
   * **Registro A (CRM):** Crear `crm` apuntando a `IP_HOSTINGER_SERVER`.
3. Activar el SSL gratuito para el dominio principal y el subdominio `crm.hrinser.cl` desde la sección **Seguridad -> SSL** en el hPanel de Hostinger.

### Paso 2.2: Despliegue de la Landing Page en Hostinger y Ley 21.719
1. **CI/CD Pipeline (GitHub Actions):** Si la landing page requiere una compilación previa (como empaquetadores de Tailwind o Vite), utilizar la plantilla de GitHub Actions `.github/workflows/deploy.yml` para compilar y desplegar vía SFTP de manera automática:

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

## 3. Fase 2: Despliegue e Instalación Nativa de EspoCRM (PHP/MySQL)

### Paso 3.1: Configuración de Base de Datos y PHP en hPanel
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

### Paso 3.2: Subida de Archivos y Tarea Cron
1. Crear el subdominio `crm.hrinser.cl`. En Hostinger, esto crea la carpeta física en `/home/uXXXXX/domains/hrinser.cl/public_html/crm/` o en `/home/uXXXXX/crm.hrinser.cl/`.
2. Subir y descomprimir el paquete de EspoCRM oficial en el directorio asignado al subdominio.
3. Completar el asistente web en `https://crm.hrinser.cl` vinculando la base de datos configurada.
4. **Programación del Cron en hPanel:**
   * Ir a **Avanzado -> Tareas Cron**.
   * Agregar la ejecución del cron cada un minuto, utilizando la ruta absoluta correcta de Hostinger y redirigiendo errores a un archivo de bitácora:
     ```text
     /usr/bin/php8.2 /home/uXXXXX/domains/hrinser.cl/public_html/crm/cron.php >> /home/uXXXXX/domains/hrinser.cl/public_html/crm/data/logs/cron_errors.log 2>&1
     ```

### Paso 3.3: Envío de Correos (SMTP Corporativo)
* **Requisito:** Para evitar que las notificaciones de leads del CRM y los emails de bienvenida caigan en SPAM o sean bloqueados por el hosting, ir a **Administración -> Outbound Emails** en EspoCRM y configurar un servidor **SMTP Externo** utilizando la casilla de correos corporativas creada en Hostinger (ej: `contacto@hrinser.cl`) con autenticación SSL/TLS y puerto `465` o `587`.

### Paso 3.4: Seguridad de Formulario (Validación RUT y reCAPTCHA v3)
Para evitar inyectar leads duplicados y proteger la capacidad de procesamiento de MySQL y CPU de la cuenta compartida de Hostinger (evitando errores *508 Resource Limit Reached*):

1. **Configuración Externa de Credenciales (`config.php`):**
   Para evitar subir las claves de API a repositorios públicos, crear el archivo `config.php` en la carpeta de la landing page para guardar los secretos del backend:
   ```php
   <?php
   // config.php - Secretos Protegidos en el Servidor
   define('RECAPTCHA_SECRET_KEY', 'TU_RECAPTCHA_SECRET_KEY_AQUI');
   define('ESPOCRM_ACCESS_KEY', 'TU_ACCESS_KEY_DE_ESPOCRM');
   ?>
   ```
2. **Crear script Proxy Seguro PHP (`submit_lead.php`):**
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

## 4. Tareas de Copia de Seguridad (Backups) del CRM
Para contar con copias de seguridad locales de la base de datos de EspoCRM independientes a Hostinger, se debe configurar un Cron Job semanal de tipo "Custom Command" en el hPanel de Hostinger que exporte la base de datos MySQL y la comprima en una carpeta segura no accesible mediante peticiones HTTP web:

```text
mysqldump -u uXXXXX_espo_usr -p'CONTRASEÑA_BD' uXXXXX_espocrm | gzip > /home/uXXXXX/backups/espocrm_$(date +\%F).sql.gz
```

---

## 5. Casos de Prueba (UAT) para el CRM

| Test Case | Objetivo | Procedimiento de Validación | Resultado Esperado |
| :--- | :--- | :--- | :--- |
| **TC-06** | Validar Lead Capture en EspoCRM | Completar el formulario de la Landing Page y verificar en la consola de administración de EspoCRM que se cree el Lead y los datos de RUT y trabajadores se guarden. | El contacto se crea automáticamente en la base de datos de EspoCRM con los campos correspondientes. |
| **TC-09** | Validar Rate Limiting y Google reCAPTCHA v3 | Utilizar herramientas de testeo de carga (ej: `ab` o script en bucle) para enviar 50 peticiones consecutivas al script `submit_lead.php`. | El script intermediario de Hostinger rechaza las peticiones por falta de token de reCAPTCHA o devuelve HTTP `429 Too Many Requests`. |

---
*(Fin del Documento)*
