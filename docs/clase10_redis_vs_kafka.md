# Clase 10 - Pregunta 3 del cuestionario: Redis vs. Kafka

## Lo que hace cada uno

**Redis pub/sub** (lo que usa nuestro channel layer) reparte cada mensaje a los que están
suscriptos **en ese momento** y no guarda nada. Si un usuario estaba desconectado cuando
se creó una tarea, ese aviso se perdió: la garantía es "como máximo una vez".

**Apache Kafka** es un registro distribuido de eventos: cada evento se escribe en disco,
en orden, y queda guardado por días o meses. Cada consumidor lleva la cuenta de hasta dónde
leyó, así que puede desconectarse y después seguir desde ahí, o volver a leer la historia
desde el principio. La garantía es "al menos una vez" (con confirmación).

| | Redis pub/sub | Kafka |
|---|---|---|
| Modelo | Pub/sub en vivo | Registro de eventos (log) |
| Persistencia | No | Sí, configurable (días/meses) |
| Si el consumidor está desconectado | Pierde el mensaje | Lo lee al volver |
| Volver a leer eventos viejos | No | Sí |
| Latencia | Muy baja | Baja, algo mayor |
| Complejidad de operar | Muy baja (un proceso) | Alta (cluster, particiones, réplicas) |
| Uso típico | Avisos en tiempo real, channel layers, caché | Grandes volúmenes, analítica, integración entre servicios |

## Mis conclusiones a partir de lo que hicimos en TaskFlow

1. **Para las notificaciones de TaskFlow alcanza con Redis.** Los avisos de `task_created`,
   `task_updated`, `task_completed` y la presencia solo tienen sentido para quien está
   conectado en ese momento. Si se pierden no pasa nada grave, porque los datos "de verdad"
   están en la base y el cliente, al reconectarse, los vuelve a pedir por la API REST
   (consistencia eventual). Guardar estos avisos en Kafka sería pagar complejidad por algo
   que no usamos.

2. **Redis además resuelve el problema que teníamos:** el `InMemoryChannelLayer` no
   permitía escalar horizontalmente. Con Redis todos los procesos de Daphne publican y
   escuchan en el mismo lugar, y se instala y opera muy fácil (un solo servicio, hoy
   localmente y en la Clase 12 en Docker con la variable `REDIS_URL`).

3. **Kafka tendría sentido si los eventos fueran datos importantes en sí mismos.** Por
   ejemplo, si quisiéramos un historial de auditoría de todo lo que pasó en cada proyecto,
   calcular estadísticas a partir de los eventos, o que otros servicios (emails, reportes)
   procesen cada `task_created` sin perderse ninguno aunque estén caídos un rato. Ahí la
   persistencia y la posibilidad de releer la historia son justo lo que se necesita.

4. **No son excluyentes.** Un sistema más grande podría usar Redis para el tiempo real
   hacia los navegadores y Kafka (o RabbitMQ / Redis Streams, si el volumen es menor) para
   la comunicación confiable entre servicios.

En resumen: Redis pub/sub es la herramienta para **avisar rápido** a quien está escuchando;
Kafka es la herramienta para **no perder** eventos y poder procesarlos o releerlos después.
Para lo que necesita TaskFlow hoy, Redis es la elección correcta.
