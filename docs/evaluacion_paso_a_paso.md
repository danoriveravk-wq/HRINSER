# 📋 Informe de Evaluación Técnica y QA: Plan de Implementación DMS & CRM HRINSER

**Código de Evaluación:** HR-QA-2026-001  
**Evaluador:** Senior Project Manager & Technical QA Lead  
**Fecha de Evaluación:** 13 de Julio de 2026  
**Documento Evaluado:** `/home/dano/Documentos/DMS-proyecto/plan_implementacion_equipo.md`

---

## 1. Introducción y Objetivo de la Evaluación

El presente informe contiene la evaluación técnica del plan de implementación consolidado para el ecosistema de **HRINSER** (Landing Page, EspoCRM y Nextcloud DMS). El objetivo es asegurar que la guía de despliegue sea lógica, ordenada, completa y directamente accionable por los desarrolladores (Frontend, Backend, DevOps), mitigando riesgos de seguridad, rendimiento y cumplimiento normativo de la **Ley 21.719** de Chile.

---

## 2. Análisis Detallado por Componente y Fase

### 2.1. Frontend (Landing Page)
* **Puntos Fuertes:**
  * Excelente consideración del diseño líquido para multi-idioma (ES/EN/ZH) abordando las diferencias de expansión del español y la densidad vertical del chino.
  * Correcta optimización del rendimiento al sugerir el uso de fuentes de sistema para el chino en lugar de descargar archivos `.woff2` de gran tamaño.
  * Inclusión del script bloqueante inline anti-flicker (FOUC) para el modo oscuro, lo cual es una buena práctica técnica para mejorar la experiencia de usuario y el Cumulative Layout Shift (CLS).
* **Brechas / Cabos Sueltos:**
  * **Configuración de CI/CD:** Se menciona conceptualmente el pipeline en GitHub Actions con SFTP, pero no se proporciona la plantilla del archivo `.github/workflows/deploy.yml` ni se listan las variables secretas requeridas (`SFTP_SERVER`, `SFTP_USERNAME`, `SFTP_KEY` o `SFTP_PASSWORD`). Un frontend dev requerirá asistencia de DevOps para configurar esto.
  * **Alternancia de Idiomas:** Aunque se sugiere usar directorios físicos para optimización SEO internacional (`/en/`, `/zh/`), no se detalla cómo se gestionará la sincronización y enrutamiento en Hostinger para estos subdirectorios (por ejemplo, mediante reglas en `.htaccess`).

### 2.2. Backend & CRM (EspoCRM en Hostinger Shared Hosting)
* **Puntos Fuertes:**
  * Correcta especificación del set de caracteres `utf8mb4` and colación `utf8mb4_unicode_ci` en MySQL para prevenir fallos con caracteres especiales.
  * Inclusión del script proxy seguro PHP `submit_lead.php` con validación backend de reCAPTCHA v3 y timeout estricto de 5 segundos para llamadas cURL a la API de EspoCRM, previniendo bloqueos del servidor.
  * Algoritmo de validación de RUT chileno en frontend bien implementado usando módulo 11.
* **Brechas / Cabos Sueltos / Errores de Configuración:**
  * **Ruta Absoluta del Cron de EspoCRM:** El comando cron propuesto es `/usr/bin/php8.2 /home/uXXXXX/public_html/crm/cron.php`. Sin embargo, en el paso B.1 se detalla que el CRM está en el subdominio `crm.hrinser.cl`. En Hostinger, los subdominios no siempre se alojan bajo `/public_html/crm/`, sino comúnmente en rutas como `/home/uXXXXX/domains/hrinser.cl/public_html/crm/` o `/home/uXXXXX/crm.hrinser.cl/`. Si la ruta es incorrecta, el cron de EspoCRM no se ejecutará, rompiendo notificaciones y automatizaciones de flujos de trabajo.
  * **Credenciales expuestas en Código:** En `submit_lead.php`, se dejan marcadores de posición (`TU_RECAPTCHA_SECRET_KEY_AQUI` y `TU_ACCESS_KEY_DE_ESPOCRM`) directamente en el archivo. No se indica que deban extraerse a un archivo de configuración no público o variables de entorno para evitar subirlos a repositorios.
  * **Rate Limiting Inexistente:** El test **TC-09** espera que el proxy devuelva `HTTP 429 Too Many Requests` ante un ataque de denegación de servicio (50 peticiones consecutivas). Sin embargo, el script `submit_lead.php` **no cuenta con lógica interna de control de tasa** (Rate Limiting). En un hosting compartido, esto causará que las 50 peticiones ejecuten cURL hacia Google y EspoCRM, consumiendo recursos de CPU y memoria de la cuenta y arriesgando un bloqueo temporal del hosting por sobrepasar límites de procesos concurrentes.
  * **Falta Plan de Backup para EspoCRM:** La sección 6.1 menciona de manera vaga que "se debe programar una rutina semanal... y descargas mediante SSH", pero no proporciona scripts ni comandos específicos para automatizar el respaldo de la base de datos MySQL en Hostinger.

### 2.3. DevOps & DMS (Elestio Multi-VPS Nextcloud)
* **Puntos Fuertes:**
  * Gran optimización de recursos mediante límites estrictos de CPU y memoria RAM en Docker Compose para prevenir caídas por Out-Of-Memory (OOM) causadas por el uso de Tesseract OCR.
  * Correcta segregación de llaves criptográficas en el respaldo de Nextcloud para cumplir con la normativa y el control TC-08.
  * El script de respaldo automatiza la activación del modo mantenimiento, dump Postgres y cifrado GPG asimétrico en el host.
* **Brechas / Cabos Sueltos / Errores Críticos de Diseño:**
  * **Falla Crítica de Cifrado en Ingesta Histórica (Paso 5.1):** El plan instruye transferir 100 GB históricos por SFTP directamente al volumen físico de datos en el host (`/var/lib/docker/volumes/hrinser-dms_nextcloud_data/_data/data/...`) y luego correr `files:scan`. **ESTO ES UN ERROR CRÍTICO DE SEGURIDAD.** Dado que el Cifrado del Lado del Servidor (Server-Side Encryption) se activa en el paso 4.1, Nextcloud **solo cifra archivos cuando estos ingresan al sistema a través de su API / capa WebDAV**. Los archivos copiados directamente en el sistema de archivos del host y luego indexados mediante `files:scan` **quedarán almacenados en TEXTO PLANO (sin cifrar)** en el disco duro. Esto viola directamente los requisitos de confidencialidad de la Ley 21.719 evaluados en el test **TC-02**.
  * **Inconsistencia de Configuración en docker-compose.yml:** El plan de directorios del Paso 3.2 menciona la creación de un archivo `nextcloud.env`. Sin embargo, el `docker-compose.yml` del Paso 3.4 **no hace uso de este archivo** (no incluye la directiva `env_file`), y en su lugar tiene contraseñas y configuraciones críticas hardcoded en texto plano dentro de la definición de cada servicio. Esto expone credenciales en el control de versiones.
  * **Riesgo en Script de Backup (`backup_nextcloud.sh`):** El comando para respaldar llaves utiliza:
    ```bash
    tar -czf "$KEYS_BACKUP_PATH.tar.gz" ... 2>/dev/null || true
    ```
    Si el comando `tar` falla por algún motivo (ej. permisos o sintaxis vacía del comando `find`), el operador `|| true` ocultará el fallo y el script continuará su ejecución eliminando los archivos temporales y reportando un estado exitoso. Esto creará un respaldo corrupto sin las llaves necesarias para recuperar la información, haciendo inservibles los respaldos cifrados.
  * **Ausencia de Procedimiento de Restauración:** El test **TC-07** valida la recuperación ante desastres en menos de 20 minutos. No obstante, el plan **no describe en ningún lugar los pasos técnicos para realizar la restauración** de los respaldos cifrados con GPG (desencriptación, restauración del dump SQL en Postgres, montaje de archivos y llaves). Sin esta guía paso a paso, el equipo de desarrollo no podrá cumplir con el RTO (Recovery Time Objective) de 20 minutos en una emergencia real.

---

## 3. Matriz de Hallazgos y Riesgos Detectados

| ID | Componente | Descripción del Hallazgo | Nivel de Riesgo | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **H-01** | DMS / Seguridad | Los archivos históricos subidos por SFTP directo al volumen no se cifrarán por Nextcloud, quedando en texto plano en el servidor. | 🔴 **CRÍTICO** | Incumplimiento directo de la Ley 21.719 y fallo en auditorías de seguridad. |
| **H-02** | CRM / Despliegue | La ruta del Cron de EspoCRM en Hostinger asume `/public_html/crm/`, lo cual es incorrecto para subdominios estándar en la plataforma. | 🟡 **MEDIO** | Fallo silencioso de todas las tareas en segundo plano del CRM. |
| **H-03** | CRM / Seguridad | Ausencia de Rate Limiting en el script PHP proxy, permitiendo denegación de servicio (DoS) y caídas del hosting compartido. | 🔴 **ALTO** | Caída de la Landing Page por consumo excesivo de recursos de CPU/RAM en Hostinger. |
| **H-04** | DMS / DevOps | El archivo `nextcloud.env` no está integrado en el Docker Compose; credenciales críticas están hardcoded en el YAML. | 🟡 **MEDIO** | Exposición de credenciales de bases de datos y Redis en repositorios de código. |
| **H-05** | DMS / Backup | El script de respaldo silencia fallos en la compresión de llaves criptográficas (`|| true`), arriesgando pérdida total de datos. | 🔴 **ALTO** | Respaldos inservibles debido a la falta de llaves para desencriptar la información restaurada. |
| **H-06** | CRM / DevOps | Falta de scripts y comandos explícitos para respaldar la base de datos MySQL de EspoCRM en Hostinger. | 🟡 **MEDIO** | Pérdida de leads y datos comerciales ante fallos en el hosting. |
| **H-07** | DMS / DR | Ausencia completa de instrucciones detalladas para restaurar el sistema DMS ante un desastre. | 🔴 **ALTO** | Incapacidad de restaurar el servicio en producción dentro del tiempo objetivo (RTO < 20 min). |

---

## 4. Estado de la Evaluación y Conclusión

> [!WARNING]
> ### ESTADO FINAL: PENDIENTE DE MEJORA
> El plan de implementación actual presenta una excelente estructura conceptual y arquitectónica, pero contiene **deficiencias técnicas críticas** en la ingesta de datos históricos, la seguridad de las credenciales, el control de tasa (Rate Limiting) y la resiliencia del sistema de copias de seguridad. No se recomienda autorizar la ejecución en producción sin subsanar las observaciones listadas en la sección de recomendaciones.

---

## 5. Recomendaciones de Mejora (Directamente Accionables)

Para transicionar el plan al estado **'LISTO PARA EJECUCIÓN'**, aplique los siguientes ajustes correctivos:

### 1. Corregir la Ingesta de Datos Históricos (Cifrado DMS)
* **Solución:** Reemplazar el método de copia directa al volumen de Docker por una ingesta mediante un script local de sincronización que use la API de WebDAV de Nextcloud (por ejemplo, configurando `rclone` con el endpoint WebDAV del usuario cliente). Esto forzará a Nextcloud a cifrar cada archivo en el momento de la ingesta.
* **Alternativa:** Si se requiere usar el volumen de Docker por velocidad, se debe realizar la copia antes de activar el cifrado global en el paso 4.1 y, posteriormente, ejecutar el comando de cifrado masivo:
  ```bash
  docker exec -u www-data hrinser-nextcloud-app php occ encryption:encrypt-all
  ```
  *(Nota: Se debe advertir que este proceso consume recursos intensivos y debe hacerse antes de abrir el servicio al cliente).*

### 2. Implementar Rate Limiting en el Proxy PHP (`submit_lead.php`)
* Modificar el script PHP para guardar registros de solicitudes en un archivo local temporal `/tmp/rate_limit_*.json` o base de datos local y bloquear IPs que realicen más de 5 peticiones por minuto, respondiendo con un encabezado `HTTP/1.1 429 Too Many Requests`.
* Ejemplo de implementación simple a añadir al inicio de `submit_lead.php`:
  ```php
  $ip = $_SERVER['REMOTE_ADDR'] ?? 'unknown';
  $limitFile = sys_get_temp_dir() . '/rate_limit_' . md5($ip) . '.json';
  $currentTime = time();
  $window = 60; // 1 minuto
  $limit = 5; // peticiones máximas

  if (file_exists($limitFile)) {
      $data = json_decode(file_get_contents($limitFile), true);
      $data['requests'] = array_filter($data['requests'], function($ts) use ($currentTime, $window) {
          return $ts > ($currentTime - $window);
      });
      if (count($data['requests']) >= $limit) {
          http_response_code(429);
          header('Retry-After: ' . $window);
          echo json_encode(['error' => 'Demasiadas solicitudes. Por favor intente más tarde.']);
          exit;
      }
      $data['requests'][] = $currentTime;
  } else {
      $data = ['requests' => [$currentTime]];
  }
  file_put_contents($limitFile, json_encode($data));
  ```

### 3. Securizar `docker-compose.yml` con `.env`
* Cambiar las contraseñas hardcoded por referencias a variables de entorno en el archivo `docker-compose.yml` (ej. `${POSTGRES_PASSWORD}`) y cargarlas en el archivo `nextcloud.env`.
* Configurar la directiva `env_file:` en el archivo compose para los servicios `app`, `db`, y `redis`.

### 4. Ajustar la Ruta del Cron en Hostinger
* Modificar la instrucción de la tarea Cron de EspoCRM para reflejar la ruta absoluta estándar en Hostinger para subdominios:
  `/usr/bin/php8.2 /home/uXXXXX/domains/hrinser.cl/public_html/crm/cron.php >> ...` (o la ruta exacta asignada por el panel).

### 5. Robustecer el Script de Backup de Nextcloud
* Eliminar el operador `|| true` del comando `tar` de llaves para que el script aborte inmediatamente (`set -e`) si el respaldo de llaves de cifrado falla, evitando crear un backup incompleto:
  ```bash
  tar -czf "$KEYS_BACKUP_PATH.tar.gz" -C "$VOLUME_DATA_DIR" "./data/files_encryption" $(find "$VOLUME_DATA_DIR/data/" -maxdepth 2 -type d -name "files_encryption" -not -path '*/appdata_*' | sed "s|$VOLUME_DATA_DIR/||g")
  ```

### 6. Agregar Procedimiento de Restauración ante Desastres (Nextcloud)
* Agregar un paso técnico explícito bajo la sección de Backup que enseñe al administrador a restaurar el DMS:
  1. Detener contenedores: `docker compose down`.
  2. Desencriptar archivos de respaldo: `gpg --decrypt nextcloud_backup_DATE.tar.gz.gpg > nextcloud_backup_DATE.tar.gz`.
  3. Descomprimir archivos de datos en el volumen Docker de Nextcloud.
  4. Descomprimir llaves en su ubicación original dentro del volumen.
  5. Levantar servicios `db` y `redis`.
  6. Restaurar el dump SQL: `cat nextcloud_backup_DATE.sql | docker exec -i hrinser-nextcloud-db psql -U nextcloud_usr -d nextcloud_db`.
  7. Levantar contenedor `app` y desactivar el modo mantenimiento: `docker exec -u www-data hrinser-nextcloud-app php occ maintenance:mode --off`.

### 7. Proveer Script de Backup para EspoCRM
* Agregar un comando en la sección 6.1 para respaldar la base de datos de EspoCRM en Hostinger mediante tareas programadas (hPanel) apuntando a un directorio seguro no accesible por HTTP:
  ```text
  mysqldump -u uXXXXX_espo_usr -p'CONTRASEÑA' uXXXXX_espocrm | gzip > /home/uXXXXX/backups/espocrm_$(date +\%F).sql.gz
  ```
