# 🛠️ Guía de Implementación y Despliegue Técnico: Ecosistema DMS & CRM HRINSER
### Manual de Ejecución Paso a Paso para Desarrolladores (Cumplimiento Ley 21.719 - Chile)
**Código del Documento:** HR-DEV-2026-001  
**Preparado por:** Lead DevOps & Systems Architect  
**Destinatarios:** Equipo de Desarrollo, DevOps e Ingenieros de Software  
**Fecha:** 13 de Julio de 2026  

---

## 1. Resumen de la Arquitectura del Sistema

Esta guía detalla las tareas técnicas y de infraestructura necesarias para implementar y desplegar el ecosistema digital de **HRINSER** (Gestión Comercial SpA). La solución está compuesta por:
1. **Landing Page Corporativa:** Un frontend estático avanzado con soporte multi-idioma (Español/Inglés/Chino) y selector de modo claro/oscuro, desplegado en **Hostinger Web Hosting**.
2. **Backoffice CRM de Código Abierto (EspoCRM):** Despliegue de la solución self-hosted EspoCRM en un **VPS de Hostinger** mediante Docker y MariaDB. Parametrización del pipeline de ventas de 10 etapas en la entidad Oportunidades y adición de campos personalizados (RUT, dotación, mandante, plataforma de acreditación) mediante el Entity Manager.
3. **Plataforma de Gestión Documental (DMS Nextcloud):** Arquitectura **Multi-VPS en Elestio**. Cada cliente contratista opera en una máquina virtual dedicada con su propio stack aislado en Docker (Nextcloud Core + PostgreSQL + Redis + Cron), cumpliendo de manera estricta las exigencias de privacidad de la **Ley 21.719**.

```
[Cliente Web] ---> [DNS Cloudflare / Hostinger]
                          │
       ┌──────────────────┴────────────────────────────────────────────────┐ (HTTPS / TLS 1.3)
       ▼                                   ▼                               ▼
┌──────────────┐                    ┌──────────────┐               ┌──────────────┐
│ VPS Hostinger│                    │ VPS Cliente1 │               │ VPS Cliente2 │
│ (EspoCRM)    │                    │ (Elestio NC) │               │ (Elestio NC) │
├──────────────┤                    ├──────────────┤               ├──────────────┤
│ EspoApp      │                    │ NC App       │               │ NC App       │
│ MariaDB DB   │                    │ Postgres DB  │               │ Postgres DB  │
│              │                    │ Redis Cache  │               │ Redis Cache  │
│              │                    │ Vol. Cifrado │               │ Vol. Cifrado │
└──────────────┘                    └──────────────┘               └──────────────┘
```

---

## 2. Fase 1: Configuración de DNS, Landing Page y Despliegue de EspoCRM en Hostinger

### Paso 2.1: Gestión de DNS y Zonas en Cloudflare o Hostinger
1. Configurar la zona DNS del dominio `hrinser.cl` (administrada en Hostinger o Cloudflare).
2. Crear registros de tipo `A` comodín (Wildcard) o subdominios específicos apuntando a las IPs correspondientes:
   * `@` (raíz para la landing page) -> `IP_HOSTINGER_WEB_HOSTING`
   * `crm.hrinser.cl` -> `IP_VPS_HOSTINGER_CRM`
   * `alfa.hrinser.cl` -> `IP_VPS_ELESTIO_ALFA`
   * `beta.hrinser.cl` -> `IP_VPS_ELESTIO_BETA`
3. Habilitar la política de seguridad SSL/TLS en modo **Strict (Estricto)** para forzar conexiones HTTPS cifradas de extremo a extremo.

### Paso 2.2: Despliegue de la Landing Page en Hostinger Web Hosting
1. **Estructura del Proyecto:** La Landing Page debe ser un desarrollo liviano (HTML/CSS Vanilla o Vite/React estático).
2. **Soporte Multi-Idioma (i18n):**
   * Configurar un selector de idiomas en el header que modifique de forma reactiva las cadenas de texto del sitio (Español, Inglés y Chino).
   * Almacenar la preferencia de idioma del navegador en el `localStorage` del cliente.
3. **Modo Claro/Oscuro (Dark Mode):**
   * Implementar mediante clases CSS (`.dark`) y variables CSS.
   * Por defecto, leer la preferencia del sistema operativo del usuario (`prefers-color-scheme`).
4. **Formulario de Captación de Leads:**
   * Diseñar un formulario de contacto seguro que valide en el cliente los campos obligatorios: Nombre, Correo Corporativo, Teléfono, Empresa, RUT y Número de Trabajadores.
   * Conectar el envío del formulario mediante una petición HTTP POST a la API de Captura de Leads de EspoCRM (`api/v1/Lead`).
5. **Despliegue:**
   * Acceder al panel de Hostinger (hPanel) -> **Hosting** -> **Administrar**.
   * Subir los archivos estáticos en el directorio `public_html` utilizando el Administrador de Archivos o mediante la integración Git nativa de Hostinger vinculando el repositorio.
   * Activar el certificado SSL autogenerado gratuito de Hostinger para el dominio principal `hrinser.cl`.

### Paso 2.3: Despliegue e Integración de EspoCRM en Hostinger VPS
El CRM se despliega en un Servidor VPS de Hostinger configurado con Docker.

#### A. Aprovisionamiento y Securización del VPS en Hostinger
1. En el panel de Hostinger (hPanel), ir a **Servidores VPS** -> **Comprar/Administrar VPS**.
2. Seleccionar la plantilla de sistema operativo recomendada por Hostinger: **Ubuntu 22.04 con Docker preinstalado** (o Ubuntu limpio e instalar Docker manualmente mediante SSH).
3. Habilitar el Firewall en el panel del VPS de Hostinger y abrir únicamente los puertos:
   * `80/tcp` (HTTP) y `443/tcp` (HTTPS).
   * `22/tcp` (SSH) restringido a la llave pública SSH del desarrollador.

#### B. Archivo `docker-compose.yml` para EspoCRM
Acceder al VPS de Hostinger por SSH y en `/opt/espocrm/` crear el siguiente archivo Docker Compose. Para producción, el contenedor de EspoCRM escuchará directamente en el puerto HTTP `80` del host (o detrás de un Nginx Proxy Manager si se requiere administrar SSL en el mismo VPS):

```yaml
version: '3.8'

services:
  espocrm-db:
    image: mariadb:10.11-jammy
    container_name: espocrm-db
    restart: always
    environment:
      MARIADB_ROOT_PASSWORD: RootSecurePass_EspoDB2026! # MODIFICAR EN PRODUCCIÓN
      MARIADB_DATABASE: espocrm
      MARIADB_USER: espocrm_user
      MARIADB_PASSWORD: SecurePass_EspoDB2026!       # MODIFICAR EN PRODUCCIÓN
    volumes:
      - espocrm_db_data:/var/lib/mysql
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1024M

  espocrm:
    image: espocrm/espocrm:latest
    container_name: espocrm
    restart: always
    ports:
      - "80:80"  # Mapeado directamente al puerto HTTP estándar en el VPS de Hostinger
    environment:
      ESPOCRM_DATABASE_HOST: espocrm-db
      ESPOCRM_DATABASE_USER: espocrm_user
      ESPOCRM_DATABASE_PASSWORD: SecurePass_EspoDB2026!
      ESPOCRM_DATABASE_NAME: espocrm
      ESPOCRM_LANGUAGE: es_ES
    volumes:
      - espocrm_custom:/var/www/html/custom
      - espocrm_data:/var/www/html/data
      - espocrm_client_custom:/var/www/html/client/custom
    depends_on:
      - espocrm-db
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 1536M

volumes:
  espocrm_db_data:
    driver: local
  espocrm_custom:
    driver: local
  espocrm_data:
    driver: local
  espocrm_client_custom:
    driver: local
```

#### C. Configuración de Entidades y Pipeline Comercial en EspoCRM
Para parametrizar el CRM según el modelo de negocio, el desarrollador debe ingresar al Panel de Administración de EspoCRM y realizar los siguientes ajustes mediante el **Entity Manager**:

1. **Campos Personalizados en la Entidad Oportunidad (`Opportunity`) / Cliente Potencial (`Lead`):**
   Añadir mediante **Administration -> Entity Manager -> Opportunity -> Fields -> Add Field**:
   * `rutEmpresa` (Type: Varchar, Label: "RUT Empresa", con validación de formato RUT chileno en el cliente).
   * `numeroTrabajadores` (Type: Integer, Label: "Número de Trabajadores").
   * `mandantePrincipal` (Type: Varchar, Label: "Mandante Principal").
   * `plataformaAcreditacion` (Type: Enum, Label: "Plataforma de Acreditación", Options: `SUCAL`, `Workmate`, `SAP`, `Sistemas Internos`).

2. **Personalización del Pipeline Comercial (Etapas de la Oportunidad):**
   Editar el campo `stage` (Etapa) en la entidad `Opportunity` para desplegar las 10 etapas comerciales del negocio:
   * **Identificada** (Identified)
   * **Investigada** (Investigated)
   * **Contactada** (Contacted)
   * **Respondió** (Replied)
   * **Reunión Agendada** (Meeting Scheduled)
   * **Diagnóstico Realizado** (Discovery / Diagnosis Done)
   * **Propuesta Enviada** (Proposal Sent)
   * **Piloto** (Pilot)
   * **Cliente Activo** (Active Client / Closed Won)
   * **No Interesado** (Not Interested / Closed Lost)

3. **Integración del Formulario API (Lead Capture):**
   * Ir a **Administration -> Lead Capture**. Crear una nueva campaña de captura.
   * EspoCRM generará un **Access Key** de API y un Payload en formato JSON.
   * Configurar el script de procesamiento del formulario de la Landing Page en Hostinger para enviar un POST HTTP con los datos ingresados al endpoint de EspoCRM `/api/v1/LeadCapture/YOUR_ACCESS_KEY`.

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

## 6. Flujo de Onboarding Técnico de un Nuevo Cliente Contratista

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

## 7. Plan de Verificación y Pruebas (UAT) para Desarrolladores

Antes de entregar los portales de clientes y el CRM a producción, los desarrolladores deben validar las siguientes pruebas de aceptación:

| Test Case | Objetivo | Procedimiento de Validación | Resultado Esperado |
| :--- | :--- | :--- | :--- |
| **TC-01** | Validar aislamiento de red y archivos | Intentar acceder desde el contenedor del Cliente A al volumen y base de datos del Cliente B. | Error de conexión (redes de Docker completamente aisladas por VPS dedicado). |
| **TC-02** | Validar cifrado en reposo | Acceder por terminal al volumen físico del host y ejecutar `cat` en un archivo PDF subido en la carpeta del cliente. | El texto devuelto debe ser ilegible / binario encriptado (Server-Side Encryption activo). |
| **TC-03** | Validar enforzamiento de 2FA | Crear un usuario de prueba en Nextcloud e intentar iniciar sesión sin configurar 2FA. | El sistema debe bloquear el dashboard y exigir obligatoriamente el enrolamiento de 2FA. |
| **TC-04** | Validar inmutabilidad de logs | Descargar un archivo con el usuario de prueba, acceder al contenedor de base de datos/archivos y leer el archivo `audit.log`. | Debe existir una entrada JSON conteniendo la IP origen, el usuario, la acción `file_download` y el archivo exacto. |
| **TC-05** | Validar OCR en horario no hábil | Subir un archivo PDF escaneado (imagen), ejecutar manualmente el comando de OCR `php occ ocr:process` y verificar que el texto del PDF ahora sea seleccionable y buscable. Verificar que el cronjob del host está activo con `crontab -l`. | El texto del PDF se digitaliza exitosamente y la tarea cron está programada a las 03:00 AM. |
| **TC-06** | Validar Lead Capture en EspoCRM | Completar el formulario de la Landing Page y verificar en la consola de administración de EspoCRM que se cree el Lead y los datos de RUT y trabajadores se guarden. | El contacto se crea automáticamente en la base de datos de EspoCRM con los campos correspondientes. |

---
*(Fin del Documento)*
