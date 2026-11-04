# Registro de comandos — Laboratorio 3

## Paso 0 — 2026-10-30T19:15:20-06:00
- `docker compose version` -> Docker Compose version v5.6.1
- `make build` -> OK (imagen con pymongo); `mkdir -p evidence`; comandos.md creado.
- Sin contenedores Lab 1/2 en ejecución (no hizo falta make lab1-down / lab2-down).
- Implementados TODO 1-5 en cargar.py, comprar.py, bench_write_concern.py, verificar_rpo.py.
Paso 0: OK

## Paso 1 — 2026-10-30T19:17:51-06:00
- `make lab3-up` + `make lab3-status` -> 3 contenedores healthy; "conjunto no inicializado ...: NotYetInitialized" (evidence/cluster-start.txt)
- Tabla de docker-compose.yml:

| Servicio | --replSet | hostname | volumen | publica puertos |
|---|---|---|---|---|
| mongo-1 | rs-lab3 | mongo-1 | mongo1:/data/db | no (solo 27017/tcp expuesto internamente) |
| mongo-2 | rs-lab3 | mongo-2 | mongo2:/data/db | no |
| mongo-3 | rs-lab3 | mongo-3 | mongo3:/data/db | no |

Comando: mongod --replSet rs-lab3 --bind_ip_all --wiredTigerCacheSizeGB 0.25
Paso 1: OK

Paso 2 — 2026-10-30T19:18:21-06:00: rs.initiate propio ({_id:rs-lab3, 3 miembros mongo-N:27017}); 1 PRIMARY (mongo-1) + 2 SECONDARY. Paso 2: OK

## Paso 3 — 2026-10-30T19:18:31-06:00
- rs-inspect.txt, rs-hello-conf.txt guardados. PRIMARY mongo-1, term=1, electionTimeoutMillis=10000, heartbeatIntervalMillis=2000, votes=1 y priority=1 en los 3.
Paso 3: OK

## Paso 4 — 2026-10-30T19:18:50-06:00: generar_datos.py -> datos-manifest.txt. Paso 4: OK

## Paso 5 — 2026-10-30T19:19:32-06:00
- `cargar.py` -> 5 OK; 400 docs en 0.17 s (2379 docs/s); evidence/cargar.txt
- `make lab3-check` -> 6/6 (config-check.txt)
- explain en PRIMARY mongo-1: IXSCAN 67 docs/67 keys/67 ret/9 ms vs COLLSCAN 400 docs/0 keys/67 ret/1 ms (explain-indices.txt)
Paso 5: OK

## Paso 6 — 2026-10-30T19:19:56-06:00
- insertOne ORD-MANUAL-1 en primario mongo-1; visible en mongo-2; insertOne en secundario -> NotWritablePrimary (code 10107) (oplog-secundario.txt); oplog op:'i' ts t:1793409575 t:1, lag 0 s (oplog.txt). Nota: rs.printReplicationInfo() no imprimió salida en este mongosh; printSecondaryReplicationInfo sí.
Paso 6: OK

## Paso 7 — 2026-10-30T19:20:29-06:00
- lectura_secundario.py --n 500: w1_secundario 95.8 % obsoletas (479/500), majority_secundario 0.6 % (3/500), causal_majority 0.0 %.
Paso 7: OK

## Paso 8 — 2026-10-30T19:21:22-06:00
- comprar.py (correcta, $elemMatch + $inc): 6 OK, "todas las verificaciones pasaron" (120 conf/80 agot, 94 compras/s).
- Versión rota (find_one + decidir en Python + $set sin condición): comprar-roto.txt -> 200 confirmadas/0 agotadas, quedan 114 (no 0), VIP 3 confirmadas (actualizaciones perdidas / sobreventa). 
- Versión correcta restaurada y re-ejecutada; comprar.txt pasa.
Paso 8: OK

## Paso 9 — 2026-10-30T19:21:52-06:00
- bench_write_concern.py --n 300 (warmup 30): p50 w1 2.61 / w1_journal 12.52 / majority 15.98 / w3 14.44 ms; ops/s 270/67/56/61. `wc -l evidence/write-concern.csv` = 1201.
Paso 9: OK

## Paso 10 — 2026-10-30T19:22:53-06:00
- verificar_rpo.py --autotest: 3 OK (rpo-autotest.txt)
- sonda Base 10 s: 20 intentos/20 confirmados/0 errores (rpo-baseline-sonda.txt); verificar_rpo: presentes 20, PERDIDOS 0 (rpo-baseline.txt)
- make lab3-status: mongo-1 PRIMARY, mongo-2/3 SECONDARY, term=1 (estado-fin-parte1.txt)
- maquina.txt guardado.
Paso 10: OK

## Paso 11 intento 1 (descartado) — 2026-10-30T19:25:28-06:00
- La sonda tardó en arrancar (docker compose run) más de 8 s; el docker stop de mongo-3 ocurrió antes de que iniciara, así que todas las filas salieron despues-falla (sin fase antes-falla). Archivos renombrados evidence/intento1-falla-a-*. Se repite el escenario esperando a que el contenedor de la sonda exista.

## Paso 11 — Falla A (reintento válido) — 2026-10-30T19:26:48-06:00
- before: mongo-1 PRIMARY; docker stop --timeout 0 ti4601-mongo-3 (stop.epoch 1793409952.1596, 2026-10-30T19:25:52-06:00); status NODE=mongo-1; sleep 15; docker start mongo-3.
- Sonda: 78 intentos, 0 errores; spike seq=41 421.7 ms (~11.5 s tras el stop); recovered 1 PRIMARY+2 SECONDARY; verificar_rpo PERDIDOS 0.
Paso 11: OK

## Paso 12 — Falla B — 2026-10-30T19:28:38-06:00
- before: mongo-1 PRIMARY, term 1. docker kill ti4601-mongo-1 (stop.epoch 1793410030.1756, 19:27:10-06:00); a +15 s status NODE=mongo-2: mongo-2 PRIMARY, term=2; +15 s docker start mongo-1; recovered: mongo-1 SECONDARY.
- Sonda: 84 intentos, 78 ok, 6 ServerSelectionTimeoutError; primer ok tras kill seq=21 a 19988 ms; verificar_rpo PERDIDOS 0.
Paso 12: OK

## Paso 13 — Falla C — 2026-10-30T19:30:58-06:00
- before: mongo-2 PRIMARY (term 2), secundarios mongo-1/3. Sonda 90 s --w majority --wtimeout-ms 2000. docker kill mongo-1 mongo-3 (stop.epoch 1793410145.7537, 19:29:05-06:00); +20 s status NODE=mongo-2: SECONDARY; +10 s docker start. 
- Sonda: 109 intentos, 96 ok, 13 errores (ServerSelectionTimeoutError 8, WTimeoutError 4, WriteConcernError 1); primer ok tras kill seq=30 a 40977 ms; verificar_rpo: PERDIDOS 0, errores que SÍ se escribieron 5 [17-21]. Recuperado: 1 PRIMARY + 2 SECONDARY.
Paso 13: OK
- estado-final.txt guardado.

## Paso 6 (complemento) — 2026-10-30T22:16:35-06:00
- Comando: `docker exec ti4601-mongo-2 mongosh --quiet --eval 'printjson(db.getReplicationInfo()); rs.printReplicationInfo();'` (mongo-2 es PRIMARY). Salida en evidence/oplog-ventana.txt.
- Resultado: logSizeMB 48352.94, usedMB 9.31, timeDiff 10719 s (2.98 h) = log length start to end; tFirst 01:17:57 UTC, tLast 04:16:36 UTC. Tomado después de las fallas (A-C), así que la ventana refleja toda la sesión. Esta vez rs.printReplicationInfo() sí imprimió salida.
Paso 6 (complemento): OK
