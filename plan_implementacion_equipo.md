# 🛠️ Guía de Implementación y Despliegue Técnico: Ecosistema DMS & CRM HRINSER
### Manual de Ejecución Paso a Paso para Desarrolladores (Cumplimiento Ley 21.719 - Chile)
**Código del Documento:** HR-DEV-2026-001  
**Preparado por:** Lead DevOps & Systems Architect  
**Destinatarios:** Equipo de Desarrollo, DevOps e Ingenieros de Software  
**Fecha:** 13 de Julio de 2026  

---

## 1. Resumen de la Arquitectura del Sistema

Esta guía detalla las tareas técnicas y de infraestructura necesarias para implementar y desplegar el ecosistema digital de **HRINSER** (Gestión Comercial SpA). La solución está compuesta por:
1. **Landing Page Corporativa:** Un frontend estático avanzado con soporte multi-idioma (Español/Inglés/Chino) y selector de modo claro/oscuro.
2. **Backoffice CRM (HubSpot):** Parametrización del pipeline de ventas de 10 etapas con campos obligatorios para el control de prospectos.
3. **Plataforma de Gestión Documental (DMS Nextcloud):** Arquitectura **Multi-VPS en Elestio**. Cada cliente contratista opera en una máquina virtual dedicada con su propio stack aislado en Docker (Nextcloud Core + PostgreSQL + Redis + Cron), cumpliendo de manera estricta las exigencias de privacidad de la **Ley 21.719**.

```
[Cliente Web] ---> [DNS Cloudflare]
                          │
       ┌──────────────────┴──────────────────┐ (HTTPS / TLS 1.3)
       ▼                                     ▼
┌──────────────┐                      ┌──────────────┐
│ VPS Cliente1 │                      │ VPS Cliente2 │
│ (Elestio)    │                      │ (Elestio)    │
├──────────────┤                      ├──────────────┤
│ NC App       │                      │ NC App       │
│ Postgres DB  │                      │ Postgres DB  │
│ Redis Cache  │                      │ Redis Cache  │
│ Vol. Cifrado │                      │ Vol. Cifrado │
└──────────────┘                      └──────────────┘
```

---

## 2. Fase 1: Configuración de DNS, Landing Page y HubSpot CRM

### Paso 2.1: Gestión de DNS y Zonas en Cloudflare
1. Configurar la zona DNS del dominio `hrinser.cl` en Cloudflare.
2. Crear registros de tipo `A` comodín (Wildcard) o subdominios específicos apuntando a las IPs de los servidores aprovisionados:
   * `alfa.hrinser.cl` -> `IP_VPS_ALFA`
   * `beta.hrinser.cl` -> `IP_VPS_BETA`
3. Habilitar la política de seguridad SSL/TLS en modo **Strict (Estricto)** en Cloudflare para forzar conexiones HTTPS cifradas de extremo a extremo.

### Paso 2.2: Despliegue de la Landing Page Corporativa
1. **Estructura del Proyecto:** La Landing Page debe ser un desarrollo liviano (HTML/CSS Vanilla o Vite/React estático) estructurado para evitar cuellos de botella de renderizado.
2. **Soporte Multi-Idioma (i18n):**
   * Configurar un selector de idiomas en el header que modifique de forma reactiva las cadenas de texto del sitio (Español, Inglés y Chino).
   * Almacenar la preferencia de idioma del navegador en el `localStorage` del cliente.
3. **Modo Claro/Oscuro (Dark Mode):**
   * Implementar mediante clases CSS (`.dark`) y variables CSS.
   * Por defecto, leer la preferencia del sistema operativo del usuario (`prefers-color-scheme`).
4. **Formulario de Captación de Leads:**
   * Diseñar un formulario de contacto seguro que valide en el cliente los campos obligatorios: Nombre, Correo Corporativo, Teléfono, Empresa, RUT y Número de Trabajadores.
   * Conectar el envío del formulario a la API de HubSpot o mediante Webhooks seguros.

### Paso 2.3: Configuración de HubSpot CRM
1. **Configuración de la Cuenta:** Acceder a la consola de desarrollador o administrador de HubSpot.
2. **Creación del Pipeline Comercial:** Configurar un pipeline de negocios con las siguientes 10 etapas exactas:
   $$\text{Identificada} \rightarrow \text{Investigada} \rightarrow \text{Contactada} \rightarrow \text{Respondió} \rightarrow \text{Reunión Agendada} \rightarrow \text{Diagnóstico Realizado} \rightarrow \text{Propuesta Enviada} \rightarrow \text{Piloto} \rightarrow \text{Cliente Activo} \rightarrow \text{No Interesado}$$
3. **Propiedades Personalizadas y Obligatoriedad:**
   Crear y exigir los siguientes campos para que un negocio pueda ser avanzado a la etapa de "Propuesta Enviada":
   * `rut_empresa` (Formato RUT chileno validado por expresión regular).
   * `numero_trabajadores` (Número entero).
   * `mandante_principal` (Texto).
   * `plataforma_acreditacion` (Menú desplegable: SUCAL, Workmate, SAP, Sistemas Internos).
   * `fecha_proximo_seguimiento` (Fecha, campo obligatorio).

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
Este archivo define el stack con límites estrictos de CPU y memoria RAM para evitar que los picos de OCR consuman toda la memoria física del host y activen el kernel **OOM Killer** del sistema operativo.

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
          cpus: '1.0'
          memory: 1024M
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
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 128M
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
          cpus: '1.5'
          memory: 2048M
        reservations:
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/cron.php"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ----------------------------------------------------------------------------
  # CRON SERVICE: Demonio asíncrono para la ejecución de tareas de fondo
  # ----------------------------------------------------------------------------
  cron:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: hrinser-nextcloud-cron
    restart: always
    volumes:
      - nextcloud_data:/var/www/html
    entrypoint: |
      bash -c "
      echo 'Iniciando demonio cron interno para Nextcloud...'
      while true; do
        su -s /bin/bash www-data -c 'php -f /var/www/html/cron.php'
        sleep 300
      done
      "
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 128M

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

### Paso 4.2: Enforzar Doble Factor de Autenticación (2FA) Obligatorio
Para impedir el uso de contraseñas débiles o accesos no autorizados:
1. Habilitar la app de autenticación de dos factores (ej: `twofactor_totp`):
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ app:enable twofactor_totp
   ```
2. Forzar que todos los grupos de usuarios deban configurar obligatoriamente el 2FA en su primer inicio de sesión:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ twofactor:enforce
   ```

### Paso 4.3: Registro de Actividades Inmutable (Audit Trail)
Activar el log administrativo de accesos para contar con registros forenses válidos ante mandantes y tribunales:
1. Habilitar el módulo de auditoría:
   ```bash
   docker exec -u www-data hrinser-nextcloud-app php occ app:enable admin_audit
   ```
2. Configurar el archivo `config.php` de Nextcloud para redirigir los logs de auditoría a la salida estándar o a un archivo dedicado `/var/www/html/data/audit.log`, asegurando que registre la IP origen del cliente y el identificador de usuario en cada lectura, descarga o borrado de archivos.

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
3. Configurar la ruta de ejecución en la configuración de la app de OCR como `/usr/bin/tesseract` y el idioma a `Spanish (spa)`.
4. Configurar el programador del host VPS para ejecutar la cola de OCR en horario no hábil (madrugada) para procesar los documentos en bloque:
   ```bash
   sudo crontab -e
   ```
5. Añadir la siguiente regla cron programada diariamente a las **03:00 AM (hora local chilena)**:
   ```text
   0 3 * * * docker exec -u www-data hrinser-nextcloud-app php occ ocr:process
   ```
6. Guardar y salir. Esto garantiza que la carga pesada de CPU de Tesseract ocurra cuando la concurrencia en la plataforma es nula.

---

## 6. Plan de Verificación y Pruebas (UAT) para Desarrolladores

Antes de entregar los portales de clientes a producción, los desarrolladores deben validar las siguientes pruebas de aceptación:

| Test Case | Objetivo | Procedimiento de Validación | Resultado Esperado |
| :--- | :--- | :--- | :--- |
| **TC-01** | Validar aislamiento de red y archivos | Intentar acceder desde el contenedor del Cliente A al volumen y base de datos del Cliente B. | Error de conexión (redes de Docker completamente aisladas por VPS dedicado). |
| **TC-02** | Validar cifrado en reposo | Acceder por terminal al volumen físico del host y ejecutar `cat` en un archivo PDF subido en la carpeta del cliente. | El texto devuelto debe ser ilegible / binario encriptado (Server-Side Encryption activo). |
| **TC-03** | Validar enforzamiento de 2FA | Crear un usuario de prueba en Nextcloud e intentar iniciar sesión sin configurar 2FA. | El sistema debe bloquear el dashboard y exigir obligatoriamente el enrolamiento de 2FA. |
| **TC-04** | Validar inmutabilidad de logs | Descargar un archivo con el usuario de prueba, acceder al contenedor de base de datos/archivos y leer el archivo `audit.log`. | Debe existir una entrada JSON conteniendo la IP origen, el usuario, la acción `file_download` y el archivo exacto. |
| **TC-05** | Validar OCR en horario no hábil | Subir un archivo PDF escaneado (imagen), ejecutar manualmente el comando de OCR `php occ ocr:process` y verificar que el texto del PDF ahora sea seleccionable y buscable. Verificar que el cronjob del host está activo con `crontab -l`. | El texto del PDF se digitaliza exitosamente y la tarea cron está programada a las 03:00 AM. |

---
*(Fin del Documento)*
