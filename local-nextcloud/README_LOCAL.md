# 🚀 Guía de Nextcloud Local (Pruebas)

Este directorio contiene la configuración mínima necesaria para levantar y evaluar **Nextcloud** localmente de forma rápida y sencilla.

---

## 📋 Requisitos Previos
*   Tener **Docker** y **Docker Compose** instalados en tu sistema.

---

## ⚡ Cómo Iniciar Nextcloud
1.  Abre una terminal y entra en este directorio:
    ```bash
    cd /home/dano/Documentos/DMS-proyecto/local-nextcloud
    ```
2.  Levanta el contenedor en segundo plano (background):
    ```bash
    docker compose up -d
    ```
3.  Espera unos 10-15 segundos a que Nextcloud termine de inicializarse.

---

## 🌐 Cómo Acceder y Configurar
1.  Abre tu navegador de internet y entra a la siguiente dirección:
    👉 **[http://localhost:8080](http://localhost:8080)**
2.  **Pantalla de Bienvenida:**
    *   **Crear una cuenta de usuario administrador:** Define el nombre de usuario (ej: `admin`) y una contraseña segura.
    *   **Base de datos:** Por defecto viene seleccionado **SQLite**, déjalo así para esta prueba (no requiere configuraciones adicionales).
3.  Haz clic en **"Instalar"** (o *Finish setup*).
4.  **Idioma Español:**
    *   Nextcloud detectará el idioma de tu navegador automáticamente.
    *   Si por algún motivo aparece en inglés, ve a tu avatar/círculo en la esquina superior derecha -> **Settings** -> **Personal Info** -> **Language** y cámbialo a **Español**.

---

## 🛑 Cómo Detener Nextcloud
Cuando termines de probar la aplicación, puedes apagar los servicios ejecutando:
```bash
docker compose down
```
*Nota: Los datos y archivos que subas no se borrarán; están guardados localmente en la carpeta `nextcloud_data` de este directorio.*
