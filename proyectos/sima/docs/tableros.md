# Tableros de SIMA en Looker Studio

Los tableros leen **solo** de BigQuery, de las vistas del dataset `sima_analitica` del
proyecto `<id-proyecto>`. Ninguno consulta Cloud SQL ni la base de una sucursal
(propuesta, sección 4.5). Las vistas las crea `infra/gcp/09-bigquery.sh`; los tableros se
arman a mano con lo que describe esta página.

## De dónde salen los datos

```
Cloud SQL (central) ──Datastream (CDC, cada 15 min)──► sima_cdc.public_<tabla>
                                                          │ MERGE completo cada hora
historia_simulada (6 meses) ─────────────────────────────►├─► hecho_movimiento ─► vistas v_*
                                                          └─► dim_producto, dim_sucursal, dim_proveedor (vistas), dim_fecha
```

Un movimiento tarda **hasta unos 75 minutos** en verse en un tablero: hasta 15 de
Datastream, hasta 60 de la consulta programada y lo que tenga la caché de Looker Studio.
Las vistas que leen `sima_cdc` directamente (salud de sincronización y traslados) no
esperan la consulta programada.

## La columna `origen`

Todas las vistas llevan `origen`, con dos valores:

- **`CDC`**: lo que de verdad ocurrió en SIMA y llegó por Datastream.
- **`SIMULADO`**: seis meses de movimientos sintéticos de las 3 sucursales y los 50
  productos de la semilla. Existen para que los tableros tengan historia que mostrar. Nunca
  se escribieron en Cloud SQL ni en las sucursales.

**Cada página lleva un control de filtro sobre `origen`.** Los dos orígenes no se suman:
la existencia simulada y la real son universos distintos de los mismos productos. Para la
presentación conviene dejar `SIMULADO` como valor predeterminado y cambiar a `CDC` para
mostrar lo que entró en vivo.

Cada origen tiene su **fecha de referencia** (vista `v_fecha_referencia`), desde la que se
cuentan "los últimos 30 días": hoy, en hora de Guatemala, para `CDC`, y el último día de la
historia para `SIMULADO`, que termina el día anterior a su generación.

**La recepción de proveedor ya existe en el flujo operativo de SIMA** (`POST /recepciones`, pantalla
«Recepción» de la caja). Cada recepción real lleva `id_proveedor` en su documento y sus movimientos son
`RECEPCION_ENTRADA`, así que con `origen = CDC` los gráficos por proveedor muestran datos reales: los
de las recepciones que se hayan registrado. La historia `SIMULADO` sigue teniendo su propia reposición
con proveedor sintético. Una recepción de un **traslado** no cuenta aquí: su movimiento es
`TRASLADO_ENTRADA` y no lleva proveedor. Mientras no haya recepciones reales, con `CDC` todo cae en
«Sin proveedor».

## Cómo conectarlo

1. En [lookerstudio.google.com](https://lookerstudio.google.com), **Crear → Fuente de datos → BigQuery**.
2. **Mis proyectos → `<id-proyecto>` → `sima_analitica`** y elegir la vista.
   Una fuente de datos por vista; las seis de abajo cubren todos los tableros.
3. En la fuente de datos:
   - **Credenciales:** las del propietario. Quien vea el tablero no necesita permisos en BigQuery.
   - **Actualización de los datos:** 1 hora. Más seguido no sirve: el hecho cambia una vez por hora.
   - Revisar los tipos de algunos campos: `fecha` como *Fecha*; `importe`, `valor_a_precio_venta` y
     `precio_venta` como *Moneda (GTQ)*; `id_*` como *Texto*, para que no se sumen.
4. En `v_movimientos`, usar `fecha` como **dimensión de período**. El hecho está
   particionado por `fecha`, así que el filtro de fechas del tablero solo lee esas particiones.
5. Crear el informe, agregar la fuente y, en cada página, un **control desplegable sobre `origen`**.

## Tablero 1 — Movimientos y ventas

Fuente: **`v_movimientos`**. Un renglón por movimiento, con fecha, sucursal, producto, categoría y proveedor.

| Gráfico | Configuración |
|---|---|
| Serie temporal de ventas | Dimensión `fecha`; métrica `SUM(importe)`; filtro `tipo = VENTA` |
| Barras por sucursal | Dimensión `sucursal`; métricas `SUM(importe)` y `SUM(cantidad)`; filtro `tipo = VENTA` |
| Ventas por categoría | Gráfico circular o barras; dimensión `categoria`; métrica `SUM(importe)` |
| Top productos | Tabla; dimensión `producto`; métricas `SUM(cantidad)` y `SUM(importe)`; orden descendente; 15 filas |
| Entradas y salidas | Columnas apiladas; dimensión `inicio_semana`; desglose `direccion`; métrica `SUM(cantidad)` |
| Recepciones por proveedor | Barras; dimensión `proveedor`; métrica `SUM(cantidad)`; filtro `tipo = RECEPCION_ENTRADA`. Con `CDC` tiene datos cuando se registran recepciones reales; con `SIMULADO`, la historia sintética |
| Ventas por día de la semana | Barras; dimensión `nombre_dia` (orden por `dia_semana`); métrica `SUM(importe)` |

Controles: `origen`, período (`fecha`), `sucursal` y `categoria`.

## Tablero 2 — Rotación

Fuente: **`v_rotacion`**. Una fila por origen, ventana (30 o 90 días), sucursal y producto.

- `unidades_vendidas`: ventas de la ventana.
- `existencia_promedio`: promedio de las existencias de cierre diarias de la ventana (ver `v_cierre_diario`).
- `rotacion` = `unidades_vendidas / existencia_promedio`.
- `venta_diaria` = `unidades_vendidas / periodo_dias`.
- `dias_inventario` = `existencia_actual / venta_diaria`. Nulo si no hubo ventas.

| Gráfico | Configuración |
|---|---|
| Tabla de rotación | Dimensiones `sucursal`, `producto`, `categoria`; métricas `rotacion`, `venta_diaria`, `existencia_promedio`, `dias_inventario`; formato condicional en `rotacion` (rojo si < 1) |
| Tabla dinámica sucursal × categoría | Filas `categoria`, columnas `sucursal`, métrica `AVG(rotacion)` |
| Dispersión | Dimensión `producto`; X `existencia_promedio`, Y `unidades_vendidas`. Los de abajo a la derecha son los que no rotan |
| Menor rotación | Barras horizontales; `producto`; `AVG(rotacion)` ascendente; 10 filas |

Controles: `origen` y **`periodo_dias` (obligatorio, con valor predeterminado 30)**, para no
mezclar las dos ventanas. Además, `sucursal` y `categoria`.

Para la evolución de la existencia de un producto, agregar la fuente **`v_cierre_diario`**:
serie temporal de `fecha` con métrica `MAX(existencia_cierre)`, desglose por `id_sucursal`
y controles de `origen` e `id_producto`. `fuente_cierre` indica de dónde sale cada valor:
un movimiento de ese día o arrastrado, el `cantidad_antes` del primer movimiento conocido,
o la existencia actual de un par que no tiene movimientos.

## Tablero 3 — Capital dormido

Fuente: **`v_capital_dormido`**. Existencia mayor que cero, con días sin venta y valor.

| Gráfico | Configuración |
|---|---|
| Indicador | `SUM(valor_a_precio_venta)` con el filtro de N días aplicado |
| Barras por sucursal | Dimensión `sucursal`; métrica `SUM(valor_a_precio_venta)` |
| Tabla | `sucursal`, `producto`, `existencia`, `ultima_venta`, `dias_sin_venta`, `valor_a_precio_venta`; orden por valor descendente |

N días: un **control deslizante o de entrada sobre `dias_sin_venta`** (por ejemplo, de 30 a
180, con 60 como valor inicial), que filtra `dias_sin_venta >= N`. Los pares que nunca
vendieron tienen `sin_ventas_registradas = verdadero` y también entran al deslizador:
`dias_sin_venta` cuenta desde el primer movimiento de ese producto en esa sucursal, o, si no
tienen ninguno, desde el inicio de la historia de su origen (el primer día con movimientos).
Un control sobre `sin_ventas_registradas` separa los que nunca vendieron de los que dejaron
de vender.

**Límite que el tablero tiene que decir en un texto visible:** el modelo no tiene costo
unitario, así que el capital se valoriza a `precio_venta`. Esto **sobrestima** el capital
invertido en el margen de cada producto. Es el valor de venta inmovilizado, no el costo.

## Tablero 4 — Sugerencia de reposición

Fuente: **`v_reposicion`**. Una fila por sucursal y producto.

- `venta_diaria_30`: venta diaria promedio de los últimos 30 días.
- `necesidad_cobertura` = `venta_diaria_30 × dias_cobertura` (14 días).
- `sugerido` = lo que falta para llegar al mayor entre `necesidad_cobertura` y `stock_min`,
  redondeado hacia arriba y nunca negativo.
- `bajo_minimo`: la existencia está por debajo de `stock_min`.
- `dias_cobertura_actual`: cuántos días alcanza la existencia al ritmo actual.

| Gráfico | Configuración |
|---|---|
| Indicadores | `COUNT` con `bajo_minimo = verdadero`; `SUM(sugerido)` |
| Tabla de pedido | `sucursal`, `producto`, `existencia`, `stock_min`, `venta_diaria_30`, `sugerido`, `dias_cobertura_actual`; filtro `sugerido > 0`; formato condicional en `bajo_minimo` |
| Barras | Dimensión `sucursal`; métrica `SUM(sugerido)`; desglose `categoria` |

## Tablero 5 — Salud de la sincronización

Fuentes: **`v_salud_sincronizacion`** (una fila por sucursal) y **`v_traslados_pendientes`**.
Solo tienen `origen = CDC`: la historia simulada no tiene mensajes ni traslados.

| Gráfico | Configuración |
|---|---|
| Tabla por sucursal | `sucursal`, `ultimo_mensaje`, `minutos_desde_ultimo_mensaje`, `mensajes_24h`, `traslados_no_finalizados`, `traslados_en_duda`; formato condicional en minutos (ámbar > 30, rojo > 120) |
| Indicador | `MAX(cdc_ultimo_cambio)`: el cambio más reciente que Datastream ya dejó en BigQuery. Es la antigüedad del dato del tablero |
| Tabla de traslados | De `v_traslados_pendientes`: `sucursal_origen`, `sucursal_destino`, `estado_traslado`, `estado_transaccion`, `situacion`, `minutos_abierto`, `intentos`; orden por minutos descendente |
| Barras | `situacion` (`ESPERANDO_SUCURSALES`, `EN_TRANSITO`, `EN_DUDA`) por cantidad de traslados |

`situacion`:

- **`ESPERANDO_SUCURSALES`**: el traslado está `SOLICITADO` y espera que las dos sucursales estén en línea.
- **`EN_TRANSITO`**: está comprometido y la decisión ya llegó a los participantes; falta la recepción en el destino.
- **`EN_DUDA`**: se está preparando, o el coordinador decidió y la decisión todavía no llegó a algún participante.

**Límites que el tablero tiene que decir:**

- Las operaciones pendientes en la outbox de cada sucursal no llegan al central: viven en
  la base de la sucursal. Este tablero ve cuándo llegó el último mensaje de cada sucursal,
  pero no cuántos tiene encolados. Para ese número está `/salud` de cada sucursal.
- `minutos_desde_ultimo_mensaje` también crece si la sucursal simplemente no vendió. Un
  número alto es una alerta para mirar `/salud` de esa sucursal, no prueba de una falla.

## En la variante autoalojada (Metabase)

> La variante autoalojada (rama `infra/on-premise`) usa Metabase en lugar de Looker Studio, con los mismos cinco
> tableros y las mismas vistas (`v_*`). El script que los crea y las consultas están en `infra/on-premise/metabase/`.

Las vistas viven en `db-analitica` (esquema `analitica`). Los tableros se conectan con `sima_tableros`, un usuario
que lee esas vistas y no lee las tablas replicadas de `public` ni escribe en `analitica`. Metabase está solo en la
red `sima-red-tableros`: no resuelve `db-central` ni ninguna sucursal, así que un tablero no puede consultar la base
operativa. Las preguntas son SQL nativo sobre las vistas, porque el constructor de consultas guarda ids propios de
cada instancia de Metabase. Los tableros se buscan por nombre: correr el script dos veces no duplica nada.

Usuario de demostración, solo para la demostración: `admin@sima.example` / `[REDACTADO]`. Abrir
`http://localhost:3100`.

### Controles

Cada tablero lleva sus controles enlazados a las variables de las preguntas. `Origen` es obligatorio y empieza en
`SIMULADO` (la historia de seis meses); `CDC` es lo que de verdad entró.

| Tablero | Controles |
|---|---|
| Movimientos y ventas | Origen · Desde · Hasta · Sucursal · Categoría |
| Rotación | Origen · **Período (días)**, obligatorio, 30 por defecto (las ventanas de 30 y 90 no se mezclan) · Sucursal · Categoría · Id de producto (solo la serie de existencia) |
| Capital dormido | Origen · **Días sin venta (mínimo)**, 60 por defecto · Sucursal · Sin ventas registradas (`si` / `no`) |
| Reposición | Origen · Sucursal · Categoría |
| Salud de la sincronización | ninguno (solo `CDC`) |

### Correspondencia con los gráficos

**Tablero 1 · Movimientos y ventas** (`v_movimientos`)

| En este documento | Pregunta de Metabase |
|---|---|
| Serie temporal de ventas | Movimientos · Ventas por día (línea) |
| Barras por sucursal | Movimientos · Ventas por sucursal (barras: importe y unidades) |
| Ventas por categoría | Movimientos · Ventas por categoría (circular) |
| Top productos | Movimientos · Top 15 productos (tabla) |
| Entradas y salidas | Movimientos · Entradas y salidas por semana (barras apiladas por `direccion`) |
| Recepciones por proveedor | Movimientos · Recepciones por proveedor (barras) |
| Ventas por día de la semana | Movimientos · Ventas por día de la semana (barras, orden por `dia_semana`) |

**Tablero 2 · Rotación** (`v_rotacion`, y `v_cierre_diario` para la serie)

| En este documento | Pregunta de Metabase |
|---|---|
| Tabla de rotación (rojo si `rotacion` < 1) | Rotación · Tabla por producto y sucursal (formato condicional) |
| Tabla dinámica sucursal × categoría | Rotación · Sucursal por categoría (tabla dinámica, `AVG(rotacion)`) |
| Dispersión | Rotación · Existencia promedio contra unidades vendidas |
| Menor rotación | Rotación · Los 10 de menor rotación (barras horizontales) |
| Serie de `v_cierre_diario` | Rotación · Existencia de cierre diaria de un producto (línea por sucursal) |

**Tablero 3 · Capital dormido** (`v_capital_dormido`)

| En este documento | Pregunta de Metabase |
|---|---|
| Indicador | Capital dormido · Valor a precio de venta |
| Barras por sucursal | Capital dormido · Por sucursal |
| Tabla | Capital dormido · Detalle por producto |
| Texto visible del límite | Tarjeta de texto: el capital se valoriza a precio de venta y sobrestima el invertido |

**Tablero 4 · Reposición** (`v_reposicion`)

| En este documento | Pregunta de Metabase |
|---|---|
| Indicadores | Reposición · Productos bajo el mínimo y Reposición · Unidades sugeridas (dos números) |
| Tabla de pedido (`sugerido > 0`, resalta `bajo_minimo`) | Reposición · Tabla de pedido |
| Barras por sucursal y categoría | Reposición · Sugerido por sucursal y categoría (apiladas) |

**Tablero 5 · Salud de la sincronización** (`v_salud_sincronizacion`, `v_traslados_pendientes`)

| En este documento | Pregunta de Metabase |
|---|---|
| Tabla por sucursal (ámbar > 30 min, rojo > 120) | Salud · Por sucursal |
| Indicador de antigüedad del dato | Salud · Último contacto de la réplica **y** Salud · Frescura de la carga |
| Tabla de traslados | Salud · Traslados no finalizados |
| Barras por `situacion` | Salud · Traslados por situación |
| Textos de límites | Tarjeta de texto con los límites del tablero y la diferencia con BigQuery |

### Diferencia con BigQuery en el tablero 5

`cdc_ultimo_cambio` no es el `source_timestamp` de Datastream (en la variante no existe): es el último contacto de la
suscripción con el central. Se acompaña de `ultima_carga_en` y `lag_bytes` en la tarjeta «Frescura de la carga».
Ver `infra/on-premise/analitica/06_vistas.sql`.

### Límites

- Las pruebas de la variante comprueban que cada pregunta devuelve filas, que los controles llegan a la consulta y que
  los números coinciden con las vistas. **No comprueban cómo se ve** cada gráfico (tipo, ejes, colores, formato
  condicional): eso se revisa abriendo Metabase.
- Si Metabase falla después de reiniciar `db-analitica`, sus conexiones a los metadatos quedan muertas y el inicio de
  sesión también falla. Tras un `stop` + `start` de la base, `docker compose restart metabase`. Insistir con el inicio
  de sesión bloquea la cuenta durante horas: no conviene reintentar en bucle. Detalle en la entrada de
  `docs/decisiones.md` del 2026-10-29 sobre Metabase.
