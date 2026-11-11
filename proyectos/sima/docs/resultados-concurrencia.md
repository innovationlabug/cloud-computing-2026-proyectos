# Resultados de la prueba de concurrencia

> Generado por `scripts/prueba-concurrencia/ejecutar.sh` el 2026-10-22 a partir de `20261022-165053.jsonl`.
> Se regenera completo en cada corrida; el análisis de decisiones va en `docs/decisiones.md`.

## Qué se probó

Dos cajas (o cincuenta) venden la última unidad al mismo tiempo. La prueba lanza **50 peticiones
`POST /ventas` simultáneas** sobre el **mismo producto** de la **misma sucursal** (sucursal 1) y mide qué pasa
con tres estrategias de control de concurrencia, seleccionadas con `ESTRATEGIA_CONCURRENCIA`:

| Estrategia | Cómo descuenta la existencia | Qué la protege de la sobreventa |
|---|---|---|
| `PESIMISTA` | `SELECT … FOR UPDATE`, valida, `UPDATE … SET cantidad = cantidad − n` | El bloqueo de fila, tomado **antes de leer**: la segunda caja espera en el `SELECT` y, al despertar, lee el valor ya confirmado |
| `OPTIMISTA` | `SELECT` sin bloqueo, valida, `UPDATE … SET cantidad = cantidad − n WHERE version = :v`; si afecta 0 filas, reintenta la transacción completa (máx. 5, con espera aleatoria corta) | La condición de versión: nunca se escribe sobre un dato que cambió. **También espera**, en el `UPDATE` (ver abajo) |
| `SIN_CONTROL` | `SELECT` sin bloqueo, valida, `UPDATE … SET cantidad = <leída> − n` (escritura absoluta calculada en la aplicación) | **Nada.** Es el *lost update* de libro. Existe únicamente para esta prueba |

Cada respuesta se clasifica por su `codigo`, separando quién rechazó la venta:

- **Lógica · existencia insuficiente** (`EXISTENCIA_INSUFICIENTE`): nuestra validación, con la existencia que leyó, decidió no vender **antes de escribir**.
- **Lógica · reintentos agotados** (`CONFLICTO_VERSION`): la optimista perdió la carrera 5 veces seguidas y se rindió.
- **Motor · CHECK** (`RESTRICCION_BD`, `existencia_cantidad_check`): la aplicación intentó escribir una existencia negativa y PostgreSQL la rechazó.
- **Motor · bloqueo** (`PRODUCTO_BLOQUEADO`, y también interbloqueo o fallo de serialización): la venta esperó la fila más que `lock_timeout` (3 s) y se rechazó sin escribir nada.

Después de cada corrida se verifica **contra la base**, no solo contra la API:

- **Sobreventa, en unidades:** `exitosas × cantidad − existencia inicial`, si es positiva. No alcanza con mirar que la existencia
  final sea ≥ 0: con escritura absoluta queda en 0 y aun así se vendió de más.
- **Existencia:** la final es exactamente `inicial − exitosas × cantidad`, y la API y la base coinciden.
- **Escrituras:** cada venta exitosa dejó 1 `documento`, 1 `movimiento`, 1 fila de `outbox` y +1 en `version`; las rechazadas, nada.
- **Bitácora fila por fila:** en cada `movimiento` `VENTA` nuevo, `cantidad_antes − cantidad = cantidad_despues`.
- **Bitácora en conjunto:** `SUM(cantidad)` de los movimientos `VENTA` nuevos `= inicial − final`. Es la que detecta el *lost update*:
  cada fila cuadra sola, pero la suma de lo vendido supera lo que bajó la existencia.

Antes de cada corrida, `reiniciar-existencia.sh` deja el producto en su existencia inicial con un **AJUSTE por conteo**
(`POST /ajustes`: documento, movimiento `AJUSTE_ENTRADA`/`AJUSTE_SALIDA` y outbox, en una transacción). La foto inicial se
toma **después** de ese ajuste, así que no entra en ninguna de las comparaciones.

Latencias medidas en el cliente, de extremo a extremo, incluido el salto por `acceso-sucursal-1` (el reenviador TCP que
publica el puerto de la sucursal en Compose). Pool de 20 conexiones por API: de las 50 peticiones, hasta 20
transacciones corren a la vez en el motor y el resto espera una conexión del pool.

**Publicador de outbox: activo en todas las corridas.** Durante cada corrida envió entre 0 y 0 filas de outbox al central. Una corrida dura entre 90.2 y 230.4 ms, menos que el intervalo del publicador (2000 ms), así que su actividad apenas se superpuso con la prueba. Cuando se superpone, usa su propia conexión (pool de 1) y toca solo filas de outbox, nunca existencia: no compite por los bloqueos que se miden aquí; sí comparte CPU y la base.

### La optimista también espera

En `READ COMMITTED` (el nivel por defecto de PostgreSQL, y el nuestro) todo `UPDATE` toma el bloqueo de la fila. Si otra
transacción ya la actualizó y todavía no confirma, el `UPDATE … WHERE version = :v` **se queda esperando** a que esa
transacción termine; cuando confirma, PostgreSQL vuelve a evaluar el `WHERE` contra la fila nueva, la versión ya no
coincide y el `UPDATE` afecta 0 filas. Como nuestra transacción de venta retiene ese bloqueo hasta el `COMMIT` —después
de insertar documento, líneas, movimiento y outbox—, la espera no es breve.

La diferencia con la pesimista no es "esperar o no esperar", sino **dónde** se espera y qué pasa al despertar: la
pesimista espera en el `SELECT`, lee el dato bueno y sigue; la optimista espera en el `UPDATE`, descubre que perdió,
deshace todo y vuelve a empezar. Lo que la optimista evita es bloquear **la lectura**, no la escritura. (La tabla de la
sección 6.1 de la propuesta lo resume como "no hay espera"; lo preciso es que no hay espera en la lectura.)

### Por qué `SELECT … FOR UPDATE` y no un `UPDATE` condicional

Para descontar un solo producto hay una tercera forma, correcta y más barata:
`UPDATE existencia SET cantidad = cantidad − :n WHERE … AND cantidad >= :n RETURNING cantidad`. La verificación y la
resta ocurren en una sola sentencia atómica sobre la fila, sin ventana entre leer y escribir. No la usamos en la venta por
dos razones:

1. **El compromiso de la sección 6.1.** La propuesta se comprometió a implementar y medir las dos estrategias clásicas
   —bloqueo pesimista y control optimista con versión— y a justificar la elección con cifras. El `UPDATE` condicional
   no es ninguna de las dos: resuelve la carrera con el bloqueo implícito de la escritura y deja sin nada que comparar.
2. **Bloquear, leer, decidir, escribir.** Hoy la venta decide con un solo dato (¿alcanza la cantidad?), y para eso la
   condición en el `WHERE` basta. Una regla que decida con más de un dato —por ejemplo, comparar contra `stock_min` en
   la misma transacción, o decidir con varias filas a la vez— necesita tener las filas bloqueadas mientras la aplicación
   decide, y una condición en el `WHERE` solo expresa lo que cabe en esa sentencia. `FOR UPDATE` es el patrón general.

**El traslado sí usa el `UPDATE` condicional** (sección 5.2). En la sucursal origen, el participante ejecuta
`UPDATE … WHERE cantidad >= :n` y luego `PREPARE TRANSACTION`. Ahí conviene porque el voto del participante es
exactamente "¿afectó una fila?" —1 es SÍ, 0 es NO— y no hay otra regla que evaluar; y porque en un protocolo que ya es
bloqueante, cada sentencia y cada ida y vuelta de menos acorta el tiempo que la fila de existencia queda retenida.

## Escenario 1 — Última unidad: existencia 1, 50 peticiones de 1

Producto `7501000000046` (Plancha de vapor). Máximo de ventas posibles: **1**.

| Estrategia | Exitosas | Rechazadas · lógica (existencia) | Rechazadas · lógica (reintentos agotados) | Rechazadas · motor (CHECK) | Rechazadas · motor (bloqueo) | Errores | Reintentos (máx/venta) | p50 ms | p95 ms | máx ms | Total ms | Existencia final | Sobreventa |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|:---:|
| `PESIMISTA` | 1 | 49 | 0 | 0 | 0 | 0 | 0 (0) | 64.3 | 85.4 | 86.2 | 90.2 | 0 | no |
| `OPTIMISTA` | 1 | 49 | 0 | 0 | 0 | 0 | 19 (1) | 92.2 | 107.9 | 110 | 112.5 | 0 | no |
| `SIN_CONTROL` | 19 | 31 | 0 | 0 | 0 | 0 | 0 (0) | 64.5 | 137.3 | 146.4 | 147.8 | 0 | **SÍ: 18 u.** |

Verificación contra la base tras la corrida:

| Estrategia | Documentos nuevos | Movimientos nuevos | Outbox nuevas | Versión antes → después | Σ cantidad de movimientos | Inicial − final | Bitácora en conjunto | Interbloqueos | Publicador durante la corrida | Invariantes |
|---|---:|---:|---:|:---:|---:|---:|:---:|---:|:---:|:---:|
| `PESIMISTA` | 1 | 1 | 1 | 0 → 1 | 1 | 1 | ✓ | 0 | activo · 0 enviadas | ✓ todas |
| `OPTIMISTA` | 1 | 1 | 1 | 2 → 3 | 1 | 1 | ✓ | 0 | activo · 0 enviadas | ✓ todas |
| `SIN_CONTROL` | 19 | 19 | 19 | 4 → 23 | 19 | 1 | **✗ no cuadra** | 0 | activo · 0 enviadas | ✗ sin_sobreventa, existencia_cuadra, bitacora_conjunto |

## Escenario 2 — Existencia 20, 50 peticiones de 1

Producto `7501000000041` (Cerveza en lata 355 ml). Máximo de ventas posibles: **20**.

| Estrategia | Exitosas | Rechazadas · lógica (existencia) | Rechazadas · lógica (reintentos agotados) | Rechazadas · motor (CHECK) | Rechazadas · motor (bloqueo) | Errores | Reintentos (máx/venta) | p50 ms | p95 ms | máx ms | Total ms | Existencia final | Sobreventa |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|:---:|
| `PESIMISTA` | 20 | 30 | 0 | 0 | 0 | 0 | 0 (0) | 105.3 | 119.6 | 120 | 123.8 | 0 | no |
| `OPTIMISTA` | 20 | 23 | 7 | 0 | 0 | 0 | 200 (5) | 150.5 | 222.5 | 227.1 | 230.4 | 0 | no |
| `SIN_CONTROL` | 50 | 0 | 0 | 0 | 0 | 0 | 0 (0) | 99.7 | 157.4 | 164.2 | 166.4 | 18 | **SÍ: 30 u.** |

Verificación contra la base tras la corrida:

| Estrategia | Documentos nuevos | Movimientos nuevos | Outbox nuevas | Versión antes → después | Σ cantidad de movimientos | Inicial − final | Bitácora en conjunto | Interbloqueos | Publicador durante la corrida | Invariantes |
|---|---:|---:|---:|:---:|---:|---:|:---:|---:|:---:|:---:|
| `PESIMISTA` | 20 | 20 | 20 | 0 → 20 | 20 | 20 | ✓ | 0 | activo · 0 enviadas | ✓ todas |
| `OPTIMISTA` | 20 | 20 | 20 | 21 → 41 | 20 | 20 | ✓ | 0 | activo · 0 enviadas | ✓ todas |
| `SIN_CONTROL` | 50 | 50 | 50 | 42 → 92 | 50 | 2 | **✗ no cuadra** | 0 | activo · 0 enviadas | ✗ sin_sobreventa, existencia_cuadra, bitacora_conjunto |

## Lectura de los resultados

- **`PESIMISTA` y `OPTIMISTA` no sobrevendieron en ninguna corrida y pasaron todas las verificaciones**: existencia final exacta, API y base coinciden, una escritura por venta exitosa y bitácora cuadrada fila por fila y en conjunto.
- **`SIN_CONTROL` sobrevendió en 2 de 2 corridas (48 unidades en total) y el `CHECK` rechazó 0 escrituras.** Ninguna escritura absoluta fue negativa, así que el motor no tuvo nada que rechazar.

**Escenario 1 — Última unidad: existencia 1**

- `PESIMISTA` vendió 1 de 1 posibles, sin reintentos, con p95 de 85.4 ms. 49 rechazos fueron de la lógica y 0 del motor: el `FOR UPDATE` pone a las cajas en fila y cada una lee el valor que dejó la anterior.
- `OPTIMISTA` vendió 1 de 1 posibles con 19 reintentos (máximo 1 en una venta) y p95 de 107.9 ms (1.3 veces el de la pesimista); 0 peticiones agotaron sus reintentos. No quedaron unidades sin vender.
- `SIN_CONTROL` vendió **19** con 1 en existencia: **18 unidades sobrevendidas**. El `CHECK` rechazó 0 escrituras. La existencia final quedó en 0, cuando lo vendido la habría dejado en -18, mercadería que no existía; la versión subió 19 veces mientras la cantidad bajó 1. Cada movimiento cuadra por sí solo (0 fallan fila por fila), pero **la bitácora en conjunto lo delata**: registra 19 unidades vendidas contra 1 que bajó la existencia.

**Escenario 2 — Existencia 20**

- `PESIMISTA` vendió 20 de 20 posibles, sin reintentos, con p95 de 119.6 ms. 30 rechazos fueron de la lógica y 0 del motor: el `FOR UPDATE` pone a las cajas en fila y cada una lee el valor que dejó la anterior.
- `OPTIMISTA` vendió 20 de 20 posibles con 200 reintentos (máximo 5 en una venta) y p95 de 222.5 ms (1.9 veces el de la pesimista); 7 peticiones agotaron sus reintentos. **Una rendida puede ser un rechazo con existencia disponible en ese momento**: solo se llega a `CONFLICTO_VERSION` después de leer existencia suficiente, así que cada una de estas 7 vio mercadería en su último intento y aun así se fue sin comprar. Aquí no quedaron unidades sin vender porque sobraban peticiones (50 para 20 unidades) y otra caja se llevó esa unidad, pero para ese cliente es un costo real que el total vendido no muestra.
- `SIN_CONTROL` vendió **50** con 20 en existencia: **30 unidades sobrevendidas**. El `CHECK` rechazó 0 escrituras. La existencia final quedó en 18, cuando lo vendido la habría dejado en -30, mercadería que no existía; la versión subió 50 veces mientras la cantidad bajó 2. Cada movimiento cuadra por sí solo (0 fallan fila por fila), pero **la bitácora en conjunto lo delata**: registra 50 unidades vendidas contra 2 que bajó la existencia.

## Conclusión

**El `CHECK` ataja la resta relativa, pero no la escritura absoluta.** En la corrida del 18 de septiembre, `SIN_CONTROL`
restaba en el motor (`SET cantidad = cantidad − n`): la resta se aplicaba sobre el valor ya confirmado, un resultado
negativo solo podía salir de ahí y el `CHECK` lo rechazaba (13 y 19 veces; ver `docs/decisiones.md`). Con la escritura
absoluta de esta corrida, la aplicación calcula el valor nuevo a partir de un dato viejo; ese valor nunca es negativo, el
`CHECK` no salta y la sobreventa pasa en silencio (48 unidades en esta corrida). El `CHECK` se queda, porque
protege toda escritura relativa —incluido el `UPDATE` condicional del traslado—, pero no es la garantía contra la
sobreventa: la garantía es el control de concurrencia.

**La bitácora inmutable detecta la sobreventa aunque el `CHECK` no salte.** La suma de lo que registran los movimientos
no coincide con lo que bajó la existencia. Es detección después del hecho, no prevención —la mercadería ya se vendió—,
y por eso la verificación en conjunto corre en cada prueba.

**Pesimista frente a optimista.** Con una sola fila caliente, la pesimista resolvió el escenario de existencia 20 sin
reintentos y con p95 de 119.6 ms; la optimista necesitó 200 reintentos, 7 ventas se rindieron y su p95
fue de 222.5 ms. Para el producto disputado, la pesimista es la que conviene. Esta prueba no mide el caso de baja
contención —muchas cajas vendiendo productos distintos—, que es donde la optimista debería rendir; esa medición queda
para las pruebas de carga con k6 y pgbench.

## Cómo reproducir

```bash
docker compose up -d && ./scripts/migrar.sh && ./scripts/sembrar.sh
./scripts/prueba-concurrencia/ejecutar.sh 50          # las 3 estrategias × 2 escenarios; al final deja todo con la imagen nueva

# Una sola corrida contra la estrategia que tenga activa la API:
./scripts/prueba-concurrencia/reiniciar-existencia.sh 1 7501000000046 1
node scripts/prueba-concurrencia/prueba.mjs --codigo 7501000000046 --n 50
```

Resultados crudos de esta corrida (un JSON por línea): `scripts/prueba-concurrencia/resultados/20261022-165053.jsonl` (no versionado).
