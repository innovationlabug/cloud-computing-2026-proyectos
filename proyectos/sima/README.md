# SIMA — Sistema de inventario distribuido multisucursal

Grupo 3 · Cloud Computing 2026 · Universidad Galileo

- **Integrantes:** David Alejandro Ortega Reyes, Dilan Jareth Arreaga Tiu
- **Proveedor cloud:** Google Cloud Platform (GCP)
- **Servicios principales:** Compute Engine (3 VMs de sucursal), Cloud Run (`api-central`), Cloud SQL para PostgreSQL, Pub/Sub, Cloud Scheduler, Datastream, BigQuery, Looker Studio, Secret Manager, Cloud Build y Artifact Registry, Cloud Monitoring
- **Repositorio:** Repositorio privado
- **Despliegue:** https://api-central-3ajtjckllq-uc.a.run.app (servicio privado: requiere autenticación)
- **Documentación cloud:** [docs/cloud.md](docs/cloud.md) (no hay PDF)
- **Video:** [Video de demostración](video.mp4)

---

Proyecto de Ingeniería de Software, Ciencias de la Computación VI y Cloud Computing
(Universidad Galileo, FISICC, 2026). Cada sucursal factura contra su propia base
PostgreSQL, sin depender de la red; un nodo central consolida. Las decisiones de
diseño están en `docs/decisiones.md`.

**Autores:** David Ortega y Dilan Arreaga. **Cursos:** Ingeniería de Software
(Ing. Axel Benavides), Ciencias de la Computación VI (Ing. Ronald López) y Cloud Computing (Ing. Adrián Catalán).

**Estado (4 nov 2026): completo.** Ventas con llave de idempotencia, costo promedio ponderado por sucursal en el
kárdex, sincronización por outbox (HTTP en Compose, Pub/Sub en GCP), traslados con commit en dos fases y su
recuperación, existencias de la red con la antigüedad de cada dato, y caja PWA con inicio de sesión y roles.
La implementación principal corre en Google Cloud (rama `main`, ver [Despliegue en Google Cloud](#despliegue-en-google-cloud));
la variante autoalojada vive en la rama `infra/on-premise`.

## Documentación

| Documento | Qué contiene |
|---|---|
| [`docs/cloud.md`](docs/cloud.md) | Documentación de Cloud Computing: arquitectura en GCP, servicios, seguridad, despliegue paso a paso y costos |
| [Video de la demo en Google Cloud](video.mp4) | Demostración grabada del sistema desplegado en GCP |
| [`docs/decisiones.md`](docs/decisiones.md) | Decisiones de diseño y de arquitectura, con fecha, alternativas y por qué |
| [`docs/despliegue-gcp.md`](docs/despliegue-gcp.md) | Procedimiento de despliegue en GCP y de la demo híbrida, con sus comprobaciones |
| [`docs/resultados-concurrencia.md`](docs/resultados-concurrencia.md) | Prueba de concurrencia: pesimista, optimista y sin control |
| [`docs/resultados-2pc.md`](docs/resultados-2pc.md) | Escenarios de falla del commit en dos fases |
| [`docs/resultados-carga.md`](docs/resultados-carga.md) | Pruebas de carga (k6 y pgbench), en local y en GCP |
| [`docs/resultados-indices.md`](docs/resultados-indices.md) | Planes de ejecución antes y después de indexar |
| [`docs/tableros.md`](docs/tableros.md) | Tableros de Looker Studio sobre BigQuery |
| `infra/terraform/README.md` | La infraestructura como código y cómo se adoptó |
| `docs/SIMA_Propuesta_Final.pdf`, `docs/SIMA_Modelo_ER_1.pdf` | Propuesta y modelo entidad-relación |

## Requisitos

- Docker con Docker Compose v2
- Node.js 20 o superior, para los scripts de prueba y de demostración (las APIs corren en contenedores)
- `bash`, `curl` y `jq`
- Para Google Cloud: `gcloud` CLI y Terraform 1.16.5 (`infra/terraform/instalar-terraform.sh` lo instala en Cloud Shell)

No hace falta `psql` en la máquina: los scripts lo ejecutan dentro de los contenedores.

## Levantar desde cero

```bash
npm install                     # dependencias locales (la prueba usa pg)
docker compose up -d --build    # 4 PostgreSQL 16 + 4 APIs
./scripts/migrar.sh             # migraciones SQL numeradas en los 4 nodos (idempotente)
./scripts/sembrar.sh            # 3 sucursales, 50 productos, 9 usuarios, existencias (idempotente)
curl localhost:3001/salud
```

Para volver a cero: `docker compose down -v` (borra los volúmenes) y repetir.

Si otro PostgreSQL de la máquina ya ocupa el 5432, levantar con otro puerto para
`db-central` y exportarlo también para los scripts:
`export PUERTO_DB_CENTRAL=5442 && docker compose up -d`.

`verificar.mjs` verifica en reposo, y la prueba de concurrencia (con `SIN_CONTROL`) y
`prueba-participante` dejan la base sin cuadrar a propósito: después de correrlas, volver a
cero.

Si después de cambiar código una API sigue respondiendo como antes, falta
`docker compose up -d --build`: las tres sucursales comparten la imagen `sima/api-sucursal`.

| Nodo | API | Base de datos |
|---|---|---|
| central | `localhost:3000` | `localhost:5432` · `sima_central` |
| sucursal 1 | `localhost:3001` | `localhost:5433` · `sima_sucursal_1` |
| sucursal 2 | `localhost:3002` | `localhost:5434` · `sima_sucursal_2` |
| sucursal 3 | `localhost:3003` | `localhost:5435` · `sima_sucursal_3` |

Usuario y contraseña de las bases: `sima` / `[REDACTADO]`.

Redes: cada sucursal tiene la suya (`sima-red-sucursal-N`, con su API y su base); las
cuatro APIs comparten `sima-red-central`, que es **el enlace**; y solo `api-central` llega
a `db-central` (`sima-red-datos-central`). Ninguna sucursal alcanza la base del central ni
la de otra sucursal.

Los puertos `3001`–`3003` los publica `acceso-sucursal-N`, un reenviador TCP que vive solo
en la red de la sucursal, no la API. Así la sucursal sigue respondiendo desde el anfitrión
aunque se le corte el enlace (en Docker Desktop, el puerto de un contenedor con varias
redes puede dejar de responder al desconectar una; ver `docs/decisiones.md`).

## Datos de prueba

- Códigos de barras: `7501` + id del producto a 9 dígitos (producto 1 → `7501000000001`).
- Productos 46–50 con **1 unidad** en cada sucursal; productos 41–45 con **20**.
- Usuarios por sucursal (`id_usuario`): sucursal 1 → 1 cajero, 2 encargado, 3 admin;
  sucursal 2 → 4, 5, 6; sucursal 3 → 7, 8, 9.
- **Inicio de sesión (datos de demostración).** Cada usuario entra con su `nombre_usuario` y la contraseña
  `[REDACTADO]`. Son datos de la semilla, no secretos de ningún ambiente real.

  | Sucursal | Cajero | Encargado | Admin |
  |---|---|---|---|
  | 1 · Centro | `cajero1` | `encargado1` | `admin1` |
  | 2 · Zona 10 | `cajero2` | `encargado2` | `admin2` |
  | 3 · Xela | `cajero3` | `encargado3` | `admin3` |

  Por ejemplo, `cajero1` / `[REDACTADO]`. Un usuario solo entra en **su** sucursal. Los hashes (scrypt) están en
  `scripts/semilla/02_usuarios.sql`, que genera `node scripts/semilla/generar-hashes.mjs`.

## Endpoints

### API de sucursal (`:3001`–`:3003`)

**Denegar por defecto.** Cada ruta declara su acceso en el código y una ruta sin declaración responde `403` (la API ni
arranca si falta alguna): `@Publica()` (`POST /sesion`, `DELETE /sesion`, `GET /salud`), `@RutaDeServicio()` (las rutas del
coordinador de traslados: `preparar`, `confirmar`, `abortar` y `reconciliacion/ejecutar`; sin sesión, pero con la **firma de
servicio** del central: ver «Autenticación entre nodos», `401 FIRMA_DE_SERVICIO_INVALIDA` sin ella) o `@Roles(...)`: sesión (cookie `sima_sesion_s<N>`, httpOnly) y un rol permitido. Sin sesión válida:
`401 NO_AUTENTICADO`; con sesión pero sin el rol: `403 PERMISO_INSUFICIENTE`. El rol sale de la base local en cada petición.

| Rol | Puede |
|---|---|
| `CAJERO` | `POST /ventas`, `GET /existencias`, `GET /sucursales`, `GET /red/enlace`, `GET /red/existencias/:cb`, `GET /sesion` |
| `ENCARGADO` | Lo del cajero, más `POST /ajustes`, **registrar la recepción de mercadería de un proveedor** (`POST /recepciones`), solicitar un traslado, ver los traslados de su sucursal, **ver lo por reponer** (`GET /existencias/por-reponer`), recibir (solo si su sucursal es el destino) y **ver las ventas de su sucursal** (solo lectura) |
| `ADMIN` | Todo lo anterior, más ver las ventas de cualquier sucursal desde el central (`/red/ventas`), administrar los empleados y renombrar sucursales (`/red/empleados`, `/red/sucursales/:id`) y **administrar el catálogo de productos** (`/red/productos`, `/productos/categorias`) |

El usuario de una venta, un ajuste, una recepción o una solicitud de traslado **sale de la sesión**: un cuerpo con
`id_usuario`, `id_usuario_origen`, `id_usuario_destino`, `id_usuario_solicitante` o `id_usuario_actor` se rechaza con `400 VALIDACION`.

| Método | Ruta | Qué hace |
|---|---|---|
| `POST` | `/sesion` | **Inicio de sesión**, validado contra la base de la sucursal (funciona sin enlace). `{ nombre_usuario, contrasena }` → 200 con `{ id_usuario, nombre, rol, sucursal }` y la cookie. `401 CREDENCIALES_INCORRECTAS` (igual para usuario inexistente, inactivo o contraseña mala); `423 CUENTA_BLOQUEADA` si la contraseña era correcta pero la cuenta está bloqueada |
| `GET` | `/sesion` | Quién tiene la sesión: `{ id_usuario, nombre, rol, sucursal }`. El rol sale de la base, no del token |
| `DELETE` | `/sesion` | Cierra la sesión (borra la cookie). 204 |
| `POST` | `/ventas` | Registra una venta en **una sola transacción local**: valida y descuenta existencia, inserta `documento`, `linea_documento`, `movimiento` y la fila de `outbox`. No toca el central. `{ id_idempotencia?, metodo_pago, referencia?, lineas }`: **`id_idempotencia`** (UUID) lo genera la caja una vez por intento de cobro; repetir la misma llave con el mismo contenido devuelve la venta ya registrada (201, `repetida: true`) sin descontar de nuevo, y con otro contenido da 409 `IDEMPOTENCIA_CONFLICTO`. Si falta, el servidor genera una y esa llamada **no queda protegida** contra el reintento (scripts, k6 y pgbench siguen igual). Funciona sin enlace |
| `POST` | `/ajustes` | Ajuste por conteo físico: deja cada producto en la cantidad contada, en **una sola transacción** (documento `AJUSTE`, movimiento `AJUSTE_ENTRADA`/`AJUSTE_SALIDA` y `outbox`). Solo ENCARGADO o ADMIN (el usuario es el de la sesión) |
| `POST` | `/recepciones` | ENCARGADO o ADMIN. **Recepción de mercadería de un proveedor** (operación local, sin enlace, como la venta): `{ id_idempotencia?, id_proveedor, referencia, lineas: [{ codigo_barras, cantidad }] }` (`id_idempotencia` funciona como en `POST /ventas`). **Sin costo en la solicitud:** el servidor toma el costo de la vigencia de la lista del distribuidor que ya está replicada en la sucursal (funciona sin enlace); si el cliente manda un `costo_unitario`, se ignora. En **una sola transacción** sube la existencia **y su valor** (crea la fila si la sucursal no la tenía), inserta el documento `RECEPCION` con `id_proveedor` y `referencia`, una línea por producto con `precio_unitario = costo de la vigencia` y la vigencia usada (`id_costo_proveedor`), un movimiento `RECEPCION_ENTRADA` valorizado por línea y la fila de outbox. Proveedor o producto inactivo, cantidad ≤ 0 o producto repetido: 400 y no se escribe nada; producto sin vigencia: 422 `SIN_COSTO_VIGENTE`; existencia sin saldo inicial: 409 `SIN_SALDO_INICIAL`. Llave de idempotencia: ver `docs/decisiones.md`, 2026-10-27 |
| `GET` | `/recepciones/catalogo` | ENCARGADO o ADMIN. Lo que la pantalla de recepción puede elegir, de la copia local (sin enlace): proveedores activos y productos activos, **incluidos los que la sucursal todavía no tiene en existencia** (cantidad 0), y `costos`: la vigencia de costo de hoy de cada proveedor y producto (solo lectura) |
| `POST` | `/traslados/:id_traslado/preparar` | **Participante de un traslado (2PC), fase 1.** Ejecuta su parte y la deja preparada (`PREPARE TRANSACTION`). Responde `{ voto: 'SI' \| 'NO', motivo }` |
| `POST` | `/traslados/:id_traslado/confirmar` | Fase 2: `COMMIT PREPARED`. Idempotente (`CONFIRMADA` / `YA_CONFIRMADA`) |
| `POST` | `/traslados/:id_traslado/abortar` | Fase 2: `ROLLBACK PREPARED`. Idempotente (`ABORTADA` / `NO_PREPARADA`) |
| `POST` | `/traslados/reconciliacion/ejecutar` | Una pasada del reconciliador de transacciones preparadas (además del intervalo) |
| `POST` | `/traslados/:id_traslado/recibir` | Recepción en el **destino** (operación local): documento `RECEPCION`, `TRASLADO_ENTRADA`, líneas a `RECIBIDO` y outbox. Lo recibe el usuario de la sesión (ENCARGADO o ADMIN) y solo si `traslado_local` dice que esta sucursal es el destino (si no, 409 `NO_ES_DESTINO`) |
| `GET` | `/ventas?desde=&hasta=&antes_de=` | ENCARGADO o ADMIN. Las ventas de **esta** sucursal, de la base local (sin enlace funciona), de la más reciente a la más antigua, **50 por página**. `desde` y `hasta` son días de Guatemala (`YYYY-MM-DD`, incluidos; por defecto, los últimos 31 días); **más de 31 días da 400**. Cada venta trae `id_documento`, `fecha_hora`, `total`, `metodo_pago`, `estado`, `usuario` (nombre) y `lineas`. La respuesta trae `siguiente`: la llave que se manda como `antes_de` para la página que sigue (paginación por llave, sin OFFSET) |
| `GET` | `/ventas/resumen?dia=YYYY-MM-DD` | ENCARGADO o ADMIN. Cantidad de ventas, total y total por método de pago de ese día de Guatemala (por defecto, hoy). No cuenta las anuladas |
| `GET` | `/ventas/:id_documento` | ENCARGADO o ADMIN. La venta con sus líneas (producto, cantidad, precio, subtotal). 404 `VENTA_NO_EXISTE` si no es una venta de esta sucursal |
| `GET` | `/existencias` | El catálogo local con la existencia de la sucursal: **todos** los productos de la copia local, con `cantidad` 0 y `version` 0 donde la sucursal todavía no tiene fila de existencia (un producto recién creado), y con `activo` (los dados de baja siguen apareciendo, marcados) |
| `GET` | `/existencias/por-reponer` | Solo ENCARGADO y ADMIN. Productos activos con existencia local menor que `stock_min` (0 si no hay fila), del mayor faltante al menor: `codigo_barras`, `nombre`, `cantidad`, `stock_min`, `faltante`. Local: funciona sin enlace |
| `GET` | `/existencias/:codigo_barras` | Existencia de un producto (con `activo`; 0 si la sucursal no tiene fila). 404 `PRODUCTO_NO_EXISTE` solo si el código no está en el catálogo |
| `GET` | `/salud` | **Pública.** Nombre de la sucursal (para la pantalla de login), estado de la base local, estrategia activa, `outbox` (pendientes, enviados, fallidos, la pendiente más antigua) y estado del publicador (ver abajo) |
| `GET` | `/caja/` | **La caja** (PWA estática, `packages/caja`): Caja, Existencias en la red, Por reponer, Recepción, Traslados, Ventas, Empleados y Productos |
| `GET` | `/usuarios` | Usuarios de la sucursal, bajas incluidas (sin correo ni contraseña), de la copia local: funciona sin enlace. ENCARGADO o ADMIN |
| `GET` | `/sucursales` | Copia local de las sucursales, con `es_esta` |
| `GET` | `/traslados/por-recibir` | ENCARGADO o ADMIN. Traslados en los que esta sucursal es destino (`traslado_local`) y que falta recibir. Local: funciona sin enlace |
| `GET` | `/red/enlace` | ¿Responde el central? Siempre 200, con `en_linea`, `central` y `latencia_ms` |
| `GET` | `/red/existencias/:codigo_barras` | Reenvía a `GET /existencias/:codigo_barras` del central |
| `GET` | `/red/traslados` | ENCARGADO o ADMIN. Reenvía a `GET /traslados?id_sucursal=` del central, con esta sucursal |
| `GET` | `/red/traslados/:id_traslado` | ENCARGADO o ADMIN. Reenvía a `GET /traslados/:id_traslado` del central; si esta sucursal no es su origen ni su destino, 404 |
| `GET` | `/red/ventas?id_sucursal=&desde=&hasta=&antes_de=` | Solo ADMIN. Las ventas de cualquier sucursal (por defecto, esta) desde el **consolidado del central**: consistencia eventual, la respuesta trae `vigente_al` y `antiguedad_segundos` (hasta cuándo está al día esa sucursal). Mismos días, tope y paginación que `GET /ventas`. Sin enlace, `503 SIN_ENLACE_CENTRAL` |
| `GET` | `/red/ventas/:id_documento` | Solo ADMIN. Detalle de una venta del consolidado (el id es el del central; la respuesta trae también `id_documento_origen`, el de su sucursal) |
| `GET` | `/red/empleados[?id_sucursal=N]` | Solo ADMIN. Reenvía a `GET /empleados` del central: los empleados de toda la red (o de una sucursal), bajas incluidas |
| `POST` | `/red/empleados` | Solo ADMIN. `{ id_sucursal, nombre, nombre_usuario, rol, contrasena }`: la sucursal valida la política (mínimo 8 caracteres), genera el hash scrypt y reenvía **solo el hash** al central. 201 con el empleado |
| `PATCH` | `/red/empleados/:id` | Solo ADMIN. `{ nombre?, rol?, activo? }` (dar de baja es `activo: false`; nunca se borra) |
| `PUT` | `/red/empleados/:id/contrasena` | Solo ADMIN. `{ contrasena }`; igual que el alta, solo viaja el hash |
| `PATCH` | `/red/sucursales/:id` | Solo ADMIN. `{ nombre }`: renombra la sucursal en el maestro del central |
| `POST` | `/red/productos` | Solo ADMIN. `{ codigo_barras, nombre, categoria, precio_venta, stock_min }`: reenvía al central. `codigo_barras` de 8 a 14 dígitos y único (409 `CODIGO_EN_USO`); `precio_venta` > 0 con hasta 2 decimales; `stock_min` entero ≥ 0; `categoria` una existente o una nueva. 201 con el producto. **No crea existencia**: aparece con 0 en todas las sucursales y se le da entrada con una recepción |
| `PATCH` | `/red/productos/:id` | Solo ADMIN. `{ nombre?, categoria?, precio_venta?, stock_min?, activo? }` (dar de baja es `activo: false`; nunca se borra). `codigo_barras` no se puede cambiar: un cuerpo que lo traiga da 400. Cambiar el precio no altera las ventas anteriores |
| `GET` | `/kardex/:codigo_barras?desde&hasta` | ENCARGADO o ADMIN. **Kárdex valorizado** de un producto en esta sucursal, de la base local (sin enlace): fecha, documento, tipo, entrada, salida, costo unitario, saldo en cantidad, costo promedio y saldo en valor; días de Guatemala, hasta 366. Lo anterior al saldo inicial sale sin costo |
| `GET` | `/inventario/valor` | ENCARGADO o ADMIN. Valor del inventario de la sucursal, total y por categoría, con y sin IVA (monto ÷ 1.12) |
| `GET` | `/auditoria/kardex` | ENCARGADO o ADMIN. **Auditoría del kárdex:** `{ cuadra, discrepancias: [...] }`; lista vacía = cuadra en cantidad y en valor (cadena de saldos sin huecos, existencia = último saldo, cero vale cero, sin saldo inicial pendiente, Σ costo de las líneas de venta = Σ costo de los movimientos VENTA) |
| `GET` | `/costos` | ENCARGADO o ADMIN. La lista de costos vigente en la copia local (la última que llegó por réplica) |
| `POST` | `/red/costos` | Solo ADMIN. `{ id_proveedor, codigo_barras, costo_unitario }`: una **vigencia nueva** en la lista de costos del distribuidor (maestra del central; nunca se actualiza una anterior). 422 `MARGEN_NO_POSITIVO` si `costo ≥ precio_venta` (también al bajar el precio de venta de un producto) |
| `GET` | `/productos/categorias` | Solo ADMIN. Las categorías distintas de la copia local de `producto`, en orden alfabético. Local: funciona sin enlace |
| `POST` | `/red/traslados` | ENCARGADO o ADMIN. **El destino solicita**: reenvía a `POST /traslados` del central con esta sucursal como destino y el usuario de la sesión como solicitante. `{ id_sucursal_origen, lineas, id_idempotencia? }` (la llave de idempotencia es un UUID por solicitud del usuario: si la caja reenvía el mismo formulario tras un `504`, el central devuelve el mismo traslado; si falta, la sucursal genera una); un cuerpo con `id_sucursal_destino` o con un usuario da 400 |

Las rutas de `/red/empleados`, `/red/sucursales` y `/red/productos` toman el actor de la sesión (un cuerpo con `id_usuario_actor` o `id_usuario` da 400) y escriben en el central, no en la sucursal: el cambio llega a las sucursales con la siguiente ronda de réplica (unos segundos). Sin enlace, `503 SIN_ENLACE_CENTRAL`.

Los `/red/*` llevan el token de identidad cuando hace falta (`CENTRAL_AUTENTICACION`). Si no se llega al central
responden `503 SIN_ENLACE_CENTRAL`; si el central responde, su estado y su cuerpo pasan tal cual. Ninguno interviene en
una venta.

```bash
# Se inicia sesión una vez y la cookie se guarda en un archivo (-c) para usarla después (-b).
curl -c /tmp/caja1.txt -X POST localhost:3001/sesion -H 'Content-Type: application/json' \
  -d '{"nombre_usuario": "cajero1", "contrasena": "[REDACTADO]"}'

curl -b /tmp/caja1.txt -X POST localhost:3001/ventas -H 'Content-Type: application/json' -d '{
  "metodo_pago": "EFECTIVO",
  "lineas": [{ "codigo_barras": "7501000000001", "cantidad": 2 }]
}'
```

```bash
# Un ajuste lo firma un ENCARGADO o ADMIN: otra sesión (encargado1).
curl -c /tmp/encargado1.txt -X POST localhost:3001/sesion -H 'Content-Type: application/json' \
  -d '{"nombre_usuario": "encargado1", "contrasena": "[REDACTADO]"}'

curl -b /tmp/encargado1.txt -X POST localhost:3001/ajustes -H 'Content-Type: application/json' -d '{
  "referencia": "conteo semanal",
  "lineas": [{ "codigo_barras": "7501000000046", "cantidad_contada": 1, "version_esperada": 0 }]
}'
```

`version_esperada` es la versión de la existencia que se vio al empezar el conteo (la da
`GET /existencias/:codigo_barras`). Si hubo movimientos durante el conteo, el ajuste
completo se rechaza con `409 CONFLICTO_VERSION` y no escribe nada: hay que volver a contar.

Pruebas del ajuste (11 casos, incluida una venta entre la lectura y el ajuste):
`node scripts/prueba-ajuste/prueba.mjs`

Respuestas de error con `codigo` estable: `EXISTENCIA_INSUFICIENTE` (409),
`CONFLICTO_VERSION` (409: optimista con reintentos agotados, o ajuste sobre un conteo desactualizado), `RESTRICCION_BD` (409, lo
rechazó una restricción del motor), `PRODUCTO_NO_EXISTE` (404), `PERMISO_INSUFICIENTE`
(403), `VALIDACION` (400), `PRODUCTO_BLOQUEADO` (409: otra transacción retiene la fila más que
`LOCK_TIMEOUT_MS`; "producto bloqueado temporalmente"), `NO_AUTENTICADO` (401: sin sesión, sesión vencida, token
alterado o de otra sucursal, o usuario dado de baja), `CREDENCIALES_INCORRECTAS` (401) y `CUENTA_BLOQUEADA` (423).

**Contraseñas y bloqueo.** scrypt asíncrono de `node:crypto`; un usuario inexistente, inactivo o con la contraseña mala recibe
exactamente la misma respuesta y gasta el mismo cómputo. Cinco contraseñas malas bloquean la cuenta `SESION_BLOQUEO_MIN`
minutos **en esa sucursal** (la tabla `intento_acceso` es local y no se replica). Pruebas: `scripts/prueba-sesion/ejecutar.sh`.
Una baja hecha en el central llega a la sucursal con la siguiente réplica; una sucursal sin enlace no se entera hasta
entonces (`docs/decisiones.md`).

### API central (`:3000`)

**Toda ruta exige la firma de una sucursal** (`X-SIMA-Firma`, ver «Autenticación entre nodos»), salvo `GET /salud` y
`POST /traslados/recuperacion/ejecutar` (la llama Cloud Scheduler, con OIDC en GCP). Sin firma o con una mala: `401`. La sucursal
que firmó manda sobre el `id_sucursal` del cuerpo o de la query: pedir la réplica, los traslados o mandar un mensaje de otra
sucursal es `403`.

| Método | Ruta | Qué hace |
|---|---|---|
| `POST` | `/sincronizacion/mensajes` | Consumidor: aplica un mensaje de `outbox` de una sucursal en **una transacción** (inbox de idempotencia, documento, líneas, movimientos y existencia consolidada con guarda de versión). `200` = acuse de recibo, también para duplicados. `4xx` solo si el mensaje es inválido; `503` si el problema es del central. Acepta el mensaje directo (adaptador HTTP) o dentro del sobre de una suscripción push de Pub/Sub |
| `GET` | `/existencias/:codigo_barras` | Existencia del producto en cada sucursal, desde el consolidado, con `vigente_al` y `antiguedad_segundos` |
| `POST` | `/traslados` | **Coordinador del 2PC.** `{ id_sucursal_origen, id_sucursal_destino, id_usuario_solicitante, id_idempotencia, lineas }`: el solicitante es ENCARGADO o ADMIN del destino, y el destino es la sucursal que firma (si no, 403). **Idempotente** por `(id_sucursal_destino, id_idempotencia)` (UUID obligatorio): repetir la solicitud devuelve el mismo traslado (`repetida: true`) sin ejecutar el protocolo otra vez; la misma llave con otro contenido da 409 `IDEMPOTENCIA_REUTILIZADA`. Registra el traslado y ejecuta el protocolo. 201 con la decisión (`COMPROMETIDO` o `ABORTADO`); 202 si quedó `SOLICITADO` porque alguna sucursal está fuera de línea |
| `GET` | `/replica/cambios?id_sucursal=N&desde=V&limite=L` | Lo que cambió en catálogo y usuarios desde la versión V, para la réplica de la sucursal N. Solo a la sucursal N (la que firma); otra, 403 |
| `GET` | `/traslados?id_sucursal=N` | Los 50 traslados más recientes en los que N es origen o destino, con sus líneas (para la caja). Solo los de la sucursal que firma |
| `GET` | `/traslados/:id_traslado` | Traslado, bitácora del coordinador (`transaccion_global`), participantes y copias recibidas. 404 si la sucursal que firma no es su origen ni su destino |
| `GET` | `/ventas?id_sucursal=N&desde=&hasta=&antes_de=` | Ventas consolidadas de una sucursal (`id_sucursal` obligatorio), de las copias de `documento` y `linea_documento`, con el nombre del usuario del maestro. Solo lectura, nunca consulta a la sucursal. Trae `vigente_al` (la fecha del último documento de esa sucursal ya aplicado, el mismo criterio que la consulta de existencias de la red). Días de Guatemala, 31 días como máximo, 50 por página |
| `GET` | `/ventas/:id_documento` | Detalle de una venta consolidada, con `vigente_al` |
| `GET` | `/empleados[?id_sucursal=N]` | Empleados del maestro (toda la red o una sucursal), bajas incluidas, sin correo ni contraseña |
| `POST` | `/empleados` | `{ id_usuario_actor, id_sucursal, nombre, nombre_usuario, rol, hash_contrasena }`. El actor tiene que ser un ADMIN activo **de la sucursal que firma**. `hash_contrasena` con el formato `scrypt$N$r$p$sal$hash`; un cuerpo con `contrasena` da 400 (el texto plano nunca llega al central). 409 `NOMBRE_USUARIO_EN_USO` si el nombre ya existe (sin distinguir mayúsculas) |
| `PATCH` | `/empleados/:id` | `{ id_usuario_actor, nombre?, rol?, activo? }`. `id_sucursal` no se puede cambiar. 409 `AUTO_BAJA` si un ADMIN se da de baja o se quita el rol a sí mismo |
| `PUT` | `/empleados/:id/contrasena` | `{ id_usuario_actor, hash_contrasena }` |
| `PATCH` | `/sucursales/:id` | `{ id_usuario_actor, nombre }`: renombra la sucursal. 409 `NOMBRE_SUCURSAL_EN_USO` |
| `POST` | `/productos` | `{ id_usuario_actor, codigo_barras, nombre, categoria, precio_venta, stock_min }`. El actor tiene que ser un ADMIN activo **de la sucursal que firma**. 201 con el producto; 409 `CODIGO_EN_USO`; 400 si algún dato es inválido |
| `PATCH` | `/productos/:id` | `{ id_usuario_actor, nombre?, categoria?, precio_venta?, stock_min?, activo? }`. `codigo_barras` no se puede cambiar (400 por la API; el trigger de la migración `central/018` lo rechaza también por SQL). 404 `PRODUCTO_NO_EXISTE` |
| `GET` | `/traslados/decision/:gid` | Para el reconciliador de una sucursal: `COMPROMETIDO`, `ABORTADO` o `PENDIENTE`; 404 `GID_DESCONOCIDO` si no existe ese `gid` o la sucursal que firma no es su participante |
| `POST` | `/traslados/recuperacion/ejecutar` | Una pasada de recuperación del coordinador (también corre al arrancar y cada 5 s; en GCP la llama Cloud Scheduler cada minuto, con OIDC; **sin firma de sucursal**) |
| `GET` | `/salud` | Estado de la base central y sucursales activas (pública) |

## Caja (PWA)

Cada sucursal sirve su caja en `/caja/`, desde la misma API:

- Sucursal 1 (Centro): http://localhost:3001/caja/
- Sucursal 2 (Zona 10): http://localhost:3002/caja/
- Sucursal 3 (Xela): http://localhost:3003/caja/

Al abrir, se inicia sesión con usuario y contraseña (se valida en la propia sucursal, así que funciona sin internet). El riel
muestra quién tiene la sesión y «Cerrar sesión». Arriba se ve siempre el enlace con el central y lo que falta
enviar. Sin enlace, la caja vende igual; la consulta de red, la solicitud de traslados y la administración de empleados y de productos se deshabilitan con el motivo.

La pantalla **Ventas** (solo lectura) la ven el encargado y el administrador: resumen del día, lista con «Cargar más» y el detalle de cada venta, por día de Guatemala. El encargado ve su sucursal desde la base local, también sin enlace. El administrador puede elegir otra sucursal; esas ventas vienen del consolidado del central y la pantalla dice cuánto hace que está al día (sin enlace, solo su sucursal). El cajero no la ve.

La pantalla **Empleados** la ven el encargado (los de su sucursal, en lectura y sin red) y el administrador (toda la red: crear, editar nombre y puesto, cambiar contraseña, dar de baja y reactivar, y renombrar sucursales). Los cambios se hacen en el central y llegan a las sucursales por la réplica en unos segundos.

La pantalla **Kárdex y valor** (encargado y administrador) muestra el valor del inventario por categoría y el kárdex valorizado de un producto por rango de fechas, imprimible como el reporte del día; **Ventas** muestra además costo de ventas, utilidad bruta, margen y merma del día, con la base sin IVA. El **cajero nunca ve costos ni ganancia**.

La pantalla **Productos** es solo del administrador: lista con buscador y filtro por categoría, crear, editar nombre, categoría, precio y mínimo, y dar de baja o reactivar. El código de barras se ve pero no se edita. Al crear un producto, «Registrar recepción» abre Recepción con ese producto precargado (se habilita cuando el producto llega a la sucursal por la réplica). Sin enlace las acciones se deshabilitan con el motivo y la lista local sigue visible.

```bash
npm run build -w @sima/caja      # compila la caja a packages/caja/out (también lo hace el Dockerfile)
npm run dev -w @sima/caja        # desarrollo de las pantallas (necesita la API de una sucursal para los datos)
```

En GCP, la caja de una VM se abre por un túnel IAP, para que quede en `localhost` y el service worker funcione:
`gcloud compute start-iap-tunnel sucursal-1 3000 --local-host-port=localhost:3001 --zone=us-central1-a`, y después
http://localhost:3001/caja/. Las limitaciones están en `docs/decisiones.md` (2026-10-22): el
service worker sobre HTTP. El doble cobro quedó cerrado el 2026-10-27 con una llave de idempotencia por cobro.

## Sincronización con el central

Cada sucursal tiene un **publicador** que envía su `outbox` al central cada 2 s, en orden
de `id_outbox`, por el transporte configurado (en Compose, HTTP directo a
`POST /sincronizacion/mensajes`). Marca `ENVIADO` solo con acuse de recibo. Sin enlace no
pasa nada visible para la caja: la fila queda `PENDIENTE` y se reintenta.

`GET /salud` de la sucursal muestra el estado:

| Campo | Qué dice |
|---|---|
| `outbox.pendientes` / `enviados` / `fallidos` | Conteo por estado |
| `outbox.pendiente_mas_antigua` | La cabeza de la cola: si su `antiguedad_segundos` crece, la cola está detenida |
| `sincronizacion.enlace` | `OK`, `SIN_ENLACE` (no se llega al central), `ATASCADA` (se llega, pero la cabeza no pasa: 503 o un rechazo por debajo del máximo) o `DESCONOCIDO` (sin envíos desde que arrancó). Refleja el último envío intentado |
| `sincronizacion.ultimo_error` | Tipo, código HTTP, código del central, fila y hora del último envío fallido |
| `sincronizacion.totales` | Aplicados, duplicados y pasados a `FALLIDO` desde que arrancó el proceso |

Solo un 4xx permanente del central (salvo 408 y 429) suma `intentos`. Al llegar a
`SINCRONIZACION_MAX_RECHAZOS`, la fila pasa a `FALLIDO` y la cola sigue con las siguientes.
Un 5xx, un timeout o la falta de enlace nunca mandan una fila a `FALLIDO`.

### Por Pub/Sub (`TRANSPORTE_SINCRONIZACION=PUBSUB`)

El otro adaptador publica cada fila de `outbox` como un mensaje en el tema
`sima-sincronizacion`. La suscripción push lo entrega al mismo `POST /sincronizacion/mensajes`
del central. Diferencias con HTTP (detalle en `docs/decisiones.md`, 2026-10-19):

- `ENVIADO` quiere decir que **Pub/Sub lo aceptó**, no que el central lo aplicó. Con la base
  del central caída, el outbox de la sucursal se vacía igual.
- Lo que el central no puede aplicar, sea inválido o por estar caído, se reintenta y, después de
  15 entregas (alrededor de 1,5 h), pasa al tema `sima-sincronizacion-muertos`. La sucursal no se
  entera: hay que revisar los mensajes muertos.
- En `/salud`, `ultimo_error.codigo` es el código gRPC (`NOT_FOUND`, `PERMISSION_DENIED`, …) y
  `mensaje` empieza con `gRPC <número>`. Tema inexistente, permisos y credenciales dejan la cola
  `ATASCADA` sin sumar intentos; red y timeouts, `SIN_ENLACE`.

En local, con el emulador de Pub/Sub:

```bash
docker compose -f docker-compose.yml -f docker-compose.pubsub.yml up -d   # emulador + sucursales en PUBSUB
scripts/prueba-pubsub/preparar.sh                                         # temas y suscripciones (15 entregas, 10–600 s)
docker compose -f docker-compose.yml -f docker-compose.pubsub.yml rm -sf pubsub-emulador
docker compose -f docker-compose.yml up -d                                # de vuelta a HTTP
```

Mensajes muertos (contra el emulador con `PUBSUB_EMULATOR_HOST=localhost:8085`; sin esa variable,
contra GCP con el token de `gcloud`):

```bash
node scripts/pubsub/muertos.mjs listar               # sin sacarlos
node scripts/pubsub/muertos.mjs reenviar [id…]       # los vuelve a publicar; seguro: el central es idempotente
node scripts/pubsub/muertos.mjs descartar id…        # los que nunca van a ser válidos (o --todos)
```

### Devolver filas `FALLIDO` a la cola

Una fila `FALLIDO` es un documento que el central rechazó y que **no está consolidado**.
Primero hay que ver por qué (`sincronizacion.ultimo_error` en `/salud`, o el registro de la
API) y corregir la causa. Después:

```bash
# Ver cuáles son (sucursal 1)
docker compose exec db-sucursal-1 psql -U sima -d sima_sucursal_1 \
  -c "SELECT id_outbox, tipo, intentos, creado_en, payload->>'id_documento' AS documento FROM outbox WHERE estado = 'FALLIDO'"

# Devolverlas a la cola (el publicador las reenvía en el siguiente ciclo)
docker compose exec db-sucursal-1 psql -U sima -d sima_sucursal_1 \
  -c "UPDATE outbox SET estado = 'PENDIENTE', intentos = 0 WHERE estado = 'FALLIDO'"
```

Es seguro: se envían antes que las filas nuevas, pero fuera de orden respecto de las que
ya pasaron, y la guarda de versión del central impide que un dato viejo pise la existencia
consolidada. Si alguna ya se hubiera aplicado, el central la descarta como duplicada.

## Traslados entre sucursales (2PC)

```bash
curl -X POST localhost:3000/traslados -H 'Content-Type: application/json' -d '{
  "id_sucursal_origen": 1, "id_sucursal_destino": 2,
  "id_usuario_solicitante": 5,
  "lineas": [{ "codigo_barras": "7501000000021", "cantidad": 4 }]
}'
# El traslado lo pide el DESTINO: el solicitante es un ENCARGADO o ADMIN de la sucursal 2 (encargado2, id 5).
# Desde la caja se hace con POST localhost:3002/red/traslados { id_sucursal_origen, lineas } y la sesión.
# → traslado COMPROMETIDO: el origen descontó (su voto es automático, según la existencia, y su documento
#   queda sin usuario); el destino tiene las líneas EN_TRANSITO
# La recepción la firma el usuario de la sesión (encargado2, en la sucursal 2):
curl -b /tmp/encargado2.txt -X POST localhost:3002/traslados/<id_traslado>/recibir
curl localhost:3000/traslados/<id_traslado>
```

Si alguna sucursal está fuera de línea, el traslado queda `SOLICITADO` y la recuperación lo
retoma cuando las dos vuelven. Cada sucursal tiene un **reconciliador** que revisa sus
transacciones preparadas de traslados más viejas que 30 s y le pregunta al central qué
decidió. Solo deshace con `ABORTADO` o con un 404 `GID_DESCONOCIDO`; sin respuesta,
espera. `/salud` muestra `transacciones_en_duda`. La bitácora del coordinador no se purga,
y el ejercicio de PITR del central se hace sin traslados en curso (ver
`docs/decisiones.md`).

El coordinador escribe la bitácora antes de cada paso. La decisión se escribe con un
`UPDATE` condicional (solo desde `INICIADO`): si la recuperación decidió primero, el
coordinador ejecuta esa decisión. Ver `docs/decisiones.md`.

### Demostración del 2PC

```bash
./scripts/demo-2pc/ejecutar.sh                  # 6 escenarios con línea de tiempo; genera docs/resultados-2pc.md
./scripts/demo-2pc/ejecutar.sh --escenario 6    # uno solo, para mostrar en vivo
```

Escenarios: camino feliz y recepción, voto NO, destino sin enlace (queda `SOLICITADO` y se
retoma), el coordinador muere después de decidir, muere en `INICIADO` con los dos SÍ, y
queda caído con un `COMPROMETIDO` en duda (`/salud` muestra las transacciones en duda, una
venta del producto recibe 409, el reconciliador espera sin deshacer). Las fallas se
inyectan con `_prueba.falla`, solo con `PERMITIR_FALLAS_INYECTADAS=1`, que define el
adaptador de plataforma. Cortar el enlace y detener el coordinador pasan por
`scripts/corte-enlace/enlace-compose.sh` y `scripts/demo-2pc/plataforma-compose.sh`; en GCP
solo cambian esos dos archivos.

## Índices

```bash
./scripts/prueba-indices/ejecutar.sh       # ~2 min; genera docs/resultados-indices.md
ESCALA=0.2 ./scripts/prueba-indices/ejecutar.sh   # menos volumen, más rápido
```

Mide consultas del código antes y después de los índices de `sucursal/009` y `central/010`,
sobre bases de prueba con volumen que crea y borra en los mismos contenedores
(`sima_indices_sucursal` en `db-sucursal-1`, `sima_indices_central` en `db-central`; nunca en
las bases de operación). Los SQL se capturan ejecutando las funciones del núcleo compilado,
así que son exactamente los del código. Incluye experimentos: B-tree contra hash en la caja,
el índice parcial de la outbox, el costo de los índices de `movimiento` y el conteo de
`/salud`. `CONSERVAR=1` deja las bases para explorar.

## Probar todo

```bash
./scripts/probar-todo.sh       # desde base limpia, todas las suites en orden (~7 min)
```

Corre primero las suites que dejan la base cuadrando y `verificar.mjs`, y al final las que
la descuadran a propósito (participante y concurrencia). Resume, regenera
`docs/resultados-2pc.md`, `docs/resultados-concurrencia.md` y `docs/resultados-indices.md`, y
sale con código 1 si algo falla. Al terminar, la base queda descuadrada: volver a cero para seguir trabajando.

### Reloj de Docker

La firma entre nodos tiene una ventana de ±60 s (ver «Autenticación entre nodos»), así que el reloj de la
VM de Docker Desktop tiene que coincidir con el del Mac. Puede atrasarse bajo carga o tras suspender el
Mac, y entonces las suites fallan con `401 FIRMA_DE_SERVICIO_INVALIDA` aunque el sistema esté bien.
`probar-todo.sh` lo mide antes de empezar y aborta si la diferencia pasa de 5 s con
`El reloj de Docker está desfasado N s: sincronizalo`. Para sincronizarlo:

```bash
docker run --rm --privileged --entrypoint hwclock postgres:16-alpine -s     # pone la VM en hora
docker run --rm --entrypoint date postgres:16-alpine -u; date -u            # comprobar: deben coincidir
```

Reiniciar Docker Desktop también lo corrige.

## Corte de enlace (demostración)

```bash
docker compose down -v && docker compose up -d --build && ./scripts/migrar.sh && ./scripts/sembrar.sh
./scripts/corte-enlace/ejecutar.sh 1
```

Imprime una línea de tiempo:
1. Corta el enlace de la sucursal 1.
2. Vende con el enlace caído y muestra el outbox creciendo y el estado de `/salud`.
3. Consulta la red en el central, que muestra el dato viejo con su antigüedad.
4. Entrega al central, en desorden, dos ventas del mismo producto.
5. Reconecta y mide cuánto tarda en vaciarse la cola.
6. Verifica contra las bases (a) existencias, (b) documentos y movimientos,
   (c) entrega repetida y (d) desorden.

Sale con código distinto de 0 si algo no cuadra. Si se interrumpe, reconecta la
sucursal al salir.

Cortar y reconectar pasan solo por `cortar_enlace N` / `restablecer_enlace N` de
`scripts/corte-enlace/enlace-compose.sh` (`docker network
disconnect/connect`). Para otra plataforma se escribe otro adaptador con las mismas
funciones y se elige con `ENLACE_ADAPTADOR=…`.

A mano: `docker network disconnect sima-red-central sima-api-sucursal-1` y
`docker network connect --alias api-sucursal-1 sima-red-central sima-api-sucursal-1`.

## Despliegue en Google Cloud

Proyecto `<id-proyecto>`, región `us-central1`. Cada pieza usa un servicio por una razón de diseño:

| Pieza | Servicio |
|---|---|
| Sucursales | Compute Engine: 3 VMs `e2-small`, una por zona (`a`, `b`, `c`), cada una con su API, su PostgreSQL 16 y su propia cuenta de servicio (`sa-sucursal-1..3`) |
| Central y coordinador del 2PC | Cloud Run `api-central`, privado: solo lo invocan cuentas con token OIDC |
| Base del central y bitácora del 2PC | Cloud SQL para PostgreSQL 16, solo con IP privada en la VPC |
| Sincronización sucursal → central | Pub/Sub, tema `sima-sincronizacion` con suscripción push (OIDC); tras 15 entregas fallidas, al tema de muertos |
| Recuperación del 2PC | Cloud Scheduler la dispara cada minuto, aunque Cloud Run esté en cero instancias |
| Analítica | Datastream (captura de cambios de Cloud SQL, por la VM `proxy-datastream`) → BigQuery `sima_analitica` (esquema estrella, MERGE cada hora; utilidad con costo en `v_utilidad`) → Looker Studio |
| Claves | Secret Manager: claves de sesión y de firma HMAC por sucursal |
| Imágenes | Cloud Build y Artifact Registry: la misma imagen corre en Cloud Run y en las VMs |
| Observabilidad y costos | Cloud Monitoring (verificación de `/salud` y alertas) y presupuesto sobre el crédito con avisos al 25, 50, 80 y 100 % |

**Infraestructura como código.** `infra/terraform/` administra 133 recursos, con el estado en
`gs://<id-proyecto>-tfstate`. Terraform adoptó la infraestructura existente con bloques `import`, sin recrearla
(ver `infra/terraform/README.md`). Los scripts de `infra/gcp/` quedan para la operación:

```bash
infra/gcp/encender-apagar.sh encender           # enciende Cloud SQL, las VMs y el stream (apagar SIEMPRE al terminar)
infra/gcp/encender-apagar.sh apagar
infra/gcp/04-imagenes.sh                         # construye y publica las imágenes con Cloud Build
infra/gcp/verificar-gcp.sh --sin-analitica       # comprobaciones contra el ambiente real
infra/gcp/verificar-gcp.sh --solo-analitica      # Datastream, BigQuery y vistas
```

El procedimiento completo, con respaldos y comprobaciones por paso, está en `docs/despliegue-gcp.md`.
Para la demo híbrida se usan los scripts de `infra/hibrido/` (sección siguiente), no `encender-apagar.sh`.

**Costos.** Al 4 nov 2026 se gastaron US$25.69 del crédito de prueba de US$300; del costo de lista, Compute Engine
es el 41 % y Cloud SQL el 23 %. Las estimaciones por escenario están en el estudio de factibilidad.

## Acceso a api-central en GCP

En GCP `api-central` es privado: solo lo invocan las cuentas de `infra/terraform/acceso-central.tf`
(push de Pub/Sub, sucursales, Cloud Scheduler, la verificación de `/salud`) y la cuenta que opera
el proyecto.

```bash
# Scripts y pruebas de carga: token de identidad de gcloud (dura una hora)
curl -H "Authorization: Bearer $(gcloud auth print-identity-token)" "$URL/salud"

# Demo en el navegador: proxy local que agrega el token (en Cloud Shell, Vista previa web → 8080)
gcloud run services proxy api-central --region=us-central1 --port=8080
```

## Demo híbrida: la sucursal 1 en esta Mac, el resto en GCP

Para la demo final, Centro corre en la Mac con Docker (`docker-compose.hibrido.yml`) y el central y las sucursales 2 y 3 en GCP; el
corte de enlace es apagar el Wi-Fi. Se llega a GCP **sin llaves de cuenta de servicio**: la API suplanta a `sa-sucursal-1` con las
credenciales de quien opera (`CENTRAL_AUTENTICACION=SUPLANTACION`). Todo se maneja con tres scripts de `infra/hibrido/`:

```bash
infra/hibrido/traspaso.sh    # una vez: copia la base de la VM a la Mac, pone el candado en la VM y la apaga
infra/hibrido/preparar.sh    # antes de presentar: enciende GCP (sin la VM 1), claves, stack, reloj, túneles, comprobaciones → «LISTO PARA LA DEMO»
infra/hibrido/terminar.sh    # respaldo de la base de Centro, baja el stack (nunca con -v) y apaga GCP
infra/hibrido/regreso.sh     # después de la revisión: la sucursal 1 vuelve a su VM con todo lo vendido en la Mac
```

Para grabar o presentar sin consola, `node infra/hibrido/panel/servidor.mjs` abre un panel en `http://127.0.0.1:4400` con
botones que corren esos mismos scripts (preparar, estado de las sucursales, túnel de Centro, recursos de GCP y apagar).
Escucha solo en `127.0.0.1` y del navegador solo recibe el id de una acción fija.

**La base de Centro vive en esta Mac: no borrar volúmenes ni resetear Docker Desktop hasta el regreso a la VM.** Procedimiento y
comprobaciones en `docs/despliegue-gcp.md` (partes A, B y C), decisiones en `docs/decisiones.md` (2026-11-02).

## Autenticación entre nodos

Las peticiones entre nodos llevan una **firma HMAC por petición** con una clave distinta por sucursal
(`X-SIMA-Firma: s=<id_sucursal>,t=<unix>,v1=<hex>`; HMAC-SHA256 sobre `sima-v1 \n t \n MÉTODO \n ruta con query \n
sha256(cuerpo)`, ventana de ±60 s). Va en las dos direcciones: el central firma hacia una sucursal con la clave de esa sucursal
(preparar, confirmar, abortar), y la sucursal firma hacia el central con la suya (réplica, `/red/*`, decisión del reconciliador,
sincronización). El central identifica a la sucursal por `s=` y esa sucursal manda sobre el `id_sucursal` del cuerpo o de la
query. Sin firma, con una inválida, vencida o de otra sucursal: `401`, y el motivo (sin la clave) queda en el registro.
`/salud` es pública. En GCP se suma al OIDC de Cloud Run, no lo reemplaza. El detalle, lo descartado y lo que queda sin cubrir
están en `docs/decisiones.md` («Autenticación entre nodos»).

Las claves de desarrollo están en `docker-compose.yml` (`SERVICIO_CLAVE_S1`, `_S2`, `_S3`; cada sucursal usa la suya como
`SERVICIO_CLAVE`) y se cambian en `.env` (`.env.ejemplo`). Los scripts que llaman rutas de servicio a mano firman con
`scripts/lib/firma-servicio.mjs`.

## Verificaciones de sincronización

```bash
node scripts/prueba-consumidor/prueba.mjs     # consumidor (pausa los publicadores mientras corre): entrega, duplicados, entrega simultánea, desorden, inválidos, base caída
node scripts/prueba-publicador/prueba.mjs     # publicador: flujo normal, central apagado, base del central caída, mensaje envenenado
node scripts/prueba-conexion-perdida/prueba.mjs   # una conexión que muere en medio de una transacción no tumba la API
node scripts/prueba-movimiento-inmutable/prueba.mjs   # UPDATE, DELETE y TRUNCATE sobre movimiento fallan en los 4 nodos (trigger)
node scripts/prueba-coordinador/prueba.mjs    # traslados de punta a punta: camino feliz, recepción, voto NO, SOLICITADO retomado, carrera coordinador vs. recuperación
node scripts/prueba-reconciliador/prueba.mjs  # reconciliador: huérfana ABORTADO, central caído (espera), sin enlace, 404 explícito
node scripts/prueba-participante/prueba.mjs   # participante sin coordinador (deja filas FALLIDO: correr sobre una base descartable)
node scripts/corte-enlace/verificar.mjs       # el central cuadra con las 3 sucursales: (a) existencias, (b) documentos, líneas, movimientos, inbox
scripts/prueba-corte-iptables/prueba.sh       # capa de iptables del corte de enlace de GCP: corta también las conexiones ya abiertas (solo Docker Desktop para Mac)
node scripts/prueba-idempotencia/prueba.mjs   # llave de idempotencia de POST /ventas y /recepciones: repetición, 10 simultáneas con la misma llave, conflicto de contenido, sin llave, enlace cortado, el central recibe cada documento una vez
node scripts/prueba-caja/prueba.mjs           # caja: /caja servida, endpoints de lectura y de reenvío, traslado pedido por el destino y recepción, sin enlace
scripts/prueba-roles/ejecutar.sh              # roles: matriz ruta × rol, rutas sin declaración, traslado pedido por el destino, traslado_local, id_usuario NULL solo en el TRASLADO del origen
node scripts/prueba-ventas/prueba.mjs            # ventas (solo lectura): rangos y días de Guatemala, resumen contra la base, paginación por llave, consolidado del central con vigente_al, sin enlace
node scripts/prueba-empleados/prueba.mjs         # empleados: alta, rol, contraseña y baja tras la réplica, último ADMIN, write skew (con control en base descartable), renombrar sucursal
node scripts/prueba-catalogo/prueba.mjs          # catálogo (ADMIN): producto nuevo con 0 en las 3 sucursales, validaciones, precio nuevo sin tocar el historial, baja y reactivación, codigo_barras inmutable (trigger), 403 a ENCARGADO y CAJERO, sin enlace
node scripts/prueba-costos/prueba.mjs            # costo promedio ponderado móvil: 8 u @10 + 4 @16, redondeo, 50 ventas concurrentes (PESIMISTA y OPTIMISTA), traslado con costo, PREPARE repetido, saldo inicial, reportes, auditoría (antes: npm run build -w @sima/nucleo)
node scripts/prueba-blindaje/prueba.mjs          # firma entre nodos: 401 sin firma / alterada / vencida / de otra sucursal, 403 al pedir lo de otra, preparada que no se confirma sin firma, POST /traslados idempotente, arranque sin clave (antes: npm run build)
node scripts/prueba-token-identidad/prueba.mjs   # token de identidad de la sucursal con un metadata server falso: caché, renovación, 401/403 (antes: npm run build)
scripts/prueba-pubsub/ejecutar.sh             # Pub/Sub con el emulador: clasificación del adaptador, venta, sobre push, corte, NOT_FOUND, mensaje inválido a muertos, base del central caída → muertos → reenvío (vuelve a HTTP al terminar)
infra/terraform/probar-revisar-plan.sh        # clasificación de revisar-plan.sh con planes falsos
```

## Variables de entorno

Se pasan por `docker-compose.yml`; las marcadas con Compose se pueden cambiar al levantar
(por ejemplo `ESTRATEGIA_CONCURRENCIA=OPTIMISTA docker compose up -d api-sucursal-2`).

| Variable | Dónde | Por defecto | Qué es |
|---|---|---|---|
| `ESTRATEGIA_CONCURRENCIA` | sucursal · Compose | `PESIMISTA` | `PESIMISTA` (`SELECT … FOR UPDATE`) u `OPTIMISTA` (`UPDATE … WHERE version = :v` con reintento). `SIN_CONTROL` existe **solo para la prueba de concurrencia** |
| `REINTENTOS_OPTIMISTA` | sucursal · Compose | `5` | Reintentos máximos de la estrategia optimista |
| `LOCK_TIMEOUT_MS` | sucursal · Compose | `3000` | Plazo para esperar una fila bloqueada en venta, ajuste y preparación de traslados. Vencido: 409 `PRODUCTO_BLOQUEADO` |
| `SESION_CLAVE` | sucursal · Compose | — (**obligatoria**) | Firma los tokens de sesión. Mínimo 32 bytes; sin ella la API no arranca. Una distinta por sucursal. En `docker-compose.yml` son claves **solo de desarrollo**; en GCP irán en Secret Manager (no está hecho) |
| `SESION_DURACION_H` | sucursal · Compose | `10` | Horas de vida de una sesión (admite decimales) |
| `SESION_BLOQUEO_MIN` | sucursal · Compose | `15` | Minutos de bloqueo tras 5 contraseñas malas (admite decimales; la prueba lo acorta) |
| `SERVICIO_CLAVE` | sucursal · Compose | — (**obligatoria**) | Clave con que esta sucursal firma sus peticiones al central y verifica las del central. Mínimo 32 bytes, distinta de `SESION_CLAVE`; sin ella la API no arranca. En Compose sale de `SERVICIO_CLAVE_S<N>` de su sucursal |
| `SERVICIO_CLAVE_S<N>` | central · Compose | — (**al menos una**) | Clave de servicio de la sucursal N (`_S1`, `_S2`, `_S3`; vale cualquier N). Mínimo 32 bytes y todas distintas; sin ellas, o con una corta o repetida, el central no arranca. Solo desarrollo en Compose; en GCP irán en Secret Manager (no está hecho) |
| `SERVICIO_VENTANA_S` | ambas APIs · Compose | `60` | Segundos de tolerancia entre el reloj del firmante y el del verificador. Pasado eso: 401 con motivo `VENCIDA` en el registro |
| `PERMITIR_SIN_CONTROL` | sucursal · Compose | `0` | Con `1`, permite `SIN_CONTROL`. Sin ella, esa estrategia hace fallar el arranque. **Solo la define `ejecutar.sh`** |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` | todos · Compose | `sima` / `[REDACTADO]` | Credenciales de las cuatro bases |
| `SINCRONIZACION_ACTIVA` | sucursal · Compose | `1` | Con `0`, el publicador no arranca: la caja vende igual y el outbox solo crece |
| `TRANSPORTE_SINCRONIZACION` | sucursal | `HTTP` | Adaptador de transporte: `HTTP` o `PUBSUB` (`docker-compose.pubsub.yml` lo pone en `PUBSUB`) |
| `PUBSUB_PROYECTO` / `PUBSUB_TEMA` | sucursal | — | Proyecto y tema de Pub/Sub; obligatorios con `PUBSUB` |
| `PUBSUB_EMULATOR_HOST` | sucursal · scripts | — | Con valor, la librería y `muertos.mjs` van al emulador en vez de a GCP |
| `CENTRAL_AUTENTICACION` | sucursal | `NINGUNA` | `METADATA`: las llamadas al central llevan un token de identidad del metadata server, con audiencia `CENTRAL_URL` (VMs de GCP, donde api-central es privado). `SUPLANTACION`: lo mismo desde fuera de GCP (la sucursal 1 de la demo híbrida): el token lo emite IAM Credentials para `SUPLANTAR_CUENTA` a pedido del ADC de usuario de `GOOGLE_APPLICATION_CREDENTIALS`, y Pub/Sub se publica con esa misma identidad |
| `SUPLANTAR_CUENTA` | sucursal | — | Con `SUPLANTACION`: correo de la cuenta de servicio de la sucursal (`sa-sucursal-N@<proyecto>.iam.gserviceaccount.com`). Obligatoria en ese modo |
| `METADATA_URL` | sucursal | `http://metadata.google.internal` | Metadata server del que sale el token (la prueba usa uno falso) |
| `CENTRAL_URL` | sucursal | `http://api-central:3000` | Destino del transporte HTTP |
| `SINCRONIZACION_INTERVALO_MS` | sucursal · Compose | `2000` | Cada cuánto corre un ciclo del publicador |
| `SINCRONIZACION_LOTE` | sucursal · Compose | `20` | Filas por ciclo |
| `SINCRONIZACION_TIMEOUT_MS` | sucursal · Compose | `3000` | Timeout por envío. Peor caso de transacción abierta del publicador: lote × timeout = 60 s |
| `SINCRONIZACION_MAX_RECHAZOS` | sucursal · Compose | `5` | Rechazos 4xx antes de pasar una fila a `FALLIDO` |
| `ID_SUCURSAL` | sucursal | — | Fijada por servicio en el Compose |
| `BD_URL` | ambas APIs | — | Cadena de conexión a su base; fijada en el Compose |
| `BD_MAX_CONEXIONES` | sucursal | `20` | Tamaño del pool de conexiones |
| `PUERTO` | ambas APIs | `3000` | Puerto dentro del contenedor |
| `PUERTO_DB_CENTRAL` | Compose y scripts | `5432` | Puerto de `db-central` en el anfitrión |
| `RECONCILIADOR_INTERVALO_MS` | sucursal · Compose | `5000` | Cada cuánto revisa `pg_prepared_xacts` el reconciliador |
| `RECONCILIADOR_PLAZO_MS` | sucursal · Compose | `30000` | Solo pregunta por transacciones preparadas más viejas que esto |
| `PLAZO_VOTACION_MS` | central · Compose | `10000` | Plazo mínimo de votación del 2PC (el efectivo crece con las líneas × `LOCK_TIMEOUT_MS`) |
| `RECUPERACION_INTERVALO_MS` | central · Compose | `5000` | Cada cuánto corre la recuperación del coordinador dentro del proceso |
| `RECUPERACION_PLAZO_INICIADO_MS` | central · Compose | `60000` | Una transacción global en `INICIADO` más vieja que esto se aborta |
| `PERMITIR_FALLAS_INYECTADAS` | central · Compose | `0` | Con `1`, `POST /traslados` acepta `_prueba` (fallas y retardos). **Solo los scripts de prueba** |

## Prueba de concurrencia

N ventas simultáneas del mismo producto contra la misma sucursal, con las tres
estrategias y dos escenarios (existencia 1 y existencia 20):

```bash
./scripts/prueba-concurrencia/ejecutar.sh 50
```

Reconstruye la imagen, corre las seis combinaciones contra la sucursal 1, verifica cada
corrida contra la base y regenera `docs/resultados-concurrencia.md`.
Al terminar hace `docker compose up -d`: las tres sucursales quedan con la imagen nueva y
en `PESIMISTA`.

`reiniciar-existencia.sh` deja el producto en su existencia inicial registrando un AJUSTE
por conteo con `POST /ajustes` (no escribe en la base por fuera); la prueba toma su foto
inicial después de ese ajuste.

Una sola corrida, contra la estrategia que tenga activa la API:

```bash
./scripts/prueba-concurrencia/reiniciar-existencia.sh 1 7501000000046 1
node scripts/prueba-concurrencia/prueba.mjs --codigo 7501000000046 --n 50   # --usuario cajero1 (inicia sesión)
```

`prueba.mjs` sale con código 2 si alguna verificación falla (por ejemplo, con
`SIN_CONTROL`, que sobrevende a propósito).

## Pruebas de carga

k6 y pgbench corren como contenedores (`grafana/k6:1.3.0` y el `pgbench` de la imagen
`postgres:16-alpine`); la app no suma dependencias.

```bash
./scripts/prueba-carga/ejecutar.sh completo     # unos 30 min (escalones hasta 400/s); regenera docs/resultados-carga.md (parte local)
./scripts/prueba-carga/ejecutar.sh corto        # unos 3 min; también regenera el informe
./scripts/prueba-carga/ejecutar.sh corto --sin-informe   # el que corre probar-todo.sh: verifica sin tocar el informe
```

- **Ventas** (`ventas.js`) contra `api-sucursal-1`, con `PESIMISTA` y `OPTIMISTA`. Escalones de
  tasa constante en tres escenarios:
  - **caliente:** un solo producto;
  - **azar:** 30 productos;
  - **agotamiento:** más ventas que unidades.
- **Consulta de existencias de la red** (`existencias-red.js`) contra `api-central`.
- **pgbench** (`venta.sql`): la misma transacción de venta, directo contra una **copia** de la
  base de la sucursal 1, que se borra al terminar.
- **Verificación contra la base** (`verificacion.sql`) después de cada corrida: sobreventa 0,
  existencia que cuadra, un documento, movimiento y outbox por venta aceptada. Al final se
  comprueba que el central cuadra.

Necesita la red levantada y sembrada. Las ventas son reales, y la base queda cuadrando. Los
resultados crudos quedan en `scripts/prueba-carga/resultados/`, sin versionar. En GCP:
`infra/gcp/carga-gcp.sh preparar` (una vez) y `infra/gcp/carga-gcp.sh correr`.

## Nota para macOS

Si `docker compose` falla con `docker-credential-desktop not found`, falta
`/Applications/Docker.app/Contents/Resources/bin` en el `PATH`. Los scripts del
repositorio lo agregan solos.

## Documentación adicional

- [SIMA en Google Cloud](docs/cloud.md): arquitectura en GCP, servicios, seguridad, despliegue paso a paso y costos
- [Decisiones de diseño](docs/decisiones.md)
- [Despliegue en GCP](docs/despliegue-gcp.md)
- [Resultados del 2PC](docs/resultados-2pc.md) · [Pruebas de carga](docs/resultados-carga.md) · [Concurrencia](docs/resultados-concurrencia.md) · [Índices](docs/resultados-indices.md)
- [Tableros de Looker Studio](docs/tableros.md)
