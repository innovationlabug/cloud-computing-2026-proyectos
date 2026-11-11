# CI/CD de desarrollo verificada

GitHub Actions construye y publica automáticamente la aplicación existente en Azure Container Apps al actualizar los archivos relevantes de `main`. No requiere la computadora de Lizardo encendida.

## Flujo

1. Checkout del commit, con credenciales de Git sin persistir.
2. Pruebas del script que conserva escala, recursos, variables y probes.
3. Identidad federada de construcción, Docker build con compilación, pruebas de Monitor y protecciones cloud.
4. Imagen privada en ACR con etiqueta de SHA completo.
5. Otro job usa una identidad distinta de publicación.
6. El script reemplaza la imagen de la app existente y fija versión y telemetría.
7. Espera una revisión lista y comprueba salud, SHA público, acceso 200 y API sin sesión 401.
8. Ante un fallo de activación o comprobación solicita restaurar la imagen previa y falla el job. Esa recuperación automática tiene pruebas de conservación de configuración, pero no se ha forzado un fallo real del despliegue para probarla.

La ejecución completa queda acreditada en `evidencias/cicd-despliegue.json` y en el enlace de Actions incluido allí. El pipeline no crea ventas, usuarios ni archivos de prueba, y no reconstruye PostgreSQL o sus reglas de firewall.

## Identidades

- `id-cloud-build-dev`: AcrPush sobre el registro.
- `id-cloud-deploy-dev`: rol personalizado de lectura/escritura de Container Apps y lectura de revisiones, asignado exclusivamente a esta aplicación.
- `id-cloud-app-dev`: identidad del servidor para Blob, Key Vault y extracción de imagen.

La federación restringe repositorio y rama mediante su subject exacto con IDs estables. No se almacenan contraseñas Azure ni de ACR en GitHub. La identidad de publicación no tiene permisos directos sobre secretos, base o facturación. Poder desplegar código implica confiar en los cambios de `main`, porque el código ejecutado usa la identidad del servidor.

## Límites preservados

0.5 vCPU, 1 GiB, mínimo 0 y máximo 1 réplica, modo Single. El script rechaza una configuración distinta para que se revise antes de publicar. No cambia el límite de gasto ni eleva capacidad. El máximo es por revisión y no es un tope en dólares. El código local de configuración verifica `spendingLimit On`; el publicador CI solo tiene permisos sobre la app y no modifica facturación.

La concurrencia del workflow serializa las publicaciones y no cancela un despliegue a mitad. Los cambios solo de documentación y presentaciones no reconstruyen la imagen. Las acciones permanecen fijadas a commits.

## Evidencias y edición

- `.github/workflows/imagen-dev.yml`: flujo de dos jobs.
- `scripts/desplegar-ci.py`: actualización y comprobación.
- `scripts/test-despliegue.py`: pruebas locales sin Azure.
- `scripts/configurar-operacion.py deployment`: identidad, rol y federación.
- `infra/deployment.json`: identificadores públicos de recursos.

Los minutos de Actions pertenecen al plan GitHub, separados del crédito Azure. Una nueva publicación puede solapar revisiones temporalmente; no se promete costo cero.
