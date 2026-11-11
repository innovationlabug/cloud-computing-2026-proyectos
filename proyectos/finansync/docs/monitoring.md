# Observabilidad — Cloud Monitoring y Logging

Cloud Run manda sus registros a Cloud Logging sin configurar nada. Lo que sí
hace falta configurar es qué vigilar y a quién avisarle, y de eso se trata esta
carpeta.

## Eventos que registra la aplicación

El código no escribe texto suelto: imprime JSON en una sola línea, que es como
Cloud Logging lo puede indexar y filtrar por campo. Cada línea lleva la clave
`evento` y un campo `severity`, que es el que Cloud Logging usa para clasificar
la entrada. Sin ese campo todo entraría como INFO, aunque se haya escrito por
la salida de error.

| Evento | Cuándo | severity |
|---|---|---|
| `reporte_mensual_inicio` | arranca el job, con cuántos usuarios va a procesar | INFO |
| `reporte_mensual_fin` | termina, con enviados / sin movimientos / omitidos / fallidos y cuánto tardó | INFO |
| `reporte_mensual_error_usuario` | falló un usuario; el resto sigue | ERROR |
| `reporte_mensual_error` | falló el job completo | ERROR |
| `reporte_mensual_rechazado` | se llamó al endpoint sin credenciales válidas, con el motivo del rechazo | ERROR |

Para verlos en la consola, en Cloud Logging:

```
resource.type="cloud_run_revision"
resource.labels.service_name="finansync"
jsonPayload.evento="reporte_mensual_fin"
```

## Quién puede llamar al endpoint

El job solo acepta dos cosas: el token OIDC que firma Cloud Scheduler con la
cuenta de servicio de `REPORTE_SERVICE_ACCOUNT`, o el token compartido de
`REPORTE_CRON_TOKEN` en la cabecera `x-cron-token`.

`REPORTE_SERVICE_ACCOUNT` no es opcional. Si no está configurada, el endpoint
rechaza cualquier token OIDC, porque verificar solo la firma dejaría pasar el
token de cualquier cuenta de Google. En Cloud Run se configura así:

```bash
gcloud run services update finansync --region=us-central1 \
  --update-env-vars="REPORTE_SERVICE_ACCOUNT=finansync-scheduler@finansync-74cd1.iam.gserviceaccount.com"
```

`finansync-scheduler` es una cuenta aparte, usada solo por el job. Tiene el rol
*Invocador de Cloud Run* únicamente sobre el servicio `finansync`, no sobre el
proyecto, y la app no puede firmar tokens a su nombre.

Después conviene forzar una ejecución del job `estado-cuenta-mensual` desde
Cloud Scheduler: tiene que seguir terminando sin errores.

## Alertas

Dos políticas, en los archivos de esta carpeta:

**`alerta-reporte-mensual.json`** — avisa cuando aparece un
`reporte_mensual_error`, un `reporte_mensual_error_usuario` o un
`reporte_mensual_rechazado`. Es la que importa: el job corre una vez al mes, así
que si falla y nadie se entera, el mes se pierde. Se limita a un aviso por hora
para que un fallo con muchos usuarios no mande cien correos.

**`alerta-errores-5xx.json`** — avisa si el servicio pasa de cinco respuestas
5xx en cinco minutos. Cubre caídas de Vertex AI, de Cloud Vision o de la base.

El filtro de la primera busca por `jsonPayload.evento` y no por severidad, a
propósito: así sigue funcionando aunque cambie cómo se clasifica la línea, y no
se dispara con errores de otra parte del servicio.

## Cómo crearlas

Primero el canal al que llegan los avisos (una sola vez):

```bash
gcloud beta monitoring channels create \
  --display-name="Correo del equipo" \
  --type=email \
  --channel-labels=email_address=TU_CORREO \
  --project=finansync-74cd1 \
  --format="value(name)"
```

Ese comando devuelve el id del canal, algo como
`projects/finansync-74cd1/notificationChannels/1234567890`. Se copia dentro de
`notificationChannels` en los dos JSON, y después:

```bash
gcloud alpha monitoring policies create \
  --policy-from-file=monitoring/alerta-reporte-mensual.json \
  --project=finansync-74cd1

gcloud alpha monitoring policies create \
  --policy-from-file=monitoring/alerta-errores-5xx.json \
  --project=finansync-74cd1
```

Si se dejan los `notificationChannels` vacíos, las políticas igual se crean y se
ven en la consola, pero no avisan a nadie: solo quedan registradas como
incidentes. Para comprobar que quedaron creadas:

```bash
gcloud alpha monitoring policies list --project=finansync-74cd1
gcloud beta monitoring channels list --project=finansync-74cd1
```

## Cómo probar que la alerta sirve

Hay que provocar un rechazo que deje registro. Un token OIDC inválido lo hace:

```bash
curl -s -X POST \
  "https://finansync-1088114787810.us-central1.run.app/api/reportes/mensual" \
  -H "Authorization: Bearer token-falso"
```

El endpoint devuelve 401 y escribe un `reporte_mensual_rechazado` con motivo
`token invalido`. Entre cinco y diez minutos después tiene que llegar el correo
de la alerta.

Mandar la cabecera `x-cron-token` con un valor incorrecto también deja el mismo
registro, con motivo `token compartido incorrecto`.
