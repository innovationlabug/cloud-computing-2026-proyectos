# Decisiones de diseño

Registro de las decisiones no obvias del proyecto. Cada entrada lleva la fecha, lo que
se decidió, la alternativa que se descartó y por qué. Es material de entrega.

---

## 2026-09-11 — Copias consolidadas en el central llevan `id_*_origen`

**Decisión.** En la base del central, `documento` y `movimiento` conservan su propio
`SERIAL` como llave primaria y agregan `id_documento_origen` / `id_movimiento_origen`
con `UNIQUE (id_sucursal, id_*_origen)`. La columna es `NULL` cuando el documento nace
en el central (traslados).

**Alternativa descartada.** Copiar las tablas tal cual el DDL base, con el `SERIAL` de
la sucursal como llave.

**Por qué.** Cada sucursal genera sus propios `SERIAL`, así que el documento 7 de la
sucursal 1 y el 7 de la sucursal 2 chocarían al consolidarse. La restricción única
compuesta, además, hace idempotente la aplicación de un mensaje repetido: reinsertar
una copia que ya llegó falla por la restricción y no duplica nada.

## 2026-09-11 — `existencia.version` es `NOT NULL`

**Decisión.** En la sucursal, `existencia.version INT NOT NULL DEFAULT 0` (el DDL base
la dejaba nullable).

**Por qué.** La estrategia optimista hace `UPDATE ... WHERE version = $v`. Con `NULL`
la comparación nunca coincide y toda venta se rechazaría como conflicto.

## 2026-09-11 — El central lleva `existencia_consolidada`, no `existencia`

**Decisión.** La tabla del central se llama `existencia_consolidada` y agrega
`sincronizado_en` (cuándo llegó el dato al central) además de `actualizado_en`
(cuándo cambió en la sucursal).

**Por qué.** Son datos de consistencia eventual y la regla de oro exige mostrar la
antigüedad del dato en pantalla; se necesitan las dos marcas para calcularla. El
nombre distinto evita confundirla con el fragmento operativo de la sucursal, que es
la única fuente de verdad sobre la mercadería física.

## 2026-09-11 — Migraciones con tabla `migracion_aplicada`, aplicadas por `psql` dentro del contenedor

**Decisión.** `scripts/migrar.sh` aplica los archivos SQL numerados en orden y
registra cada uno en `migracion_aplicada` dentro de la misma transacción del archivo.
Usa `docker compose exec ... psql`.

**Alternativa descartada.** Una herramienta de migraciones de Node (node-pg-migrate,
Prisma Migrate, etc.).

**Por qué.** Las convenciones del proyecto piden SQL escrito a mano y versionado, sin
migraciones generadas. Un script de 60 líneas cumple y no agrega dependencias. Se usa `psql` del
contenedor porque la máquina de desarrollo no lo tiene instalado y así el flujo es el
mismo para cualquier integrante.

## 2026-09-11 — `max_prepared_transactions` habilitado desde el Compose inicial

**Decisión.** Los cuatro PostgreSQL arrancan con `max_prepared_transactions=20`.

**Por qué.** Por defecto es 0 y `PREPARE TRANSACTION` falla. El hito de 2PC lo va a
necesitar y cambiarlo después obliga a recrear contenedores en todos los nodos. No
implementa nada del protocolo; solo evita que la configuración de hoy lo estorbe.

## 2026-09-18 — La prueba de concurrencia es Node puro, no k6

**Decisión.** `scripts/prueba-concurrencia/prueba.mjs` lanza las N peticiones con `fetch`
y `Promise.all` desde un mismo tick del bucle de eventos, tras calentar N conexiones
keep-alive para que los `POST` no se desfasen por el handshake TCP. Cruza el resultado
contra la base con `pg` (ya presente en el monorepo).

**Alternativa descartada.** k6 (que sigue siendo la herramienta prevista para las pruebas
de carga del informe de CC6) y pgbench.

**Por qué.** k6 no estaba instalado y no queríamos sumar un binario la noche antes de la
demo. Lo que importa en esta prueba es que las peticiones salgan de verdad al mismo tiempo
y que el resultado se verifique en la base, y eso Node lo hace en 300 líneas sin
dependencias nuevas. La medición de carga sostenida (operaciones por segundo bajo carga
prolongada) queda para k6/pgbench en el hito de mediciones.

## 2026-09-18 — Existe una tercera estrategia, `SIN_CONTROL`, solo para la prueba

**Decisión.** `ESTRATEGIA_CONCURRENCIA` acepta `SIN_CONTROL`: lectura sin bloqueo y
`UPDATE` sin condición de versión. Está en el código de dominio porque la prueba corre
contra la API real, pero no es una opción de operación y así se documenta en el código,
en el informe y aquí.

**Alternativa descartada.** Simular la falla desde fuera (por ejemplo, un script SQL que
hiciera el `UPDATE` ingenuo directamente contra la base).

**Por qué.** El informe tiene que mostrar que la garantía última contra la sobreventa la
da el motor (`CHECK (cantidad >= 0)`) aunque la aplicación falle. Con las dos estrategias
correctas el `CHECK` no salta nunca, así que "0 rechazos del motor" no demuestra nada.
Con `SIN_CONTROL` se ve al `CHECK` rechazar 13 y 19 escrituras negativas en la misma
corrida y con el mismo camino de código que la venta real.

## 2026-09-18 — Hallazgo: el `CHECK` evita la sobreventa, pero no evita una bitácora falsa

**Qué se encontró.** En la corrida de `SIN_CONTROL` con existencia 20, el motor detuvo las
19 escrituras negativas y la existencia final fue correcta, pero **19 de las 20 ventas
exitosas registraron un `cantidad_antes` falso** en `movimiento`: leyeron 20 (dato viejo),
el `UPDATE` sin condición restó sobre el valor ya actualizado por otra caja y quedó, por
ejemplo, `antes=20, cantidad=1, despues=14`. Ninguna invariante de existencias lo detecta;
solo se ve revisando que `cantidad_antes − cantidad = cantidad_despues` en cada movimiento.

**Consecuencia de diseño.** El `CHECK` se queda (es la última línea de defensa), pero no
sustituye al control de concurrencia: sin `FOR UPDATE` o versión, el sistema puede no
sobrevender y aun así producir un histórico inmutable que miente. Como `movimiento` no
se corrige después, ese daño es permanente. La prueba de concurrencia verifica esa
igualdad en cada corrida desde ahora.

**Efecto en la base de desarrollo.** Las 19 filas falsas quedaron en `db-sucursal-1` (la
bitácora no se edita). Para partir limpio: `docker compose down -v`, migrar y sembrar.

## 2026-09-26 — Corrección: `SIN_CONTROL` pasa a escritura absoluta; la "bitácora falsa" del 18-sep venía de una resta relativa

**Qué estaba mal.** La primera versión de `SIN_CONTROL` hacía `SET cantidad = cantidad − n`.
Esa resta la evalúa el motor sobre el valor ya confirmado, así que no es un *lost update*:
bajo `READ COMMITTED` el segundo `UPDATE` espera al primero y resta sobre el resultado. Por
eso el `CHECK (cantidad >= 0)` alcanzaba a atrapar las escrituras negativas (13 y 19 en la
corrida del 18-sep), y la entrada del 18-sep sobre "`SIN_CONTROL` solo para la prueba"
sobrestimaba lo que el `CHECK` protege.

**De dónde salió la "bitácora falsa" del 18-sep.** De combinar una lectura vieja con una
resta relativa. La aplicación leía 20 (dato viejo) y lo guardaba como `cantidad_antes`,
pero el `UPDATE` restaba 1 sobre el valor que ya habían dejado las otras cajas y devolvía,
por ejemplo, 14 como `cantidad_despues`. La fila quedaba `antes=20, cantidad=1, despues=14`:
incoherente en sí misma. No era sobreventa; era un histórico que no cuadraba fila por fila.

**Decisión.** `SIN_CONTROL` escribe el valor absoluto calculado en la aplicación:
`SET cantidad = <leída> − n`. Es el *lost update* de libro: dos cajas leen 1 y las dos
escriben 0. Ninguna escritura es negativa, el `CHECK` no salta y hay sobreventa
silenciosa (12 y 30 unidades en la corrida del 26-sep). `version` sigue siendo relativa
(`version + 1`) para que cuente cada escritura y se vea el contraste con la cantidad.

**Cambios en la verificación.** Mirar que la existencia final sea ≥ 0 ya no detecta nada:
queda en 0. La prueba ahora:
- mide la sobreventa en unidades (`exitosas × cantidad − inicial`);
- verifica la bitácora también en conjunto: `SUM(cantidad)` de los movimientos nuevos debe
  ser igual a `inicial − final`. Con el *lost update* cada fila cuadra por sí sola
  (`antes=1, cantidad=1, despues=0`), pero la suma de lo vendido supera lo que bajó la
  existencia. Esa verificación sustituye a la columna "movimientos con `cantidad_antes`
  falso" del informe; la de fila por fila se mantiene como invariante.

**Conclusión que cambia.** El `CHECK` ataja la resta relativa, pero no la escritura
absoluta. Se queda —protege toda escritura relativa, incluido el `UPDATE` condicional del
traslado—, pero la garantía contra la sobreventa es el control de concurrencia. La
bitácora inmutable detecta la sobreventa después del hecho aunque el `CHECK` no salte.

**Alternativa descartada.** Mantener la resta relativa y documentar el `CHECK` como
"última línea de defensa". Descartada porque la afirmación es falsa para la falla más
común en código de aplicación —leer, calcular y escribir el valor—, y el informe la
habría presentado como garantía.

## 2026-09-26 — `SIN_CONTROL` exige `PERMITIR_SIN_CONTROL=1`

**Decisión.** La fábrica de estrategias rechaza `SIN_CONTROL` salvo que
`PERMITIR_SIN_CONTROL=1` esté definida. Sin el permiso, la API de sucursal no arranca y el
log dice por qué. El Compose la deja en `0` por defecto, y el único lugar del repositorio
que la pone en `1` es `scripts/prueba-concurrencia/ejecutar.sh`, solo para la corrida de
esa estrategia. Al terminar, el script hace `docker compose up -d` sin la variable y la
sucursal vuelve a `PESIMISTA` con el permiso en `0`.

**Alternativa descartada.** Dejarla seleccionable como las otras dos y confiar en la
documentación.

**Por qué.** Una estrategia que sobrevende no puede estar a una variable de entorno de
distancia de producción. Un `ESTRATEGIA_CONCURRENCIA=SIN_CONTROL` puesto por error (un
`.env` copiado, un valor de prueba que se quedó en un despliegue) dejaría una sucursal
facturando mercadería que no existe sin que ningún error lo avise: la corrida del
26-sep demostró que el `CHECK` no la detiene. Exigir dos variables, una de ellas con un
nombre que dice exactamente lo que habilita, convierte el error de configuración en un
fallo de arranque visible. No elimina el riesgo —alguien puede poner las dos—, pero lo
vuelve deliberado.

## 2026-09-26 — Payload común para todo documento local, con la cantidad de cada movimiento

**Decisión.** VENTA y AJUSTE encolan en `outbox` el mismo payload: encabezado del documento
(ahora con `id_proveedor` y `estado`), sus líneas y sus movimientos, cada uno con `cantidad`
explícita además de `cantidad_antes`, `cantidad_despues` y `version_despues`. Las
inserciones de documento, línea y movimiento y el encolado viven en
`nucleo/src/sincronizacion/documento-local.ts`, y las usan la venta y el ajuste.

**Alternativa descartada.** Un payload distinto por tipo de operación, cada uno construido
en su propio módulo.

**Por qué.** El consumidor del central aplica "un documento local con sus movimientos" sin
importar el tipo: un solo camino de código y un solo formato que mantener. La `cantidad`
del movimiento se podía derivar de antes y después, pero el central guarda `movimiento`
completo y no debe reconstruir la bitácora a partir de otras columnas. La fecha del
movimiento no viaja aparte porque es la del documento: se insertan en la misma
transacción y `CURRENT_TIMESTAMP` es el inicio de esa transacción.

## 2026-09-26 — Ajuste por conteo físico; `reiniciar-existencia.sh` deja de escribir por fuera

**Decisión.** `POST /ajustes` recibe cantidades contadas y, en una transacción, deja la
existencia en ese valor: documento `AJUSTE`, una línea y un movimiento por producto que
cambió, y la fila de `outbox`. Detalles:
- **Bloqueo pesimista siempre** (`SELECT … FOR UPDATE`), sin importar
  `ESTRATEGIA_CONCURRENCIA`. El ajuste no es la operación que se mide, y es justamente el
  patrón bloquear, leer, decidir, escribir.
- **Convención de signo.** `movimiento.cantidad` es siempre positiva y el sentido va en el
  tipo: `AJUSTE_ENTRADA` o `AJUSTE_SALIDA`, como `VENTA`. Obliga a eso
  `linea_documento.cantidad > 0`, y mantiene una regla verificable fila por fila:
  salida `antes − cantidad = despues`, entrada `antes + cantidad = despues`.
- **La línea lleva `precio_unitario = 0`** y el documento `total` y `metodo_pago` en
  `NULL`: no es una operación comercial.
- **Un conteo que coincide** no genera movimiento. Si ninguno difiere, no se crea
  documento ni outbox y la API responde `200` con `sin_cambios`.
- **Solo ENCARGADO o ADMIN** de la sucursal (`403 PERMISO_INSUFICIENTE` para un cajero).

`reiniciar-existencia.sh` ya no hace `UPDATE existencia` directo: llama a `POST /ajustes`
firmado por el encargado de la sucursal.

**Alternativa descartada.** Hacer el ajuste en SQL dentro de `psql`, con la fila de outbox
armada con `jsonb_build_object`.

**Por qué.** Un `UPDATE` directo dejaba cambios de existencia sin movimiento que los
explicara y sin aviso al central: la existencia consolidada se habría desviado para
siempre de la de la sucursal. Armar el payload en SQL duplicaría el formato en dos
lugares que terminarían divergiendo.

**Efecto en la prueba de concurrencia.** La foto inicial se toma después del ajuste de
reinicio, y las verificaciones de bitácora (fila por fila y en conjunto) y el conteo de
movimientos nuevos consideran solo movimientos `VENTA`. El ajuste queda fuera de todo lo
que se compara.

## 2026-09-26 — El ajuste por conteo lleva `version_esperada`: tenía el mismo *lost update* que `SIN_CONTROL`

**Qué estaba mal.** El ajuste por conteo escribe un valor absoluto (la cantidad contada) y
lo calcula con una lectura hecha **fuera** de la transacción: el conteo físico. Es la misma
forma que `SIN_CONTROL` —leer, calcular afuera, escribir el absoluto— y tiene el mismo
defecto. El encargado cuenta 10, una caja vende 1 mientras tanto, y el ajuste escribe 10:
la venta desaparece de la existencia aunque siga en la bitácora. El `SELECT … FOR UPDATE`
no lo evitaba porque solo protege lo que se lee **dentro** del bloqueo, y la lectura que
importa es anterior.

**Relación con `SIN_CONTROL`.** La corrida del 26-sep mostró que una escritura absoluta
basada en una lectura vieja no la detiene el `CHECK`: el valor escrito nunca es negativo.
Aquí tampoco: escribir 10 sobre 9 es perfectamente válido para el motor. La única defensa
es comprobar que el dato sobre el que se decidió sigue vigente, que es control optimista.
El conteo es, por naturaleza, una transacción larga que no puede tener la fila bloqueada
(el encargado tarda minutos y las cajas no pueden esperar), así que la versión es la
herramienta que corresponde.

**Decisión.** Cada línea de `POST /ajustes` lleva `version_esperada`: la versión de
`existencia` que se vio al empezar el conteo. Dentro del `FOR UPDATE` se comparan **todas**
las líneas antes de escribir nada. Si alguna difiere, el ajuste completo se rechaza con
`409 CONFLICTO_VERSION`, que lista cada producto con la versión esperada y la actual, y no
se escribe ninguna línea, ni siquiera las que no tenían conflicto. No hay reintento
automático: la lectura desactualizada es el conteo, y hay que volver a contar. Por eso el
error es una clase aparte de la del conflicto de la venta optimista, aunque comparta el
código: la venta reintenta ante `ErrorConflictoVersion`, y el ajuste no debe hacerlo.

`reiniciar-existencia.sh` lee la versión con `GET /existencias/:codigo_barras` y ajusta
sobre ella.

**Demostración.** `scripts/prueba-ajuste/prueba.mjs`, casos 9 y 10: una venta entre la
lectura y el ajuste da 409; la venta se conserva y el ajuste no escribe documento,
movimiento ni outbox. Con la verificación desactivada a propósito, el caso 9 reproduce el
*lost update*: la existencia vuelve al valor contado y la venta se pierde.

**Alternativa descartada.** Calcular la diferencia contra la existencia que se vio al
empezar el conteo y aplicarla en forma relativa sobre la actual (`cantidad + diferencia`).
Eso no borra la venta, pero supone que el conteo refleja el instante en que empezó, y no
es así: un conteo dura minutos, y por cada venta hecha en ese lapso no se sabe si la
unidad salió del estante antes o después de que el encargado pasara por él. La
corrección quedaría errada, en un sentido que nadie puede determinar, por hasta lo que se
vendió durante el conteo. Rechazar y volver a contar es la única salida que no inventa
un dato.

## 2026-09-26 — Consumidor del central: inbox, una transacción y una segunda barrera

**Decisión.** `POST /sincronizacion/mensajes` aplica cada mensaje en una sola transacción:
1. `INSERT INTO inbox_idempotencia … ON CONFLICT DO NOTHING RETURNING`. Si no insertó, el
   mensaje ya se procesó y se responde `200` con `duplicado: true`, sin aplicar nada.
2. Documento, líneas y movimientos, con `id_documento_origen` e `id_movimiento_origen`.
3. `existencia_consolidada`, con la guarda de versión (entrada siguiente).

Si cualquier paso falla, se deshace todo, **incluida la fila de inbox**: un mensaje
rechazado no queda marcado como procesado y una versión corregida no se descartaría por
duplicada. `UNIQUE (id_sucursal, id_documento_origen)` es una segunda barrera: un mensaje
con una llave de idempotencia nueva pero un documento ya consolidado choca con ella y se
rechaza.

Dos entregas simultáneas del mismo mensaje no se cuelan: la segunda espera en el índice
único de `inbox_idempotencia` la fila todavía no confirmada de la primera y, cuando esta
confirma, no inserta. Probado con 5 entregas en paralelo: 1 aplicada, 4 duplicadas.

**Alternativa descartada.** Consultar la inbox con un `SELECT` antes de aplicar. Entre el
`SELECT` y el `INSERT` hay una ventana en la que dos entregas del mismo mensaje pasan las
dos. El `INSERT … ON CONFLICT` resuelve la pregunta y la reserva en un solo paso atómico.

## 2026-09-26 — El consumidor responde 4xx solo si el mensaje es inválido; todo lo demás es 5xx

**Decisión.**
- `400 MENSAJE_INVALIDO`: la forma o la coherencia interna del mensaje. Se valida antes
  de tocar la base: tipos, que el payload diga la misma sucursal que el sobre, y que cada
  movimiento cuadre (`antes ∓ cantidad = despues` según el tipo).
- `409 CONFLICTO_DATOS`: el mensaje choca con datos ya consolidados (violación de unicidad).
- `422 MENSAJE_INCONSISTENTE`: el mensaje referencia algo que no existe (llave foránea) o
  viola otra restricción.
- **`503 ERROR_TRANSITORIO` para todo lo demás**: la base del central caída, un
  interbloqueo, un error no previsto.

**Por qué.** El publicador decide con esto si una fila puede llegar a `FALLIDO`. Si una
caída de `db-central` respondiera 4xx, la cabeza de la cola de cada sucursal se iría a
`FALLIDO` igual que con un corte de enlace largo. Probado: con `db-central` detenida, el
consumidor responde 503 y la API central sigue viva; al volver la base, el mismo mensaje
se aplica.

**Consecuencia aceptada.** Un error de programación en el consumidor (un 5xx permanente)
bloquea la cola de esa sucursal hasta que se corrija, en vez de mandar mensajes a
`FALLIDO`. Preferimos una cola detenida y visible en `/salud` a datos descartados.

## 2026-09-26 — `existencia_consolidada` se actualiza con upsert condicionado a la versión

**Decisión.** Por cada movimiento: `INSERT … ON CONFLICT (id_sucursal, id_producto) DO UPDATE
SET … WHERE existencia_consolidada.version < EXCLUDED.version`. `cantidad` y `version` se
copian tal cual de la sucursal (`cantidad_despues`, `version_despues`), `actualizado_en`
toma la fecha del documento y `sincronizado_en` la hora del central.

**Alternativa descartada.** Aplicar el delta del movimiento (`cantidad = cantidad − n`).

**Por qué.** La existencia la gana siempre la sucursal, que es la única que ve la
mercadería: el central copia su valor, no lo recalcula. Con valores absolutos y versión,
un mensaje atrasado o desordenado no pisa un dato más nuevo: el `WHERE` lo ignora. Con
deltas, un mensaje perdido o un mensaje en `FALLIDO` desviaría el consolidado para
siempre, y el orden de aplicación importaría. Probado entregando al revés dos ventas del
mismo producto: la más nueva se aplica, la vieja registra su documento pero no toca la
existencia, y el consolidado queda igual a la sucursal. Esta es la escritura absoluta
correcta, a diferencia de `SIN_CONTROL` y del ajuste: aquí el valor escrito es el dato
autoritativo de la sucursal y la versión garantiza que es el más nuevo.

## 2026-09-26 — `TIMESTAMP` sin zona = UTC, y la fecha viaja con microsegundos

**Decisión.** Convención del proyecto: toda columna `TIMESTAMP` guarda UTC. Las bases corren
en UTC (imagen de postgres y Cloud SQL por defecto). El núcleo fija el parser de `pg` para
leer esas columnas como UTC, sin depender de la zona del proceso Node. La fecha del
documento viaja en el payload como ISO 8601 UTC con **microsegundos**, generada por
PostgreSQL (`to_char(… 'US')`). El consumidor la guarda con
`$n::timestamptz AT TIME ZONE 'UTC'`.

**Por qué.** Un `Date` de JavaScript solo tiene milisegundos: la copia del central habría
quedado distinta a la de la sucursal, y la verificación (b) lo detecta (se probó alterando
un microsegundo). Leer los `TIMESTAMP` en la zona del proceso daría resultados distintos
según dónde corra la API; en una máquina de desarrollo en Guatemala, seis horas de error.

**Alternativa descartada.** Migrar las columnas a `TIMESTAMPTZ`. Resuelve la zona pero no
la precisión, y cambia el DDL base en todos los nodos por algo que se resuelve con una
convención y dos conversiones explícitas.

## 2026-09-26 — `vigente_al` es una aproximación; para un mismo producto, el orden de `id_outbox` es el de commit

**Decisión.** La consulta de existencias en la red devuelve por sucursal `cantidad`,
`version`, `actualizado_en` y `sincronizado_en` tal como están, más:
- `vigente_al`: la `fecha_hora` del último documento de esa sucursal ya aplicado en el
  central, o `sincronizado_en` de la carga inicial si todavía no llegó ninguno;
- `antiguedad_segundos`: ahora menos `vigente_al`. Es lo que se muestra en pantalla.

`sincronizado_en` sola no sirve como antigüedad: solo se mueve cuando cambia ese
producto, y uno que no se vende en tres días aparecería con tres días de antigüedad
aunque el enlace esté bien.

**Lo que `vigente_al` NO garantiza.** No afirma que el dato sea correcto hasta esa fecha.
`id_outbox` se asigna al insertar en `outbox` y `fecha_hora` al **inicio** de la
transacción, no al commit. Una venta que empezó antes puede confirmar después: tiene una
`fecha_hora` anterior a `vigente_al` y, sin embargo, todavía no llegó al central. Es una
aproximación cuyo error está acotado por la duración de una transacción en vuelo: una
venta local dura milisegundos. Una sucursal conectada pero sin movimiento también se ve
atrasada; resolverlo pide un latido, que no está en el alcance.

**Lo que sí se garantiza: para un mismo producto, el orden de `id_outbox` coincide con el
de commit.** En la venta y en el ajuste, el bloqueo de la fila de `existencia` se toma
**antes** de insertar en `outbox` (pesimista en el `SELECT … FOR UPDATE`; optimista y
`SIN_CONTROL` en el `UPDATE`; ajuste en el `FOR UPDATE`), y se retiene hasta el commit. La
segunda transacción sobre ese producto no llega a su `INSERT INTO outbox` hasta que la
primera confirmó, así que recibe un `id_outbox` mayor. Esa es la razón por la que la fila
de outbox se escribe al final en `documento-local.ts`. Entre productos distintos no hay
tal garantía: dos ventas de productos diferentes pueden confirmar en orden inverso a su
`id_outbox`. No importa para la existencia, porque son filas distintas y cada una tiene
su propia guarda de versión.

## 2026-09-26 — El pool de conexiones maneja la pérdida de conexiones inactivas

**Decisión.** `crearPool` registra un manejador del evento `error` del pool de `pg`.

**Por qué.** Cuando la base se reinicia, las conexiones inactivas del pool mueren y `pg`
emite `error` en el pool. Sin manejador, ese evento tumba el proceso completo: una
sucursal perdería su API por un reinicio de su propia base, y el central por uno de la
suya. Con el manejador, el pool descarta la conexión rota y abre otra en el siguiente
uso. Probado deteniendo y levantando `db-central`: la API central no se reinició
(`RestartCount` sin cambios) y siguió respondiendo.

## 2026-09-26 — Transporte de sincronización como adaptador; en Compose, HTTP directo

**Decisión.** El núcleo define el contrato `TransporteSincronizacion`: `enviar(mensaje)`
devuelve un resultado ya clasificado. El publicador no sabe qué hay detrás. En Docker
Compose el adaptador es `TransporteHttp`: un `POST` por mensaje a
`CENTRAL_URL/sincronizacion/mensajes`, con timeout (`SINCRONIZACION_TIMEOUT_MS`, 3 s).
Clasificación:

| Respuesta | Resultado | Qué pasa con la fila |
|---|---|---|
| 2xx | `OK` | `ENVIADO` |
| 4xx salvo 408 y 429 | `RECHAZADO` (el mensaje en sí es inválido) | suma un intento; al máximo, `FALLIDO` |
| 5xx, 408, 429 | `TRANSITORIO` (el central respondió, pero no pudo ahora) | sigue `PENDIENTE`, sin sumar intentos |
| error de red, DNS, timeout | `SIN_ENLACE` | sigue `PENDIENTE`, sin sumar intentos |

**Alternativa descartada.** Que el publicador llame a `fetch` directamente.

**Por qué.** En GCP el transporte será Pub/Sub con suscripción push hacia Cloud Run, que
llega al mismo endpoint consumidor. Con el contrato, ese cambio es un adaptador nuevo
(`TRANSPORTE_SINCRONIZACION=PUBSUB`), no una reescritura del publicador. Un mensaje por
petición es la misma unidad que un mensaje de Pub/Sub. 408 y 429 son 4xx, pero significan
"ahora no", no "este mensaje está mal": tratarlos como rechazo mandaría a `FALLIDO`
mensajes válidos durante un pico de carga.

## 2026-09-26 — Publicador: un trabajador, `FOR UPDATE SKIP LOCKED` y transacción abierta durante el envío

**Decisión.** Un solo trabajador por sucursal, dentro de la API de sucursal, cada
`SINCRONIZACION_INTERVALO_MS` (2 s). Cada ciclo es una transacción sobre la base local:
1. `SELECT … WHERE estado = 'PENDIENTE' ORDER BY id_outbox LIMIT n FOR UPDATE SKIP LOCKED`,
   con `n = SINCRONIZACION_LOTE` (20).
2. Envío de las filas de a una, en orden.
3. `ENVIADO` y `enviado_en` solo con acuse de recibo.
4. En la primera falla transitoria se corta el lote, para no mandar las siguientes antes
   que esa.

Si el lote salió completo, el siguiente ciclo arranca de inmediato, así la cola se vacía
rápido después de un corte. El publicador usa un pool propio de **una** conexión: nunca
le quita conexiones a las cajas.

**Lo que bloquea.** Solo filas de `outbox`, nunca `existencia`. Las ventas insertan filas
nuevas de `outbox`, que no chocan con las que el publicador tiene bloqueadas. La venta y
el publicador no compiten por ningún bloqueo.

**Cota del peor caso de la transacción abierta.** El lote se corta en la primera falla,
así que el caso más largo es un lote completo en el que cada envío tarda casi el timeout
y aun así tiene éxito: `SINCRONIZACION_LOTE × SINCRONIZACION_TIMEOUT_MS` = 20 × 3 s =
**60 s**, más las escrituras locales. Durante ese tiempo quedan retenidas a lo sumo 20
filas de `outbox` y el horizonte de `VACUUM` no avanza. En la red local cada envío tarda
milisegundos. Si hiciera falta bajar la cota, se reduce el lote o el timeout.

**Alternativas descartadas.**
- **Marcar las filas como "en proceso" y enviar fuera de la transacción.** Requiere un
  estado que el `CHECK` de `outbox` no tiene, y un proceso que recupere las filas que
  quedaron "en proceso" si el publicador se cae. Con la transacción abierta, si el
  proceso se cae antes del `COMMIT`, las filas simplemente vuelven a salir y el central
  descarta las que ya había aplicado.
- **Varios trabajadores.** `SKIP LOCKED` evita que dos trabajadores tomen la misma fila,
  pero no que envíen filas distintas en desorden. El orden por sucursal depende de que
  haya uno solo.

**Demostración.** La prueba del consumidor había entregado a mano
las 18 filas de outbox de las 3 sucursales sin marcarlas. Al levantar el publicador las
reenvió todas: el central respondió **18 duplicados y 0 aplicados**. Documentos,
movimientos, inbox y la huella de `existencia_consolidada` quedaron idénticos, y las 18
filas pasaron a `ENVIADO`.

## 2026-09-26 — `intentos` cuenta solo rechazos; después de un `FALLIDO`, la cola sigue

**Decisión.** `SIN_ENLACE` y `TRANSITORIO` no suman intentos: la fila espera
indefinidamente. Solo un `RECHAZADO` (4xx permanente) suma, y al llegar a
`SINCRONIZACION_MAX_RECHAZOS` (5) la fila pasa a `FALLIDO`. Por debajo del máximo, el
rechazo también corta el lote. Al pasar a `FALLIDO`, el publicador **sigue con las
siguientes** en el mismo ciclo.

**Por qué.** Si todo fallo sumara intentos, un corte de seis horas o una caída de
`db-central` mandarían la cabeza de la cola a `FALLIDO` en segundos, que es exactamente
lo que el outbox tiene que evitar. Un 4xx permanente, en cambio, no se arregla
reintentando.

**Consecuencia aceptada: se rompe el orden.** Las filas posteriores a un `FALLIDO` llegan
al central antes que él. La existencia consolidada no se corrompe, porque la guarda de
versión ignora cualquier dato más viejo que el guardado. Lo que falta es el documento del
`FALLIDO`, y eso queda visible: `/salud` cuenta los `FALLIDO` y `verificar.mjs` detecta que
esa fila de outbox no está en la inbox del central. Devolver la fila a `PENDIENTE` una vez
corregida la causa es seguro: se aplica fuera de orden y la guarda de versión protege el
consolidado (procedimiento en el README).

**Demostración.** `scripts/prueba-publicador/prueba.mjs`, caso 4: una fila envenenada en
la cabeza retuvo 5 ciclos a la venta que tenía detrás (enlace `ATASCADA`, `400
MENSAJE_INVALIDO`), pasó a `FALLIDO` con 5 intentos y la venta llegó al central en el
mismo ciclo.

## 2026-09-26 — Bloqueo de cabeza de cola: se acepta, pero tiene que verse

**Qué es.** Como el publicador corta el lote en la primera falla, la fila más vieja
retiene a todas las que tiene detrás. Hay dos situaciones distintas:
- **Sin enlace** (`enlace: SIN_ENLACE`): no se llega al central (red, DNS, timeout). Es el
  caso normal de una sucursal desconectada: la caja vende y la cola crece.
- **Enlace arriba pero cola atascada** (`enlace: ATASCADA`): el central responde, pero la
  cabeza no pasa. Puede ser un 503 porque su base está caída, o un rechazo que todavía no
  llegó al máximo. Si el 503 lo provoca un mensaje en particular (un error de programación
  en el consumidor), la cola queda detenida hasta que se corrija: **nunca** pasa a
  `FALLIDO`, porque un 5xx es transitorio por definición.

**Decisión.** Se acepta: no se pierde nada y se respeta el orden. Pero `/salud` de la
sucursal lo hace visible:
- `outbox.pendiente_mas_antigua`: `id_outbox`, `creado_en`, `antiguedad_segundos` e
  `intentos` de la cabeza. Si la antigüedad crece, la cola está detenida.
- `sincronizacion.enlace`: `OK`, `SIN_ENLACE`, `ATASCADA` o `DESCONOCIDO` (todavía no hubo
  ningún envío desde que arrancó el proceso). Refleja el **último envío intentado**: con
  la cola vacía no se intenta nada y el estado no cambia.
- `sincronizacion.ultimo_error`: tipo, código HTTP, código del central, mensaje, fila y
  hora del último envío fallido.
- `outbox.fallidos`: filas que agotaron sus rechazos.

El registro de la API avisa solo cuando cambia el estado del enlace, no en cada ciclo:
"sin enlace falla en silencio" para la caja, pero no para quien opera.

**Demostración.** `scripts/prueba-publicador/prueba.mjs`:
- **Caso 2**, central apagado: `SIN_ENLACE`, `ENOTFOUND`, cabeza envejeciendo con 0
  intentos, 0 `FALLIDO`.
- **Caso 3**, `db-central` caída: `ATASCADA`, `TRANSITORIO 503 ERROR_TRANSITORIO`, 0
  intentos, 0 `FALLIDO`.

En los dos casos, al volver, la cola se vació en menos de 2 s y el central cuadró.

## 2026-09-26 — El publicador se puede apagar con `SINCRONIZACION_ACTIVA=0`

**Decisión.** Con `SINCRONIZACION_ACTIVA=0` el publicador no arranca: la caja vende igual y
el outbox solo crece. `/salud` informa `sincronizacion.activo`. La prueba de concurrencia
registra si el publicador estaba activo y cuántas filas envió durante cada corrida, y el
informe lo dice.

**Por qué.** El informe de concurrencia tiene que poder decir si había otra carga sobre la
base mientras se medía. En las corridas del 26-sep el publicador estuvo activo, pero envió
0 o 1 filas por corrida: una corrida dura entre 70 y 220 ms, menos que su intervalo de
2 s.

## 2026-09-26 — Redes separadas: cada sucursal, el enlace y los datos del central

**Decisión.** Tres clases de red en Docker Compose, con nombre fijo:
- **`sima-red-sucursal-N`:** `api-sucursal-N` y `db-sucursal-N`. Es la sucursal completa: si
  pierde todo lo demás, sigue vendiendo.
- **`sima-red-central`:** las cuatro APIs. Es **el enlace**.
- **`sima-red-datos-central`:** solo `api-central` y `db-central`.

La ruta por defecto de cada API de sucursal sale por su red local (`gw_priority`), y la
sucursal no tiene `depends_on` hacia el central: arranca sin él.

**Alternativa descartada.** Poner `db-central` en la red del enlace, junto a las APIs de
sucursal.

**Por qué.** La sección 7.2 dice que el nodo de sucursal "no tiene permiso de leer la base
del central". Con la red aparte, eso vale también a nivel de red, no solo de credenciales.
Probado desde adentro de los contenedores:

| Desde | Llega a | Resultado |
|---|---|---|
| `api-sucursal-1` | su base | sí |
| `api-sucursal-1` | `api-central` | sí |
| `api-sucursal-1` | `db-central` | no (`ENOTFOUND`) |
| `api-sucursal-1` | la base de otra sucursal | no (`ENOTFOUND`) |
| `api-central` | una base de sucursal | no (`ENOTFOUND`) |

Los nombres fijos (`name:`) hacen que el comando de corte no dependa del nombre del
directorio del proyecto.

## 2026-09-26 — El corte de enlace pasa solo por un adaptador de dos funciones

**Decisión.** `scripts/corte-enlace/enlace-compose.sh` define `cortar_enlace N`,
`restablecer_enlace N`, `enlace_cortado N` y `nombre_adaptador`. En Compose:
- cortar es `docker network disconnect sima-red-central sima-api-sucursal-N`;
- restablecer es `docker network connect --alias api-sucursal-N …`. El alias devuelve el
  nombre de servicio que el hito de 2PC necesita para que el central encuentre a la
  sucursal.

`ejecutar.sh` carga el adaptador indicado en `ENLACE_ADAPTADOR` y deja un `trap` que
reconecta la sucursal si el script termina por cualquier causa. `corte.mjs` llama a las
funciones del adaptador y no sabe cómo se implementan.

**Por qué.** En GCP (sección 7.3) cortar el enlace será aplicar una regla de firewall de
salida por etiqueta con `gcloud`. Ese cambio es un archivo nuevo con las mismas cuatro
funciones; la línea de tiempo, las ventas y las verificaciones no cambian.

**Qué verifica.**
- **(a)** La existencia consolidada es igual a la de cada sucursal.
- **(b)** Documentos, líneas, movimientos e inbox son iguales a los de cada sucursal.
- **(c) Entrega repetida:** 10 mensajes ya enviados se devuelven a `PENDIENTE` y el
  publicador los reenvía; además, un mensaje se entrega 5 veces en paralelo. La foto del
  central debe quedar idéntica.
- **(d) Desorden:** con el enlace caído, los mensajes de dos ventas del mismo producto se
  entregan al central al revés, simulando un transporte que no respeta el orden; queda la
  versión más nueva.

Probado también el `trap`: se mató el script con el enlace cortado y la sucursal quedó
reconectada.

## 2026-09-26 — Corrección: el pool tampoco puede caerse con una conexión en uso

**Qué estaba mal.** La entrada del 26-sep sobre el pool cubría solo las conexiones
**inactivas**. `pg-pool` le quita al cliente su manejador de `error` mientras está
prestado. `pool.query()` se protege solo, pero `pool.connect()` no, y es lo que usa
`enTransaccion`, es decir, toda transacción manual del proyecto. Si la base termina una
conexión en medio de una transacción, `pg` emite `error` sobre el cliente y el proceso se
cae. Vale para el central y para las sucursales: una base de sucursal que se reinicia en
medio de una venta tumbaba su API.

**Cómo apareció.** La prueba del consumidor detiene `db-central`. En la parte 2 no había
publicador y nadie tenía una transacción abierta en ese instante, así que pasaba. Con el
publicador activo, una entrega estaba en curso al detener la base y la API central se
reinició. Pasar dependía de la suerte.

**Decisión.** `enTransaccion` agrega un manejador de `error` mientras el cliente está
prestado y lo quita al devolverlo. Si la conexión se rompió, la devuelve al pool con el
error (`release(error)`) para que la destruya en lugar de reutilizarla. La consulta en
curso igual falla y el error sigue su camino normal: 500 en la sucursal, 503 en el
central.

**Demostración determinista.** `scripts/prueba-conexion-perdida/prueba.mjs` no depende del
momento en que se detiene una base:
1. Se toma desde afuera el bloqueo de una fila.
2. Una transacción de la API queda esperándolo con su conexión prestada.
3. Se termina esa conexión con `pg_terminate_backend`.

Sin la corrección, las dos APIs se cayeron (la caja recibió "fetch failed"). Con la
corrección, la venta responde 500 sin escribir nada y la siguiente entra. El consumidor
responde 503, el publicador lo registra como `ATASCADA` y, al liberar el bloqueo, el
mensaje se aplica. Ninguna API se reinicia.

**Consecuencia para la prueba del consumidor.** Esa prueba entrega mensajes a mano y sus
casos 3 y 4 serían carreras contra el publicador. Ahora lo pausa en las tres sucursales
(`SINCRONIZACION_ACTIVA=0`) mientras corre y lo reanuda al terminar.

## 2026-09-26 — Los puertos de las sucursales los publica un reenviador de una sola red

**Qué pasó.** En la corrida final desde base limpia, con el enlace cortado, la primera
venta no respondió desde el anfitrión. La sucursal estaba viva: desde adentro del
contenedor, la venta entraba en 76 ms. Lo que se perdía era el **puerto publicado**
(`localhost:3001`). En Docker Desktop, el reenvío de un puerto del anfitrión hacia un
contenedor conectado a varias redes puede romperse al desconectar una de ellas, y cortar
el enlace es exactamente eso.

Se probó con un Compose descartable, recreando el contenedor varias veces:
- falla o no **al azar**, sin importar el orden alfabético de los nombres de red;
- no lo arreglan `priority` ni `gw_priority`;
- a veces se arregla al reconectar y a veces no.

Una corrida de prueba anterior había pasado por esa casualidad.

**Decisión.** Las APIs de sucursal ya no publican puertos. Cada sucursal tiene un
`acceso-sucursal-N`:
- es un reenviador TCP de 20 líneas (`infra/reenvio-tcp.js`, imagen oficial
  `node:24-alpine`, la misma base del Dockerfile, sin dependencias de npm);
- vive **solo** en la red de la sucursal y publica el puerto `300N`;
- es la "LAN de la tienda" por la que entra la caja. Como tiene una sola red, el corte
  del enlace no lo toca.

La topología aprobada no cambia: la API de sucursal sigue en su red y en la del enlace.
Probado cinco veces con contenedores recién creados: con el enlace cortado, `/salud` da
200 y la venta 201 desde el anfitrión, todas las veces.

**Alternativas descartadas.**
- **Hablar con la sucursal cortada por `docker exec`.** Arreglaba el script, pero no un
  `curl localhost:3001` hecho a mano en la presentación, que habría fallado una vez sí y
  otra no.
- **Un "router de la tienda" por sucursal** por el que pasara todo el tráfico hacia el
  central, con la API en una sola red. Modela mejor la realidad, pero cambia la topología
  aprobada y agrega un salto también en el camino del 2PC. Queda como opción si hiciera
  falta.

**Costo.** Las latencias que mide la prueba de concurrencia incluyen un salto más, del
anfitrión al reenviador y de ahí a la API. El informe lo aclara. Es un artefacto de la
máquina de desarrollo: en GCP la caja llega a la VM de la sucursal directamente.

## 2026-09-26 — Traslados: identificador global y documentos por nodo

**Decisión.**
- **Identificador global.** Un traslado se identifica en toda la red por `id_traslado UUID`,
  que genera el central al registrar la solicitud.
- **`gid` de cada transacción preparada.** Se deriva del identificador:
  `tr_<uuid sin guiones>_o` (origen) o `_d` (destino). Son 37 caracteres, así que entran en
  `participante.id_tx_preparada VARCHAR(50)`. Un nombre legible como
  `traslado_<uuid>_origen` ocuparía 52.
- **En la sucursal** (migración `007`): `documento.id_traslado UUID NULL` y un índice
  único **`(id_traslado, tipo)`**, no `id_traslado` solo. El destino tiene un `TRASLADO` y,
  al recibir, un `RECEPCION` con el mismo `id_traslado`. Un `CHECK` limita `id_traslado` a
  esos dos tipos.
- **En el central** (migración de la parte 2) hay por traslado:
  - el documento del coordinador (el `Traslado` del ER, con `id_traslado` nulo; lo vincula
    `traslado.id_documento`);
  - la copia del `TRASLADO` de cada participante;
  - la copia del `RECEPCION` del destino.

  Solo las copias llevan `id_traslado`, con un índice único parcial
  `(id_sucursal, id_traslado, tipo)` y una llave foránea a `traslado`. El plan decía
  `UNIQUE (id_sucursal, id_traslado)`, que habría hecho chocar el `TRASLADO` y el
  `RECEPCION` del destino.

**Por qué UUID y no el `SERIAL` del central.** Existe antes de insertar nada, hace
idempotentes los reintentos de preparar, confirmar y abortar, y no ata a la sucursal a
la secuencia de otro nodo.

## 2026-09-26 — La API de la sucursal es el participante del 2PC

**Decisión.** La sucursal expone `POST /traslados/:id/preparar`, `/confirmar` y `/abortar`.
El coordinador nunca se conecta a la base de una sucursal (mínimo privilegio, sección 7.2).

**Fase 1, en una transacción que termina en `PREPARE TRANSACTION`** (`enTransaccionPreparada`):
- **Origen:** documento `TRASLADO` con líneas `EN_TRANSITO` y precio 0 (no es una
  operación comercial); `UPDATE … WHERE cantidad >= n RETURNING` sobre la existencia (el
  `UPDATE` condicional de la sección 5.2); movimiento `TRASLADO_SALIDA`; y la fila de
  outbox.
- **Destino:** documento `TRASLADO` con líneas `EN_TRANSITO` y la fila de outbox. **No toca
  la existencia**: la mercadería está en tránsito hasta la recepción, que será una
  operación local aparte.
- **La fila de outbox va dentro de la transacción preparada:** nadie la ve hasta el
  `COMMIT PREPARED` y desaparece con un `ROLLBACK PREPARED`.
- **El voto.** Un rechazo de negocio (existencia, producto, usuario, bloqueo) es un voto NO
  con su motivo, en un 200. Un error de forma, como un rol dirigido a la sucursal
  equivocada, es un 400.

**Idempotencia, por `gid`:**

| Llamada | Estado del `gid` | Respuesta |
|---|---|---|
| preparar | ya preparado | SÍ, sin hacer nada |
| confirmar | preparado | `CONFIRMADA` |
| confirmar | ya confirmado | `YA_CONFIRMADA` |
| confirmar | ni preparado ni confirmado | 409 `NO_PREPARADA` |
| abortar | preparado | `ABORTADA` |
| abortar | nada preparado | `NO_PREPARADA` |
| abortar | ya confirmado | 409 `YA_CONFIRMADA` |

Los dos 409 significan que alguien violó el protocolo.

**Detalles.**
- `PREPARE`, `COMMIT PREPARED` y `ROLLBACK PREPARED` no aceptan parámetros, así que el
  `gid` se interpola en el SQL. Antes se valida contra `^[a-z0-9_]{1,200}$`.
- Demostrado que la transacción preparada sobrevive a un reinicio del motor (caso 4 de
  `scripts/prueba-participante/prueba.mjs`).

## 2026-09-26 — Preparar también espera con plazo, y ante un choque decide `pg_prepared_xacts`

**Decisión.** La fase 1 corre con `lock_timeout` (3 s).
- **Choque con otra preparación del mismo traslado:** vence `lock_timeout`, choca con el
  índice único `(id_traslado, tipo)` o el `gid` está repetido. En los tres casos el
  participante vuelve a consultar `pg_prepared_xacts`: si el `gid` quedó preparado, vota
  SÍ (`ya_preparada: true`); si no, vota NO.
- **Fila retenida por otra transacción:** una venta lenta u otro traslado. El participante
  vota NO con motivo `PRODUCTO_BLOQUEADO` en lugar de colgarse.

**Demostración.** Caso 9. Para que las dos preparaciones pasen el chequeo inicial antes
de que alguna prepare, se retiene la fila desde afuera y se suelta a los 500 ms. La
primera prepara (511 ms); la segunda queda esperando a una transacción ya preparada,
vence `lock_timeout`, consulta y vota SÍ (3027 ms). Queda una sola transacción preparada,
y al confirmar la existencia baja una sola vez. Una primera versión del caso respondía en
15 ms porque la segunda llegaba cuando la primera ya estaba preparada: no probaba el
choque, y se corrigió.

## 2026-09-26 — `lock_timeout` en venta y ajuste: 409 `PRODUCTO_BLOQUEADO`

**Decisión.** Venta, ajuste y preparación de traslado esperan una fila bloqueada como
máximo `LOCK_TIMEOUT_MS` (3 s). Vencido el plazo, PostgreSQL responde `55P03` y la API
devuelve **409 `PRODUCTO_BLOQUEADO`**, con el mensaje "producto bloqueado temporalmente".
El mensaje no dice que haya un traslado en curso: el bloqueo puede venir de una venta
lenta.

**Por qué.** Una transacción preparada retiene la fila de existencia del origen hasta la
decisión, y si el coordinador se cae eso puede durar mucho (sección 5.4). Sin plazo, la
caja se quedaría esperando sin respuesta. Con plazo, la caja recibe un rechazo claro y
el resto de los productos sigue vendiéndose: el caso 1 mostró el producto retenido con
409 a los 3 s y otro producto vendido en 29 ms.

**Medición.** La prueba de concurrencia completa con `lock_timeout` activo dio 0 rechazos
por bloqueo en las seis corridas y los mismos resultados que antes: las esperas de la
estrategia pesimista son de milisegundos. El informe tiene una columna nueva, "motor
(bloqueo)".

## 2026-09-26 — Ventana conocida: "abortar" antes que "preparar" deja una transacción preparada huérfana

**Qué pasa.**
1. El coordinador decide abortar mientras el "preparar" de un participante todavía está
   en curso (por ejemplo, esperando una fila con `lock_timeout`).
2. El participante responde `NO_PREPARADA` al "abortar", porque todavía no hay nada que
   deshacer.
3. Después el "preparar" termina y prepara.

Queda una transacción preparada que nadie va a confirmar ni abortar, y mientras tanto
retiene las filas: una venta de ese producto recibe 409 `PRODUCTO_BLOQUEADO`. Demostrado
en el caso 10: "abortar" respondió `NO_PREPARADA`, el "preparar" votó SÍ 835 ms después y
la venta recibió 409 a los 3 s.

**Cómo se cierra.** El reconciliador de la parte 3 revisa `pg_prepared_xacts`, encuentra el
`gid` con más antigüedad que su plazo, le pregunta al coordinador y, como la bitácora dice
`ABORTADO`, ejecuta `ROLLBACK PREPARED`. La ventana dura desde que "preparar" termina hasta
que pasa el plazo del reconciliador más un ciclo. En la parte 1 la limpieza se hace a mano
al final del caso.

**Cómo se achica.** La huérfana solo aparece si el coordinador aborta mientras un
"preparar" todavía está en vuelo, es decir, si su plazo de votación es más corto que lo
que puede tardar una preparación. El coordinador de la parte 2 usará un plazo de votación
mayor que `líneas × lock_timeout` más un margen. Eso la vuelve rara, no imposible: un
mensaje de red puede demorarse más que cualquier plazo.

**Alternativa descartada.** Que "abortar" deje una marca persistente en la sucursal, para
que un "preparar" posterior del mismo `gid` vote NO (*presumed abort* con lápida). Cierra
la ventana, pero necesita una tabla nueva de estado del protocolo en la sucursal, que
el diseño reserva al central, y una política de limpieza de esas marcas. Con el
reconciliador, la ventana queda acotada y visible.

## 2026-09-26 — Riesgo conocido del hito: sin autenticación entre coordinador y participante

**Qué es.** Los endpoints `preparar`, `confirmar` y `abortar` de la sucursal no autentican a
quien los llama. En Docker Compose solo se llega a ellos por la red del enlace o por
`acceso-sucursal-N` desde la máquina de desarrollo. Aun así, cualquiera con acceso de red
podría confirmar o abortar un traslado ajeno. El resto del hito tampoco tiene
autenticación.

**Decisión.** Se deja para el hito de GCP: cada componente tendrá su cuenta de servicio con
el rol mínimo (sección 7.2) y Cloud Run autenticará las llamadas con IAM. La alternativa
intermedia, un token compartido en una cabecera, se descartó por ahora: sería una
credencial más que manejar a mano y que habría que reemplazar igual en GCP.

**Actualización (2026-10-25).** Resuelto por otra vía: firma HMAC por petición con una clave por
sucursal, que sirve igual en Compose y en GCP. Ver «Autenticación entre nodos».

## 2026-09-26 — Hasta la parte 2, el central rechaza los mensajes de traslado

**Qué pasa.** La parte 1 del hito solo toca las sucursales. Los participantes ya encolan su
documento `TRASLADO` al confirmar, pero el consumidor del central todavía no acepta ese
tipo: responde `400 MENSAJE_INVALIDO` y, tras 5 rechazos, esas filas pasan a `FALLIDO`,
como corresponde a un mensaje que el central no puede aplicar. En la corrida de la parte 1
quedaron 6 en la sucursal 1 y 2 en la sucursal 2. La parte 2 extiende el consumidor.

## 2026-10-08 — Migración del central para traslados, y `id_usuario_destino`

**Decisión.** La migración `central/008_traslado.sql` implementa lo decidido en "Traslados:
identificador global y documentos por nodo":
- `traslado.id_traslado`;
- `documento.id_traslado`, con llave foránea a `traslado`;
- `CHECK`: solo las copias llevan `id_traslado`, y solo de tipo `TRASLADO` o `RECEPCION`;
- índice único parcial `(id_sucursal, id_traslado, tipo)`.

Agrega dos cosas que no estaban en esa entrada:
- **`traslado.id_usuario_destino`.** El documento del destino lo firma un usuario de esa
  sucursal, porque la base de la sucursal solo tiene sus propios usuarios. El coordinador
  lo necesita para pedir el voto del destino.
- **`CHECK (id_sucursal_origen <> id_sucursal_destino)`.**

## 2026-10-08 — Coordinador: bitácora antes de cada paso, y la decisión se escribe condicionalmente

**Decisión.** `POST /traslados` ejecuta el protocolo:
1. **Registro.** Documento del coordinador en `BORRADOR` y traslado `SOLICITADO`.
2. **Las dos sucursales en línea.** Si alguna no responde a `/salud`, el traslado queda
   `SOLICITADO` (202) y no se pide ningún voto (sección 5.4).
3. **Bitácora antes de pedir el primer voto.** `transaccion_global` en `INICIADO`, los dos
   participantes con su `gid`, y el traslado en `PREPARANDO`.
4. **Votos en paralelo.** El plazo es `max(PLAZO_VOTACION_MS, líneas × lock_timeout del
   participante + 2 s)`: así un participante que espera filas con `lock_timeout` alcanza a
   votar antes de que se decida sin él, lo que achica la ventana de la huérfana. Una
   respuesta que no llega, o que no es un voto válido, cuenta como NO.
5. **Decisión condicional.** Ver la entrada siguiente.
6. **Propagación de la decisión vigente.** Con todos los acuses, `FINALIZADO`. El traslado
   y el documento del coordinador reflejan la decisión en la misma transacción que la
   escribe: `COMPROMETIDO` y `CONFIRMADO`, o `ABORTADO` y `ANULADO`.

## 2026-10-08 — Carrera coordinador vs. recuperación: gana una sola escritura

**El riesgo.** El coordinador y la recuperación pueden querer decidir el mismo traslado:
el coordinador tiene los dos votos SÍ pero se demora antes de escribir, y la recuperación
ve un `INICIADO` vencido y lo aborta. Si las dos escrituras fueran incondicionales, la
segunda pisaría a la primera, y una punta podría confirmar mientras la otra deshace.

**Decisión.** Toda escritura de la decisión es condicional:
- el coordinador hace `UPDATE transaccion_global SET estado = :decision … WHERE … AND
  estado = 'INICIADO'`;
- la recuperación hace lo mismo para `ABORTADO`, tomando la fila con `FOR UPDATE SKIP
  LOCKED` y exigiendo además que esté vencida.

Quien actualiza la fila decide. Quien encuentra 0 filas lee la decisión vigente y ejecuta
**esa**, aunque haya querido otra. La propagación también toma la fila con `SKIP LOCKED`:
si alguien más la está propagando, no hace nada.

**Demostración** (`scripts/prueba-coordinador/prueba.mjs`, caso 5):
1. El coordinador recibe los dos SÍ y, por un retardo inyectado, espera 9 s antes de decidir.
2. Vencido el plazo de `INICIADO` (6 s en la prueba), se disparan dos recuperaciones a la
   vez. Una aborta y propaga; la otra omite la transacción por `SKIP LOCKED`.
3. El coordinador despierta, su `UPDATE` no encuentra la fila en `INICIADO`, lee `ABORTADO`
   y lo ejecuta.

Resultado: traslado `ABORTADO`, transacción `FINALIZADO`, nada preparado ni escrito en
ninguna sucursal.

Con la condición quitada a propósito, el coordinador pisó el `ABORTADO`: la bitácora quedó
en `COMPROMETIDO` y el documento en `CONFIRMADO`, mientras los dos participantes ya habían
hecho `ROLLBACK PREPARED`. El registro afirma un traslado que no se aplicó en ningún lado,
y `confirmar` responde 409 `NO_PREPARADA` para siempre.

## 2026-10-08 — La recuperación se dispara de tres formas y puede correr en paralelo

**Decisión.** `ejecutarRecuperacion` es una sola función con tres disparadores:
- **al arrancar el proceso**, que es la recuperación después de una caída;
- **un intervalo** dentro del proceso (`RECUPERACION_INTERVALO_MS`, 5 s);
- **`POST /traslados/recuperacion/ejecutar`**.

En GCP, Cloud Run escala a cero y no hay un intervalo vivo: Cloud Scheduler llamará al
endpoint. Como cada transacción global se toma con `FOR UPDATE SKIP LOCKED` y la decisión
se escribe condicionalmente, los disparadores pueden coincidir sin pisarse. Probado: dos
ejecuciones simultáneas sobre la misma transacción (una la procesa, la otra la omite), y
dos ejecuciones sin pendientes (ninguna hace nada).

**Qué hace con cada estado:**
- `INICIADO` con más de `RECUPERACION_PLAZO_INICIADO_MS` (60 s) → `ABORTADO`. Ese plazo
  tiene que superar el de votación; si no, la recuperación abortaría votaciones en curso.
- `COMPROMETIDO` → reenvía `confirmar`, sin límite de intentos.
- `ABORTADO` → reenvía `abortar`.
- `FINALIZADO` → nada.

**Alternativa descartada.** Solo el intervalo dentro del proceso. En Cloud Run no hay
proceso vivo entre peticiones, así que las transacciones pendientes esperarían hasta el
próximo traslado.

## 2026-10-08 — El consumidor verifica las copias de traslados contra el coordinador

**Decisión.** Una copia `TRASLADO` o `RECEPCION` solo se aplica si concuerda con el registro
del coordinador:
- el traslado existe y está `COMPROMETIDO` o `RECIBIDO`;
- la sucursal tiene el papel que dice (`RECEPCION` solo del destino);
- en la copia `TRASLADO`, el origen y el destino coinciden con los del registro;
- el tipo de movimiento corresponde al tipo de documento (`TRASLADO_SALIDA` en un
  `TRASLADO`, `TRASLADO_ENTRADA` en un `RECEPCION`).

Si no, responde **422 `MENSAJE_INCONSISTENTE`**: el mensaje contradice lo que el central
sabe y reenviarlo no lo arregla. El `TRASLADO` del destino es el único documento que viaja
sin movimientos. Una `RECEPCION` marca `RECIBIDO` las líneas de la copia del destino y pone
el traslado en `RECIBIDO`.

**Caso límite aceptado.** Si la `RECEPCION` llegara antes que la copia del `TRASLADO` del
destino, las líneas de esa copia quedarían en `EN_TRANSITO`. El orden por sucursal lo
impide: el `TRASLADO` del destino se confirma antes que la recepción y tiene un
`id_outbox` menor. Solo podría pasar si esa fila quedara en `FALLIDO`, y la verificación
(b) lo detecta.

## 2026-10-08 — Recepción local en el destino

**Decisión.** `POST /traslados/:id/recibir` en el destino, en una transacción local:
- documento `RECEPCION` firmado por quien recibe (ENCARGADO o ADMIN);
- movimientos `TRASLADO_ENTRADA`, y la existencia sube;
- las líneas del `TRASLADO` pasan de `EN_TRANSITO` a `RECIBIDO`;
- la fila de outbox.

El documento `TRASLADO` se bloquea con `FOR UPDATE` para que dos recepciones simultáneas
se serialicen. Si ya se recibió, responde 200 con `sin_cambios`. En el origen responde 409
`NO_ES_DESTINO`: el documento del origen es el que tiene movimientos `TRASLADO_SALIDA`. Se
recibe lo enviado completo; una diferencia se corrige con un AJUSTE.

## 2026-10-08 — Corrección: la `referencia` del payload no coincidía con la del documento local

**Qué estaba mal.** En la parte 1, el participante guardaba en su documento una
`referencia` ("Traslado hacia la sucursal 2") y en el payload mandaba `null`. La copia del
central no habría sido idéntica. No se vio porque en la parte 1 el central todavía
rechazaba esos mensajes. Se corrigió en el participante y en la recepción, y la
verificación (b) ahora compara también `id_traslado`.

## 2026-10-08 — Verificaciones de red: libro, conservación de unidades y traslados

**Decisión.** Con las tres sucursales, `verificar.mjs` agrega:
- **Libro de cada sucursal:** semilla + Σ movimientos (con signo) = existencia, producto
  por producto. La semilla sale de la misma fórmula que `03_existencia_sucursal.sql`.
- **Conservación de unidades, con los datos de las sucursales:** Σ existencias + en
  tránsito = Σ semilla + Σ movimientos que no son de traslado. "En tránsito" son las
  líneas `EN_TRANSITO` del documento `TRASLADO` del **destino**: si el 2PC se aplicara en
  el origen y no en el destino, esas unidades no estarían en ninguna parte y la ecuación
  no cerraría.
- **La misma conservación con los datos del central:** consolidado más líneas `EN_TRANSITO`
  de la copia del destino.
- **Traslados:**
  - ninguna transacción global sin `FINALIZADO`;
  - ninguna transacción preparada `tr_*` en las sucursales;
  - en los comprometidos y recibidos, la copia de cada participante tiene las mismas
    líneas que el documento del coordinador;
  - en los abortados o solicitados, ningún documento en las sucursales.

Son verificaciones **en reposo**: durante un 2PC en curso no tienen por qué cumplirse.

**Dos pruebas dejan la base sin cuadrar a propósito.** Hay que correrlas sobre una base
descartable:
- **La prueba de concurrencia**, con `SIN_CONTROL`, rompe el libro: sobrevende, que es lo
  que demuestra.
- **`prueba-participante`**, que prueba el participante sin coordinador. Deja traslados que
  el central no conoce, sus copias reciben 422 y quedan en `FALLIDO`.

## 2026-10-08 — `PUERTO_DB_CENTRAL`: el puerto del anfitrión de `db-central` es configurable

**Qué pasó.** Apareció un PostgreSQL 18 instalado en la máquina de desarrollo
(`/Library/PostgreSQL/18`) escuchando en el 5432, y `db-central` no pudo publicar su puerto.

**Decisión.** `PUERTO_DB_CENTRAL`, que por defecto sigue siendo 5432, fija el puerto del
anfitrión. Los scripts lo respetan (`verificar.mjs` arma `URL_CENTRAL` con él). Dentro de
las redes de Compose nada cambia.

## 2026-10-08 — Inyección de fallas en el coordinador, con doble protección

**Decisión.** `POST /traslados` acepta `_prueba` con dos opciones:
- `falla: 'despues_de_votos' | 'despues_de_decidir'`: el proceso termina en ese punto;
- `retardo_antes_de_decidir_ms`.

Solo se acepta si `PERMITIR_FALLAS_INYECTADAS=1`; si no, responde 400 sin registrar nada.
El Compose la deja en `0`, y solo los scripts de prueba recrean `api-central` con `1`.
Mismo criterio que `SIN_CONTROL`: algo que rompe el protocolo a propósito no puede estar
a una variable de distancia de producción.

## 2026-10-09 — La recuperación retoma los traslados en `SOLICITADO`

**Qué faltaba.** Un traslado que quedaba en `SOLICITADO` porque alguna sucursal estaba
fuera de línea no lo retomaba nadie cuando la sucursal volvía.

**Decisión.** Los pasos 2 a 5 del protocolo son una función, `ejecutarProtocolo`, que lee
todo de la bitácora: las líneas del documento del coordinador, las sucursales y los dos
usuarios. La usan `POST /traslados` y la recuperación. En cada pasada, la recuperación
revisa los `SOLICITADO`:
- si las dos sucursales responden, ejecuta el protocolo;
- si no, el traslado sigue en espera y aparece en `solicitados_en_espera` del resumen.

El traslado se **toma** con `FOR UPDATE SKIP LOCKED` y la condición `estado = 'SOLICITADO'`,
en la misma transacción que escribe `INICIADO` y los participantes. Si dos procesos lo
intentan a la vez, uno solo ejecuta el protocolo; el otro responde
`TOMADO_POR_OTRO_PROCESO`. Un `SOLICITADO` no vence: espera hasta que las dos sucursales
estén en línea.

**Demostración.** `prueba-coordinador`, caso 4:
1. Con el destino cortado, el traslado queda en `SOLICITADO` y una recuperación lo deja
   en espera.
2. Se reconecta el destino y se disparan dos recuperaciones a la vez: lo retoma una sola.
3. El traslado queda `COMPROMETIDO` y `FINALIZADO`, y el origen descontó.

## 2026-10-09 — Reconciliador del participante: deshacer solo con ABORTADO o con un 404 explícito

**Decisión.** Cada sucursal tiene un reconciliador que corre cada
`RECONCILIADOR_INTERVALO_MS` (5 s) y también con `POST /traslados/reconciliacion/ejecutar`.
Usa un pool propio de una conexión. Revisa `pg_prepared_xacts`, toma los `gid` `tr_*` con
más antigüedad que `RECONCILIADOR_PLAZO_MS` (30 s) y le pregunta al coordinador con
`GET /traslados/decision/:gid`:

| Respuesta del central | Acción |
|---|---|
| `COMPROMETIDO` | `COMMIT PREPARED` |
| `ABORTADO` | `ROLLBACK PREPARED` |
| 404 con código `GID_DESCONOCIDO` | `ROLLBACK PREPARED` |
| `PENDIENTE` (la bitácora sigue en `INICIADO`) | esperar |
| timeout, error de red, 5xx, o un 404 sin ese código | **esperar, nunca deshacer** |

**Por qué.** Si "sin respuesta" significara deshacer, un central caído haría que la
sucursal deshaga un traslado que el coordinador ya comprometió, mientras la otra punta lo
confirma. El 404 tiene que traer el código propio: un 404 genérico, como el de una
versión del central sin este endpoint, no prueba que el `gid` no exista.

Si otro proceso resolvió el `gid` entre la lectura y la acción (el coordinador, o el
reconciliador disparado por el endpoint), `COMMIT` o `ROLLBACK PREPARED` responde "no
existe" y se ignora. `/salud` muestra `transacciones_en_duda`: cuántas hay, la más vieja,
sus `gid` y la última pasada del reconciliador, con el motivo de cada espera.

**Demostración** (`scripts/prueba-reconciliador/prueba.mjs`):
- **Caso 3, central caído con una transacción `COMPROMETIDO` en duda.** El coordinador
  escribe `COMPROMETIDO` y `api-central` se detiene antes de propagar. Durante 7 s, el
  reconciliador del origen registra `SIN_RESPUESTA: ENOTFOUND`, no deshace nada y las dos
  puntas siguen preparadas; una venta del producto recibe 409. Al volver el central, las
  dos puntas se confirman en 0,1 s (lo hizo la recuperación del coordinador al arrancar).
- **Caso 4, sin enlace.** El destino pierde el enlace justo cuando se decide
  `COMPROMETIDO`. Sin enlace, su reconciliador espera; al volver el enlace confirma solo,
  y la recuperación del coordinador finaliza recibiendo `YA_CONFIRMADA`.
- **Caso 2, huérfana con bitácora `ABORTADO`.** El coordinador aborta porque el voto del
  origen no llegó dentro de un plazo corto inyectado, y el origen prepara después. El
  reconciliador ve `ABORTADO` y deshace.
- **Caso 5, 404 explícito.** Un `gid` que el coordinador no conoce se deshace.

## 2026-10-09 — El caso "abortar antes que preparar" se cierra solo

El caso 10 de `prueba-participante` ya no tiene limpieza manual: la huérfana la deshace el
reconciliador. Como esa prueba no usa coordinador, el central responde 404
`GID_DESCONOCIDO`. Con la configuración por defecto, la ventana medida fue de 30,8 s:
plazo de 30 s más un ciclo. Durante ese tiempo, una venta del producto recibe 409
`PRODUCTO_BLOQUEADO`. La misma situación con coordinador, con la bitácora en `ABORTADO`,
está en `prueba-reconciliador`, caso 2.

## 2026-10-09 — La bitácora del coordinador no se purga

**Decisión.** El central no borra nunca registros de `transaccion_global` ni de
`participante`. No existe ningún proceso de purga en el proyecto.

**Por qué.** El reconciliador interpreta un 404 `GID_DESCONOCIDO` como "nunca se decidió
nada: deshacer". Si la bitácora de un traslado comprometido se purgara mientras alguna
sucursal todavía tiene preparado ese `gid`, el reconciliador recibiría 404 y deshacería
una punta que debía confirmarse.

**Si algún día se quisiera purgar,** solo podría hacerse con registros en `FINALIZADO` y
con la certeza de que ninguna sucursal conserva el `gid` en `pg_prepared_xacts`, lo que
exige consultar a todas las sucursales. Por simplicidad, no se purga.

## 2026-10-09 — Límite conocido: restaurar el central con PITR durante un traslado

**Qué pasa.** El ejercicio de la sección 6.3 restaura la base del central a un punto
anterior en el tiempo. Si ese punto es anterior a la escritura de `COMPROMETIDO` de un
traslado, la restauración borra el registro del coordinador. Lo que pasa después:
- la sucursal que ya confirmó (porque recibió `confirmar` antes de la restauración) queda
  confirmada;
- la sucursal que todavía tenía su transacción preparada pregunta, recibe 404
  `GID_DESCONOCIDO` y la deshace.

Resultado: una punta aplicada y la otra no, exactamente la inconsistencia que el 2PC
evita. Lo mismo vale para restaurar a un punto anterior a `INICIADO`, que hace olvidar un
traslado cuyas copias igual van a llegar por sincronización y el consumidor va a
rechazar con 422.

**Por qué no se resuelve.** Lo correcto sería que la bitácora del coordinador sobreviva a
la restauración (por ejemplo, en otro almacenamiento), o reconciliar las sucursales contra
el central restaurado. Las dos cosas están fuera del alcance.

**Regla.** El ejercicio de PITR se hace **sin traslados en curso**: ninguna transacción
global fuera de `FINALIZADO` y ninguna transacción preparada `tr_*` en las sucursales.
`verificar.mjs` sirve para comprobarlo antes de empezar ("traslados: bitácora finalizada,
nada preparado").

## 2026-10-09 — Dos inyecciones más en el coordinador

`_prueba` acepta también:
- `retardo_despues_de_decidir_ms`: deja a los participantes en duda con la decisión ya
  escrita (caso 3 de `prueba-reconciliador`);
- `plazo_votacion_ms`: fuerza un plazo de votación corto para producir una huérfana con
  la bitácora en `ABORTADO` (caso 2).

La protección es la misma: solo con `PERMITIR_FALLAS_INYECTADAS=1`.

## 2026-10-09 — La inyección de fallas es por petición (`_prueba.falla`), no una variable `FALLA_INYECTADA`

**Aclaración.** El plan de traslados hablaba de una variable `FALLA_INYECTADA`. Lo que quedó
implementado y registrado en la parte 2 ("Inyección de fallas en el coordinador, con doble
protección") es el campo `_prueba.falla` (`despues_de_votos` | `despues_de_decidir`) en el
cuerpo de `POST /traslados`, junto con los retardos y el plazo de votación forzado. Las dos
protecciones son las acordadas: solo se acepta con `PERMITIR_FALLAS_INYECTADAS=1`, y esa
variable solo la definen los scripts de prueba.

**Por qué por petición.** Cada escenario elige su punto de falla sin recrear el contenedor
entre uno y otro, y una petición sin `_prueba` nunca falla aunque el permiso esté activo.

## 2026-10-09 — "Al reiniciar, INICIADO aborta" se implementa con un plazo

**Decisión.** La regla del proyecto dice "al reiniciar: INICIADO aborta". La recuperación lo hace
cuando el registro en `INICIADO` supera `RECUPERACION_PLAZO_INICIADO_MS`, no apenas arranca.

**Por qué.** En Cloud Run puede haber varias instancias del coordinador. Una instancia que
arranca no sabe si el `INICIADO` lo dejó un proceso que murió o una instancia vecina que
está votando en ese momento. Abortarlo apenas arranca podría deshacer una votación sana.
Con el plazo, que es mayor que el de votación, solo se aborta lo que ningún proceso vivo
puede estar decidiendo, y la escritura condicional cubre la carrera si el plazo se
equivoca.

**Costo.** Las filas del origen quedan retenidas hasta que vence el plazo: 60 s por
defecto, y en la demostración 8 s (el escenario 5 abortó a los 5,9 s de la caída). Una
venta de ese producto recibe 409 durante ese tiempo. Es el bloqueo de la sección 5.4,
acotado y visible en `/salud`.

## 2026-10-09 — Demostración del 2PC con adaptador de plataforma

**Decisión.** `scripts/demo-2pc/ejecutar.sh` corre seis escenarios con línea de tiempo:
1. camino feliz y recepción;
2. voto NO;
3. destino sin enlace y retoma;
4. el coordinador muere después de decidir;
5. el coordinador muere en `INICIADO`;
6. el coordinador queda caído.

Genera `docs/resultados-2pc.md`, que es la evidencia del informe de CC6: bitácora, origen,
destino, transacciones preparadas y conservación de unidades por escenario. Con
`--escenario N` corre uno solo, para mostrar en vivo.

Lo que depende de la plataforma está en dos adaptadores. El de enlace (`cortar_enlace` /
`restablecer_enlace`) ya existía. El de plataforma, `scripts/demo-2pc/plataforma-compose.sh`,
es nuevo y define `preparar_entorno_2pc`, `restaurar_entorno_2pc`, `detener_coordinador`
y `arrancar_coordinador`. En GCP se escriben otros dos archivos con las mismas funciones
(firewall por etiqueta, Cloud Run) y `demo.mjs` no cambia. Que el coordinador vuelva se
detecta por HTTP (`/salud`), no con comandos de Docker.

**Observación del modelo.** La copia del origen conserva sus líneas en `EN_TRANSITO`
después de la recepción: la recepción es una operación local del destino y el origen no se
entera. El estado `RECIBIDO` lo tienen el destino y el central (`traslado.estado`). Para el
origen, "en tránsito" significa "despachado". Si hiciera falta que el origen lo sepa, sería
una sincronización del central hacia las sucursales, que no está en el alcance.

## 2026-10-09 — `scripts/probar-todo.sh`

**Decisión.** Un solo comando corre, desde base limpia, todas las suites en orden:
1. las que dejan la base cuadrando: ajuste, consumidor, publicador, conexión perdida,
   coordinador, reconciliador, demostración del 2PC y corte de enlace;
2. `verificar.mjs` completo, que tiene que pasar;
3. al final, las que descuadran la base a propósito: el participante sin coordinador y la
   prueba de concurrencia con `SIN_CONTROL`.

Resume resultado y duración de cada suite, guarda los registros, regenera los dos informes
y sale con código 1 si algo falla. La prueba de concurrencia siempre termina en 0, así
que su criterio se toma del informe: `PESIMISTA` y `OPTIMISTA` pasan todo y `SIN_CONTROL`
sobrevende. Es lo que se corre antes de pasar a GCP.

## 2026-10-09 — `movimiento` es inmutable a nivel de base de datos

**Decisión.** Las migraciones `sucursal/008_movimiento_inmutable.sql` y
`central/009_movimiento_inmutable.sql` agregan sobre `movimiento`:
- un trigger `BEFORE UPDATE OR DELETE … FOR EACH ROW`;
- un trigger `BEFORE TRUNCATE … FOR EACH STATEMENT`, porque `TRUNCATE` no dispara los
  triggers de fila.

Los dos llaman a `movimiento_inmutable()`, que rechaza la operación con SQLSTATE `23001`
(`restrict_violation`), el mensaje "movimiento es una bitácora inmutable: UPDATE no está
permitido (id_movimiento N)" y una pista: las correcciones se registran con un asiento de
ajuste. Al ser de la clase 23, si alguna vez lo disparara la API, se traduciría como las
demás restricciones de la base: 409 `RESTRICCION_BD` en la sucursal y 422 en el
consumidor del central.

**Por qué ahora.** Hasta aquí esta regla del proyecto la cumplía el código: ninguna
sentencia del proyecto modifica `movimiento`, y se verificó buscando `UPDATE`, `DELETE` y
`TRUNCATE` en el código, los scripts y las migraciones, y que no hubiera borrados en
cascada. Con el trigger la garantiza el motor para cualquier cliente que se conecte.
INSERT sigue permitido.

**Demostración.** `scripts/prueba-movimiento-inmutable/prueba.mjs`, incluida en
`probar-todo.sh`, intenta en los cuatro nodos `UPDATE` de una fila y de todas, `DELETE` de
una fila y de todas, y `TRUNCATE`, cada intento dentro de `BEGIN … ROLLBACK`. Verifica el
código `23001`, el mensaje con la operación rechazada, la huella de toda la bitácora antes
y después, y que una venta nueva se sigue registrando y llega al central. Corrida antes
de aplicar la migración, falló en todos los intentos ("NO FALLÓ"); después, 29 de 29.

**Límite conocido.** El usuario `sima` es superusuario y podría hacer `ALTER TABLE
movimiento DISABLE TRIGGER`. Quitarle ese poder al usuario de la aplicación (un rol sin
privilegios de dueño, con solo `INSERT` y `SELECT` sobre `movimiento`) es parte del hito
de endurecimiento: los permisos y el trigger se complementan.

**Corrección a `probar-todo.sh` (2026-10-09).** En una corrida, el corte de enlace falló
apenas arrancó (`fetch failed`). Al terminar, `demo-2pc/ejecutar.sh` recrea el central y
las sucursales 1 y 2 para restaurar la configuración, y salía sin esperar a que
respondieran: el corte de enlace las encontró reiniciando. La corrida anterior había
pasado por suerte de tiempos. Ahora la demostración espera a que las cuatro APIs
respondan antes de salir, y `corte.mjs` espera al stack antes de empezar.

## 2026-10-13 — Informe de indexamiento: cómo se midió

**Decisión.** Los índices nuevos se justifican con mediciones, no con intuición.
`scripts/prueba-indices/ejecutar.sh` carga volumen con `generate_series` (parámetro
`ESCALA`; con 1: 20 000 productos, 400 000 ventas, 1 000 000 de movimientos y 400 000
filas de outbox en la sucursal; 30 sucursales, 600 000 copias de ventas y 20 000
traslados en el central), mide cada consulta con el SQL que de verdad ejecuta el código
(capturado del núcleo compilado, no reescrito a mano), aplica las migraciones nuevas y
vuelve a medir. Por consulta: 3 corridas de calentamiento, 15 medidas, mediana y p95, y
el plan de `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)`. El resultado queda en
`docs/resultados-indices.md`.

**Bases aparte.** La carga va en `sima_indices_sucursal` (contenedor `db-sucursal-1`) y
`sima_indices_central` (contenedor `db-central`), que se borran al terminar con
`DROP DATABASE … WITH (FORCE)` (`CONSERVAR=1` las deja). Se descartó cargar volumen en
`sima_sucursal_N` y `sima_central`: `movimiento` es inmutable desde 008/009 y la carga no
se podría limpiar sin desactivar el trigger.

**Regla aplicada.** Si la medición no muestra mejora, el índice no se agrega. Pasó con
uno de los seis candidatos (ver la entrada del candidato descartado).

**Corrida citada.** Todas las cifras de estas entradas salen de la corrida del informe
actual, `20261013-230455` (`docs/resultados-indices.md`). Las demás entradas la llaman "la
corrida del informe".

**Qué se sostiene entre corridas.** Las conclusiones se apoyan en el plan (`Seq Scan` →
`Index Scan`), los bloques leídos y las filas descartadas por filtro, que no cambian entre
corridas sobre la misma carga. Los tiempos sí cambian: H2 antes de su índice dio 16.04 ms
de ejecución en el servidor en una corrida y 40.16 ms en otra (tiempo de cliente, 17.39 y
25.15 ms), con el mismo `Seq Scan movimiento` y los mismos 9 346 bloques en las dos. Por
eso los milisegundos se dan como "en esa corrida".

## 2026-10-13 — Índice `idx_movimiento_documento` (sucursal, migración 009)

**Decisión.** `CREATE INDEX idx_movimiento_documento ON movimiento (id_documento)`.

**Consulta.** `traslados/recepcion.ts`, `registrarRecepcion`: antes de recibir, verifica
que el documento TRASLADO tenga salidas con `SELECT 1 FROM movimiento WHERE id_documento
= $1 LIMIT 1`. Sin índice, cuando la respuesta es "no hay salidas" (traslado anulado),
recorre toda la bitácora.

**Medición** (1 000 000 de movimientos). Plan: `Seq Scan movimiento` → `Index Only Scan
idx_movimiento_documento`; 9 346 → 3 bloques; 999 999 → 0 filas descartadas por filtro. En
la corrida del informe, la ejecución en el servidor pasó de 40.16 a 0.03 ms. Ocupa 17.2 MB.

**Costo.** Lo mantiene cada `INSERT` en `movimiento`, es decir, cada venta. Con el método
del experimento c (tablas de prueba sin FK, rondas intercaladas), lo que suma sobre los
índices de 006 no fue medible en la corrida del informe. Se acepta: la recepción es parte del 2PC y
su tiempo crecería con toda la historia de la sucursal.

**Alternativa descartada.** Guardar en el documento TRASLADO una marca de "salidas
aplicadas" y consultar esa columna. Se descartó porque cambia el modelo y duplica un hecho
que la bitácora ya registra.

## 2026-10-13 — Índice `idx_documento_traslado` (central, migración 010)

**Decisión.** `CREATE INDEX idx_documento_traslado ON documento (id_traslado) WHERE
id_traslado IS NOT NULL`.

**Consulta.** `traslados/coordinador.ts`, `consultarTraslado` (`GET /traslados/:id`):
busca las copias de documentos del traslado (TRASLADO de origen y destino, RECEPCION) por
`id_traslado`. El único índice con `id_traslado` era `uq_documento_traslado_copia`, que
no lo tiene como primera columna; el planificador lo recorría entero.

**Medición.** Plan: `Index Scan uq_documento_traslado_copia` (recorrido completo del
índice) → `Index Scan idx_documento_traslado`; 326 → 3 bloques. No hay filas descartadas
por filtro en ninguna de las dos fases: lo que cambia es cuánto índice se recorre. En la
corrida del informe, 0.38 → 0.02 ms de ejecución en el servidor. Ocupa 0.9 MB.

**Por qué parcial.** Las ventas, que son casi todo `documento`, tienen `id_traslado` nulo;
indexarlas no sirve a ninguna consulta y encarece cada copia que llega por sincronización.

**Alternativa descartada.** Reemplazar `uq_documento_traslado_copia` (central/008) por el
mismo índice único con las columnas en orden `(id_traslado, id_sucursal, tipo)`. Serviría a
esta consulta sin un segundo índice. Se descartó en este hito porque obliga a borrar y
recrear el índice que garantiza una sola copia por participante y por tipo: durante ese
intervalo la garantía no existe, y el índice nuevo cuesta 0.9 MB. Si el equipo prefiere
un solo índice, es una migración aparte.

## 2026-10-13 — Índice `idx_participante_tx_preparada` (central, migración 010)

**Decisión.** `CREATE INDEX idx_participante_tx_preparada ON participante
(id_tx_preparada) WHERE id_tx_preparada IS NOT NULL`.

**Consulta.** `traslados/coordinador.ts`, `consultarDecision`: el reconciliador de cada
sucursal pregunta por cada transacción preparada huérfana qué decidió el coordinador,
identificándola por su gid (`WHERE p.id_tx_preparada = $1`).

**Medición** (40 000 participantes). Plan: `Seq Scan participante` → `Index Scan
idx_participante_tx_preparada`; 500 → 10 bloques; 39 969 → 0 filas descartadas por filtro.
En la corrida del informe, 2.46 → 0.04 ms de ejecución en el servidor. Ocupa 2.3 MB.

**Por qué parcial.** Los participantes que no llegaron a preparar no tienen gid y nunca se
buscan por él.

**Alternativa descartada.** Un `UNIQUE` sobre `id_tx_preparada`. El gid es único por
construcción, pero no se agregó la restricción en este hito: el pedido era de índices, y
cambiar una garantía del modelo merece su propia decisión. Queda como posible mejora.

## 2026-10-13 — Índice `idx_traslado_solicitado` (central, migración 010)

**Decisión.** `CREATE INDEX idx_traslado_solicitado ON traslado (fecha_solicitud) WHERE
estado = 'SOLICITADO'`.

**Consulta.** `traslados/coordinador.ts`, `ejecutarRecuperacion`: al arrancar, cada cierto
intervalo y desde el endpoint de recuperación, busca los traslados en `SOLICITADO`
ordenados por `fecha_solicitud`. Son pocos en cualquier momento, y la tabla crece con toda
la historia de traslados.

**Medición** (20 000 traslados, 15 en `SOLICITADO`). Plan: `Seq Scan traslado` → `Index
Scan idx_traslado_solicitado`; 239 → 16 bloques; 19 985 → 0 filas descartadas por filtro.
En la corrida del informe, 0.92 → 0.02 ms de ejecución en el servidor. Ocupa menos de
0.1 MB, porque solo contiene los pendientes.

**Alternativa descartada.** Un índice completo sobre `(estado, fecha_solicitud)`. Serviría
igual, pero indexaría los miles de traslados terminados que nunca se consultan por estado.

## 2026-10-13 — `consultarTraslado` compara el uuid como uuid

**Decisión.** En `traslados/coordinador.ts`, `consultarTraslado` pasa de `WHERE
t.id_traslado::text = lower($1)` a `WHERE t.id_traslado = $1::uuid`, y antes de consultar
valida en TypeScript que el parámetro tenga forma de uuid. Si no la tiene, responde 404
`TRASLADO_NO_EXISTE`, igual que con un uuid que no existe.

**Por qué.** Al convertir la columna a texto, el índice único `traslado_id_traslado_key`
no se podía usar y cada `GET /traslados/:id` recorría la tabla. Plan: `Seq Scan traslado`
→ `Index Scan traslado_id_traslado_key`; 243 → 7 bloques; 19 999 → 0 filas descartadas por
filtro. En la corrida del informe, 3.46 → 0.02 ms de ejecución en el servidor. No hizo falta
índice nuevo.

**Por qué validar antes.** Con `$1::uuid`, un texto que no es uuid haría fallar el cast en
la base (SQLSTATE `22P02`) y la API respondería 500. La comparación anterior no distinguía
mayúsculas porque usaba `lower`; con el tipo uuid tampoco, porque PostgreSQL acepta el uuid
en mayúsculas.

**Alternativa descartada.** Un índice funcional sobre `(id_traslado::text)`. Se descartó
porque arregla el síntoma y deja una conversión innecesaria.

**Demostración.** `scripts/prueba-coordinador/prueba.mjs`, caso 1: el id en mayúsculas
responde 200, `no-es-un-uuid` responde 404 `TRASLADO_NO_EXISTE` y un uuid válido
inexistente responde 404.

## 2026-10-13 — Candidato descartado: `existencia_consolidada (id_producto, id_sucursal)`

**Hipótesis.** `existencias/consultar-existencias-red.ts`, `consultarExistenciasRed`,
filtra por producto, y la PK de `existencia_consolidada` empieza por `id_sucursal`. Sin
skip scan en PostgreSQL 16, se esperaba que un índice que empiece por `id_producto`
ayudara.

**Medición.** La hipótesis de un recorrido completo no se confirmó. El plan no tiene `Seq
Scan` sobre el consolidado: usa la PK con un Nested Loop sobre `sucursal`, con una búsqueda
exacta `(id_sucursal, id_producto)` por cada una de las 30 sucursales, y no descarta filas
por filtro. Con el candidato, el plan pasa a `Bitmap Heap Scan` y baja de 241 a 154
bloques. Es el único caso del informe en que los bloques bajan sin que haya filas
descartadas que evitar: los dos planes llegan directo a las 30 filas. En la corrida del informe, la
ejecución en el servidor fue de 0.23 ms con la PK y 0.21 ms con el candidato, y el tiempo de
cliente de 0.79 y 1.01 ms: diferencias de centésimas, sin una dirección clara. El candidato
ocuparía 12.9 MB y lo mantendría cada escritura del consumidor sobre el consolidado. Se
midió dentro de una transacción con `ROLLBACK` y **no se agregó** a la migración 010; el
informe lo registra como experimento.

**Cuándo revisarlo.** Con muchas más sucursales el Nested Loop hace una búsqueda por
sucursal; el argumento de las quince sucursales está fuera de alcance de implementación,
pero si se midiera ahí, este es el índice a probar primero.

## 2026-10-13 — B-tree contra hash en `producto.codigo_barras`: recomendación, sin ejecutar

**Medición** (experimento a del informe, base de sucursal, 20 000 productos). Para la
venta (`= ANY` de 5 códigos) y para `GET /existencias/:codigo` (`=` de 1 código), el
B-tree del `UNIQUE` (`producto_codigo_barras_key`) y el hash de 006
(`idx_producto_codigo_hash`) dan el mismo tipo de plan: búsqueda por índice, con 5 a 15
bloques (el hash lee algunos menos: 10 contra 15 con 5 códigos, 5 contra 6 con uno). Sin
índice, `Seq Scan producto` con 223 bloques. En la corrida del informe, la ejecución en el
servidor con cualquiera de los dos fue de 0.02 a 0.03 ms, y sin índice la venta tardó
2.52 ms. El hash ocupa 1.2 MB contra 0.6 MB del B-tree.

**Conclusión.** El hash no aporta una diferencia medible. El B-tree no se puede quitar:
es el que hace cumplir el `UNIQUE`, y un hash no puede ser único ni servir para rangos ni
para ordenar. Mantener los dos cuesta espacio y una escritura más por cada producto que
llega replicado del central.

**Recomendación: quitar `idx_producto_codigo_hash`** (sucursal/006 y central/007). **No se
ejecutó.** Queda escrita hasta que el equipo lo decida; si se aprueba, va en una migración
nueva con `DROP INDEX`, sin editar 006 ni 007. La medición se hizo en la sucursal; el del
central se incluye en la recomendación porque responde a las mismas consultas por código,
pero no se midió aparte.

**Por qué se creó.** El comentario de 006 lo dice: se declaró aparte a propósito para
compararlo con el B-tree en el informe de indexamiento. La comparación ya quedó hecha y
documentada; quitarlo no la borra.

## 2026-10-13 — Índices de 006 sobre `movimiento`: hoy no los usa ningún endpoint

**Hallazgo.** `idx_movimiento_fecha` e `idx_movimiento_suc_prod` (sucursal/006,
central/007) no los usa ninguna consulta del código actual. Se midieron con dos consultas de
referencia escritas para el experimento. El kardex de un producto en 30 días pasa de `Seq
Scan movimiento` (9 346 bloques) a `Index Scan idx_movimiento_suc_prod` (7 bloques); la
bitácora de ayer, de `Seq Scan movimiento` (9 418 bloques) a `Index Scan
idx_movimiento_fecha` (34 bloques). En la corrida del informe, la ejecución en el servidor
bajó de 35.45 a 0.08 ms y de 118.07 a 0.47 ms. Ocupan 65.1 MB con 1 000 000 de movimientos.

**Costo en escritura (medición corregida, 2026-10-13).** La primera medición no servía y el
"+17 %" que se daba con ella se retira. Hacía el `INSERT` sobre `movimiento` real, donde el
costo lo dominan las verificaciones de llaves foráneas por fila, y corría seguidas todas
las corridas de cada variante: la variante con solo la PK salía más lenta que una con un
índice más.

El método nuevo (experimento c del informe) aísla el mantenimiento de índices:
- tres tablas de prueba con `CREATE TABLE … (LIKE movimiento INCLUDING DEFAULTS)`, sin FK,
  sin CHECK y sin trigger (el script lo verifica), cargadas con el mismo millón de filas;
- cada una con solo los índices de su variante: PK; PK + 006; PK + 006 +
  `idx_movimiento_documento`;
- rondas intercaladas (A, B, C, A, B, C…), 3 de calentamiento y 30 medidas por variante;
  cada corrida es `BEGIN` · `INSERT` de 10 000 · `ROLLBACK`;
- una diferencia se da como porcentaje solo si el p95 de la variante más rápida queda por
  debajo de la mediana de la más lenta.

Resultado en la corrida del informe:

| Índices | Mediana (ms) | p95 (ms) | µs por fila | Respecto de "solo la PK" |
|---|---:|---:|---:|---:|
| solo la PK | 35.88 | 56.66 | 3.59 | referencia |
| PK + 006 | 71.42 | 182.24 | 7.14 | +99 % |
| PK + 006 + `idx_movimiento_documento` | 87.40 | 243.74 | 8.74 | +144 % |

- **Los dos índices de 006: el costo es medible.** El p95 de "solo la PK" (56.66 ms) queda
  por debajo de la mediana de "PK + 006" (71.42 ms). Aproximadamente duplican el costo de
  mantener los índices de `movimiento`: unos 3.5 µs más por fila. El porcentaje es el de
  esa corrida; como todo tiempo, cambia entre corridas, y lo que se sostiene es que la
  diferencia es medible y su orden de magnitud.
- **`idx_movimiento_documento` sobre "PK + 006": no es medible con este método.** El p95 de
  "PK + 006" (182.24 ms) pasa la mediana de la variante con los tres índices (87.40 ms).

**Decisión.** Se dejan como están. Quitarlos no es parte de este trabajo y no se decide sin
el equipo. Si un endpoint de kardex o bitácora llega a existir en la base operativa, ya
tiene índice; si esas consultas son analíticas, la regla del proyecto las manda al almacén
analítico y estos índices sobrarían en la base de la sucursal.

## 2026-10-13 — `idx_outbox_pendiente` se confirma; el conteo de `/salud` queda como límite conocido

**Confirmado.** Con 400 000 filas en la outbox y 20 pendientes, el ciclo del publicador
(`FOR UPDATE SKIP LOCKED` sobre las pendientes) usa `Index Scan idx_outbox_pendiente` con 42
bloques y sin filas descartadas. Sin el índice hace `Seq Scan outbox`, con 14 855 bloques y
399 980 filas descartadas por filtro. En la corrida del informe, 0.06 ms contra 56.63 ms de
ejecución en el servidor. Como la outbox no se purga,
sin el índice el publicador, que corre cada 2 s en cada sucursal, se volvería más lento con
cada venta.

**Límite conocido.** `consultarEstadoOutbox`, usado por `/salud`, cuenta las filas por
estado sobre toda la tabla: `Seq Scan outbox`, 14 815 bloques; en la corrida del informe,
mediana de 27.72 ms de tiempo de cliente. Un índice
no evita contar todo, y la pendiente más antigua, que es lo que importa para operar, ya usa
`idx_outbox_pendiente`. No se cambia. Si `/salud` se volviera lenta, las alternativas son
archivar los ENVIADO viejos o informar solo los pendientes.

## 2026-10-13 — Las migraciones de índices no usan `CREATE INDEX CONCURRENTLY`

**Decisión.** `sucursal/009_indices_consultas.sql` y `central/010_indices_consultas.sql`
usan `CREATE INDEX` simple, porque `scripts/migrar.sh` aplica cada migración dentro de una
transacción y `CREATE INDEX CONCURRENTLY` no puede ejecutarse dentro de una.

**En producción** irían con `CREATE INDEX CONCURRENTLY`, fuera de transacción: el
`CREATE INDEX` simple toma un bloqueo que impide escribir en la tabla mientras se
construye, y en una sucursal con un millón de movimientos eso detendría las ventas durante
la construcción de `idx_movimiento_documento`. Cada migración lo anota en un comentario.

**Alternativa descartada.** Que `migrar.sh` aplique ciertas migraciones fuera de
transacción. Se descartó en este hito: complica el script y pierde la garantía de que una
migración se aplica completa o no se aplica, para un problema que en desarrollo no existe.

## 2026-10-13 — Datastream llega a Cloud SQL por emparejamiento VPC y una VM de reenvío

**Problema.** `sima-central` solo tiene IP privada, dentro del rango de acceso privado a
servicios (`10.20.0.0/20`), que es un emparejamiento entre `red-sima` y la red de Google
que administra Cloud SQL. La conexión privada de Datastream es otro emparejamiento, entre
su red y `red-sima`. El emparejamiento no es transitivo: Datastream ve `red-sima`, pero no
lo que hay detrás de su otro emparejamiento.

**Decisión.** Conexión privada de Datastream por emparejamiento VPC con un /29 libre
(`10.30.0.0/29`), más una VM `proxy-datastream` (e2-micro, Debian 12, sin IP pública, IP
interna `10.10.0.250` reservada como estática) que reenvía el puerto 5432 a la IP privada de Cloud SQL con iptables
(DNAT y SNAT). El perfil de origen de Datastream apunta a la VM. El firewall solo deja
entrar al 5432 de la VM desde el /29 (y desde la subred, que ya tenía permiso).

- **El /29.** No se cruza con la subred (`10.10.0.0/24`), con el acceso a servicios
  (`10.20.0.0/20`) ni con los rangos que reserva Datastream (`10.1.0.0/16` y `10.2.2.0/24`).
- **Por qué este camino.** Es el que la documentación de conectividad privada de Datastream
  describe para fuentes administradas que no están en la red emparejada: una VM en la VPC
  con reenvío por iptables.
- **La IP del proxy está reservada** como dirección interna estática (`ip-proxy-datastream`),
  y `08-datastream.sh` la reserva antes de crear la VM. El perfil de origen de Datastream
  apunta a esa IP, así que no puede cambiar ni quedar en manos de otro recurso. Se eligió una
  IP alta de la subred, lejos de las que se asignan primero.
- **La VM no tiene cuenta de servicio.** No llama a ninguna API, solo reenvía paquetes.
- **Se rearma en cada arranque.** `ip_forward` y las reglas de iptables viven en memoria, y
  `encender-apagar.sh` reinicia la VM en cada sesión. Por eso las aplica su script de
  arranque (`infra/gcp/proxy/arranque.sh`).
- **Demostración.** El arranque se probó en un contenedor Debian 12 contra un PostgreSQL 16:
  antes no hay respuesta en el 5432; después, `pg_isready` responde y una consulta real pasa
  con la IP de la VM como cliente. Dos arranques seguidos dejan exactamente las dos reglas.
  En GCP lo comprueba `verificar-gcp.sh` con `pg_isready` desde la sucursal 2, y
  `--reiniciar-proxy` además reinicia la VM y espera a que el reenvío vuelva.

**Corrección (2026-10-14): por qué se reserva la IP.** La primera corrida de `08` en Cloud
Shell falló al crear la VM con `10.10.0.20`: "IP '10.10.0.20' is already being used by
another resource". La había tomado Cloud Run. `api-central` y los jobs (`migrar-central`,
`cdc-central`) salen a la VPC con Direct VPC egress, que **comparte `subred-sima`** y asigna
IPs de ella a medida que crea instancias (había una `serverless-ipv4-cloudrun` en
`10.10.0.16`). Fijar la IP con `--private-network-ip` no la protege: solo funciona si la IP
está libre en ese momento, y Cloud Run puede tomarla antes. Una dirección interna estática
reservada queda fuera de lo que Cloud Run puede asignar.

- **Alternativa descartada:** una subred aparte para Cloud Run. Separaría los rangos, pero
  obliga a cambiar `05-cloud-run.sh` y la red de un despliegue que ya funciona, para
  proteger una sola IP.
- **Las IPs de las sucursales también se reservan (corrección del 2026-10-14).** Las
  sucursales (`10.10.0.11` a `.13`) tenían IP fija pero efímera: si una se borraba y se
  volvía a crear, Cloud Run podía ganarle la IP, y el coordinador del 2PC la busca ahí
  (`SUCURSAL_URL_PLANTILLA`). Ahora `06-sucursales.sh` las reserva como direcciones internas
  estáticas (`ip-sucursal-1` a `-3`).
  - **Con las VMs ya creadas no hay que recrearlas.** `gcloud compute addresses create …
    --subnet --addresses=<la IP en uso>` es la forma documentada de promover a estática una
    IP interna efímera que ya usa un recurso.
  - **Mismos controles que en el proxy:** la reserva tiene la IP esperada, la VM tiene esa IP
    y la reserva queda asociada a la VM (`users`). Si algo no cuadra, el script se corta.

**Alternativas descartadas.**

- **Interfaces de Private Service Connect.** Datastream las soporta y la documentación de
  VPC dice que una interfaz PSC es transitiva. Pero la página de Datastream sobre interfaces
  PSC no dice en qué estado de lanzamiento están ni que lleguen al rango de acceso privado
  de Cloud SQL, y no hay una guía de Datastream con Cloud SQL por ese camino. No se apostó la
  entrega a algo sin documentar. Además, el adjunto de red no se puede cambiar después de
  crear la conexión.
- **Cloud SQL Auth Proxy en la VM.** Funcionaría igual, pero necesita una cuenta de servicio
  con `cloudsql.client` en la VM. El reenvío por iptables es el camino documentado y no
  necesita credenciales.
- **IP pública con lista de IPs permitidas.** La instancia es solo privada a propósito (M2).

**Límites.**

- La VM es un punto único de falla para el CDC. Si cae, Datastream reintenta y el slot
  retiene el WAL hasta que vuelva: se atrasan los tableros, nunca la operación. La alta
  disponibilidad documentada (dos VMs detrás de un balanceador interno) queda fuera de alcance.
- El tramo Datastream → VM → Cloud SQL va sin TLS en el protocolo de PostgreSQL. No sale de
  la red privada; exigir TLS es parte del hito de endurecimiento.

## 2026-10-13 — La preparación de la base para Datastream no es una migración

**Decisión.** El usuario de replicación, la publicación y el slot lógico se crean con
`infra/gcp/migraciones/replicacion.sh`, que corre el job `cdc-central` de Cloud Run dentro
de la VPC. No van en `migraciones/central`.

**Por qué.**

- **No es esquema: es la integración de una plataforma con la base.** El slot y la
  publicación solo existen porque en GCP hay un Datastream que los lee.
- **En Compose no va en las migraciones.** Un slot sin consumidor retiene WAL para siempre,
  y en Compose de `main` no hay ningún Datastream que lo lea. *Corregido el 2026-10-28:* esta
  nota decía además que la base de Compose no corre con `wal_level=logical`. Ya no es cierto:
  `db-central` corre con `wal_level=logical` desde esa fecha (ver la entrada de la variante
  autoalojada). Sin ningún slot creado eso no retiene nada, así que el motivo de fondo sigue
  igual: el slot y la publicación no son esquema.
- **El job es aparte de `migrar-central`.** `05-cloud-run.sh` redefine los secretos de
  `migrar-central` con `--set-secrets`, y en la siguiente corrida borraría el de replicación.
  `cdc-central` usa la misma imagen, con otro comando y su propia cuenta de servicio
  (`sa-cdc`), que solo lee los dos secretos que necesita.

**Privilegio mínimo.**

- `sima_replicacion` tiene `LOGIN` y `REPLICATION` y solo `SELECT` sobre las 10 tablas
  replicadas. No es superusuario ni miembro de `cloudsqlsuperuser`.
- Su contraseña se genera en `08-datastream.sh` y vive solo en Secret Manager.
- El slot lo crea el propio `sima_replicacion`, que es quien tiene el atributo REPLICATION;
  la publicación la crea `sima`, dueño de las tablas.
- `ALTER PUBLICATION … SET TABLE` deja la publicación exactamente igual a la lista de
  `variables.sh`, así que el script se puede correr las veces que haga falta.

**Demostración.** El script se corrió contra un PostgreSQL 16 temporal con
`wal_level=logical` y las 10 migraciones del central:

- Dos corridas de `preparar` dan el mismo resultado: 10 tablas y un slot `pgoutput`.
- `sima_replicacion` lee `movimiento` y recibe permiso denegado al escribir en `sucursal`
  y al leer `usuario`, que no se replica.
- `pg_logical_slot_peek_binary_changes` con la publicación entrega un insert nuevo.

En Cloud SQL queda una diferencia que no se pudo probar fuera de la plataforma: ahí `sima`
es miembro de `cloudsqlsuperuser`, no superusuario. La documentación de replicación lógica
de Cloud SQL muestra que esos usuarios pueden dar `REPLICATION`. Si fallara, el job lo
dice y el remedio es correr el `CREATE ROLE` como `postgres`.

## 2026-10-13 — Grano del hecho y orden de los movimientos

**Decisión.** `hecho_movimiento`:

- **Grano:** un renglón de `movimiento`, la bitácora inmutable de existencias.
- **Llave:** `(origen, id_movimiento)`.
- **Partición** por `fecha` (en hora de Guatemala) y **agrupación** por `id_sucursal`,
  `id_producto`, como pide la propuesta para no pagar exploraciones completas.
- **Cantidades.** `cantidad` siempre es positiva, como en la base, y la dirección la da
  `tipo`. Se agrega `delta = cantidad_despues − cantidad_antes`, negativa en las salidas.
- **Del documento y la línea** se toman `tipo_documento`, `id_proveedor` y el precio, que
  sale del subtotal de la línea por (documento, producto).
- **Se llena con una consulta programada.** Es una tabla, no una vista: así se puede
  particionar y agrupar.

**Fecha en hora de Guatemala.** `fecha_hora` está en UTC. Con la fecha en UTC, una venta
de las 19:00 del lunes en Xela caería en martes, y "ventas de ayer" no sería el ayer de la
tienda.

**Orden de "el último" movimiento.** Todo lo que depende del último movimiento (el cierre
diario, la existencia actual de `SIMULADO`) ordena por `fecha_hora` e
`id_movimiento_origen` dentro de cada `id_sucursal`. Nunca por el `id_movimiento` del
central:

- El `id_movimiento` del central es el **orden de llegada**. Una sucursal que estuvo sin
  enlace sincroniza tarde, y sus movimientos reciben ids del central mayores que los de otras
  que vendieron después.
- `fecha_hora` es el momento en que ocurrió en la sucursal, e `id_movimiento_origen` es el
  orden dentro de ella. Juntos dan el **orden de ocurrencia**.
- Para la existencia de un producto en una sucursal, eso es lo que cuenta.

**Dimensiones.**

- `dim_producto`, `dim_sucursal` y `dim_proveedor` son vistas tipo 1 sobre `sima_cdc`:
  siempre el valor vigente. El catálogo es chico; materializarlas no ahorra nada y obligaría
  a refrescarlas.
- `dim_fecha` es una tabla generada, de 2025 a 2028, con nombres en español y la semana
  empezando en lunes.

**Rotación.**

- **Denominador:** el promedio de las existencias de cierre diarias de la ventana (30 o 90
  días). El cierre de un día es `cantidad_despues` de su último movimiento; los días sin
  movimiento arrastran el cierre anterior sobre `dim_fecha`.
- **Antes del primer movimiento** conocido, el cierre es el `cantidad_antes` de ese primero.
  Un par sin ningún movimiento vale su existencia actual.
- **Al lado:** venta diaria (unidades ÷ días de la ventana) y días de inventario
  (existencia actual ÷ venta diaria).
- **Alternativa descartada:** la existencia promedio de dos puntos, (inicio + fin) ÷ 2. No
  ve lo que pasa en medio de la ventana: un producto que se agota y se repone a mitad de mes
  daría el mismo promedio que uno que nunca se movió.

**Límites.**

- Con `CDC`, las ventanas son fijas. Si el sistema tiene menos historia que la ventana, la
  venta diaria sale subestimada.
- Las vistas con fecha de referencia por origen (hoy para `CDC`, el fin de la historia para
  `SIMULADO`) no aprovechan la poda de particiones, porque el límite no es una constante. Con
  este volumen no importa. A escala, el tablero filtraría `v_movimientos` por fecha, que sí
  poda.

## 2026-10-13 — La consulta programada hace un MERGE completo, sin marca de agua

**Decisión.** Cada hora, `05_merge_hecho.sql` hace un `MERGE` de toda la fuente (CDC más la
historia simulada) contra `hecho_movimiento`, por la llave `(origen, id_movimiento)`:

- inserta lo nuevo;
- corrige lo que cambió (por ejemplo, un movimiento que llegó antes que su documento y se
  insertó sin `tipo_documento`);
- borra lo que ya no está en la fuente (la historia simulada regenerada).

**Alternativa descartada.** Un MERGE incremental con una ventana sobre
`datastream_metadata.source_timestamp` (por ejemplo, lo del último día). Con el stream
pausado varios días (un fin de semana con todo apagado), los cambios que llegan al reanudar
pueden caer fuera de la ventana y perderse sin aviso. El MERGE completo no puede perder filas.

**Costo.** Con el volumen del proyecto, el MERGE completo cuesta prácticamente nada: lee
unos pocos MB, y BigQuery cobra un mínimo de 10 MB por tabla referenciada. Son unos 36 GB
al mes, dentro del TiB gratuito.

**A escala real** iría incremental, con una **marca de agua de llegada**: el momento en que
cada fila entró a BigQuery, no el de la base. El MERGE procesaría lo que llegó desde la
última marca guardada, con un traslape, y el `MERGE` por llave absorbería las filas
repetidas. Así no se pierde nada tras una pausa, porque lo que llega tarde llega después de
la marca. En modo merge, Datastream no expone esa hora de llegada (`source_timestamp` es la
de la base), así que haría falta el modo append-only de Datastream, que conserva cada cambio
como fila, o una tabla de aterrizaje con su propia columna de ingreso.

## 2026-10-13 — Historia sintética para los tableros, con origen SIMULADO

**Decisión.** `sima_analitica.historia_simulada` tiene seis meses de movimientos de las 3
sucursales y los 50 productos de la semilla. Se genera dentro de BigQuery con SQL
(`infra/gcp/bigquery/04_historia_simulada.sql`). `hecho_movimiento` la une con lo que llega
por CDC y la marca con `origen = 'SIMULADO'`; todas las vistas llevan `origen` y los tableros
filtran por él. **Nunca se escribe en Cloud SQL ni en las sucursales.**

**Por qué.** Los tableros de rotación, capital dormido y reposición necesitan meses de
historia, y SIMA en GCP solo tiene las ventas de las pruebas. Generar esa historia en la base
operativa la contaminaría: `movimiento` es inmutable y no se podría limpiar. Además rompería
la regla de que lo analítico no toca lo transaccional.

**Modelo de la historia.**

- **Determinista.** El azar sale de `FARM_FINGERPRINT` de una clave, así que con la misma
  fecha de fin la historia es la misma. `09-bigquery.sh` no la regenera si ya existe.
- **Demanda.** Diaria, sesgada hacia productos lentos, con más venta el fin de semana.
- **Reposición.** Cada lunes entra lo vendido la semana anterior. La existencia vuelve a su
  nivel inicial (la mayor venta semanal más `stock_min`) y nunca queda negativa.
  `verificar-gcp.sh` comprueba que no haya negativos y que cada movimiento parta del
  `cantidad_despues` del anterior.
- **Capital dormido.** Un 12 % de los pares deja de venderse los últimos 75 días.
- **Fecha de referencia propia.** La historia termina el día anterior a su generación, y
  "los últimos 30 días" de `SIMULADO` se cuentan desde su último día. Si no, semanas después
  todo lo simulado parecería dormido.

**La recepción de proveedor no está implementada en el flujo operativo.** En SIMA, ningún
documento real lleva `id_proveedor`: ventas y ajustes lo dejan en nulo, y el único documento
`RECEPCION` es el del traslado. Para que la historia tenga reposiciones con proveedor, y la
dimensión de proveedor tenga datos, la historia usa documentos `RECEPCION` con proveedor y
el tipo de movimiento **`RECEPCION_ENTRADA`**, que sigue la convención de los demás
(`AJUSTE_ENTRADA`, `TRASLADO_ENTRADA`). **Ese tipo solo existe en `SIMULADO`.** Con datos
`CDC`, todo movimiento queda "Sin proveedor". Si la recepción de proveedor se implementa, ese
es el tipo que debería usar, y el hecho ya está listo para recibirlo.

## 2026-10-13 — El capital dormido se valoriza a precio de venta (límite)

**Decisión.** `v_capital_dormido` valoriza la existencia sin ventas como
`existencia × producto.precio_venta`.

**Límite.** El modelo no tiene costo unitario: ni `producto` ni `linea_documento` guardan lo
que costó la mercadería, porque no hay recepción de proveedor con precio de compra. Valorizar
a precio de venta **sobrestima** el capital inmovilizado en el margen de cada producto. Es el
valor de venta detenido, no la inversión. La columna se llama `valor_a_precio_venta` para
que no se lea como costo, y `docs/tableros.md` pide decirlo en el tablero.

**Alternativa descartada.** Inventar un costo (por ejemplo, un porcentaje fijo del precio).
Daría un número con apariencia de exacto, sin ningún dato detrás.

## 2026-10-13 — Salud de la sincronización: lo que el central no ve (límite)

**Decisión.** `v_salud_sincronizacion` muestra, por sucursal, el último mensaje que el
central procesó (`inbox_idempotencia.procesado_en`), los mensajes de las últimas 24 horas y
los traslados no finalizados en los que participa. También muestra el cambio más reciente
que Datastream ya dejó en BigQuery, que es la antigüedad del dato del tablero.

**Límite.** Las operaciones pendientes en la outbox de cada sucursal no llegan al central:
viven en la base de la sucursal, y el plano analítico no la lee (propuesta, 4.5). El tablero
dice cuándo habló cada sucursal por última vez, no cuánto tiene encolado. Ese número lo da
`/salud` de cada sucursal. Llevarlo a BigQuery requeriría que cada sucursal publicara su
estado al central: es funcionalidad nueva y no está pedida.

## 2026-10-15 — Presupuesto y alertas en GCP (10-presupuesto.sh, 11-monitoreo.sh)

**Presupuesto.** `10-presupuesto.sh` es el dueño del presupuesto "SIMA dev"; el bloque que lo
creaba en `01-proyecto.sh` se quitó. Si el presupuesto ya existe, se actualiza en el lugar.

- **Periodo personalizado, no mensual.** El crédito de prueba es un total de USD 300 que vence
  el 2027-01-10. Un presupuesto mensual de 300 no avisaría nunca, y uno mensual de 100
  avisaría por un gasto que el crédito todavía cubre. El periodo va de `PRESUPUESTO_INICIO`
  (por defecto 2026-10-01) a `PRESUPUESTO_FIN` (2027-01-10), con USD 300.
- **Umbrales al 25, 50, 80 y 100 % del gasto real.** El 25 % es el primer aviso temprano; con
  el 80 % hay que apagar lo que no haga falta.
- **Sin umbral por gasto proyectado.** La API de presupuestos solo lo admite con periodos de
  calendario (mes, trimestre o año), no con un periodo personalizado. Se pierde el aviso
  anticipado por tendencia; los cuatro umbrales de gasto real lo compensan en parte.
- **Excluye los créditos,** como ya hacía `01`. Si los incluyera, el costo neto con el crédito
  de prueba sería 0 y ningún umbral se alcanzaría.
- **Avisos.** Van a los destinatarios por defecto de la cuenta de facturación (administradores
  y usuarios) y también al canal de correo de las alertas. Crear o actualizar un presupuesto no
  toca la cuenta de facturación ni la pasa a pago.
- **El canal se crea en un archivo común.** El presupuesto necesita que el canal exista. En
  vez de imponer un orden entre `10` y `11`, el canal se crea en `canal-alertas.sh`, que
  cargan los dos: el primero que corre lo crea.
- **El correo no está en el repositorio.** `ALERTA_CORREO` va vacío en `variables.sh` y se pasa
  al correr `10` y `11`, que se cortan con un mensaje si falta.

**Qué se monitorea (11-monitoreo.sh) y por qué esos umbrales.** Las cinco políticas avisan
al canal de correo y llevan la etiqueta `sima=alertas`. Sus definiciones están en
`infra/gcp/monitoreo/*.json`.

- **`api-central` no responde en `/salud`.** Verificación de disponibilidad HTTPS cada 5 min,
  que exige 2xx y el texto `"estado":"OK"`. Es el endpoint que ya existe; no se agregó otro.
  Como responde 503 si la base no contesta, también detecta una Cloud SQL caída. Avisa si
  falla desde más de una región, para no confundir un problema de red de una región con una
  caída.
- **Cloud SQL, CPU > 80 % durante 10 min.** `db-g1-small` es de núcleo compartido: un pico
  corto (backfill, prueba de carga) es normal; diez minutos seguidos ya no.
- **Cloud SQL, disco > 80 %.** El disco es de 10 GB. En SIMA, lo que puede llenarlo es el WAL
  retenido por el slot de Datastream con el stream pausado y la base encendida.
- **Errores de `api-central`, más de 5 en 10 min.** La métrica `sima_api_central_errores`
  cuenta los registros con severidad ERROR o mayor **o escritos en stderr**. Nest escribe
  `Logger.error` como texto plano en stderr, y Cloud Run no le asigna severidad a esas
  líneas: con solo `severity >= ERROR`, la alerta no vería los errores de la aplicación
  ("Recuperación fallida", "Error transitorio"). Costo: una traza de varias líneas puede
  contar más de una vez. Un error suelto (un 503 mientras Cloud SQL arranca) no debería
  despertar a nadie; más de 5 en 10 minutos ya es un patrón.
  - **Alternativa descartada:** que la aplicación escriba logs JSON con `severity`. Sería lo
    correcto a escala, pero cambia el código de la API, y el alcance de esta tarea era solo
    `infra/gcp` y `docs`.
- **Traslado 2PC sin resolver, al menos 1 en 10 min.** El coordinador no loguea ningún
  "EN_DUDA". La métrica `sima_2pc_sin_resolver` cuenta los dos mensajes que corresponden:
  - `Traslado <id>: <accion> en <rol> (sucursal N) sin acuse: …`
    (`packages/nucleo/src/traslados/coordinador.ts`, en `propagarDecision`). La decisión no
    llegó a un participante, que queda con la transacción preparada en duda. La recuperación
    lo reintenta cada pocos segundos, así que el mensaje se repite mientras siga sin resolver.
  - `Recuperación fallida: …` (`packages/api-central/src/traslados/recuperacion.service.ts`).
    La pasada de recuperación misma falló.

  Avisa con uno solo, porque un traslado en duda deja mercadería bloqueada. Queda fuera
  `… sigue SOLICITADO: sucursal N fuera de línea`, que es la espera normal del protocolo.

**Límite: con GCP apagado no hay monitoreo activo.** `encender-apagar.sh apagar` deshabilita
las cinco políticas antes de apagar nada; si no, la verificación de `/salud` y las de Cloud
SQL avisarían solo por estar apagado. `encender` las habilita al final, con todo arriba.
Mientras tanto la verificación sigue corriendo pero no avisa a nadie, y lo único que vigila es
el presupuesto. Una caída real con GCP apagado no se detecta, porque no hay nada que caer. Una
política que quedó deshabilitada por error se ve en `verificar-gcp.sh --solo-alertas`.

**Lo que no se pudo probar fuera de GCP.** Los scripts se probaron con `gcloud` falsos que
registran cada llamada:

- `10` desde cero y sobre un presupuesto existente;
- `11` desde cero, con una métrica de logs que Monitoring todavía no ve (reintenta) y con un
  error distinto (se corta);
- la segunda corrida de cada uno, que no crea nada;
- `encender-apagar.sh`, que deshabilita y habilita las cinco políticas;
- la sección 8 de `verificar-gcp.sh`.

Que Cloud Run no asigne severidad a las líneas de stderr, y que `budgets update
--start-date` pase un presupuesto mensual a periodo personalizado, se confirman en la primera
corrida en Cloud Shell. La sección 8 de `verificar-gcp.sh` revisa lo segundo.

**Correcciones de la primera corrida en Cloud Shell (2026-10-15).** `10`, `11` y
`encender-apagar.sh` funcionaron contra GCP real (`verificar-gcp.sh --solo-alertas`, 14 de
14), pero aparecieron tres errores:

- **Umbrales del presupuesto al actualizar.** Hubo dos fallas.
  - `--clear-threshold-rules` en una llamada aparte no borró los umbrales viejos.
  - `--add-threshold-rule=percent=25` quedó guardado como 2500 %: en `update` el porcentaje
    también va de 0 a 1, como en `create`. La referencia de gcloud decía lo contrario.

  Lo que funcionó contra la API real es una sola llamada con `--clear-threshold-rules` y los
  cuatro `--add-threshold-rule=percent=0.25 … 1.0`.
- **Filtros que gcloud ignora.** En `channels list`, `uptime list-configs` y `policies list`,
  gcloud avisaba "The following filter keys were not present in any resource: displayName" (y
  `type` en el canal) y devolvía todo. Con `head -1`, una segunda corrida de `11` podía hacer
  que las cinco plantillas actualizaran la misma política. Ahora todas las búsquedas usan
  `--format=json` y `jq` con `select(.displayName == …)`. También la etiqueta
  `sima=alertas` en `encender-apagar.sh` y `verificar-gcp.sh`: que antes contara 5 no probaba
  que el filtro funcionara, porque solo existían esas cinco políticas.
  `verificar-gcp.sh --solo-alertas` ahora también cuenta las políticas etiquetadas y las
  compara con las plantillas de `infra/gcp/monitoreo/`.
- **La reanudación del stream al encender.** Falló una vez con `VALIDATION_FAILURE` /
  `POSTGRES_CONNECTION_ERROR` ("Connection … refused"): la VM proxy recién arrancada todavía
  no había corrido su script de arranque. Ahora `encender` reintenta la reanudación cada 30 s,
  hasta 8 veces (unos 4 minutos). Las alertas se habilitan al final pase lo que pase, y si el
  stream no arranca, el script lo dice, indica qué revisar y sale con error.
  - **Alternativa descartada:** esperar la línea `proxy-datastream: …` en la salida serie de
    la VM. Esa salida conserva las líneas de arranques anteriores, así que la línea podría
    ser vieja. Reintentar la reanudación prueba exactamente lo que importa: que Datastream
    llegue a la base.

## 2026-10-16 — Terraform adopta la infraestructura de GCP (fase 1: red)

**Decisión.** La infraestructura de `<id-proyecto>` pasa a describirse en
`infra/terraform/`, conforme a la propuesta. Como ya existe y tiene datos (Cloud SQL, el slot
y la publicación de Datastream, los datasets de BigQuery), Terraform la **adopta** con bloques
`import` en vez de recrearla. Los scripts `01`–`11` de `infra/gcp` quedan como referencia
histórica; `encender-apagar.sh` y `verificar-gcp.sh` siguen siendo los scripts de operación.
Se adopta por fases, de lo que no tiene dependencias a lo más delicado.

**Terraform en Cloud Shell.** Las notas de versión de Cloud Shell dicen que desde el 20 de
junio de 2026 ya no incluye la CLI de Terraform. `instalar-terraform.sh` instala Terraform en
`~/bin`, que se conserva entre sesiones. La versión es 1.16.5 y el script verifica la suma
SHA-256 publicada por HashiCorp.

**Versiones.** Se eligieron versiones con al menos dos semanas de publicadas:
- Terraform 1.16.5, publicada el 2026-10-02 (la más nueva era 1.16.6, del 2026-10-14).
- Los proveedores `google` y `google-beta` en `8.5.0`, publicados el 2026-09-29 (el más nuevo
  era 8.7.0, del 2026-10-13).

`required_version = "~> 1.16.5"`, y los proveedores fijados con `=`. `.terraform.lock.hcl`
se versiona con las sumas para Linux (Cloud Shell) y macOS.

**Estado remoto.** Va en `gs://<id-proyecto>-tfstate`, en `us-central1`, con versionado,
acceso uniforme y sin acceso público. El backend de GCS bloquea el estado, así que dos `apply`
a la vez no se pisan. El bucket lo crea `arranque-estado.sh` y queda fuera de Terraform: el
estado no puede vivir en un recurso que él mismo administra. En el estado van a quedar la
dirección del canal de correo y el texto de las consultas, nunca los valores de los secretos.

**`prevent_destroy` y no `deletion_policy`.** El proveedor 8.x tiene en casi todos los
recursos un campo `deletion_policy` que puede impedir el borrado. Pero ese campo existe solo en
Terraform: al importar, el estado queda con `DELETE`, y ponerlo en `PREVENT` aparecería como
cambio en cada recurso adoptado. El primer plan nunca saldría "No changes". La protección se da
con `lifecycle { prevent_destroy = true }`, que no produce diferencias en el plan. Va en la
red, la subred, el rango y el emparejamiento de acceso privado y las IPs reservadas.

**Sin la etiqueta de atribución.** Desde la versión 5, el proveedor agrega
`goog-terraform-provisioned=true` a todo lo que tiene etiquetas. En la adopción eso sería un
cambio en cada recurso, así que se desactiva con `add_terraform_attribution_label = false`.

**Las descripciones de las IPs se copian exactas.** En `google_compute_address` la descripción
no se puede cambiar sin reemplazar la dirección. Las de `ip-sucursal-N` e
`ip-proxy-datastream` son las que pusieron `06` y `08`, comprobadas en las versiones de los
scripts que se corrieron.

**Adopción con GCP apagado.** Importar solo lee la configuración. El encendido de Cloud SQL,
de las VMs y del stream va en `ignore_changes`, así que ninguna fase necesita encender. Si una
lectura falla por estar apagado, se resuelve en esa fase; no se enciende para importar.

**Cómo se revisa cada fase.** `revisar-plan.sh` guarda el plan en `plan.bin` y lo clasifica
con su JSON:
- **Salida 0:** solo importaciones. Se aplica exactamente ese plan con
  `terraform apply plan.bin`.
- **Salida 2:** hay cambios en el lugar, listados con sus atributos. Se corrigen en la
  configuración o se justifican antes de aplicar.
- **Salida 1:** algo se crea, se destruye o se reemplaza. No se aplica.

El script se probó con planes de ejemplo para los cuatro casos. La configuración pasó
`terraform fmt` y `terraform validate` con los proveedores 8.5.0. El plan real contra GCP solo
se puede correr en Cloud Shell.

**Al terminar.** Con todas las fases adoptadas y `terraform plan` en "No changes",
`importaciones.tf` se borra. La configuración queda describiendo la infraestructura completa,
y el README lo va a explicar.

## 2026-10-17 — Terraform: fase 1 adoptada; fases 2 y 3 escritas

**Fase 1 adoptada.** En Cloud Shell se importaron los 13 recursos de red. Después,
`terraform plan -detailed-exitcode` devolvió 0 ("No changes").

**Las IPs de las sucursales no estaban reservadas.** Al importar, `ip-sucursal-1` a `-3` no
existían: el cambio de `06-sucursales.sh` que las reserva (la corrección del 2026-10-14) nunca
se había corrido en GCP. Se reservaron a mano con el mismo comando de `06`:
`gcloud compute addresses create ip-sucursal-N --region=us-central1 --subnet=subred-sima
--addresses=10.10.0.1N` y la misma descripción. Ese comando promovió a estáticas las IPs
`.11`, `.12` y `.13` que ya usaban las VMs, sin tocarlas, y después se importaron. Queda
claro que adoptar con Terraform también sirve para descubrir que un script versionado nunca
se aplicó.

**Credenciales: token de gcloud, no ADC.** En esta cuenta, ni "Autorizar" de Cloud Shell ni
`gcloud auth application-default login` sirvieron: el login de ADC tomó otra cuenta, sin
acceso al proyecto. Lo que funcionó fue
`export GOOGLE_OAUTH_ACCESS_TOKEN="$(gcloud auth print-access-token)"`, que leen el proveedor
y el backend de GCS. Dura alrededor de una hora.
- `credenciales.sh` lo hace solo. Si la variable está vacía y no hay ADC de la misma cuenta
  que `gcloud`, toma el token de `gcloud` y lo avisa. Para saber de qué cuenta es ADC, el
  token se compara con el endpoint `tokeninfo` de Google.
- `revisar-plan.sh` lo carga por su cuenta.
- Sigue sin haber cuentas de servicio con llaves.

**`revisar-plan.sh` guarda stdout y stderr.** Antes, cuando el plan fallaba, `plan.log` solo
tenía stdout y el motivo del error se perdía. Ahora guarda las dos salidas. Si lo que frenó el
plan es `prevent_destroy`, dice qué recurso protegido iba a destruirse o reemplazarse y
muestra la línea marcada `# forces replacement`. Se comprobó con Terraform 1.16.5 que en ese
caso el plan se imprime antes del error, así que esa línea está en la salida.

**Fase 2: solo los permisos que dieron los scripts.** Se revisaron `03`–`09` y se importan
exactamente las 9 asignaciones que hicieron:
- **`logging.logWriter`** en el proyecto para `sa-api-central`, `sa-sucursal` y `sa-cdc`;
- **`monitoring.metricWriter`** para `sa-sucursal`;
- **`bigquery.jobUser`** para `sa-analitica`;
- **lectura de `sima-bd-url-central`** para `sa-api-central` y `sa-cdc`;
- **lectura de `sima-db-replicacion-password`** para `sa-cdc`;
- **`iam.serviceAccountTokenCreator`** sobre `sa-analitica` para el agente de Data Transfer.

A eso se suman el lector del repositorio de imágenes (`sa-sucursal`) y el invoker público de
`api-central` (`allUsers`), que van en la fase 3. Todo es `*_iam_member`, que no es
autoritativo: los permisos que GCP pone por defecto (agentes de servicio, cuenta de Compute,
Cloud Build) quedan fuera de Terraform y no se tocan. Los permisos de los datasets de `09` van
con BigQuery, en la fase 7.

**Fase 2: campos que existen solo en Terraform.** Ponerlos al importar aparecería como cambio
en el primer plan. Se resolvió así:
- **`disable_on_destroy` de las APIs:** en el proveedor 8.5.0 no tiene valor por defecto, y
  sin declararlo vale `false`, así que un destroy no deshabilita la API. No se declara.
- **`deletion_protection` de los secretos:** se deja como está, y la protección la da
  `prevent_destroy`, porque borrar un secreto pierde la contraseña.

**Fase 3: Cloud Run.**
- **Imagen.** La sigue publicando Cloud Build. Terraform ignora la imagen del servicio y de
  los jobs, y también `client`, `client_version`, las anotaciones y etiquetas de la plantilla y
  el nombre de la revisión, que deja `gcloud run deploy`.
- **`deletion_protection` de Cloud Run** vale `true` por defecto en el proveedor y no se
  declara: impide que Terraform borre el servicio o un job.
- **Lo que hay que escribir aunque coincida con lo de gcloud.** Según el esquema del
  proveedor 8.5.0:
  - `cpu_idle` y `startup_cpu_boost` no se completan solos: si no se declaran y GCP tiene otro
    valor, el plan los cambia. Se escribieron los valores por defecto de `gcloud run deploy`
    (CPU solo durante las solicitudes y, confirmado en el plan, con aumento de CPU al arrancar:
    `startup_cpu_boost = true`). Si el plan muestra otra cosa,
    se ajustan a lo real.
  - `max_retries` de los jobs vale 3 por defecto en el proveedor y los scripts lo dejaron en
    0, así que se declara.
  - Las variables de entorno son un conjunto: el orden no genera diferencias.

## 2026-10-18 — Terraform: fases 4 (Cloud SQL) y 5 (VMs) escritas desde el borrador

**Los valores salen de GCP, no de memoria.** Se corrió `terraform plan
-generate-config-out` sobre los 7 recursos de estas fases. Con ese borrador se escribieron
`sql.tf` y `vms.tf`. No se copió tal cual:
- Se quitó lo que GCP completa solo: versión de mantenimiento, zona elegida, caché de datos,
  IOPS, funciones del sistema operativo invitado.
- Se resolvieron los tres errores de validación del borrador:
  - `server_certificate_rotation_mode` venía `UNSPECIFIED`: no se declara.
  - `read_pool_auto_scale_config` venía en cero: el bloque se quitó, porque no se usan read
    pools.
  - `vlan = 0` en las VMs: no se declara.

**Para decidir qué declarar se miró el esquema del proveedor 8.5.0.** Tres campos que,
sin declararlos, harían que el plan cambiara GCP:
- **`activation_policy`** vale `ALWAYS` por defecto en el proveedor. Sin `ignore_changes`, un
  plan con GCP apagado **encendería la base**. Va en `ignore_changes`, porque la maneja
  `encender-apagar.sh`.
- **`ipv4_enabled`** vale `true` por defecto. Se declara `false`: la instancia es solo
  privada.
- **`enable_dataplex_integration`** está en `true` en la instancia aunque `03` nunca lo pidió:
  lo puso GCP al crearla. El proveedor no lo completa solo, así que se declara `true`.

En las VMs pasa lo mismo con `enable_secure_boot`: vale `false` por defecto y está en `true`,
así que se declara.

**Otras decisiones de estas dos fases:**
- **`disk_size` de Cloud SQL va también en `ignore_changes`.** Con `disk_autoresize`, GCP
  puede agrandar el disco, y Terraform no debe intentar volver a 10 GB.
- **Protección contra borrado de GCP.** `settings.deletion_protection_enabled` está en
  `false` y se copia así: encenderla sería un cambio en el lugar en la instancia. El
  `deletion_protection` de Terraform queda en su valor por defecto, `true`, sin declararlo.
- **Usuario `sima` sin contraseña en Terraform.** La contraseña vive solo en Secret Manager.
  Va con `ignore_changes` en `password`.
- **Ventana de mantenimiento.** La instancia no tiene una configurada; GCP elige.
- **Script de arranque.** Se comprobó que el `startup-script` de las cuatro VMs en GCP es
  idéntico, byte a byte, a `infra/gcp/sucursal/arranque.sh` y `infra/gcp/proxy/arranque.sh`.
  Por eso se lee con `file()` de esos archivos en vez de duplicarlo. Va en
  `metadata["startup-script"]`, no en `metadata_startup_script`, que reemplazaría la VM al
  cambiar el script.
- **Metadata que sale de otros recursos.** El `central-url` de las sucursales sale de
  `google_cloud_run_v2_service.api_central.uri`, y el `ip-sql` del proxy de
  `google_sql_database_instance.central.private_ip_address`, en vez de copiar los valores.
- **`prevent_destroy` en la instancia, en la base y en las cuatro VMs.**
  `revisar-plan.sh` bloquea además cualquier cambio en el lugar en la instancia o en las VMs.

**Adopción de las fases 4 y 5 (2026-10-19).** Se importaron los 7 recursos y el estado tiene
58.
- **`deletion_policy` de la base y del usuario de SQL.** En `google_sql_database` y
  `google_sql_user`, el proveedor completa `deletion_policy` con su valor por defecto
  (`DELETE`) al importar. Ese cambio se aplicó con los imports. Es un campo que existe solo en
  Terraform: tocó el estado, no GCP.
- **Paralelismo de `revisar-plan.sh`.** Con el paralelismo por defecto (10), Cloud Shell dio
  errores de red pasajeros al leer muchos recursos a la vez ("dial tcp [IPv6]:443: connect:
  cannot assign requested address"). Con `-parallelism=3` el plan salió sin errores.
  `revisar-plan.sh` corre con 3 por defecto, y se puede cambiar con `PARALELISMO=…`.
- **La diferencia de `google_sql_user.sima` es por leer con la base apagada.** Después de los
  imports, el plan mostraba un update de `google_sql_user.sima` con
  `sql_server_user_details` y `password_policy` como "known after apply". Con Cloud SQL
  apagado, la API de usuarios responde 400 ("Invalid request since instance is not
  running"). El proveedor deja esos dos atributos en `null` y el plan los marca como
  desconocidos, aunque lo conocido de antes y de después es idéntico.
  - **No es un cambio real y no se aplica.** Tampoco se corrige en `sql.tf`: agregar
    `sql_server_user_details` a `ignore_changes` no tiene efecto, porque es un atributo que
    decide el proveedor (Terraform lo advierte).
  - **Cómo se resuelve:** la próxima vez que GCP esté encendido, con
    `terraform apply -refresh-only -target=google_sql_user.sima`. Lee el usuario con la base
    corriendo y actualiza solo el estado.

## 2026-10-19 — Terraform: fase 6 (Datastream) escrita desde el borrador

**Recursos.** `datastream.tf` describe:
- la conexión privada `sima-conexion-privada`;
- los perfiles `sima-central-postgres` y `sima-bigquery`;
- el stream `sima-central-cdc`;
- el dataset `sima_cdc`, que el plan original dejaba para la fase 7 y va aquí porque lo crea
  `08` y lo usa el stream.

Los valores salen del borrador de `terraform plan -generate-config-out`. Lo que ya existe en
Terraform se referencia, no se copia: la red y el /29 (`red.tf`), la IP reservada del proxy,
la base `sima_central` y la lista de tablas que publica `cdc-central`. Los cinco recursos
llevan `prevent_destroy`.

**Contraseña del perfil de PostgreSQL: sin declarar.** En el proveedor 8.5.0,
`postgresql_profile.password` es opcional y no fuerza un reemplazo. La API no la devuelve, y
el proveedor la toma solo de la configuración o del estado. Sin declararla, configuración y
estado valen `null`: no hay diferencia, no hay contraseña de relleno y nada queda en el estado.
Va en `ignore_changes` por si alguna vez se cambia desde `gcloud`. La contraseña real vive en
Secret Manager (`sima-db-replicacion-password`) y en el perfil, que la guarda cifrada.

**Mejora pendiente: `secret_manager_stored_password`.** El proveedor 8.5.0 permite que el
perfil tome la contraseña de Secret Manager. Es lo correcto para crear el proyecto desde cero,
porque la contraseña nunca pasaría por un comando. Hoy no se hace: el perfil se creó con la
contraseña directa, y declarar la referencia sería un update del perfil durante la adopción. Va
en un cambio aparte, después de terminar la fase 8, con el stream pausado y validando la
conexión.

**El dataset sin bloques `access`.** El borrador trae los cinco permisos del dataset, entre
ellos el OWNER de una cuenta personal y el READER de `sa-analitica` que dio `09`. En
`google_bigquery_dataset`, `access` es autoritativo: declararlo haría que Terraform quitara
cualquier permiso que no esté en la lista, y obligaría a copiar al repositorio el de la cuenta
personal para no perderlo. Es opcional y Computed, así que sin declararlo no hay diferencia y
Terraform no lo administra. La
dirección de la cuenta personal no queda en el repositorio. El permiso de `sa-analitica` se
adopta en la fase 7 como `google_bigquery_dataset_iam_member` o `google_bigquery_dataset_access`,
junto con el de `sima_analitica`.

**Detalles del proveedor que evitan diferencias:**
- **Los perfiles se referencian por su `id`.** GCP los guarda en el stream con el número de
  proyecto (`projects/506391465588/…`), y el proveedor suprime esa diferencia de formato
  (`ProjectNumberDiffSuppress`). Lo mismo pasa con `dataset_id` (`proyecto:dataset`).
- **`location` del dataset vale `US` por defecto y cambiarla lo reemplaza.** Se declara
  `us-central1`.
- **`desired_state` del stream vale `NOT_STARTED` por defecto.** Va en `ignore_changes`, porque
  lo maneja `encender-apagar.sh`. Hoy el stream está `PAUSED`.
- **`max_concurrent_backfill_tasks` viene en 0** y no se declara.

**Errores del borrador.** `default_table_expiration_ms = 0` y
`default_partition_expiration_ms = 0` no pasan la validación del proveedor (el mínimo es una
hora). El dataset no tiene expiración, así que no se declaran.

## 2026-10-19 — Terraform: fase 7 (BigQuery) escrita desde el borrador

**Solo la infraestructura de BigQuery.** `bigquery.tf` describe el dataset `sima_analitica`,
los dos permisos que `09` le dio a `sa-analitica` y la consulta programada
`sima-hecho-movimiento`. Las tablas (`dim_fecha`, `historia_simulada`, `hecho_movimiento`) y
las vistas (`02_dimensiones.sql`, `06_vistas.sql`) quedan en el script de operación: el SQL de
BigQuery está fuera de Terraform por la decisión 1.

**Permisos de `sa-analitica`.** `09` dio READER sobre `sima_cdc` y WRITER sobre
`sima_analitica`, con `bq update --source`. El proveedor los lee como IAM
(`roles/bigquery.dataViewer` y `roles/bigquery.dataEditor`) y se adoptan como
`google_bigquery_dataset_iam_member`, que no es autoritativo.
`google_bigquery_dataset_access` no se puede importar.

**El dataset, igual que `sima_cdc`.** `location` va declarada: el valor por defecto del
proveedor es `US` y cambiarla lo reemplaza. No lleva bloques `access`, que son autoritativos, y
el borrador traía como OWNER la cuenta personal del proyecto, que no queda en el repositorio.
Las dos expiraciones en 0 no se declaran. `prevent_destroy`.

**El `MERGE` sale del mismo archivo que usa `09`.** La consulta de la consulta programada es
`templatefile("…/infra/gcp/bigquery/05_merge_hecho.sql", { PROYECTO = var.proyecto })`. El
archivo solo contiene `${PROYECTO}`, y ningún `%{`. Se comprobó que el resultado es idéntico,
byte a byte, al `sed` de `09` y al texto que tiene GCP. Queda una sola fuente.

**Nada que dependa de cuándo corrió.** El esquema del proveedor 8.5.0 no expone
`next_run_time`, `state` ni `user_id` de la consulta programada. `disabled = false` y
`schedule = "every 1 hours"` coinciden con GCP, así que un plan no la pausa ni la reprograma.
`prevent_destroy`.

**`service_account_name` va en `ignore_changes`.** La consulta corre como `sa-analitica`
(`09`). El proveedor manda ese campo como parámetro de la URL al crear o actualizar, y no lo
lee de la API: después de importar vale `null`. Se declara, para que quede escrito y sirva al
crear el proyecto desde cero, y se ignora para que no aparezca como cambio.

**El ID de la consulta programada** (`6ad6ef14-0000-21b6-9346-d4f547fdaf4c`) lo asignó GCP al
crearla y no estaba en ningún script. Se leyó con `bq ls --transfer_config` y queda solo en el
bloque `import`, que se borra al terminar la adopción.

## 2026-10-19 — Terraform: fase 8 (presupuesto y alertas) escrita desde el borrador

**Recursos.** `monitoreo.tf` describe:
- el presupuesto "SIMA dev";
- el canal "SIMA alertas por correo";
- la verificación de disponibilidad de `/salud`;
- las métricas `sima_api_central_errores` y `sima_2pc_sin_resolver`;
- las 5 políticas.

Los valores salen del borrador de `terraform plan -generate-config-out`. Los IDs que asignó GCP
(presupuesto, canal, verificación y políticas) se leyeron con `borradores/ids-fase-8.sh` y
quedan solo en los bloques `import`.

**Las políticas salen de `infra/gcp/monitoreo/*.json`.** El recurso no acepta un JSON
entero, así que cada política se arma con `jsondecode(templatefile(...))` y bloques `dynamic`
que traducen los nombres del JSON (camelCase) a los del recurso (snake_case). `templatefile()`
completa los `${…}` con referencias a otros recursos: el canal, el ID de la verificación, las
métricas y la instancia. Hay una sola fuente, la misma que usa `11`. Se comprobó de dos
maneras:
- campo por campo, que las 5 políticas del borrador coinciden con los JSON completados. Lo
  que el JSON no tiene viene vacío o en su valor por defecto: `severity`, `subject`,
  `denominator_filter`, `evaluation_missing_data` y `trigger.percent`;
- evaluando en Terraform las mismas expresiones de `monitoreo.tf`, que producen esos valores
  sin diferencias.

`enabled` va en `ignore_changes`, porque lo manejan `encender-apagar.sh` y `11`.

**El correo de las alertas no está en el repositorio.** El borrador trae la dirección en
`labels` del canal. `labels` va en `ignore_changes`, y la dirección solo se pasa para crear el
canal desde cero, con la variable `alerta_correo`. Esa variable vale `null` por defecto: si no
tuviera valor por defecto, habría que pasarla en cada plan, aunque esté ignorada.

**Presupuesto.**
- **Proveedor con proyecto de cuota.** Con credenciales de usuario, la API de presupuestos
  exige un proyecto de cuota, así que el presupuesto usa el proveedor `google.facturacion`, que
  lo tiene. Es el único recurso que lo usa: los demás se siguen leyendo igual.
- **La cuenta de facturación** sale de `data.google_project.proyecto.billing_account`, la
  cuenta a la que está ligado el proyecto, sin escribir su ID. Primero se intentó con
  `data "google_billing_account" { open = true }`, como busca `10`, pero en 8.5.0 ese data
  source exige `billing_account` o `display_name`. Según el código del proveedor
  (`resource_google_project.go`), el atributo viene sin el prefijo `billingAccounts/`, que es
  como lo guarda el presupuesto. Es importante porque `billing_account` es `ForceNew`, y un
  formato distinto lo reemplazaría.
- **Si la cuenta llega vacía, el plan se corta.** Si la API de Cloud Billing no está
  habilitada, el proveedor solo deja una advertencia y `billing_account` queda vacío, y un
  plan querría reemplazar el presupuesto. Una `precondition` en el presupuesto lo corta con un
  mensaje claro.
- **El monto.** El borrador traía `specified_amount` y `last_period_amount` a la vez, que es su
  único error de validación. Queda solo `specified_amount` (USD 300), que es lo que creó `10`.
- **Lo que hay que declarar.** `credit_types_treatment = "EXCLUDE_ALL_CREDITS"`, porque el
  valor por defecto del proveedor es incluirlos. El periodo sale de las variables
  `presupuesto_inicio` y `presupuesto_fin`, con los mismos valores por defecto que
  `infra/gcp/variables.sh`.

**Verificación de disponibilidad y métricas.** El host sale de la URL de `api-central` y los
filtros de las métricas, del nombre del servicio. Se comprobó que dan exactamente el texto que
tiene GCP. Además hay que declarar `use_ssl` y `accepted_response_status_codes`: no se
completan solos.

## 2026-10-19 — Terraform: fin de la adopción

**Fase 8 adoptada.** Se importaron los 10 recursos; las 5 políticas siguen deshabilitadas,
porque GCP está apagado. Con eso las ocho fases están aplicadas: 77 recursos.

**La API de Cloud Billing hacía falta y no estaba habilitada.** En la fase 8,
`data.google_project.proyecto.billing_account` llegaba vacío. El proveedor lee la cuenta de
facturación del proyecto con la API `cloudbilling.googleapis.com`, y si no está habilitada no
da error: solo deja una advertencia. Ninguno de los scripts de `infra/gcp` la había
habilitado. Se habilitó a mano con
`gcloud services enable` y se agregó a `apis.tf` con su `import`: son 78 recursos. La
`precondition` del presupuesto corta el plan con un mensaje si la cuenta vuelve a llegar vacía.

**Limpieza.**
- Se borraron `importaciones.tf` y `borradores/` (los `.importar` de las fases 4 a 8 e
  `ids-fase-8.sh`). Quitar un bloque `import` que ya se aplicó no cambia nada: el recurso sigue
  en el estado.
- Queda un solo `import`, el de la API de Cloud Billing en `apis.tf`, que se borra una vez
  aplicado.

**De ahora en adelante.** Todo cambio de infraestructura se hace en los `.tf`, se revisa con
`revisar-plan.sh` y se aplica el plan revisado (`terraform apply plan.bin`). Con GCP apagado
el plan trae siempre el update conocido de `google_sql_user.sima`, que no se aplica: se aplica
solo lo que se cambió, con `-target`, o se refresca el usuario con GCP encendido. Está en
`infra/terraform/README.md`.

**Pendiente de la decisión 1.** Los scripts de operación en `infra/terraform/operacion/`
(imágenes, ejecución de los jobs y SQL de BigQuery, leyendo `terraform output`) todavía no
están escritos. Hasta entonces, esos pasos siguen en `infra/gcp/04`, `05`, `08` y `09`.

## 2026-10-19 — Sincronización por Pub/Sub (adaptador PUBSUB, consumidor push, mensajes muertos)

Para la entrega en GCP, la sincronización sucursal → central puede ir por Pub/Sub en vez de
por HTTP directo. El dominio no cambia: `publicar-pendientes.ts`, el `outbox` y
`aplicar-mensaje.ts` son los mismos, y el adaptador HTTP sigue como está. Lo nuevo es un
adaptador de transporte (`transporte-pubsub.ts`, elegido con `TRANSPORTE_SINCRONIZACION=PUBSUB`),
el desempaque del sobre push en el consumidor (`sobre-pubsub.ts`) e `infra/terraform/pubsub.tf`
(escrito, no aplicado).

**Un mensaje por fila de outbox.** `data` es el JSON del `MensajeSincronizacion`, el mismo cuerpo
del POST HTTP. Los atributos son `id_sucursal` e `id_mensaje` (= `id_idempotencia`). Sin lotes
(`maxMessages: 1`): el publicador espera cada confirmación antes de la fila siguiente, igual que
con HTTP.

**Qué quiere decir OK.** Con HTTP, OK es "el central lo aplicó". Con Pub/Sub, OK es "el tema lo
aceptó": la fila pasa a `ENVIADO` aunque el central esté caído (la prueba lo muestra: con la
base del central detenida, el outbox de la sucursal se vacía igual). Si después el central lo
rechaza, la sucursal no se entera: el mensaje termina en el tema de muertos. Por eso con PUBSUB
`duplicado` es siempre `false`, y `FALLIDO` en el outbox solo aparece si el mensaje no se puede
serializar o Pub/Sub responde `INVALID_ARGUMENT`.

**Clasificación de errores al publicar.**

| Error | Resultado | Por qué |
|---|---|---|
| red, DNS, timeout propio, `UNAVAILABLE`, `DEADLINE_EXCEEDED` | `SIN_ENLACE` | No se llegó a Pub/Sub |
| `NOT_FOUND`, `PERMISSION_DENIED`, `UNAUTHENTICATED` | `TRANSITORIO` | Son de configuración (tema, permisos, credenciales), no del mensaje. Como `RECHAZADO`, quemarían filas sanas hasta pasarlas a `FALLIDO` |
| los demás códigos gRPC | `TRANSITORIO` | Se arreglan solos o no son del mensaje |
| no serializable, `INVALID_ARGUMENT` | `RECHAZADO` | El problema es el mensaje |

El último error queda en `sincronizacion.ultimo_error` de `/salud`: `codigo` es el nombre gRPC
(`NOT_FOUND`, …) y `mensaje` empieza con `gRPC <número>`. La llamada a `publish` va con
`timeout` = `SINCRONIZACION_TIMEOUT_MS` y sin reintentos de la librería; además, un plazo propio
de respaldo, para que el publicador no se cuelgue si la librería no responde. Si un mensaje
cuyo plazo venció se publica igual más tarde, el central lo descarta por `inbox_idempotencia`.

**Sin claves de orden.** *Alternativa descartada:* `ordering_key` = `id_sucursal`. Con claves
de orden, un mensaje que falla detiene todos los de su clave hasta que se resuelva, y la
librería pausa la publicación de esa clave después de un error: es el bloqueo de cabeza de cola
que el diseño ya evita en el central. No hace falta: el central aplica cada mensaje una sola vez
(`inbox_idempotencia`) y la guarda de versión de `existencia_consolidada` solo deja pasar lo más
nuevo, sin importar el orden de llegada (es lo que prueba el caso de desorden de
`prueba-consumidor`). Los documentos y movimientos son inserciones idempotentes.

**Consumidor: el mismo endpoint.** `POST /sincronizacion/mensajes` acepta el sobre de la
suscripción push (`{message: {data en base64, attributes, messageId}, subscription}`) y también
el POST directo del adaptador HTTP. Las respuestas no cambian: 2xx si se aplicó o era
duplicado (Pub/Sub hace ack), 4xx si es inválido, 5xx si el problema es transitorio. Un sobre
con `data` que no es base64 o no es JSON es 400. *Alternativa descartada:* un endpoint aparte
para Pub/Sub; habría dos rutas al mismo `aplicarMensaje` sin ninguna diferencia de fondo.

**Mensajes muertos: 15 entregas, reintentos de 10 a 600 s.** Pub/Sub reintenta todo lo que no
es 2xx, también los 4xx: no distingue un mensaje inválido de un central caído. Con
`max_delivery_attempts = 15`, el mensaje pasa a `sima-sincronizacion-muertos`, que tiene su
suscripción `sima-sincronizacion-muertos-sub` (sin ella, el tema de muertos descartaría lo que
recibe) con 7 días de retención.

*El compromiso:* si el central (la API o su base) está caído más tiempo del que cubren las 15
entregas, también los mensajes **válidos** terminan en muertos. Si el backoff se duplica desde
10 s hasta el tope de 600 s, las 14 esperas suman 5 430 s, alrededor de 1,5 h. En el plan
aprobado se estimó 1,7 h, y Pub/Sub no documenta la curva exacta. No se pierden: se reenvían con
`scripts/pubsub/muertos.mjs reenviar`, y reenviar es seguro porque el central es idempotente y
no depende del orden. *Alternativa descartada:* más entregas (hasta 100). Un mensaje inválido
de verdad se quedaría reintentando horas, mezclado con los válidos.

Para que un mensaje pase a muertos, el agente de servicio de Pub/Sub necesita publicar en el
tema de muertos y ser suscriptor de la suscripción de origen (en `pubsub.tf`). Sin esos
permisos, los mensajes se reintentan para siempre.

**El reenvío** (`scripts/pubsub/muertos.mjs`) va contra el emulador o contra GCP, con el token
de `gcloud`. Tiene tres acciones:
- `listar` los muestra sin sacarlos;
- `reenviar` vuelve a publicar `data` y los atributos originales, sin los `CloudPubSubDeadLetter*`
  que agrega Pub/Sub. Hace ack en muertos **después** de publicar, así que si la publicación
  falla, el mensaje se queda;
- `descartar` saca los mensajes que nunca van a ser válidos. Pide sus id, o `--todos` a propósito.

No se agregó una alerta sobre los mensajes muertos (no estaba pedida). Se revisan con
`muertos.mjs listar`.

**Autenticación del push: token OIDC.** La suscripción entrega con un token OIDC de
`sa-pubsub-push`, que tiene `roles/run.invoker` sobre `api-central`; la audiencia es la URL del
servicio. Por ahora `api-central` sigue admitiendo `allUsers` y el token sobra, pero queda
puesto para cuando se quite. Lo verifica Cloud Run, no el código del central. El proyecto se
creó después del 8 de abril de 2021, así que el agente de Pub/Sub no necesita
`roles/iam.serviceAccountTokenCreator` para firmar el token. *Alternativa descartada:* un token
compartido en la URL del push; es un secreto en la configuración de la suscripción y no se rota.

**Las VMs siguen en HTTP.** Pasar las sucursales a PUBSUB cambia su script de arranque, y eso
es un cambio en el lugar de las VMs, que `revisar-plan.sh` bloquea a propósito. Se hace en el
despliegue, como cambio planificado aparte (aprobado así). El scope de las VMs ya es
`cloud-platform`, así que no hay que cambiarlo, y el rol `roles/pubsub.publisher` de
`sa-sucursal` está en `pubsub.tf`.

**`revisar-plan.sh` acepta recursos nuevos.** Durante la adopción, crear algo era un error:
todo tenía que ser importado. Ahora que la infraestructura se cambia en los `.tf`, crear es
normal. Un plan que solo crea, o que crea y cambia en el lugar, sale con 2 y lista cada
recurso. Destruir, reemplazar, o cambiar en el lugar Cloud SQL o una VM sigue saliendo con 1.
`infra/terraform/probar-revisar-plan.sh` lo prueba con 8 planes falsos.

**El emulador, sin `retryPolicy`.** En el emulador (`google-cloud-cli:587.0.0-emulators`), con
`retryPolicy` y **varios** mensajes fallando a la vez, la suscripción push deja de entregar
después de unos 12 intentos en total y no vuelve a intentar. Se midió:
- se esperó 8 minutos y no retomó;
- se probaron backoffs de 0 s, 0–1 s, 0,2–0,5 s y 1–2 s, y en todos se trabó;
- con un solo mensaje, los 5 intentos y el paso a muertos funcionan;
- sin `retryPolicy`, 4 mensajes llegan a muertos en 6 s, con 20 intentos.

Por eso la prueba local usa `INTENTOS=5 SIN_BACKOFF=1`: lo que se prueba (que un mensaje pasa a
muertos al agotar las entregas, y que el reenvío lo aplica) no depende del backoff.
`preparar.sh` admite `BACKOFF_MIN` y `BACKOFF_MAX` (por defecto, 10 s y 600 s, como en GCP),
pero en el emulador solo sirven con un mensaje a la vez. *Límite:* la política de reintentos de
10 a 600 s de GCP no está probada en local; se verifica en GCP cuando se despliegue.

**Reconexión después de un corte.** Al volver la red, el canal gRPC de la librería reconecta
con su propio backoff. En la prueba, la cola se vació unos 2 s después de restablecer el
enlace. Después de un corte largo puede tardar más, hasta que el canal vuelve a intentar; no se
midió. No hace falta hacer nada: el publicador sigue intentando en cada ciclo.

**Dependencia nueva:** `@google-cloud/pubsub` 6.1.0, fijada exacta en `api-sucursal`. `npm audit`
da las mismas 8 vulnerabilidades antes y después de agregarla. Todas son de la cadena de
NestJS (multer, proxy-addr, qs, body-parser, file-type); Pub/Sub no suma ninguna. Se tratan
aparte.

**Pruebas.**
- `scripts/prueba-pubsub/clasificacion.mjs`: el adaptador sin Pub/Sub, 34 verificaciones.
- `scripts/prueba-pubsub/ejecutar.sh`: con el emulador, 7 casos:
  1. venta aplicada;
  2. sobre push en el consumidor;
  3. corte de enlace;
  4. tema inexistente: `NOT_FOUND` en `/salud`;
  5. mensaje inválido que pasa a muertos y se descarta;
  6. base del central caída: muertos, reenvío y aplicación, sin duplicados;
  7. verificación de las 3 sucursales.
- Las dos están en `probar-todo.sh`.

## 2026-10-20 — api-central privado y Cloud Scheduler para la recuperación del 2PC

Hasta ahora `api-central` admitía peticiones sin autenticar (`allUsers` como invoker, desde
`05-cloud-run.sh`). Pasa a ser **privado**: Cloud Run solo deja pasar peticiones con un token de
identidad de una cuenta con `roles/run.invoker` sobre el servicio. El código del central no
verifica nada: lo verifica Cloud Run. Está escrito en `infra/terraform/acceso-central.tf`, sin
aplicar; se despliega junto con Pub/Sub (orden y lista de verificación en
`infra/terraform/README.md`).

**Quién invoca.**

| Quién | Cuenta | Cómo |
|---|---|---|
| Entrega push de Pub/Sub | `sa-pubsub-push` | Token OIDC de la suscripción (`pubsub.tf`) |
| Sucursales: reconciliador y transporte HTTP | `sa-sucursal` | Token del metadata server de la VM |
| Cloud Scheduler | `sa-scheduler` | Token OIDC del job |
| Verificación de `/salud` | Agente de servicio de Monitoring | OIDC (`service_agent_authentication`) |
| Quien opera el proyecto | La cuenta de `gcloud` | Su rol de proyecto ya incluye invocar |

Las llamadas de la sucursal al central son dos, y las dos están en `api-sucursal`:
- `POST /sincronizacion/mensajes`, del transporte HTTP. En GCP las sucursales pasan a PUBSUB,
  pero el HTTP queda como respaldo y también lleva el token.
- `GET /traslados/decision/:gid`, del reconciliador.

El dominio (`nucleo`) no hace llamadas HTTP. El central llama a las sucursales (coordinador →
participantes) por IP interna (`SUCURSAL_URL_PLANTILLA=http://10.10.0.1{id}:3000`, salida por la
VPC): la autenticación de Cloud Run controla lo que entra al servicio, no lo que sale, así que
eso no cambia.

**Token de identidad en la sucursal.** Va en `api-sucursal/src/autenticacion/token-identidad.ts`,
sin dependencias nuevas, con `fetch` al metadata server
(`instance/service-accounts/default/identity?audience=<CENTRAL_URL>`).
- **Opcional:** `CENTRAL_AUTENTICACION=NINGUNA|METADATA`. Por defecto es `NINGUNA`, así que en
  Docker Compose no cambia nada: no hay metadata server. `METADATA_URL` permite apuntar a uno
  falso en la prueba.
- **Caché:** se lee el `exp` del JWT y se renueva cuando faltan menos de 5 minutos (los tokens
  duran una hora). Varias llamadas a la vez esperan la misma petición. Un token sin `exp`
  legible no se guarda.
- **Si no hay token:** el metadata server necesita llegar a Google para emitirlo. Si no
  responde, se vence el plazo o responde 5xx, es `SIN_ENLACE`, que es lo que pasa en una tienda
  sin internet. Si responde 4xx (cuenta o audiencia mal configuradas), es `TRANSITORIO` con
  código `TOKEN_IDENTIDAD`, visible en `/salud`, y no suma intentos.
- *Alternativa descartada:* `google-auth-library`. Es una dependencia nueva para una sola
  petición HTTP.
- *Alternativa descartada:* detectar sola si corre en GCE. Que la configuración lo diga explícitamente es
  más fácil de leer y de probar.

**401 y 403 son `TRANSITORIO` en `TransporteHttp`.** Antes, todo 4xx salvo 408 y 429 era
`RECHAZADO`. Con el central privado, Cloud Run responde 401 o 403 si el token no sirve o la
cuenta no tiene permiso. No es culpa del mensaje, y como `RECHAZADO` quemaría filas sanas hasta
pasarlas a `FALLIDO`. Es el mismo criterio que con `PERMISSION_DENIED` y `UNAUTHENTICATED` en
Pub/Sub. El central nunca responde 401 ni 403 por su cuenta. Un 401 descarta el token guardado.
`FuenteDecisionHttp` ya trataba todo lo que no fuera 200 o `404 GID_DESCONOCIDO` como
`SIN_RESPUESTA`: un 401 o 403 nunca autoriza a deshacer una transacción preparada.

**La verificación de `/salud` con OIDC.**
- El proveedor 8.5.0 lo soporta: `http_check.service_agent_authentication { type = "OIDC_TOKEN" }`.
  Se confirmó en el esquema del binario instalado.
- La documentación de GCP dice que solo sirve en checks HTTPS (este lo es) y que, para Cloud
  Run, quien crea el check necesita `run.routes.invoke`. No nombra al agente.
- El aviso TRA-2026-05 de Tenable dice que el agente es
  `service-<número>@gcp-sa-monitoring-notification.iam.gserviceaccount.com` y que ya trae
  `run.routes.invoke` por defecto. Igual se le da `roles/run.invoker` explícito sobre
  `api-central`, para no depender de ese valor por defecto. Que funcione se comprueba en GCP.

**Las personas.** Solo la cuenta que ya opera el proyecto: su rol de proyecto incluye invocar, y
no hay ningún permiso para personas en Terraform. Nadie más se agrega al proyecto.
- **Scripts y pruebas de carga (k6):** `Authorization: Bearer $(gcloud auth print-identity-token)`.
  El token dura una hora y no agrega ningún salto que distorsione la medición.
  `verificar-gcp.sh` ya lo usa.
- **Demo en el navegador:** `gcloud run services proxy api-central --region=us-central1
  --port=8080`, con la Vista previa web de Cloud Shell. Renueva el token solo.
- *Alternativa descartada:* el proxy para las pruebas de carga. Un solo proxy local sería el
  cuello de botella, y se mediría el proxy y no el central.

**Frontend.** Hoy no hay ninguna pantalla de navegador: no existe el paquete del frontend, ni
HTML, ni CORS habilitado en las APIs. Las pantallas del hito posterior que llamen a su sucursal
no se ven afectadas. Si alguna necesita datos del central, no podrá pedirlos directo desde el
navegador. Se resuelve en ese hito.

**Cloud Scheduler: la recuperación del coordinador cada minuto.**
- En GCP, el intervalo interno de la recuperación (5 s) no corre: Cloud Run escala a cero y,
  con `cpu_idle`, no da CPU entre peticiones. El job `sima-recuperacion-2pc` llama a
  `POST /traslados/recuperacion/ejecutar` con OIDC de `sa-scheduler`.
- **Por qué 1 minuto.** Si el coordinador muere a mitad del protocolo, la transacción queda en
  `INICIADO` y `/traslados/decision` responde `PENDIENTE`. Mientras tanto, la transacción
  preparada del participante retiene los bloqueos de `existencia`, y las ventas de esos
  productos en esa sucursal responden `409 PRODUCTO_BLOQUEADO`. Solo la recuperación aborta
  `INICIADO`, a los 60 s: el bloqueo dura hasta unos 60 s + N. Con N = 1 son unos 2 minutos;
  con N = 5, unos 6.
- **Costo:** unas 43 000 invocaciones al mes, dentro del nivel gratuito de Cloud Run. Cloud
  Scheduler da 3 jobs gratis por cuenta de facturación.
- *Alternativa descartada:* `min_instance_count = 1`. Cuesta una instancia siempre encendida, y
  con `cpu_idle` los temporizadores igual no corren.
- `retry_count = 0`: el reintento es la ejecución del minuto siguiente. `attempt_deadline = 120s`.
- **Al apagar GCP:** `encender-apagar.sh` pausa el job antes de apagar Cloud SQL, porque si no,
  cada minuto habría un 503 y un error en los registros de `api-central`. Lo reanuda al
  encender, con la base arriba. `paused` va en `ignore_changes`.

**Ejecuciones cruzadas.** Si una ejecución tarda más de un minuto, se cruza con la siguiente.
Con varias instancias, o con el intervalo interno y el endpoint, también. No pasa nada:
- cada transacción global se toma con `FOR UPDATE SKIP LOCKED`;
- la decisión se escribe con la condición `estado = 'INICIADO'`;
- `SOLICITADO` se retoma con `SKIP LOCKED` más una condición;
- la que llega tarde a una fila tomada o ya resuelta la cuenta en `omitidas`.

Ya estaban probadas dos ejecuciones sin pendientes (caso 6 de `prueba-coordinador`) y la carrera
del coordinador contra la recuperación (caso 5). El caso 7 nuevo prueba **cuatro** ejecuciones a
la vez **con pendientes**:
- 3 traslados quedan `COMPROMETIDO` sin acuse del destino (se corta su enlace entre los votos y
  la decisión);
- con el enlace de vuelta, las 4 recuperaciones se reparten los 3 traslados, uno cada una, y la
  cuarta no encuentra nada;
- ningún traslado se propaga dos veces, los 3 quedan `FINALIZADO` y `verificar.mjs` cuadra.

**Quitar `allUsers`.** No es un cambio del servicio de Cloud Run: es un **destroy** de
`google_cloud_run_v2_service_iam_member.api_central_publico`.
- Ahora el recurso depende de la variable `api_central_publico`, por defecto `false`. Lleva un
  bloque `moved`, porque antes estaba sin `count` en `cloud_run.tf` y sin él el plan lo
  destruiría y lo volvería a crear.
- Para volver atrás: `-var "api_central_publico=true"`.
- `revisar-plan.sh` no se afloja: el destroy sale con 1 y se aprueba explícito, como paso aparte
  del despliegue. `probar-revisar-plan.sh` suma ese caso: son 9 planes.
- `05-cloud-run.sh` lleva una advertencia: volver a correrlo devolvería `allUsers`.

**`arranque.sh` de las sucursales, con METADATA y PUBSUB.** Se escribe hoy para que el
despliegue sea un solo cambio planificado de las VMs. Agrega `CENTRAL_AUTENTICACION=METADATA`,
`TRANSPORTE_SINCRONIZACION=PUBSUB`, `PUBSUB_PROYECTO` (leído del metadata server) y
`PUBSUB_TEMA`. Cambia el metadata `startup-script` de las 3 VMs: es un update en el lugar, que
`revisar-plan.sh` bloquea con 1 y se aprueba explícito. No reinicia la VM: el script corre en
el próximo arranque.

**Pruebas.**
- `scripts/prueba-token-identidad/prueba.mjs`: 32 verificaciones, con un metadata server y un
  central falsos. Cubre la caché, la renovación, la audiencia, las fallas del metadata server,
  401/403 en los dos clientes y que con `NINGUNA` no hay encabezado.
- `prueba-coordinador`, caso 7.
- `probar-revisar-plan.sh`: 9 planes.
- El resto de `probar-todo.sh` corre igual, con `NINGUNA`: demuestra que el flujo local no
  cambia.

Lo que solo se puede probar en GCP está en la lista de verificación de
`infra/terraform/README.md`.

## 2026-10-21 — Despliegue: el agente de Monitoring, el job de Scheduler y el corte de enlace con Pub/Sub

En el despliegue de Pub/Sub y del central privado, con `allUsers` todavía puesto,
`verificar-gcp.sh` dio 51 de 55. Tres cosas no funcionaron. Las tres estaban escritas a partir de
supuestos que no se comprobaron en GCP antes de aplicar.

### Verificación de `/salud` con OIDC

**Qué pasó.** El permiso de invoker para el agente falló con "Service account
service-…@gcp-sa-monitoring-notification.iam.gserviceaccount.com does not exist". El cambio de la
verificación falló con "Service agent authentication is not supported for arbitrary URLs".

**El agente.** La dirección era la correcta: según la lista de agentes de servicio de IAM, el
agente principal de `monitoring.googleapis.com` es
`service-PROJECT_NUMBER@gcp-sa-monitoring-notification.iam.gserviceaccount.com`, con el rol
`roles/monitoring.notificationServiceAgent`. Lo que pasaba es que en este proyecto **no existía**:
Google crea los agentes cuando un servicio los necesita, y acá nada lo había pedido. El 2026-10-20
se dio por hecho que existía y que ya tenía `run.routes.invoke`, a partir de un aviso de un tercero
(Tenable), sin comprobarlo en el proyecto.

- **Ahora se crea explícito**, como indica la guía de IAM "Create and grant roles to service
  agents": `google_workload_identity_service_agent` (proveedor GA 8.5.0, API
  `workloadidentity.googleapis.com`) con `parent = projects/<número>/locations/global/serviceProducers/monitoring.googleapis.com`.
- **Crearlo no le da roles:** su rol de proyecto se da aparte (`google_project_iam_member.agente_monitoring`),
  y el invoker sobre `api-central` también.
- **Los permisos usan la dirección escrita a mano,** para que el plan no dependa de valores que
  recién se conocen después del apply. Una `postcondition` comprueba que el agente creado sea
  ese; si no lo es, el apply lo dice.
- *Alternativa descartada:* `google_project_service_identity`. Está solo en google-beta y usa la
  API anterior (`generateServiceIdentity`); la guía actual de IAM usa la de Workload Identity.

**La verificación.** La autenticación con el agente de servicio no se admite sobre una
`uptime_url`, que es una URL cualquiera.
- Ahora la verificación es sobre el servicio: `monitored_resource` de tipo `cloud_run_revision`,
  con `project_id`, `service_name` y `location` (las etiquetas del descriptor del recurso;
  `revision_name` y `configuration_name` no se fijan, porque se verifica el servicio y no una
  revisión). `cloud_run_revision` es uno de los tipos válidos para verificaciones según la API.
- **Reemplazo:** `monitored_resource` es `ForceNew` en el proveedor, así que el cambio reemplaza
  la verificación y le cambia el ID. Con `create_before_destroy` se crea la nueva, la política
  de salud pasa a su ID en el mismo apply y recién después se borra la vieja.
- **La política tenía un segundo problema:** su filtro decía `resource.type="uptime_url"`. Según
  la lista de métricas de Google Cloud, `uptime_check/check_passed` se escribe con el tipo del
  recurso verificado (`cloud_run_revision` incluido): con el filtro viejo, la política no habría
  visto nunca una falla. Ahora el tipo es una variable de la plantilla
  (`${TIPO_RECURSO_VERIFICACION}`): Terraform pasa el de la verificación, y `11-monitoreo.sh`,
  que crea una `uptime_url`, pasa `uptime_url`.
- **`verificar-gcp.sh` (sección 8)** ahora comprueba el tipo y el servicio de la verificación, que
  haya una sola, y que la política de salud mire su ID y su tipo actuales.

**Lo que solo se ve en GCP.** El proveedor suprime diferencias solo en `project_id`: si GCP
devolviera otras etiquetas, el plan querría reemplazar la verificación cada vez. Por eso, después
del apply, el plan tiene que salir con 0 (paso 3 del arreglo en `infra/terraform/README.md`).

**Seguimiento, después del apply.** Pasó eso: la API guarda además `configuration_name = ""` y
`revision_name = ""`, y el plan quería reemplazar la verificación por `monitored_resource.0.labels`.
Ahora las cinco etiquetas del descriptor están declaradas, las dos de revisión vacías. Se comprobó
sin GCP con `terraform plan -refresh=false`, sobre un estado armado con lo que guarda la API:
- con las tres etiquetas de antes, el plan reemplaza (`- "configuration_name" = null`,
  `- "revision_name" = null`, `# forces replacement`);
- con las cinco, no hay ninguna diferencia en `labels`: el proveedor 8.5.0 conserva los valores
  vacíos de un mapa.

La confirmación en GCP es que `revisar-plan.sh` no muestre más la verificación.

### Job de Cloud Scheduler

**Qué pasó.** El job `sima-recuperacion-2pc` nunca se ejecutó:
- `status.code` en -1, sin `lastAttemptTime` y con `scheduleTime` trabado;
- `jobs run` no hizo nada;
- `jobs resume` falló con NOT_FOUND sobre un recurso interno
  (`…/streams/http-target-dynamic-stream/retryPolicies/cs-…`).

Además, el plan mostraba una diferencia perpetua en `retry_config`.

**Qué hace el proveedor 8.5.0** (`resource_cloud_scheduler_job.go`, en la etiqueta v8.5.0):
- El bloque `retry_config` es `Optional` pero no `Computed`.
- Con `retry_config { retry_count = 0 }`, el expand descarta el 0 por considerarlo vacío, y los
  demás campos no están puestos. Al **crear** no se manda `retryConfig`.
- Lo que queda en el estado no coincide con la configuración, y de ahí la diferencia perpetua.
- En cada **update**, el PATCH, que no lleva máscara, manda `"retryConfig": {}`, porque la
  condición mira el bloque de la configuración y no el valor expandido.

Lo más probable es que el job haya quedado roto por esos PATCH con una política de reintentos
vacía. Es una inferencia a partir del código y de los síntomas, no está demostrado.

> **Corregido el 2026-10-22:** esa no era la causa. El job recreado sin `retry_config` tampoco
> despachaba. La causa real era un permiso que le faltaba al agente de Cloud Scheduler (ver la
> entrada del 2026-10-22). La diferencia perpetua de `retry_config` sí existía, y quitar el
> bloque sigue estando bien.

**Arreglo.**
- **Se quita el bloque.** La referencia de la API dice que `retryCount` vale 0 por defecto: "If
  retryCount is 0 (and if maxRetryDuration is also 0), a job attempt won't be retried". Es lo
  mismo que se quería.
- **Sin el bloque, el proveedor no manda `retryConfig` al crear,** y el job queda con los valores
  por defecto de Cloud Scheduler.
- **`retry_config` va en `ignore_changes`,** junto a `paused`. Así Terraform nunca planifica un
  cambio, devuelva lo que devuelva la API. Un update de otro campo manda, como mucho, el mismo
  `retryConfig` que la API devolvió.
- **El job se recrea:** `./revisar-plan.sh … -replace=google_cloud_scheduler_job.recuperacion_2pc`.
  Los comandos están en `infra/terraform/README.md`, "Arreglo del 2026-10-21".

**Lección.** `revisar-plan.sh` ya había listado la diferencia en `retry_config` como cambio en
el lugar. Una diferencia que vuelve a aparecer después de aplicar no se aplica de nuevo: se
investiga.

### El corte de enlace no cortaba Pub/Sub

**Qué pasó.** Con el corte de `07-corte-enlace.sh`, que niega toda la salida de la VM y la entrada
al 3000, la sucursal siguió publicando, y el central cambió durante el corte.

**Por qué.** Las reglas de firewall de la VPC solo se aplican a conexiones nuevas: "Cloud NGFW
enforces this change only on new connections. Existing connections persist" (documentación de
Cloud NGFW, seguimiento de conexiones; una conexión seguida sigue activa si pasa al menos un
paquete cada 10 minutos).
- La librería de Pub/Sub mantiene una conexión gRPC (HTTP/2) abierta y viva: la regla nueva no
  la toca.
- Con el transporte HTTP el corte funcionaba porque `fetch` cierra la conexión ociosa a los
  pocos segundos, y cada envío abría una conexión nueva, que la regla sí negaba.
- En Docker Compose también funcionaba: `docker network disconnect` saca la interfaz y corta todo.

**Arreglo: una segunda capa, iptables dentro de la VM.** `infra/gcp/sucursal/corte-iptables.sh`,
que `07` corre por SSH (IAP), agrega a `DOCKER-USER` una cadena `SIMA-CORTE`. La cadena descarta
todo lo que entra o sale de los contenedores por la interfaz de la VM, salvo el metadata server.
iptables descarta cada paquete, también los de las conexiones abiertas, que es lo que pasa cuando
se cae el internet de una tienda.
- **Las reglas de la VPC se quedan,** para las conexiones nuevas, como en la sección 7.3 de la
  propuesta.
- **El metadata server sigue alcanzable,** igual que en una VM sin internet: las reglas de
  firewall nunca se le aplican. La sucursal consigue el token, pero no llega ni a Pub/Sub ni al
  central.
- **La caja y el SSH no se cortan:** `curl localhost:3000` en la VM y el SSH por IAP no pasan por
  `DOCKER-USER`.
- **Al restaurar** se quita la cadena y se borran las reglas, cada paso por separado. Las reglas
  de iptables también desaparecen si se reinicia la VM.
- *Alternativa descartada:* reiniciar el contenedor al cortar, para que se cierren las
  conexiones. Simula una caída de la API, no del internet, y no ejercita lo que pasa en una
  caída real: un `publish` sobre una conexión abierta que ya no responde termina en
  `DEADLINE_EXCEEDED`, que es `SIN_ENLACE`.

**Efecto sobre el 2PC.** El corte simula una sucursal fuera de línea en las dos direcciones,
igual que antes con HTTP, y el 2PC se comporta como está diseñado para ese caso:
- **Un traslado pedido durante el corte** con esa sucursal queda `SOLICITADO`, porque el
  coordinador no la alcanza. Al restaurar, la recuperación (Cloud Scheduler, cada minuto) lo
  retoma.
- **Un traslado que ya estaba preparado** en esa sucursal sigue preparado durante el corte: su
  reconciliador no llega al central, y los productos del traslado quedan bloqueados en esa
  sucursal (`409 PRODUCTO_BLOQUEADO`). Es el bloqueo propio del 2PC sin enlace, aceptado en el
  diseño. Al restaurar, el reconciliador y la recuperación lo resuelven.
- **Las demás sucursales no se ven afectadas.**

**Pruebas.**
- **`scripts/prueba-corte-iptables/prueba.sh`**, 11 verificaciones, sobre la VM Linux de Docker
  Desktop:
  - una conexión TCP ya abierta, de un contenedor a un servidor en la Mac, deja de pasar datos
    durante el corte y se reanuda sola al restaurar, sin reconectar;
  - durante el corte, una conexión nueva no llega, y desde afuera no se llega al puerto publicado;
  - la caja, dentro del host, sigue llegando al contenedor.
  - El corte de la prueba se limita a una red propia, con el tercer argumento del script. La
    primera versión, sin ese límite, cortó `eth0` para todos los contenedores, y eso incluye el
    tráfico interno de Docker Desktop: su API dejó de responder (500) hasta que la regla se quitó
    por el socket directo de `dockerd`. Por eso la prueba usa ese socket, y solo corre sobre
    Docker Desktop: en un Linux nativo tocaría el iptables del propio equipo.
- **`verificar-gcp.sh`, sección 5,** comprueba ahora en el `/salud` de la sucursal que durante el
  corte la bandeja acumula con `SIN_ENLACE`, y que al restaurar queda vacía.

## 2026-10-22 — La causa real del job de Scheduler que no despachaba

**Qué pasaba.** El job `sima-recuperacion-2pc`, también recreado sin `retry_config`, no se
ejecutaba nunca:
- estaba `ENABLED`, con `status.code -1` y `lastAttemptTime` vacío;
- no había ningún registro de `resource.type="cloud_scheduler_job"`;
- `scheduleTime` se quedaba siempre en el minuto siguiente.

El agente de Cloud Scheduler existía y tenía su rol (`roles/cloudscheduler.serviceAgent`).

**Cómo se encontró.** Se descartó primero lo más barato, de menor a mayor impacto:
- **Un job de diagnóstico sin autenticación** (`sima-prueba-sin-auth`, `GET` a
  `https://www.gstatic.com/generate_204`) **sí se ejecutó**. Cloud Scheduler despachaba en el
  proyecto y en la región: el problema era el token OIDC del nuestro.
- **Se le dio al agente** (`service-<número>@gcp-sa-cloudscheduler.iam.gserviceaccount.com`)
  `roles/iam.serviceAccountTokenCreator` sobre `sa-scheduler`, con `gcloud`. Al reanudar el job,
  **corrió**: tuvo `lastAttemptTime` y respondió con código 14, porque Cloud SQL estaba apagado.

**La causa.** El agente no podía emitir el token OIDC de `sa-scheduler`. Sin token, Cloud Scheduler
no despachaba nada: ni un intento, ni un registro, ni siquiera un 403. Por eso no había nada que
diagnosticar en los registros.
- **Contradice la guía oficial,** que dice que al agente le alcanza `roles/cloudscheduler.serviceAgent`
  y que, si falta un permiso, hay respuestas 403. La pista vino de un PR de un proyecto de terceros
  con el mismo síntoma.
- **Corrige la hipótesis del 2026-10-21,** que atribuía el problema a los PATCH de `retry_config`.
  El NOT_FOUND al reanudar el primer job, sobre un recurso interno de Google
  (`…/retryPolicies/…`), queda sin explicar.

**En Terraform.** `google_service_account_iam_member.scheduler_emite_tokens`, en
`acceso-central.tf`, al lado del job:
- La dirección del agente se arma con el número de proyecto, igual que la del agente de BigQuery
  Data Transfer (`cuentas.tf`). No hace falta `google_project_service_identity`: el agente ya
  existe, porque Google lo crea al habilitar la API.
- El job depende del permiso (`depends_on`). En un proyecto nuevo, el job se crea después de que
  el agente puede emitir el token.
- El permiso ya existía, porque se dio a mano. Lo adopta un bloque `import` que se borra una vez
  aplicado, sin recrearlo. El plan tiene que mostrar 1 importación y ningún recurso nuevo.

**Lección.** Un job de Scheduler que no deja ningún rastro no tiene un problema de destino, sino de
despacho. Por eso probar primero un job sin autenticación es lo más rápido para saber qué parte
falla. Igual que con el agente de Monitoring (2026-10-21), la documentación no alcanzó para
saber qué permisos hacían falta: hubo que probarlo en el proyecto.


## 2026-10-22 — Pruebas de carga con k6 y pgbench

Complementan la prueba de concurrencia (`scripts/prueba-concurrencia/`). Esa lanza **una ráfaga**
de N ventas simultáneas y mira qué pasa con la última unidad. Estas sostienen **una tasa**
durante un tiempo y miden latencias (p50, p95, p99), throughput, rechazos por bloqueo y si el
sistema da abasto. Están en `scripts/prueba-carga/`, y los resultados en `docs/resultados-carga.md`.

**Tasa constante (modelo abierto), no usuarios fijos.** k6 usa `constant-arrival-rate`: lanza la
tasa pedida haga falta los usuarios virtuales que haga falta, y si no le alcanzan cuenta las
iteraciones descartadas.
- *Alternativa descartada:* un número fijo de usuarios virtuales (modelo cerrado). Cuando la API
  se pone lenta, cada usuario manda menos, la carga ofrecida baja sola y la saturación se esconde
  (*coordinated omission*). Con tasa constante, la cola aparece en la latencia y en las descartadas.
- **Escalones** de 25, 50, 100, 200 y 400 ventas/s en el perfil completo (al principio eran hasta
  100/s, y no llegaban al techo), de 30 s, con 6 s entre uno y otro para que no se solapen. El
  perfil corto sigue igual: un escalón de 20/s en local, y 20 y 50/s en GCP.
- **Calentamiento de 5 s** antes de los escalones, que no entra en las tablas: el primer escalón
  medía también el arranque del JIT y del pool. Agotamiento no lo lleva, porque se llevaría las
  unidades antes del escalón medido.

**Tres escenarios de ventas,** con las dos estrategias:
- **caliente:** un solo producto, la contención máxima sobre una fila;
- **azar:** 30 productos, la línea base sin contención;
- **agotamiento:** más ventas que unidades; se tiene que vender exactamente lo que había.

Los tres escenarios usan productos distintos (31, 1 a 30 y 32): ninguno hereda la existencia de otro.
La existencia se repone antes de cada corrida con un AJUSTE por conteo (`POST /ajustes`), como en
la prueba de concurrencia: nada se escribe en la base por fuera.

**Cada corrida empieza con el outbox vacío.** En la primera corrida completa, "azar" a 100/s dio un
p95 de 313 ms, contra 6 ms de "caliente" a la misma tasa. Repetido solo, con el outbox vacío, dio
6,1 ms: la diferencia era el publicador vaciando la cola de las corridas anteriores, que comparte
la CPU con la medición. Ahora el orquestador espera a que el outbox de las 3 sucursales esté vacío
antes de cada corrida. Durante la corrida el publicador sigue trabajando en paralelo, como en
producción. Las ventas son reales, llegan al
central y la base queda cuadrando. k6 entra por la red de Docker directo a la API, sin el
reenviador de la caja que sí usa la prueba de concurrencia: así se mide la API, no el reenviador.

**Qué decide si pasa.**
- Las verificaciones contra la base (`verificacion.sql`), con una referencia propia de cada corrida:
  - documentos = ventas aceptadas por k6;
  - un movimiento y una fila de outbox por documento;
  - las unidades de movimientos, de líneas y lo vendido coinciden, y la bitácora cuadra fila por fila;
  - existencia final = inicial − vendido;
  - sobreventa 0.
- Que no haya 5xx, errores de red ni otros 4xx.
- Que el central cuadre al final.

Las latencias se informan, pero no deciden: en local dependen de lo que haga la máquina en ese
momento. *Alternativa descartada:* umbrales de latencia en k6. Fallarían por ruido de la máquina,
no por un defecto.

**pgbench, sobre una copia de la base.** `venta.sql` repite las sentencias de una venta de 1 unidad
con la estrategia PESIMISTA, en el mismo orden (`registrar-venta.ts`, `estrategia-pesimista.ts`),
incluido el outbox con un contenido de forma y tamaño parecidos.
- **Corre sobre una copia** (`pg_dump` a `carga_pgbench`, en el mismo contenedor), que se borra al
  terminar, pase lo que pase. En la base real, las filas de prueba no se podrían borrar
  (`movimiento` es inmutable), y el publicador mandaría al central un outbox falso. En la copia, la
  existencia de los productos 1 a 40 se sube a 1 000 000 con un `UPDATE` directo: es una copia
  descartable.
- **Solo PESIMISTA:** pgbench no puede repetir una transacción cuando el `UPDATE … WHERE version`
  afecta 0 filas, y esa lógica de reintento está en la aplicación.
- **Sirve como referencia:** el techo de la base para esta transacción, sin HTTP, sin Node y sin pool.
  Los percentiles salen del registro por transacción (`-l`), y la latencia por sentencia (`-r`)
  muestra cuánto se espera en el `SELECT … FOR UPDATE`.

**Local y GCP.**
- **Local:** todo comparte la misma máquina. El informe lo dice **al principio**: los números
  comparan estrategias y escenarios entre sí; no son la capacidad del sistema.
- **GCP** (`infra/gcp/carga-gcp.sh`), con un perfil corto para no gastar crédito y solo con
  PESIMISTA, la estrategia de producción:
  - k6 de ventas corre en `sucursal-2` y le pega a `sucursal-1` por IP interna, así el generador
    y la sucursal están en VMs distintas;
  - k6 de la consulta corre en Cloud Shell, con `gcloud auth print-identity-token`;
  - pgbench corre en el contenedor de la base de `sucursal-1`;
  - la imagen de k6 se copia a Artifact Registry, porque las VMs no bajan de Docker Hub.
- **El informe tiene dos partes,** separadas por marcadores (`<!-- GCP:INICIO -->`). Cada entorno
  regenera la suya y conserva la otra, porque se corren en máquinas distintas.

**Lo que corre `probar-todo.sh`:** el perfil corto, sin regenerar el informe (`--sin-informe`), para
comprobar en cada corrida que todo funciona sin pisar los resultados del perfil completo.

**Dependencias.** Ninguna en la aplicación. Se suma una imagen, `grafana/k6:1.3.0` (Docker Hub,
publicada hace 13 meses). pgbench viene en `postgres:16-alpine`, que ya se usaba.

**No probado:** `carga-gcp.sh` solo pasó el chequeo de sintaxis. Sus piezas sí corrieron en local
tal como van a correr en las VMs: `reponer.mjs` dentro del contenedor de la API, y
`verificacion.sql` y `pgbench.sh` dentro del de la base. También la parte de GCP del informe,
con datos de prueba. Se corre el día que GCP esté encendido.

**Lo que mostró la corrida completa con escalones hasta 100/s** (local, 2026-10-22, resultados en
`20261022-003908-completo`). Es la segunda corrida completa: la primera se descartó por el
problema del outbox de arriba. `docs/resultados-carga.md` ahora muestra una corrida posterior,
con escalones hasta 400/s, así que sus números difieren un poco de estos. Son números para
comparar entre sí, no capacidad.
- **Hasta 100 ventas/s, nada satura:** ni bloqueos, ni iteraciones descartadas, y el p95 queda
  entre 5 y 12 ms con las dos estrategias. En la base, la transacción tarda alrededor de 1 ms
  (pgbench con 1 cliente: p50 de 0,9 ms), así que a 100 ventas/s la fila caliente está ocupada
  cerca del 10 % del tiempo. Los escalones de 25, 50 y 100/s no llegaban al techo de la API: por
  eso el perfil completo suma 200 y 400/s (ver "Hasta 400 ventas/s", abajo).
- **La optimista rechaza ventas que la pesimista no rechaza:** 7 de 3 000 en "caliente" a 100/s,
  y 2 en "agotamiento", por conflicto de versión después de 5 reintentos. La pesimista espera su
  turno en el `SELECT … FOR UPDATE`; la optimista, bajo contención, a veces se rinde.
- **Sobreventa 0 en todas las corridas.** En "agotamiento" se vendieron exactamente las 300
  unidades, con las dos estrategias.
- **pgbench muestra el techo de una fila caliente.** Con un solo producto, la base se queda
  entre 1 000 y 1 120 TPS con 1, 5, 10 o 20 clientes, porque la fila serializa. Lo que crece es
  la espera: el p99 pasa de 2,4 ms a 86 ms. Con 30 productos al azar sube a 3 940 TPS con 10
  clientes y 3 747 con 20. El límite de una sucursal con un producto muy vendido es la fila, no
  la base, y queda muy por encima del ritmo de una caja real.
- **Consulta de existencias en el central:** 200 consultas/s con p95 de 2,7 ms.
- **La latencia baja al subir la tasa:** en todos los escenarios de ventas, el p50 pasa de
  alrededor de 8 ms a 25/s a 3 ms a 100/s. En el central, en esa corrida, bajó el p95 (de 5,0 a
  2,7 ms entre 50 y 200/s). En la corrida siguiente no bajó de forma pareja (5,0, 3,7 y 4,3 ms).
  Es lo contrario de lo esperable con una cola.
  No se investigó la causa: puede ser que, con más trabajo, la CPU de la máquina y el JIT de Node
  se mantengan más activos. Es otro motivo para no leer estos números como capacidad, y para
  repetir la medición en GCP.

**Hasta 400 ventas/s** (perfil completo con escalones de 200 y 400/s, 2026-10-22, resultados en
`20261022-104300-completo`, que es lo que muestra `docs/resultados-carga.md`). Dónde empieza a
degradar cada estrategia, en local:

| Escenario | Estrategia | Sostiene sin degradar | Primer escalón con degradación | Qué pasa |
|---|---|---|---|---|
| caliente | PESIMISTA | 200/s (p95 20 ms) | 400/s | Las acepta todas (399,4/s), pero el p95 sube a 947 ms y el p99 a 1 063 ms; 18 descartadas por k6. Ningún 409: la espera en el `SELECT … FOR UPDATE` no llega al `lock_timeout` de 3 s |
| caliente | OPTIMISTA | 100/s (p95 6,8 ms) | 200/s | 236 conflictos de versión (4 %) y p95 de 113 ms. A 400/s colapsa: 5 403 conflictos (46 %), solo 211 aceptadas/s, p95 1 823 ms y 257 descartadas |
| azar | PESIMISTA | 400/s (p95 42 ms) | — | No degrada en los escalones probados |
| azar | OPTIMISTA | 400/s (p95 38 ms) | — | No degrada en los escalones probados |

- **Con contención, la pesimista sostiene sin degradar el doble de tasa que la optimista** (200/s
  contra 100/s). Bajo carga, la optimista repite la transacción completa cada vez que pierde la
  carrera, y los reintentos suman más carga sobre la misma fila. Al duplicar la carga ofrecida
  (de 200 a 400/s), lo aceptado casi no crece (de 192 a 211/s), mientras los conflictos pasan del
  4 % al 46 % y el p95 de 113 ms a 1 823 ms: es el colapso típico del control optimista con alta
  contención. La pesimista hace cola en el bloqueo y no repite trabajo: a 400/s todavía acepta
  399,4/s, aunque con p95 de 947 ms.
- **La pesimista empieza a degradar en "caliente" entre 200 y 400/s a través de la API.** No es un
  techo de throughput, porque a 400/s acepta casi todo: lo que se dispara es la latencia (p95 de
  20 ms a 947 ms). La base sola sostiene unas 1 000 TPS sobre esa fila (pgbench). La diferencia
  es lo que agrega cada venta fuera de la base: HTTP, Node, el pool de 20 conexiones, y k6 y el
  publicador compartiendo la misma CPU.
- **Ningún 409 `PRODUCTO_BLOQUEADO` en ningún escalón.** Con 20 conexiones por API, como mucho 20
  ventas esperan la fila a la vez, y la cola no llega a los 3 s del `lock_timeout`. El resto
  espera una conexión del pool, en Node, no en la base. Por eso la degradación se ve en la
  latencia y en las descartadas, no en los 409.
- **Sin contención (azar), las dos estrategias son equivalentes** hasta 400/s.
- **Sobreventa 0 en las 6 corridas,** también en los escalones que degradan: la latencia sube y la
  optimista rechaza, pero nunca se vende de más.

## 2026-10-22 — La caja: punto de venta PWA servido por cada sucursal

Es la interfaz que compromete la propuesta: "punto de venta funcional sin conexión (PWA)". Tiene tres pantallas (Caja,
Existencias en la red y Traslados) y un selector de usuario. Está en `packages/caja`. Reemplaza lo que decía la sección
"Frontend" del 2026-10-20: ya hay pantallas, y no llaman al central desde el navegador.

**Cómo está hecha.**
- **Next.js exportado como estático** (`output: 'export'`, `basePath: '/caja'`), con Tailwind, sin librerías de componentes
  y con la tipografía del sistema, para que abra sin red.
- **Versiones exactas, todas con más de dos semanas al instalarlas:**
  - `next` 16.3.8;
  - `react` y `react-dom` 19.3.0;
  - `tailwindcss` y `@tailwindcss/postcss` 4.3.3;
  - `postcss` 8.5.29;
  - `@types/react` y `@types/react-dom` 19.3.0.
- **La sirve la misma api-sucursal en `/caja`** (`useStaticAssets` de `@nestjs/platform-express`, sin dependencias nuevas):
  mismo origen que el API y sin CORS. Cada sucursal sirve la suya, en Docker Compose y en la VM de GCP.
- *Alternativa descartada:* un servidor de Next aparte. Sería otro proceso por sucursal, y para una app estática no hace falta.

**El navegador solo le habla a su sucursal.** Lo que necesita del central pasa por api-sucursal, con la misma
autenticación que el reconciliador (`CENTRAL_AUTENTICACION`, token del metadata server en GCP). Los endpoints nuevos son
todos de lectura o de reenvío; los existentes no cambiaron.

| Endpoint (api-sucursal) | Qué hace |
|---|---|
| `GET /usuarios` | Usuarios de esta sucursal, sin correo ni contraseña (local) |
| `GET /sucursales` | Copia local de las sucursales, con cuál es esta |
| `GET /traslados/por-recibir` | Traslados en los que esta sucursal es destino y que falta recibir (local) |
| `GET /red/enlace` | Le pregunta al central si responde (siempre 200, con `en_linea`) |
| `GET /red/existencias/:codigo_barras` | Reenvía a `GET /existencias/:codigo_barras` del central |
| `GET /red/usuarios?id_sucursal=N` | Reenvía a `GET /usuarios` del central: el ENCARGADO o ADMIN que firma en el destino |
| `GET /red/traslados`, `GET /red/traslados/:id` | Reenvían al listado nuevo y al detalle del central |
| `POST /red/traslados` | Reenvía a `POST /traslados`. El origen lo fija la sucursal, no el navegador |

- **En el central:** `GET /traslados?id_sucursal=N` y `GET /usuarios?id_sucursal=N&rol=…`, los dos de solo lectura.
- **Si no se llega al central:** 503 `SIN_ENLACE_CENTRAL`.
- **Si el central no responde a tiempo una solicitud de traslado:** 504 `RESULTADO_DESCONOCIDO`. No se reintenta, porque
  pudo haberse registrado.
- **Si el central responde:** su estado y su cuerpo pasan tal cual.
- **La caja nunca le pide nada al central para vender:** la venta es `POST /ventas`, local.

**Login fuera de alcance (reemplazado el 2026-10-23: ver «Inicio de sesión con sesión firmada en la sucursal»).** No hay inicio de sesión: un selector muestra los usuarios de la sucursal (`GET /usuarios`), y
el elegido queda en el almacenamiento local del navegador. Cualquiera en la tienda puede elegir cualquier usuario. Los
permisos los sigue validando el API: solo un ENCARGADO o ADMIN solicita traslados y confirma recepciones, y la pantalla
deshabilita esos botones con el motivo. *Alternativa descartada:* contraseñas o sesiones. Exigen hash, sesión y
recuperación, que no son parte del alcance. Queda para un hito de seguridad.

**Quién firma en el destino.** `POST /traslados` pide un ENCARGADO o ADMIN de la sucursal destino, y la base de una
sucursal solo tiene sus usuarios. Por eso el central tiene ahora `GET /usuarios` de solo lectura, y la sucursal lo
reenvía.

**El indicador de enlace, siempre visible.**
- La cantidad por enviar sale de `GET /salud` (local, cada 5 s).
- "En línea / sin enlace" sale de `GET /red/enlace`, cada 15 s, porque en GCP cada consulta es una petición a Cloud Run.
- Las dos consultas corren solo con la pestaña visible (`document.visibilityState`).
- *Alternativa descartada:* tomar el enlace de `/salud`. Arranca en `DESCONOCIDO`, no se actualiza mientras el outbox
  esté vacío y, en GCP, refleja Pub/Sub, no el central.

**Sin enlace.**
- La caja vende igual.
- "Existencias en la red" dice que la consulta no está disponible.
- En Traslados se deshabilitan "Solicitar" y la lista del central, con el motivo: el 2PC necesita las dos sucursales en línea.
- "Por recibir" sigue disponible, porque la recepción es local (`POST /traslados/:id/recibir`, con su propio outbox).

**Limitación conocida: el doble cobro — CERRADA para la caja el 2026-10-27.** Cuando se escribió esta entrada, `POST /ventas`
no tenía clave de idempotencia: dos peticiones iguales eran dos ventas, y la caja solo se defendía deshabilitando «Cobrar»
hasta la respuesta y bloqueando el cobro tras una petición sin respuesta hasta que alguien confirmara que había revisado.
Ese bloqueo manual ya no existe: la caja manda una llave por intento y «Reintentar» la reusa. Ver «Llave de idempotencia en
`POST /ventas` y `POST /recepciones`» (2026-10-27). Las llamadas sin llave (scripts, k6, pgbench) conservan el comportamiento
de entonces.

**El service worker, escrito a mano** (sin next-pwa). `scripts/generar-sw.mjs` corre después de `next build` y le pone la
lista de archivos del shell.
- Al instalarse, guarda en caché todo el shell: las páginas, el JS y CSS de Next, el manifest y los íconos.
- El API queda fuera de su alcance (`/caja/`): nunca se cachea.
- "Sin conexión" quiere decir sin el central: la caja habla con su sucursal por la red de la tienda. El service worker
  agrega que la pantalla abra aunque la sucursal no responda en ese momento.
- La venta igual necesita a la sucursal. *Alternativa descartada:* una cola de ventas en el navegador. Sería un segundo
  camino de escritura, fuera de la transacción ACID local.

**Limitación conocida: el service worker en GCP.** Un service worker solo se registra en un contexto seguro (`localhost` o
HTTPS). En las VMs, la caja está en `http://10.10.0.1N:3000/caja/`: HTTP, con IP interna y alcanzable solo desde la
subred. Ahí no hay service worker. La caja funciona igual mientras la sucursal responda, y la pantalla lo avisa al pie.
- **Forma de usarla en la demo:** un túnel IAP la deja en `localhost`, donde el service worker sí se registra:
  ```bash
  gcloud compute start-iap-tunnel sucursal-1 3000 --local-host-port=localhost:3001 --zone=us-central1-a
  # http://localhost:3001/caja/
  ```
- **La solución de fondo** (HTTPS en las sucursales) no se resuelve hoy.

**Pendiente: la imagen de una sola etapa.** El `Dockerfile` compila la caja en la misma imagen que las dos APIs, así que
lleva las dependencias de Next aunque en ejecución solo se usen los estáticos de `out/`. La imagen pesa 1,39 GB, y la del
central también los lleva. Una imagen de varias etapas, que copie solo `dist/` y `out/`, lo resuelve; queda para después.

**Vulnerabilidades de `next`.** `npm audit` marca tres avisos críticos y altos para `next` de la 16.0.0 a la 16.4.0:
- GHSA-2464-fhr4-w9xw: optimización de imágenes con SVG;
- GHSA-p3m5-59mp-jv5r: Server Actions;
- GHSA-673q-mjm6-r5xr: `use cache`.

Se corrigen en la 16.4.2, que tenía 7 días al instalar, así que no cumplía la regla de las dos semanas. Los tres son del
**servidor** de Next, y con `output: 'export'` no hay servidor de Next en ejecución: la caja son archivos estáticos que
sirve Nest. Se queda la 16.3.8, y se sube a la 16.4.2 cuando cumpla dos semanas (el 2026-10-29), antes de la entrega del
4 de noviembre.

**Pruebas.**
- `scripts/prueba-caja/prueba.mjs` (en `probar-todo.sh`), 6 casos:
  - las tres sucursales sirven la caja, el manifest y el service worker;
  - la lectura local;
  - el reenvío, incluido un 404 del central que pasa tal cual;
  - un traslado solicitado desde la caja y recibido en el destino;
  - un CAJERO rechazado con 403;
  - sin enlace: la red responde 503, la caja vende y se sigue sirviendo, y al restaurar el central cuadra.
- Las pantallas se revisaron en el navegador:
  - una venta;
  - el mensaje de existencia insuficiente;
  - la antigüedad del dato en la red;
  - el estado sin enlace;
  - el service worker registrado en `localhost`.

## 2026-10-22 — "Sin enlace desde las HH:MM" es la hora en que la caja lo detectó

**Decisión.** La franja ámbar de la caja dice "Sin enlace con el central desde las HH:MM". Esa hora **no es la del corte
real**: es la primera vez que esa pantalla, al consultar `GET /red/enlace` (cada 15 s mientras está visible), recibió
`en_linea: false`. La guarda el navegador (`sin_enlace_desde` en `ProveedorCaja.tsx`) y se borra al volver el enlace.

**Consecuencias.**
- La hora puede ir hasta unos 15 s detrás del corte, o más si la pestaña estaba oculta: con la pestaña oculta no se consulta.
- Si se recarga la página o se abre otra pestaña durante el corte, la hora vuelve a empezar desde ese momento.
- Dos cajas de la misma sucursal pueden mostrar horas distintas para el mismo corte.

**Alternativa descartada: tomar la hora de la sucursal.** La sucursal no registra cuándo cayó el enlace. Lo más
parecido que tiene es `ultimo_envio_ok_en` en `/salud`, que es el último envío del outbox aceptado por el central. Pero
con la cola vacía el publicador no contacta al central (`registrar()` sale sin tocar nada si no leyó filas), así que esa
hora puede ser muy anterior al corte y haría parecer que la sucursal lleva más tiempo aislada. Registrar el instante del
corte en la sucursal es un cambio de backend que no entra en el pulido de la caja. El texto dice "desde las", y para un
cajero alcanza con saber que hace un rato que no hay enlace.

## 2026-10-23 — Réplica central → sucursal: pull con contador serializado

**Contexto.** `sucursal`, `proveedor`, `producto` y `usuario` son maestros del central y copias de solo lectura en cada
sucursal, pero hasta hoy solo llegaban por la semilla (`sembrar.sh`, y `aplicar.sh` en el arranque de las VMs). Ningún
camino propagaba un cambio: un precio nuevo o un usuario dado de baja en el central no llegaba nunca a las cajas.

**Decisión.** Réplica por **pull**. Cada sucursal pide al central `GET /replica/cambios?id_sucursal=N&desde=V&limite=L` cada
`REPLICA_INTERVALO_MS` (10 s) y aplica cada página en una transacción local. El central da a cada fila de esas cuatro
tablas una `version_replica` tomada de **un contador global** (`contador_replica`, una sola fila), y el cursor de cada
sucursal es la última versión aplicada (`estado_replica`).

**Por qué el contador está serializado.** Un cursor por timestamp o por `nextval()` pierde filas: dos transacciones toman
su número en un orden y confirman en otro. Si la sucursal ya leyó hasta la versión 10 y la transacción que tenía la 9
confirma después, esa fila queda detrás del cursor y no vuelve a llegar jamás. Con la fila de contador bloqueada hasta el
commit, las escrituras de catálogo y de usuarios se serializan: la versión N+1 solo existe cuando la N ya confirmó, y el
orden de versión es el orden de commit. El central lee el corte y las filas en una transacción `REPEATABLE READ READ
ONLY`, así que toda versión visible en ese snapshot tiene visibles a todas las menores. El costo (escrituras de catálogo
de una en una) no importa: son pocas y manuales. Ningún camino caliente del central escribe en estas tablas (se
comprobó con grep: el consumidor de sincronización, el coordinador y la recuperación solo las leen), así que el contador no
está en el camino de una venta ni de un traslado.

**El orden de bloqueo que evita el deadlock: contador → filas.** La primera versión era un trigger `BEFORE ... FOR EACH
ROW` que incrementaba el contador. Ese trigger corre cuando Postgres ya bloqueó la fila de catálogo, y eso deja un
deadlock con cualquier escritor de varias filas: T1 bloquea A y toma el contador; T2 bloquea B y espera el contador; T1
va por B y espera a T2. Se corrigió con un segundo trigger `BEFORE INSERT OR UPDATE ... FOR EACH STATEMENT` en las cuatro
tablas, que hace `SELECT ... FROM contador_replica FOR UPDATE` **antes** de que la sentencia toque ninguna fila. El de
fila queda solo para incrementar. Así nadie espera el contador con una fila de catálogo en la mano, y los bloqueos se
toman siempre en el mismo orden. `scripts/prueba-replica` lo demuestra: en 4b, T2 espera el contador sin tener la fila
(`pg_locks` y `FOR UPDATE NOWAIT`), T1 toma esa fila y pasa, y el log de `db-central` no tiene "deadlock detected"; en
4c, sin el trigger de sentencia, el mismo escenario da `40P01`, y el mismo chequeo de log sí lo ve.

**Alternativas descartadas.**
- **Cursor por timestamp o por `nextval()`:** pierde filas cuando las transacciones confirman en desorden (arriba).
- **Instantánea completa en cada ronda:** correcta con 50 productos, pero no escala al argumento de 15 sucursales con
  catálogos reales: cada ronda reenviaría todo aunque no haya cambiado nada.
- **Pub/Sub central → sucursal:** otro adaptador por variante (Compose, emulador, GCP) y sin orden garantizado entre
  mensajes. El pull reutiliza el HTTP y el token de identidad que ya existen y deja el orden en el cursor.

**`usuario.id_sucursal` es inmutable.** Un trigger rechaza cualquier `UPDATE` que lo cambie (`23514`: un empleado que
cambia de sucursal se desactiva y se crea de nuevo). La réplica de usuarios es por sucursal y **nunca borra**: si un
usuario pudiera pasar de la sucursal 1 a la 2, la 1 conservaría una copia activa que ya no le corresponde y ninguna
página le diría que la retire. Con el id fijo, un usuario vive y muere en una sola sucursal, y la baja es `activo = false`.
Lo mismo vale para el catálogo: nada se borra, se desactiva.

**Detalles.**
- **Huecos en la numeración:** un `INSERT ... ON CONFLICT DO NOTHING` consume una versión por fila propuesta aunque no
  inserte (la semilla, al repetirse). Es inofensivo: el cursor necesita orden, no continuidad.
- **Una página termina cuando `hasta` es igual a `desde`**, sin campo `hay_mas`. Cuesta una consulta vacía al final de cada
  ronda. De usuarios solo se envían los de la sucursal que pregunta, pero las versiones de los demás cuentan para el corte:
  la sucursal avanza el cursor igual.
- **Aplicación idempotente:** upsert con la guarda `version_replica IS NULL OR version_replica < EXCLUDED.version_replica` y
  `ultima_version = GREATEST(...)`: aplicar dos veces la misma página, o una vieja después de una nueva, no cambia nada.
- **Pool propio de una conexión** en el replicador, como el publicador: no le quita conexiones a las cajas, y no toca ventas.
- **`nombre_usuario` y `activo`** se agregaron a `usuario` (central y sucursal). Las filas existentes toman la parte local del
  correo, cortada a 30 caracteres, con el id al final si chocan, porque la base de GCP ya está sembrada.

**El sembrado queda como arranque en frío.** `sembrar.sh` y `aplicar.sh` siguen poniendo el catálogo y los usuarios
iniciales en una sucursal nueva, para que arranque sin red. Ya no son la fuente de verdad: la primera ronda de réplica
reemplaza lo sembrado (las filas sembradas tienen `version_replica` NULL, y la guarda las deja actualizar).

**Consecuencia: sin enlace, la sucursal sigue con el catálogo y los usuarios viejos.** Un precio cambiado o un usuario dado de
baja en el central no llega a una sucursal cortada hasta la primera ronda con enlace. La caja sigue vendiendo con el precio
local (probado en 2). `GET /salud` expone `replica.ultima_version`, `replica.ultima_replica_en` y el último error: si
`ultima_replica_en` envejece, el catálogo puede estar desactualizado.

**Riesgos que quedan abiertos (no se resuelven aquí).**
- **Los hashes de contraseña salen a cualquiera que pueda invocar el central.** El central no sabe qué sucursal llama:
  `id_sucursal` llega por query y `sa-sucursal` es una sola cuenta para las 3 VMs. Cualquier sucursal puede pedir los hashes
  (y los datos) de los usuarios de otra. Es el hueco de autenticación entre nodos que ya existía, ahora con una carga más
  sensible. Resolverlo pide una identidad por sucursal, que hoy no existe.
- **`activo` todavía no se aplica.** `verificarUsuario` en ventas, ajustes, preparar y recepción no lo mira (ventas no se
  tocó en este paso): un usuario desactivado puede seguir vendiendo hasta que el siguiente paso lo exija.
- **GCP, sin verificar y sin tocar.** Las migraciones nuevas se aplican solas: en las VMs, por `arranque.sh`, y en el central,
  por el job `migrar-central`. Falta comprobar que Datastream recoja las columnas nuevas de `sucursal`, `proveedor` y
  `producto` (la publicación es `FOR TABLE`), y que las vistas `dim_*` de BigQuery no dependan de un `SELECT *`. La migración
  también actualiza todas las filas de esas tablas una vez, lo que genera eventos de CDC. `usuario` no se replica a BigQuery.
  No hace falta variable nueva: `REPLICA_*` tienen valores por defecto, y el replicador usa `CENTRAL_URL` y
  `CENTRAL_AUTENTICACION` como el publicador.
- **Un `UNIQUE` en la copia frena la réplica.** Si una página choca con una restricción de la copia (por ejemplo, un
  `nombre_usuario` que ya existe con otro id), la transacción hace rollback, el cursor no avanza y el replicador reintenta cada
  ronda; el error queda en `/salud`.

**Pruebas.** `scripts/prueba-replica/prueba.mjs` (en `probar-todo.sh`), 7 casos: altas y cambios que llegan a donde
corresponde; sucursal sin enlace que conserva el precio viejo, vende y converge; páginas de 7 versiones sobre una base
vacía que dan una copia idéntica y no cambian al repetirse (también en orden inverso); serialización y orden de commit;
ausencia de deadlock con su control negativo; y el rechazo del cambio de `id_sucursal`.

## 2026-10-23 — Inicio de sesión con sesión firmada en la sucursal

**Contexto.** No había sesión: `id_usuario` viajaba en el cuerpo de `/ventas`, `/ajustes`, `/traslados/:id/recibir` y
`/red/traslados`, y la caja lo guardaba en `localStorage`. Cualquiera podía firmar una venta con el usuario que quisiera, y
`usuario.hash_contrasena` valía `SIN_HASH_HITO_1`. La réplica central → sucursal (2026-10-23) ya baja a cada sucursal sus
usuarios, sus hashes y `activo`, que es lo que hace posible validar el login sin red.

**Decisión.** Login contra la base **local** de la sucursal (`POST /sesion`), token firmado en una cookie httpOnly, y un guard
global que exige sesión en toda ruta salvo `POST/DELETE /sesion`, `GET /salud` y las del coordinador de traslados
(`@RutaDeServicio()`, con su comentario: su autenticación entre nodos es un paso posterior). En cada petición protegida el
guard verifica la firma y el vencimiento, que el token sea de **esta** sucursal, y lee el usuario de la base local exigiendo
`activo`. El rol sale de la base, no del token. El usuario de una venta, un ajuste, una recepción o una solicitud de traslado
sale de la sesión; un cuerpo con `id_usuario` o `id_usuario_origen` responde 400 en vez de ignorarse (un cliente que cree
que está eligiendo el usuario está equivocado y conviene que lo sepa). La verificación de usuario de ventas del núcleo exige
además `activo`, como defensa en profundidad.

**Contraseñas: scrypt nativo en vez de bcrypt o argon2.** `node:crypto` trae scrypt: no agrega dependencias. Se usa la
versión **asíncrona** (`scrypt`, nunca `scryptSync`): la síncrona bloquea el event loop durante todo el cálculo y frenaría las
ventas de las otras cajas de la sucursal. Sal aleatoria de 16 bytes, parámetros por defecto de Node (N=16384, r=8, p=1),
formato `scrypt$N$r$p$sal_b64$hash_b64` (los parámetros viajan en el hash, así que se pueden subir sin invalidar los viejos),
comparación con `timingSafeEqual`. *Alternativas descartadas:* bcrypt y argon2 (una dependencia nativa cada una, con
compilación en la imagen de Docker y en las VMs), y PBKDF2 (más barato de atacar con GPU que scrypt).

**Sin revelar qué usuarios existen.** Si el usuario no existe, está inactivo, o su hash no tiene formato válido, igual se
ejecuta un scrypt contra un hash ficticio, y la respuesta es la misma 401 «Usuario o contraseña incorrectos»: ni el mensaje ni
el tiempo distinguen los casos. La única excepción es la cuenta **bloqueada con la contraseña correcta**, que responde 423: quien
conoce la contraseña no está adivinando y necesita saber por qué no entra; con una contraseña mala, un bloqueo no se revela.

**Token HMAC propio en vez de JWT.** `base64url(payload).base64url(HMAC-SHA256)` con `id_usuario`, `id_sucursal`, `emitido` y
`vence`. No agrega dependencias (JWT pediría una librería, o implementar a mano lo mismo con más superficie: `alg`, `kid`,
`none`), y el formato es nuestro, así que lo único que el token afirma es «esta sucursal emitió esto». Lo que cambia con el
tiempo (si el usuario sigue activo, su rol) **no** va en el token: se lee de la base en cada petición, de modo que una baja o un
cambio de rol que llega por la réplica surte efecto en la siguiente petición, sin esperar a que venza la sesión. `SESION_CLAVE`
es obligatoria (mínimo 32 bytes, la API no arranca sin ella) y distinta por sucursal: un token de la 1 no sirve en la 2 ni
aunque alguien conociera la clave de la 1. La cookie es `HttpOnly; SameSite=Strict; Path=/`, y `Secure` solo si la petición
llegó por HTTPS. Su **nombre lleva el id de la sucursal** (`sima_sesion_s1`): las cookies no se separan por puerto y en
desarrollo las tres sucursales viven en `localhost:3001-3003`; con un nombre único, iniciar sesión en una pisaría la de otra.
*Consecuencia asumida:* el token no tiene estado en el servidor, así que **cerrar sesión solo lo quita del navegador**; un
token copiado sigue valiendo hasta `SESION_DURACION_H` (10 h) o hasta que el usuario se dé de baja. Revocarlo pediría una
tabla de sesiones, que cada petición tendría que consultar; la baja del usuario ya cubre el caso grave.
`DELETE /sesion` es pública a propósito: con la sesión vencida igual hay que poder limpiar la cookie.

**Login validado en la sucursal.** Funciona sin red: es lo que exige el requerimiento principal (la tienda factura aunque no
tenga internet). *Alternativa descartada:* validar contra el central, que haría que no se pudiera abrir la caja sin enlace.
**Consecuencia:** una baja hecha en el central **no surte efecto en una sucursal sin enlace hasta la siguiente réplica**. Un
usuario dado de baja puede seguir entrando y vendiendo en una sucursal aislada mientras dure el aislamiento (tanto como el
corte, más hasta 10 s). Es el mismo límite de la réplica (2026-10-23) y se acepta; `GET /salud` expone
`replica.ultima_replica_en` para saber cuánto hace que la sucursal habló con el central.

**Bloqueo por intentos, por sucursal.** La tabla `intento_acceso` es local y no se replica. Cinco contraseñas malas bloquean
la cuenta `SESION_BLOQUEO_MIN` minutos (15). **Un atacante puede bloquear a un cajero 15 minutos** equivocándose a propósito
con su usuario: se acepta y se declara. Bloquear por usuario y no por origen es lo que impide adivinar contraseñas sin límite;
bloquear por IP no sirve detrás del reenviador de la tienda. Un administrador no queda inmune: la caja de un ADMIN también se
puede bloquear. Reglas finas: `bloqueado_hasta` se fija **solo cuando `fallidos` cruza el umbral**; los fallos durante el
bloqueo **cuentan pero no lo extienden** (si lo extendieran, seguir intentando mantendría bloqueado a un cajero para siempre);
al vencer el bloqueo, el contador **reinicia en 1** (si no, un solo error tras los 15 minutos lo bloquearía de nuevo); un login
correcto borra la fila; no se registran intentos de usuarios que no existen (la FK lo impide). El bloqueo es por sucursal:
fallar 4 veces en la 1 y 4 en la 2 no bloquea a nadie.

**El contador de fallos es un upsert atómico (lost update).** Un `SELECT fallidos` seguido de un `UPDATE fallidos = n + 1`
pierde fallos cuando dos intentos llegan a la vez: los dos leen 3 y los dos escriben 4. El fallo se cuenta con **un solo**
`INSERT … ON CONFLICT (id_usuario) DO UPDATE SET fallidos = intento_acceso.fallidos + 1 … RETURNING`; el motor serializa los
conflictos sobre la fila y cada intento parte del valor que dejó el anterior. Todas las comparaciones de tiempo usan el reloj
de la base (UTC), no el de Node. `scripts/prueba-sesion` lanza 10 intentos simultáneos y exige `fallidos = 10` exactos.

**Baja lógica forzada por trigger (solo en el central).** `BEFORE DELETE` por fila y `BEFORE TRUNCATE` por sentencia en
`sucursal`, `proveedor`, `producto` y `usuario` rechazan el borrado con `restrict_violation` («la baja es lógica»), la misma
familia que la bitácora de `movimiento` (la API lo ve como 409 `RESTRICCION_BD`). La razón es la réplica: **la sucursal nunca
borra** (aplica por upsert), así que un `DELETE` en el central dejaría una copia viva y divergente en cada sucursal, y ninguna
página le diría que se retire. Antes de ponerlo se buscó en scripts, pruebas y migraciones: nada borra en esas tablas.
`TRUNCATE` lleva trigger propio porque no dispara los de fila. *Límite:* un superusuario puede desactivarlos
(`ALTER TABLE … DISABLE TRIGGER`); quitarle ese poder a la aplicación es parte del hito de endurecimiento. *Hueco del modelo:*
`proveedor` no tiene columna `activo`, así que hoy un proveedor no se puede dar de baja de ninguna manera; agregarla es una
migración y un cambio en la réplica, y no era parte de este paso.

**`nombre_usuario` siempre en minúsculas.** El login normaliza lo que se escribe, y un `CHECK (nombre_usuario = lower(nombre_usuario))`
en el central y en la sucursal impide que existan dos formas de escribir el mismo usuario. Las filas existentes se
normalizan en la migración (en el central solo las que lo necesitan, para no subirle la `version_replica` a todas).

**Semilla.** `scripts/semilla/generar-hashes.mjs` regenera `02_usuarios.sql` con hashes reales. La contraseña de cada usuario de
demostración es `[REDACTADO]` (README). Una base ya sembrada tiene el marcador `SIN_HASH_HITO_1`: la semilla lo
reemplaza (`ON CONFLICT DO UPDATE … WHERE hash_contrasena = 'SIN_HASH_HITO_1'`) y no toca un hash que ya sea otro. En el
central esa actualización sube la `version_replica` y baja sola a las sucursales.

**Pendiente, a propósito.**
- `id_usuario_destino` de `POST /red/traslados` sigue viniendo del cuerpo: es el firmante del **destino**, otro usuario de otra
  sucursal, no sale de la sesión de esta caja. Se corrige en el paso de roles (está comentado en el código). **Resuelto el 2026-10-23**: el traslado lo pide el destino y el solicitante sale de la sesión (ver «Roles, denegar por defecto…»).
- Los chequeos de rol de ajustes y recepción quedan como estaban; la matriz completa de roles es el paso siguiente. **Resuelto el 2026-10-23**: matriz de roles en el servidor; esos chequeos quedan como defensa en profundidad.
- Las rutas del coordinador (`preparar`, `confirmar`, `abortar`, `reconciliacion/ejecutar`) siguen sin autenticación: es el
  hueco entre nodos que ya estaba documentado (2026-10-19), no se agranda ni se cierra aquí.
- **GCP, sin tocar.** La API no arranca sin `SESION_CLAVE`: las VMs necesitan una clave por sucursal en **Secret Manager**
  antes de desplegar esta versión. `infra/gcp/carga-gcp.sh` sigue pasando `ENCARGADO=2` (un id) a `reponer.mjs`, que ahora
  espera un `nombre_usuario` (`encargado1`).

**Pruebas.** `scripts/prueba-sesion/ejecutar.sh` (en `probar-todo.sh`), 11 casos con plazos acortados por variable
(`scripts/prueba-sesion/compose.sesion.yml`): login y cookie; mismo 401 para contraseña mala y usuario inexistente; bloqueo con
423 y reinicio del contador; 10 intentos simultáneos = 10; token alterado y vencido; token de otra sucursal (incluido uno bien
firmado con la clave de la otra); baja en el central y 401 tras la réplica; `id_usuario` en el cuerpo = 400 y la venta registra
al usuario de la sesión; login y venta con el enlace cortado; `DELETE`/`TRUNCATE` rechazados en el central; el `CHECK` de
minúsculas. Los demás scripts (`prueba-caja`, `prueba-ajuste`, …) inician sesión con `scripts/lib/sesion.mjs`.

## 2026-10-23 — Roles, denegar por defecto y traslados iniciados por el destino

**Contexto.** Con el inicio de sesión ya había usuario en cada petición, pero cualquier rol podía llamar a cualquier ruta
(solo ajuste y recepción comprobaban el rol dentro del núcleo), y el traslado lo pedía el origen eligiendo el firmante del
destino desde su propia caja: un usuario de otra sucursal, que no salía de la sesión. Este paso cierra las dos cosas.

**El traslado lo inicia el destino.** La sucursal donde está la sesión es el DESTINO: pide mercadería a otra. El cuerpo es
`{ id_sucursal_origen, lineas }`; el destino y el solicitante los pone el nodo (el destino es él mismo, el solicitante es la
sesión, ENCARGADO o ADMIN). Un cuerpo con `id_sucursal_destino`, `id_usuario_destino`, `id_usuario_origen` o
`id_usuario_solicitante` da 400 en lugar de ignorarse. El central valida: origen ≠ destino, ambas sucursales activas, y que
el solicitante sea un usuario activo del destino con rol ENCARGADO o ADMIN. *Alternativa descartada:* que lo envíe el origen
(el modelo anterior). Obligaba a que la caja del origen eligiera a un firmante de otra base, y ese dato no podía salir de la
sesión ni validarse localmente. *Consecuencia que se declara:* el voto del origen es **automático según la existencia**:
nadie del origen aprueba nada. La cadena es dueña del inventario, así que un ENCARGADO de una sucursal puede pedir
mercadería a otra sin que la otra lo apruebe; el origen sólo puede responder NO si no le alcanza el stock o el producto está
bloqueado. Si el negocio quisiera una aprobación del origen sería otro flujo (otra fase antes del 2PC), no un parche a este.
Sigue pendiente, como ya estaba documentado (2026-10-19), que el central confíe en el `id_usuario_solicitante` que le manda
la sucursal: la autenticación entre nodos es un paso posterior.

**`documento.id_usuario` es NULL solo en el TRASLADO del origen.** En el documento TRASLADO del origen nadie del origen
actuó, y la base de la sucursal no tiene usuarios de otras sucursales (`usuario` es un fragmento: solo los suyos). Por eso
`id_usuario` admite NULL con `CHECK (id_usuario IS NOT NULL OR tipo = 'TRASLADO')`, en la sucursal (012) y en el central
(013, porque el documento llega por sincronización y `aplicar-mensaje` lo inserta tal cual). *Límite declarado:* el CHECK
permite NULL en cualquier TRASLADO, también en el del destino; que solo el del origen lo lleve lo exigen el participante (el
destino exige solicitante) y `validar-mensaje` (NULL solo con `traslado.rol = 'ORIGEN'`; en lo demás, entero positivo). El
documento del destino lleva al solicitante. *Alternativas descartadas:* (a) replicar a cada sucursal los usuarios de todas
las demás, para que el FK exista: rompe el reparto de tablas (cada sucursal guarda solo sus usuarios y hashes) y amplía lo que
queda expuesto si una sucursal se compromete, para firmar un documento que nadie firmó; (b) un usuario «sistema» por
sucursal: inventa una identidad que no actuó y contamina cualquier análisis por usuario. El NULL dice la verdad.

**Cambio de columnas en el central (013).** `traslado.id_usuario_destino` **no se renombra**: en las filas anteriores esa
columna es el firmante del destino, que no fue quien solicitó (con el flujo viejo solicitaba el origen), y un rename falsearía
la historia (además, Datastream se comporta mal con un rename). Se agrega `id_usuario_solicitante` (FK a `usuario`), se rellena
en las filas existentes con `documento.id_usuario` del documento del coordinador (quien de verdad inició), se pone `NOT NULL`
(si alguna queda NULL la migración falla con un mensaje, no inventa el dato) y se elimina `id_usuario_destino`. **En los
traslados anteriores a este cambio el solicitante es un usuario del origen**; la regla «el solicitante pertenece al destino»
rige desde la migración. El documento del coordinador de un traslado nuevo se escribe con `id_sucursal` = destino (la
sucursal del solicitante) e `id_usuario` = solicitante. Antes de cambiarlo se buscó quién lee `documento.id_sucursal` de los
TRASLADO del central como si fuera el origen: `verificar.mjs` (todas sus consultas al central filtran por
`id_documento_origen IS NOT NULL` o por `id_traslado`, o sea, las copias), `consultar-existencias-red.ts` (filtra por
`id_documento_origen IS NOT NULL`), `consultarTraslado`/`listarTraslados` (parten de `traslado.id_sucursal_origen`) y
`infra/gcp/bigquery` (el hecho sale de `movimiento` con `LEFT JOIN` a `documento` por `id_documento`; el documento del
coordinador no tiene movimientos). Nada lo usaba como origen; no hubo que ajustar nada.

**Analítica (solo reportado, GCP no se toca).** Ninguna vista ni consulta de `infra/gcp/bigquery` hace `INNER JOIN` con
`usuario` ni existe una dimensión de usuario, así que un `id_usuario` NULL en `documento` no hace perder filas. Dos cosas
para quien despliegue en GCP: Datastream copia `traslado`, y quitar `id_usuario_destino` y agregar `id_usuario_solicitante`
es un cambio de esquema en la fuente que hay que revisar en la tabla de BigQuery (no se probó aquí); y
`infra/gcp/README.md` (línea 54) todavía muestra el cuerpo viejo de `POST /traslados`.

**`traslado_local`: la sucursal sabe su papel.** Antes la recepción adivinaba que era destino porque su documento no tenía
movimientos, y sacaba al origen con una regex sobre `documento.referencia`. Ahora existe `traslado_local (id_documento PK,
rol ORIGEN|DESTINO, id_sucursal_contraparte)`. La fila se escribe **dentro de la transacción que se deja con PREPARE
TRANSACTION**: es lo que la mantiene atómica con el 2PC. Si el coordinador decide `ROLLBACK PREPARED`, la fila desaparece
junto con el documento, las líneas, el movimiento y el outbox; si decide `COMMIT PREPARED`, aparece con ellos. Nunca hay un
documento de traslado sin papel ni un papel sin documento. Escribirla fuera (antes del `PREPARE` o después del `COMMIT
PREPARED`) dejaría una ventana en la que el rollback no la deshace. `recepcion.ts` y `GET /traslados/por-recibir` leen de
esa tabla; se eliminaron la inferencia y la regex. Los traslados que ya existían se rellenan **una sola vez** en la migración
012 con la regla anterior (tiene movimientos → ORIGEN, si no → DESTINO; la contraparte, de la referencia); después la regla no
vive en el código. La migración 012 empieza con un bloque `DO` que **aborta con un mensaje claro si `pg_prepared_xacts` tiene
filas**: una transacción preparada con el código anterior no escribiría su fila y, al confirmarse, dejaría un traslado sin papel.
Se probó contra una base con datos del formato anterior (aborto con una preparada; relleno ORIGEN/DESTINO correcto).
*Costo:* `registrarRecepcion` ya no consulta `movimiento` por `id_documento`, que era la consulta que justificaba
`idx_movimiento_documento` (2026-10-13). Se retiró la medición H2 de `prueba-indices`; el índice se deja (borrarlo es otra
decisión) pero hoy ninguna consulta de la aplicación lo usa.

**Denegar por defecto.** Cada ruta de api-sucursal declara `@Roles(...)`, `@Publica()` o `@RutaDeServicio()`. Una ruta sin
declaración responde `403 RUTA_SIN_DECLARAR` (la declaración se revisa antes que la sesión: con o sin sesión, igual), y
además la API **no arranca** si algún controlador tiene una ruta sin declaración o con dos (`verificar-rutas.ts` lee la
metadata de los controladores sin instanciarlos): una ruta nueva no queda abierta —ni cerrada sin que nadie lo note— por
olvido. `@Roles` lleva una lista explícita (ADMIN también se escribe), y el rol que cuenta es el de la base local en esa
petición, no el del token. Matriz: CAJERO: ventas, existencias locales, `/red/existencias`, `/red/enlace`, `/sucursales` y
`GET /sesion`; ENCARGADO: lo del cajero más `POST /ajustes`, solicitar y ver traslados de su sucursal, `por-recibir` y recibir;
ADMIN: todo, más `GET /usuarios`. `GET /sucursales` se agregó al CAJERO (la caja lo lee al abrir sesión para el nombre de la
sucursal). `GET /red/traslados/:id` responde 404 si la sucursal no es el origen ni el destino (antes reenviaba cualquier id).
`GET /red/usuarios` **se eliminó**: solo existía para elegir el `id_usuario_destino` y ya no hay nada que elegir; el
`GET /usuarios` del central se **queda**, porque el panel de empleados del paso siguiente lee de él. `GET /usuarios` de la
sucursal pasa a ADMIN (la caja no lo usa). Los chequeos de rol del núcleo (ajuste, recepción, preparación del destino) se
quedan como defensa en profundidad. La PWA oculta Traslados al CAJERO y deja de consultar `por-recibir`: es solo presentación,
el servidor ya lo impide. El menú no tiene pantalla de Ajustes, así que no había nada que ocultar ahí.

**Pruebas.** `scripts/prueba-roles/ejecutar.sh` (en `probar-todo.sh`), 7 casos: la matriz completa ruta × rol con el código
esperado en una tabla del script (los POST con cuerpo vacío esperan 400: pasan el guard y el servicio los rechaza sin
escribir), la verificación de que ninguna ruta registrada queda sin declaración y de que la tabla las cubre todas; un CAJERO
recibe 403 en ajustes, solicitar y recibir; el ENCARGADO del destino solicita y queda COMPROMETIDO con el documento del origen
sin usuario, el del destino con el solicitante y `traslado_local` correcto en las dos puntas (y el central igual); recibir
desde el origen se rechaza y la decisión sale de `traslado_local` (cambiando el papel del destino a ORIGEN, ni él recibe);
origen sin existencia → ABORTADO y ninguna fila de `traslado_local`; el CHECK rechaza `id_usuario` NULL fuera del TRASLADO y
el validador de mensajes solo lo deja pasar en el TRASLADO del origen; cuerpos con campos que no se eligen → 400 sin escribir.
`demo-2pc` (los 6 escenarios), `prueba-coordinador`, `prueba-participante`, `prueba-reconciliador`, `prueba-caja` y
`prueba-sesion` se adaptaron al flujo nuevo.


## 2026-10-24 — Panel de empleados y renombrar sucursales (ADMIN)

**Contexto.** Los usuarios y las sucursales son maestros del central y bajan a las sucursales por la réplica (contador
serializado, migración central 011). Hasta ahora nada los escribía desde una API. Este paso agrega la pantalla «Empleados» de
la caja y los endpoints que la sostienen: el ADMIN crea empleados, cambia nombre, puesto y contraseña, da de baja y reactiva,
y renombra sucursales. Nunca se borra a nadie (la baja es lógica; el trigger de la migración 012 ya lo impide).

**El ADMIN administra toda la red.** `GET/POST/PATCH/PUT /red/empleados…` y `PATCH /red/sucursales/:id` de la sucursal son
solo ADMIN y reenvían al central (`/empleados`, `/sucursales/:id`), que escribe en el maestro. El ENCARGADO ve a los
empleados de su sucursal en lectura (`GET /usuarios` local pasó de ADMIN a ENCARGADO y ADMIN y funciona sin enlace); el
CAJERO no ve la pantalla. *Alternativa descartada:* que cada ADMIN administre solo su sucursal. Los usuarios de otra
sucursal son maestros del mismo central y el costo de restringirlo era una regla más sin un riesgo que la justifique.

**El hash se genera en la sucursal y el texto plano nunca viaja al central.** El navegador manda `contrasena` a su sucursal;
esta valida la política (8 a 1024 caracteres; el máximo es el que acepta el inicio de sesión: una contraseña más larga no
podría entrar), genera el hash con el mismo módulo de scrypt del login y reenvía solo `hash_contrasena`. El central rechaza con
400 un cuerpo que traiga `contrasena` y un hash que no tenga la forma `scrypt$N$r$p$sal$hash`. *Alternativa descartada:*
hashear en el central. Obligaba a que la contraseña cruzara la red entre nodos, y el hash ya se calcula con el scrypt
asíncrono de la sucursal, que es donde se verifica.

**Última baja de ADMIN: el bloqueo del contador va antes de leer al actor.** En toda la red tiene que quedar al menos un
ADMIN activo. Con dos ADMIN, A y B, dándose de baja el uno al otro al mismo tiempo, en READ COMMITTED cada transacción lee
«el actor es ADMIN activo» (y «queda otro») con datos que la otra todavía no confirmó, y cada una actualiza una fila distinta:
write skew, cero ADMIN. El orden dentro de la transacción es obligatorio: (1) `SELECT … FROM contador_replica FOR UPDATE`, el
mismo bloqueo que ya serializa toda escritura de usuario; (2) validar al actor (existe, activo, ADMIN); (3) `AUTO_BAJA` y el
conteo; (4) el UPDATE. El trigger de sentencia de `usuario` toma el contador recién en el UPDATE, es decir, después de esas
lecturas, y por eso no evita nada. Con el bloqueo primero, la segunda transacción espera; cuando la primera confirma, la lectura
del actor (una sentencia nueva, que en READ COMMITTED ve lo ya confirmado) lo encuentra dado de baja y responde
**403 PERMISO_INSUFICIENTE**. *Lo que protege del write skew es esa lectura del actor DESPUÉS del bloqueo.*
El conteo de `ULTIMO_ADMIN` (409) queda como defensa en profundidad para caminos que no pasen por esta API y **hoy no se
alcanza**: el actor es un ADMIN activo distinto del objetivo (si fuera el mismo, ya salta `AUTO_BAJA`), así que siempre queda
al menos él. Un ADMIN no puede darse de baja ni quitarse el rol a sí mismo (409 `AUTO_BAJA`).
*Alternativas descartadas:* SERIALIZABLE con reintentos (también es correcta, pero obliga a reintentar en el cliente y el
reintento tendría que viajar por la sucursal) y REPEATABLE READ (en PostgreSQL sigue permitiendo el write skew: es aislamiento
por snapshot). *Costo:* las escrituras de empleados y de sucursales se serializan entre sí y con las de catálogo; son pocas y
manuales, el mismo costo que ya acepta la réplica.

**`usuario.correo` pasa a admitir NULL** (migraciones central 014 y sucursal 013). El panel no pide correo y el inicio de sesión
usa `nombre_usuario`; dejarlo `NOT NULL` obligaba a inventar uno por empleado. El UNIQUE se queda: PostgreSQL admite varios NULL
en una columna UNIQUE, así que solo los correos que existan siguen sin poder repetirse. La réplica ya copiaba
`correo` tal cual; el tipo de la fila replicada pasó a `string | null`.

**`nombre_usuario`.** Se normaliza a minúsculas y debe cumplir `^[a-z0-9._-]{3,30}$`; el CHECK de la migración 012 ya exige
minúsculas. Un nombre repetido, aunque difiera en mayúsculas, responde 409 `NOMBRE_USUARIO_EN_USO`: el chequeo se hace con el
contador tomado (nadie más escribe usuarios), y la violación del UNIQUE queda como respaldo.

**`GET /usuarios` del central se eliminó.** Su único consumidor previsto era el panel de empleados y ahora lo reemplaza
`GET /empleados[?id_sucursal=]`, que trae también `nombre_usuario` y `activo` y no exige filtrar por sucursal. Nadie más lo usaba.

**Los cambios llegan con la réplica y no se aplican en una sucursal sin enlace.** El central es el único que escribe; cada
sucursal lo ve en la siguiente ronda (10 s por defecto). La pantalla lo avisa tras cada cambio. Sin enlace, las acciones del
ADMIN se deshabilitan con el motivo y la sucursal responde 503 `SIN_ENLACE_CENTRAL`; la caja sigue vendiendo con los usuarios
que ya tiene. A diferencia de solicitar un traslado (504 `RESULTADO_DESCONOCIDO` si el central no responde a tiempo), aquí un
tiempo agotado también es 503: repetir cualquiera de estas operaciones es seguro (un alta repetida da 409, lo demás es
idempotente), así que no hace falta el aviso de «pudo haberse registrado».

**Sesiones.** Cambiar la contraseña **no** invalida las sesiones abiertas hasta que vencen (el token es sin estado y firmado,
ver «Inicio de sesión con sesión firmada»). Una baja sí corta la sesión: el guard lee `activo` de la base local en cada
petición y responde 401 en cuanto la réplica la trae. Un cambio de rol también surte efecto en la siguiente petición: el rol sale
de la base, no del token.

**Modelo de confianza.** El central confía en el actor que afirma la sucursal (`id_usuario_actor`): verifica que exista, esté
activo y sea ADMIN, pero no puede comprobar que esa persona tenga una sesión. Es el mismo modelo ya declarado para el
solicitante de un traslado; la autenticación entre nodos es un paso posterior. La sucursal nunca acepta el actor del
navegador: un cuerpo con `id_usuario_actor` o con cualquier `id_usuario` da 400.

**Pruebas.** `scripts/prueba-empleados/prueba.mjs` (en `probar-todo.sh`), 11 casos: alta con entrada en la sucursal correcta y no
en otra; nombre repetido (también con mayúsculas); cambio de rol con la misma sesión; cambio de contraseña (la vieja falla, la
nueva entra, la sesión abierta sigue); baja y reactivación; un solo ADMIN (`AUTO_BAJA` al darse de baja o quitarse el rol, 403 al
actor dado de baja); write skew con dos ADMIN dándose de baja a la vez (dos rondas: un 200, un 403, un ADMIN al final);
403 para CAJERO y ENCARGADO en las rutas de escritura y lectura local del ENCARGADO; enlace cortado (503 y la venta sigue);
renombrar la sucursal 3 (se ve en su `/salud`); cuerpos con `contrasena`, hash inválido o `id_usuario*`. Las pruebas esperan la
réplica mirando la base de la sucursal y no el login: probar el login con una contraseña que todavía no llegó cuenta fallos y
bloquea la cuenta. **Control del write skew:** en una base descartable de `db-central`, la misma secuencia (leer al actor,
contar, escribir, con una pausa en medio) en dos sesiones `psql` simultáneas **sin** el bloqueo deja **cero** ADMIN, y **con** el
bloqueo deja uno: sin el bloqueo la prueba sí fallaría. La base se borra al terminar. Los casos 6 y 7 crean dos ADMIN propios
(los ADMIN de la semilla cuentan en la red), dan de baja a admin1–3 y, pase lo que pase, un bloque `finally` los reactiva y
espera la réplica; los empleados de la prueba quedan de baja (los usuarios no se borran). `prueba-roles` suma las cinco rutas
nuevas a su matriz ruta × rol.


## 2026-10-25 — Ver ventas (solo lectura)

**Contexto.** Hasta ahora una venta solo se registraba; no había dónde verlas. Este paso agrega la pantalla «Ventas» de la
caja y sus endpoints: el ENCARGADO ve las ventas de su sucursal (base local, sin enlace), el ADMIN además las de cualquier
sucursal desde el consolidado del central, y el CAJERO no ve ventas (403 en el servidor, sin entrada en el menú). Nada de esto
escribe: es lectura, y una venta sigue sin tocar nunca al central.

**Consulta operativa, con tope de 31 días; no es analítica.** Responde «qué se vendió, quién y cuándo» en un rango corto, sobre
`documento` y `linea_documento`, con índice por `(id_sucursal, fecha_hora)`. La rotación, el capital dormido y los reportes por
periodo se resuelven en BigQuery (regla de la tabla de garantías: la analítica nunca se resuelve contra la base operativa). Sin
tope, una pantalla de ventas se volvería, de a poco, el reporte del año y recorrería media tabla operativa de una sucursal que
tiene que seguir vendiendo. Un rango de más de 31 días responde 400, también en el central. *Alternativa descartada:* resúmenes
por mes o por producto en esta pantalla; eso es BigQuery y repetirlo aquí duplica un criterio de cálculo en dos lugares.

**Días de Guatemala sobre TIMESTAMP sin zona.** `documento.fecha_hora` es `TIMESTAMP` sin zona y guarda UTC (convención del
proyecto, la de `conexion.ts`). Un día para el usuario es un día de Guatemala (UTC-6, sin horario de verano): de las 06:00 UTC
de ese día a las 06:00 UTC del siguiente. La conversión es `(día)::timestamp AT TIME ZONE 'America/Guatemala' AT TIME ZONE
'UTC'` en SQL, con límite superior exclusivo; nunca se lee `fecha_hora` como hora local ni se hace aritmética de fechas con
`Date` de Node. Una venta de las 23:30 de Guatemala (05:30 UTC del día siguiente) cae en su día. Es el error que ya hubo en una
prueba que leyó horas locales como UTC; está comentado en `consultar-ventas.ts`. La pantalla formatea las horas con
`America/Guatemala` y no con la zona del navegador.

**Paginación por llave, no OFFSET.** Orden `(fecha_hora DESC, id_documento DESC)`, 50 por página; cada página devuelve
`siguiente` (la llave de su última fila) y la petición siguiente la manda como `antes_de`: `WHERE (fecha_hora, id_documento) <
llave`. Con OFFSET, una venta nueva entre dos páginas desplaza todo y la página siguiente repite una venta, y el costo de la página
N crece con N porque el motor recorre y descarta N×50 filas. La llave retoma exactamente después de la última fila vista: no
repite ni salta aunque entren ventas nuevas (las nuevas son más recientes que la llave y quedan fuera) y el desempate por
`id_documento` cubre las ventas con la misma `fecha_hora`. La llave se arma en SQL con los microsegundos completos de
PostgreSQL: pasarla por un `Date` de JavaScript (milisegundos) redondearía y saltaría o repetiría filas. La prueba lo comprueba
con 122 ventas reales (una más llega entre dos páginas) y con 120 ventas de exactamente la misma fecha en una base descartable.
*Costo:* no se puede saltar a «la página 7»; la pantalla solo ofrece «Cargar más», que es lo que necesita.

**El consolidado del central y `vigente_al`.** `GET /ventas` del central lee las copias consolidadas de `documento` y
`linea_documento` (con el nombre del usuario del maestro) y es de consistencia eventual: la respuesta trae siempre `vigente_al`
y `antiguedad_segundos`. No se inventó otra fuente: es la misma de la consulta de existencias de la red, la fecha del último
documento de la sucursal ya aplicado en el central (o, si no llegó ninguno, la última sincronización de existencias). *Alternativa
descartada:* `MAX(procesado_en)` de `inbox_idempotencia` filtrado por sucursal; `inbox_idempotencia` solo tiene índice por su
llave primaria y crece con cada mensaje, así que sería una lectura secuencial de la tabla de mensajes en cada consulta, mientras
que el último documento sale de `idx_documento_suc_fecha` recorrido hacia atrás con `LIMIT 1`. *Límite que se declara:* una
sucursal sin actividad parece vieja aunque su enlace esté sano, porque el dato se mueve con los documentos y no con el latido.
El id de una venta en el consolidado es el del central; la respuesta trae también `id_documento_origen`, el que ve la sucursal.

**Quién ve qué.** Las rutas de la sucursal son `@Roles('ENCARGADO','ADMIN')` (`GET /ventas`, `/ventas/resumen`,
`/ventas/:id_documento`) y `@Roles('ADMIN')` (`/red/ventas`, `/red/ventas/:id_documento`, que reenvían al central y sin enlace
responden 503 `SIN_ENLACE_CENTRAL`). El detalle de una venta que no es de la sucursal (o que no es una VENTA) responde 404 como
si no existiera. El resumen del día solo existe para la sucursal propia (base local): el central no tiene un resumen y no se
agregó uno; al mirar otra sucursal la pantalla muestra la lista y el «Actualizado hace X», sin resumen. Sin enlace, el selector
de sucursal de la pantalla deja solo la propia, con el motivo.

**El resumen excluye las ventas anuladas** (`estado <> 'ANULADO'`) y suma en centavos enteros para no acumular error de punto
flotante. Hoy ningún flujo anula ventas; la lista sí las muestra, marcadas.

**Pruebas.** `scripts/prueba-ventas/prueba.mjs` (en `probar-todo.sh`), 9 casos: el ENCARGADO ve las ventas recién hechas con el
nombre del cajero y su detalle; 403 para CAJERO (y para ENCARGADO en `/red/ventas`); el resumen cuadra al centavo con la suma de
`documento.total` calculada en la base con otra fórmula (pasa `fecha_hora` a hora de Guatemala y agrupa por fecha local), total y
por método; la paginación recorre más de 50 ventas sin repetir ni saltar, con una venta nueva entre dos páginas; un detalle de
otra sucursal responde 404; el ADMIN de la 1 ve las ventas de la 2 por el central con `vigente_al` y el consolidado paginado
coincide con la base de la sucursal; con el enlace cortado el ENCARGADO sigue viendo todo y `/red/ventas` da 503; el tope de 31
días (31 exactos pasa, 32 no); y el borde de las 23:30. Este último ejecuta el SQL real contra **bases descartables** de
`db-central` con el esquema de la sucursal y del central (se aplican sus migraciones) y fechas controladas (23:30, 23:59:59.999999,
00:00:00 y 23:59:59 de Guatemala), sin escribir por fuera de la API en ninguna base de operación; ahí mismo se prueba la paginación
con 120 ventas de la misma `fecha_hora`. Las bases se borran al terminar. `prueba-roles` suma las cinco rutas nuevas a su matriz.

## 2026-10-25 — Autenticación entre nodos: firma HMAC por petición, una clave por sucursal

**Contexto.** Dos huecos que dejaba el diseño. (1) Las rutas de servicio de la sucursal (`/traslados/:id/preparar`, `confirmar`,
`abortar` y `/traslados/reconciliacion/ejecutar`) no validaban quién llamaba: cualquiera con acceso a la red de la sucursal podía
confirmar o abortar una transacción preparada. (2) El central no sabía qué sucursal le hablaba: cualquiera podía pedir
`GET /replica/cambios` con el `id_sucursal` de otra y llevarse sus hashes de contraseña, y no podía verificar el `id_sucursal` de
un mensaje de sincronización. En GCP el OIDC de Cloud Run protege la entrada a `api-central`, pero solo dice «una sucursal», no
«cuál»; y no existe en Compose ni entre las VMs de las sucursales.

**Mecanismo.** Cada petición entre nodos lleva `X-SIMA-Firma: s=<id_sucursal>,t=<unix>,v1=<hex>`. `v1` es HMAC-SHA256 sobre
`sima-v1 \n t \n MÉTODO \n ruta-con-query \n sha256(cuerpo)` (un campo por línea para que no haya ambigüedad al concatenar). El
cuerpo que se firma son los bytes exactos que viajan (`rawBody` de Nest, sin dependencias nuevas). Ventana de ±60 s
(`SERVICIO_VENTANA_S`), comparación con `timingSafeEqual`. Cualquier fallo es el mismo `401 FIRMA_DE_SERVICIO_INVALIDA`; el motivo
(`SIN_FIRMA`, `FORMATO`, `VENCIDA` con el desfase, `SUCURSAL_DESCONOCIDA`, `SUCURSAL_NO_ESPERADA`, `FIRMA_INVALIDA`) va solo al
registro, junto con método y ruta sin query, nunca la clave ni la firma. La cabecera es aparte de `Authorization`: en GCP se suma
al OIDC, no lo reemplaza. Viaja en las dos direcciones:
- central → sucursal: el cliente de participantes firma con la clave de la sucursal **destino** y `s=<destino>`; la sucursal exige
  que `s` sea la suya;
- sucursal → central: réplica, `/red/*` (existencias, traslados, ventas, empleados, sucursales), decisión del reconciliador y
  sincronización por HTTP. El central toma la sucursal de `s=` y verifica con la clave de **esa** sucursal.

**Claves.** Una distinta por sucursal: la sucursal N tiene `SERVICIO_CLAVE`; el central tiene `SERVICIO_CLAVE_S<N>` para cada N (se
leen del entorno por patrón, así que sirve para quince sucursales sin tocar código). Mínimo 32 bytes; la API no arranca sin
ellas, ni con una corta, ni con dos iguales en el central (una clave repetida deja a una sucursal firmar como la otra, que es
justo lo que se evita), ni con `SERVICIO_CLAVE` igual a `SESION_CLAVE`. Una sucursal comprometida solo puede firmar como ella.

**Alternativas descartadas.**
- *Un secreto compartido único:* una sucursal comprometida podría firmar como cualquier otra, y no se arregla el hueco (2).
- *OIDC central → sucursal:* obligaría a cada sucursal a verificar un JWT de Google en cada petición (más código, red hacia
  Google para las claves públicas) y no existe en Compose, donde viven las pruebas.
- *mTLS:* la gestión de certificados queda fuera del alcance del proyecto.
- *Derivar las claves de una maestra (HKDF):* el central guardaría una sola clave cuya filtración compromete a todas; con claves
  independientes cada secreto se crea y rota por separado en Secret Manager.
- *Caché de firmas vistas en memoria contra el replay:* no sirve con varias instancias de Cloud Run. La defensa contra el
  replay es que lo importante sea idempotente (más abajo).

**Qué identifica ahora al central: manda la sucursal autenticada, no el `id_sucursal` del cuerpo o de la query.**
| Ruta del central | Regla |
|---|---|
| `GET /replica/cambios` | `id_sucursal` distinto del autenticado → 403 |
| `POST /sincronizacion/mensajes` | `id_sucursal` del mensaje distinto del autenticado → 403 (no entra al inbox) |
| `POST /traslados` | `id_sucursal_destino` distinto del autenticado → 403: una sucursal solo pide traslados para sí misma |
| `GET /traslados?id_sucursal=` | distinto del autenticado → 403 |
| `GET /traslados/:id` | si el autenticado no es el origen ni el destino → 404 (antes lo filtraba la sucursal; ahora también el central) |
| `GET /traslados/decision/:gid` | solo responde al participante de ese gid; para otra sucursal el gid «no existe» (404) |
| `POST/PATCH/PUT /empleados`, `PATCH /sucursales/:id` | el actor (`id_usuario_actor`) tiene que ser un ADMIN activo **de la sucursal autenticada** (403 si no) |
| `GET /existencias/:cb` | firma de cualquier sucursal; sin atar (lectura global por diseño) |
| `GET /empleados`, `GET /ventas`, `GET /ventas/:id` | firma de cualquier sucursal; **sin atar** a propósito: el ADMIN de una sucursal ve la red, y el central no sabe qué rol tiene quien pregunta |
| `GET /salud` | pública (el uptime check y la sonda de enlace) |
| `POST /traslados/recuperacion/ejecutar` | `@RutaProgramada`: la llama Cloud Scheduler con OIDC, no una sucursal; sin firma de sucursal |

El central **deniega por defecto**: una ruta nueva sin `@Publica` o `@RutaProgramada` exige firma.

**Modelo de confianza que queda.** El central sigue confiando en el usuario que afirma una sucursal autenticada (el actor de
empleados, el solicitante de un traslado): no tiene la sesión de la caja. Lo que cambió es que ese usuario tiene que ser de la
sucursal que firmó, y que una sucursal no puede hablar por otra. Límite que se declara: una sucursal comprometida puede leer las
ventas y la lista de empleados (sin hashes ni correos) de toda la red, porque esas lecturas son de red por diseño, y puede actuar
como cualquiera de **sus** usuarios.

**Sincronización por Pub/Sub.** El push lo hace Pub/Sub, no la sucursal, así que no hay cabecera. La sucursal firma el `data` del
mensaje y la firma viaja como atributo `firma_servicio` (mismo formato, método `PUBSUB`); el central la verifica con la misma
función y exige que el `s=` coincida con el `id_sucursal` del mensaje (403). **Sin ventana de tiempo** en este camino: Pub/Sub
reentrega durante días y una ventana de 60 s dejaría mensajes legítimos en el tema de muertos; el replay de uno viejo lo neutraliza
`inbox_idempotencia` (ya aplicado = duplicado). Un sobre sin atributo o con una firma mala da 401.

**GCP.** La firma se suma al OIDC, no lo reemplaza. Siguen solo con OIDC: `POST /traslados/recuperacion/ejecutar` (Cloud Scheduler,
`sa-scheduler`; en Compose queda abierta, y solo fuerza una pasada que el proceso ya hace cada 5 s y que no puede decidir nada
distinto) y `GET /salud` (el uptime check usa el token del agente de Monitoring). El push de Pub/Sub lleva OIDC (`sa-pubsub-push`)
más el atributo firmado. *Para hacer allá, sin hacerlo todavía:* crear tres secretos en Secret Manager —`sima-servicio-clave-s1`,
`-s2` y `-s3`, cada uno con `openssl rand -base64 48`—; `sa-api-central` lee los tres y la cuenta de la VM de la sucursal N lee solo
el suyo; `cloud_run.tf` los pasa a api-central como `SERVICIO_CLAVE_S<N>` por `secret_key_ref`; `infra/gcp/sucursal/arranque.sh` pasa
`SERVICIO_CLAVE` a la VM (junto a `SESION_CLAVE`, que tampoco está cableada todavía); `infra/gcp/carga-gcp.sh` necesita
`-e SERVICIO_CLAVE` para el k6 de existencias. Desplegar en este orden: secretos, central, sucursales. Una sucursal sin clave no
arranca; un central sin las claves deja la cola de las sucursales en espera (401 es transitorio para el publicador), sin perder nada.

**`POST /traslados` no era idempotente, y ahora lo es.** Un replay de la petición firmada dentro de la ventana, o el reintento del
usuario tras un `504 RESULTADO_DESCONOCIDO`, registraba un segundo traslado. La sucursal genera un UUID por solicitud del usuario
(`id_idempotencia`; en `/red/traslados` lo manda la caja y lo conserva si recibe un 504, o lo genera la sucursal si falta) y el
central lo guarda en `traslado.id_idempotencia` con `UNIQUE (id_sucursal_destino, id_idempotencia)` (migración
`central/015_idempotencia_traslado.sql`; las filas anteriores quedan en NULL). Con una llave que ya existe devuelve el traslado
registrado (200, o 202 si sigue en curso; `repetida: true`) sin volver a correr el protocolo; con la misma llave y otro contenido,
409 `IDEMPOTENCIA_REUTILIZADA`; sin llave, 400. El `INSERT … ON CONFLICT DO NOTHING` resuelve la carrera de peticiones iguales
simultáneas (una gana; las otras deshacen su documento y devuelven el de la ganadora). La llave es **por sucursal de destino**, no
global: una sucursal no recupera el traslado de otra con una llave ajena. La caja la genera con `getRandomValues`, no con
`crypto.randomUUID()`, porque se sirve por http desde la red de la tienda, que no es un contexto seguro. *Alternativa descartada:*
que la llave fuera opcional; dejaría el replay posible con solo omitirla.

**Qué replay sigue siendo posible (lo que de verdad no es idempotente).** Se necesita estar en la red con una petición capturada
(hacia el central, en GCP, viaja por HTTPS), y solo dentro de ±60 s:
- `preparar` repetido **después de un `ABORTADO`**: vuelve a preparar un gid ya deshecho y retiene los bloqueos de esas filas hasta
  que el reconciliador lo deshace (plazo de 30 s más un intervalo). No pierde ni duplica unidades. Antes de la decisión o tras
  confirmar es inofensivo (idempotente por gid / `YA_REGISTRADO`).
- `PATCH /empleados/:id`, `PUT /empleados/:id/contrasena` y `PATCH /sucursales/:id` fijan un valor: repetidos dentro de la ventana
  deshacen un cambio posterior hecho sobre ese mismo dato en esos segundos.
Todo lo demás es idempotente: sincronización (`inbox_idempotencia`), `confirmar`, `abortar`, la reconciliación, la recuperación,
`POST /empleados` (el `nombre_usuario` único da 409) y todas las lecturas. Cerrar los dos de arriba exigiría un nonce
compartido por todas las instancias (una tabla), que queda fuera del alcance.

**Riesgo operativo.** Si el reloj de un nodo se desvía más de la ventana, todo da 401 `VENCIDA`; el registro dice el desfase. En
Docker Desktop pasa tras suspender el equipo. `SERVICIO_VENTANA_S` lo ajusta; en GCP los relojes los mantiene NTP.

**Pruebas.** `scripts/prueba-blindaje/prueba.mjs` (en `probar-todo.sh`), 11 casos: la firma (el helper de los scripts es una
implementación aparte y coincide con la del código; 13 variantes de rechazo, cada una con su motivo; claves cortas, repetidas y
ausentes); las cuatro rutas de servicio de la sucursal sin firma, alterada, de hace 5 minutos, del futuro, con el cuerpo o la ruta
cambiados después de firmar y con formato roto → 401, con el motivo en el registro y sin la clave; la firma de la 1 enviada a la 2 →
401; una transacción preparada que no se puede confirmar ni abortar sin firma (sigue en `pg_prepared_xacts`) y que después resuelve
el reconciliador preguntando al central con firma; réplica, sincronización y las ataduras de traslados y de actor; la idempotencia
de `POST /traslados` (replay exacto, reintento con firma nueva, diez peticiones simultáneas, contando filas en `traslado`,
`transaccion_global` y `participante`); y el arranque sin clave. Los scripts que llaman rutas de servicio a mano firman con
`scripts/lib/firma-servicio.mjs`; el k6 de existencias firma con `k6/crypto`.


## 2026-10-25 — «Por reponer»: consulta operativa local, no analítica

**Decisión.** `GET /existencias/por-reponer` (ENCARGADO y ADMIN) responde en la base de la sucursal: productos activos con
existencia menor que `producto.stock_min` (sin fila de existencia cuenta como 0), del mayor faltante al menor. Es la pregunta
«qué me falta hoy», la misma naturaleza que consultar una existencia: lee el estado actual de la propia sucursal, no necesita
el central ni historia, y por eso funciona sin enlace. `stock_min` llega por la réplica del catálogo.

**Alternativa descartada.** Resolverlo en el almacén analítico (BigQuery). Se descartó porque el almacén llega con retraso,
no sirve sin enlace y aquí la respuesta es un umbral fijo contra una cifra del momento. La regla de «analítica jamás contra la
base operativa» no se rompe: esto no es analítica, es una lectura puntual indexada por producto de la propia sucursal.
La **sugerencia de reposición con historia** (rotación, demanda, cuánto pedir) sigue siendo analítica y vive en BigQuery.

**Atajos de la PWA.** «Buscar en la red» (desde Por reponer) y «Ver en la red» (desde un producto agotado o una venta rechazada
por `EXISTENCIA_INSUFICIENTE`) abren la consulta de red con `?codigo=`; desde ahí, ENCARGADO y ADMIN abren el formulario de
traslado con `?origen=&codigo=&cantidad=`. Lo precargado se valida como si lo hubiera escrito el usuario (origen activo y distinto
de esta sucursal, producto del catálogo local, cantidad entera ≥ 1) y lo inválido se descarta con aviso; no hay endpoint nuevo ni
se salta ninguna regla del servidor. El CAJERO ve la red pero no el botón de solicitar.



## 2026-10-26 — `proveedor.activo`: baja lógica replicada como `producto.activo`

**Decisión.** `proveedor` gana `activo BOOLEAN NOT NULL DEFAULT TRUE` (migración 016 del central y 014 de la sucursal). Hasta
ahora un proveedor no se podía dar de baja de ninguna forma: el DELETE está impedido por trigger y no había otra columna que lo
marcara. La baja es `UPDATE proveedor SET activo = FALSE` en el central; el trigger de `version_replica` (011) numera el cambio y
viaja por la réplica central → sucursal igual que `producto.activo`, sin tocar el cursor ni el orden de versiones. La sucursal
nunca borra filas: solo refleja el valor.

**Alternativa descartada.** Permitir el DELETE en el central y propagarlo. La réplica solo transporta filas con versión y no sabe
retirar filas; habría que inventar lápidas, y un proveedor borrado dejaría huérfano el histórico que lo referencia. La baja
lógica reutiliza el mecanismo existente y conserva el histórico.

**Pruebas.** `scripts/prueba-replica/prueba.mjs`, caso 1b: desactivar un proveedor en el central llega a las tres sucursales.
Los proveedores que crea la prueba no se pueden borrar y quedan desactivados.


## 2026-10-27 — Recepción de mercadería de un proveedor: local, con la fila de existencia creada en el momento

**Decisión.** `POST /recepciones` (ENCARGADO o ADMIN) registra lo que llegó de un proveedor y la existencia sube. Es una
**operación local, sin red, como la venta**: una sola transacción en la base de la sucursal (existencia, documento `RECEPCION`
con `id_proveedor` y `referencia`, líneas, movimientos `RECEPCION_ENTRADA` y fila de outbox) y el aviso viaja al central por
el outbox cuando hay enlace. No va por 2PC: no cruza sucursales, y meterla ahí la volvería inservible sin red. El central
aplica el documento como cualquier otro, con la misma guarda de versión sobre `existencia_consolidada`.

**Dos clases de `RECEPCION`.** El tipo de documento ya existía para la recepción de un traslado. Ahora hay dos y no se mezclan:

| | de un traslado | de un proveedor |
|---|---|---|
| `id_traslado` | sí | no |
| `id_proveedor` | no | sí |
| movimiento | `TRASLADO_ENTRADA` | `RECEPCION_ENTRADA` |

Se hace cumplir en tres niveles: el código, el validador de mensajes del central (`validar-mensaje.ts`: uno y solo uno de los dos
orígenes, y el tipo de movimiento que le toca) y un `CHECK` en el motor (`documento_recepcion_origen_check`, migraciones
sucursal 015 y central 017): una `RECEPCION` lleva `id_traslado` o `id_proveedor`, nunca los dos ni ninguno, y solo una
`RECEPCION` puede llevar proveedor. El modelo ya tenía lo necesario (`documento.id_proveedor`, `documento.tipo = RECEPCION`,
`movimiento.tipo` sin `CHECK`); la migración solo cierra la puerta a una mezcla.

**`INSERT … ON CONFLICT DO NOTHING` antes del `FOR UPDATE`.** Un producto del catálogo puede no tener fila de existencia en la
sucursal (se creó en el central después de la semilla). `SELECT … FOR UPDATE` sobre una fila que no existe no bloquea nada: dos
recepciones simultáneas del mismo producto nuevo verían las dos «no hay fila», harían las dos `INSERT` y una caería en la llave
primaria (`23505`, que la API devolvería como `RESTRICCION_BD`). Con `ON CONFLICT DO NOTHING` la segunda `INSERT` espera a que la
primera confirme y entonces no inserta nada; desde ahí las dos pasan por el `FOR UPDATE` de siempre y se serializan. La espera
respeta el `lock_timeout` de la venta (409 `PRODUCTO_BLOQUEADO` si vence). *Alternativa descartada:* capturar el `23505` y
reintentar: dentro de una transacción de PostgreSQL el error la deja abortada, habría que reiniciarla entera.

**Orden de bloqueo por `id_producto`.** Como la venta y el ajuste, la recepción toma los bloqueos en orden ascendente de
`id_producto`, no en el orden en que vienen las líneas. Una recepción `[B, A]` y una venta `[B, A]` que coinciden en dos productos
toman primero A y después B las dos: nadie espera a nadie en círculo. *Alternativa descartada:* bloquear en el orden de la
solicitud (un deadlock esperando a ocurrir). Está **demostrado** en `scripts/prueba-recepcion/prueba.mjs`, caso 7: 20 pares de
recepción + venta simultáneos sobre dos productos, la mitad con las líneas en `[B, A]` y la mitad en `[A, B]`: las 40 dan 201, la
existencia y la versión finales cuadran exacto y el log de `db-sucursal-1` no tiene `deadlock detected` desde el inicio del caso
(filtrado por la hora de la base, como en la prueba de réplica 4b). El caso incluye un **control**: con el orden opuesto escrito a
mano sí hay `40P01` y el mismo filtro del log lo ve; sin él, el «0» no probaría nada.

**El costo se guarda, pero no hay costo promedio.** `costo_unitario` queda en `linea_documento.precio_unitario` y el total en
`documento.total` (suma de subtotales redondeados por línea, calculada en SQL). El sistema no valúa el inventario ni lleva costo
promedio, costeo ni cuentas por pagar: contabilidad está fuera de alcance. Un producto sin costo se registra con `0`.

**Catálogo de recepción propio.** `GET /recepciones/catalogo` devuelve proveedores y productos activos de la copia local, con
`LEFT JOIN existencia`. `GET /existencias` no sirve para esto: hace `JOIN` y un producto sin fila no aparece, así que la pantalla
no habría podido ofrecer justo el caso que la recepción resuelve (el producto nuevo). No cambié `/existencias`: la caja de venta
sigue viendo solo lo que se puede vender.

**Se valida la baja de proveedor en la sucursal, no en el central.** La sucursal rechaza (400) un proveedor dado de baja. El
central no lo vuelve a verificar: una baja posterior a una recepción ya confirmada no debe hacer rechazar el mensaje, y la `FK`
garantiza que el proveedor existe.

**Límite conocido: el doble envío — CERRADO para la pantalla el 2026-10-27.** Era el mismo de la venta (`POST /recepciones` no
tenía clave de idempotencia y dos peticiones iguales sumaban dos veces). Ahora la pantalla manda una llave por envío y
«Reintentar» la reusa; ver «Llave de idempotencia en `POST /ventas` y `POST /recepciones`» (2026-10-27).

**Los atajos de «Por reponer».** El botón «Registrar recepción» abre la pantalla con `?codigo=&cantidad=` (el faltante). Se valida
como si lo hubiera escrito el usuario (producto activo del catálogo, cantidad > 0 con hasta dos decimales) y lo inválido se
descarta con aviso. Se ofrece con y sin enlace, porque la recepción es local.

**Efectos en lo que ya existía.** `verificar.mjs` conoce `RECEPCION_ENTRADA` (suma) y trata como existencia sembrada 0 a los
productos posteriores a la semilla (id > 50): el libro de una sucursal es «semilla + movimientos». `prueba-roles` agrega las dos
rutas a la matriz y su caso 6 inserta la `RECEPCION` de prueba con proveedor para que solo falle `documento_usuario_check`.

**Pruebas.** `scripts/prueba-recepcion/prueba.mjs` (7 casos; está en `probar-todo.sh`): recepción de dos productos con el central
cuadrando, producto sin fila, concurrencia (10 recepciones del mismo producto sin fila + 10 ventas), rechazos con 400 y el `CHECK`
escrito directo en la base y el validador de mensajes, CAJERO 403, recepción sin enlace con el outbox pendiente y llegada al
reconectar, y el orden de bloqueo.

**Pendiente de GCP (no se tocó infra en este hito).**
- Aplicar las migraciones `sucursal/015` y `central/017` en los nodos de GCP al desplegar (`infra/gcp/migraciones`).
- Actualizar textos de `infra/gcp/bigquery` que dicen que la recepción de proveedor no existe. La lógica no cambia: el `MERGE`
  (`05_merge_hecho.sql`) ya toma `d.id_proveedor` y `m.tipo` de CDC sin filtrar por tipo, así que un `RECEPCION_ENTRADA` real entra
  con su proveedor. Los textos:
  - `02_dimensiones.sql`, comentario sobre `dim_proveedor` («ningún documento lo llena todavía… Solo la historia simulada lo usa»);
  - `03_hecho_movimiento.sql`, `description` de `id_proveedor` («Solo SIMULADO lo llena») y de `tipo` («RECEPCION_ENTRADA solo en
    SIMULADO»);
  - `04_historia_simulada.sql`, comentario «RECEPCION_ENTRADA no existe en el flujo operativo».
- Comprobar en el tablero «Recepciones por proveedor» (origen `CDC`) que aparece la primera recepción real, ya con Datastream.

## 2026-10-27 — Catálogo de productos administrado por el ADMIN (sin imágenes)

**Contexto.** `producto` es un maestro del central que baja a las sucursales por la réplica (migración central 011), igual que
`usuario`. Hasta ahora solo se escribía con la semilla o con SQL. Este paso agrega la pantalla «Productos» de la caja y los
endpoints que la sostienen: el ADMIN crea productos, cambia nombre, categoría, precio y mínimo, y los da de baja o reactiva.

**El catálogo es maestro del central y solo lo edita el ADMIN.** `POST /red/productos` y `PATCH /red/productos/:id` de la sucursal
son solo ADMIN, toman el actor de la sesión (un cuerpo con `id_usuario_actor` da 400) y reenvían a `POST /productos` y
`PATCH /productos/:id` del central. Misma mecánica que el panel de empleados (decisión del 2026-10-24): UNA transacción que toma
primero `contador_replica FOR UPDATE`, después valida que el actor sea un ADMIN activo de la sucursal que firmó la petición, y
recién entonces valida contra el catálogo y escribe; la réplica existente lleva el cambio a las tres sucursales. Se reutilizó
`bloquearYValidarActor` de empleados (se exportó; el mensaje del 403 ahora dice «empleados, sucursales y productos»).
*Alternativa descartada:* que cada sucursal edite su copia del catálogo y la sincronice hacia el central. Rompe la regla de que el
catálogo tiene un solo dueño y abriría conflictos de escritura entre sucursales que el contador serializado evita por diseño.
Sin enlace no se puede crear ni cambiar un producto (503 `SIN_ENLACE_CENTRAL`); la caja sigue vendiendo con el catálogo que tiene.

**`codigo_barras` es inmutable.** Es lo que viaja en cada línea de venta y de recepción (`codigo_barras` en el outbox) y lo que busca
el lector de la caja. Si se pudiera editar, durante los segundos que tarda la réplica cada sucursal conocería al producto con un
código distinto, y una venta ya en el outbox llegaría al central con un código que ya no corresponde al producto. La API lo rechaza
con 400 antes de escribir, y la migración `central/018` agrega un trigger `BEFORE UPDATE` que lo rechaza por cualquier otro camino
(psql, un script), con el mismo criterio que `usuario.id_sucursal` (011). Un código mal escrito se corrige dando de baja el producto
y creando otro. *Alternativa descartada:* permitir el cambio y propagarlo; exige reescribir las líneas ya enviadas o mantener una
tabla de alias, a cambio de un caso (error de captura) que la baja más el alta resuelve.

**La baja es lógica y no borra existencia.** Dar de baja es `activo = false`; el trigger de DELETE de la migración 012 ya impedía
borrar. Un producto de baja no se vende (`registrarVenta` ya rechazaba `activo = false` con 400), no se recibe (la recepción ya lo
validaba), no se traslada (el coordinador solo toma productos activos) y no sale en el buscador de la caja ni en el de traslados ni
en el catálogo de recepción. Su existencia **no se toca**: sigue en cada sucursal, sigue visible en «Existencias en la red» marcada
«de baja», y se puede ajustar a 0 con un conteo (el ajuste nunca filtró por `activo`). *Alternativa descartada:* poner la existencia
en 0 al dar de baja: sería una escritura de inventario en el maestro del central sobre filas que pertenecen a cada sucursal, sin
documento ni movimiento que la respalde.

**Un producto nuevo no crea existencia, y por eso se ajustaron dos lecturas de la sucursal.** Crear el producto solo escribe en
`producto`; aparece con 0 en las tres sucursales y se le da entrada con una recepción (que ya crea la fila de existencia). Para que
«aparezca con 0» fuera verdad hubo que cambiar dos cosas que antes suponían una fila de existencia por producto:
- `GET /existencias` y `GET /existencias/:cb` ahora parten de `producto` con `LEFT JOIN existencia` (cantidad 0 y versión 0 si no hay
  fila) y traen `activo`. Antes hacían `JOIN` y un producto sin fila no aparecía; la decisión del 2026-10-27 sobre la recepción había
  dejado `/existencias` así a propósito («la caja de venta sigue viendo solo lo que se puede vender»). Se revierte para el catálogo
  completo, y la caja filtra `activo` en el cliente: los productos recién creados salen en la caja como «Agotado», con su atajo.
- Las tres estrategias de venta (pesimista, optimista, sin control) lanzaban `PRODUCTO_NO_EXISTE` (404) cuando no había fila de
  existencia, aunque el producto existiera en el catálogo. Ahora lanzan `EXISTENCIA_INSUFICIENTE` (409, `disponible: 0`): el producto
  ya se resolvió contra el catálogo antes de llegar ahí, así que «sin fila» solo puede significar «esta sucursal no ha recibido ninguno».
  `PRODUCTO_NO_EXISTE` queda para códigos que no están en el catálogo.

**El precio vive en cada línea de venta, así que cambiarlo no altera el historial.** `linea_documento.precio_unitario` guarda el precio
con que se cobró (y `documento.total` la suma). Cambiar `producto.precio_venta` solo afecta a las ventas que se hagan después de que la
réplica lleve el precio a esa sucursal; hasta entonces la sucursal cobra el precio que tiene. La prueba lo comprueba en la línea de la
venta anterior en la sucursal, en su detalle y en el central. Es la misma ventana de unos segundos de cualquier cambio de catálogo.

**Validaciones.** `codigo_barras` de 8 a 14 dígitos y único (409 `CODIGO_EN_USO`); `nombre` obligatorio y de hasta 100 caracteres;
`categoria` obligatoria y de hasta 50; `precio_venta` número > 0 con hasta 2 decimales y dentro de `NUMERIC(10,2)`; `stock_min`
entero ≥ 0. Las categorías no son una tabla: la lista es `SELECT DISTINCT categoria FROM producto` (`GET /productos/categorias`, local).
Una categoría nueva que solo difiere de una existente en las mayúsculas usa la existente, para que «lácteos» y «Lácteos» no partan el
filtro en dos. Las reglas están en el código y no en `CHECK`: la semilla trae códigos que un `CHECK` de 8 a 14 dígitos habría que
revisar, y el alcance de este paso no incluye tocar datos existentes.

**La pantalla.** «Productos» (solo ADMIN, en el riel) lista la copia local con buscador y filtro por categoría. Como la lista sale de
la copia local, un cambio recién hecho tardaría unos segundos en verse; por eso la pantalla guarda lo que el central confirmó y lo
muestra encima de la copia hasta que la réplica la alcanza (consulta la copia cada 4 s mientras haya algo pendiente), con la marca
«llegando a las sucursales». Tras crear un producto, «Registrar recepción» abre Recepción con el producto precargado, y se habilita
recién cuando el producto está en la copia local (Recepción valida contra ella). La inicial del nombre sobre un color por categoría
reemplaza a las imágenes.

**Imágenes: descartadas por alcance.** Subir, guardar y servir imágenes pide almacenamiento de archivos, tamaños y formatos, y
réplica de binarios a sucursales que deben seguir funcionando sin red. Nada de eso es el problema que evalúa el proyecto. Se
sustituye por la inicial con color; si hiciera falta, una imagen sería una URL en una columna nueva, sin tocar el modelo de réplica.

**Pruebas.** `scripts/prueba-catalogo/prueba.mjs` (7 casos; está en `probar-todo.sh`): alta con réplica a las tres sucursales y venta
con y sin existencia; duplicado e inválidos sin escribir; cambio de precio contra el historial; baja (vender, recibir, trasladar,
catálogo de recepción, existencia consultable y ajustable) y reactivación; el trigger de `codigo_barras` y el de DELETE; 403 a
ENCARGADO y CAJERO y actor no ADMIN en el central; y sin enlace. `prueba-roles` suma las tres rutas nuevas a la matriz. `prueba-caja`
compara ahora el catálogo con la tabla `producto` y no con las filas de existencia.

**Pendiente de GCP (no se tocó infra en este hito).** Aplicar `central/018` en la base de Cloud SQL del central al desplegar
(`infra/gcp/migraciones`). La sucursal no lleva migración nueva.

## 2026-10-27 — `prueba-reconciliador`, caso 2: el orden lo fijan los votos y los retardos, no un plazo

**Qué pasaba.** El caso 2 ("preparada con bitácora ABORTADO") dependía de que el voto del origen llegara después del
plazo de votación de 800 ms (`_prueba.plazo_votacion_ms`). Con la máquina cargada el origen a veces no quedaba preparado
y la prueba fallaba con "no se formó la huérfana". Competir contra un reloj no es determinista.

**Decisión.** El escenario ya no compite contra ningún plazo: el origen vota SÍ y queda preparado; el destino vota NO
porque la prueba retiene la tabla `existencia` de esa sucursal (`LOCK TABLE … ACCESS EXCLUSIVE`) hasta que vence su
`lock_timeout`; el coordinador espera 4 s (`retardo_antes_de_decidir_ms`, más que el `lock_timeout` de la venta, para que
el 409 se compruebe antes de la decisión), escribe ABORTADO y retiene la propagación (`retardo_despues_de_decidir_ms`).
Lo que se comprueba no cambia: el origen preparado, la venta con 409, y el reconciliador que pregunta, ve ABORTADO y deshace.

**Alternativa descartada.** Bloquear la fila de `existencia` del destino con `FOR UPDATE`: el "preparar" del destino solo
lee esa fila (`SELECT 1`), así que un bloqueo de fila no lo frena y votaba SÍ.

**Nota.** Con esto, `_prueba.plazo_votacion_ms` queda sin uso en los scripts; se conserva porque es parte de la inyección
documentada del coordinador.

## 2026-10-27 — Llave de idempotencia en `POST /ventas` y `POST /recepciones`

**Decisión.** Las dos operaciones aceptan `id_idempotencia` (UUID, opcional). `documento` lleva `id_idempotencia UUID` y
`huella_solicitud TEXT` (migración `sucursal/016`), con un índice **único parcial** `uq_documento_idempotencia` sobre la llave
`WHERE id_idempotencia IS NOT NULL`; las filas anteriores y los ajustes y traslados quedan en `NULL`. Con una llave que ya
existe y el mismo contenido, la respuesta es **201 con el documento ya registrado** (mismo cuerpo que la primera vez, más
`repetida: true`) y no se mueve la existencia; con otro contenido, **409 `IDEMPOTENCIA_CONFLICTO`** y no se escribe nada. Es el
mismo patrón que `POST /traslados` (`central/015`), pero la unicidad la decide la base **local**: funciona sin enlace, que es lo
que importa para una venta. Cierra la limitación del doble cobro de la caja (2026-10-22) y el doble envío de la recepción.

**Por qué la llave la genera el cliente.** Solo el cliente sabe que dos peticiones son «la misma»: el servidor las ve idénticas
en dos casos opuestos, el reintento de un cobro cuya respuesta se perdió y dos clientes distintos que compran lo mismo un minuto
después. La caja genera el UUID al presionar «Cobrar» y lo conserva mientras no haya respuesta definitiva. Con la llave en el
servidor no habría forma de distinguirlos.

**Qué conserva y qué descarta la PWA** (Caja y Recepción, igual): la llave vive en un `useRef`, junto con la huella del contenido.
Se **conserva** ante un error de red, un plazo vencido o un 5xx (la operación pudo haberse registrado), y entonces «Cobrar» pasa a
«Reintentar», que manda la misma llave. Se **descarta** tras un 201, tras un 4xx (la sucursal contestó que no registró nada) y
si cambia el carrito, el método de pago o el formulario: eso es otra operación y lleva llave nueva (el aviso lo dice). Si la
respuesta trae `repetida: true`, la venta se muestra como cobrada con «Esta venta ya se había registrado» (en recepción, «Esta
recepción…»). **Límite:** la llave no se guarda en `localStorage`; si se recarga la página se pierde el carrito y también la llave,
y lo que se cobre después es una venta nueva. Persistirla exigiría persistir el carrito; no se hizo.

**Por qué el UNIQUE serializa los reintentos simultáneos.** El `INSERT` del documento va **primero** en la transacción, antes
de tocar la existencia. Si dos peticiones con la misma llave llegan a la vez, la segunda espera en el índice único hasta que la
primera termina; si ésta confirma, la segunda falla con `23505` sobre `uq_documento_idempotencia` y deshace todo lo suyo
(todavía no había descontado nada, y de todos modos el `ROLLBACK` deshace el resto); si ésta se deshace, la segunda sigue como
si fuera la primera. El código captura ese `23505` ya fuera de la transacción, lee el documento existente con el pool (una
lectura nueva, que ya ve el commit de la ganadora) y lo devuelve. Nunca hay dos documentos con la misma llave ni dos descuentos.
La espera respeta `lock_timeout` (3 s): si la primera tardara más, la segunda responde `409 PRODUCTO_BLOQUEADO`, que no escribió
nada y se puede reintentar con la misma llave.

**Por qué el INSERT va antes de descontar** (cambia el orden de los pasos de `registrarVenta` y de la recepción: documento →
existencia → líneas → movimientos → outbox). Con la existencia primero, el reintento de una venta que dejó el producto en cero
habría respondido `EXISTENCIA_INSUFICIENTE` en lugar de `repetida: true`, y con la estrategia pesimista habría esperado el
bloqueo de la fila que tiene la ganadora. La prueba lo cubre (caso 1: reintento con la existencia en 0; caso 2, tercera ronda:
10 simultáneos con la existencia justa para una sola venta). Lo que no cambia: el outbox sigue al final, tras los bloqueos de
existencia, así que el orden de `id_outbox` sigue coincidiendo con el de commit por producto; y `fecha_hora` es la hora de inicio
de la transacción, no la del `INSERT`. Costo: cada intento chocado consume un `id_documento` de la secuencia (huecos en los ids).
Antes del `INSERT` quedan `verificarUsuario` y, en la recepción, `verificarProveedor`: son llaves foráneas del documento y un
error ahí debe seguir siendo un 400 con su mensaje y no un 409 `RESTRICCION_BD`. Consecuencia: reintentar una operación ya
registrada después de dar de baja al usuario o al proveedor responde 400, no `repetida`.

**La huella contra la reutilización de llaves.** `huella_solicitud` es el sha256 del contenido ya validado y normalizado: el tipo
de documento, `id_usuario`, el método de pago y la referencia (o proveedor y factura, en la recepción) y las líneas **ordenadas
por código de barras** (mandarlas en otro orden no cambia la huella). Con la llave sola, reusar un UUID para otra venta
devolvería en silencio una venta ajena como si fuera la nueva; con la huella es un 409. Incluye al usuario y el tipo: un CAJERO
no recibe como suya la venta de otro, y una llave de venta no sirve para una recepción. La prueba cubre cada variante (cantidad,
producto, método de pago, usuario, tipo, costo, factura).

**Las llamadas sin llave conservan el comportamiento anterior.** Si `id_idempotencia` falta, el servidor genera uno: la operación
se registra y `documento` queda con llave y huella, pero **esa llamada no está protegida**: dos iguales son dos documentos. Así
siguen funcionando los scripts, k6 y pgbench sin tocarlos. Una llave que no es UUID da 400. La prueba lo documenta con un control:
10 ventas simultáneas sin llave son 10 documentos.

**Alternativas descartadas.**
- *Que el servidor derive la llave del contenido:* dos clientes que compran lo mismo en el mismo minuto son dos ventas legítimas.
- *Consultar la llave primero y insertar después:* tiene una carrera (las dos peticiones no la ven y las dos descuentan) y no
  resuelve el reintento tras una venta que agotó la existencia. El índice único es lo que cierra la carrera.
- *Reusar `outbox.id_idempotencia` como llave del documento:* ese UUID lo genera el servidor, es la clave primaria del inbox
  **global** del central y viaja en el mensaje; una llave elegida por el cliente podría chocar con la de otra sucursal.
  Son dos llaves con dos funciones y quedan separadas.
- *`INSERT … ON CONFLICT DO NOTHING`:* también serializa en el índice, pero deja la transacción abierta y obliga a comprobar si
  insertó; atrapar el `23505` y deshacer todo es más difícil de usar mal.
- *Llevar la llave al central:* innecesario. Una repetición se deshace antes de escribir el outbox, así que el central nunca
  recibe un segundo mensaje; su inbox ya es idempotente por mensaje. El payload de sincronización no cambia.
- *Un índice sobre `outbox` para encontrar el mensaje de un documento:* la respuesta repetida debe llevar el `id_idempotencia` del
  outbox (el de la primera respuesta) y `outbox` no tiene índice por documento. En vez de encarecer cada venta, la búsqueda
  recorre la clave primaria de más reciente a más antigua y se detiene en el primer acierto, que para un reintento está al final.
  Medido en una tabla temporal de 500 000 filas con la forma de `outbox`: un documento de las últimas filas se encuentra en 0,08 ms
  (`Index Scan Backward`); uno antiguo recorre las 500 000 filas, unos 120 ms. Es el costo de un reintento tardío, no de una venta.

**Diferencias que tiene una respuesta repetida frente a la primera** (el resto del cuerpo es igual, y la prueba lo compara):
`reintentos` vale 0, `estrategia` es la vigente, y `nombre` es el que tiene hoy el producto en el catálogo local.

**No medido.** El costo de mantener un segundo índice único por venta no se midió (no se repitió `prueba-carga` en su perfil
completo); el índice es parcial y solo cubre ventas y recepciones.

**Demostrado.** `scripts/prueba-idempotencia/prueba.mjs` (7 casos, agregada a `probar-todo.sh`): repetición secuencial, 10
simultáneos (tres rondas, la última con la existencia justa), conflictos de contenido, sin llave, recepciones, enlace cortado, y
que el central recibió cada documento una vez y `verificar.mjs` cuadra. Corrió con las estrategias PESIMISTA y OPTIMISTA. **No
demostrado:** la lógica de la PWA (reusar la llave tras un error de red o un 5xx, descartarla tras un 201 o un 4xx) compila
(`next build`) pero no se ejecutó en un navegador ni tiene prueba automática; la prueba del servidor ejerce el contrato del que
depende, no el cliente.

**Efecto en `prueba-indices`.** `medir.mjs` captura el SQL de `registrarVenta` con un pool falso que devuelve filas vacías. Como
el `INSERT` del documento ahora va antes de leer el catálogo, el pool falso tiene que devolverle su fila o la venta termina antes de
pedir el catálogo y la captura falla. Se ajustó solo la respuesta falsa; la consulta medida es la misma.

**Pendiente de GCP (no se tocó infra en este hito).** Aplicar `sucursal/016` en las bases de las tres sucursales al desplegar.
*Corregido el 2026-10-28:* esta nota decía que las dos columnas nuevas de `documento` llegarían al CDC. No es así: `id_idempotencia`
y `huella_solicitud` se agregan solo en la base de la **sucursal** (`sucursal/016`); el `documento` del central, que es el que lee
Datastream, no las tiene y no las recibe (no viajan en el payload del outbox). Esta migración no cambia nada en BigQuery.

## 2026-10-28 — Despliegue en GCP: una cuenta de servicio por sucursal y claves en Secret Manager

**Contexto.** Lo que entró a `main` desde el último despliegue (`28d0382`) necesita en GCP tres secretos por nodo que no existían:
`SESION_CLAVE` y `SERVICIO_CLAVE` en cada sucursal y `SERVICIO_CLAVE_S1..S3` en api-central (la API no arranca sin ellos), y las
migraciones central 011–018 y sucursal 010–016. El procedimiento está en `docs/despliegue-gcp.md`; aquí, las decisiones.

**Una cuenta de servicio por sucursal (`sa-sucursal-1..3`).** Las tres VMs usaban `sa-sucursal`. Con una cuenta compartida,
cualquier VM podía leer en Secret Manager la clave de servicio de otra y firmar como ella, lo que anula la clave por sucursal
(2026-10-25). Ninguna condición de IAM distingue VMs que usan la misma cuenta, así que la única separación real es una cuenta por
VM: la de la sucursal N lee solo `sima-sesion-clave-sN` y `sima-servicio-clave-sN`; `sa-api-central` lee las tres de servicio y
ninguna de sesión. Cada cuenta nueva recibe los mismos permisos que tenía la compartida (registros, métricas, lectura de Artifact
Registry, publicación en el tema de sincronización, invocar api-central). Cambiar la cuenta de una VM es un cambio en el lugar
que exige detenerla (`allow_stopping_for_update`); no la recrea, y el disco, con la base de la sucursal, se conserva.
`sa-sucursal` se queda con sus permisos hasta que las VMs pasen un ciclo de apagar y encender limpio con la cuenta nueva, y
después se retira en un apply aparte (se puede volver atrás mientras tanto). Con esto también se cierra el riesgo abierto del
2026-10-23 («`sa-sucursal` es una sola cuenta para las 3 VMs»): qué sucursal habla lo dice la firma, y cada VM solo tiene su clave.
*Alternativas descartadas:* las claves en los metadatos de cada VM (los lee cualquiera con `compute.instances.get` y, escritos
por Terraform, quedarían en el estado); una clave maestra de la que se derivan las demás (descartada el 2026-10-25).

**Los valores de las claves no pasan por el repositorio ni por Terraform.** Terraform declara solo los contenedores
(`secretos.tf`, con `prevent_destroy`) y quién los lee; `infra/gcp/12-claves.sh` genera cada valor con `openssl rand -base64 48`
y lo manda por una tubería a `gcloud secrets versions add --data-file=-`: no queda en disco, en la línea de comandos ni en el
estado. Es idempotente: no rota una clave existente salvo con `--rotar <secreto>`. Al final comprueba, sin mostrar valores, que
las seis existen, tienen 32 bytes o más y son distintas. api-central las recibe con `secret_key_ref` (`latest`, que se resuelve al
arrancar cada revisión); cada VM las lee en `arranque.sh` con el token de su cuenta y la API REST de Secret Manager (COS no trae
`gcloud`) y las pasa al contenedor por nombre (`-e SESION_CLAVE`), sin el valor en la línea de comandos.

**Copia local de las claves para arrancar sin internet.** Secret Manager es la fuente de verdad: cada arranque que lo alcanza
reescribe `/var/lib/sima/claves/{sesion,servicio}` (0600, solo root, fuera de `pgdata`). Si no lo alcanza (la VM arranca sin
enlace), usa esa copia y lo deja en el registro de arranque; sin Secret Manager ni copia, no arranca. La sucursal tiene que poder
arrancar y vender sin internet, que es el requerimiento principal. *La copia no amplía lo que expone un disco comprometido:* ese
disco ya contiene la base de la sucursal, con los hashes de las contraseñas, y `clave-bd`. *Alternativa descartada:* solo Secret
Manager, que dejaría a una sucursal reiniciada durante un corte sin poder abrir la caja. *Límite:* rotar una clave no llega a una
VM aislada hasta que vuelve a alcanzar Secret Manager en un arranque.

**La semilla de usuarios se aplica siempre.** `aplicar.sh` aplicaba la semilla solo con la base vacía. Las bases de GCP se
sembraron antes del inicio de sesión, con el marcador `SIN_HASH_HITO_1` en lugar de la contraseña, así que después de desplegar
nadie habría podido entrar. Ahora, con `SEMBRAR=1` y la base ya sembrada, `aplicar.sh` vuelve a aplicar `02_usuarios.sql`, que es
idempotente: solo reemplaza el marcador y no toca un hash que ya sea otro. En el central, ese `UPDATE` sube la `version_replica` y
la réplica baja las contraseñas a las sucursales. Las contraseñas son las de demostración del README.

**Variables explícitas en la VM.** `arranque.sh` pasa `SESION_DURACION_H`, `SESION_BLOQUEO_MIN`, `SERVICIO_VENTANA_S` y `REPLICA_*`
con los valores por defecto del código (réplica cada 10 s también en GCP), para que la configuración de la sucursal se lea en un
solo lugar.

**Analítica.** Ninguna vista ni el `MERGE` dependen de una columna que cambió (`traslado.id_usuario_destino` eliminada,
`id_usuario_solicitante` e `id_idempotencia` nuevas, `version_replica`, `proveedor.activo`, `documento.id_usuario` con NULL).
`dim_proveedor` suma `activo`. Las descripciones de `hecho_movimiento` y los comentarios que decían que la recepción de proveedor
no existía se actualizaron; como `03` es `CREATE TABLE IF NOT EXISTS`, las descripciones de la tabla existente las pone
`07_descripciones_hecho.sql`. Lo que Datastream haga con una columna eliminada o con un cambio de NULL en BigQuery no se pudo
probar sin GCP: el procedimiento lo comprueba antes y después (modo `REQUIRED` de las columnas, esquema de `public_traslado` y
`public_proveedor`).

**Verificación.** `verificar-gcp.sh` se actualizó al inicio de sesión (las ventas y lecturas de la sucursal llevan sesión; el
cuerpo ya no lleva `id_usuario`) y a la firma (las consultas al central van firmadas como una sucursal, con su clave leída de Secret
Manager a memoria), y suma la sección 4b: login y roles, réplica al día (y la de otra sucursal, 403), un cambio de catálogo que
llega a las tres y se deshace, idempotencia de ventas, recepción de proveedor que llega al central, un traslado pedido por el
destino con 2PC firmado y una ruta de servicio sin firma rechazada. `carga-gcp.sh` pasa la clave de servicio al k6 de existencias.

## 2026-10-28 — Variante autoalojada (`infra/on-premise`): qué cambia, qué no y cómo no divergir

**Decisión.** La propuesta (sección 8) compromete una rama `infra/on-premise` con Docker Compose, PostgreSQL
administrado por nosotros, replicación lógica y Metabase. Se hace así:

- **La rama solo agrega archivos** (`docker-compose.on-premise.yml` e `infra/on-premise/`). Todo cambio a un
  archivo que ya existe entra primero por `main`. Sincronizar es un `git merge main` sin conflictos, y
  `git diff --stat main...infra/on-premise` muestra 0 líneas en `packages/` y `migraciones/`: la lógica de
  negocio no cambia.
- **`db-central` corre con `wal_level=logical` también en `main`** (`max_slot_wal_keep_size=1GB`). Iguala la
  base local a Cloud SQL (`cloudsql.logical_decoding=on`) y no retiene WAL mientras no haya un slot, que en
  `main` no se crea. Corrige la nota del 2026-10-13 ("La preparación de la base para Datastream no es una
  migración").
- **`scripts/probar-todo.sh` tiene ganchos** que solo actúan con `SIMA_VARIANTE=on-premise`: omiten las suites
  que prueban piezas de GCP (Pub/Sub, iptables, token de identidad, Terraform), corren
  `infra/on-premise/preparar.sh` tras la base limpia y las pruebas de `infra/on-premise/pruebas/`. Sin la
  variable, el script es el de siempre.
- **La carga analítica es replicación lógica de PostgreSQL**, no una carga directa. Datastream ya lee un slot
  `pgoutput` y una publicación (`infra/gcp/migraciones/replicacion.sh`); en la variante esa publicación la
  consume una suscripción de PostgreSQL en `db-analitica`, con el mismo script. La fuente es idéntica en las
  dos ramas y solo cambia el consumidor, que es lo que hace comparable el informe.
- **Hash de la historia SIMULADO con `md5`.** `FARM_FINGERPRINT` no existe en PostgreSQL. La historia sigue
  siendo determinista, pero las cifras no coinciden con las de BigQuery: los tableros se comparan por forma.
- **Ciclo de carga cada 60 s** (`CARGA_INTERVALO_S`). En GCP la consulta programada es de 1 hora.
- **PostgreSQL 16 también en la analítica.** `MERGE … WHEN NOT MATCHED BY SOURCE` es de PostgreSQL 17, así que
  el MERGE se traduce en una transacción: tabla temporal con la fuente, `MERGE` de insertar y actualizar, y
  `DELETE` de lo que ya no está. `LAST_VALUE(… IGNORE NULLS)` no existe: se usa el truco de grupos.

**Transporte de sincronización: el outbox es la "cola en tabla" y el publicador es su trabajador.** No se agrega
una cola nueva. Pub/Sub es un transporte entre la cola y el central, no la cola:

| Garantía | Con Pub/Sub (`main`, GCP) | Variante autoalojada |
|---|---|---|
| Durabilidad sin enlace | `outbox` de la sucursal (Pub/Sub no ayuda: la sucursal sin internet tampoco llega a Pub/Sub) | `outbox` de la sucursal |
| Reintentos | Push de Pub/Sub con espera creciente | Ciclo del publicador cada `SINCRONIZACION_INTERVALO_MS` |
| Mensajes envenenados | Tema de muertos y `muertos.mjs` | `outbox.estado = 'FALLIDO'` tras `SINCRONIZACION_MAX_RECHAZOS`, visible en `/salud` |
| Sin duplicados | `inbox_idempotencia` | `inbox_idempotencia` (la misma) |
| Acuse | Pub/Sub acepta el mensaje; el central lo aplica después | El central ya aplicó cuando responde 200: más fuerte, a cambio de que esté arriba |
| Central caído | Pub/Sub guarda los mensajes hasta 7 días | El outbox crece en la sucursal |

**Alternativa descartada: una tabla de entrada en el central con un trabajador que la vacíe.** Sería una copia
de Pub/Sub, pero duplica la idempotencia, agrega latencia y debilita el acuse. Además la sucursal no puede
escribir en esa tabla directamente (la red no llega a `db-central`, 7.2), así que igual pasaría por la API.

**Alternativa descartada: carga incremental directa desde el contenedor tipo cron.** No cumple "replicación
lógica", consulta la base operativa del central, y su marca de agua tiene el problema que ya descartó la
ventana en BigQuery (2026-10-13, "La consulta programada hace un MERGE completo"): no ve las correcciones de
un documento que llegó después de su movimiento.

**Relojes sincronizados para la firma entre nodos.** La firma HMAC («Autenticación entre nodos») tiene una ventana de ±60 s, así
que exige relojes sincronizados entre quien firma y quien verifica. En GCP lo garantiza NTP. En Docker Desktop la
VM puede atrasarse bajo carga o tras suspender el Mac (se midieron desfases de varios minutos), y entonces toda
petición firmada responde `401 FIRMA_DE_SERVICIO_INVALIDA`: las suites de 2PC, corte de enlace y otras fallan con
síntomas que parecen del sistema (`TypeError … undefined` al leer respuestas de error, transacciones globales
sin cerrar). Se reprodujo atrasando a propósito la VM 120 s, sin la variante. Por eso `scripts/probar-todo.sh`
mide el desfase antes de empezar y aborta si pasa de 5 s (README, «Reloj de Docker»). Es un límite del entorno de
pruebas, no del diseño: no se amplía la ventana, que es lo que la firma promete.

**Riesgos de la replicación lógica que se asumen.** (1) Si la analítica está caída, el slot retiene WAL en el
central: se acota con `max_slot_wal_keep_size` y se vigila con `replicacion.sh estado`. (2) La replicación
lógica no copia DDL: una migración del central sobre una tabla replicada hay que replicarla en el suscriptor;
el ciclo de carga agrega las columnas que falten y se detiene ante cualquier otro cambio. (3) Si el central se
recrea (`down -v`), el slot desaparece: el ciclo detecta la suscripción huérfana y la vuelve a crear.

---

## 2026-10-29 — Fuentes de la caja servidas localmente con `next/font/local`

**Decisión.** La caja (`packages/caja`) carga Onest e IBM Plex Mono con `next/font/local`. Los woff2
(Onest 400–800 en latin y latin-ext; IBM Plex Mono 400/500/600 en latin) viven en `packages/caja/app/fuentes/`,
junto con las licencias `LICENCIA-onest.txt` y `LICENCIA-ibm-plex-mono.txt`. Son de Fontsource, bajo OFL 1.1. Onest
va en dos llamadas (latin y latin-ext) porque cada `@font-face` necesita su propio `unicode-range`. Las variables
`--font-onest` y `--font-plex-mono` se definen en `globals.css`, con el nombre de familia que declara cada
`@font-face` y un fallback de Arial con las métricas de cada fuente. Así el cambio de fuente no mueve el texto.

**Alternativa descartada.** Seguir con `next/font/google`, que descarga las fuentes de Google Fonts en cada build.

**Por qué.** El build ya no depende de internet: la variante autoalojada y el trabajo sin red lo necesitan, y se
verificó con un proxy inválido en `HTTPS_PROXY` (el build original falla con esa condición). Además se evita la
descarga de Google Fonts en cada compilación. La variable CSS va en `globals.css` y no con la opción `variable`
porque, con Turbopack, esa opción deriva el nombre de la familia del identificador de la constante
(`onestLatin`), que no coincide con el `@font-face` (`Onest`): las fuentes no se aplicarían.

---

## 2026-10-29 — Analítica en `sima-red-datos-central`; `db-central` fuera del override

> El override `docker-compose.on-premise.yml` y `infra/on-premise/` viven en la rama `infra/on-premise`, no en main.

**Decisión.** La analítica de la variante autoalojada entra a la red del central, `sima-red-datos-central`: `db-analitica` y `carga-analitica` están ahí, y `db-analitica` además en `sima-red-tableros`. El override `docker-compose.on-premise.yml` no menciona `db-central`: queda como está en el compose base. Metabase (fase 2) va solo en `sima-red-tableros`, que no resuelve `db-central` ni ninguna sucursal. Las sucursales no están en ninguna de las dos redes.

**Alternativa descartada.** Una red de replicación propia (`sima-red-replicacion`) a la que se sumaba `db-central`, que quedaba en dos redes.

**Por qué.** Los scripts de prueba recrean `db-central` con solo `docker-compose.yml`, y al recrearlo le quitaban la red extra: la analítica perdía el camino hacia el central y el slot lógico dejaba de llegar. Con la analítica en la red que ya tiene el central, no hace falta tocar su definición. La regla de los tableros sigue impuesta por la red: lo que se conecte a `sima-red-tableros` no alcanza `db-central` ni las sucursales, y la prueba `carga-analitica` (caso 8) lo verifica.

---

## 2026-10-29 — Metabase en la variante autoalojada

> El override `docker-compose.on-premise.yml` y `infra/on-premise/` viven en la rama `infra/on-premise`, no en main.

**Decisión.** Los tableros de la variante autoalojada se hacen en Metabase (edición de código abierto), en la imagen `metabase/metabase:v0.64.1`, versión fijada: su API de tableros cambia entre versiones y el script que los configura depende de ella. Esa versión salió como estable el 2026-10-06. Los metadatos de Metabase viven en una base propia, `metabase_app`, dentro de `db-analitica`. El contenedor está solo en `sima-red-tableros`. Los cinco tableros de `docs/tableros.md` se crean y actualizan por API con `infra/on-premise/metabase/tableros.mjs`: cada pregunta y cada tablero se buscan por nombre, así que correrlo dos veces no duplica nada. Las consultas son SQL nativo sobre las vistas `v_*`, y la conexión usa `sima_tableros`, un usuario de solo lectura sobre el esquema `analitica`.

**Alternativa descartada.** H2, la base embebida de Metabase: guarda los metadatos en un archivo dentro del contenedor, que se pierde al recrearlo y no se puede consultar. También se descartó la serialización de Metabase (exportar e importar la configuración), que es de la edición de pago.

**Por qué.** Los metadatos en PostgreSQL sobreviven a recrear el contenedor y se pueden inspeccionar. La versión fijada evita que una actualización cambie la API bajo el script. Crear los tableros por API en lugar de a mano deja la configuración en el repositorio y permite repetirla.

**Riesgo asumido.** Metabase bloquea una cuenta tras muchos intentos fallidos de inicio de sesión, y cuenta como fallido un inicio de sesión que falla porque su propia base tiene conexiones muertas, que es lo que pasa tras reiniciar `db-analitica` con `stop` + `start`. El bloqueo dura horas y su contador está en memoria: se borra reiniciando el contenedor. Por eso el script no insiste: como mucho reintenta el inicio de sesión una vez y después aborta con el remedio. Pasó en la práctica: un bucle de intentos bloqueó la cuenta de demostración durante más de 11 horas. Para reiniciar la base, `docker compose restart db-analitica` reinicia también Metabase (`depends_on … restart: true`); tras un `stop` + `start` hay que hacer `docker compose restart metabase`.

---

## 2026-10-29 — El «último movimiento» se ordena por `id_movimiento_origen`, no por `fecha_hora`

**Decisión.** Las vistas de BigQuery (`infra/gcp/bigquery/06_vistas.sql`) que necesitan el último movimiento de un producto en una sucursal (existencia de SIMULADO, cierre diario, primer movimiento) ordenan solo por `id_movimiento_origen`. Antes ordenaban primero por `fecha_hora`. `fecha_hora` se sigue mostrando en `v_movimientos`. Las vistas siempre agrupan por sucursal y producto, así que no hace falta comparar ids entre sucursales. El cambio se aplica en el próximo despliegue de `06_vistas.sql`; no se ejecutó contra BigQuery.

**Alternativa descartada.** Ordenar por `fecha_hora` y desempatar con `id_movimiento_origen`, que era el orden anterior.

**Por qué.** `fecha_hora` es `CURRENT_TIMESTAMP`, que en PostgreSQL es el inicio de la transacción, no el momento del cambio. Una transacción puede empezar antes y tomar el bloqueo de la existencia después de otra: entonces su `fecha_hora` es menor aunque su cambio se aplicó después. `id_movimiento_origen` se asigna al insertar el movimiento, después de tomar el bloqueo y de calcular `cantidad_despues`, así que sigue el orden en que cambió la existencia, con bloqueo pesimista (`packages/nucleo/src/ventas/estrategia-pesimista.ts`). En la variante, con las pruebas de concurrencia, el orden por `fecha_hora` salió al revés en siete pares de movimientos del mismo producto; en uno, el cierre del día daba 32 cuando la existencia real era 20. Si la existencia se escribe sin bloqueo (la estrategia sin control), `cantidad_despues` ya no encadena y ningún orden lo arregla.

**Pendiente.** La copia de la variante (`infra/on-premise/analitica/06_vistas.sql`, rama `infra/on-premise`) tiene el mismo orden por `fecha_hora` y el cambio queda pendiente en esa rama.

## 2026-11-01 — Existencias en la red sin enlace: copia local en la sucursal

**Decisión.** Cada sucursal guarda una copia de la existencia de las otras sucursales en `copia_existencia_red` (migración sucursal 017). La refresca cada `COPIA_RED_INTERVALO_MS` (60 s por defecto) con `GET /existencias` del central, que devuelve la red completa: todas las sucursales × productos activos, con `cantidad`, `version` y `vigente_al` calculados igual que en `consultar-existencias-red.ts`. Cada consulta `GET /red/existencias/:codigo_barras` que responde el central también actualiza la copia de ese producto. La consulta responde con `fuente`:

- `CENTRAL` si el central responde;
- `COPIA_LOCAL` si no se llega al central, o si el central responde con un 5xx. Es un 200 con `copiado_en` y, por sucursal, `vigente_al` y `antiguedad_segundos` = ahora − `vigente_al`;
- 503 `SIN_ENLACE_CENTRAL`, como antes, si no hay enlace y la copia no tiene ese producto.

En los tres casos la fila de esta sucursal sale de su tabla `existencia` (fuente `LOCAL`). La caja (`/red`) muestra un aviso con la hora de la copia y la antigüedad de cada fila, y deshabilita «Solicitar traslado» mientras la respuesta viene de la copia.

**Por qué la copia vive en la sucursal.** Lo que hay que cubrir es la falta de enlace con el central. Un dato guardado en el central o en otro nodo remoto no se puede leer justo cuando no hay enlace. La copia es lo único de la red que la sucursal puede leer en ese momento.

**Por qué la fila propia sale de la base local.** La sucursal es la fuente de verdad de su propia existencia; el consolidado del central y la copia siempre van detrás. Mostrar la fila propia desde la copia haría que una venta durante el corte no se viera en la pantalla de la misma tienda donde se hizo. Por eso la copia nunca guarda la fila propia (`guardarCopiaRed` la descarta) y la respuesta la toma de `existencia` también cuando hay enlace.

**La copia nunca decide nada.** Ninguna venta, ajuste, recepción, traslado ni el 2PC la lee. Las ventas usan su existencia local. Un traslado sigue necesitando a las dos sucursales en línea y bloquea las filas reales en las bases de origen y destino al preparar. Solo cambia lo que se ve en pantalla: un dato viejo de la copia no puede producir una sobreventa ni un traslado que se aplique en una sola punta.

**Partición de red (CAP).** Si una sucursal queda aislada del central, tiene que elegir entre no mostrar nada (consistencia) o mostrar el último dato conocido (disponibilidad). Para esta consulta elige disponibilidad, y el dato va siempre marcado: la fuente, la hora de la copia y la antigüedad de cada sucursal. Con eso el encargado sabe cuánto puede confiar en la cifra. Es la misma regla de oro del proyecto: una consulta global de solo lectura tiene consistencia eventual con su antigüedad a la vista; las operaciones que cruzan sucursales, no.

**La guarda del upsert.** Una fila solo se reemplaza por una con `version` mayor, o con la misma `version` y un `vigente_al` igual o posterior. Así, si una consulta puntual responde tarde y se cruza con un refresco, no hace retroceder la copia. El upsert de cada refresco se hace en una sola transacción. Sin enlace la copia no se toca.

**Temporizador propio, no colgado del replicador.** El replicador central → sucursal avanza un cursor de versiones del catálogo y los usuarios, por páginas, cada 10 s. La copia es una foto completa que se pide cada 60 s. Si compartieran el ciclo, un error en uno detendría al otro, y el intervalo de uno decidiría el del otro. Como el replicador, el servicio (`copia-red.service.ts`) usa su propio pool de una conexión y avisa en el registro solo cuando cambia de estado.

**Alternativas descartadas.**
- *Pedir a las otras sucursales directamente.* Sin enlace tampoco se llega a ellas, y contradice que la consulta de red sale del consolidado del central, sin tocar ninguna sucursal.
- *Guardar en la caja (almacenamiento del navegador).* Cada caja tendría su propia copia, con su propia antigüedad, y la copia se perdería al cambiar de equipo. En la base de la sucursal la comparten todas las cajas, y se puede probar con un script.
- *Mostrar la copia incluso con enlace.* Sería más rápido, pero con enlace el dato del central es más nuevo y no hay motivo para mostrar uno más viejo.

**Límite conocido.** `GET /existencias` crece con sucursales × productos. Con 3 sucursales y 50 productos son 150 filas por minuto por sucursal. Para escalar a quince sucursales con un catálogo real habría que pedir solo lo que cambió desde la última versión copiada, como hace el replicador. Queda argumentado, no implementado. En la VM de GCP, `COPIA_RED_INTERVALO_MS` no se pasa en `infra/gcp/sucursal/arranque.sh` y vale el valor por defecto.

Prueba: `scripts/prueba-red-sin-enlace/prueba.mjs` (en `probar-todo.sh`).

## 2026-11-01 — Costo promedio ponderado móvil, por sucursal

**Decisión.** El inventario se valúa con costo promedio ponderado móvil, **por sucursal** (el art. 41 del Decreto 10-2012 lo permite). El método es fijo y no se configura. Cada nodo valúa con sus datos: la venta, el ajuste, la recepción y el origen de un traslado calculan el costo en la base de la sucursal, bajo el mismo bloqueo de fila que ya tomaban, y ninguna consulta al central participa. `existencia` guarda `cantidad` y `valor_inventario`; el promedio es `valor / cantidad` y no se guarda, se deriva. Dos CHECK: `valor_inventario >= 0` y `cantidad = 0 ⇒ valor = 0`.

- **Entrada:** `valor += costo_total`.
- **Salida:** `costo_total = costo_salida(valor, cantidad, q) = round(valor·q/cantidad, 2)`; si `q = cantidad`, la salida se lleva todo el valor y no quedan centavos huérfanos. La función vive en la base (migración sucursal 018), así que la venta, el ajuste y el origen de un traslado comparten **una sola fórmula**.
- El cálculo es SQL en NUMERIC, dentro del mismo `UPDATE` que cambia la cantidad (`costos/existencia-valorada.ts`). El valor anterior que usa es el que se leyó bajo el bloqueo (pesimista y ajuste) o bajo la versión (optimista): con `version = $v` en el `UPDATE`, un conflicto reintenta y **vuelve a valuar**. No se agrega ningún bloqueo.

**Por qué por sucursal.** Es lo que el requerimiento principal permite: una sucursal sin enlace sigue valuando. Un promedio global tendría que consultar al central en cada venta y rompería la autonomía.

**`costo_total` además de `costo_unitario` (el modelo no cuadraba).** Con salidas redondeadas a centavos, `costo_unitario × cantidad` no da el centavo exacto: 3 u que valen 10.00, vendidas de a una, cuestan 3.33, 3.34 y 3.33. Se agregó `costo_total` a `linea_documento` y a `movimiento`; **manda**, y `costo_unitario` es informativo (4 decimales). La ganancia es `subtotal − costo_total`.

**Alternativas descartadas.**
- *PEPS (FIFO).* Exige capas por lote, o sea una tabla nueva, otro bloqueo y reglas de agotamiento de capas en cada venta. El promedio móvil cabe en una fila que ya se bloquea.
- *Promedio global calculado en el central.* Rompe «una venta nunca consulta al central».
- *Guardar el costo promedio en la fila.* Es un dato derivado que podría desincronizarse de `valor` y `cantidad`; derivarlo no puede.
- *Calcular el costo en TypeScript.* Flotantes de JS acumulan error; el resto del sistema ya hace la aritmética de existencias en SQL.

Prueba: `scripts/prueba-costos/prueba.mjs` (casos 1 a 3, 7, 8 y 11).

## 2026-11-01 — Lista de costos del distribuidor: replicada, inmutable, con regla de margen

**Decisión.** Un solo distribuidor con lista de costos fija: `costo_proveedor (id_proveedor, id_producto, costo_unitario, vigente_desde)`, maestra en el central (migración 019) y copia en cada sucursal (018), que baja con la réplica existente (la `version_replica` sale del mismo contador serializado). Solo un ADMIN crea una vigencia nueva (`POST /red/costos`, que se reenvía al central como `/red/productos`); **nunca se actualiza ni se borra una vigencia** (trigger, como `movimiento`): la nueva reemplaza a la anterior desde su `vigente_desde`. La vigencia que aplica es la más reciente con `vigente_desde` ya pasado. Se escribe con `clock_timestamp()` y con el contador ya tomado, para que dos vigencias del mismo par queden en el orden en que se crearon.

**Regla de margen.** `costo < precio_venta`, o 422 `MARGEN_NO_POSITIVO`. Se valida al crear la vigencia y al cambiar `precio_venta`. Las dos escrituras toman primero `contador_replica FOR UPDATE`, el bloqueo que ya serializa toda escritura de catálogo, así que una vigencia y un cambio de precio nunca se validan a la vez cada uno contra el valor viejo del otro. En inventario no entra ningún bloqueo nuevo.

**Alternativa descartada.** *Un CHECK en la base.* El costo y el precio viven en tablas distintas y el CHECK entre tablas no existe; un trigger sí podría, pero la validación de API devuelve el 422 con el detalle y la serialización por contador evita la carrera sin trigger.

Prueba: `prueba-costos`, caso 6.

## 2026-11-01 — La recepción la valúa el servidor, con la lista replicada

**Decisión.** `POST /recepciones` ya no recibe `costo_unitario`: el servidor toma la vigencia de la copia **local** de la lista, así que funciona sin enlace con la última lista que llegó. Si el cliente manda un costo, se ignora y **no entra en la huella de idempotencia**. Sin vigencia para algún producto, 422 `SIN_COSTO_VIGENTE` con todos los que faltan y no se escribe nada. La línea guarda `id_costo_proveedor`: qué vigencia usó. `GET /recepciones/catalogo` trae `costos` (las vigencias de hoy) y la pantalla de recepción las muestra **de solo lectura**.

**Por qué.** Un costo escrito a mano en cada recepción permite cualquier valor y vuelve el promedio imposible de auditar. Con la lista, el costo es un dato del negocio, con historia y con responsable.

**Efecto en las pruebas.** Cambian recepción, idempotencia y catálogo (necesitan vigencias). En idempotencia, el caso «otro costo» deja de ser un 409 (el costo ya no es parte de la solicitud) y pasa a ser «otra cantidad».

## 2026-11-01 — Traslado: el costo viaja en el voto del origen; la preparación es en secuencia

**Decisión.** El origen valúa lo que sale a su promedio, en su `PREPARE` y bajo su bloqueo, lo guarda en sus líneas y en el movimiento y lo devuelve en su voto SÍ (`VotoParticipante.costos`). El coordinador lo escribe en las líneas de su propio documento (su bitácora) **antes** de hablar con el destino, y solo entonces le pide preparar, con los costos en el cuerpo. El destino los guarda en sus líneas `EN_TRANSITO` **dentro de su transacción preparada**; la recepción los lee de ahí. Resultado: la recepción del destino sigue siendo local y funciona con o sin enlace, exactamente como hoy; usa `valor += costo_total` (el mismo del origen, sin redondear de nuevo) y recalcula su promedio. Lo que sale del origen es, al centavo, lo que entra al destino; un traslado no genera ganancia. La conservación del valor en la red cuenta lo que está en tránsito (`verificar.mjs`).

- **La preparación pasa de paralela a secuencial.** El costo es lo que el origen decide al preparar; el destino no puede prepararse sin conocerlo. El costo es latencia: hasta dos plazos de votación (10 s cada uno por defecto) en vez de uno. `RECUPERACION_PLAZO_INICIADO_MS` (60 s) sigue siendo mayor; y la decisión es condicional (`UPDATE … WHERE estado = 'INICIADO'`), así que aunque la recuperación se adelantara el resultado sería correcto.
- Si el origen vota NO, o SÍ sin costos válidos, el destino **no se consulta** y la decisión es ABORTADO. Abortar antes de decidir siempre es seguro. El destino queda `NO_CONSULTADO` y recibe igual el `ROLLBACK` (responde `NO_PREPARADA`).
- **El voto va firmado.** La respuesta de `preparar` no estaba firmada (solo las peticiones lo estaban). Como ahora lleva los costos, la sucursal firma el cuerpo exacto de su respuesta con su clave (`X-SIMA-Firma-Respuesta`, método `RESPUESTA`, ruta de la petición) y el coordinador la verifica antes de creerla; un voto sin firma válida cuenta como «sin respuesta». Es el mismo HMAC y la misma ventana de ±60 s.

**Alternativas descartadas.**
- *Mandar el costo en «confirmar».* `COMMIT PREPARED` no puede escribir nada: el costo habría que guardarlo en una transacción aparte; si el nodo cae entre las dos, el traslado queda confirmado y sin costo, y la recepción tendría que pedírselo al central. Se rompe lo que se quería conservar: que la recepción sea local.
- *Que el destino recalcule con su propio promedio.* La mercadería no vale lo que valía en el destino, sino lo que salió del origen; recalcularla crearía o destruiría valor entre las dos puntas.
- *Valuar el traslado en el central.* El central no tiene el promedio de ninguna sucursal y la regla es que no valúa.

Un traslado anterior al costo promedio no trae costo en sus líneas: la recepción lo valúa al costo de lista vigente (caso raro, solo en el cambio de versión).

Prueba: `demo-2pc` (escenarios 1 y 7) y `prueba-costos` (casos 9 y 10).

## 2026-11-01 — El coordinador muere después del voto SÍ del origen: se aborta por plazo

**Caso.** El origen votó SÍ (con sus costos), el coordinador los escribió en su bitácora y murió **antes** de preguntarle al destino. La transacción global queda `INICIADO`; el origen tiene una transacción preparada que retiene la fila; el destino nunca fue consultado.

**Decisión: la recuperación aborta por plazo**, igual que el escenario 5 (`INICIADO` con más de `RECUPERACION_PLAZO_INICIADO_MS` → `ABORTADO` → se propaga `ROLLBACK PREPARED`).

**Por qué.** En `INICIADO` nadie pudo haber confirmado: abortar es siempre seguro, y la tabla de recuperación ya dice exactamente eso. Retomar (volver a preguntarle al destino con los **mismos** costos guardados) habría hecho a la recuperación un segundo coordinador del protocolo: tendría que reconstruir la solicitud, reenviar el `PREPARE` y decidir si el plazo original sigue valiendo. A cambio de esa complejidad solo se ahorra que el usuario repita el traslado. Mientras tanto la fila del origen queda retenida (visible: `/salud`, 409 `PRODUCTO_BLOQUEADO` en las ventas del producto), acotado por el plazo.

**Lo que sí se conserva.** Los costos quedan en la bitácora del coordinador (líneas de su documento, que queda `ANULADO`) y, de forma persistente, en el origen (`voto_traslado`): el rastro de lo que se votó sobrevive a la caída.

Prueba: `demo-2pc`, escenario 7 (aparece en `docs/resultados-2pc.md`).

## 2026-11-01 — Un PREPARE repetido del origen contesta con los costos de la primera vez

**Decisión.** Si el coordinador reintenta el `PREPARE` del origen (se perdió la respuesta), el origen contesta SÍ con `ya_preparada: true` y los costos que calculó **la primera vez**; nunca vuelve a valuar. Esos costos viven dentro de la transacción preparada, que nadie puede leer hasta el `COMMIT PREPARED`; por eso el origen los guarda también en `voto_traslado` (migración sucursal 018) con **otra conexión**, justo antes del `PREPARE`, mientras aún tiene el bloqueo de la fila.

**Por qué esa tabla es segura.**
- Se lee solo cuando `pg_prepared_xacts` confirma que el gid está preparado. Si el `PREPARE` falló, queda una fila huérfana que nadie lee y que el siguiente intento reemplaza.
- Si dos intentos del mismo traslado chocan, el segundo espera en el índice único `(id_traslado, tipo)` del documento, antes de calcular nada; no puede pisar los costos del primero.
- Es local: no se replica ni va al central.

**Lo que se descubrió al diseñar la prueba.** El caso pedido era «preparar, recibir mercadería en el origen (cambia el promedio) y reintentar el PREPARE». Con una transacción preparada real **no se puede**: la fila de existencia está retenida por la preparada, así que la recepción da 409 `PRODUCTO_BLOQUEADO`, y el promedio **no puede cambiar mientras el traslado está en duda**. Es la propiedad que hace correcto todo el diseño. La prueba lo comprueba y demuestra lo que importa con tres cosas: (1) la recepción se rechaza mientras está preparado; (2) el reintento devuelve los mismos costos, también tras reiniciar la API; (3) alterando a mano el costo guardado, el reintento devuelve el alterado, o sea que **lee la tabla y no recalcula**. Y para mostrar la diferencia con un intento nuevo: tras abortar y recibir mercadería, un `PREPARE` nuevo sí valúa con el promedio de hoy.

**Alternativa descartada.** *Leer los costos del documento de la transacción preparada.* No es visible desde otra sesión; el motor lo impide.

Prueba: `prueba-costos`, caso 10.

## 2026-11-01 — Saldo inicial, histórico sin costo y `SIN_SALDO_INICIAL`

**Decisión.** Lo que ya existía no tiene costo y **no se inventa**. La migración deja `valor_inventario` en NULL (en 0 si la cantidad es 0), y el histórico de movimientos queda explícitamente sin costo (sus cuatro campos de costo, en NULL). El cárdex valorizado **arranca** con un saldo inicial: `registrar_saldo_inicial(id_admin)` valúa cada existencia al **costo de lista vigente** y deja el rastro: un documento AJUSTE «Saldo inicial al costo de lista» por sucursal, un movimiento `SALDO_INICIAL` por producto (no mueve unidades: `cantidad_antes = cantidad_despues = cantidad`, `valor_antes = 0`), la existencia con su valor y `version + 1`, y la fila de outbox, todo en una transacción.

- **Idempotente.** Solo toca las existencias con valor NULL: correrla dos veces no encuentra nada y no duplica documentos, movimientos ni avisos. Devuelve los productos que no pudo valuar (sin lista).
- **Mientras haya una existencia sin valor**, una venta, recepción, ajuste o traslado de ese producto responde 409 `SIN_SALDO_INICIAL` (el origen de un traslado vota NO por esa razón). Es mejor bloquear que operar sobre un valor que no existe. La caja muestra el 409 al encargado con un mensaje claro.
- En desarrollo no ocurre nunca: `sembrar.sh` corre el saldo inicial. La semilla trae la lista del distribuidor para los 50 productos con márgenes por categoría (18 % a 35 %), y la existencia sembrada entra con su saldo inicial.
- **En GCP es un paso obligatorio** de la ventana (`docs/despliegue-gcp.md`, 4.3 bis): la base ya está sembrada y no se vuelve a sembrar. `infra/gcp/migraciones/aplicar.sh` agrega la lista de costos al central si está vacía, y `verificar-gcp.sh` **falla** si queda alguna existencia sin saldo inicial o algún producto activo sin costo en la lista.

**Alternativa descartada.** *Valuar en la migración.* En el momento de migrar, la lista de costos aún no llegó a las sucursales por la réplica; haría falta inventar un costo.

Prueba: `prueba-costos`, caso 12.

## 2026-11-01 — Sincronización (HTTP y Pub/Sub): los campos de costo son opcionales

**Decisión.** Los documentos y los movimientos del mensaje de sincronización llevan los campos de costo (`costo_unitario`, `costo_total`, `id_costo_proveedor` en las líneas; `costo_unitario`, `costo_total`, `valor_antes`, `valor_despues` en los movimientos) y `existencia_consolidada` agrega `valor_inventario`, con la misma guarda de versión que la cantidad. Hay **un solo consumidor** para los dos transportes, así que el cambio vale para ambos. **Los campos son opcionales**: un mensaje anterior al costo promedio (en el outbox de una sucursal que aún no se actualizó, o redelivery de Pub/Sub) se valida y se aplica con costo NULL, y deja el valor del consolidado en NULL (desconocido, no un dato viejo).

Si el movimiento trae costo, el validador exige los cuatro campos juntos y la aritmética (`valor_antes ± costo_total = valor_despues`); el motor lo repite con CHECK en las dos bases. `SALDO_INICIAL` solo es válido dentro de un AJUSTE y no mueve unidades. Las suites `consumidor` y `publicador`, que arman mensajes sin esos campos, siguen pasando sin cambios; el caso 15 de `prueba-costos` y la existencia histórica del caso 12 (que viaja al central en el formato viejo) lo comprueban.

**Montos con IVA incluido.** Precios y costos llevan el 12 % incluido. Los reportes muestran también la base sin IVA (monto ÷ 1.12).

## 2026-11-01 — Reportes locales y auditoría del cárdex

**Decisión.** Todo es local a la sucursal y funciona sin enlace, para ENCARGADO y ADMIN (el CAJERO nunca ve costos ni ganancia: la respuesta de la venta no los trae y las rutas dan 403).

- `GET /kardex/:codigo_barras?desde&hasta`: fecha, documento, tipo, entrada, salida, costo unitario, saldo en cantidad, costo promedio y saldo en valor; hasta 366 días (es el cárdex de un solo producto). Lo anterior al saldo inicial sale «sin costo».
- Reporte del día y ventas: costo de ventas, utilidad bruta, margen % y por producto, merma del día (las salidas de ajuste son merma, al promedio) y las líneas sin costo aparte. Las ventas anteriores al costo promedio no entran en la utilidad.
- `GET /inventario/valor`: total y por categoría.
- `GET /auditoria/kardex`: devuelve las discrepancias (`cuadra: true` y lista vacía si todo cuadra): cadena de saldos sin huecos en cantidad y en valor, `existencia = último saldo del cárdex`, `cantidad 0 ⇒ valor 0`, existencias sin saldo inicial, y Σ costo de las líneas de venta = Σ costo de los movimientos VENTA (por documento y en total).
- `verificar.mjs` compara el valor entre central y sucursales, y verifica el **libro de valor** de cada sucursal y la **conservación del valor en la red**: Σ valor de las existencias + valor en tránsito = Σ saldos iniciales + recepciones + ajustes de entrada − costo de ventas − mermas.

**Por qué no en el central ni en BigQuery.** Son consultas operativas de una sucursal; la analítica de rotación y capital dormido sigue en el almacén analítico. Queda pendiente, y no se hizo en esta rama, llevar `costo_total`, `valor_antes` y `valor_despues` a `hecho_movimiento` y valuar el capital dormido al costo, en BigQuery y en la variante autoalojada (analítica, esquema de carga y consultas de Metabase).

## 2026-11-02 — Demo híbrida: la sucursal 1 corre en la Mac y habla con GCP como `sa-sucursal-1`, sin llaves

**Decisión.** Para la demo final, la sucursal 1 (Centro) corre en la Mac con Docker (`docker-compose.hibrido.yml`: `db-sucursal-1`, `api-sucursal-1`,
`acceso-sucursal-1` y `reloj-docker`; el resto de los servicios del compose base queda en el perfil `solo-local`). El central y las sucursales 2 y 3 siguen en
GCP. El corte de enlace es apagar el Wi-Fi de la Mac. La API fuera de GCP usa `CENTRAL_AUTENTICACION=SUPLANTACION`: las credenciales de usuario de quien opera
(ADC, montado **de solo lectura**) piden a IAM Credentials un token de identidad de `sa-sucursal-1` para api-central (que es privado) y el mismo cliente
`Impersonated` es el `authClient` con el que se publica en Pub/Sub. Resultado: Centro llama al central y publica **con la misma identidad que la VM**, con
el mismo adaptador de transporte (`PUBSUB`) y el mismo consumidor push en el central. Las claves `SESION_CLAVE` y `SERVICIO_CLAVE` se leen de Secret Manager
al preparar (también suplantando a `sa-sucursal-1`, que es la única que las puede leer) y quedan en `~/.sima-hibrido/` (0700; archivo 0600), fuera del repositorio.

**Terraform (`infra/terraform/cuentas.tf`).** El permiso es `roles/iam.serviceAccountTokenCreator` **solo sobre `sa-sucursal-1`**, para el usuario que corre
Terraform (`data "google_client_openid_userinfo"`: el correo no queda escrito en el repositorio; `TF_VAR_operador_correo` lo reemplaza, y una `precondition` hace
fallar el plan, nunca quitar el permiso, si no hay un correo de usuario). `iamcredentials.googleapis.com` se habilita en `apis.tf`.

**Alternativas descartadas.**
- *Una llave JSON de `sa-sucursal-1` descargada en la Mac.* Es una credencial de larga vida sobre un disco portátil; justo lo que se evitó con Secret Manager y una cuenta por sucursal (2026-10-28).
- *El token de identidad del usuario (`gcloud auth print-identity-token`).* El central privado solo admite `roles/run.invoker` a las cuentas de servicio listadas (`acceso-central.tf`) y la firma HMAC es por sucursal; el usuario no es `sa-sucursal-1` y no podría publicar en Pub/Sub como ella.
- *Sincronizar por HTTP directo al central, sin Pub/Sub.* Habría sido otro camino de código que el de las VMs, y lo que se demuestra (acuse de Pub/Sub, publicación durable, reintentos del push) no se vería. Con la suplantación el camino es idéntico.
- *Un metadata server falso en la Mac.* Reproduce una API de Google en un contenedor para evitar usar la API real de suplantación.
- *Dar el permiso de suplantación sobre el proyecto.* Abriría las claves de las otras sucursales y del central.

**Lo que se probó y lo que no.** `scripts/prueba-token-identidad/prueba.mjs` (secciones 5, 5b y 5c) usa el cliente `Impersonated` **real** de `google-auth-library` contra un IAM Credentials
falso: ruta, cuenta, audiencia, `includeEmail`, credencial de origen, caché, 403 → `TRANSITORIO`, 5xx/sin red/plazo → `SIN_ENLACE`, y que un archivo ADC que falta o que es una llave de
cuenta de servicio **no** deja arrancar la suplantación (la API arranca igual y lo dice en `/salud` con `ADC_NO_DISPONIBLE`). Se levantó el stack híbrido con un ADC falso y se vio el
mensaje `TOKEN_IDENTIDAD` en `/salud`. **No se probó contra Google real** (necesita el permiso aplicado por Terraform): lo primero que hace `preparar.sh` es exactamente esa prueba.

## 2026-11-02 — Nunca dos nodos «sucursal 1»: candado en la VM; sin guarda en el central

**Decisión.** La VM `sucursal-1` lleva el metadato `nodo-externo=<marca del traspaso>` mientras la sucursal 1 vive en la Mac. Con el candado, `arranque.sh`
borra el contenedor `api-sucursal`, deja arriba solo `db-sucursal` (para poder restaurarle la base en el regreso), y sale **sin migrar, sin leer claves y sin arrancar la API**.
El traspaso además le quita el reinicio automático al contenedor (`docker update --restart=no`), porque Docker lo reinicia antes de que corra el script. `encender-apagar.sh encender` se niega con el
candado puesto y `encender-hibrido` enciende todo menos la VM 1 (y exige el candado). `preparar.sh` aborta si la VM está `RUNNING`, si no tiene candado, o si el candado no es el de la
marca de **esta** Mac (`~/.sima-hibrido/traspaso.env`): así una Mac con una base vieja no se hace pasar por la sucursal 1. `verificar-gcp.sh` se niega a correr en modo híbrido. El metadato lo pone y
lo quita `gcloud` (el traspaso y el regreso), no Terraform (`ignore_changes = [metadata["nodo-externo"]]`).

**Guarda en el central: descartada.** Para rechazar a un segundo nodo con el mismo id, el central necesitaría distinguir dos nodos que presentan **la misma clave de servicio y la misma cuenta**. Habría que
agregar una identidad de nodo al protocolo, una migración (020) y una regla de traspaso; y esa regla rechazaría justamente el cambio VM → Mac que se hace a propósito. Además no hace falta para que
el daño sea visible: los dos nodos nacen del mismo volcado, así que repiten `id_documento` y `id_movimiento`, y `UNIQUE (id_sucursal, id_documento_origen)` hace que el central responda 409
`CONFLICTO_DATOS` al segundo y los mensajes terminen en el tema de muertos, que `preparar.sh` exige vacío. No se mezcla en silencio. El procedimiento (candado + comprobaciones + una sola Mac) basta para dos personas y una demo; con quince
sucursales reales sería otra conversación (identidad de nodo con certificados), que queda argumentada, no implementada.

## 2026-11-02 — La base de Centro se traspasa (pg_dump), no se reinicia a la semilla; y se protege

**Decisión.** El estado inicial de la Mac es **la base de la VM, copiada**: `traspaso.sh` comprueba que no se pierde nada (cola vacía y sus últimas 50 llaves ya en el inbox del central,
sin transacciones preparadas, sin traslados abiertos, última migración, sin existencias sin saldo inicial), detiene la API de la VM, vuelca con `pg_dump -Fc` por IAP, restaura en la Mac y **compara una firma
de las dos bases** (documentos, movimientos, líneas, existencias, cantidad, valor, outbox, usuarios, lista de costos, copia de la red, migración). Recién después pone el candado y apaga la VM. Si algo falla
antes de ese punto, repone la API de la VM. El **regreso** (`regreso.sh`) es el camino inverso con las mismas comprobaciones: se niega con traslados `SOLICITADO` con la sucursal 1 (los lista con su id y su fecha) salvo `--forzar`,
espera la cola, respalda la base vieja **en la propia VM** (`/var/lib/sima/antes-regreso.dump`), restaura la de la Mac, compara firmas y recién entonces quita el candado.

**Alternativa descartada: reiniciar todo a la semilla en ambos lados.** Borraría la historia de Cloud SQL, del CDC y de BigQuery justo antes de la revisión de Cloud, que es lo que se quiere mostrar.
Costo asumido: es un paso más, con una ventana en la que nadie vende.

**Protección.** Después del traspaso, el volumen `sima-hibrido_datos-sucursal-1` es la **única copia al día** de la sucursal 1. Por eso: (1) ningún script usa `down -v`, `volume rm` ni `prune` (el único comando de
bajada es `bajar_stack`, `docker compose down` sin volúmenes); (2) `terminar.sh` guarda un volcado verificado (`pg_restore -l`) en `~/sima-respaldos/` (0700/0600, con fecha, los últimos 5) **antes** de bajar y no baja si falla;
(3) `preparar.sh` avisa si el último respaldo tiene más de 24 h; (4) la leyenda «La base de Centro vive en esta Mac: no borrar volúmenes ni resetear Docker Desktop hasta el regreso a la VM» aparece en `preparar.sh`, `terminar.sh`,
los scripts de traspaso y el compose. La base y la API de la Mac escuchan solo en `127.0.0.1` (el compose base las publica en todas las interfaces con la contraseña de desarrollo).

## 2026-11-02 — Traslados con la sucursal 1 en la Mac: `SOLICITADO`, nunca en duda; en producción iría por VPN

**Qué pasa.** El coordinador (central) llama a las sucursales, no al revés. El paso 2 de `ejecutarProtocolo` pregunta primero `/salud` a las dos (`coordinador.ts`): si una no responde
(la Mac no es alcanzable desde la VPC), **no se escribe nada de la fase 1**: el traslado queda `SOLICITADO`, sin transacción global, sin participantes y sin ninguna transacción preparada, con el motivo
`SUCURSAL_FUERA_DE_LINEA: 1`. Es el escenario 3 de `demo-2pc` (`docs/resultados-2pc.md`). Nunca una transacción en duda. La caja lo dice claro (`packages/caja/app/traslados/page.tsx`, `explicarEspera`): «El coordinador del
central no llegó a Sucursal Centro: no se movió nada en ninguna sucursal… Esta sucursal vende y recibe con normalidad, pero el central no puede llamarla para coordinar el traslado».
La recepción de un traslado que **ya** estaba comprometido sí funciona en Centro: es local.

**El riesgo que sí hay.** Un `SOLICITADO` con la sucursal 1 no desaparece: en cuanto la VM vuelve, la recuperación del central (cada minuto) lo retoma y lo **ejecuta**. Por eso `regreso.sh` se niega mientras exista alguno
(salvo `--forzar`), y `preparar.sh` y `terminar.sh` los listan.

**En producción** la tienda iría por **VPN** (HA VPN y Cloud Router, o un túnel de la tienda a la VPC) con una IP privada estable, y el central la llamaría como a las demás. **No se hace aquí** porque (1) una HA VPN cobra por hora
de túnel y de puerta de enlace durante días para una demostración de horas, con un crédito de prueba que vence el 2027-01-10; (2) la Mac está detrás de un NAT doméstico y de un Wi-Fi que cambia de red, y un túnel IPsec
exigiría un dispositivo de borde con IP pública o un equipo con cliente VPN estable, y apagar el Wi-Fi para el corte lo tiraría como cualquier otro tráfico; y (3) tiempo: son dos personas con una revisión en dos días.
*Alternativa descartada:* un conector inverso (la sucursal mantiene abierta una conexión saliente y el coordinador la usa). Cambiaría el protocolo del 2PC para un caso que solo existe en la demo.

## 2026-11-02 — Cajas de las sucursales 2 y 3: túneles por SSH sobre IAP, no `start-iap-tunnel`

**Decisión.** `infra/hibrido/tuneles.sh` abre `gcloud compute ssh sucursal-N --tunnel-through-iap -- -N -L 127.0.0.1:300N:localhost:3000` con `ServerAliveInterval=5`, `ServerAliveCountMax=3` y
`ExitOnForwardFailure=yes`, bajo un supervisor en segundo plano que lo reabre a los 3 s si se cae (al apagar y prender el Wi-Fi, en ~15 a 20 s). No se pisan puertos ocupados por el compose local.

**Alternativa descartada: `gcloud compute start-iap-tunnel sucursal-N 3000`.** La regla de firewall `permitir-ssh-iap` (`infra/terraform/red.tf`) solo deja entrar a los rangos de IAP por **tcp:22**; el 3000 nunca recibe la conexión.
El mismo error estaba en el paso 4.5 de `docs/despliegue-gcp.md` (corregido). Abrir el 3000 a IAP era otra opción, descartada: es una regla de firewall más, con un puerto de aplicación expuesto por IAP, para algo que SSH ya resuelve sin cambiar la red.

## 2026-11-02 — Reloj: la firma tolera ±60 s contra Cloud Run, y la VM de Docker se atrasa

**Decisión.** El stack híbrido incluye un servicio `reloj-docker` (privilegiado, sin red) que corre `hwclock -s` cada 30 s, el arreglo que ya documenta el README («Reloj de Docker»). `preparar.sh` mide dos desfases: VM de Docker contra la Mac, y
Mac contra Google (encabezado `Date` de googleapis.com); avisa pasados 5 s (el umbral de `probar-todo.sh`) y **falla pasados 30 s**, la mitad de la ventana de la firma. La ventana (`SERVICIO_VENTANA_S=60`) no se amplía.
**Alternativa descartada:** ampliar la ventana. Es lo que la firma promete (2026-10-25) y ampliarla debilitaría la protección contra repetición para arreglar un problema de la laptop.

## 2026-11-02 — El costo llega al esquema estrella de BigQuery

**Decisión.** `hecho_movimiento` suma `costo_total`, `valor_antes` y `valor_despues` (del movimiento) y `costo_linea` (Σ `linea_documento.costo_total` del documento y el producto, solo VENTA). El costo del movimiento es el que manda;
`costo_linea` existe para la **auditoría cruzada** (`costo_total = costo_linea` en toda venta, que `verificar-gcp.sh` y `v_utilidad.ventas_costo_distinto` comprueban), igual que la auditoría local compara líneas y movimientos.
`v_utilidad` (sucursal, día, producto) calcula unidades, venta, costo, utilidad, margen y la base sin IVA (÷ 1.12) **solo con las ventas que tienen costo**; las anteriores al costo promedio van aparte (`unidades_sin_costo`, `venta_sin_costo`) y
no inflan la utilidad. La historia SIMULADO queda con NULL: «sin costo», no cero. El `MERGE` compara con `IS DISTINCT FROM`, que trata el NULL como un valor, así que el histórico sin costo no se marca como cambiado cada hora.
Los movimientos `SALDO_INICIAL` entran al hecho (`delta = 0`) y arrancan el cárdex valorizado; las vistas de rotación y capital dormido filtran `tipo = 'VENTA'` y no los ven.

**Datastream.** La publicación `sima_cdc` incluye las tablas enteras, así que las columnas nuevas llegan a `sima_cdc.public_*` cuando llega el primer evento que las trae (el mismo comportamiento que `proveedor.activo`, 2026-10-28); las ventas de la primera
pasada de `verificar-gcp.sh` los producen. `costo_proveedor` **no** se replica a BigQuery: la utilidad usa el costo que ya viaja en cada movimiento. Como el `MERGE` falla con «Unrecognized name» si las columnas no existen, su cambio
**no** se aplica con Terraform hasta que existan (`docs/despliegue-gcp.md`, A10), y `09-bigquery.sh` lo comprueba antes de correrlo. Plan B si una columna no aparece: `backfill_tabla` (`operaciones-datastream.sh`).
**No se pudo probar contra BigQuery ni Datastream**: el SQL se revisó a mano y Terraform se validó con `terraform validate`; la prueba es A10/A11.
*Pendiente (igual que el 2026-11-01):* valuar el capital dormido al costo (hoy a precio de venta) y llevar el costo a la variante autoalojada.

## 2026-11-02 — Lo que la demo híbrida cambia en el sistema, y lo que no

**Decisión (sin cambio de diseño).** La demo no agrega funcionalidad al sistema: el corte de enlace de la copia de red (`COPIA_LOCAL`, 2026-11-01) y la auditoría vacía se muestran tal como existen. Lo único que se
toca en el código de las sucursales es la identidad hacia GCP (suplantación), el mensaje de espera de un traslado `SOLICITADO` y la dependencia explícita `google-auth-library` 11.3.1 en `packages/api-sucursal`,
que ya estaba en el lockfile como dependencia transitiva de `@google-cloud/pubsub` en esa misma versión (no se descarga nada nuevo). `verificar-gcp.sh` suma los chequeos del costo (4c), la copia de red durante el corte por `iptables`
(sección 5) y la auditoría vacía (5b), y una opción `--sin-analitica` para la primera pasada del redespliegue.

## 2026-11-03 — Un campo nuevo en la página de réplica lo descarta un nodo viejo que igual avanza el cursor

**Qué pasó (despliegue en GCP sobre bases existentes).** El job de migración del central (A4) siembra `costo_proveedor` con `version_replica` nuevas (161 a 210). Entre A4 y A7
(cuando las VMs pasan al código nuevo) las sucursales seguían con el código **viejo**: pidieron esas páginas de réplica, el código viejo no conocía el campo `costos` de la página
y lo **descartó**, y de todos modos avanzó `estado_replica.ultima_version` (hasta 212). Con el código nuevo, el central ya no tenía nada «más nuevo que 212» para ellas: 0 costos en las
sucursales, saldo inicial imposible (`SIN_SALDO_INICIAL`) y 22 fallas en cascada en `verificar-gcp.sh`. Se corrigió a mano con `UPDATE estado_replica SET ultima_version = 160` en cada
sucursal. Es seguro porque la réplica solo pisa filas con una versión **mayor** (`GREATEST` en el cursor; las vigencias de costo son inmutables y se insertan sin duplicar): repetir páginas ya
aplicadas no cambia nada.

**La lección general.** En una réplica por cursor, el cursor es una promesa de «ya tengo todo hasta aquí», y un nodo viejo la hace sin tener con qué cumplirla cuando la página trae un campo
que no entiende (JSON tolerante a campos desconocidos: lo ignora sin quejarse). Agregar un campo a la página de réplica es, por lo tanto, un cambio **ordenado en el tiempo**: lo que se
sembró con versiones nuevas antes de que todos los nodos lean el campo, hay que volver a mandarlo.

**Decisión.** El runbook (`docs/despliegue-gcp.md`, **A7 bis**, entre A7 y A8) compara la cantidad de `costo_proveedor` del central con la de cada sucursal y, si a una le faltan, regresa
su cursor a `min(version_replica) − 1` de la lista de costos del central. Va **después** de A7 (con el código viejo la réplica volvería a descartar el campo) y es el único paso que escribe
`estado_replica` a mano. `verificar-gcp.sh` ya lo detecta (la lista tiene que ser idéntica en el central y en las tres sucursales) y su mensaje remite a ese paso.

**Alternativas descartadas.**
- *Reordenar el despliegue (VMs antes que el job del central).* La lista de costos la siembra el job de migración del central (`aplicar.sh`), y el saldo inicial depende de ella: el orden «central primero» se
  mantiene para todo lo demás de la ventana y un paso de comprobación cubre este caso sin cambiarlo.
- *Corregirlo en el código ahora* (que la sucursal vieja no avance el cursor ante campos desconocidos). Ese código es justo el que ya está desplegado: no se puede cambiar retroactivamente.

**Pendiente (no implementado, a propósito).** Para la próxima vez que la página de réplica cambie: que el replicador **falle cerrado** ante un campo desconocido en la página (no avanzar el cursor y
registrar el campo), o que la página lleve una versión de esquema que el nodo viejo pueda rechazar. Queda argumentado; el alcance recortado no lo incluye.

## 2026-11-03 — Centro alcanzable por el coordinador: túnel SSH inverso por IAP hacia la VM sucursal-1, que sigue con candado

**Qué pasaba.** En el modo híbrido el coordinador del 2PC (Cloud Run) no podía llamar a la API de la sucursal 1, que corre en la Mac detrás de un NAT, y todo traslado con Centro quedaba `SOLICITADO` (2026-11-02). Para la demo se quiere el traslado
Zona 10 → Centro con 2PC de verdad.

**Decisión.** La Mac abre hacia afuera un túnel SSH **inverso** (`-R`) por IAP hasta la VM `sucursal-1`, que se enciende **con el candado `nodo-externo`** (solo su base; su API nunca arranca). En la VM, un reenviador TCP escucha **solo en `10.10.0.11:3000`** (la IP
reservada de siempre de la sucursal 1) y manda el tráfico al extremo del túnel, que lo lleva a `127.0.0.1:3001` de la Mac (la API de Centro). **El central no cambia**: sigue llamando a `http://10.10.0.1{id}:3000` (`SUCURSAL_URL_PLANTILLA`).
La firma entre nodos cubre método, ruta y cuerpo, no el host (`rutaFirmable`), así que un reenvío TCP no la altera.

```
api-central ─▶ 10.10.0.11:3000 ─▶ reenvio-centro (VM, solo IP interna) ─▶ 127.0.0.1:131xx ═ túnel SSH inverso por IAP ═▶ Mac 127.0.0.1:3001 ─▶ api-sucursal-1
```

**Piezas.** `infra/hibrido/tuneles.sh` (el túnel 1 lo vigila el mismo supervisor que los de 3002 y 3003 y se reabre a los 3 s; con el Wi-Fi apagado, el sshd tarda ~15 s en notarlo por `ServerAliveInterval=5 × 3`), `infra/hibrido/reenvio-centro.sh` (corre
en la VM, dentro de la sesión SSH) y `infra/reenvio-tcp.js` (el reenviador de Compose, con `HOST_ESCUCHA` opcional). `preparar.sh` ya no aborta si la VM está `RUNNING`: exige el candado del traspaso de ESTA Mac y que `api-sucursal` **no corra**;
`encender-apagar.sh encender-hibrido` ahora enciende la VM 1, y solo si el candado sigue puesto.

**Nunca dos nodos «sucursal 1» (sigue siendo la regla de oro).** Cuatro guardas, la última nueva:
1. `arranque.sh` con candado no arranca la API (sin cambios).
2. `encender-hibrido` se niega sin candado y lo vuelve a leer justo antes de encender la VM 1; `encender` normal se niega con candado.
3. `preparar.sh` verifica candado = el de la marca de esta Mac, **y que `api-sucursal` no corra en la VM** (un «no se pudo saber» aborta: nunca se toma por «no corre»), antes y después de encender y al final.
4. `reenvio-centro.sh` se niega a reenviar si el candado de la VM no es **exactamente** el token del traspaso que abrió el túnel, o si `api-sucursal` corre; y lo vuelve a comprobar cada 3 s: si cualquiera falla (por ejemplo, alguien quita el candado para el regreso), borra
   el reenviador y sale.

**Cada reapertura usa otro extremo en la VM (13100 a 13199).** Si la Mac pierde el Wi-Fi, el sshd de la VM no se entera: la sesión muerta sigue reteniendo su puerto hasta que expire. Con un puerto fijo, el túnel nuevo fallaría con «remote port forwarding failed».
Con otro puerto, la sesión colgada no estorba; el reenviador lleva una etiqueta de sesión y una sesión vieja que muere tarde **no** borra el de la sesión nueva (solo borra el suyo). Medido en la prueba local: el central vuelve a llegar 21 s después de prender el Wi-Fi.

**Sin tocar Terraform ni `arranque.sh`.** El reenviador no es parte del arranque de la VM: lo levanta la sesión SSH, como contenedor `--rm` sin reinicio automático. Así desaparece solo con un reinicio de la VM y **no le ocupa el 3000 a la API** cuando la sucursal 1
vuelva a la VM (`regreso.sh` además lo borra y verifica que no quede). El puerto 3000 de la VM ya estaba abierto en iptables por `arranque.sh` (antes de mirar el candado) y en la VPC por `permitir-interno` (`10.10.0.0/24`, la subred de Cloud Run); **no se agregó
ninguna regla de firewall**. La VM no tiene IP pública y al exterior solo se abre el 22 para IAP, así que el 3000 nunca es público (`preparar.sh` lo comprueba: sin `natIP` y ninguna regla de `red-sima` abierta a `0.0.0.0/0` salvo el 22).

**Si el sshd de COS no permite `-R`.** `tuneles.sh iniciar 1` lo sondea (`sshd -T`) **antes de escribir nada** en la VM: sale con 3, no copia archivos ni deja supervisor; `preparar.sh` lo dice, vuelve a apagar la VM y termina en «LISTO PARA LA DEMO» con la
nota «Traslados con Centro: NO». La demo sigue con Zona 10 ↔ Xela. No es una falla. *No se pudo comprobar contra el sshd real de COS desde aquí*; queda en los pasos de `docs/despliegue-gcp.md`, B3b.

**Los `SOLICITADO` que ya existen con la sucursal 1 se ejecutan solos al abrir el túnel** (la recuperación del central corre cada minuto). Es seguro por cómo está escrito el coordinador: un `SOLICITADO` no tiene transacción global ni nada preparado; `tomarSolicitado` toma
cada uno con `FOR UPDATE SKIP LOCKED` y una condición sobre `SOLICITADO`, así que no se ejecuta dos veces; y el origen vota con su existencia y costo **de hoy** (si no alcanza, vota NO y se aborta limpio). `preparar.sh` los lista **antes** de abrir el túnel y luego espera
a que dejen de estar `SOLICITADO`. Los que queden `COMPROMETIDO` esperan el «Recibir» del destino.

**Lo que esto NO garantiza.** «Nunca en duda» vale para el caso de la demo: con el Wi-Fi apagado **antes** de pedir, `/salud` falla en 3 s y el traslado queda `SOLICITADO` (o Centro, que es el destino y quien solicita, ni siquiera llega al central: 503, no se registra nada). Si el enlace se
corta **entre** los votos y el `COMMIT PREPARED`, Centro queda con una transacción preparada hasta que vuelve el enlace: es inherente al 2PC y pasa igual con la VM. Se resuelve sola (la recuperación reenvía la decisión y el reconciliador de Centro la consulta); solo se bloquean esos
productos. Con el Wi-Fi apagado, en la demo solo se muestra el botón de solicitar traslado deshabilitado en Centro.

**Alternativas descartadas.**
- *HA VPN / Cloud Router:* costo por hora y NAT doméstico (2026-11-02).
- *Tailscale o WireGuard:* dependencia nueva, servicio en COS que no sobrevive al reinicio y una cuenta más.
- *ngrok o un puerto público:* expone la API de la sucursal a internet.
- *Un conector inverso a nivel de protocolo* (la sucursal mantiene una conexión y el coordinador la usa), ya descartado el 2026-11-02: cambiaría el protocolo del 2PC. **Esto no lo es**: es transporte; el coordinador y las sucursales hablan el mismo HTTP firmado.
- *`GatewayPorts` en el sshd de la VM, `socat` o el reenviador en `arranque.sh`:* en COS `/etc` se reescribe en cada arranque y habría que aplicar Terraform y reiniciar la VM, con el riesgo de dos nodos; no hay `socat`, y la imagen `sima` ya trae `node`.
- *Puerto fijo para el extremo del túnel:* falla justo después de un corte de Wi-Fi (ver arriba).

**Efecto sobre las decisiones del 2026-11-02.** «Traslados con la sucursal 1 en la Mac: `SOLICITADO`, nunca en duda» sigue valiendo cuando el túnel está caído; ya no es el estado normal de la demo. «Nunca dos nodos»: la VM ya no está apagada sino **encendida con candado**
(el costo es una e2-small encendida durante la demo). `regreso.sh` y `terminar.sh` cierran el túnel y borran el reenviador.

**Prueba.** Contra GCP real, en los pasos de verificación de `docs/despliegue-gcp.md`, B3b.

**Corrección tras la primera corrida contra GCP real (2026-11-03, `preparar.sh` dio 44/5).** (1) En COS, `/var/lib/sima` está montado `noexec`: `sudo /var/lib/sima/reenvio-centro.sh` daba «Permission denied». Ningún archivo copiado a la VM se ejecuta directo: `reenvio-centro.sh` se invoca con `sudo bash <archivo>` y
`reenvio-tcp.js` lo lee `node` dentro del contenedor (volumen de solo lectura). La «VM» de la prueba local ahora monta ese directorio como `tmpfs noexec`; con la invocación vieja la prueba falla en 13 comprobaciones. (2) «No se pudo medir el desfase entre la Mac y Google»: la medición lee el encabezado `Date` de Google y lo
convierte con `date -j -f "%a, %d %b …"`, que depende del idioma del sistema; con la Mac en español (`lun`, `nov`) fallaba con «illegal time format». Ahora se fuerza `LC_ALL=C`, se toma el punto medio de la petición y hay una segunda fuente. Si aun así no se puede medir, es un **aviso** y el paso 9 decide: sin errores de firma ni de token
en la sucursal 1 se da por bueno; con errores es una falla. (3) Recién encendida, la VM pasa por `PROVISIONING` y `STAGING`: `preparar.sh` espera hasta 120 s a `RUNNING` en vez de abortar.

**Segunda corrección contra GCP real: el chequeo de vida del reenviador (2026-11-03).** `reenvio-centro.sh` comprobaba cada 3 s que el túnel existiera abriendo una conexión a `127.0.0.1:<extremo>` en la VM. Esa conexión viaja por el túnel hasta la API de la Mac y se cierra de golpe: daba falsos negativos («SE CIERRA: el túnel NNNNN ya no existe» a los ~9 s, en bucle) y metía tráfico basura en la API.
Se quitó. Ahora la vida de la sesión se detecta **por la propia sesión SSH**, sin tocar el túnel: (a) `tuneles.sh` le manda por stdin una línea cada 2 s (`latidos()`, vía `< <(latidos)`, que termina sola por SIGPIPE cuando el `ssh` muere) y el script lee con `read -t`; EOF = la sesión se cerró; (b) si pasan más de 12 s sin latido después del primero,
la red se cortó; (c) se vigila además el proceso `sshd` del que cuelga la sesión. *Por qué no basta el EOF/SIGHUP que se propuso:* un corte de red (Wi-Fi apagado) no manda FIN y el sshd de la VM no se entera hasta su propio `ClientAlive`, así que el script nunca vería el EOF; el latido sí lo detecta (el contenedor desaparece ~13 s después del corte).
*Alternativa descartada:* una conexión de prueba por el túnel (lo que falló). **Salvaguarda:** si stdin es `/dev/null` o se cierra antes del primer latido (por ejemplo, un `gcloud` que no reenvíe stdin), el script no se cierra por eso: avisa y solo vigila el sshd padre, para no volver al bucle de reaperturas. La limpieza sigue borrando solo el contenedor de ESTA sesión (`mio`).
El chequeo nuevo no usa el túnel, y una sesión estable no se cierra.
