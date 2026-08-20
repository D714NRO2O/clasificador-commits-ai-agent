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


## Endpoints

| Método | Ruta | Parámetros de entrada | Respuesta | Códigos de error |
|---|---|---|---|---|
| GET | `/health` | Ninguno | Estado de la API y conexión con PostgreSQL | `503` si PostgreSQL no está disponible |
| POST | `/clasificar` | `texto` obligatorio, `motor` opcional | Motor utilizado, modelo, entrada, tipo clasificado y latencia | `400` si el motor no es `eco` u `ollama` |
| GET | `/inferencias` | `limite` opcional | Lista de las últimas inferencias registradas | No tiene manejo de errores específico |

### POST /clasificar

Ejemplo de solicitud:

```json
{
  "texto": "agrega el endpoint de historial",
  "motor": "eco"
}```


## Modelo de datos

La aplicación utiliza la tabla `inferencias` para registrar las
clasificaciones realizadas.

| Campo | Tipo | Significado |
|---|---|---|
| `id` | SERIAL | Identificador único de la inferencia |
| `fecha` | TIMESTAMP | Fecha y hora en que se registró la inferencia |
| `motor` | VARCHAR(20) | Motor utilizado para clasificar |
| `modelo` | VARCHAR(120) | Modelo o versión utilizada |
| `entrada` | TEXT | Mensaje de commit recibido |
| `salida` | TEXT | Categoría obtenida |
| `latencia_ms` | INTEGER | Tiempo empleado por la clasificación en milisegundos |

El usuario `app_ia` tiene permisos para conectarse a la base de datos,
utilizar el esquema `public`, consultar e insertar registros en
`inferencias` y utilizar la secuencia `inferencias_id_seq`.


## Respaldo y restauración

### Crear el respaldo

Crear el directorio para almacenar temporalmente los respaldos:

```bash
mkdir -p backups
```

Generar el respaldo de la base de datos `iadb`:

```bash
docker compose exec -T db pg_dump -U postgres iadb > backups/respaldo_$(date +%F).sql
```

### Simular una pérdida de datos

Consultar la cantidad de registros antes de realizar la prueba:

```bash
docker compose exec -T db psql -U postgres -d iadb -c "SELECT COUNT(*) FROM inferencias;"
```

Vaciar los registros de la tabla `inferencias` para simular una pérdida de información:

```bash
docker compose exec -T db psql -U postgres -d iadb -c "TRUNCATE inferencias;"
```

Comprobar que la tabla quedó vacía:

```bash
docker compose exec -T db psql -U postgres -d iadb -c "SELECT COUNT(*) FROM inferencias;"
```

### Restaurar el respaldo

Restaurar los datos desde el archivo de respaldo:

```bash
cat backups/respaldo_$(date +%F).sql | docker compose exec -T db psql -U postgres -d iadb
```

Comprobar que los registros fueron recuperados:

```bash
docker compose exec -T db psql -U postgres -d iadb -c "SELECT COUNT(*) FROM inferencias;"
```

### Resultado de la prueba

El procedimiento fue probado correctamente durante el paso 1. La tabla
`inferencias` contenía **867 registros** antes de simular la pérdida.
Después del `TRUNCATE` quedó en **0 registros** y, tras restaurar el
respaldo, se recuperaron nuevamente **867 registros**.

Los archivos generados en `backups/` están excluidos del repositorio
mediante la regla `backups/` de `.gitignore`.


## Decisiones de diseño y limitaciones

### Dockerfile multi-etapa

El `Dockerfile` utiliza dos etapas. La primera utiliza `python:3.12-slim`
para instalar las dependencias en `/install`. La segunda utiliza una
imagen limpia de `python:3.12-slim` y copia únicamente las dependencias
instaladas y el código de la aplicación.

Esta separación evita mantener en la imagen final elementos utilizados
únicamente durante la construcción y mantiene separado el entorno de
construcción del entorno de ejecución.

### Usuario sin privilegios

La imagen crea el usuario `appuser` con UID `1001` y la aplicación se
ejecuta mediante `USER appuser` en lugar de utilizar `root`.

Esto limita los permisos del proceso de la API dentro del contenedor y
reduce el impacto de un posible compromiso de la aplicación.

### Motor `eco`

El motor `eco` funciona mediante reglas definidas en la aplicación y
sirve como línea base de clasificación sin utilizar un modelo de
lenguaje.

Esta alternativa permite resolver mensajes sencillos con una latencia
mucho menor. En la prueba P-08 obtuvo un p95 de **15,9 ms** y **0 % de
errores**, mientras que la caracterización del modelo local en P-09
obtuvo un promedio de **2029 ms** y un p95 de **2073 ms**.

### Privilegios mínimos en PostgreSQL

La aplicación utiliza el usuario `app_ia` para sus operaciones normales
en PostgreSQL y no utiliza el usuario administrador `postgres`.

El usuario `app_ia` tiene permisos para conectarse a la base de datos,
utilizar el esquema `public`, consultar e insertar registros en la
tabla `inferencias` y utilizar la secuencia `inferencias_id_seq`.

Esto limita las operaciones que puede realizar la aplicación sobre la
base de datos.

### Limitaciones conocidas

- El motor `eco` depende de reglas simples y puede producir
  clasificaciones incorrectas cuando el mensaje no coincide con los
  patrones definidos.
- El modelo local presenta una latencia considerablemente mayor que el
  motor `eco`.
- La ejecución de Ollama depende de los recursos disponibles en el
  equipo, especialmente CPU y memoria.
- La aceleración mediante GPU depende del hardware disponible y del
  soporte del entorno de ejecución.
- El modelo utilizado es `qwen2.5:0.5b`, por lo que sus capacidades son
  limitadas frente a modelos de mayor tamaño.
- Cuando la API se ejecuta dentro de Docker y Ollama directamente en el
  host, el contenedor necesita conectividad con el servicio de Ollama
  para utilizar el motor `ollama`.
