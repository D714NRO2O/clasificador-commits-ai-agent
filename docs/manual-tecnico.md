# Manual técnico

## Arquitectura

```text
Cliente (Navegador / Postman)
            |
            | HTTP
            v
FastAPI (Puerto 8000)
            |
      +-----+-----+
      |           |
      v           v
 Motor eco    Ollama
 (reglas)   Puerto 11434
      |           |
      +-----+-----+
            |
            v
 PostgreSQL
 Puerto 5432
```

## Seguridad

### Puertos expuestos

- Puerto 8000: API FastAPI.
- Puerto 11434: API REST de Ollama.
- Puerto 5432: PostgreSQL.

### Roles de la base de datos

- **postgres:** usuario administrador de la base de datos.
- **app_ia:** usuario de la aplicación con permisos únicamente para consultar e insertar datos en la tabla `inferencias`.

### Manejo de secretos

Las credenciales de la base de datos y la configuración del modelo se almacenan en el archivo `.env`, el cual está incluido en `.gitignore` para evitar que se publique en el repositorio.

### Qué hacer si se filtra una contraseña

Cambiar inmediatamente la contraseña comprometida, actualizar el archivo `.env` con la nueva credencial y revocar la contraseña anterior para impedir accesos no autorizados.
