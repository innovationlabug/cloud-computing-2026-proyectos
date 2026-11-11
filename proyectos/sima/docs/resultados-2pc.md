# Resultados del commit en dos fases (traslados)

> Generado por `scripts/demo-2pc/ejecutar.sh` el 2026-11-02 a partir de `20261102-125322.jsonl`.
> Se regenera completo en cada corrida; el análisis de decisiones va en `docs/decisiones.md`.

## Qué se probó

Un traslado de mercadería de la sucursal 1 (origen) a la sucursal 2 (destino) toca dos bases de datos distintas y no puede
quedar a medias. El coordinador (central) ejecuta un commit en dos fases sobre las transacciones preparadas de PostgreSQL
(`PREPARE TRANSACTION`, `COMMIT PREPARED`, `ROLLBACK PREPARED`), con su bitácora (`transaccion_global`, `participante`)
escrita **antes** de cada paso. Cada escenario se ejecuta contra el sistema real (Docker Compose: cuatro PostgreSQL 16 y
cuatro APIs) y termina con una verificación contra las bases:

- **Bitácora** del coordinador: estado final de la transacción global y del traslado.
- **Origen y destino**: documentos del traslado en cada base, existencia antes y después, y si queda alguna
  transacción preparada.
- **Conservación de unidades en la red**: Σ existencias de las tres sucursales + unidades en tránsito = Σ semilla +
  movimientos que no son de traslado, con los datos de las sucursales y con los del central. Si el traslado se
  aplicara en una punta y no en la otra, aparecerían o desaparecerían unidades.
- **Conservación del VALOR** (costo promedio): Σ valor de las existencias + valor en tránsito = Σ saldos iniciales +
  recepciones + ajustes − costo de ventas − mermas. En un traslado, lo que sale del origen (a su costo promedio) es,
  al centavo, lo que entra al destino; el costo viaja en el voto del origen y de ahí al destino.
- **`verificar.mjs` completo**: el central idéntico a cada sucursal, sin duplicados, bitácora finalizada.

La preparación es **en secuencia**: primero el origen, cuyo voto SÍ trae el costo de lo que sale; el coordinador lo
escribe en su bitácora y solo entonces le pide preparar al destino, con ese costo en el cuerpo (firmado).

Las fallas del coordinador se inyectan con `_prueba.falla` en `POST /traslados`, que solo se acepta con
`PERMITIR_FALLAS_INYECTADAS=1`. Para que la demostración dure segundos, el plazo de `INICIADO` de la recuperación
se baja a 8 s (60 s por defecto) y el del reconciliador a 5 s (30 s por defecto).

## Escenarios

| # | Escenario | Bitácora final | Traslado | Origen | Destino | Preparadas al final | Conservación | Duración | Resultado |
|---|---|---|---|---|---|:---:|:---:|---:|:---:|
| 1 | Camino feliz y recepción | FINALIZADO (1 intento) | **RECIBIDO** | TRASLADO (EN_TRANSITO); existencia 44 → 40 (-4) | TRASLADO (RECIBIDO), RECEPCION (—); existencia 47 → 51 (+4) | 0 | ✓ | 1.6 s | ✓ |
| 2 | Voto NO: el origen no tiene existencia suficiente | FINALIZADO (1 intento) | **ABORTADO** | ninguno; existencia 51 → 51 (±0) | ninguno; existencia 54 → 54 (±0) | 0 | ✓ | 0.3 s | ✓ |
| 3 | Destino sin enlace: SOLICITADO; al volver, la recuperación lo retoma | FINALIZADO (1 intento) | **COMPROMETIDO** | TRASLADO (EN_TRANSITO); existencia 57 → 55 (-2) | TRASLADO (EN_TRANSITO); existencia 61 → 61 (±0) | 0 | ✓ | 9.4 s | ✓ |
| 4 | El coordinador muere después de escribir COMPROMETIDO | FINALIZADO (1 intento) | **COMPROMETIDO** | TRASLADO (EN_TRANSITO); existencia 12 → 9 (-3) | TRASLADO (EN_TRANSITO); existencia 15 → 15 (±0) | 0 | ✓ | 2.1 s | ✓ |
| 5 | El coordinador muere en INICIADO con los dos SÍ | FINALIZADO (1 intento) | **ABORTADO** | ninguno; existencia 19 → 19 (±0) | ninguno; existencia 22 → 22 (±0) | 0 | ✓ | 9.3 s | ✓ |
| 6 | El coordinador queda caído con un COMPROMETIDO en duda | FINALIZADO (1 intento) | **COMPROMETIDO** | TRASLADO (EN_TRANSITO); existencia 26 → 24 (-2) | TRASLADO (EN_TRANSITO); existencia 29 → 29 (±0) | 0 | ✓ | 12.7 s | ✓ |
| 7 | El coordinador muere después del voto SÍ del origen, con los costos ya guardados | FINALIZADO (1 intento) | **ABORTADO** | ninguno; existencia 33 → 32 (-1) | ninguno; existencia 36 → 36 (±0) | 0 | ✓ | 10.2 s | ✓ |

Detalle de cada escenario:

- **1. Camino feliz y recepción.** Participantes al final: ORIGEN:SI/CONFIRMADA DESTINO:SI/CONFIRMADA. Costo de 4 u.: salió 89.6 del origen y entró 89.6 al destino.
- **2. Voto NO: el origen no tiene existencia suficiente.** Participantes al final: ORIGEN:NO/ABORTADA DESTINO:—/ABORTADA.
- **3. Destino sin enlace: SOLICITADO; al volver, la recuperación lo retoma.** Participantes al final: ORIGEN:SI/CONFIRMADA DESTINO:SI/CONFIRMADA. Retomado 1.4 s después de reconectar.
- **4. El coordinador muere después de escribir COMPROMETIDO.** Participantes al final: ORIGEN:SI/CONFIRMADA DESTINO:SI/CONFIRMADA. En duda tras la caída: origen 1, destino 1; resuelto 0.7 s después.
- **5. El coordinador muere en INICIADO con los dos SÍ.** Participantes al final: ORIGEN:SI/ABORTADA DESTINO:SI/ABORTADA. Venta del producto: 409; abortado por plazo 5.9 s después.
- **6. El coordinador queda caído con un COMPROMETIDO en duda.** Participantes al final: ORIGEN:SI/CONFIRMADA DESTINO:SI/CONFIRMADA. 7 s caído: en duda origen 1, destino 1; venta 409; reconciliador sin respuesta, no deshizo nada; resuelto 0.6 s después de volver.
- **7. El coordinador muere después del voto SÍ del origen, con los costos ya guardados.** Participantes al final: ORIGEN:SI/ABORTADA DESTINO:—/ABORTADA. Costos guardados antes de caer; origen preparado, destino sin consultar; abortado por plazo 5.7 s después.

## Conservación de unidades en la red

En **los 7 escenarios** la ecuación cerró producto por producto (50 productos), con los datos de las sucursales y con los del central.

| # | Escenario | En tránsito al terminar | Conservación | verificar.mjs |
|---|---|---:|:---:|:---:|
| 1 | Camino feliz y recepción | 10 u. | ✓ | ✓ |
| 2 | Voto NO: el origen no tiene existencia suficiente | 10 u. | ✓ | ✓ |
| 3 | Destino sin enlace: SOLICITADO; al volver, la recuperación lo retoma | 12 u. | ✓ | ✓ |
| 4 | El coordinador muere después de escribir COMPROMETIDO | 15 u. | ✓ | ✓ |
| 5 | El coordinador muere en INICIADO con los dos SÍ | 15 u. | ✓ | ✓ |
| 6 | El coordinador queda caído con un COMPROMETIDO en duda | 17 u. | ✓ | ✓ |
| 7 | El coordinador muere después del voto SÍ del origen, con los costos ya guardados | 17 u. | ✓ | ✓ |

Las unidades en tránsito son acumuladas: las de un traslado comprometido sin recepción quedan en tránsito en los
escenarios siguientes, y la ecuación las incluye.

## Lectura de los resultados

- **Camino feliz.** Las dos puntas confirmaron; el origen bajó 4 u., que quedaron en tránsito hasta la recepción local del destino (operación local con su propio outbox, no una segunda ronda de 2PC).
- **Voto NO.** El origen no tenía existencia y votó NO; el coordinador escribió ABORTADO y nada quedó escrito ni preparado en ninguna base.
- **Sucursal fuera de línea.** El traslado esperó en SOLICITADO sin pedir votos (sección 5.4) y la recuperación lo retomó al volver el enlace (retomado 1.4 s después de reconectar).
- **Caída después de decidir.** La decisión ya estaba en la bitácora, que es el punto de no retorno: al reiniciar, la recuperación la leyó y reenvió COMMIT PREPARED (en duda tras la caída: origen 1, destino 1; resuelto 0.7 s después).
- **Caída antes de decidir.** Nadie pudo haber confirmado, así que lo seguro es abortar: la recuperación lo hizo al vencer el plazo de INICIADO. Mientras tanto la fila del origen estuvo retenida (venta del producto: 409; abortado por plazo 5.9 s después). Es el bloqueo de la sección 5.4: acotado por el plazo y visible.
- **Coordinador caído.** Los participantes no pueden decidir solos: el reconciliador preguntó, no obtuvo respuesta y **esperó sin deshacer**, porque deshacer un COMPROMETIDO sería aplicar el traslado en una sola punta. Al volver el coordinador, se resolvió (7 s caído: en duda origen 1, destino 1; venta 409; reconciliador sin respuesta, no deshizo nada; resuelto 0.6 s después de volver).
- **Caída después del voto SÍ del origen.** Los costos ya estaban en la bitácora del coordinador y, de forma persistente, en el origen; al destino nunca se le preguntó. Lo seguro y lo más simple es abortar: la recuperación lo hizo al vencer el plazo de INICIADO, igual que en el escenario 5, en lugar de retomar preguntándole al destino con los mismos costos (costos guardados antes de caer; origen preparado, destino sin consultar; abortado por plazo 5.7 s después).

**7 de 7 escenarios** terminaron con el traslado aplicado en las dos puntas o en ninguna, sin transacciones preparadas y con las unidades de la red conservadas.

## Cómo reproducir

```bash
docker compose down -v && docker compose up -d --build && ./scripts/migrar.sh && ./scripts/sembrar.sh
./scripts/demo-2pc/ejecutar.sh                  # los 7 escenarios y este informe
./scripts/demo-2pc/ejecutar.sh --escenario 6    # uno solo, para mostrar en vivo
```

Resultados crudos de esta corrida (un JSON por línea): `scripts/demo-2pc/resultados/20261102-125322.jsonl` (no versionado).
