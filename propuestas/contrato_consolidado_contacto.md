# Contrato Consolidado Frontend-Backend: Formulario de Contacto HRINSER

Este documento consolida el acuerdo técnico entre UI/UX y Seguridad DevOps para el formulario de captación de leads de HRINSER.

---

## 1. Campos del Formulario y Reglas de Validación

| Campo UI | Nombre JSON | Tipo | Requerido | Validación Frontend / UI | Validación de Seguridad (Backend) | Mensaje de Error UX Sugerido |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Nombre Completo** | `fullName` | `string` | Sí | Mín. 2, máx. 80 caracteres. Solo letras y espacios. | Sanitización HTML. Regex: `/^[a-zA-ZáéíóúÁÉÍÓÚñÑüÜ\s'\-]+$/` | "Por favor, introduce tu nombre completo (solo letras)." |
| **RUT Empresa / Persona** | `rut` | `string` | Sí | Formato estándar chileno (ej. 12.345.678-9 o 12345678-9). | Limpieza de puntos y guion. Algoritmo Módulo 11 obligatorio. Máx. 12 caract. | "Introduce un RUT válido (ej. 12345678-9 o 12.345.678-K)." |
| **Empresa** | `companyName` | `string` | Sí | Mín. 2, máx. 100 caracteres. | Sanitización HTML. Escape de caracteres especiales. | "Por favor, introduce el nombre de tu empresa." |
| **Correo Electrónico** | `email` | `string` | Sí | Formato email estándar. | Formato RFC 5322. Máx. 254 caracteres. Validación sintáctica estricta. | "Introduce una dirección de correo válida (ej. nombre@empresa.com)." |
| **Teléfono** | `phoneNumber` | `string` | No (Opcional) | Formato E.164 (ej. +56912345678). | Regex: `/^\+?[1-9]\d{1,14}$/`. Máx. 15 caracteres. | "Por favor, introduce un número de teléfono válido (ej. +56912345678)." |
| **Cant. Trabajadores** | `employeeCountRange` | `string` (enum) | Sí | Dropdown premium. | Validación estricta contra whitelist:<br>`["1-10", "11-50", "51-200", "201-500", "500+"]` | "Selecciona el rango de trabajadores de tu empresa." |
| **Checkbox Privacidad** | `privacyAccepted` | `boolean` | Sí | Debe ser `true`. | Debe ser explícitamente `true`. | "Debes aceptar la Política de Privacidad para continuar." |

---

## 2. Definición de Endpoints y Payloads

### A. POST `/api/v1/contact`
**Payload de Entrada:**
```json
{
  "fullName": "Francisca Pérez",
  "rut": "12.345.678-9",
  "companyName": "HRINSER Chile",
  "email": "francisca.perez@hrinser.com",
  "phoneNumber": "+56987654321",
  "employeeCountRange": "51-200",
  "privacyAccepted": true
}
```

### B. Respuesta Exitosa (201 Created)
```json
{
  "success": true,
  "message": "Formulario de contacto recibido correctamente.",
  "data": {
    "referenceId": "CONT-2026-98741",
    "receivedAt": "2026-07-14T15:20:00Z"
  }
}
```

### C. Respuesta de Error de Validación (400 Bad Request - Híbrido RFC 7807)
Para mantener compatibilidad con el mapeo directo de UI propuesto por UX y seguir el estándar de seguridad de API:
```json
{
  "type": "https://api.hrinser.cl/errors/validation-failed",
  "title": "Error de validación en los datos enviados",
  "status": 400,
  "detail": "Uno o más campos del formulario no cumplen con los requisitos de seguridad y formato.",
  "instance": "/api/v1/contact",
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Uno o más campos contienen errores de validación.",
    "details": [
      {
        "field": "rut",
        "message": "El RUT ingresado no es válido o no cumple con el algoritmo Módulo 11."
      },
      {
        "field": "email",
        "message": "La dirección de correo electrónico no cumple con el formato requerido."
      }
    ]
  }
}
```

---

## 3. Seguridad de Infraestructura y Rate Limiting

1. **Límites de Peticiones (Rate Limiting):**
   - **Por IP**: Máximo de 5 solicitudes por minuto por dirección IP única.
   - **Por Identidad (RUT / Correo)**: Máximo de 3 solicitudes por hora para evitar ataques dirigidos de denegación de servicio de correo (mail-bombing).
2. **Encabezados HTTP en Respuestas de Rate-Limit (429 Too Many Requests):**
   - `Retry-After`: Tiempo de espera recomendado en segundos.
   - `X-RateLimit-Limit`: Límite máximo de la ventana.
   - `X-RateLimit-Remaining`: Solicitudes restantes.
   - `X-RateLimit-Reset`: Timestamp Unix del fin del bloqueo.
3. **Payload Limit:**
   - Se restringe el tamaño total de la petición POST a un máximo de 50 KB para prevenir ataques por desbordamiento o buffer exhaust.
4. **Sanitización del Backend:**
   - Todos los inputs textuales son sanitizados (escapado de caracteres `<`, `>`, `&`, `"`, `'`) antes de cualquier persistencia o envío de correos electrónicos.
