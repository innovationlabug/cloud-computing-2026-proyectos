Actualización de publicación: la app HTTPS en Container Apps ya accede a esta misma base PostgreSQL; la prueba técnica y el acceso Google del usuario están confirmados. Ver docs/publicacion.md.

# Operaciones y sesiones en PostgreSQL

## Estado verificado

El servidor quedó Ready y la app fue cambiada a PostgreSQL. Se verificaron 22 tablas y 50 registros por cantidad y hash, con 2 negocios, 5 movimientos y 1 adjunto respaldado. La prueba real validó TLS, el rol de datos, una operación revertida y HTTP 200/401. Las escrituras están reanudadas. SQLite y sus copias previas se conservan. La app todavía se ejecuta en localhost.

## Alcance

La app puede usar SQLite o PostgreSQL mediante `DATABASE_PROVIDER`. Los servicios de negocio, sesiones Google, membresías, inventario y metadatos de comprobantes comparten el mismo adaptador asíncrono. Azure Blob conserva los archivos respaldados; sus nombres e identificadores no cambian al trasladar la base. La app todavía se ejecuta localmente hasta preparar Container Apps.

## Configuración de desarrollo activa

- Azure Database for PostgreSQL Flexible Server, `pg-cloud-lizardo-dev`, Central US.
- PostgreSQL 16, Burstable `Standard_B1ms`, 32 GB Premium SSD, una instancia, sin alta disponibilidad ni respaldo geográfico, retención de siete días y crecimiento automático desactivado.
- Acceso público únicamente desde la IP de desarrollo, no `All` ni la regla global de todos los servicios Azure. TLS con validación del certificado.
- Cuenta de ejecución `asistente_app`, con permisos de datos y secuencias; la app no usa la cuenta administradora de migración.
- Pool de cuatro conexiones y tiempos máximos de conexión y consulta de 15 segundos. Transacciones en una misma conexión, aislamiento SERIALIZABLE y hasta tres intentos ante serialización o deadlock. Los callbacks transaccionales no hacen cargas a Blob ni otros efectos externos.

East US informó restricción de aprovisionamiento para esta suscripción. Central US no informó esa restricción en la consulta previa; no es una garantía de capacidad hasta que el servidor termine de crearse.

## Costo y crédito

La API oficial de precios consultada para Central US devolvió B1MS a USD 0.01921 por hora y almacenamiento estándar Flexible Server a USD 0.13/GB-mes. A 730 horas: 0.01921 × 730 + 0.13 × 32 = USD 18.1833/mes. Es una proyección de cómputo y almacenamiento, no una factura; no incluye impuestos, transferencia, almacenamiento de backup que exceda lo incluido u otros servicios. El respaldo LRS consultado fue USD 0.108/GB-mes cuando sea facturable.

La suscripción fue comprobada como FreeTrial activa con spendingLimit=On. El usuario autorizó crear el servidor condicionado al crédito promocional. No se quitó ese límite ni se cambió a pago por uso. El crédito puede vencer o agotarse; su saldo exacto se consulta en el portal. El costo no equivale a un pago mensual adelantado y el beneficio gratuito de 12 meses no se asume como confirmado.

## Migración y cambio de fuente

1. Verificar que la cuenta conserve el plan y límite autorizados, registrar el proveedor Microsoft.DBforPostgreSQL si hace falta y crear un solo servidor con la configuración descrita. Generar contraseñas seguras privadamente; no ponerlas en la documentación ni en Git.
2. Crear la base `negocios` con UTF8. La configuración de administración queda en `.data/private/postgres-admin.json`, ignorada por Git.
3. `node scripts/migrar-postgres.mjs --keep-lock` pausa las escrituras mediante un archivo de mantenimiento, espera el bloqueo de SQLite y crea una copia consistente en `.data/backups`.
4. La migración exige que el destino esté vacío, crea el esquema en una transacción, copia todas las tablas, conserva el orden de SQLite en `_seq` y compara hashes SHA-256 y cantidades por tabla. Ante cualquier diferencia revierte PostgreSQL y deja SQLite como fuente activa. No modifica los registros fuente.
5. Preparar el rol de datos con `node scripts/preparar-rol-postgres.mjs`. Su URL privada queda en `.data/private/postgres-runtime.json`.
6. Ejecutar la prueba real acotada `node scripts/verificar-postgres-real.mjs`: lee negocios, comprueba una operación y la revierte deliberadamente, y verifica el acceso HTTP y la respuesta 401 sin sesión. No deja gastos ficticios ni vuelve a cargar archivos a Blob.
7. Solo después de validar, configurar en `.env.local` `DATABASE_PROVIDER=postgres` y `DATABASE_URL=<URL privada del rol de aplicación>`, respaldando la configuración anterior. Reiniciar/recargar el servidor y retirar el archivo `.data/negocios.sqlite.migration-lock` para reanudar escrituras. Conservar SQLite y sus backups.

No ejecutar otra migración encima de un destino con datos. Si la migración fue correcta pero falta el cambio de configuración, revisar la evidencia y completar el cambio manteniendo la pausa de escrituras. Los identificadores de usuario Google y las sesiones hash se conservan; los negocios anteriores siguen vinculados al mismo propietario.

## Modo local y recuperación

`DATABASE_PROVIDER=sqlite` mantiene SQLite como alternativa explícita. Las pruebas fijan SQLite y borran DATABASE_URL para no tocar la base real. No hay fallback automático: si PostgreSQL falla, la app devuelve un error en lugar de guardar silenciosamente operaciones en otra base.

La copia SQLite es una copia del momento de migración. Después de comenzar a registrar nuevas operaciones en PostgreSQL ya no debe tratarse como una copia actualizada: volver a SQLite sin trasladar los datos nuevos los dejaría fuera de la vista. La recuperación operativa de PostgreSQL requiere sus respaldos y un procedimiento de restauración; la copia previa sirve para revisar el traslado, no como sincronización bidireccional.

La identidad administrada para conectarse a PostgreSQL y el almacenamiento de su credencial en Key Vault quedan para el despliegue seguro. En esta etapa se usa una contraseña de aplicación privada y TLS. Google identifica usuarios; la autorización de sus negocios sigue en la app, no en roles SQL por persona.

## Verificación

- `npm run test:postgres`: motor PostgreSQL PGlite local, sin Azure. Valida SQL, transacciones, rollback, sesiones, aislamiento, archivos y migración por hashes. No sustituye pruebas de red/certificados de Azure.
- `npm run test:workspace`, `test:google`, `test:auth` y `test:attachments`: regresión aislada del modo SQLite.
- Evidencias reales, cuando existan: `evidencias/migracion-postgresql.json` y `evidencias/postgresql-real.json`. No contienen contraseñas, URLs de conexión privadas ni contenido de comprobantes.

Referencias oficiales: [node-postgres y transacciones](https://node-postgres.com/features/transactions), [TLS](https://node-postgres.com/features/ssl), [crédito Azure](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/create-free-services) y [precios](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/).
