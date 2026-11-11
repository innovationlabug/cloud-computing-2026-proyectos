# Despliegue Azure - guía por etapas

Entorno: **Azure Cloud Shell, Bash**, desde la carpeta que contiene `package.json`. Estos comandos fueron revisados con Microsoft Learn, pero no se ejecutaron contra una suscripción. La disponibilidad de región, cuotas, ACR Tasks y tipos de cuenta se comprueba con la cuenta real. No activar ni ampliar facturación automáticamente.

## 1. Acceso y suscripción (hacer primero)
Jessel inicia sesión en Azure Portal con su cuenta. Abre Suscripciones, selecciona la real y revisa nombre, estado, créditos y acceso. Da acceso individual al compañero mediante IAM; cada persona inicia sesión con su cuenta. No compartir contraseñas. Si no aparece suscripción habilitada, detener el despliegue y resolverlo con la titular.

Abrir Cloud Shell (icono de terminal en el portal), seleccionar Bash y ejecutar solo:

```bash
az account list --query '[].{Nombre:name,Id:id,Estado:state}' -o table
```

Con el ID real, seleccionar y comprobar la suscripción:

```bash
az account set --subscription ID_REAL
az account show --query '{Nombre:name,Id:id,Estado:state}' -o table
```

Guardar la salida sin información de cuenta innecesaria. Crear un presupuesto de seguimiento en Cost Management según el acuerdo del equipo; las alertas no detienen el gasto. No se asegura costo cero. ACR Basic, tareas de compilación, almacenamiento y otros recursos tienen consumo aunque Container Apps escale a cero.

## 2. Preparar archivos y revisar permisos
Subir este ZIP usando el botón de archivos de Cloud Shell; descomprimir y entrar a `Proyecto_Cloud-main` (la carpeta con package.json). Revisar `az version` y actualizar/instalar la extensión oficial si falta:

```bash
az extension add --name containerapp --upgrade
```

La persona que despliega necesita crear recursos en el grupo dedicado y asignar roles a la identidad del proyecto. Contributor por sí solo no concede asignación de roles. Si falta permiso, resolver el permiso exacto; no dar Owner a todas las identidades. La identidad de ejecución recibe solo los roles del script.

Comprobar proveedores de la suscripción:

```bash
az provider show -n Microsoft.App --query registrationState -o tsv
az provider show -n Microsoft.Storage --query registrationState -o tsv
az provider show -n Microsoft.ContainerRegistry --query registrationState -o tsv
az provider show -n Microsoft.ManagedIdentity --query registrationState -o tsv
```

Si alguno no está Registered, la titular o administrador puede registrar únicamente el necesario con `az provider register --namespace NOMBRE --wait`. Revisar la región y cuotas disponibles en el portal antes de continuar.

## 3. Definir nombres reales
Configurar estas variables en Bash, reemplazando cada marcador y usando recursos dedicados. Los nombres Storage y ACR deben ser globalmente únicos; no usar recursos ajenos. La región debe estar habilitada para Container Apps y Storage.

```bash
export SUBSCRIPTION_ID=ID_REAL
export RESOURCE_GROUP=NOMBRE_GRUPO_DEL_PROYECTO
export LOCATION=REGION_CONFIRMADA
export STORAGE_ACCOUNT=NOMBRE_UNICO_MINUSCULAS
export ACR_NAME=NOMBRE_UNICO_REGISTRO
export APP_NAME=asistente-negocios-m2
export ENV_NAME=entorno-negocios-m2
export IDENTITY_NAME=identidad-negocios-m2
export CONTAINER_NAME=comprobantes
```

## 4. Preparar recursos
Revisar el script antes de ejecutar. Esta etapa crea recursos y puede generar cargos:

```bash
bash scripts/desplegar-azure.sh preparar
```

Crea grupo, StorageV2/LRS, contenedor privado, ACR Basic sin administrador, identidad administrada y entorno Container Apps sin destino persistente de logs. Asigna datos solo al contenedor y AcrPull solo al registro. La etapa no es una transacción: si falla, inspeccionar recursos ya creados antes de reintentar. El script es para este proyecto nuevo; no se promete reconciliación completa de una infraestructura existente. Los roles pueden tardar en propagarse.

## 5. Compilar y publicar

```bash
bash scripts/desplegar-azure.sh publicar
```

ACR compila el Dockerfile, sin requerir Docker local. El script pide un token de al menos 32 caracteres (letras, números, guion o guion bajo) sin mostrarlo. La titular genera/guarda el token en privado. Se guarda como secreto de Container Apps, no en archivos. No ejecutar con depuración `set -x` ni `az --debug`.

La aplicación usa identidad administrada para descargar su imagen y acceder a Blob Storage. Ingress HTTPS, puerto interno 8080, 0-2 réplicas, 0.25 CPU/0.5 GiB. El resultado muestra FQDN, revisión lista y estado de aprovisionamiento. Una respuesta de creación no basta: verificar revisión disponible y probar URL. Si ACR Tasks no está disponible para el tipo de suscripción, conservar el error y acordar una vía de compilación autorizada; no cambiar modalidad de pago automáticamente.

## 6. Probar integración real
Con la URL HTTPS real, sin barra final:

```bash
export URL=https://FQDN_REAL
bash scripts/verificar-m2.sh
```

El script crea un comprobante Q50, lo recupera y compara el contenido completo en otra petición. Comprueba rechazos, IDs y acceso anónimo. Guarda `evidencia-http-m2.json` sin token. Esta prueba no reemplaza las evidencias administrativas siguientes.

En Azure Portal, abrir la cuenta Storage -> Contenedores -> comprobantes -> comprobantes-demo y localizar el ID devuelto. El usuario del portal necesita permiso de lectura de datos; iniciar sesión o ser Contributor de recursos no concede necesariamente ese permiso. Con permisos sobre los datos se puede verificar por CLI:

```bash
az storage blob show --account-name "$STORAGE_ACCOUNT" --container-name "$CONTAINER_NAME" --name comprobantes-demo/ID_REAL.json --auth-mode login --query '{nombre:name,bytes:properties.contentLength}' -o json
```

Guardar también los controles de privacidad (no contienen secretos):

```bash
az storage account show -g "$RESOURCE_GROUP" -n "$STORAGE_ACCOUNT" --query '{publico:allowBlobPublicAccess,clavesCompartidas:allowSharedKeyAccess,https:enableHttpsTrafficOnly,tls:minimumTlsVersion,region:location}' -o json
az storage container-rm show -g "$RESOURCE_GROUP" --storage-account "$STORAGE_ACCOUNT" --name "$CONTAINER_NAME" --query publicAccess -o tsv
```

Se espera false, false, true, TLS1_2 y acceso de contenedor None. Localizar en Container Apps -> flujo de registros el mensaje `comprobante creado` con el mismo objeto. Se puede consultar con:

```bash
az containerapp logs show -g "$RESOURCE_GROUP" -n "$APP_NAME" --type console --tail 50
```

Con cero réplicas puede ser necesario abrir la aplicación antes de consultar. Los logs de consola no son evidencia de una solución Monitor/Log Analytics configurada. Guardar salida/capturas depuradas con fecha; nunca mostrar el secreto ni la vista completa de configuración.

## 7. Completar informe
Rellenar suscripción, grupo, región, app, FQDN, cuenta, contenedor y revisión en `docs/avance-M2.md`. Añadir evidencia real de cuenta, recursos, comprobante, lectura y logs. Mantener Key Vault, PostgreSQL, IA y Monitor como planeados mientras no estén integrados.

## Referencias oficiales
- https://learn.microsoft.com/en-us/azure/container-apps/managed-identity
- https://learn.microsoft.com/en-us/azure/container-apps/managed-identity-image-pull
- https://learn.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-nodejs
- https://learn.microsoft.com/en-us/cli/azure/storage/container-rm
- https://learn.microsoft.com/en-us/cli/azure/containerapp
- https://learn.microsoft.com/en-us/azure/container-apps/manage-secrets
- https://nextjs.org/blog/nextjs-security-update-october-14-2026
