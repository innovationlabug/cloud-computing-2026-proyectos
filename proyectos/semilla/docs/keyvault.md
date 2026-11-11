Actualización de publicación: Key Vault ya fue ejercitado desde Container Apps con la identidad administrada `id-cloud-app-dev`. El usuario confirmó el acceso Google desde HTTPS. Ver docs/publicacion.md para la URL y evidencias; las notas que describen la etapa local corresponden al paso anterior.

# Key Vault integrado en Asistente Negocios

Estado verificado el 21 de octubre de 2026 (Guatemala): el servidor obtiene dos credenciales reales desde el almacén existente `kv-cloud-lizardo-dev`:

| Referencia en configuración | Secreto en Key Vault | Uso en la app |
| --- | --- | --- |
| `DATABASE_URL_KEY_VAULT_SECRET` | `asistente-postgresql-url` | Pool PostgreSQL del usuario limitado `asistente_app`; operaciones, negocios y sesiones. |
| `GOOGLE_CLIENT_SECRET_KEY_VAULT_SECRET` | `asistente-google-client-secret` | Configuración confidencial OIDC de Google en el servidor. |

Las variables `DATABASE_URL` y `GOOGLE_CLIENT_SECRET` se retiraron del archivo activo `.env.local`. Los nombres de los secretos y del almacén no son credenciales. El cliente Google público, URL de retorno y selección de proveedor permanecen como configuración normal.

## Flujo y permisos

App local → credencial Azure del desarrollador mediante `az login` → Key Vault con RBAC → credencial en memoria del servidor → PostgreSQL o cliente OIDC Google. El navegador no recibe los secretos ni los tokens de Azure.

La identidad administrada `id-cloud-app-dev` tiene su rol preparado, pero no se ha ejercitado en un despliegue: la aplicación sigue ejecutándose en el equipo. En Container Apps deberá vincularse y configurarse su clientId; allí el adaptador exige ManagedIdentityCredential. No atribuir la prueba local a esa identidad.

El lector conserva los valores en memoria hasta ocho horas, agrupa solicitudes concurrentes y descarta lecturas fallidas para permitir reintento. No acepta redirecciones de red ni nombres que puedan cambiar el host de Key Vault. Los errores no contienen valores ni respuestas del proveedor. Si existe una referencia y Key Vault falla, no se sustituye silenciosamente por una contraseña de entorno.

El pool PostgreSQL y la configuración OIDC se reutilizan durante la vida del proceso. **Después de rotar una credencial hay que reiniciar la aplicación**; la caché no implementa rotación automática del pool ni de OIDC. No se creó un sondeo periódico.

## Ejecutar

En PowerShell, desde `D:\Usuario\Lizardo\cloud`:

```powershell
az login
$env:PORT="3010"
npm.cmd run dev:local
```

Abrir `http://localhost:3010/` para conservar el retorno OAuth. El puerto debe coincidir con `AUTH_BASE_URL` y el autorizado en Google. La IP del equipo debe estar permitida en el firewall PostgreSQL; mantener la regla limitada, sin abrir todas las direcciones.

Clonar el repositorio no concede acceso a las credenciales ni a los recursos. La configuración cloud necesita `.env.local` privado con las referencias, una cuenta Azure autorizada y acceso de red. Sin configuración cloud, la alternativa local SQLite sigue disponible como antes.

## Transferencia inicial y verificación

`scripts/configurar-keyvault.mjs` es una herramienta de transferencia inicial desde las variables privadas existentes. Crea un respaldo privado, carga cada valor usando un archivo temporal (no el valor en argumentos) y solo después de completar ambas escrituras cambia `.env.local` a referencias. Borra esos archivos temporales. No repetir para rotar automáticamente: con referencias ya activas no hay valores locales que transferir. No registrar ni publicar sus archivos privados.

Se conservaron los respaldos previos, archivos de recuperación de PostgreSQL y descarga OAuth en sus ubicaciones privadas. Este cambio reduce las credenciales del archivo activo; no borra todas las copias anteriores del equipo. `.data`, `.env.local` y descargas OAuth están excluidos de Git.

Validación realizada:

- Compilación de producción y TypeScript correctos.
- Pruebas simuladas: caché concurrente, aislamiento por nombre, referencia inválida, error sin datos, reintento, secreto deshabilitado y rechazo sin fallback.
- Once comprobaciones PostgreSQL locales sin Azure.
- Consulta real PostgreSQL leyendo su URL desde Key Vault.
- API real autenticada 200 y sin sesión 401; sesión temporal de prueba revocada.
- Inicio Google redirige al proveedor usando la credencial recuperada y descubrimiento OIDC real. No se ejecutó otro callback ni otro inicio completo por el usuario.
- Sin movimientos nuevos, sin cargas Blob y sin recursos nuevos. La prueba Google inició un flujo temporal que se consumió para limpiarlo.

Evidencia pública sin valores: `evidencias/keyvault-integracion-real.json`. Para una comprobación dirigida existe `npm run verify:keyvault-real`; no es necesario ejecutarla repetidamente.

Guía de Microsoft: https://learn.microsoft.com/en-us/azure/key-vault/secrets/secure-secrets

## Siguiente etapa

Preparar Container Apps con HTTPS, configuración de red PostgreSQL y retorno Google de producción, y después activar la identidad administrada. Azure Monitor y CI/CD se incorporarán por etapas. Key Vault ya tiene una función real; la URL pública sigue pendiente.
