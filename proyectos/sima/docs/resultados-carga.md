# Resultados de las pruebas de carga

> Parte local generada por `scripts/prueba-carga/ejecutar.sh completo` el 2026-10-22 a partir de `20261022-104300-completo`.
> La parte de GCP, al final, la genera `infra/gcp/carga-gcp.sh`. Cada una se regenera completa en su corrida;
> el análisis va en `docs/decisiones.md`.

## Antes de leer los números

**En local, todo comparte la misma máquina.** k6, las cuatro APIs, las cuatro bases y los reenviadores corren en
Docker Desktop sobre los mismos 5 CPU y 5.8 GiB de memoria, y compiten entre sí por la CPU.
Los números de esta parte sirven para **comparar estrategias y escenarios entre sí, no como capacidad del sistema**:
una tasa que aquí no se sostiene puede ser la CPU que se lleva k6, no la sucursal. La parte de GCP separa el generador
de carga de la sucursal.

## Qué se probó

- **Ventas** (`ventas.js`, k6 1.3.0) contra `api-sucursal-1`, con cada estrategia de concurrencia
  (`PESIMISTA` y `OPTIMISTA`). Escalones de **tasa constante**: k6 lanza la tasa pedida haga falta los usuarios virtuales que
  haga falta; si no le alcanzan, las cuenta como *descartadas por k6* (el sistema no dio abasto). Antes de los escalones
  hay un calentamiento de 5 s que no entra en las tablas (salvo en agotamiento, que no lo lleva). Cada venta es de 1 unidad.
  Cada corrida empieza con el outbox de las sucursales vacío; durante la corrida, el publicador trabaja en paralelo, como en producción. k6 entra por la red de Docker directo a la API, sin el
  reenviador de la caja.
- **Clasificación de cada respuesta:** aceptada (201), *bloqueo* (409 `PRODUCTO_BLOQUEADO`: esperó la fila más que el
  `lock_timeout` de 3 s), *existencia insuficiente*, *conflicto de versión* (la optimista agotó sus 5 reintentos), 5xx, error de red u otro 4xx.
- **Latencias** medidas en k6, de extremo a extremo; *p99 aceptadas* cuenta solo las ventas que se registraron.
- **Qué decide si la prueba pasa:** las verificaciones contra la base (sobreventa 0, existencia que cuadra, un documento,
  movimiento y outbox por venta aceptada), que no haya 5xx ni errores de red, y que el central cuadre al final. Las latencias
  se informan, pero no deciden.

## Ventas · caliente

Todas las ventas sobre **un solo producto** (el 31): todas compiten por la misma fila de `existencia`. Existencia inicial: 100,000 unidades; escalones de 30 s.

| Estrategia | Tasa (/s) | Enviadas | Aceptadas/s | Descartadas por k6 | Aceptadas | Bloqueo (409) | Existencia insuficiente | Conflicto de versión | 5xx, red u otros | p50 (ms) | p95 (ms) | p99 (ms) | p99 aceptadas (ms) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| PESIMISTA | 25 | 750 | 25.0 | 0 | 750 | 0 | 0 | 0 | 0 | 7.7 | 11 | 19 | 19 |
| PESIMISTA | 50 | 1,501 | 50.0 | 0 | 1,501 | 0 | 0 | 0 | 0 | 5.3 | 8.4 | 28 | 28 |
| PESIMISTA | 100 | 3,000 | 100.0 | 0 | 3,000 | 0 | 0 | 0 | 0 | 2.9 | 7.4 | 29 | 29 |
| PESIMISTA | 200 | 6,000 | 200.0 | 0 | 6,000 | 0 | 0 | 0 | 0 | 2.3 | 20 | 75 | 75 |
| PESIMISTA | 400 | 11,983 | 399.4 | 18 | 11,983 | 0 | 0 | 0 | 0 | 3.8 | 947 | 1063 | 1063 |
| OPTIMISTA | 25 | 751 | 25.0 | 0 | 751 | 0 | 0 | 0 | 0 | 8.2 | 12 | 28 | 28 |
| OPTIMISTA | 50 | 1,501 | 50.0 | 0 | 1,501 | 0 | 0 | 0 | 0 | 5.4 | 8.7 | 32 | 32 |
| OPTIMISTA | 100 | 3,001 | 100.0 | 0 | 3,001 | 0 | 0 | 0 | 0 | 3.0 | 6.8 | 23 | 23 |
| OPTIMISTA | 200 | 6,000 | 192.1 | 0 | 5,764 | 0 | 0 | 236 | 0 | 2.5 | 113 | 348 | 147 |
| OPTIMISTA | 400 | 11,743 | 211.3 | 257 | 6,340 | 0 | 0 | 5,403 | 0 | 244 | 1823 | 2155 | 1787 |

## Ventas · azar

Ventas sobre **los productos 1 a 30, al azar**: casi sin contención; es la línea base. Existencia inicial: 100,000 unidades por producto; escalones de 30 s.

| Estrategia | Tasa (/s) | Enviadas | Aceptadas/s | Descartadas por k6 | Aceptadas | Bloqueo (409) | Existencia insuficiente | Conflicto de versión | 5xx, red u otros | p50 (ms) | p95 (ms) | p99 (ms) | p99 aceptadas (ms) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| PESIMISTA | 25 | 751 | 25.0 | 0 | 751 | 0 | 0 | 0 | 0 | 7.8 | 11 | 23 | 23 |
| PESIMISTA | 50 | 1,500 | 50.0 | 0 | 1,500 | 0 | 0 | 0 | 0 | 5.1 | 8.4 | 20 | 20 |
| PESIMISTA | 100 | 3,001 | 100.0 | 0 | 3,001 | 0 | 0 | 0 | 0 | 3.0 | 12 | 61 | 61 |
| PESIMISTA | 200 | 6,000 | 200.0 | 0 | 6,000 | 0 | 0 | 0 | 0 | 2.3 | 11 | 56 | 56 |
| PESIMISTA | 400 | 12,001 | 400.0 | 0 | 12,001 | 0 | 0 | 0 | 0 | 2.4 | 42 | 104 | 104 |
| OPTIMISTA | 25 | 751 | 25.0 | 0 | 751 | 0 | 0 | 0 | 0 | 7.9 | 13 | 42 | 42 |
| OPTIMISTA | 50 | 1,501 | 50.0 | 0 | 1,501 | 0 | 0 | 0 | 0 | 5.4 | 8.5 | 14 | 14 |
| OPTIMISTA | 100 | 3,001 | 100.0 | 0 | 3,001 | 0 | 0 | 0 | 0 | 3.0 | 6.4 | 25 | 25 |
| OPTIMISTA | 200 | 6,001 | 200.0 | 0 | 6,001 | 0 | 0 | 0 | 0 | 2.5 | 26 | 110 | 110 |
| OPTIMISTA | 400 | 12,000 | 400.0 | 0 | 12,000 | 0 | 0 | 0 | 0 | 2.6 | 38 | 114 | 114 |

## Ventas · agotamiento

El producto 32 con **poca existencia y más ventas que unidades**: tiene que venderse exactamente lo que había, ni una más. Sin calentamiento, que se llevaría las unidades antes del escalón medido. Existencia inicial: 300 unidades; escalones de 20 s.

| Estrategia | Tasa (/s) | Enviadas | Aceptadas/s | Descartadas por k6 | Aceptadas | Bloqueo (409) | Existencia insuficiente | Conflicto de versión | 5xx, red u otros | p50 (ms) | p95 (ms) | p99 (ms) | p99 aceptadas (ms) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| PESIMISTA | 50 | 1,001 | 15.0 | 0 | 300 | 0 | 701 | 0 | 0 | 3.9 | 7.4 | 28 | 71 |
| OPTIMISTA | 50 | 1,001 | 15.0 | 0 | 300 | 0 | 701 | 0 | 0 | 3.9 | 8.3 | 54 | 68 |

## Verificación contra la base

Después de cada corrida, `verificacion.sql` cuenta en la base de la sucursal lo que dejó k6 (los documentos llevan una referencia propia de la corrida).

| Estrategia | Escenario | Aceptadas (k6) | Documentos | Vendido (u.) | Existencia final mínima | Sobreventa (u.) | Existencia cuadra | Resultado |
|---|---|---:|---:|---:|---:|---:|---|---|
| PESIMISTA | caliente | 23,360 | 23,360 | 23,360 | 76,640 | 0 | sí | ✓ |
| OPTIMISTA | caliente | 17,483 | 17,483 | 17,483 | 82,517 | 0 | sí | ✓ |
| PESIMISTA | azar | 23,378 | 23,378 | 23,378 | 99,158 | 0 | sí | ✓ |
| OPTIMISTA | azar | 23,380 | 23,380 | 23,380 | 99,157 | 0 | sí | ✓ |
| PESIMISTA | agotamiento | 300 | 300 | 300 | 0 | 0 | sí | ✓ |
| OPTIMISTA | agotamiento | 300 | 300 | 300 | 0 | 0 | sí | ✓ |

## pgbench: la misma transacción, directo contra la base

`venta.sql` repite las sentencias de una venta de 1 unidad con la estrategia PESIMISTA, en el mismo orden, sobre una **copia** de la base de la sucursal (`pg_dump` a una base temporal que se borra al terminar). Sin HTTP, sin Node, sin pool: es lo más que dio la base sola para esta transacción, con los clientes probados.

| Escenario | Clientes | Transacciones | TPS | Latencia promedio (ms) | p50 (ms) | p95 (ms) | p99 (ms) | `SELECT … FOR UPDATE` (ms promedio) | Fallidas |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| caliente | 1 | 29,912 | 997 | 1.0 | 0.9 | 1.4 | 2.4 | 0.0 | 0 |
| caliente | 5 | 33,750 | 1,125 | 4.4 | 3.3 | 6.0 | 36 | 3.4 | 0 |
| caliente | 10 | 30,905 | 1,030 | 9.7 | 5.2 | 29 | 49 | 8.6 | 0 |
| caliente | 20 | 28,861 | 961 | 21 | 15 | 58 | 95 | 20 | 0 |
| azar | 1 | 30,473 | 1,016 | 1.0 | 0.9 | 1.3 | 2.3 | 0.1 | 0 |
| azar | 5 | 82,815 | 2,761 | 1.8 | 1.5 | 2.8 | 5.1 | 0.2 | 0 |
| azar | 10 | 105,530 | 3,518 | 2.8 | 2.3 | 5.1 | 8.9 | 0.5 | 0 |
| azar | 20 | 105,562 | 3,518 | 5.7 | 4.6 | 12 | 19 | 1.6 | 0 |

**k6 frente a pgbench.** En **caliente**, lo más alto que midió pgbench fue 1,125 TPS (con 5 clientes); a través de la API, lo más alto fue 399.4 ventas aceptadas/s (PESIMISTA, escalón de 400/s, p95 947 ms); en **azar**, lo más alto que midió pgbench fue 3,518 TPS (con 20 clientes); a través de la API, lo más alto fue 400.0 ventas aceptadas/s (PESIMISTA, escalón de 400/s, p95 42 ms; OPTIMISTA, escalón de 400/s, p95 38 ms). Los escalones de k6 piden una tasa fija: si la API la sostiene, las aceptadas/s quedan en la tasa pedida y no muestran el máximo. En la diferencia de latencia entre las dos herramientas entran HTTP, Node, el pool de conexiones de la API y la red de Docker y, en local, la CPU que la API y la base comparten con k6. No se midió cuánto aporta cada parte.

## Consulta de existencias de la red (api-central)

`GET /existencias/:codigo` con productos al azar. Lee `existencia_consolidada`, sin bloqueos.

| Tasa (/s) | Enviadas | Descartadas por k6 | Correctas | Errores | p50 (ms) | p95 (ms) | p99 (ms) |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 50 | 1,501 | 0 | 1,501 | 0 | 3.3 | 5.0 | 8.4 |
| 100 | 3,001 | 0 | 3,001 | 0 | 2.3 | 3.7 | 6.3 |
| 200 | 6,001 | 0 | 6,001 | 0 | 1.6 | 4.3 | 14 |

## Resultado: 47 de 47 verificaciones pasaron

- ✓ ventas PESIMISTA · caliente: documentos = ventas aceptadas por k6 (23360 documentos, 23360 aceptadas)
- ✓ ventas PESIMISTA · caliente: un movimiento y una fila de outbox por documento (23360 movimientos, 23360 outbox)
- ✓ ventas PESIMISTA · caliente: unidades: movimientos = líneas = vendido (23360 / 23360 / 23360)
- ✓ ventas PESIMISTA · caliente: bitácora fila por fila (antes − cantidad = después) (0 no cuadran)
- ✓ ventas PESIMISTA · caliente: existencia final = inicial − vendido (0 de 1 productos no cuadran)
- ✓ ventas PESIMISTA · caliente: sobreventa = 0 (0 unidades)
- ✓ ventas PESIMISTA · caliente: sin 5xx, sin errores de red ni otros 4xx (5xx 0, red 0, otros 0)
- ✓ ventas OPTIMISTA · caliente: documentos = ventas aceptadas por k6 (17483 documentos, 17483 aceptadas)
- ✓ ventas OPTIMISTA · caliente: un movimiento y una fila de outbox por documento (17483 movimientos, 17483 outbox)
- ✓ ventas OPTIMISTA · caliente: unidades: movimientos = líneas = vendido (17483 / 17483 / 17483)
- ✓ ventas OPTIMISTA · caliente: bitácora fila por fila (antes − cantidad = después) (0 no cuadran)
- ✓ ventas OPTIMISTA · caliente: existencia final = inicial − vendido (0 de 1 productos no cuadran)
- ✓ ventas OPTIMISTA · caliente: sobreventa = 0 (0 unidades)
- ✓ ventas OPTIMISTA · caliente: sin 5xx, sin errores de red ni otros 4xx (5xx 0, red 0, otros 0)
- ✓ ventas PESIMISTA · azar: documentos = ventas aceptadas por k6 (23378 documentos, 23378 aceptadas)
- ✓ ventas PESIMISTA · azar: un movimiento y una fila de outbox por documento (23378 movimientos, 23378 outbox)
- ✓ ventas PESIMISTA · azar: unidades: movimientos = líneas = vendido (23378 / 23378 / 23378)
- ✓ ventas PESIMISTA · azar: bitácora fila por fila (antes − cantidad = después) (0 no cuadran)
- ✓ ventas PESIMISTA · azar: existencia final = inicial − vendido (0 de 30 productos no cuadran)
- ✓ ventas PESIMISTA · azar: sobreventa = 0 (0 unidades)
- ✓ ventas PESIMISTA · azar: sin 5xx, sin errores de red ni otros 4xx (5xx 0, red 0, otros 0)
- ✓ ventas OPTIMISTA · azar: documentos = ventas aceptadas por k6 (23380 documentos, 23380 aceptadas)
- ✓ ventas OPTIMISTA · azar: un movimiento y una fila de outbox por documento (23380 movimientos, 23380 outbox)
- ✓ ventas OPTIMISTA · azar: unidades: movimientos = líneas = vendido (23380 / 23380 / 23380)
- ✓ ventas OPTIMISTA · azar: bitácora fila por fila (antes − cantidad = después) (0 no cuadran)
- ✓ ventas OPTIMISTA · azar: existencia final = inicial − vendido (0 de 30 productos no cuadran)
- ✓ ventas OPTIMISTA · azar: sobreventa = 0 (0 unidades)
- ✓ ventas OPTIMISTA · azar: sin 5xx, sin errores de red ni otros 4xx (5xx 0, red 0, otros 0)
- ✓ ventas PESIMISTA · agotamiento: documentos = ventas aceptadas por k6 (300 documentos, 300 aceptadas)
- ✓ ventas PESIMISTA · agotamiento: un movimiento y una fila de outbox por documento (300 movimientos, 300 outbox)
- ✓ ventas PESIMISTA · agotamiento: unidades: movimientos = líneas = vendido (300 / 300 / 300)
- ✓ ventas PESIMISTA · agotamiento: bitácora fila por fila (antes − cantidad = después) (0 no cuadran)
- ✓ ventas PESIMISTA · agotamiento: existencia final = inicial − vendido (0 de 1 productos no cuadran)
- ✓ ventas PESIMISTA · agotamiento: sobreventa = 0 (0 unidades)
- ✓ ventas PESIMISTA · agotamiento: sin 5xx, sin errores de red ni otros 4xx (5xx 0, red 0, otros 0)
- ✓ ventas PESIMISTA · agotamiento: se vendió exactamente lo que había (vendido 300 de 300)
- ✓ ventas OPTIMISTA · agotamiento: documentos = ventas aceptadas por k6 (300 documentos, 300 aceptadas)
- ✓ ventas OPTIMISTA · agotamiento: un movimiento y una fila de outbox por documento (300 movimientos, 300 outbox)
- ✓ ventas OPTIMISTA · agotamiento: unidades: movimientos = líneas = vendido (300 / 300 / 300)
- ✓ ventas OPTIMISTA · agotamiento: bitácora fila por fila (antes − cantidad = después) (0 no cuadran)
- ✓ ventas OPTIMISTA · agotamiento: existencia final = inicial − vendido (0 de 1 productos no cuadran)
- ✓ ventas OPTIMISTA · agotamiento: sobreventa = 0 (0 unidades)
- ✓ ventas OPTIMISTA · agotamiento: sin 5xx, sin errores de red ni otros 4xx (5xx 0, red 0, otros 0)
- ✓ ventas OPTIMISTA · agotamiento: se vendió exactamente lo que había (vendido 300 de 300)
- ✓ central: consulta de existencias sin errores (0 errores)
- ✓ pgbench: sin transacciones fallidas ni clientes abortados (ninguna)
- ✓ sincronización: el central cuadra con las sucursales al final (verificar.mjs) (✔ El central cuadra con las sucursales)

<!-- GCP:INICIO -->
## En GCP

> Generado por `infra/gcp/carga-gcp.sh` el 2026-10-22 a partir de `20261022-181714-gcp` (perfil corto).

**Dónde corre cada cosa.** k6 de ventas corre en `sucursal-2` y le pega a `sucursal-1` por IP interna: el generador de carga
y la sucursal están en VMs distintas (e2-small). La base de la sucursal está en la misma VM que su API, como en producción.
k6 de la consulta del central corre en Cloud Shell, con token de identidad (api-central es privado): su latencia incluye
el viaje de Cloud Shell a Cloud Run. pgbench corre dentro del contenedor de la base de `sucursal-1`, sobre una copia.
Solo la estrategia de producción (`PESIMISTA`). Perfil corto, para no gastar crédito.

### Ventas · caliente

Todas las ventas sobre **un solo producto** (el 31): todas compiten por la misma fila de `existencia`. Existencia inicial: 100,000 unidades; escalones de 30 s.

| Estrategia | Tasa (/s) | Enviadas | Aceptadas/s | Descartadas por k6 | Aceptadas | Bloqueo (409) | Existencia insuficiente | Conflicto de versión | 5xx, red u otros | p50 (ms) | p95 (ms) | p99 (ms) | p99 aceptadas (ms) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| PESIMISTA | 20 | 601 | 20.0 | 0 | 601 | 0 | 0 | 0 | 0 | 10 | 13 | 28 | 28 |
| PESIMISTA | 50 | 1,501 | 50.0 | 0 | 1,501 | 0 | 0 | 0 | 0 | 9.8 | 13 | 193 | 193 |

### Ventas · agotamiento

El producto 32 con **poca existencia y más ventas que unidades**: tiene que venderse exactamente lo que había, ni una más. Sin calentamiento, que se llevaría las unidades antes del escalón medido. Existencia inicial: 200 unidades; escalones de 20 s.

| Estrategia | Tasa (/s) | Enviadas | Aceptadas/s | Descartadas por k6 | Aceptadas | Bloqueo (409) | Existencia insuficiente | Conflicto de versión | 5xx, red u otros | p50 (ms) | p95 (ms) | p99 (ms) | p99 aceptadas (ms) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| PESIMISTA | 30 | 601 | 10.0 | 0 | 200 | 0 | 401 | 0 | 0 | 5.0 | 10 | 11 | 13 |

### Verificación contra la base

Después de cada corrida, `verificacion.sql` cuenta en la base de la sucursal lo que dejó k6 (los documentos llevan una referencia propia de la corrida).

| Estrategia | Escenario | Aceptadas (k6) | Documentos | Vendido (u.) | Existencia final mínima | Sobreventa (u.) | Existencia cuadra | Resultado |
|---|---|---:|---:|---:|---:|---:|---|---|
| PESIMISTA | caliente | 2,203 | 2,203 | 2,203 | 97,797 | 0 | sí | ✓ |
| PESIMISTA | agotamiento | 200 | 200 | 200 | 0 | 0 | sí | ✓ |

### pgbench: la misma transacción, directo contra la base

`venta.sql` repite las sentencias de una venta de 1 unidad con la estrategia PESIMISTA, en el mismo orden, sobre una **copia** de la base de la sucursal (`pg_dump` a una base temporal que se borra al terminar). Sin HTTP, sin Node, sin pool: es lo más que dio la base sola para esta transacción, con los clientes probados.

| Escenario | Clientes | Transacciones | TPS | Latencia promedio (ms) | p50 (ms) | p95 (ms) | p99 (ms) | `SELECT … FOR UPDATE` (ms promedio) | Fallidas |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| caliente | 1 | 4,527 | 226 | 4.4 | 4.3 | 4.8 | 5.3 | 0.2 | 0 |
| caliente | 10 | 4,893 | 245 | 41 | 20 | 134 | 220 | 36 | 0 |

**k6 frente a pgbench.** En **caliente**, lo más alto que midió pgbench fue 245 TPS (con 10 clientes); a través de la API, lo más alto fue 50.0 ventas aceptadas/s (PESIMISTA, escalón de 50/s, p95 13 ms). Los escalones de k6 piden una tasa fija: si la API la sostiene, las aceptadas/s quedan en la tasa pedida y no muestran el máximo. En la diferencia de latencia entre las dos herramientas entran HTTP, Node, el pool de conexiones de la API y la red de la VPC entre `sucursal-2` (k6) y `sucursal-1`. No se midió cuánto aporta cada parte.

### Consulta de existencias de la red (api-central)

`GET /existencias/:codigo` con productos al azar. Lee `existencia_consolidada`, sin bloqueos.

| Tasa (/s) | Enviadas | Descartadas por k6 | Correctas | Errores | p50 (ms) | p95 (ms) | p99 (ms) |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 20 | 601 | 0 | 601 | 0 | 47 | 53 | 65 |
| 50 | 1,500 | 0 | 1,500 | 0 | 47 | 51 | 65 |

### Resultado: 17 de 17 verificaciones pasaron

- ✓ ventas PESIMISTA · caliente: documentos = ventas aceptadas por k6 (2203 documentos, 2203 aceptadas)
- ✓ ventas PESIMISTA · caliente: un movimiento y una fila de outbox por documento (2203 movimientos, 2203 outbox)
- ✓ ventas PESIMISTA · caliente: unidades: movimientos = líneas = vendido (2203 / 2203 / 2203)
- ✓ ventas PESIMISTA · caliente: bitácora fila por fila (antes − cantidad = después) (0 no cuadran)
- ✓ ventas PESIMISTA · caliente: existencia final = inicial − vendido (0 de 1 productos no cuadran)
- ✓ ventas PESIMISTA · caliente: sobreventa = 0 (0 unidades)
- ✓ ventas PESIMISTA · caliente: sin 5xx, sin errores de red ni otros 4xx (5xx 0, red 0, otros 0)
- ✓ ventas PESIMISTA · agotamiento: documentos = ventas aceptadas por k6 (200 documentos, 200 aceptadas)
- ✓ ventas PESIMISTA · agotamiento: un movimiento y una fila de outbox por documento (200 movimientos, 200 outbox)
- ✓ ventas PESIMISTA · agotamiento: unidades: movimientos = líneas = vendido (200 / 200 / 200)
- ✓ ventas PESIMISTA · agotamiento: bitácora fila por fila (antes − cantidad = después) (0 no cuadran)
- ✓ ventas PESIMISTA · agotamiento: existencia final = inicial − vendido (0 de 1 productos no cuadran)
- ✓ ventas PESIMISTA · agotamiento: sobreventa = 0 (0 unidades)
- ✓ ventas PESIMISTA · agotamiento: sin 5xx, sin errores de red ni otros 4xx (5xx 0, red 0, otros 0)
- ✓ ventas PESIMISTA · agotamiento: se vendió exactamente lo que había (vendido 200 de 200)
- ✓ central: consulta de existencias sin errores (0 errores)
- ✓ pgbench: sin transacciones fallidas ni clientes abortados (ninguna)

<!-- GCP:FIN -->
