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
