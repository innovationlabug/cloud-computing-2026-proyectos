# Despliegue en GCP: de `28d0382` a la versión con inicio de sesión y firma entre nodos

Procedimiento comando por comando para llevar a `<id-proyecto>` todo lo que entró después del
último despliegue:
- réplica central → sucursal;
- inicio de sesión, roles, empleados y ventas;
- firma HMAC entre nodos;
- por reponer, `proveedor.activo`, recepción de proveedor y catálogo;
- reconciliador;
- idempotencia en ventas y recepciones;
- ticket y reporte del día;
- costo promedio ponderado móvil, cárdex valorizado y lista de costos del distribuidor (**incluye un paso
  obligatorio: el saldo inicial, 4.3 bis**).

Las decisiones están en `docs/decisiones.md` (2026-10-28, "Despliegue en GCP: una cuenta de
servicio por sucursal y claves en Secret Manager").

**Dónde se corre:** todo en **Cloud Shell**, en `~/sima`, con GCP encendido
(`infra/gcp/encender-apagar.sh encender`).

**Cómo leer los pasos:**
- Cada paso dice qué tiene que salir.
- Si sale otra cosa, **se para** y se resuelve antes de seguir.
- Las consultas marcadas "**tiene que dar 0**" son condiciones para migrar.

**La ventana:** de 1 a 1,5 horas sin ventas ni traslados. Va de la parte 2 a la 4. Las
sucursales están detenidas desde 2.6 hasta 4.3.

**Punto de no retorno:** paso **3.2**, las migraciones del central. Antes de ese paso todo se
deshace sin pérdida; la parte 6 explica cómo, según dónde haya fallado.

---

# Orden de ejecución del 2026-11-02 (leer primero)

Todo lo que sigue se usa **en este orden**. El traspaso de la sucursal 1 a la Mac (parte B) **no se hace hasta que la parte A
esté completa y `verificar-gcp.sh` esté en verde** (A9 y A11): el traspaso copia la base de la VM, y esa base tiene que ser la
ya migrada y con su saldo inicial.

| # | Dónde | Qué |
|---|---|---|
| 1 | Cloud Shell | Parte A: redespliegue de GCP con esta rama, BigQuery y `verificar-gcp.sh` en verde |
| 2 | La Mac | Parte B: configuración de la Mac **una vez**, `infra/hibrido/traspaso.sh` y un ensayo con `infra/hibrido/preparar.sh` |
| 3 | La Mac | El 4 nov: `infra/hibrido/preparar.sh` → demo → `infra/hibrido/terminar.sh` |
| 4 | La Mac | Después de la revisión de Cloud: `infra/hibrido/regreso.sh` |

Quién corre qué: los `terraform apply`, los `gcloud` que escriben y las escrituras en bases de GCP los corre una persona, a
mano o con estos scripts; nada de esto lo ejecuta el repositorio por su cuenta.

## Parte A — Redespliegue de GCP con esta rama (`demo-hibrida`)

### A0. Traer la rama a Cloud Shell y averiguar desde qué versión se parte

Después de revisar la rama, se hace el commit y el push, y en Cloud Shell:

```bash
cd ~/sima && git fetch origin && git checkout demo-hibrida && git pull && git log -1 --oneline
```

**GCP puede estar apagado**, y sin él no hay VMs ni Cloud SQL que consultar: se enciende **antes** de cualquier consulta (tarda unos
minutos; termina cuando el stream de Datastream vuelve a `RUNNING`):

```bash
infra/gcp/encender-apagar.sh encender
```

Después se pegan los ayudantes de **0.1** (más abajo: `source infra/gcp/variables.sh`, `zona`, `en_vm`, `sql_sucursal`, `sql_central`,
`pendientes_pubsub`) y recién entonces se mira qué migraciones tiene cada base:

```bash
sql_central "SELECT max(nombre) FROM migracion_aplicada;"
for N in 1 2 3; do sql_sucursal "$N" "SELECT max(nombre) FROM migracion_aplicada;"; done
```

| Central | Sucursales | Qué camino |
|---|---|---|
| `010_indices_consultas.sql` | `009_indices_consultas.sql` | GCP sigue en `28d0382`: **runbook completo** (partes 0 a 5 de este documento, que ya incluyen el saldo inicial 4.3 bis), con los agregados marcados «(2026-11-02)». Después, A9 a A11 |
| `018_codigo_barras_inmutable.sql` | `016_idempotencia_documento.sql` | Ya tiene el inicio de sesión y la firma: **camino corto**, A1 a A11 |
| otra cosa | | **Parar** y averiguar qué se aplicó |

Los pasos A1 a A8 de abajo (con A7 bis) son el camino corto. Cada paso dice qué tiene que salir; si sale otra cosa, se para.

### A1. Comprobaciones con todo encendido (ventana sin ventas ni traslados: 40 a 60 minutos)

```bash
sql_central "SELECT COUNT(*) FROM traslado WHERE estado IN ('SOLICITADO', 'PREPARANDO');"        # 0
sql_central "SELECT COUNT(*) FROM transaccion_global WHERE estado <> 'FINALIZADO';"              # 0
for N in 1 2 3; do
  sql_sucursal "$N" "SELECT COUNT(*) FROM pg_prepared_xacts WHERE database = current_database();"   # 0
  sql_sucursal "$N" "SELECT COUNT(*) FROM outbox WHERE estado <> 'ENVIADO';"                          # 0
done
pendientes_pubsub                                   # 0 sin entregar
node scripts/pubsub/muertos.mjs listar              # 0 mensaje(s) muerto(s)
```

### A2. Respaldos

Los de **1.5** (volcado lógico de cada sucursal) y **2.4** (respaldo de Cloud SQL), tal cual. Anotar el id del respaldo.

### A3. Imágenes (construidas desde la rama)

```bash
cd ~/sima && infra/gcp/04-imagenes.sh
```

Tiene que terminar con «Imágenes publicadas con la etiqueta `<sha>`». Anotar la etiqueta.

### A4. Migración del central

```bash
gcloud run jobs execute migrar-central --region=us-central1 --wait
gcloud logging read 'resource.type="cloud_run_job" AND resource.labels.job_name="migrar-central"' --freshness=15m \
  --format='value(textPayload)' | grep -E 'aplicada|Migraciones|Semilla|Lista|ERROR' | tac
```

Tiene que mostrar: `aplicada 019_costo_promedio.sql`, `Migraciones de central: 1 aplicadas, 18 ya estaban` y
`Lista de costos: 50 vigencias sembradas en el central`. **Desde aquí no hay vuelta atrás sin restaurar el respaldo de A2.**

### A5. api-central con la imagen nueva

Terraform ignora la imagen: se estrena con `gcloud`.

```bash
gcloud run services update api-central --region=us-central1 --image="$IMAGEN:<etiqueta de A3>"
gcloud run services describe api-central --region=us-central1 --format='value(status.latestReadyRevisionName,status.traffic[0].revisionName)'
```

La revisión lista y la del tráfico son la misma, y distinta de `REV_ANTERIOR`. Después, el bloque de verificación de **3.3**
(`/salud` en OK, 401 sin firma, 200 firmado, Scheduler en 200).

### A6. Terraform: API de IAM Credentials, permiso de suplantación y script de arranque de las VMs

Tres cosas, con `-target` para no tocar la consulta programada de BigQuery (esa va en A10):
- habilita `iamcredentials.googleapis.com`;
- da a **tu usuario** (no al de otra persona) `roles/iam.serviceAccountTokenCreator` **solo sobre `sa-sucursal-1`**;
- actualiza el script de arranque de las VMs (el candado `nodo-externo` del modo híbrido).

```bash
cd ~/sima/infra/terraform && source ./credenciales.sh
./revisar-plan.sh \
  -target=google_project_service.api \
  -target=google_service_account_iam_member.operador_suplanta_sucursal_1 \
  -target=google_compute_instance.sucursal
```

**Tiene que salir 1** (porque cambian VMs; `revisar-plan.sh` no las da por buenas solas) con exactamente esto:
- `+ crear google_project_service.api["iamcredentials.googleapis.com"]`;
- `+ crear google_service_account_iam_member.operador_suplanta_sucursal_1`, con `member = "user:<tu correo>"` y
  `role = roles/iam.serviceAccountTokenCreator`;
- `~ cambiar google_compute_instance.sucursal["1"]`, `["2"]` y `["3"]`, **solo** `metadata` (el `startup-script`).

Comprobar que no se reemplaza nada y a quién se da el permiso:

```bash
grep -E "will be updated in-place|must be replaced|forces replacement" plan.log      # tres «updated in-place», ninguno de los otros
grep -n 'member' plan.log | grep user:                                                # TU correo, no una cuenta de servicio
```

- Si el plan falla con «No se pudo saber qué usuario opera la sucursal 1», las credenciales de Cloud Shell no traen el correo:
  `export TF_VAR_operador_correo=<correo de gcloud de quien va a correr la demo>` y repetir. El plan nunca propone quitar el permiso.
- **El usuario del permiso es quien opera la Mac.** Si Terraform lo corre otra persona, usar `TF_VAR_operador_correo`.

```bash
terraform apply plan.bin
```

### A7. Reiniciar las VMs con el arranque y la imagen nuevos (una por vez)

```bash
for N in 1 2 3; do
  gcloud compute instances stop "sucursal-$N" --zone="$(zona "$N")"
  gcloud compute instances start "sucursal-$N" --zone="$(zona "$N")"
  sleep 90
  gcloud compute instances get-serial-port-output "sucursal-$N" --zone="$(zona "$N")" 2>/dev/null \
    | grep -E 'Clave |AVISO clave|ERROR clave|aplicada|Migraciones de|Semilla|CANDADO|Sucursal . lista|rror' | tail -12
  en_vm "$N" 'curl -s localhost:3000/salud' | jq '{estado, transporte: .sincronizacion.transporte, replica: .replica.enlace, error_sinc: .sincronizacion.ultimo_error.codigo}'
done
```

Por cada VM tiene que mostrar, en este orden: `Clave sesion: leída de Secret Manager`, `aplicada 017_copia_existencia_red.sql`,
`aplicada 018_costo_promedio.sql`, `Migraciones de sucursal: 2 aplicadas, 16 ya estaban`, `Sucursal N lista`; **sin** la línea
`CANDADO` (todavía no hay traspaso); y `estado: "OK"`, `transporte: "PUBSUB"`, `replica: "OK"`.

### A7 bis. La lista de costos llegó a las tres sucursales (cursor de réplica)

**Por qué hace falta.** El job de A4 siembra `costo_proveedor` en el central con `version_replica` nuevas (en la semilla, 161 a 210). Entre A4 y A7 las
sucursales todavía corrían el código **viejo**: pidieron esas páginas de réplica, **descartaron el campo `costos`** (no lo conocían) y, aun así,
avanzaron `estado_replica.ultima_version` (a 212 en el despliegue del 2026-11-03). Con el código nuevo el central ya no tiene nada «más nuevo» que
mandarles: las sucursales quedan con **0 costos**, el saldo inicial (A8) es imposible y `verificar-gcp.sh` falla en cascada. Pasa en cualquier despliegue
sobre bases existentes en el que A4 corra antes que A7. Ver `docs/decisiones.md` (2026-11-03).

Se hace con las tres VMs ya reiniciadas en A7. **No antes**: con el código viejo la réplica volvería a descartar el campo.

1. Comparar el central con cada sucursal:

```bash
MIN_COSTO="$(sql_central "SELECT min(version_replica) FROM costo_proveedor;")"
N_CENTRAL="$(sql_central "SELECT COUNT(*) FROM costo_proveedor;")"
echo "central: $N_CENTRAL vigencias; versión mínima de réplica $MIN_COSTO"
for N in 1 2 3; do
  sql_sucursal "$N" "SELECT 'sucursal $N: cursor ' || ultima_version || ', costos ' || (SELECT COUNT(*) FROM costo_proveedor) FROM estado_replica;"
done
```

   **Tiene que decir**, en cada sucursal, `costos` igual a `$N_CENTRAL` (50 en la semilla). Si es así, **no hay nada que corregir**: se sigue a A8.
   Si `MIN_COSTO` o `N_CENTRAL` salen vacíos o en 0, la lista no se sembró: volver a A4.

2. Si una sucursal tiene **menos** costos que el central, su cursor ya pasó la versión mínima. Se regresa a una versión anterior a la mínima:

```bash
for N in 1 2 3; do
  FALTAN="$(sql_sucursal "$N" "SELECT COUNT(*) < $N_CENTRAL FROM costo_proveedor;")"      # t = le faltan costos
  if [ "$FALTAN" = "t" ]; then
    echo "sucursal $N: se regresa el cursor a $((MIN_COSTO - 1))"
    sql_sucursal "$N" "UPDATE estado_replica SET ultima_version = $((MIN_COSTO - 1));"
  fi
done
```

   Tiene que imprimir `UPDATE 1` por cada sucursal corregida. Es seguro: la réplica solo pisa filas con una versión **mayor** (las vigencias de costo
   son inmutables y se insertan sin duplicar), así que volver a pedir páginas ya aplicadas no cambia lo que ya está al día.
   Es una escritura a mano en las bases de las sucursales: es lo único del despliegue que toca `estado_replica`.

3. Esperar una ronda de réplica (cada 10 s) y repetir el paso 1. Las tres sucursales tienen que decir `costos` igual a `$N_CENTRAL`.
   Si siguen en 0, mirar `/salud` de la sucursal (`replica.ultimo_error`) antes de repetir el paso 2.

### A8. Saldo inicial valorado ⚠️ OBLIGATORIO antes de reabrir las ventas

Es el paso **4.3 bis**, tal cual (lista de costos igual en las cuatro bases y `registrar_saldo_inicial` en cada sucursal). Tiene que
dar **0 filas** en las tres y **0** existencias sin valor.

### A9. Primera pasada de `verificar-gcp.sh`: secciones 1 a 5 y 8

```bash
cd ~/sima && infra/gcp/verificar-gcp.sh --sin-analitica
```

**Tiene que terminar con «0 fallaron».** Corre, además de lo de siempre, los chequeos del costo (sección 4c: lista de costos igual
en el central y las tres sucursales, la venta de la sección 4 valorada con su cárdex encadenado, el valor llegando al consolidado
del central, el saldo inicial en cada sucursal), de la copia de red sin enlace (sección 5: durante el corte por `iptables`,
`/red/existencias` responde `COPIA_LOCAL` con la antigüedad de la 2 y la 3 y la fila propia al día) y de la auditoría del cárdex
vacía en las tres sucursales (sección 5b). Sus ventas son las que producen los primeros eventos con costo hacia Datastream.

### A10. Datastream y BigQuery: el costo en el esquema estrella

**Cómo llegan las columnas nuevas.** La migración del central agregó `movimiento.costo_total/valor_antes/valor_despues`,
`linea_documento.costo_total/…` y `existencia_consolidada.valor_inventario`. Datastream lee los cambios de la publicación
`sima_cdc` (que incluye las tablas enteras, no una lista de columnas) y agrega la columna a `sima_cdc.public_*` cuando llega el
primer evento que la trae, como pasó con `proveedor.activo` (5.1). Las ventas de A9 ya los produjeron. El MERGE (05) lee esas
columnas: si todavía no existen en `sima_cdc`, falla con «Unrecognized name», por eso su cambio **no** va en el apply de A6 y se
aplica al final.

Esperar unos 15 minutos (el `data_freshness` del stream) y comprobar:

```bash
for c in "public_movimiento costo_total" "public_movimiento valor_antes" "public_movimiento valor_despues" \
         "public_linea_documento costo_total" "public_existencia_consolidada valor_inventario"; do
  set -- $c
  printf '%s.%s: ' "$1" "$2"
  bq show --format=prettyjson --schema "$PROYECTO:sima_cdc.$1" | jq -r --arg n "$2" '[.[] | select(.name == $n) | "\(.type) \(.mode // "NULLABLE")"][0] // "FALTA"'
done
gcloud datastream streams describe sima-central-cdc --location=us-central1 --format='value(state,errors)'
```

Las cinco tienen que decir `NUMERIC NULLABLE` y el stream `RUNNING` sin errores. `costo_proveedor` (la lista de costos) **no** se
replica a BigQuery a propósito: la utilidad usa el costo que ya viaja en cada movimiento.

- **Si una columna sigue en `FALTA` pasados 20 minutos** y el stream está bien: plan B, sin probar contra GCP. Vuelve a leer la tabla
  completa (con el modo `merge` del stream las filas se actualizan, no se duplican):
  ```bash
  source infra/gcp/operaciones-datastream.sh && backfill_tabla movimiento     # o linea_documento / existencia_consolidada
  ```
- **Si el stream tiene errores:** lo mismo que en 5.1 (casi siempre una columna `REQUIRED`).

Con las columnas en `sima_cdc`:

```bash
cd ~/sima
for f in 08_costo_hecho.sql 06_vistas.sql 07_descripciones_hecho.sql; do
  sed "s/\${PROYECTO}/$PROYECTO/g" "infra/gcp/bigquery/$f" | bq --location="$REGION" query --use_legacy_sql=false --quiet --format=none && echo "✓ $f"
done
# El MERGE una vez, a mano, antes de dejarlo a la consulta programada (muestra si el costo llega y si el histórico sin costo no rompe nada):
sed "s/\${PROYECTO}/$PROYECTO/g" infra/gcp/bigquery/05_merge_hecho.sql | bq --location="$REGION" query --use_legacy_sql=false --quiet --format=none && echo "✓ 05_merge_hecho.sql"
bq query --use_legacy_sql=false "SELECT origen, tipo, COUNT(*) AS n, COUNTIF(costo_total IS NOT NULL) AS con_costo FROM \`$PROYECTO.sima_analitica.hecho_movimiento\` GROUP BY 1, 2 ORDER BY 1, 2"
```

Tiene que verse: **SIMULADO** con `con_costo = 0` en todos los tipos; **CDC** con `con_costo > 0` en `VENTA`, `SALDO_INICIAL` y las
recepciones nuevas, y `con_costo = 0` en lo anterior al costo promedio (las filas históricas siguen sin costo, NULL, no cero).
Ejemplo de la vista nueva:

```bash
bq query --use_legacy_sql=false "SELECT fecha, sucursal, producto, unidades, venta, costo, utilidad, ROUND(margen, 3) AS margen, unidades_sin_costo FROM \`$PROYECTO.sima_analitica.v_utilidad\` ORDER BY fecha DESC, sucursal LIMIT 10"
```

Por último, la consulta programada pasa a la versión con costo (único cambio de Terraform que queda):

```bash
cd ~/sima/infra/terraform && source ./credenciales.sh && ./revisar-plan.sh
```

**Tiene que salir 2, con solo** `~ cambiar google_bigquery_data_transfer_config.hecho_movimiento` (el atributo `params.query`).
Si aparece algo más, se para. Entonces:

```bash
terraform apply plan.bin
```

### A11. Segunda pasada: la analítica

```bash
cd ~/sima && infra/gcp/verificar-gcp.sh --solo-analitica
```

**Tiene que terminar con «0 fallaron».** Además de lo de siempre: las cuatro columnas de costo en `hecho_movimiento`, ventas reales con costo,
`costo_total = costo_linea` en todas las ventas, la historia simulada sin costo, y `v_utilidad` con filas y sin discrepancias. La
consulta programada corre cada hora: la primera corrida después del apply es la que queda; si algo falta, esperar a la
corrida siguiente o lanzarla desde la consola de BigQuery (Consultas programadas → Ejecutar ahora).

> **Puerta del traspaso:** A9 y A11 en «0 fallaron». Recién entonces, la parte B.

---

## Parte B — La sucursal 1 (Centro) en la Mac

**Qué es.** Para la demo, la sucursal 1 corre en la Mac con Docker (su propia base PostgreSQL, su API y su caja). El central y las
sucursales 2 y 3 siguen en GCP. El corte de enlace se hace apagando el Wi-Fi de la Mac: sin comandos. Decisiones y alternativas en
`docs/decisiones.md` (2026-11-02). Los scripts están en `infra/hibrido/`.

> **La base de Centro vive en esta Mac: no borrar volúmenes ni resetear Docker Desktop hasta el regreso a la VM.**
> Después del traspaso, el volumen `sima-hibrido_datos-sucursal-1` es la **única copia al día** de la sucursal 1.
> `terminar.sh` nunca usa `down -v` y guarda un volcado en `~/sima-respaldos/` antes de bajar (0600, con fecha; quedan los últimos 5).

**Nunca dos nodos «sucursal 1».** Dos nodos con el mismo id repetirían los mismos números de documento y el central rechazaría los
del segundo (409 `CONFLICTO_DATOS`, mensajes muertos). Se evita con un **candado** en la VM (metadato `nodo-externo`): con él, el
arranque de la VM deja arriba solo su base y no levanta la API; `encender-apagar.sh encender` se niega mientras exista, y
`preparar.sh` aborta si la VM está `RUNNING`. Por qué no se agregó una guarda en el central: `docs/decisiones.md`.

### B1. Configurar la Mac (una vez; no escribe nada en GCP)

```bash
gcloud auth login                     # con la cuenta a la que A6 le dio el permiso de suplantación
gcloud config set project <id-proyecto>
gcloud auth application-default login
gcloud auth application-default set-quota-project <id-proyecto>
```

El ADC de usuario queda en `~/.config/gcloud/application_default_credentials.json` y se monta **de solo lectura** en el contenedor.
**No se descarga ninguna llave de cuenta de servicio.** Comprobar que la suplantación funciona (no imprime ninguna clave):

```bash
gcloud secrets versions access latest --secret=sima-sesion-clave-s1 \
  --impersonate-service-account=sa-sucursal-1@<id-proyecto>.iam.gserviceaccount.com >/dev/null && echo "suplantación OK"
```

Tiene que decir `suplantación OK`. Si da `PERMISSION_DENIED … iam.serviceAccounts.getAccessToken`: A6 no se aplicó, se aplicó a otro
usuario, o el permiso aún no se propagó (esperar unos minutos).

Si el compose local de la demo completa está arriba (`docker ps` muestra `sima-api-sucursal-1`…), bajarlo **sin borrar volúmenes**:
`docker compose down`. Ocupa los puertos 3001-3003 y es otra «sucursal 1» con datos propios.

### B2. El traspaso: VM → Mac (solo después de A9 y A11 en verde)

```bash
infra/hibrido/traspaso.sh
```

Pregunta una vez si la parte A está completa y verde (es la única barrera). Después, cada paso se comprueba antes de seguir:

1. El ADC es de usuario y de la cuenta de gcloud; la VM 1 está `RUNNING` y sin candado; su script de arranque conoce el candado;
   esta Mac no tiene el compose local arriba; las claves de la sucursal 1 se leen de Secret Manager como `sa-sucursal-1`.
2. **No se pierde nada:** ninguna transacción preparada; cola de salida de la VM vacía **y** sus últimas 50 llaves en el inbox del
   central («N de N»); base en la última migración; ninguna existencia sin saldo inicial; ningún traslado `SOLICITADO` o
   `PREPARANDO` con la sucursal 1; ninguna transacción global sin finalizar.
3. Detiene la API de la VM y le quita el reinicio automático (desde aquí nadie vende en la VM). Si algo falla después, **la repone**.
4. `pg_dump` de la VM por IAP → `~/.sima-hibrido/traspaso-<fecha>.dump` (0600) → `pg_restore` en la Mac → **compara una firma de las
   dos bases** (documentos, movimientos, existencias, valor, outbox, usuarios, costos, migración): tienen que ser idénticas.
5. Pone el candado `nodo-externo=mac-<fecha>-<equipo>` en la VM, guarda la marca en `~/.sima-hibrido/traspaso.env` y **apaga la VM**.

Termina con `Resultado: N pasaron, 0 fallaron` y «La sucursal 1 ahora es ESTA MAC». Si falla antes del paso 5, la VM sigue siendo la
sucursal 1 y se puede repetir (`--reemplazar` si la Mac quedó con una restauración a medias).

### B3. Ensayo (el día antes)

```bash
infra/hibrido/preparar.sh
```

Es lo mismo que se corre el día de la demo (parte C). Ensayar también el corte: **apagar el Wi-Fi 2 minutos**, hacer 3 ventas en la
caja de Centro (siguen entrando), mirar la cola crecer; **prender el Wi-Fi** y cronometrar cuánto tarda la cola en vaciarse (el canal gRPC
de Pub/Sub puede tardar hasta un minuto en reconectar) y cuánto tarda en volver a abrirse la caja de Zona 10 (el túnel se reabre solo, ~20 s).
Al terminar: `infra/hibrido/terminar.sh`.

### B3b. Verificar en GCP el túnel inverso hacia Centro (lo que la prueba local NO cubre)

Desde 2026-11-03 la VM sucursal-1 **se enciende con el candado** (solo su base, sin API) y hace de puerta: `tuneles.sh` abre desde la Mac un
túnel SSH inverso por IAP y la VM reenvía su `10.10.0.11:3000` a la API de Centro (`docs/decisiones.md`, 2026-11-03). Se verifica contra GCP real
en este orden y con `preparar.sh` ya en «LISTO PARA LA DEMO»:

```bash
# 1. El sshd de COS permite el reenvío remoto (-R). preparar.sh ya lo sondea; si dijera «AllowTcpForwarding=no|local», la demo sigue con
#    Zona 10 ↔ Xela y la VM se vuelve a apagar sola (nada queda a medias).
gcloud compute ssh sucursal-1 --zone=us-central1-a --tunnel-through-iap --command="sudo /usr/sbin/sshd -T | grep -i allowtcpforwarding"   # yes (o all/remote)

# 2. Estado del túnel y de la VM: reenvio-centro corriendo, api-sucursal NO.
infra/hibrido/tuneles.sh estado 1        # «túnel 1 (inverso): OK … REENVIO_LISTO 10.10.0.11:3000 → 127.0.0.1:131xx»
gcloud compute ssh sucursal-1 --zone=us-central1-a --tunnel-through-iap --command="sudo docker ps --format '{{.Names}}'"   # db-sucursal y reenvio-centro; NO api-sucursal

# 3. El camino del coordinador: desde la VM 2 (misma subred que Cloud Run) a la IP de siempre de la sucursal 1.
gcloud compute ssh sucursal-2 --zone=us-central1-b --tunnel-through-iap --command="curl -s -m 5 http://10.10.0.11:3000/salud"   # "id_sucursal":1

# 4. El puerto 3000 no es público: sin IP externa y ninguna regla de red-sima abierta a 0.0.0.0/0 salvo el 22.
gcloud compute instances describe sucursal-1 --zone=us-central1-a --format='value(networkInterfaces[0].accessConfigs[0].natIP)'   # vacío
gcloud compute firewall-rules list --filter='network:red-sima' --format='table(name,direction,sourceRanges.list(),allowed[].map().firewall_rule().list())'
#    permitir-interno (10.10.0.0/24), permitir-ssh-iap (35.235.240.0/20, tcp:22), permitir-datastream-proxy (10.30.0.0/29, tcp:5432); ninguna corte-*.

# 5. Un traslado de verdad (Zona 10 → Centro): en la caja de Centro, encargado1 → Traslados → solicitar desde Sucursal Zona 10.
#    Debe pasar a COMPROMETIDO y luego «Recibir». En el central:
source infra/hibrido/comun.sh
sql_central "SELECT t.id_traslado, t.estado, tg.estado FROM traslado t LEFT JOIN transaccion_global tg ON tg.id_documento = t.id_documento ORDER BY t.fecha_solicitud DESC LIMIT 3"

# 6. Con el Wi-Fi de la Mac apagado, el central NO alcanza a Centro y falla dentro de su plazo. Se corre desde OTRO equipo (Cloud Shell):
#    la Mac sin Wi-Fi no tiene IAP.
gcloud compute ssh sucursal-2 --zone=us-central1-b --tunnel-through-iap --command="time curl -s -m 3 -o /dev/null -w '%{http_code}\n' http://10.10.0.11:3000/salud"   # 000 a los ~3 s
#    Al prender el Wi-Fi: medir cuántos segundos tarda el paso 3 en volver a dar "id_sucursal":1.
```

Antes del primer `preparar.sh` con el túnel, ver qué pasará con los traslados `SOLICITADO` que ya existen con la sucursal 1: **se ejecutan
solos** en cuanto el coordinador alcance a Centro (menos de un minuto). Un `SOLICITADO` no tiene transacción global ni nada preparado; el
origen vota con su existencia y su costo de **hoy** y, si no alcanza, vota NO y el traslado se aborta limpio.

```bash
source infra/hibrido/comun.sh
sql_central "SELECT t.id_traslado, t.estado, t.id_sucursal_origen, t.id_sucursal_destino, t.fecha_solicitud, tg.estado AS tx FROM traslado t LEFT JOIN transaccion_global tg ON tg.id_documento = t.id_documento WHERE t.estado = 'SOLICITADO' AND 1 IN (t.id_sucursal_origen, t.id_sucursal_destino)"
```

### B4. Regreso a la VM (después de la revisión de Cloud del 4 nov)

```bash
infra/hibrido/regreso.sh            # --forzar solo si se decide a sabiendas, ver abajo
```

1. Comprueba que esta Mac es la del traspaso (el candado de la VM es el de la marca) y que GCP está encendido (`encender-hibrido`). La VM
   puede estar `RUNNING` (con el candado, como puerta del túnel) o `TERMINATED`; si está `RUNNING`, se niega si `api-sucursal` corre en ella.
2. **Se niega si hay traslados `SOLICITADO` con la sucursal 1**: los lista con su id y su fecha. Un traslado `SOLICITADO` espera a que
   el coordinador alcance a las dos sucursales; en cuanto la sucursal 1 vuelve a ser alcanzable (túnel abierto o VM de regreso), la
   recuperación del central lo retoma y lo **ejecuta** sola. Con `--forzar` sigue sabiendo eso.
3. Espera a que la cola de salida de la Mac se vacíe.
4. **Cierra el túnel 1 y borra el reenviador de la VM**, detiene la API de la Mac, respalda su base en `~/sima-respaldos/` y calcula la firma.
5. Enciende la VM **con el candado** (solo su base; si ya estaba encendida no la toca), comprueba que la API y el reenviador de la VM no están
   corriendo, **respalda la base vieja de la VM en la propia VM** (`/var/lib/sima/antes-regreso.dump`), restaura la de la Mac y compara las firmas.
6. Quita el candado y reinicia la VM: arranca como la sucursal 1 de siempre, con todo lo vendido en la Mac. Comprueba `/salud`.

El volumen de la Mac **no se borra**: queda como respaldo hasta que se borre a propósito. Después: `infra/gcp/verificar-gcp.sh` en Cloud Shell
(0 fallaron).

### B5. Si la Mac se pierde antes del regreso

Lo único al día es el último volcado de `~/sima-respaldos/` (o el del traspaso en `~/.sima-hibrido/`); se pierde lo vendido después.
Con ese archivo, desde cualquier equipo con `gcloud`:

```bash
gcloud compute instances start sucursal-1 --zone=us-central1-a            # con el candado: solo su base, sin API
gcloud compute ssh sucursal-1 --zone=us-central1-a --tunnel-through-iap \
  --command="sudo docker exec -i db-sucursal sh -c 'dropdb -U sima --if-exists sima_sucursal_1 && createdb -U sima sima_sucursal_1 && pg_restore -U sima -d sima_sucursal_1 --no-owner --exit-on-error'" < centro-AAAAMMDD-HHMMSS-....dump
gcloud compute instances remove-metadata sucursal-1 --zone=us-central1-a --keys=nodo-externo
gcloud compute instances reset sucursal-1 --zone=us-central1-a
```

---

## Parte C — El día de la demo

Un solo comando antes de presentar, y otro al terminar. **Durante la demo no se corre nada**: corte = apagar el Wi-Fi; reconexión = prenderlo.

```bash
infra/hibrido/preparar.sh          # 3 a 8 minutos si GCP estaba apagado
```

En orden, y se detiene en lo que no puede seguir: cuenta, ADC de usuario (misma cuenta que gcloud, con proyecto de cuota) · **la VM sucursal-1
tiene el candado, es el del traspaso de esta Mac y, si ya está encendida, `api-sucursal` NO corre en ella** · `encender-apagar.sh encender-hibrido`
(todo, y la VM 1 solo con el candado: su base, sin API) · la VM 1 responde y su API sigue sin correr ·
api-central responde · claves de la sucursal 1 de Secret Manager (archivo 0600 fuera del repo) · stack local (solo `db-sucursal-1`,
`api-sucursal-1`, `acceso-sucursal-1` y `reloj-docker`; la base de Centro es la del traspaso y está en la última migración) · reloj
(Docker↔Mac y Mac↔Google; avisa pasados 5 s, falla pasados 30 s) · túneles a las cajas 2 y 3 · **el túnel inverso hacia Centro** (sondea que el
sshd de la VM permita `-R`; si no, lo dice, apaga la VM y la demo sigue sin traslados con Centro, sin contar como falla) y la comprobación de punta
a punta desde la VM 2 a `10.10.0.11:3000` · salud de las tres sucursales (Pub/Sub,
sin errores de firma/token/permisos, réplica al día, cola de Centro vacía) · saldo inicial y **auditoría vacía** en las tres · copia
de la red poblada y reciente · traslados `SOLICITADO` con la sucursal 1 (aviso) y mensajes muertos (0) · edad del último respaldo
(aviso pasadas 24 h) · y otra vez que la VM 1 sigue con candado y sin API (y con el reenviador, si hay túnel).

Termina con las tres URL de caja y **«LISTO PARA LA DEMO»**, o con la lista de lo que falta (y sale con 1):

```
Caja de Centro      http://localhost:3001/caja/     (en esta Mac)
Caja de Zona 10     http://localhost:3002/caja/     (GCP, por túnel)
Caja de Xela        http://localhost:3003/caja/     (GCP, por túnel)
```

```bash
infra/hibrido/terminar.sh          # guarda el respaldo, baja el stack SIN borrar volúmenes, cierra los túneles y apaga GCP
```

Antes de bajar nada guarda un volcado verificado de la base de Centro en `~/sima-respaldos/` (si falla, **no baja nada**), espera a que
la cola se vacíe, lista los traslados `SOLICITADO` con la sucursal 1, cierra el túnel inverso y borra el reenviador de la VM, y al final **verifica** que las tres VMs y el proxy están
`TERMINATED`, Cloud SQL en `NEVER`, el stream `PAUSED` y el job de recuperación del 2PC `PAUSED`. Con `--sin-apagar-gcp` deja GCP
encendido (segundo ensayo); con `--solo-respaldo` solo guarda el volcado.

**Si falla el Wi-Fi o GCP el día de la demo (plan B):** la demo completa en local, `docker compose up -d`, con el corte por
`scripts/corte-enlace/enlace-compose.sh` (`source scripts/corte-enlace/enlace-compose.sh; cortar_enlace 1`). Antes hay que bajar el stack
híbrido sin borrar volúmenes (`infra/hibrido/terminar.sh --sin-apagar-gcp`); la base local de ese compose es otra, con datos de
desarrollo.

---

# Procedimiento base (todo lo que entró desde `28d0382`)

## 0. Antes de la ventana

### 0.1 Ayudantes de la sesión de Cloud Shell

Se pegan una vez en la terminal; los usan todos los pasos. Ninguno imprime una clave ni una
contraseña.

```bash
cd ~/sima
source infra/gcp/variables.sh
ZONAS=(us-central1-a us-central1-b us-central1-c)
zona() { echo "${ZONAS[$(($1 - 1))]}"; }
en_vm() { gcloud compute ssh "sucursal-$1" --zone="$(zona "$1")" --tunnel-through-iap --quiet --command="$2"; }
# psql en la base de la sucursal N; la consulta va por stdin
sql_sucursal() { en_vm "$1" "sudo docker exec -i db-sucursal psql -U sima -d sima_sucursal_$1 -tA -v ON_ERROR_STOP=1" <<<"$2"; }
# psql en Cloud SQL desde la VM de la sucursal 2 (misma VPC). La cadena de conexión viaja por stdin,
# no en la línea de comandos de ssh; la consulta va detrás.
sql_central() {
  { gcloud secrets versions access latest --secret=sima-bd-url-central; echo; printf '%s\n' "$1"; } \
    | en_vm 2 "IFS= read -r U; sudo HOME=/var/lib/sima docker run --rm -i $IMAGEN_POSTGRES psql \"\$U\" -tA -v ON_ERROR_STOP=1"
}
# Mensajes sin entregar en la suscripción push (métrica de Monitoring, con unos minutos de retraso)
pendientes_pubsub() {
  curl -s -G -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://monitoring.googleapis.com/v3/projects/$PROYECTO/timeSeries" \
    --data-urlencode 'filter=metric.type="pubsub.googleapis.com/subscription/num_undelivered_messages" AND resource.labels.subscription_id="sima-sincronizacion-push"' \
    --data-urlencode "interval.startTime=$(date -u -d '-15 min' +%Y-%m-%dT%H:%M:%SZ)" \
    --data-urlencode "interval.endTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    | jq -r '.timeSeries[0].points[0] | "\(.value.int64Value // 0) sin entregar (medido \(.interval.endTime))"'
}
```

### 0.2 Línea base de Terraform

Con Cloud Shell **todavía en el commit anterior a este despliegue** (sin esta rama):

```bash
cd ~/sima && git log -1 --oneline
cd infra/terraform && source ./credenciales.sh
./revisar-plan.sh
```

**Tiene que salir 0.** Con Cloud SQL apagado puede aparecer el cambio conocido de
`google_sql_user.sima`, pero aquí está encendido. Si sale otra cosa, hay una diferencia anterior
que se resuelve primero.

Después:

```bash
cd ~/sima && git pull && git log -1 --oneline    # el commit con este despliegue
```

### 0.3 Anotar el estado actual (para volver atrás)

```bash
REV_ANTERIOR="$(gcloud run services describe api-central --region=us-central1 --format='value(status.latestReadyRevisionName)')"
DIGEST_API="$(gcloud artifacts docker images describe "$IMAGEN:latest" --format='value(image_summary.digest)')"
DIGEST_MIG="$(gcloud artifacts docker images describe "$IMAGEN_MIGRACIONES:latest" --format='value(image_summary.digest)')"
printf 'REV_ANTERIOR=%s\nDIGEST_API=%s\nDIGEST_MIG=%s\n' "$REV_ANTERIOR" "$DIGEST_API" "$DIGEST_MIG" | tee ~/antes-despliegue.txt
```

`~/antes-despliegue.txt` queda en el home de Cloud Shell, fuera del repositorio.

### 0.4 Las VMs tienen lo que usa el arranque nuevo, y espacio

```bash
for N in 1 2 3; do en_vm "$N" 'command -v base64 sed tr curl; df -h /var | tail -1'; done
```

**Tienen que aparecer las cuatro rutas** en cada VM. La imagen nueva pesa unos 1,4 GB: tiene
que haber al menos 3 GB libres en `/var`. Si no, `sudo docker image prune -a` en esa VM; borra
las imágenes que no usa ningún contenedor.

### 0.5 BigQuery: columnas que pasan a admitir NULL

```bash
for c in "public_traslado id_usuario_destino" "public_documento id_usuario"; do
  set -- $c
  bq show --format=prettyjson --schema "$PROYECTO:sima_cdc.$1" 2>/dev/null \
    | jq -r --arg n "$2" '.[] | select(.name == $n) | "\($n): \(.mode // "NULLABLE")"'
done
```

**Tiene que decir `NULLABLE` (o nada, si la tabla no existe).** Si alguna dice `REQUIRED`, se
relaja antes de migrar. Es un cambio permitido en BigQuery y no reescribe datos:

```bash
T=public_traslado; C=id_usuario_destino     # o public_documento / id_usuario
bq show --format=json --schema "$PROYECTO:sima_cdc.$T" \
  | jq --arg c "$C" 'map(if .name == $c then .mode = "NULLABLE" else . end)' > /tmp/esquema.json
bq update "$PROYECTO:sima_cdc.$T" /tmp/esquema.json && rm /tmp/esquema.json
```

¿Por qué? Después de las migraciones, el central escribe NULL en esas columnas: la primera porque
se elimina, la segunda en el TRASLADO del origen. Una columna `REQUIRED` haría fallar el stream.

### 0.6 Apply A: cuentas, secretos y permisos (sin efecto en lo que corre)

Solo recursos nuevos. Las VMs y api-central no se tocan todavía, por eso el plan va con
`-target`:

```bash
cd ~/sima/infra/terraform && source ./credenciales.sh
./revisar-plan.sh \
  -target=google_service_account.cuenta \
  -target=google_project_iam_member.permiso \
  -target=google_secret_manager_secret.secreto \
  -target=google_secret_manager_secret_iam_member.lector \
  -target=google_artifact_registry_repository_iam_member.sucursal_n_lee_imagenes \
  -target=google_pubsub_topic_iam_member.sucursal_n_publica
```

**Tiene que salir 2, con exactamente 30 `+ crear` y nada más:**

| Recursos | Cantidad |
|---|---:|
| `google_service_account.cuenta["sa-sucursal-1".."3"]` | 3 |
| `google_project_iam_member.permiso["sucursal-N-logs"]` y `["sucursal-N-metricas"]` | 6 |
| `google_secret_manager_secret.secreto["sima-sesion-clave-sN"]` y `["sima-servicio-clave-sN"]` | 6 |
| `google_secret_manager_secret_iam_member.lector["sesion-sN"]`, `["servicio-sN"]` y `["servicio-sN-central"]` | 9 |
| `google_artifact_registry_repository_iam_member.sucursal_n_lee_imagenes["1".."3"]` | 3 |
| `google_pubsub_topic_iam_member.sucursal_n_publica["1".."3"]` | 3 |

Terraform avisa que `-target` no es para uso habitual; aquí es a propósito.

```bash
terraform apply plan.bin
cd ~/sima && infra/gcp/12-claves.sh
```

**`12-claves.sh` tiene que imprimir** seis "versión nueva" y al final "Las 6 claves existen,
tienen 32 bytes o más y son todas distintas".

Comprobar quién lee cada clave:

```bash
for s in sima-sesion-clave-s2 sima-servicio-clave-s2; do
  echo "$s:"; gcloud secrets get-iam-policy "$s" --format=json | jq -r '.bindings[]?.members[]'
done
```

- `sima-sesion-clave-s2`: **solo** `sa-sucursal-2@…`;
- `sima-servicio-clave-s2`: `sa-sucursal-2@…` y `sa-api-central@…`.

Hasta aquí nada de lo que corre cambió. Se puede parar sin consecuencias.

---

## 1. Comprobaciones previas (con todo encendido, antes de detener nada)

### 1.1 Avisar

Desde ahora no se vende ni se traslada en ninguna sucursal.

### 1.2 El central: cada consulta tiene que dar 0

```bash
# 013: todo traslado tiene el usuario de su documento de coordinador (se vuelve id_usuario_solicitante)
sql_central "SELECT COUNT(*) FROM traslado t LEFT JOIN documento d ON d.id_documento = t.id_documento WHERE d.id_usuario IS NULL;"
# 017: ningún documento viola el CHECK nuevo de recepción
sql_central "SELECT COUNT(*) FROM documento WHERE NOT (CASE WHEN tipo = 'RECEPCION' THEN (id_traslado IS NULL) = (id_proveedor IS NOT NULL) ELSE id_proveedor IS NULL END);"
# 011: todo usuario tiene correo (de ahí sale nombre_usuario)
sql_central "SELECT COUNT(*) FROM usuario WHERE correo IS NULL;"
# Nada del 2PC abierto
sql_central "SELECT COUNT(*) FROM traslado WHERE estado IN ('SOLICITADO', 'PREPARANDO');"
sql_central "SELECT COUNT(*) FROM transaccion_global WHERE estado <> 'FINALIZADO';"
```

Y estas dos se miran:

```bash
sql_central "SELECT max(nombre) FROM migracion_aplicada;"       # 010_indices_consultas.sql
sql_central "SELECT id_usuario, correo, hash_contrasena = 'SIN_HASH_HITO_1' AS sin_contrasena FROM usuario ORDER BY 1;"
```

- **Última migración:** tiene que ser `010_indices_consultas.sql`.
- **Usuarios:** los ids 1 a 9, con `sin_contrasena = t`. Un usuario fuera de esos nueve se queda
  sin contraseña después del despliegue, y un ADMIN se la pone desde el panel de empleados.
- **Si hay traslados o transacciones abiertos:** la recuperación del central (Cloud Scheduler,
  cada minuto) y el reconciliador de cada sucursal los cierran solos. Se espera un par de
  minutos y se repite. No se resuelven a mano.

### 1.3 Cada sucursal: cada consulta tiene que dar 0

```bash
for N in 1 2 3; do
  echo "== sucursal $N"
  # 012: sin transacciones preparadas (sobreviven al reinicio de PostgreSQL)
  sql_sucursal "$N" "SELECT COUNT(*) FROM pg_prepared_xacts WHERE database = current_database();"
  # 012: todo TRASLADO viejo dice "sucursal N" en referencia (de ahí sale la contraparte)
  sql_sucursal "$N" "SELECT COUNT(*) FROM documento WHERE tipo = 'TRASLADO' AND id_traslado IS NOT NULL AND substring(referencia FROM 'sucursal ([0-9]+)') IS NULL;"
  # 015: ningún documento viola el CHECK de recepción
  sql_sucursal "$N" "SELECT COUNT(*) FROM documento WHERE NOT (CASE WHEN tipo = 'RECEPCION' THEN (id_traslado IS NULL) = (id_proveedor IS NOT NULL) ELSE id_proveedor IS NULL END);"
  # 010: todo usuario tiene correo
  sql_sucursal "$N" "SELECT COUNT(*) FROM usuario WHERE correo IS NULL;"
  # outbox: nada pendiente ni fallido
  sql_sucursal "$N" "SELECT COUNT(*) FROM outbox WHERE estado <> 'ENVIADO';"
  # (se mira) última migración: 009_indices_consultas.sql
  sql_sucursal "$N" "SELECT max(nombre) FROM migracion_aplicada;"
done
```

Las cinco primeras de cada sucursal **tienen que dar 0**, y la última,
`009_indices_consultas.sql`.

Si hay transacciones preparadas, se espera a que el reconciliador las resuelva (las de más de
30 s, cada 5 s) y se repite. Si hay outbox pendiente, se espera a que salga.

### 1.4 Pub/Sub: todo lo publicado ya llegó al central

Un mensaje viejo (sin firma) que quede en Pub/Sub cuando cambie el central **nunca se va a
aceptar**:
- el central nuevo lo rechaza con 401;
- termina en el tema de muertos;
- `muertos.mjs reenviar` no lo arregla, porque lo vuelve a publicar con los mismos atributos,
  sin firma.

Por eso todo lo publicado tiene que estar aplicado antes de seguir. Se comprueba de tres formas.

**a) Las últimas 50 llaves de cada outbox están en el inbox del central.**

```bash
for N in 1 2 3; do
  LLAVES="$(sql_sucursal "$N" "SELECT string_agg(quote_literal(id_idempotencia::text), ',') FROM (SELECT id_idempotencia FROM outbox WHERE estado = 'ENVIADO' ORDER BY id_outbox DESC LIMIT 50) x;")"
  ESPERADAS="$(sql_sucursal "$N" "SELECT COUNT(*) FROM (SELECT 1 FROM outbox WHERE estado = 'ENVIADO' ORDER BY id_outbox DESC LIMIT 50) x;")"
  if [ -z "$LLAVES" ]; then echo "sucursal $N: nada publicado"; continue; fi
  echo "sucursal $N: $(sql_central "SELECT COUNT(*) FROM inbox_idempotencia WHERE id_idempotencia::text IN ($LLAVES);") de $ESPERADAS en el central"
done
```

**Tiene que decir "50 de 50"**, o "N de N" si hay menos de 50.

**b) Nada sin entregar en la suscripción push.**

```bash
pendientes_pubsub
```

**Tiene que decir "0 sin entregar".** La métrica llega con unos minutos de retraso: hay que
fijarse en que la hora medida sea posterior al último envío.

**c) Nada en muertos.**

```bash
node scripts/pubsub/muertos.mjs listar
```

**Tiene que listar 0.**

### 1.5 Respaldo lógico de cada sucursal (en su propio disco)

```bash
for N in 1 2 3; do
  en_vm "$N" "sudo sh -c 'docker exec db-sucursal pg_dump -U sima -Fc sima_sucursal_$N > /var/lib/sima/antes-despliegue.dump && chmod 600 /var/lib/sima/antes-despliegue.dump && ls -l /var/lib/sima/antes-despliegue.dump'"
done
```

Cada archivo tiene que pesar más de 0 bytes. Contiene la base, con los hashes: se queda en la VM.

---

## 2. Detener las sucursales y respaldar

### 2.1 Detener las tres VMs

Así no publican nada sin firma mientras cambia el central.

```bash
gcloud compute instances stop sucursal-1 --zone=us-central1-a
gcloud compute instances stop sucursal-2 --zone=us-central1-b
gcloud compute instances stop sucursal-3 --zone=us-central1-c
gcloud compute instances list --filter='name~^sucursal-' --format='value(name,status)'    # las tres TERMINATED
```

Desde aquí, `sql_sucursal` y `sql_central` no funcionan: usan las VMs.

### 2.2 Pub/Sub, otra vez, con las VMs detenidas

Pasados unos 3 minutos:

```bash
pendientes_pubsub          # 0 sin entregar, medido después de la detención
node scripts/pubsub/muertos.mjs listar   # 0
```

### 2.3 Snapshots de los discos (VMs detenidas: consistentes)

```bash
for N in 1 2 3; do
  DISCO="$(gcloud compute instances describe "sucursal-$N" --zone="$(zona "$N")" --format='value(disks[0].source.basename())')"
  gcloud compute disks snapshot "$DISCO" --zone="$(zona "$N")" --snapshot-names="sucursal-$N-antes-despliegue-$(date +%Y%m%d)"
done
gcloud compute snapshots list --filter='name~antes-despliegue' --format='value(name,status,diskSizeGb)'   # tres READY
```

### 2.4 Respaldo de Cloud SQL

```bash
gcloud sql backups create --instance=sima-central --description="antes-despliegue-$(date +%Y%m%d)"
gcloud sql backups list --instance=sima-central --limit=3 --format='value(id,status,description)'
```

**Tiene que quedar `SUCCESSFUL`.** Anotar el id en `~/antes-despliegue.txt`.

---

## 3. El central

### 3.1 Imágenes nuevas

Se construyen **dentro** de la ventana. Cualquier VM que arranque baja `:latest`: si las
imágenes nuevas existieran antes, un reinicio no planeado tomaría el código nuevo sin su
cuenta ni sus claves.

```bash
cd ~/sima && infra/gcp/04-imagenes.sh
```

**Tiene que terminar con "Imágenes publicadas con la etiqueta `<sha>`".** Anotar la etiqueta.

### 3.2 Migraciones del central ⚠️ PUNTO DE NO RETORNO

- `013` elimina `traslado.id_usuario_destino`, y no hay migraciones de bajada.
- Desde aquí, volver atrás es restaurar el respaldo de 2.4 (parte 6).
- Antes de correrlo, confirmar que 1.2, 1.3 y 1.4 dieron lo esperado.

```bash
gcloud run jobs execute migrar-central --region=us-central1 --wait
gcloud logging read 'resource.type="cloud_run_job" AND resource.labels.job_name="migrar-central"' --freshness=15m \
  --format='value(textPayload)' | grep -E 'aplicada|Migraciones|Semilla|ERROR' | tac
```

**Tiene que mostrar:**
- `aplicada 011_replica.sql` … `aplicada 019_costo_promedio.sql`;
- `Migraciones de central: 9 aplicadas, 10 ya estaban`;
- `Semilla: … usuarios al día (0 sin contraseña)`;
- `Lista de costos: 50 vigencias sembradas en el central` (el catálogo ya estaba sembrado: `aplicar.sh` agrega solo
  la lista de costos, que es lo que baja a las sucursales por la réplica y de donde sale el saldo inicial de 4.3 bis).

Si una migración falla:
- queda aplicado todo hasta la anterior, porque cada archivo es atómico;
- **no se vuelve atrás**: se corrige la causa y se vuelve a correr el job;
- las comprobaciones de 1.2 son justamente las causas posibles.

### 3.3 Apply B: api-central con sus claves (el código nuevo entra aquí)

Este apply crea una revisión nueva con la imagen `…/sima:latest`, que desde 3.1 es la nueva.

```bash
cd ~/sima/infra/terraform && source ./credenciales.sh
./revisar-plan.sh \
  -target=google_cloud_run_v2_service.api_central \
  -target=google_cloud_run_v2_service_iam_member.invoca_central
```

**Tiene que salir 2, con solo esto:**
- `~ cambiar google_cloud_run_v2_service.api_central: template` (las tres `SERVICIO_CLAVE_SN`);
- `+ crear google_cloud_run_v2_service_iam_member.invoca_central["sucursal-1".."3"]`.

```bash
terraform apply plan.bin
gcloud run services describe api-central --region=us-central1 --format='value(status.latestReadyRevisionName,status.traffic[0].revisionName)'
```

**La revisión lista y la del tráfico tienen que ser la nueva**, distinta de `REV_ANTERIOR`.

Si la revisión no arranca (falta una clave o no la puede leer):
- el apply falla y Cloud Run deja el tráfico en la anterior;
- el motivo está en `gcloud run revisions describe <revisión> --region=us-central1`.

Verificar el central:

```bash
URL="$(gcloud run services describe api-central --region=us-central1 --format='value(status.url)')"
TOKEN="$(gcloud auth print-identity-token)"
curl -s -H "Authorization: Bearer $TOKEN" "$URL/salud" | jq '{estado, base_datos}'           # "OK", "OK"
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $TOKEN" "$URL/existencias/7501000000001"   # 401: sin firma
SERVICIO_CLAVE_S1="$(gcloud secrets versions access latest --secret=sima-servicio-clave-s1)" TOKEN_IDENTIDAD="$TOKEN" \
  node scripts/lib/firma-servicio.mjs pedir 1 GET "$URL/existencias/7501000000001" | tail -1          # 200: firmada
gcloud logging read 'resource.type="cloud_run_revision" AND resource.labels.service_name="api-central" AND httpRequest.requestUrl:"/traslados/recuperacion/ejecutar"' \
  --freshness=5m --limit=3 --format='value(httpRequest.status)'                                 # 200 (Cloud Scheduler)
```

Opcional, para que la revisión quede atada a la etiqueta y no a `latest`:

```bash
gcloud run services update api-central --region=us-central1 --image="$IMAGEN:<etiqueta de 3.1>"
```

---

## 4. Las sucursales

### 4.1 Apply C: cuenta propia y arranque nuevo (VMs detenidas)

```bash
cd ~/sima/infra/terraform && source ./credenciales.sh
./revisar-plan.sh
```

**(2026-11-02) Con esta rama el plan completo trae además tres cosas que no son de este paso.** Se aplica con `-target` para
dejar fuera la consulta programada de BigQuery (va en A10, cuando las columnas de costo ya existen en `sima_cdc`):

```bash
./revisar-plan.sh \
  -target=google_project_service.api \
  -target=google_service_account_iam_member.operador_suplanta_sucursal_1 \
  -target=google_compute_instance.sucursal
```

**Tiene que salir 1, con solo `~ cambiar google_compute_instance.sucursal["1"]`, `["2"]` y
`["3"]`, más `+ crear` de `google_project_service.api["iamcredentials.googleapis.com"]` y de
`google_service_account_iam_member.operador_suplanta_sucursal_1` (a TU usuario, solo sobre `sa-sucursal-1`).** Los atributos que cambian en las VMs son:
- `service_account`;
- `metadata` (`startup-script`, `secreto-sesion`, `secreto-servicio`);
- `allow_stopping_for_update`.

Antes de aplicar, se comprueba que es un cambio en el lugar y no un reemplazo:

```bash
grep -E "will be updated in-place|must be replaced|forces replacement" plan.log
```

**Tiene que haber tres "will be updated in-place" y ningún "must be replaced" ni "forces
replacement".**
- Un reemplazo perdería la base de la sucursal.
- `prevent_destroy` lo impediría con error, pero si apareciera, **se para**.

Este cambio está aprobado: es la salida 1 esperada de este paso.

```bash
terraform apply plan.bin
gcloud compute instances list --filter='name~^sucursal-' --format='value(name,status,serviceAccounts[0].email)'
```

**Las tres tienen que estar `TERMINATED`**, cada una con `sa-sucursal-N@…`. El apply no las
enciende porque ya estaban detenidas; si alguna quedó `RUNNING`, se sigue igual: su arranque ya
es el nuevo.

### 4.2 Encender la sucursal 1 y mirar su arranque

```bash
gcloud compute instances start sucursal-1 --zone=us-central1-a
sleep 90
gcloud compute instances get-serial-port-output sucursal-1 --zone=us-central1-a 2>/dev/null \
  | grep -E 'Clave |AVISO clave|ERROR clave|aplicada|Migraciones de|Semilla|Sucursal 1 lista|rror' | tail -25
```

**Tiene que mostrar, en este orden:**
1. `Clave sesion: leída de Secret Manager (sima-sesion-clave-s1)` y lo mismo para `servicio`;
2. `aplicada 010_replica.sql` … `aplicada 018_costo_promedio.sql`;
3. `Migraciones de sucursal: 9 aplicadas, 9 ya estaban`;
4. `Semilla: … usuarios al día (0 sin contraseña)`;
5. `Sucursal 1 lista`.

Si el arranque tarda, se repite el `get-serial-port-output`.

Cómo leer los errores:
- **"ERROR clave":** la cuenta no puede leer el secreto (revisar el lector de 0.6) o el nombre
  en los metadatos está mal.
- **Una migración falla:** el arranque se corta y queda corriendo el contenedor viejo. Ver la
  parte 6.

En la VM:

```bash
en_vm 1 'curl -s localhost:3000/salud' | jq '{estado, transporte: .sincronizacion.transporte, enlace: .sincronizacion.enlace, error_sinc: .sincronizacion.ultimo_error, replica: {enlace: .replica.enlace, version: .replica.ultima_version, en: .replica.ultima_replica_en, error: .replica.ultimo_error}}'
```

**Tiene que dar** `estado: "OK"`, `transporte: "PUBSUB"`, `replica.enlace: "OK"` y una
`ultima_replica_en` de hace menos de un minuto. No tiene que haber ningún error con `FIRMA`,
`TOKEN` ni `PERMISSION`.

### 4.3 Encender las sucursales 2 y 3

Los mismos comandos de 4.2, con `sucursal-2` (`us-central1-b`) y `sucursal-3`
(`us-central1-c`). Una por vez, mirando el arranque de cada una.

### 4.3 bis Saldo inicial valorado ⚠️ OBLIGATORIO antes de reabrir las ventas

La migración `018` de la sucursal agrega `existencia.valor_inventario`, que queda **NULL** para todo lo que ya
existía: esas unidades entraron antes de llevar costo y el sistema no inventa un costo en la migración. Una
existencia sin valor **bloquea** las ventas, recepciones, ajustes y traslados del producto (409
`SIN_SALDO_INICIAL`): la caja le muestra al encargado un aviso claro. Por eso este paso va **dentro de la
ventana**, antes de reabrir las ventas. Se hace en cada sucursal, una vez que la lista de costos ya bajó por la
réplica (el saldo inicial valúa lo que hay **al costo de lista vigente**).

1. La lista existe en el central y llegó a cada sucursal (50 vigencias en la semilla; tiene que dar lo mismo en las
   cuatro bases):

```bash
sql_central "SELECT COUNT(*) FROM costo_proveedor"
for n in 1 2 3; do sql_sucursal $n "SELECT COUNT(*) FROM costo_proveedor"; done
```

Si una sucursal da menos, esperar la siguiente ronda de réplica (hasta ~10 s) y repetir. **Si sigue dando menos** (típicamente 0), el cursor de
réplica de esa sucursal ya pasó las versiones de la lista de costos mientras corría el código viejo: **A7 bis** (comparar y regresar el cursor). Si el **central** da 0, la
lista no se sembró: volver a correr el job de 3.2 con `SEMBRAR=1` (`aplicar.sh` la agrega si está vacía).

2. Registrar el saldo inicial en cada sucursal, con el ADMIN de esa sucursal (`admin1`, `admin2`, `admin3`):

```bash
for n in 1 2 3; do
  sql_sucursal $n "SELECT * FROM registrar_saldo_inicial((SELECT id_usuario FROM usuario WHERE rol = 'ADMIN' AND activo ORDER BY id_usuario LIMIT 1))"
done
```

**Tiene que dar 0 filas** en las tres: la función devuelve solo los productos que **no** pudo valuar (sin costo de
lista; `motivo = SIN_COSTO_VIGENTE`). Si devuelve alguno, un ADMIN registra la vigencia de costo de ese producto
(`POST /red/costos`, pantalla de Productos), se espera la réplica y se vuelve a correr.

La función es **idempotente**: solo toca las existencias con `valor_inventario IS NULL`; correrla dos veces no
duplica documentos, movimientos ni avisos. Por cada sucursal crea **un** documento AJUSTE «Saldo inicial al costo de
lista», un movimiento `SALDO_INICIAL` por producto (que arranca el cárdex valorizado; no mueve unidades) y su fila de
`outbox`, que viaja al central. Lo anterior queda **explícitamente sin costo**: el histórico no se reescribe.

3. Comprobar que no quedó nada sin valor y que el cárdex cuadra:

```bash
for n in 1 2 3; do sql_sucursal $n "SELECT COUNT(*) FROM existencia WHERE valor_inventario IS NULL"; done   # 0, 0, 0
curl -s -b <cookie de encargado1> https://<sucursal-1>/auditoria/kardex | jq '{cuadra, discrepancias}'     # cuadra: true
```

`infra/gcp/verificar-gcp.sh` (4.4) **falla** si queda alguna existencia sin saldo inicial o algún producto activo sin
costo en la lista.

### 4.4 Verificación completa

```bash
cd ~/sima && infra/gcp/verificar-gcp.sh
```

Corre todo, incluido el corte de enlace de la sucursal 1. **Tiene que terminar con
"0 fallaron".**

| Sección | Qué comprueba |
|---|---|
| 2 | El central exige la firma; tiene las tres claves; Cloud Scheduler sigue respondiendo bien |
| 3 | Cada sucursal tiene su cuenta, sin errores de firma; réplica al día; `cajeroN` inicia sesión |
| 4 | Una venta con sesión llega al central |
| 4b | Login y roles; réplica (y la de otra sucursal, 403); un cambio de catálogo que llega a las tres y se deshace; idempotencia de ventas (repetida y 409); recepción de proveedor que llega al central; traslado 1 → 2 pedido por el destino, `COMPROMETIDO`, `FINALIZADO` y recibido, con `traslado_local` correcto; confirmar sin firma da 401 |
| 4c | (2026-11-02) Costo promedio: lista de costos idéntica en el central y las tres sucursales, venta valorada con cárdex encadenado, el valor llega al consolidado, saldo inicial en cada sucursal |
| 5 | Durante el corte, login y venta; al restaurar, la bandeja se vacía y la réplica vuelve. (2026-11-02) Con el corte puesto, las existencias de la red salen de la copia local (`COPIA_LOCAL`, con antigüedad) y la fila propia está al día |
| 5b | (2026-11-02) La auditoría del cárdex viene vacía (`cuadra: true`) en las tres sucursales |

Los fallos de las secciones 6 a 8 (Datastream, BigQuery, alertas) no bloquean el fin de la
ventana: se ven en la parte 5. (2026-11-02) En este despliegue la analítica se verifica aparte, con `--sin-analitica` ahora y
`--solo-analitica` en A11.

Revisión adicional de los registros del central:

```bash
gcloud logging read 'resource.type="cloud_run_revision" AND resource.labels.service_name="api-central" AND httpRequest.status>=401 AND httpRequest.status<=403' \
  --freshness=30m --format='value(timestamp,httpRequest.status,httpRequest.requestUrl)'
```

Solo tienen que aparecer los rechazos que provoca `verificar-gcp.sh` a propósito:
- `/existencias` sin firma (401);
- `/replica/cambios?id_sucursal=2` firmada como la 1 (403).

Ninguno de `/sincronizacion/mensajes` ni de `/replica/cambios` con la sucursal que corresponde.

### 4.5 La caja, a mano

(2026-11-02) El firewall de la VPC solo deja pasar a IAP por el **puerto 22** (`permitir-ssh-iap`, `infra/terraform/red.tf`), así que
`gcloud compute start-iap-tunnel sucursal-1 3000` **no funciona** (la conexión al 3000 nunca llega). El túnel va por SSH:

```bash
gcloud compute ssh sucursal-1 --zone=us-central1-a --tunnel-through-iap -- -N -L 3001:localhost:3000
```

En la vista previa web de Cloud Shell (puerto 3001) se abre `/caja/` y se prueba:
- login con `cajero1`, una venta y el ticket;
- con `encargado1`: Ventas y el reporte del día.

**Fin de la ventana:** se avisa que se puede vender. Desde aquí, solo se arregla hacia adelante.

---

## 5. Después de la ventana

### 5.1 Datastream y BigQuery

`proveedor.activo` se agregó **después** de que `011` actualizara todas las filas de
`proveedor`. Para que Datastream vea la columna nueva hace falta un evento posterior en esa
tabla.

Se fuerza con una actualización que no cambia nada. Sí sube la `version_replica`, y la réplica
la propaga sin efecto visible.

```bash
sql_central "UPDATE proveedor SET activo = activo;"
```

Pasados unos 15 minutos (el `data_freshness` del stream):

```bash
infra/gcp/verificar-gcp.sh --solo-analitica       # stream RUNNING, sin errores, slot sin lag
bq show --format=prettyjson --schema "$PROYECTO:sima_cdc.public_proveedor" | jq -r '.[].name' | grep -E '^(activo|version_replica)$'
bq show --format=prettyjson --schema "$PROYECTO:sima_cdc.public_traslado"  | jq -r '.[].name' | grep -E '^(id_usuario_solicitante|id_idempotencia|id_usuario_destino)$'
```

Lo que tiene que aparecer:
- **en `public_proveedor`:** `activo` y `version_replica`;
- **en `public_traslado`:** `id_usuario_solicitante` (por el traslado de 4.4) y
  `id_idempotencia`. `id_usuario_destino` sigue ahí: Datastream no borra columnas de BigQuery.

Si el stream tiene errores:

```bash
gcloud datastream streams describe sima-central-cdc --location=us-central1 --format='value(errors)'
```

- **Lo más probable es una columna `REQUIRED`:** se relaja como en 0.5 y se reanuda el stream:
  ```bash
  gcloud datastream streams update sima-central-cdc --location=us-central1 --state=RUNNING
  ```

Con `activo` ya en `public_proveedor`, se ponen al día la vista `dim_proveedor` y las
descripciones del hecho:

```bash
for f in 02_dimensiones.sql 07_descripciones_hecho.sql; do
  sed "s/\${PROYECTO}/$PROYECTO/g" "infra/gcp/bigquery/$f" | bq --location="$REGION" query --use_legacy_sql=false --quiet --format=none && echo "✓ $f"
done
bq query --use_legacy_sql=false "SELECT id_proveedor, nombre, activo FROM \`$PROYECTO.sima_analitica.dim_proveedor\` LIMIT 5"
```

Después de la siguiente corrida horaria de la consulta programada, la recepción de 4.4 aparece
en el hecho:

```bash
bq query --use_legacy_sql=false "SELECT tipo, COUNT(*) FROM \`$PROYECTO.sima_analitica.hecho_movimiento\` WHERE origen = 'CDC' GROUP BY 1 ORDER BY 1"
# aparece RECEPCION_ENTRADA
```

### 5.2 Carga (opcional)

```bash
infra/gcp/carga-gcp.sh correr
```

### 5.3 Limpieza (con todo estable, unos días después)

```bash
gcloud compute snapshots delete sucursal-1-antes-despliegue-AAAAMMDD sucursal-2-antes-despliegue-AAAAMMDD sucursal-3-antes-despliegue-AAAAMMDD
gcloud sql backups delete <ID de 2.4> --instance=sima-central
for N in 1 2 3; do en_vm "$N" 'sudo rm -f /var/lib/sima/antes-despliegue.dump'; done
```

### 5.4 Apply D: retirar `sa-sucursal`

Solo después de **un ciclo completo de apagar y encender limpio**:

```bash
infra/gcp/encender-apagar.sh apagar && infra/gcp/encender-apagar.sh encender
infra/gcp/verificar-gcp.sh --sin-funcional    # 0 fallaron, y en la sección 3 cada VM con sa-sucursal-N
```

En los `.tf` se quitan los cinco recursos y permisos de la cuenta compartida:
- `"sucursal-logs"` y `"sucursal-metricas"` de `permisos_de_proyecto` (`cuentas.tf`);
- `google_artifact_registry_repository_iam_member.sucursal_lee_imagenes` (`cloud_run.tf`);
- `google_pubsub_topic_iam_member.sucursal_publica` (`pubsub.tf`);
- `"sucursales"` de `invocadores_central` (`acceso-central.tf`).

Después:

```bash
./revisar-plan.sh          # salida 1: solo esos cinco "- destruir"
terraform apply plan.bin
```

La cuenta `sa-sucursal` se puede borrar después, en otro apply (quitarla de `local.cuentas`), o
dejarla sin permisos.

---

## 6. Volver atrás

Cada caso se ordena según hasta dónde se llegó.

| Falla en | Estado | Qué hacer |
|---|---|---|
| 0.x – 1.x | Nada de lo que corre cambió | Nada. Las cuentas y los secretos nuevos no molestan. Si se quieren quitar, los secretos tienen `prevent_destroy`: hay que sacarlo primero |
| 2.x – 3.1 | VMs detenidas; quizá imágenes nuevas en `:latest` | Volver `:latest` a las imágenes anteriores (comandos abajo) y encender las VMs: arrancan con su cuenta y su script viejos, porque 4.1 todavía no se aplicó |
| 3.2 a medias | Central migrado hasta la anterior a la que falló | **No se vuelve atrás:** se corrige la causa y se repite 3.2. Las comprobaciones de 1.2 son las causas posibles |
| 3.2 – 3.3, si se decide abandonar | Central migrado | El camino más caro: restaurar Cloud SQL (comandos abajo) |
| 3.3 (la revisión no arranca) | El tráfico sigue en la revisión anterior | Ver el motivo con `gcloud run revisions describe`. Casi siempre es un permiso de lectura de un secreto (0.6) o un secreto sin versión (`12-claves.sh --comprobar`). Se corrige y se repite 3.3 |
| 4.1 | VMs con la cuenta nueva, detenidas | Si hay que volver a la cuenta vieja: `git revert` del cambio de `vms.tf` y de `arranque.sh`, `./revisar-plan.sh` (salida 1, solo las VMs) y `terraform apply plan.bin`. `sa-sucursal` conserva todos sus permisos hasta 5.4 |
| 4.3 bis (saldo inicial) | Documentos AJUSTE de saldo inicial ya confirmados en alguna sucursal | **No se deshace**: la función es idempotente y lo ya valuado queda. Se corrige la causa (casi siempre una vigencia que falta) y se repite |
| 4.2 / 4.3 (una sucursal no arranca bien) | Corre el contenedor **viejo** (Docker lo reinicia al encender) sobre una base quizá migrada a medias | Leer la salida serie. Si es una clave o un permiso, corregir y `gcloud compute instances reset sucursal-N`. Si es una migración, restaurar el respaldo de 1.5 (comandos abajo) |
| Después de reabrir las ventas | — | **Solo hacia adelante.** Restaurar cualquier respaldo pierde lo vendido después |

**Volver `:latest` a las imágenes anteriores** (valores de `~/antes-despliegue.txt`):

```bash
source ~/antes-despliegue.txt
gcloud artifacts docker tags add "$IMAGEN@$DIGEST_API" "$IMAGEN:latest"
gcloud artifacts docker tags add "$IMAGEN_MIGRACIONES@$DIGEST_MIG" "$IMAGEN_MIGRACIONES:latest"
```

**Restaurar Cloud SQL** (pisa la instancia: se pierde todo lo posterior al respaldo):

```bash
gcloud sql backups restore <ID de 2.4> --restore-instance=sima-central
gcloud run jobs execute cdc-central --region=us-central1 --wait            # publicación y slot de Datastream
gcloud datastream streams describe sima-central-cdc --location=us-central1 --format='value(state,errors)'
#   si el stream quedó con error por el slot: pausarlo y reanudarlo; si no se recupera, un backfill
gcloud run services update-traffic api-central --region=us-central1 --to-revisions="$REV_ANTERIOR=100"
```

Además hay que volver las imágenes a `:latest` (bloque anterior) y encender las VMs con la
cuenta vieja. Ojo: `update-traffic` deja una diferencia en el plan de Terraform (`traffic`) hasta
que se vuelva a desplegar.

**Restaurar la base de una sucursal desde su respaldo lógico** (pierde lo que se vendió en ella
después de 1.5):

```bash
N=1
en_vm "$N" "sudo sh -c 'docker stop api-sucursal \
  && docker exec db-sucursal dropdb -U sima sima_sucursal_$N \
  && docker exec db-sucursal createdb -U sima sima_sucursal_$N \
  && docker exec -i db-sucursal pg_restore -U sima -d sima_sucursal_$N < /var/lib/sima/antes-despliegue.dump'"
```

Después hay que volver a `:latest` las imágenes anteriores y a la cuenta y el arranque viejos
(4.1), y reiniciar la VM.

Último recurso: el snapshot de 2.3. Restaurarlo es crear un disco nuevo y cambiar el disco de
arranque. Terraform lo vería como un reemplazo de la VM: **se planifica aparte**.

---

## 7. Costo

- **Secretos:** seis nuevos, unos USD 0,20 al mes.
- **Snapshots y respaldo de Cloud SQL:** centavos mientras existan (5.3).
- **VMs:** ninguna nueva.
- **Réplica:** cada sucursal consulta al central cada 10 s, unas 26.000 peticiones por día a
  Cloud Run con GCP encendido. Están dentro de las peticiones gratuitas, pero le dan algo de
  CPU facturable al central mientras está encendido.
