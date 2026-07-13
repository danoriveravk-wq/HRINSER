# Guía de Despliegue de Infraestructura DMS - HRINSER
**Arquitectura Multi-VPS Optimizada para el Cumplimiento de la Ley 21.719 (Chile)**

Esta carpeta contiene la plantilla de orquestación `docker-compose.yml` para desplegar portales independientes de Nextcloud, optimizados con PostgreSQL para la persistencia relacional y Redis para el almacenamiento en caché en memoria y bloqueo de archivos (*file locking*). Esta configuración reduce sustancialmente el impacto de I/O y previene caídas del sistema por consumo de RAM en cargas de trabajo concurrentes y procesamiento OCR.

---

## 🚀 1. Despliegue Paso a Paso en Elestio

Elestio es una plataforma de despliegue gestionado (Fully Managed Cloud) ideal para arquitecturas multi-instancia ya que automatiza las tareas de red, backups, monitoreo y certificados SSL/TLS de manera nativa.

### Paso 1: Configurar el Entorno en Elestio
1. Inicie sesión en su consola de **Elestio**.
2. Haga clic en **"Deploy My First Service"** (o "Create Service").
3. Seleccione la opción de despliegue mediante plantilla personalizada: **"Docker Compose"**.
4. Defina el proveedor de la nube (ej. Hetzner, Scaleway, AWS) y elija la ubicación geográfica del servidor (se recomienda una latencia baja para usuarios en Chile, ej. US East o localizaciones optimizadas).
5. Seleccione el tamaño de la instancia. Basado en el Reporte de Auditoría:
   * **Requisito mínimo por VPS independiente:** 2 vCPUs dedicadas y 4 GB de RAM (para alojar cómodamente una instancia de Nextcloud con OCR, PostgreSQL y Redis).

### Paso 2: Importar el Docker Compose
1. En la pestaña de configuración del servicio en Elestio, copie y pegue el contenido del archivo `docker-compose.yml` provisto en este directorio.
2. **Parámetros Críticos a Modificar:**
   * Cambie las contraseñas por defecto (`your_secure_postgres_password_here` y `your_secure_redis_password_here`) por claves robustas auto-generadas.
   * Modifique los límites de recursos (`cpus` y `memory`) en la directiva `deploy` si escala el tamaño del VPS asignado.

### Paso 3: Configurar SSL/TLS y Dominio
1. Elestio expone automáticamente el puerto de su contenedor marcado para tráfico web. En este compose, el servicio `app` está expuesto en el puerto `80`. Elestio lo detectará y le asignará una URL aleatoria segura HTTPS (ej: `https://xxxx.elestio.app`).
2. Diríjase a la sección **"Domain Settings"** en el panel del servicio en Elestio para apuntar su dominio personalizado (ej: `empresa1.hrinser.cl`) mediante un registro CNAME en su proveedor DNS. Elestio gestionará la renovación automática del certificado SSL de Let's Encrypt de forma transparente.

---

## 🔒 2. Configuración de Cifrado del Lado del Servidor (Server-Side Encryption)

El cumplimiento de la **Ley 21.719** y la normativa chilena sobre protección de datos sensibles (salud y contratos laborales) exige que los datos estén cifrados en reposo para evitar fugas si el almacenamiento del hosting es comprometido.

### Paso 1: Activar el Módulo de Cifrado
1. Inicie sesión en Nextcloud con la cuenta de administrador creada en el primer inicio.
2. Diríjase a su perfil en la esquina superior derecha y seleccione **Apps** (Aplicaciones).
3. Busque y habilite la aplicación oficial llamada **"Default encryption module"** (Módulo de cifrado predeterminado).

### Paso 2: Habilitar el Cifrado en los Ajustes del Sistema
1. Vaya a **Settings** (Ajustes de administración) -> **Security** (Seguridad).
2. En la sección **Server-side encryption**, marque la casilla **"Enable server-side encryption"** (Habilitar cifrado del lado del servidor).
3. Lea atentamente las advertencias: el cifrado de archivos añade un ligero overhead de CPU al cifrar/descifrar en caliente y aumenta el tamaño de almacenamiento en disco en aproximadamente un 35%.
4. Seleccione el método de recuperación de claves preferido. Para entornos multi-tenant empresariales se recomienda habilitar las claves de recuperación global administradas por el oficial de seguridad informática de HRINSER.

> [!WARNING]
> **Seguridad de las Claves:** Una vez activado el cifrado, la pérdida de las llaves de recuperación y las contraseñas de los usuarios imposibilitará por completo la recuperación de la información. Asegúrese de incluir el directorio `/var/www/html/data/files_encryption/` en las rutinas de backup externo en frío.

---

## ⚙️ 3. Instalación de Tesseract OCR y Programación de Cron fuera de Horario Laboral

Para habilitar la búsqueda indexada de texto en PDFs escaneados de asistencia y salud sin saturar el procesador del VPS en horario laboral, implemente los siguientes pasos:

### Paso A: Incluir Tesseract OCR en el Contenedor
La imagen oficial de Nextcloud no incluye Tesseract. Para resolver esto sin crear repositorios complejos, modifique su despliegue para usar una imagen personalizada o configure la instalación en el arranque del compose modificando el servicio `app` en su compose temporalmente, o bien, use una imagen que herede de nextcloud:

#### Opción Recomendada (Custom Dockerfile):
Cree un archivo llamado `Dockerfile` en el VPS con:
```dockerfile
FROM nextcloud:28-apache
RUN apt-get update && apt-get install -y \
    tesseract-ocr \
    tesseract-ocr-spa \
    && rm -rf /var/lib/apt/lists/*
```
Y reemplace `image: nextcloud:28-apache` en su `docker-compose.yml` por:
```yaml
build:
  context: .
  dockerfile: Dockerfile
```

### Paso B: Instalar la App de OCR en Nextcloud
1. Desde el panel de administración de Nextcloud, instale la aplicación **OCR** (de la categoría "Tools" o herramientas).
2. En los ajustes de administración de OCR, defina la ruta del binario de tesseract como `/usr/bin/tesseract` y el idioma por defecto a **Spanish (spa)**.

### Paso C: Configurar la Tarea Programada Fuera de Horario Laboral
Por defecto, Nextcloud procesa las tareas en segundo plano usando AJAX (ejecuta una tarea pequeña con cada clic del usuario) o un Cron regular (cada 5 minutos). 
Para evitar picos de CPU del OCR en horario laboral:

1. **Configurar el Cron de Nextcloud a modo Sistema (Host):**
   Asegúrese de cambiar la configuración de tareas en segundo plano en **Settings -> Basic Settings -> Background jobs** a la opción **Cron**.
2. **Programar el procesamiento OCR intensivo en el Host (o VPS):**
   Las colas de OCR se acumulan en la cola de Nextcloud. En lugar de procesarlas inmediatamente, podemos ejecutar la indexación y procesamiento de archivos en bloque mediante una tarea `cron` programada en el sistema operativo del host VPS a las **02:00 AM (hora local de baja demanda)**.

   Abra el editor de cron del host:
   ```bash
   sudo crontab -e
   ```
   Añada la siguiente línea al final del archivo para forzar la ejecución del procesamiento en bloque de OCR y la limpieza de caché de manera diaria a las 2 AM:
   ```micro
   0 2 * * * docker exec -u www-data hrinser-nextcloud-app php occ ocr:process
   ```
   *(Nota: Reemplace `hrinser-nextcloud-app` con el nombre del contenedor de su aplicación Nextcloud activa).*

---

## 🛠️ Monitoreo y Mantenimiento Preventivo

Para verificar que el sistema no está experimentando cuellos de botella de I/O o consumo excesivo de RAM:
* Ejecute en la consola del VPS:
  ```bash
  docker stats
  ```
  Esto le permitirá vigilar en tiempo real que los contenedores `hrinser-nextcloud-app` y `hrinser-nextcloud-db` se mantengan estables dentro de los límites de 2 GB asignados.
* Supervise el uso del procesador y el I/O Wait mediante:
  ```bash
  top
  ```
  Ponga especial atención al porcentaje `%wa`. Si supera de forma sostenida el 15%, evalúe migrar el volumen de base de datos a un almacenamiento con IOPS garantizados o SSD/NVMe dedicados.
