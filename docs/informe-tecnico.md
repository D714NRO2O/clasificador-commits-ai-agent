# Informe técnico del modelo local

| Dato | Cómo obtenerlo | Valor |
|------|----------------|-------|
| Perfil de hardware | Sección 2 de la guía | Perfil B (8 GB) |
| RAM total del equipo | `free -h` | 7,2 GiB |
| Modelo y etiqueta | `ollama list` | qwen2.5:0.5b |
| Tamaño en disco | `ollama list` | 397 MB |
| Latencia de 5 ejecuciones (ms) | `time curl ...` cinco veces | 2299, 1972, 1694, 1852 y 2162 ms |
| Latencia promedio | Promedio de las cinco | 1996 ms |
| RAM usada durante la inferencia | `free -h` mientras responde | 2,5 GiB (la memoria usada del sistema aumentó aproximadamente 0,5 GiB durante la inferencia)|
| Calidad percibida (1 a 5) | Su criterio, con una frase que lo justifique | 3/5 = El modelo responde de forma coherente, mantiene el contexto de la conversación y presenta una buena velocidad de respuesta de aprox 2 segundos para instrucciones sencillas. Sin embargo, muestra limitaciones para seguir algunas instrucciones específicas, como generar textos con un número exacto de palabras o interpretar correctamente operaciones matemáticas expresadas en lenguaje natural.|

## Sección de pruebas

| ID | Tipo | Qué se verifica | Resultado esperado | Obtenido | Estado |
|---|---|---|---|---|---|
| P-01 | Funcional | GET /health responde | Código 200 y estado ok | Código 200, `{"estado":"ok","base_datos":"ok"}` | OK |
| P-02 | Funcional | POST /clasificar con motor eco | Código 200 y tipo correcto | Código 200 y clasificación correcta | OK |
| P-03 | Funcional | Motor inválido | Código 400 | Código 400 | OK |
| P-04 | Acceso | Rol app_ia intenta DROP TABLE | Error de permisos | Operación rechazada por permisos | OK |
| P-05 | Conectividad | La API resuelve el host db | Devuelve una IP interna | La API resuelve `db` dentro de la red Docker | OK |
| P-06 | Disponibilidad | Reinicio del contenedor de BD | La API se recupera sola | PostgreSQL vuelve a estado `healthy` y `/health` responde correctamente | OK |
| P-07 | Persistencia | down y up conservan los datos | Los registros siguen existiendo | Las inferencias anteriores permanecen después de `docker compose down` y `up -d` | OK |
| P-08 | Carga | 10 usuarios sobre el motor eco | p95 < 800 ms y errores < 5 % | p95 = 15,9 ms; errores = 0 % | OK |
| P-09 | Caracterización | 10 inferencias con modelo | Promedio, mediana y p95 | Promedio = 2029 ms; mediana = 2026 ms; p95 = 2073 ms | OK |

### Análisis del cuello de botella

P-08 obtuvo un **p95 de 15,9 ms** con el motor `eco` y **0 % de errores**. P-09 obtuvo un **promedio de 2029 ms**, una **mediana de 2026 ms**, un **p95 de 2073 ms** y **0 % de errores** con `qwen2.5:0.5b`.

Ambas pruebas utilizan la misma API y registran las inferencias en PostgreSQL. La diferencia principal es el motor utilizado: P-08 clasifica mediante reglas locales, mientras P-09 ejecuta una inferencia con Ollama.

El aumento de aproximadamente **2 segundos por solicitud** aparece al utilizar el modelo. Por esto, el principal cuello de botella está en la **inferencia**, no en la API ni en PostgreSQL.

No se puede afirmar que la API o PostgreSQL no aporten latencia, porque no se midieron por separado, pero los resultados muestran que **no son los responsables principales del aumento observado entre P-08 y P-09**.

### Propuestas de mejora

1. **Mantener el modelo cargado en memoria:** configurar Ollama para
   mantener el modelo disponible entre solicitudes cuando el equipo tenga
   memoria suficiente. Esto evita el costo de cargar nuevamente el modelo
   y reduce la latencia de las solicitudes posteriores.

2. **Usar el motor `eco` como filtro previo:** resolver automáticamente con
   reglas los mensajes que puedan clasificarse de forma clara y enviar a
   Ollama únicamente los casos que requieran una interpretación más
   compleja. Esto disminuye la cantidad de inferencias y libera recursos
   de CPU y memoria.

3. **Controlar la concurrencia de las inferencias:** establecer un límite
   de solicitudes simultáneas hacia Ollama para evitar que varias
   inferencias compitan por los recursos del equipo. Esto permite
   mantener tiempos de respuesta más estables cuando aumenta la carga.

4. **Optimizar la ejecución según el hardware disponible:** utilizar
   modelos y configuraciones de inferencia acordes con los recursos del
   equipo. La cantidad de RAM, el rendimiento del procesador y la
   disponibilidad de una GPU compatible pueden afectar directamente el
   tiempo de inferencia. En equipos con mejores recursos puede utilizarse
   un modelo con mayor capacidad sin asumir el mismo impacto de latencia.

5. **Evaluar un modelo más pequeño o una cuantización más eficiente:**
   seleccionar un modelo que mantenga una calidad suficiente para la
   clasificación pero requiera menos recursos. Esto puede reducir el
   tiempo de generación y permitir ejecutar más inferencias con el mismo
   hardware.
