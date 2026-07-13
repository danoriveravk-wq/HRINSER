# Informe de Evaluación Técnica y de Calidad (QA)
## Evaluación del Plan de Implementación Consolidado - Ecosistema DMS & CRM HRINSER
**Código de Evaluación:** HR-QA-EVAL-2026-001  
**Evaluador:** Senior Project Manager & Technical QA Lead  
**Fecha:** 13 de Julio de 2026  
**Estado de la Evaluación:** 🔴 PENDIENTE DE MEJORA  

---

## 1. Resumen Ejecutivo
Se ha realizado una revisión exhaustiva del documento `/home/dano/Documentos/DMS-proyecto/plan_implementacion_equipo.md` que detalla el plan de implementación de infraestructura, despliegue y hardening para el ecosistema digital de **HRINSER** (Landing Page, EspoCRM y Nextcloud DMS). 

La simplificación del plan eliminando a Cloudflare y delegando el enrutamiento y resolución DNS de forma nativa al gestor de Hostinger es **viable y simplifica la arquitectura de red**, eliminando intermediarios innecesarios para el tráfico hacia Elestio y Hostinger. Sin embargo, la revisión de control de calidad ha detectado **brechas de seguridad críticas (cabos sueltos)** en la gestión de credenciales, rate-limiting, validación en backend y segregación física de respaldos de llaves criptográficas, así como omisiones en el cumplimiento estricto de la **Ley 21.719 en Chile**.

---

## 2. Análisis de Configuración DNS y Direccionamiento (Hostinger a Elestio)
La configuración planteada en la **Fase 1 (Paso 2.1)** es técnicamente correcta y viable:
* **Enrutamiento Directo:** Apuntar registros A desde el editor de zona DNS de Hostinger directamente hacia las IPs del Shared Hosting y de las instancias VPS en Elestio es el estándar óptimo para esta arquitectura simplificada.
* **Aprovisionamiento SSL/TLS:** Es correcto que Hostinger gestione el SSL nativo para el dominio principal y el subdominio `crm.hrinser.cl`, y que Elestio maneje el ciclo de vida de Let's Encrypt de manera automática para las instancias del DMS al apuntar los registros A.
* **Viabilidad:** Excelente. Se reduce la latencia y se evitan problemas de configuración de certificados SSL de origen ("SSL Handshake Errors") que suelen ocurrir al usar Cloudflare mal configurado en modo flexible/estricto.

---

## 3. Evaluación de Brechas Críticas de QA y Cabos Sueltos

A pesar de que el plan propone soluciones estructuradas, se identificaron las siguientes brechas críticas de seguridad y automatización:

### A. Gestión de Contraseñas y Hardcoding (Crítico)
* **docker-compose.yml:** La contraseña de Redis (`SecurePass_Redis_32Char!`) se encuentra cableada (hardcoded) directamente en la directiva `command` y en el `healthcheck` de Redis (Líneas 407 y 420), así como en la variable `REDIS_HOST_PASSWORD` del servicio `app` (Línea 452). A pesar de tener un archivo `.env` (`nextcloud.env`), no se están interpolando estas variables en Compose, lo cual viola las buenas prácticas de GitOps y seguridad.
* **backup_nextcloud.sh:** La contraseña de la base de datos PostgreSQL (`SecurePass_Postgres_32Char!`) está harcodeada en la línea 666. Además, el script no carga el archivo de configuración `nextcloud.env` al ser ejecutado por Cron, por lo que depende enteramente del valor harcodeado en texto plano dentro del script.

### B. Rate-Limiting y Seguridad en la Pasarela de Leads (`submit_lead.php`)
* **Uso del Directorio `/tmp`:** El script utiliza `sys_get_temp_dir()` para almacenar el archivo JSON de tasa de peticiones. En Hostinger Shared Hosting, el directorio `/tmp` es compartido globalmente por múltiples cuentas de usuarios en el mismo servidor físico. Esto introduce riesgos de seguridad (colisiones de archivos, lectura/escritura no autorizada si los permisos son laxos).
* **Condición de Carrera (Race Condition):** La lógica de lectura y escritura del archivo JSON (`file_get_contents` seguido de `file_put_contents` sin bloqueos de archivo) tiene una condición de carrera inherente. Peticiones concurrentes desde la misma IP pueden evadir el límite debido a que PHP no serializa las operaciones mediante un candado exclusivo (`flock`).
* **Falta de Validación del RUT en el Backend:** El script sanitiza el RUT usando `FILTER_SANITIZE_SPECIAL_CHARS`, pero **no realiza ninguna validación estructural (Módulo 11) en el backend**. Un atacante puede saltarse el script JavaScript del frontend y enviar peticiones directas POST con RUTs inválidos o maliciosos, inyectando basura en EspoCRM.

### C. Segregación Física de Copias de Seguridad (Respaldos)
* **Brecha en TC-08 / Script de Backups:** El script `backup_nextcloud.sh` separa lógicamente los datos (`nextcloud_backup_DATE.tar.gz.gpg`) y las llaves de encriptación (`keys_backup_DATE.tar.gz.gpg`), lo cual es correcto. Sin embargo, **ambos archivos cifrados se almacenan localmente en el mismo Host y directorio físico (`/var/backups/nextcloud`)**. El plan no define la transmisión automática de la copia de llaves a una bóveda o servidor físicamente separado. Si el servidor host es comprometido, el atacante tendría acceso a ambos componentes (datos cifrados y llaves cifradas con GPG, aunque protegidas por frase de paso).

### D. Procedimiento de Restauración ante Desastres (Disaster Recovery)
* **Comando Interactivo de Restauración de BD:** En la Fase 7 (Paso 5), el comando para restaurar la base de datos es:
  `docker exec -i hrinser-nextcloud-db psql -U nextcloud_usr -d nextcloud_db < ...`
  Al no pasar la variable `PGPASSWORD` en el entorno del comando `docker exec`, `psql` solicitará la contraseña interactivamente, lo que rompe cualquier intento de automatización y detiene el flujo de restauración si no hay un operador presente.
* **Permisos del Volumen:** Al restaurar datos de Nextcloud en caliente en el Host de Docker, es común que se altere el propietario de los archivos. El manual no incluye una tarea explícita para asegurar la propiedad de los archivos en el contenedor (`chown -R www-data:www-data`), lo que puede causar errores *HTTP 500* por permisos denegados.

---

## 4. Cumplimiento de la Ley 21.719 en Chile
La nueva **Ley 21.719** (que entra en vigor pleno en diciembre de 2026) exige estándares muy altos para el tratamiento de datos personales, especialmente sensibles (como los datos de salud e ingresos que procesa HRINSER). 

### Aspectos Cumplidos en el Plan:
* **Seguridad y Cifrado:** El uso de cifrado a nivel de servidor (Server-Side Encryption) en Nextcloud y discos cifrados (LUKS) en Elestio cumple con el deber de seguridad (Art. 11).
* **Inmutabilidad y Auditoría:** La redirección de logs a syslog/stdout del host (`admin_audit`) previene alteraciones y asegura la trazabilidad de accesos (registro de actividades).
* **Aislamiento Multi-Tenant:** El aprovisionamiento de un VPS dedicado por cliente evita fugas de información cruzada.

### Brechas de Cumplimiento (Pendientes):
1. **Consentimiento Explícito e Informado:** El formulario de captura de leads de la Landing Page no incluye un campo de verificación (checkbox no pre-marcado) y enlace a la Política de Privacidad para que el usuario otorgue su consentimiento explícito de tratamiento de datos, lo cual es obligatorio bajo la Ley 21.719.
2. **Procedimiento de Ejercicio de Derechos ARCO:** El plan no describe cómo los titulares pueden ejercer sus derechos de Acceso, Rectificación, Cancelación y Oposición dentro del ecosistema DMS/CRM.
3. **Plan de Notificación de Brechas:** La ley exige notificar a la Agencia de Protección de Datos Personales (APDP) y a los afectados sobre cualquier brecha de seguridad de manera inmediata. Esta obligación debe formalizarse en el manual de recuperación ante desastres.

---

## 5. Veredicto Definitivo y Recomendaciones

### Veredicto: 🔴 PENDIENTE DE MEJORA

Para autorizar el paso del plan a la fase de **"LISTO PARA EJECUCIÓN"**, el equipo de desarrollo debe subsanar las siguientes observaciones obligatorias en el archivo de planificación:

### Recomendaciones de Modificación Técnica:

#### 1. Corregir Hardcoding de Contraseñas en Docker y Scripts:
* Reemplazar la contraseña de Redis en el `docker-compose.yml` usando variables de entorno interpoladas:
  ```yaml
  command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 180mb --maxmemory-policy allkeys-lru
  ```
* En el script `backup_nextcloud.sh`, cargar las variables de entorno de Nextcloud al inicio:
  ```bash
  if [ -f "/opt/hrinser-dms/nextcloud.env" ]; then
      export $(grep -v '^#' /opt/hrinser-dms/nextcloud.env | xargs)
  fi
  DB_PASS="${POSTGRES_PASSWORD}"
  ```

#### 2. Asegurar la Pasarela de Leads (`submit_lead.php`):
* Cambiar la ruta del archivo de rate limiting fuera del directorio temporal global `/tmp` a una carpeta privada en el home del usuario de Hostinger:
  ```php
  $limitFile = '/home/uXXXXX/rate_limits/rate_limit_' . md5($ip) . '.json';
  ```
  *(Asegurando que la carpeta `/home/uXXXXX/rate_limits` exista con permisos 700).*
* Implementar bloqueo de archivos (`flock`) al abrir y escribir el JSON de control de tasa para evitar condiciones de carrera.
* Duplicar la lógica de validación de RUT en PHP en el backend para rechazar leads maliciosos antes de llamar a EspoCRM.

#### 3. Garantizar la Segregación Física de Respaldos (TC-08):
* Modificar el paso final del script de backup para que envíe el archivo de llaves criptográficas (`keys_backup_DATE.tar.gz.gpg`) a un servidor de almacenamiento externo (ej: AWS S3, SFTP seguro secundario o bóveda de llaves) y luego lo elimine del VPS local, evitando la coincidencia física de datos y llaves en la misma máquina.

#### 4. Ajustar Comandos del Manual de Restauración:
* Modificar la línea de restauración de PostgreSQL para inyectar la contraseña sin provocar prompts interactivos:
  ```bash
  docker exec -i -e PGPASSWORD="SecurePass_Postgres_32Char!" hrinser-nextcloud-db psql -U nextcloud_usr -d nextcloud_db < ...
  ```
* Añadir el comando de corrección de permisos al final del proceso de restauración:
  ```bash
  docker exec -u root hrinser-nextcloud-app chown -R www-data:www-data /var/www/html
  ```

#### 5. Adaptación Legal Ley 21.719:
* Diseñar un checkbox obligatorio de aceptación de Política de Privacidad en la Landing Page.
* Añadir un protocolo básico de notificación de brechas de seguridad (dentro del manual de desastres) para alertar a la gerencia y activar el reporte ante la APDP de ser necesario.
