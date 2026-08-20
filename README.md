# clasificador-commits-ai-agent

Aplicación de inteligencia artificial que funciona localmente mediante Ollama y Python, utilizando Docker para proporcionar un entorno de desarrollo portable entre diferentes equipos. La solución integra una API REST con FastAPI, un modelo de inteligencia artificial local y PostgreSQL para el almacenamiento de información.

## Integrante y perfil de hardware

- **Integrante:** DYLAN.
- **Perfil de hardware utilizado:** Perfil B (ollama pull qwen2.5:0.5b).
- **RAM:** 8 GB.
- **Procesador:** AMD Ryzen 5 4600H con gráficos Radeon integrados.
- **Almacenamiento:** SSD de 250 GB.

## Requisitos mínimos

### Hardware

- Procesador de 4 núcleos o equivalente.
- 8 GB de RAM.
- Al menos 5 GB de espacio libre para Docker, imágenes, dependencias y el modelo local.
- Conexión a Internet para descargar las dependencias, imágenes Docker y el modelo durante la instalación.

### Software

- Linux de 64 bits.
- Docker Engine.
- Docker Compose.
- Git.
- Python 3.12 o superior.
- Ollama.

## Instalación

### 1. Clonar el repositorio

```bash
git clone git@github.com:D714NRO2O/clasificador-commits-ai-agent.git
cd clasificador-commits-ai-agent
```

### 2. Preparar el entorno Linux

Ejecutar el script de aprovisionamiento:

```bash
chmod +x setup.sh
./setup.sh
```

El script instala las utilidades necesarias, Docker Engine, Docker Compose
y Ollama cuando no están instalados.

Al finalizar, cierre y vuelva a abrir la terminal para aplicar los permisos
del grupo `docker`.

### 3. Crear y activar el entorno virtual

Desde la raíz del proyecto:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Mantenga el entorno virtual activo durante los pasos que utilicen Python
o `pip`.

### 4. Instalar las dependencias de Python

Con el entorno virtual activo:

```bash
pip install -r requirements.txt
```

### 5. Configurar las variables de entorno

Crear el archivo `.env` a partir del archivo de ejemplo incluido en el
proyecto:

```bash
cp .env.example .env
```

Abrir el archivo:

```bash
nano .env
```

Para reproducir la configuración utilizada en el ejercicio, completar el
archivo con:

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=iadb
DB_USER=app_ia
DB_PASSWORD=claveApp456
DB_ADMIN_PASSWORD=<debes_agregar_contraseña_del_usuario_administrador>
OLLAMA_URL=http://localhost:11434/api/generate
MODELO_OLLAMA=qwen2.5:0.5b
MOTOR_POR_DEFECTO=eco
```

La credencial `claveApp456`, utilizada como valor de `DB_PASSWORD`, se
encuentra actualmente hardcodeada en un archivo de configuración que ya
está expuesto en el repositorio. Las demás credenciales y valores de
configuración utilizados por el proyecto también se encuentran definidos
en los archivos incluidos en el repositorio. Estos valores se documentan
aquí únicamente para facilitar la reproducción de la solución como parte
del ejercicio académico.

La variable `DB_ADMIN_PASSWORD` debe completarse con la contraseña del
usuario administrador de PostgreSQL.

> **Importante:** estos valores corresponden a la configuración utilizada
> durante el ejercicio académico. Las credenciales no deben reutilizarse
> en entornos reales o de producción. En un entorno real, las credenciales
> deben mantenerse fuera del código fuente, del README y del control de
> versiones. El archivo `.env` utilizado localmente tampoco debe publicarse
> en el repositorio.

### 6. Preparar el modelo local

Comprobar que Ollama esté disponible:

```bash
ollama --version
```

Descargar el modelo utilizado por la aplicación:

```bash
ollama pull qwen2.5:0.5b
```

Comprobar que el modelo esté disponible:

```bash
ollama list
```

### 7. Construir e iniciar la solución

Desde la raíz del proyecto:

```bash
docker compose up -d --build
```

Comprobar el estado de los servicios:

```bash
docker compose ps
```

## Verificación y pruebas

La API expone tres endpoints principales:

- `GET /health`
- `POST /clasificar`
- `GET /inferencias`

También se puede acceder a la documentación interactiva de FastAPI en:

```text
http://localhost:8000/docs
```

### 1. Verificar el estado de la API

```bash
curl http://localhost:8000/health
```

La respuesta esperada es un código `200` con el estado de la API y de
la conexión con PostgreSQL:

```json
{
  "estado": "ok",
  "base_datos": "ok"
}
```

### 2. Probar `/clasificar`

Probar con el motor `eco`:

```bash
curl -X POST http://localhost:8000/clasificar \
  -H "Content-Type: application/json" \
  -d '{"texto":"agrega el endpoint de historial","motor":"eco"}'
```

También se puede probar el modelo local mediante Ollama:

```bash
curl -X POST http://localhost:8000/clasificar \
  -H "Content-Type: application/json" \
  -d '{"texto":"corrige el error de conexión a la base de datos","motor":"ollama"}'
```

La respuesta contiene el motor utilizado, el modelo, la entrada, el
resultado obtenido y la latencia de la operación.

### 3. Consultar `/inferencias`

```bash
curl http://localhost:8000/inferencias
```

La API devuelve las últimas inferencias registradas en PostgreSQL.

## Solución de problemas

### 1. Docker mostró `permission denied`

Al ejecutar Docker se presentó un error de permisos al intentar acceder al
daemon. El usuario actual no tenía autorización para comunicarse con el
servicio de Docker sin utilizar privilegios de administrador.

Se solucionó agregando el usuario al grupo `docker`:

```bash
sudo usermod -aG docker $USER
```

Después se aplicó la pertenencia al nuevo grupo:

```bash
newgrp docker
```

Finalmente se verificó que el usuario pudiera ejecutar contenedores
directamente, sin utilizar `sudo`.

### 2. El contenedor de la API no tenía instalado `curl`

Al intentar comprobar desde el contenedor la conectividad con Ollama
mediante `curl`, la ejecución falló porque la imagen de la API no incluye
esa herramienta.

En lugar de modificar innecesariamente la imagen Docker para instalar
`curl`, se utilizó Python con `requests`, dependencia que ya utiliza la
aplicación:

```bash
docker exec api-ia python -c "import requests; r=requests.get('http://host.docker.internal:<puerto_ollama>/api/tags', timeout=5); print(r.status_code); print(r.text)"
```

Esto permitió comprobar la conectividad sin modificar la imagen de
ejecución.

### 3. La API no podía conectarse con Ollama desde Docker

La API devolvía un error de conexión al intentar comunicarse con el
servicio de Ollama.

La causa era que Ollama estaba escuchando únicamente en la interfaz local,
por lo que el contenedor no podía acceder al servicio del host.

Se configuró Ollama para aceptar conexiones desde el entorno Docker y se
verificó posteriormente que el servicio estuviera disponible en el puerto
configurado para Ollama.

### 4. Ruff detectó un manejo de excepciones demasiado general

Durante la verificación de calidad con Ruff se detectó que el endpoint
`/health` capturaba errores mediante la excepción general `Exception`.

Se corrigió el manejo de errores reemplazando `Exception` por
`psycopg2.Error`, ya que la operación que puede fallar en ese bloque
corresponde a PostgreSQL y la aplicación ya utiliza `psycopg2` para
establecer la conexión con la base de datos.

Con este cambio, el endpoint mantiene la respuesta `503` cuando
PostgreSQL no está disponible, pero evita capturar de forma indiscriminada
otras excepciones de Python. No fue necesario modificar la configuración
de Ruff, porque el problema se resolvió directamente en el código.

### 5. Las pruebas de `pytest` no podían conectarse inicialmente con PostgreSQL

Al ejecutar las pruebas de la API desde el entorno Python, PostgreSQL no
podía ser localizado inicialmente mediante la configuración utilizada por
las pruebas.

Para comprobar la conexión con el contenedor de PostgreSQL se ejecutaron
las pruebas proporcionando temporalmente el host y el puerto mediante
variables de entorno:

```bash
DB_HOST=<HOST_INTERNO_POSTGRESQL> DB_PORT=<PUERTO_POSTGRESQL> python -m pytest -v
```

Esto permitió que `pytest` utilizara la dirección correspondiente al
servicio de PostgreSQL durante la ejecución y que las pruebas pudieran
comunicarse correctamente con la base de datos.

Posteriormente, en el flujo de integración continua, PostgreSQL se
configuró como servicio de pruebas y las variables de conexión se
definieron en el propio workflow, evitando depender de una dirección IP
interna fija.


## Rendimiento

El modelo local utilizado es `qwen2.5:0.5b`, con un tamaño aproximado de
397 MB en disco.

Durante la caracterización del modelo se obtuvieron los siguientes
resultados:

| Métrica | Resultado |
|---|---:|
| Promedio | 2029 ms |
| Mediana | 2026 ms |
| p95 | 2073 ms |
| Errores | 0 % |

En la prueba de carga del motor `eco` se obtuvo:

| Métrica | Resultado |
|---|---:|
| p95 | 15,9 ms |
| Errores | 0 % |

La principal diferencia de rendimiento se encuentra en la inferencia del
modelo local, que requiere más tiempo y recursos de CPU y memoria que el
motor `eco`.

## Detener la solución

Para detener los servicios:

```bash
docker compose down
```

Para volver a iniciar la solución:

```bash
docker compose up -d
```
