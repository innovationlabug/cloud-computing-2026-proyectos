# Resultados de indexamiento

> Generado por `scripts/prueba-indices/ejecutar.sh` el 2026-10-22 a partir de `20261022-164028.jsonl`.
> Se regenera completo en cada corrida; las decisiones van en `docs/decisiones.md`.

## Qué se midió y con qué volumen

Consultas del código de SIMA sobre bases de prueba con volumen, en PostgreSQL 16.15 (los mismos contenedores
de Docker Compose, en bases aparte: `sima_indices_sucursal` y `sima_indices_central`, que se borran al terminar). No se
carga volumen en las bases de operación: `movimiento` es inmutable.

| Base | Volumen (ESCALA=1) |
|---|---|
| Sucursal | producto: 20,000 · documento: 400,001 · movimiento: 1,000,000 · outbox: 400,000 · outbox PENDIENTE: 20 · un TRASLADO anulado al final, sin movimientos |
| Central | sucursal: 30 · producto: 20,000 · existencia_consolidada: 600,000 · documento: 661,967 · traslado: 20,000 · transaccion_global: 19,985 · participante: 39,970 · traslados en SOLICITADO: 15 |

Las fechas crecen con el id, como en la operación real. Método:

- **Los textos SQL son los del código.** `medir.mjs` los captura ejecutando las funciones del núcleo compilado contra un
  cliente grabador, que anota cada consulta enviada. No hay copias a mano que puedan desviarse. La única excepción es la
  versión anterior de `consultarTraslado` (fase "antes"), verificada contra `git show 6319696`.
- **Por consulta:** 3 corridas de calentamiento, 15 medidas (mediana y p95, tiempo de cliente, que incluye la ida y vuelta
  a la base) y un `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)`, del que se toman los nodos de acceso, los índices usados, los
  bloques (compartidos, leídos o en caché), las filas descartadas por filtro y el tiempo de ejecución en el servidor.
- **"Antes":** migraciones hasta sucursal/008 y central/009. **"Después":** con sucursal/009 y central/010, `VACUUM ANALYZE`
  en las dos fases.
- **Experimentos:** cada variante corre dentro de `BEGIN … DROP INDEX … ROLLBACK`, así que no queda nada cambiado.

## Consultas del código: antes y después

| Id | Consulta | Dónde | Índice | Mediana / p95 antes (ms) | Mediana / p95 después (ms) | Ejecución en el servidor (ms) | Bloques | Filas descartadas por filtro | Acceso antes → después |
|---|---|---|---|---:|---:|---:|---:|---:|---|
| H1 | Existencia de un producto en la red | `existencias/consultar-existencias-red.ts` · `consultarExistenciasRed` | (ninguno: el candidato se descartó, ver experimento) | 1.16 / 1.76 | 0.98 / 2.79 | 0.23 → 0.26 | 241 → 241 | 0 → 0 | `Seq Scan sucursal`, `Index Scan existencia_consolidada_pkey`, `Index Scan idx_documento_suc_fecha` → `Seq Scan sucursal`, `Index Scan existencia_consolidada_pkey`, `Index Scan idx_documento_suc_fecha` |
| H2 | ¿El TRASLADO tiene salidas? (recepción de un traslado anulado) | `traslados/recepcion.ts` · `registrarRecepcion` | `idx_movimiento_documento` | 20.78 / 33.35 | 0.38 / 0.79 | 20.34 → 0.03 | 9,346 → 3 | 999,999 → 0 | `Seq Scan movimiento` → `Index Only Scan idx_movimiento_documento` |
| H3 | Traslado por id_traslado | `traslados/coordinador.ts` · `consultarTraslado` | (cambio de código: uuid = $1::uuid usa traslado_id_traslado_key) | 4.09 / 4.83 | 0.43 / 0.7 | 3.45 → 0.06 | 243 → 7 | 19,999 → 0 | `Seq Scan traslado`, `Index Scan documento_pkey` → `Index Scan traslado_id_traslado_key`, `Index Scan documento_pkey` |
| H4 | Copias de un traslado | `traslados/coordinador.ts` · `consultarTraslado` | `idx_documento_traslado` | 0.92 / 1.38 | 0.26 / 0.71 | 0.36 → 0.02 | 313 → 3 | 0 → 0 | `Index Scan uq_documento_traslado_copia` → `Index Scan idx_documento_traslado` |
| H5 | Decisión por gid (reconciliador) | `traslados/coordinador.ts` · `consultarDecision` | `idx_participante_tx_preparada` | 2.81 / 5.1 | 0.47 / 0.51 | 3.27 → 0.04 | 500 → 10 | 39,969 → 0 | `Seq Scan participante`, `Index Scan transaccion_global_pkey`, `Index Scan traslado_pkey` → `Index Scan idx_participante_tx_preparada`, `Index Scan transaccion_global_pkey`, `Index Scan traslado_pkey` |
| H6 | Traslados en SOLICITADO (recuperación) | `traslados/coordinador.ts` · `ejecutarRecuperacion` | `idx_traslado_solicitado` | 2.86 / 6.28 | 0.32 / 1.13 | 2.34 → 0.02 | 239 → 16 | 19,985 → 0 | `Seq Scan traslado` → `Index Scan idx_traslado_solicitado` |

- **H1, existencia de un producto en la red.** La hipótesis era un Seq Scan del consolidado, y no es lo que hace el plan: el `JOIN` con `sucursal` dirige un Nested Loop que busca en la PK `(id_sucursal, id_producto)` con las dos columnas, una vez por sucursal. Ejecución en el servidor: 0.23 → 0.26 ms. **El índice candidato no se agregó**: ver "Candidato descartado". La consulta queda igual en las dos fases.
- **H2, recepción de un traslado.** Sin índice por `id_documento`, `SELECT 1 … LIMIT 1` recorrió la bitácora entera (999,999 filas descartadas, 9,346 bloques), porque el TRASLADO anulado no tiene salidas, que es el caso normal del destino. Con `idx_movimiento_documento` es un Index Only Scan de 3 bloques: 678.0× menos tiempo de servidor.
- **H3, traslado por id.** Con `id_traslado::text = lower($1)`, el cast anulaba el índice UNIQUE: Seq Scan sobre 20,000 traslados. Con `id_traslado = $1::uuid` (formato validado en TypeScript) usa `traslado_id_traslado_key`: 243 → 7 bloques. Es un cambio de código, no un índice.
- **H4, copias de un traslado.** Antes no había Seq Scan: el planificador recorría completo el índice parcial `uq_documento_traslado_copia`, que es chico porque solo tiene copias pero empieza por `id_sucursal` (313 bloques). Con `idx_documento_traslado` va directo: 3 bloques, 18.0× menos tiempo de servidor. La mejora crece con el número de traslados.
- **H5, decisión por gid.** La consulta que hace el reconciliador de cada sucursal recorría `participante` (39,969 filas descartadas). Con `idx_participante_tx_preparada`, 10 bloques. La bitácora no se purga: sin el índice, el costo crecería con la historia.
- **H6, SOLICITADO en la recuperación.** Corre cada 5 s y recorría todos los traslados de la historia para encontrar 15 (19,985 descartados). El índice parcial `idx_traslado_solicitado` solo contiene los que esperan: 239 → 16 bloques, y su tamaño no crece con la historia.

Las medianas de cliente incluyen la ida y vuelta a la base (unas décimas de ms): por eso las consultas que ya eran rápidas
no bajan en la misma proporción que su tiempo de servidor. La comparación más limpia es la de ejecución en el servidor y bloques.

## Candidato descartado: `(id_producto, id_sucursal)` para H1

| Variante | Mediana / p95 (ms) | Ejecución en el servidor (ms) | Bloques | Acceso | Tamaño del índice |
|---|---:|---:|---:|---|---:|
| PK (id_sucursal, id_producto), como queda | 0.85 / 2.03 | 0.24 | 241 | `Seq Scan sucursal`, `Index Scan existencia_consolidada_pkey`, `Index Scan idx_documento_suc_fecha` | — |
| con el candidato (id_producto, id_sucursal) | 2.33 / 4.11 | 0.27 | 154 | `Bitmap Heap Scan existencia_consolidada`, `Bitmap Index Scan idx_candidato_existencia_producto`, `Seq Scan sucursal`, `Index Scan idx_documento_suc_fecha` | 12.9 MB |

El índice cambia el plan, pero no el tiempo de ejecución (0.24 contra 0.27 ms) y cuesta 12.9 MB más el
mantenimiento de cada escritura del consumidor. Con 30 sucursales, las búsquedas por PK que hace el Nested Loop son baratas. Si el
número de sucursales creciera mucho, convendría volver a medir. No se agregó.

## Experimento a) Caja: B-tree del UNIQUE contra el hash de 006

Tamaños: B-tree (`producto_codigo_barras_key`, el del UNIQUE) **0.6 MB**; hash (`idx_producto_codigo_hash`) **1.2 MB**, con 20,000 productos.

| Variante | Consulta | Mediana / p95 (ms) | Ejecución en el servidor (ms) | Bloques | Acceso |
|---|---|---:|---:|---:|---|
| B-tree (UNIQUE) + hash | = ANY de 5 códigos (venta) | 0.48 / 1.57 | 0.04 | 15 | `Index Scan producto_codigo_barras_key` |
| B-tree (UNIQUE) + hash | = de 1 código (GET /existencias/:codigo) | 0.41 / 0.62 | 0.03 | 5 | `Index Scan idx_producto_codigo_hash`, `Index Scan existencia_pkey` |
| solo B-tree | = ANY de 5 códigos (venta) | 0.49 / 1.05 | 0.04 | 15 | `Index Scan producto_codigo_barras_key` |
| solo B-tree | = de 1 código (GET /existencias/:codigo) | 0.55 / 0.65 | 0.03 | 6 | `Index Scan producto_codigo_barras_key`, `Index Scan existencia_pkey` |
| solo hash | = ANY de 5 códigos (venta) | 0.46 / 1.04 | 0.02 | 10 | `Bitmap Heap Scan producto`, `Bitmap Index Scan idx_producto_codigo_hash` |
| solo hash | = de 1 código (GET /existencias/:codigo) | 0.38 / 1.17 | 0.02 | 5 | `Index Scan idx_producto_codigo_hash`, `Index Scan existencia_pkey` |
| sin índice | = ANY de 5 códigos (venta) | 3.46 / 5.07 | 2.45 | 223 | `Seq Scan producto` |
| sin índice | = de 1 código (GET /existencias/:codigo) | 1.45 / 1.69 | 0.83 | 226 | `Seq Scan producto`, `Index Scan existencia_pkey` |

**Lectura.** Con un solo código, B-tree y hash ejecutan en 0.03 y 0.02 ms; con `= ANY` de 5 códigos, en 0.04 y 0.02 ms (el hash
resuelve `= ANY` con un Bitmap Index Scan). Sin índice, la caja recorre el catálogo: 2.45 ms. El hash no aporta una diferencia medible sobre
el B-tree, ocupa 1.9 veces más, y el B-tree no se puede quitar: es el que hace cumplir el `UNIQUE`. Un hash no puede ser UNIQUE
ni servir para rangos ni para ordenar. **Recomendación: quitar `idx_producto_codigo_hash`.** Queda escrita en `docs/decisiones.md`; no se
borra hasta que el equipo lo decida.

## Experimento b) Publicador y el índice parcial `idx_outbox_pendiente`

| Variante | Consulta | Mediana / p95 (ms) | Ejecución en el servidor (ms) | Bloques | Filas descartadas | Acceso |
|---|---|---:|---:|---:|---:|---|
| con idx_outbox_pendiente | publicador: lote de pendientes FOR UPDATE SKIP LOCKED | 0.42 / 1.03 | 0.06 | 42 | 0 | `Index Scan idx_outbox_pendiente` |
| con idx_outbox_pendiente | /salud: pendiente más antigua | 0.27 / 1.79 | 0.04 | 2 | 0 | `Index Scan idx_outbox_pendiente` |
| sin idx_outbox_pendiente | publicador: lote de pendientes FOR UPDATE SKIP LOCKED | 48.49 / 91.95 | 43.31 | 14,855 | 399,980 | `Seq Scan outbox` |
| sin idx_outbox_pendiente | /salud: pendiente más antigua | 23.92 / 30.9 | 23.26 | 14,887 | 399,981 | `Seq Scan outbox` |

Con 400,000 filas en la outbox y 20 pendientes, el índice parcial solo contiene las pendientes: el ciclo del publicador
pasa de 43.31 a 0.06 ms (721.8×). El publicador corre cada 2 s en cada sucursal y la outbox no se purga, así que sin el
índice el costo crecería con toda la historia de la sucursal. Se queda.

## Experimento c) Índices de 006 sobre `movimiento`: cuánto ganan y cuánto cuestan

**Hoy ningún endpoint los usa.** `idx_movimiento_fecha` e `idx_movimiento_suc_prod` se crearon en 006 para consultas de la
bitácora por rango de fechas (kardex, bitácora del día) que todavía no existen en el código. Las dos consultas de abajo son de
referencia, escritas para este experimento.

| Variante | Consulta de referencia | Mediana / p95 (ms) | Ejecución en el servidor (ms) | Bloques | Acceso |
|---|---|---:|---:|---:|---|
| con los dos índices de 006 | kardex de un producto, 30 días | 0.39 / 0.67 | 0.03 | 7 | `Index Scan idx_movimiento_suc_prod` |
| con los dos índices de 006 | bitácora de ayer | 4.25 / 9.97 | 0.52 | 34 | `Index Scan idx_movimiento_fecha` |
| sin idx_movimiento_fecha | kardex de un producto, 30 días | 0.3 / 0.63 | 0.05 | 7 | `Index Scan idx_movimiento_suc_prod` |
| sin idx_movimiento_fecha | bitácora de ayer | 112.92 / 247.74 | 103.2 | 9,418 | `Seq Scan movimiento` |
| sin idx_movimiento_suc_prod | kardex de un producto, 30 días | 5.84 / 6.94 | 5.02 | 910 | `Index Scan idx_movimiento_fecha` |
| sin idx_movimiento_suc_prod | bitácora de ayer | 3.72 / 6.9 | 0.42 | 34 | `Index Scan idx_movimiento_fecha` |
| sin ninguno de los dos | kardex de un producto, 30 días | 25.2 / 26.98 | 25.2 | 9,346 | `Seq Scan movimiento` |
| sin ninguno de los dos | bitácora de ayer | 108.56 / 148.36 | 102.96 | 9,418 | `Seq Scan movimiento` |

Costo en escritura: `INSERT` de 10,000 movimientos en una transacción, sobre tablas de prueba creadas con
`CREATE TABLE … (LIKE movimiento INCLUDING DEFAULTS)`: sin llaves foráneas, sin CHECK y sin el trigger de inmutabilidad, cargadas
con las mismas 1,000,000 filas de `movimiento` y cada una con solo los índices de su variante. Así lo único que
cambia entre variantes es el mantenimiento de índices. Las variantes se corren intercaladas por ronda (A, B, C, A, B, C…), con
3 rondas de calentamiento y 30 medidas por variante; cada corrida termina en `ROLLBACK`. Las fechas y los documentos
del INSERT son crecientes, como en una tanda de ventas, y los productos se reparten por el catálogo.

Regla de lectura: una diferencia se da como porcentaje solo si el p95 de la variante más rápida queda por debajo de la mediana de
la más lenta. Si no, se escribe "no medible con este método". El p95 de las variantes con más índices recoge corridas lentas
aisladas (checkpoints, escritura de WAL), así que el porcentaje es el de esta corrida y cambia entre corridas: lo que se sostiene es
si la diferencia es medible y su orden de magnitud.

| Índices sobre la tabla de prueba | Índices, tabla recién cargada (MB) | Mediana (ms) | p95 (ms) | µs por fila (mediana / p95) | Respecto de "solo la PK" |
|---|---:|---:|---:|---:|---:|
| solo la PK | 21.4 | 48.87 | 234.74 | 4.89 / 23.47 | referencia |
| PK + 006 | 73.0 | 78.41 | 280.21 | 7.84 / 28.02 | no medible con este método |
| PK + 006 + idx_movimiento_documento | 90.1 | 88.64 | 524.08 | 8.86 / 52.41 | no medible con este método |

**Lectura.** El kardex de 30 días baja de 25.2 ms (Seq Scan) a 0.03 ms con `idx_movimiento_suc_prod`; la bitácora de ayer, de
102.96 a 0.52 ms con `idx_movimiento_fecha`. En escritura, el costo que los dos índices de 006 suman al insertar movimientos no es medible con este método (la diferencia queda dentro del ruido); lo que suma `idx_movimiento_documento` sobre "PK + 006" no es medible con este método.
Los dos índices de 006 ocupan 65.1 MB con 1,000,000 movimientos. Mientras ningún endpoint los use, son
costo sin beneficio en la base operativa de la sucursal. Además, la regla del proyecto manda las consultas analíticas al almacén
analítico, no a la base operativa. Se dejan como están y la decisión queda anotada en `docs/decisiones.md`.

## Experimento d) `/salud`: el conteo de la outbox recorre toda la tabla (límite conocido)

`consultarEstadoOutbox` cuenta las filas por estado (`COUNT(*) FILTER …`) sobre toda la outbox, a propósito: `/salud` informa enviados y fallidos. Con
400,000 filas: mediana 30.56 ms, p95 36.87 ms, `Seq Scan outbox`, 14,815 bloques. Crece con la historia de la sucursal, porque
la outbox no se purga. No se indexa: un índice no evita contar todo, y lo que importa operativamente (la pendiente más
antigua) ya usa `idx_outbox_pendiente` (experimento b). Si `/salud` se volviera lenta, las alternativas serían purgar o archivar
los ENVIADO viejos, o informar solo los pendientes. Queda como límite conocido.

## Tamaños de los índices (fase "después")

**Sucursal.** Tablas: `documento` 38.6 MB · `movimiento` 73.0 MB · `outbox` 115.7 MB · `producto` 1.7 MB.

| Tabla | Índice | Tamaño (MB) |
|---|---|---:|
| documento | `documento_pkey` | 8.6 |
| documento | `idx_documento_suc_fecha` | 12.1 |
| documento | `uq_documento_traslado_tipo` | 0.0 |
| movimiento | `idx_movimiento_documento` **(nuevo)** | 17.2 |
| movimiento | `idx_movimiento_fecha` | 21.4 |
| movimiento | `idx_movimiento_suc_prod` | 43.6 |
| movimiento | `movimiento_pkey` | 21.4 |
| outbox | `idx_outbox_pendiente` | 0.0 |
| outbox | `outbox_id_idempotencia_key` | 16.7 |
| outbox | `outbox_pkey` | 8.6 |
| producto | `idx_producto_codigo_hash` | 1.2 |
| producto | `producto_codigo_barras_key` | 0.6 |
| producto | `producto_pkey` | 0.4 |

**Central.** Tablas: `documento` 64.8 MB · `existencia_consolidada` 39.1 MB · `participante` 3.9 MB · `transaccion_global` 1.5 MB · `traslado` 1.9 MB.

| Tabla | Índice | Tamaño (MB) |
|---|---|---:|
| documento | `documento_id_sucursal_id_documento_origen_key` | 14.6 |
| documento | `documento_pkey` | 14.2 |
| documento | `idx_documento_suc_fecha` | 23.1 |
| documento | `idx_documento_traslado` **(nuevo)** | 0.9 |
| documento | `uq_documento_traslado_copia` | 2.5 |
| existencia_consolidada | `existencia_consolidada_pkey` | 12.9 |
| participante | `idx_participante_tx_preparada` **(nuevo)** | 2.3 |
| participante | `participante_pkey` | 1.2 |
| transaccion_global | `idx_transaccion_global_pendiente` | 0.0 |
| transaccion_global | `transaccion_global_id_documento_key` | 0.4 |
| transaccion_global | `transaccion_global_pkey` | 0.4 |
| traslado | `idx_traslado_solicitado` **(nuevo)** | 0.0 |
| traslado | `traslado_id_traslado_key` | 0.8 |
| traslado | `traslado_pkey` | 0.4 |

## Textos SQL medidos

<details><summary>Consultas capturadas del código</summary>

**H1 (antes)**, `existencias/consultar-existencias-red.ts` · `consultarExistenciasRed`:

```sql
SELECT s.id_sucursal,
  s.nombre AS sucursal,
  ec.cantidad,
  ec.version,
  ec.actualizado_en,
  ec.sincronizado_en,
  COALESCE(ultimo.fecha_hora, ec.sincronizado_en) AS vigente_al,
  ultimo.fecha_hora IS NOT NULL AS hay_documentos,
  EXTRACT(EPOCH FROM (now() AT TIME ZONE 'UTC') - COALESCE(ultimo.fecha_hora, ec.sincronizado_en))::float8
  AS antiguedad_segundos
  FROM existencia_consolidada ec
  JOIN sucursal s ON s.id_sucursal = ec.id_sucursal
  LEFT JOIN LATERAL (
  SELECT d.fecha_hora
  FROM documento d
  WHERE d.id_sucursal = ec.id_sucursal
  AND d.id_documento_origen IS NOT NULL
  ORDER BY d.fecha_hora DESC
  LIMIT 1
  ) ultimo ON TRUE
  WHERE ec.id_producto = $1
  ORDER BY s.id_sucursal
```

**H2 (antes)**, `traslados/recepcion.ts` · `registrarRecepcion`:

```sql
SELECT 1 FROM movimiento WHERE id_documento = $1 LIMIT 1
```

**H3 (antes)**, `traslados/coordinador.ts` · `consultarTraslado`:

```sql
SELECT t.id_traslado::text, t.estado, t.id_sucursal_origen, t.id_sucursal_destino, t.fecha_solicitud, t.fecha_resolucion,
  t.id_documento, d.estado AS estado_documento
  FROM traslado t JOIN documento d USING (id_documento)
  WHERE t.id_traslado::text = lower($1)
```

**H3 (después)**, `traslados/coordinador.ts` · `consultarTraslado`:

```sql
SELECT t.id_traslado::text, t.estado, t.id_sucursal_origen, t.id_sucursal_destino, t.fecha_solicitud, t.fecha_resolucion,
  t.id_documento, d.estado AS estado_documento
  FROM traslado t JOIN documento d USING (id_documento)
  WHERE t.id_traslado = $1::uuid
```

**H4 (antes)**, `traslados/coordinador.ts` · `consultarTraslado`:

```sql
SELECT d.id_documento, d.tipo, d.id_sucursal, d.id_documento_origen FROM documento d
  WHERE d.id_traslado = $1 ORDER BY d.id_documento
```

**H5 (antes)**, `traslados/coordinador.ts` · `consultarDecision`:

```sql
SELECT p.rol, tg.estado, t.estado AS estado_traslado, t.id_traslado::text
  FROM participante p JOIN transaccion_global tg USING (id_tx_global) JOIN traslado t USING (id_documento)
  WHERE p.id_tx_preparada = $1
```

**H6 (antes)**, `traslados/coordinador.ts` · `ejecutarRecuperacion`:

```sql
SELECT id_traslado::text FROM traslado WHERE estado = 'SOLICITADO' ORDER BY fecha_solicitud LIMIT $1
```

</details>

## Cómo reproducir

```bash
docker compose up -d db-central db-sucursal-1        # solo hacen falta las bases
./scripts/prueba-indices/ejecutar.sh                   # ESCALA=1, ~2 min; genera este informe
ESCALA=0.2 MEDICIONES=10 ./scripts/prueba-indices/ejecutar.sh   # más rápido
CONSERVAR=1 ./scripts/prueba-indices/ejecutar.sh       # deja sima_indices_sucursal y sima_indices_central para explorar
node scripts/prueba-indices/medir.mjs --solo-captura   # muestra los SQL que se capturan del código
```

Resultados crudos de esta corrida (un JSON por línea): `scripts/prueba-indices/resultados/20261022-164028.jsonl` (no versionado).
