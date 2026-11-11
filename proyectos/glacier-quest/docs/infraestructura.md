# Infraestructura y operación

[Presentación](../README.md) · [Aplicación](aplicacion.md)

## Recursos de la demo

Proyecto `glacier-quest-1791678996`, región principal `us-central1`. Configuración contrastada el 24 de octubre de 2026; comprobar la consola para cambios posteriores.

| Componente | Recurso y configuración |
| --- | --- |
| Cloud Run | `glacierquest-demo`: 1 CPU, 512 MiB, mínimo 0, máximo 1 instancia, concurrencia 40, timeout 30 s. |
| Realtime Database | `glacier-quest-1791678996-demo`: progreso, chat y presencia. Reglas de cliente cerradas; acceso del servidor mediante IAM. |
| Authentication | Correo y contraseña. La API verifica identidad y emite una sesión propia del juego. |
| Storage | `glacier-quest-1791678996-demo-assets`, `glacier-quest-1791678996-demo-build`, `glacier-quest-1791678996_cloudbuild`: recurso visual y fuentes de construcción. |
| Artifact Registry | Repositorio Docker `glacierquest-demo`, imagen `game`. |
| Secret Manager | `glacierquest-demo-session` y `glacierquest-demo-code`, referenciados por Cloud Run. |
| Scheduler | `glacierquest-cleanup`: `17 * * * *`, UTC; POST `/internal/cleanup` con OIDC. |
| Monitoring | Panel `Glacier Quest - Operacion de la demo`: solicitudes, latencia, instancias, memoria y logs. |

## Identidades y seguridad

- `glacierquest-runtime`: ejecuta el juego; lee Storage y secretos, y accede a Database.
- `glacierquest-build`: escribe imágenes, lee fuentes de construcción y escribe logs.
- `glacierquest-cleanup`: identidad OIDC autorizada para mantenimiento.
- `glacierquest-deploy`: identidad utilizada en la comprobación de federación; su nombre no implica despliegue automático.

El rol personalizado `glacierquestDatabaseData` incluye `firebasedatabase.instances.get` y `firebasedatabase.instances.update`. El segundo también permite modificar reglas y habilitar/deshabilitar la base: no es únicamente escritura de partidas.

Cloud Run permite abrir la web públicamente, mientras la API protege las acciones. Secret Manager entrega valores al servidor sin incorporarlos al repositorio ni a la imagen.

## GitHub Actions

Los workflows activos están en `.github/workflows` de la raíz:

- `ci.yml`: instala dependencias, construye el contenedor y valida formato y sintaxis de Terraform. Se activa en push a `main` o `codex/preentrega-iam-actions`, pull requests y ejecución manual. No publica imágenes ni despliega.
- `verify-cloud-identity.yml`: comprueba manualmente OIDC con Workload Identity Federation. No despliega.

La federación restringe repositorio, propietario, rama y workflow. La rama permitida es `codex/preentrega-iam-actions`; cambiarla exige revisar esa condición. No se usa una clave JSON persistente de cuenta de servicio en este flujo.

La versión consolidada está en `main`, donde se ejecuta la validación de código. La comprobación manual de identidad se sigue ejecutando seleccionando `codex/preentrega-iam-actions`; en `main` su job se omite por la condición de rama. La integración a `main` no cambia permisos en Google Cloud ni publica una revisión de Cloud Run.

## Construcción y despliegue

`demo/deploy.ps1` distingue `Provision`, `Build` y `Deploy`. Usa la configuración gcloud `glacierquest-creditos`, comprueba proyecto y cuenta de facturación esperados y exige `-TrialVerified`. Ese parámetro es confirmación del operador, no consulta automática del crédito.

Autenticar primero la cuenta autorizada y comprobar modalidad, saldo y vigencia en Billing. El script contiene identificadores específicos de esta demo; revisarlo antes de usar otro proyecto o cuenta.

Desde `glacierquest/`:

1. Revisar los cambios y el resultado de construcción en Actions.
2. Elegir una etiqueta nueva de `us-central1-docker.pkg.dev/glacier-quest-1791678996/glacierquest-demo/game` y ejecutar `infra/demo/deploy.ps1 -Stage Build -TrialVerified -Image <imagen:etiqueta>`.
3. Esperar que Cloud Build termine correctamente. `cloudbuild.demo.yaml` ejecuta Docker y publica en Artifact Registry. Build devuelve un identificador sin esperar el resultado final.
4. Obtener y verificar el digest SHA256.
5. Ejecutar `infra/demo/deploy.ps1 -Stage Deploy -TrialVerified -Store realtime -Image <imagen@sha256:digest> -FirebaseConfigPath <configuracion-local.json>`.
6. Comprobar revisión, tráfico, `/health`, acceso y funcionamiento. Un push a GitHub por sí solo no actualiza el juego público.

Los valores entre `<...>` son marcadores que hay que sustituir. La configuración local de Firebase debe contener `projectId` y `apiKey` del proyecto; mantenerla fuera de Git. `Provision` modifica recursos y permisos, y no se necesita en cada actualización.

## Scripts y configuración

| Archivo | Uso |
| --- | --- |
| `demo/configure-firebase.ps1` | Database, permisos y Scheduler; modifica recursos. |
| `demo/database.rules.json` | Reglas e índices de RTDB. |
| `demo/firebase-data-role.yaml` | Rol personalizado de Database. |
| `demo/build-lifecycle.json` | Retención de fuentes temporales. |
| `demo/configure-monitoring.py` | Crea o actualiza el panel de `monitoring-dashboard.json`. |
| `bootstrap-identity.sh` | Identidades y federación GitHub/Google. |
| `terraform/` | Infraestructura histórica validada por CI; no representa íntegramente la demo. |

No aplicar Terraform sin conciliar recursos, nombres, estado, permisos y proveedor de datos. Conserva Cloud SQL y Firestore como alternativas; la demo usa Realtime Database. Validar no aplica cambios ni demuestra que Terraform gestione todos los recursos.

## Operación y límites

El cliente consulta periódicamente; no hay WebSockets. El servidor consulta cantidades limitadas de mensajes y presencia antes de filtrar por zona. Las partidas PvP comparten un registro de arena. El mantenimiento borra cantidades acotadas de datos temporales, no el progreso. La descripción histórica del job menciona siete días, pero el código usa plazos más cortos para chat y presencia.

La revisión del 24/10 encontró el panel de Monitoring y ninguna política de alerta. El presupuesto de US$10 tiene avisos y excluye créditos en su medición: no corta automáticamente el consumo. Los límites de instancias no limitan toda la factura. Firebase también puede generar cargos según plan y uso.

Medir latencia, errores, tráfico, consumo de Database y costo por sesión antes de ampliar capacidad. No se han demostrado 50 mil usuarios concurrentes ni un procedimiento completo de restauración.

## Consolas

- [Cloud Run](https://console.cloud.google.com/run?project=glacier-quest-1791678996)
- [Storage](https://console.cloud.google.com/storage/browser?project=glacier-quest-1791678996)
- [Cloud Build](https://console.cloud.google.com/cloud-build/builds?project=glacier-quest-1791678996)
- [Artifact Registry](https://console.cloud.google.com/artifacts?project=glacier-quest-1791678996)
- [IAM](https://console.cloud.google.com/iam-admin/iam?project=glacier-quest-1791678996)
- [Secret Manager](https://console.cloud.google.com/security/secret-manager?project=glacier-quest-1791678996)
- [Scheduler](https://console.cloud.google.com/cloudscheduler?project=glacier-quest-1791678996)
- [Logging](https://console.cloud.google.com/logs/query?project=glacier-quest-1791678996)
- [Monitoring](https://console.cloud.google.com/monitoring?project=glacier-quest-1791678996)
- [Firebase](https://console.firebase.google.com/project/glacier-quest-1791678996/overview)
- [Actions](https://github.com/angelprogt3985/Cloud-Computing-proyecto/actions)

Si Build no muestra historial, seleccionar `us-central1` y comprobar proyecto, cuenta y filtros.
