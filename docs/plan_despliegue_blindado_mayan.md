# 🗺️ Plan de Despliegue Blindado para Mayan EDMS (Elestio)
### Enfoque de Seguridad y Cumplimiento Normativo (Ley 21.719)

Este documento detalla la planificación técnica y de seguridad necesaria para desplegar Mayan EDMS de forma segura en la nube usando Elestio, mitigando los riesgos legales de la Ley 21.719 de Protección de Datos Personales en Chile.

---

## 📦 Paso 1: Configurar la Cuenta e Infraestructura en Elestio

### 1. Registro Seguro y Control de Acceso
*   **Cuenta Corporativa:** Crear la cuenta a nombre de la persona jurídica (Gestión Comercial SpA) utilizando un correo corporativo dedicado como `seguridad@hrinser.cl`.
*   **Doble Factor de Autenticación (2FA) Obligatorio:** Activar 2FA (Google Authenticator o compatible) en la cuenta de Elestio de todos los administradores autorizados.
*   **Restricción de Acceso SSH:** Configurar en el firewall de Elestio para que el acceso SSH al servidor requiera llave privada (sin contraseña) y restringir el puerto SSH (22) únicamente a las IPs de administración.

### 2. Dimensionamiento del Servidor y Control de Recursos (OOM Prevention)
*   **Tamaño del Servidor:** Elegir un plan en Elestio de mínimo **4 vCPUs y 8 GB o 16 GB de RAM** con almacenamiento NVMe/SSD de **200 GB**.
*   **Cifrado en Reposo:** Habilitar la opción de cifrado de volumen de almacenamiento (Encryption at Rest) ofrecida por el proveedor de cloud subyacente.
*   **Configuración de Límites en Docker (Crítico para OCR):**
    En la pestaña de configuración del servicio en Elestio (o editando el `docker-compose.yml`), limitar los recursos de los contenedores para evitar que el procesamiento de OCR (Tesseract) consuma toda la memoria física del host y cause caídas de PostgreSQL.
    ```yaml
    services:
      mayan-app:
        deploy:
          resources:
            limits:
              cpus: '3.0'
              memory: 6G
            reservations:
              memory: 2G
    ```

---

## 🌐 Paso 2: Vinculación del Dominio y Seguridad Perimetral

### 1. Vinculación del Subdominio
*   Crear el registro CNAME o A apuntando a la dirección IP provista por Elestio desde el panel de administración del dominio `hrinser.cl` (ej: `documentos.hrinser.cl`).
*   Configurar el subdominio en el panel de Elestio. El proxy reverso integrado generará automáticamente el certificado **SSL/TLS de Let's Encrypt**.

### 2. Endurecimiento de TLS
*   Forzar la redirección permanente de HTTP a HTTPS.
*   Desactivar protocolos TLS antiguos (1.0 y 1.1) y permitir únicamente **TLS 1.2 y TLS 1.3** para encriptar los datos en tránsito.

---

## 👥 Paso 3: Configurar el Aislamiento Multi-empresa (Muros Lógicos)

Mayan EDMS no tiene multi-tenancy nativo. Para separar de forma segura a las empresas clientes, implementaremos aislamiento lógico mediante ACLs estrictas.

### 1. Estructura de Roles y Grupos
*   **Grupos en Mayan:** Crear un grupo por cada cliente (ej: `Grupo_Cliente_Alfa`, `Grupo_Cliente_Beta`).
*   **Roles:** Crear un rol por cliente asignado a su respectivo grupo.
*   **Usuarios:** Cada usuario de un cliente se registra en Mayan y se añade únicamente a su respectivo grupo.

### 2. Listas de Control de Acceso (ACLs) y Document Types
*   **Tipos de Documento por Cliente:** Crear tipos de documentos específicos por cliente (ej: `Alfa_Liquidaciones`, `Beta_Liquidaciones`).
*   **Configuración de ACLs:**
    1.  Ir a la configuración del Rol `Rol_Cliente_Alfa`.
    2.  Asociar los permisos de "Ver documento", "Descargar documento" y "Buscar" únicamente a los tipos de documento pertenecientes a la empresa Alfa.
    3.  **Denegar por defecto:** Asegurarse de que el rol de un cliente no tenga ningún permiso (ni de lectura ni de metadatos) sobre los tipos de documentos de otros clientes.

### 3. Autenticación de Usuarios
*   Activar la opción de **Doble Factor de Autenticación (2FA) obligatorio en Mayan EDMS** para todos los usuarios que ingresen a la plataforma.

---

## 📂 Paso 4: Ingesta Controlada y Monitoreo de los 100 GB

Para evitar la saturación del servidor con el procesamiento inicial de OCR de los 100 GB históricos, el proceso se realizará en lotes.

### 1. Ingesta por Lotes (Batching)
*   No subir los 100 GB en un solo bloque.
*   Dividir el contenido en lotes de máximo 10 GB o por carpetas de empresas.
*   Subir el lote 1, verificar que los workers de Celery/Tesseract comiencen el procesamiento y monitorear el consumo de CPU/RAM en Elestio. Una vez completado el procesamiento del lote, proceder con el siguiente.

### 2. Monitoreo de Recursos
*   Monitorear el uso de disco y memoria desde el dashboard de Elestio durante el proceso.
*   Configurar alertas de uso de disco superior al 85%.

---

## 📊 Paso 5: Gestión de Backups, Auditoría y Cumplimiento de la Ley 21.719

### 1. Copias de Seguridad Encriptadas
*   Configurar backups diarios automatizados en Elestio en la madrugada.
*   **Destino seguro:** Almacenar los backups en un storage S3 externo compatible configurado con llaves de acceso restringidas y cifrado del lado del servidor.

### 2. Logs de Auditoría Inmutables
*   Activar el sistema de **Event Logging** interno de Mayan EDMS.
*   Asegurar el registro persistente de los eventos críticos:
    *   Inicio de sesión y fallas de autenticación (IP y origen).
    *   Descarga de documentos confidenciales.
    *   Modificación de permisos y ACLs.
*   Establecer una política de retención de logs de auditoría de mínimo 2 años.
