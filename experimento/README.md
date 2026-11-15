# Experimento: replicación, consistencia y tolerancia a fallos en ClickHouse

Esta carpeta contiene el clúster local de ClickHouse y los comandos del experimento de la TA-IND-01.

## Entorno

- Windows con Docker Desktop (motor WSL 2) y PowerShell.
- Clúster de la receta oficial `cluster_1S_2R` del repositorio [ClickHouse/examples](https://github.com/ClickHouse/examples/tree/main/docker-compose-recipes/recipes/cluster_1S_2R), sin cambios. Su licencia (Apache 2.0) está en `cluster/LICENSE-receta-ClickHouse.txt`.
- Imágenes: `clickhouse/clickhouse-server:latest` y `clickhouse/clickhouse-keeper:latest-alpine`. La versión exacta usada es la que devuelve `SELECT version()`: en mi caso, 26.10.4.207.

| Contenedor | Rol | Puertos en el equipo |
|---|---|---|
| `clickhouse-01` | Réplica 1 del fragmento 01 | 8123 (HTTP), 9000 (TCP) |
| `clickhouse-02` | Réplica 2 del fragmento 01 | 8124 (HTTP), 9001 (TCP) |
| `clickhouse-keeper-01` a `03` | Coordinación de la replicación con Raft | 9181, 9182, 9183 |

## Estructura

| Ruta | Contenido |
|---|---|
| `cluster/` | `docker-compose.yaml` y configuración de los 5 contenedores (receta oficial) |
| `sql/05_latencia.sql` | 60 inserciones intercaladas para la prueba D |
| `resultados/` | Salidas guardadas del experimento |

## Cómo ejecutarlo

Todos los comandos se ejecutan en PowerShell **desde la carpeta raíz del repositorio**.

### 0. Levantar el clúster

```powershell
docker compose -f experimento\cluster\docker-compose.yaml up -d
docker compose -f experimento\cluster\docker-compose.yaml ps
```

### 1. Verificar el clúster

```powershell
docker exec clickhouse-01 clickhouse-client --query "SELECT version()"
docker exec clickhouse-01 clickhouse-client --format PrettySpace --query "SELECT cluster, shard_num, replica_num, host_name FROM system.clusters WHERE cluster = 'cluster_1S_2R'"
foreach ($k in "clickhouse-keeper-01","clickhouse-keeper-02","clickhouse-keeper-03") { "== $k"; docker exec clickhouse-01 clickhouse keeper-client -h $k -p 9181 -q "flwc srvr" | Select-String "Mode" }
```

### 2. Crear la base de datos y la tabla replicada

```powershell
docker exec clickhouse-01 clickhouse-client --query "CREATE DATABASE IF NOT EXISTS ta01 ON CLUSTER cluster_1S_2R"
docker exec clickhouse-01 clickhouse-client --query "CREATE TABLE IF NOT EXISTS ta01.eventos ON CLUSTER cluster_1S_2R (id UInt32, nodo String, mensaje String, creado DateTime DEFAULT now()) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/{uuid}', '{replica}') ORDER BY id"
docker exec clickhouse-01 clickhouse-client --query "CREATE TABLE IF NOT EXISTS ta01.eventos_dist ON CLUSTER cluster_1S_2R AS ta01.eventos ENGINE = Distributed('cluster_1S_2R', 'ta01', 'eventos', id)"
docker exec clickhouse-01 clickhouse-client --format PrettySpace --query "SELECT database, table, replica_name, is_leader, is_readonly, total_replicas, active_replicas FROM system.replicas WHERE database = 'ta01'"
```

### 3. Prueba A: replicación asíncrona (consistencia eventual)

La réplica 2 deja de descargar datos (`STOP FETCHES`) mientras la réplica 1 sigue recibiendo escrituras.

```powershell
docker exec clickhouse-01 clickhouse-client --query "INSERT INTO ta01.eventos (id, nodo, mensaje) VALUES (1, 'clickhouse-01', 'primer evento')"
Start-Sleep -Seconds 2; docker exec clickhouse-02 clickhouse-client --query "SELECT count() FROM ta01.eventos"
docker exec clickhouse-02 clickhouse-client --query "SYSTEM STOP FETCHES ta01.eventos"
docker exec clickhouse-01 clickhouse-client --query "INSERT INTO ta01.eventos (id, nodo, mensaje) SELECT number + 2, 'clickhouse-01', 'evento con fetches detenidos' FROM numbers(5)"
docker exec clickhouse-01 clickhouse-client --query "SELECT count() FROM ta01.eventos"
docker exec clickhouse-02 clickhouse-client --query "SELECT count() FROM ta01.eventos"
docker exec clickhouse-02 clickhouse-client --format PrettySpace --query "SELECT table, queue_size, inserts_in_queue, absolute_delay FROM system.replicas WHERE database = 'ta01'"
```

Luego se reanuda la copia:

```powershell
docker exec clickhouse-02 clickhouse-client --query "SYSTEM START FETCHES ta01.eventos"
docker exec clickhouse-02 clickhouse-client --query "SYSTEM SYNC REPLICA ta01.eventos"
docker exec clickhouse-01 clickhouse-client --query "SELECT count() FROM ta01.eventos"
docker exec clickhouse-02 clickhouse-client --query "SELECT count() FROM ta01.eventos"
docker exec clickhouse-02 clickhouse-client --format PrettySpace --query "SELECT table, queue_size, inserts_in_queue, absolute_delay FROM system.replicas WHERE database = 'ta01'"
```

Resultado obtenido (capturas `05` y `06`): la réplica 2 tenía 1 fila al inicio; con la copia detenida, la réplica 1 tenía 6 filas y la réplica 2 seguía con 1 (`queue_size 1`, `inserts_in_queue 1`, `absolute_delay 46`). Al reanudar la copia, las dos quedaron con 6 filas y la cola en 0.

### 4. Prueba B: caída de una réplica

```powershell
docker stop clickhouse-02
docker exec clickhouse-01 clickhouse-client --format PrettySpace --query "SELECT table, total_replicas, active_replicas FROM system.replicas WHERE database = 'ta01'"
docker exec clickhouse-01 clickhouse-client --query "INSERT INTO ta01.eventos (id, nodo, mensaje) VALUES (7, 'clickhouse-01', 'escrito con la replica 2 caida')"
docker exec clickhouse-01 clickhouse-client --query "INSERT INTO ta01.eventos (id, nodo, mensaje) SETTINGS insert_quorum = 2, insert_quorum_timeout = 10000 VALUES (8, 'clickhouse-01', 'requiere quorum de 2')"
docker exec clickhouse-01 clickhouse-client --query "SELECT count() FROM ta01.eventos"
```

Luego se enciende de nuevo la réplica 2:

```powershell
docker start clickhouse-02
Start-Sleep -Seconds 15
docker exec clickhouse-02 clickhouse-client --query "SYSTEM SYNC REPLICA ta01.eventos"
docker exec clickhouse-02 clickhouse-client --query "SELECT count() FROM ta01.eventos"
docker exec clickhouse-02 clickhouse-client --query "SELECT id, mensaje FROM ta01.eventos WHERE id >= 7 ORDER BY id"
docker exec clickhouse-01 clickhouse-client --format PrettySpace --query "SELECT table, total_replicas, active_replicas FROM system.replicas WHERE database = 'ta01'"
```

Resultado obtenido (capturas `07` y `08`): con la réplica 2 apagada quedó `active_replicas 1`. La escritura sin quórum (fila 7) se aceptó; la que pedía `insert_quorum = 2` (fila 8) se rechazó con el error 285 (`TOO_FEW_LIVE_REPLICAS`), y el conteo fue 7. Al volver, la réplica 2 tenía 7 filas, incluida la fila 7, y no la fila 8; `active_replicas` volvió a 2.

### 5. Prueba C: pérdida de quórum en ClickHouse Keeper

Se detienen 2 de los 3 nodos de Keeper, así que Raft pierde la mayoría.

```powershell
docker stop clickhouse-keeper-02 clickhouse-keeper-03
Start-Sleep -Seconds 40
docker exec clickhouse-01 clickhouse-client --format PrettySpace --query "SELECT table, is_readonly, is_session_expired FROM system.replicas WHERE database = 'ta01'"
docker exec clickhouse-01 clickhouse-client --query "SELECT count() FROM ta01.eventos"
docker exec clickhouse-01 clickhouse-client --query "INSERT INTO ta01.eventos (id, nodo, mensaje) VALUES (9, 'clickhouse-01', 'escrito sin quorum en Keeper')"
```

Luego se encienden de nuevo los dos nodos:

```powershell
docker start clickhouse-keeper-02 clickhouse-keeper-03
Start-Sleep -Seconds 40
docker exec clickhouse-01 clickhouse-client --format PrettySpace --query "SELECT table, is_readonly, is_session_expired FROM system.replicas WHERE database = 'ta01'"
docker exec clickhouse-01 clickhouse-client --query "INSERT INTO ta01.eventos (id, nodo, mensaje) VALUES (9, 'clickhouse-01', 'escrito con Keeper recuperado')"
docker exec clickhouse-01 clickhouse-client --query "SELECT id, mensaje FROM ta01.eventos WHERE id >= 9 ORDER BY id"
docker exec clickhouse-01 clickhouse-client --query "SELECT count() FROM ta01.eventos"
foreach ($k in "clickhouse-keeper-01","clickhouse-keeper-02","clickhouse-keeper-03") { "== $k"; docker exec clickhouse-01 clickhouse keeper-client -h $k -p 9181 -q "flwc srvr" | Select-String "Mode" }
```

Resultado obtenido (capturas `09` y `10`): sin mayoría en Keeper, la tabla quedó con `is_readonly 1` e `is_session_expired 1`. La lectura devolvió 7 filas, pero la escritura esperó 120 s y terminó con el error 159 (`TIMEOUT_EXCEEDED`, *Wait for async insert timeout*), porque desde la versión 26.3 las inserciones son asíncronas por defecto. Al recuperar Keeper, la tabla volvió a `is_readonly 0`, la nueva escritura se aceptó, la fila que falló no se guardó (conteo 8) y Raft eligió como líder a `clickhouse-keeper-01` (antes era `clickhouse-keeper-03`). Si la tabla siguiera en `is_readonly = 1`, se puede ejecutar `docker exec clickhouse-01 clickhouse-client --query "SYSTEM RESTART REPLICA ta01.eventos"`.

### 6. Prueba D: latencia de escritura con y sin quórum (PACELC)

Con todos los nodos funcionando. Cada inserción del script usa `async_insert = 0`, para medir solo el efecto del quórum.

```powershell
docker exec clickhouse-01 clickhouse-client --format PrettySpace --query "SELECT name, value FROM system.settings WHERE name IN ('async_insert', 'wait_for_async_insert')"
docker cp experimento\sql\05_latencia.sql clickhouse-01:/tmp/05_latencia.sql
docker exec clickhouse-01 clickhouse-client --queries-file /tmp/05_latencia.sql
docker exec clickhouse-01 clickhouse-client --query "SYSTEM FLUSH LOGS"
docker exec clickhouse-01 clickhouse-client --format PrettySpace --query "SELECT log_comment AS modo, count() AS inserciones, round(avg(query_duration_ms), 2) AS promedio_ms, quantileExact(0.5)(query_duration_ms) AS mediana_ms, quantileExact(0.95)(query_duration_ms) AS p95_ms, max(query_duration_ms) AS maximo_ms FROM system.query_log WHERE type = 'QueryFinish' AND log_comment IN ('sin_quorum', 'quorum_2') GROUP BY log_comment ORDER BY log_comment"
```

Resultado obtenido (captura `11` y `resultados/prueba-d-latencia.txt`): `async_insert` y `wait_for_async_insert` valen 1 por defecto. Sin quórum, las 30 inserciones tardaron en promedio 18,83 ms (mediana 19, p95 25, máximo 34); con quórum de 2, 77,23 ms (mediana 73, p95 100, máximo 111), unas 4,1 veces más.

Nota: en cada captura, el bloque de comandos empieza con `cls; whoami; hostname; Get-Date` (o `Get-Date -Format "dddd dd/MM/yyyy HH:mm:ss"`) para mostrar el usuario, el equipo y la hora.

### 7. Guardar los resultados

```powershell
docker exec clickhouse-01 clickhouse-client --query "SYSTEM FLUSH LOGS"
docker exec clickhouse-02 clickhouse-client --query "SYSTEM FLUSH LOGS"
docker exec clickhouse-01 clickhouse-client --format PrettySpace --query "SELECT log_comment AS modo, count() AS inserciones, round(avg(query_duration_ms), 2) AS promedio_ms, quantileExact(0.5)(query_duration_ms) AS mediana_ms, quantileExact(0.95)(query_duration_ms) AS p95_ms, max(query_duration_ms) AS maximo_ms FROM system.query_log WHERE type = 'QueryFinish' AND log_comment IN ('sin_quorum', 'quorum_2') GROUP BY log_comment ORDER BY log_comment" | Out-File -Encoding utf8 experimento\resultados\prueba-d-latencia.txt
docker exec clickhouse-01 clickhouse-client --format TSVWithNames --query "SELECT event_time, type, query_duration_ms, exception_code, exception, query FROM system.query_log WHERE query ILIKE '%ta01%' AND type != 'QueryStart' ORDER BY event_time_microseconds" | Out-File -Encoding utf8 experimento\resultados\query_log_clickhouse-01.tsv
docker exec clickhouse-02 clickhouse-client --format TSVWithNames --query "SELECT event_time, type, query_duration_ms, exception_code, exception, query FROM system.query_log WHERE query ILIKE '%ta01%' AND type != 'QueryStart' ORDER BY event_time_microseconds" | Out-File -Encoding utf8 experimento\resultados\query_log_clickhouse-02.tsv
```

Los archivos `query_log_*.tsv` registran cada consulta del experimento con su hora, duración y, si falló, el código y el mensaje de error.

### 8. Apagar el clúster

```powershell
docker compose -f experimento\cluster\docker-compose.yaml stop
```

Para borrar todo y repetir el experimento desde cero: `docker compose -f experimento\cluster\docker-compose.yaml down -v`.
