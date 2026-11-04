# Predicciones (escritas ANTES de ejecutar cada paso)

## Paso 4/5 — réplicas de cada documento — 2026-10-30T19:18:25-06:00
- Cada documento terminará en los 3 nodos (el primario lo escribe y los 2 secundarios copian el oplog; ahora todos tienen datos y votan).
- Antes de que `insert_many` responda deben tenerlo al menos 2 de 3 (mayoría, `w:"majority"` = ⌊3/2⌋+1 = 2): el primario más un secundario. El tercero lo recibe milisegundos después.

## Paso 7 — % de lecturas obsoletas — 2026-10-30T19:19:56-06:00
- w1_secundario: ~30-60 %. El primario responde en cuanto aplica; la lectura en el secundario ocurre tras un viaje de red (<1 ms), y el secundario replica por pull/oplog con latencia de ~1-10 ms. Es una carrera que la lectura suele perder a menudo.
- majority_secundario: ~5-25 %. Al responder, ya hay 2 de 3 nodos con el dato (primario + uno); si la lectura cae en ese secundario es 0 %, si cae en el otro (50 % de probabilidad) todavía puede no tenerlo. Mejor que w1 pero no 0 %.
- causal_majority: 0 %. La sesión causal pasa afterClusterTime con el operationTime de la escritura y el secundario espera hasta tenerlo antes de responder.

## Paso 9 — costo de las garantías — 2026-10-30T19:21:22-06:00
- Orden de más rápida a más lenta: w1 < w1_journal < majority ≈ w3.
- w1 solo espera al primario en memoria (~0.3-1 ms). j:true agrega esperar el fsync del journal (WiredTiger agrupa cada ~100 ms o antes; puede costar varios ms). majority y w3 esperan replicación al/los secundarios (round trip del oplog, sin red real) y no difieren mucho porque con 3 nodos majority=2 y w3=3 y los secundarios replican en paralelo, de modo que w3 solo espera al más lento de los dos.
- majority será ~3-6 veces más lenta que w1 (estimo 4x).

## Paso 11 — Falla A (cae un secundario) — 2026-10-30T19:24:05-06:00
- Errores: ninguno. Quedan 2 de 3 nodos (primario + 1 secundario) = mayoría (⌊3/2⌋+1 = 2), así que w:"majority" se sigue cumpliendo y el primario no renuncia.
- Latencia: casi igual; posible pico aislado si una escritura estaba esperando al secundario caído, y después podría incluso bajar levemente (solo hay un secundario que confirmar). p99 algo mayor que antes por ese pico.
- Al volver, el secundario retoma desde su último optime y copia del oplog lo que se perdió (~15 s de escrituras), con un retraso breve que se cierra en segundos. PERDIDOS 0.

## Paso 12 — Falla B (cae el primario, mongo-1) — 2026-10-30T19:26:51-06:00
- Sin primario unos 10-15 s. Depende de electionTimeoutMillis (10 s por defecto): los secundarios esperan ese tiempo sin heartbeat del primario antes de convocar elección (más un jitter aleatorio y ~1 s de votación); el driver además debe redescubrir al nuevo primario (heartbeat 1 s en la sonda). Estimo RTO ≈ 11-14 s.
- La sonda verá errores (ServerSelectionTimeoutError tras 3 s de espera, o errores de red/NotWritablePrimary en la primera escritura) durante toda la ventana, y luego vuelven los ok con la latencia normal. retryWrites reintenta una sola vez y no oculta la falta de primario por más de serverSelectionTimeout.
- term pasará de 1 a 2; el nodo nuevo primario será mongo-2 o mongo-3. mongo-1 volverá como SECONDARY (el rol lo da la elección, y ya existe un primario con term mayor).

## Paso 13 — Falla C (dos secundarios caídos, primario vivo) — 2026-10-30T19:28:42-06:00
1. Sin mayoría no puede haber primario efectivo: el sobreviviente (mongo-1) ve 1 de 3 votos y se degrada a SECONDARY por checkQuorum/heartbeats tras ~10 s (electionTimeoutMillis) de no ver mayoría. Hasta entonces sigue creyéndose primario. Estimo step down a ~10-12 s del kill.
2. Con w:"majority" ningún insert se confirma: tras 2 s (wtimeout) devuelve WTimeoutError (antes del step down) y luego, cuando ya no es primario, error de selección de servidor/NotWritablePrimary (ServerSelectionTimeoutError tras 3 s). Sí: los documentos del periodo WTimeout quedan escritos localmente en el nodo vivo aunque la respuesta sea error (y se replicarán al volver los otros).
3. Misma mayoría 2 de 3 que en Lab 1 (Raft de CockroachDB, que bloquea las escrituras sin quórum en el rango): aquí hay un primario que aplica primero y espera la confirmación después, así que "error" puede significar "escrito"; en Raft el líder no aplica hasta tener quórum. Resultado: indisponibilidad de escritura en ambos, pero con resultado ambiguo en Mongo.
