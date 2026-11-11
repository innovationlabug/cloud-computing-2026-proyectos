> Versión vigente de exposición (22/10/2026): [guion y defensa actualizada](https://github.com/lizardoenm12/semilla-cloud/blob/main/docs/presentaciones/Guion_y_defensa_actualizada.md). Este documento conserva análisis del 19/10 y cifras históricas. La web HTTPS y las integraciones PostgreSQL, Key Vault e identidad administrada ya están activas.

> Actualización: Google ya fue verificado con la cuenta real y los datos del negocio y sesiones están en PostgreSQL. La app continúa local; los archivos respaldados permanecen en Blob. Ver [estado PostgreSQL](postgresql.md). Las referencias anteriores a SQLite describen el modo alternativo o el estado previo al traslado.

# Defensa técnica y decisiones del proyecto cloud

Equipo: Lizardo Gonzalez y Jessel Contreras. Fecha de revisión: 19 de octubre de 2026. Nombre del producto provisional. Interpretamos “WS” como Amazon Web Services, AWS.

Este documento reúne respuestas para la exposición, las decisiones de arquitectura y sus alternativas. Elegimos Azure por la viabilidad del equipo y la integración prevista entre servicios. No afirmamos que sea universalmente más rápido, barato o seguro que GCP, AWS o Vercel. Los costos y usuarios futuros son supuestos de planificación, no facturas ni resultados de pruebas de carga.

## Respuestas a las cinco preguntas recomendadas

### 1 Por qué Azure y en qué le gana a GCP y AWS

**Respuesta para decir:** “Elegimos Azure porque pudimos activar una suscripción, ya verificamos Blob y Key Vault desde el backend y podemos continuar con una misma estrategia de identidades y permisos. Nos permite demostrar la arquitectura cloud solicitada sin repartir la operación entre varios proveedores. Esa ventaja es de encaje para nuestro equipo, no una superioridad universal”.

Originalmente consideramos GCP y el equipo no consiguió activar el beneficio esperado. Eso no significa que GCP carezca de créditos ni que el crédito Azure sea permanente. La suscripción Azure mostró USD 200 con vencimiento en 30 días al configurarla. No comprobamos el saldo actual ni usamos ese beneficio para justificar un costo mensual de cero.

| Necesidad | Azure elegido | Equivalente posible en GCP | Equivalente posible en AWS |
|---|---|---|---|
| Ejecutar contenedores | Container Apps | Cloud Run | App Runner o ECS con Fargate, según operación |
| Archivos | Blob Storage | Cloud Storage | S3 |
| PostgreSQL administrado | Azure Database for PostgreSQL | Cloud SQL for PostgreSQL | RDS for PostgreSQL |
| Identidades y permisos | Entra ID, RBAC e identidades administradas | IAM y cuentas de servicio | IAM y roles de ejecución |
| Secretos | Key Vault | Secret Manager | Secrets Manager |
| Observabilidad | Azure Monitor | Cloud Logging y Cloud Monitoring | CloudWatch |

Son equivalencias de función, no igualdad de precios, límites o configuración. Los otros proveedores también ofrecen identidad y secretos. No hicimos un benchmark entre ellos, por lo que no tenemos evidencia para afirmar menor latencia o mayor rendimiento de Azure. Una comparación económica válida requeriría la misma región, carga, disponibilidad, retención, red y soporte. [S1][S2][S3][S4]

### 2 Proyección de costos

**Respuesta para decir:** “La fase actual usa ejecución local y pruebas pequeñas de servicios existentes. Para un piloto cloud de diez usuarios planteamos una reserva de USD 20 a 80 al mes, antes de impuestos y sin IA generativa. Es una estimación de planificación que debemos cotizar por región y medir, no una tarifa garantizada”.

**Supuestos del piloto futuro:** región East US, un entorno de desarrollo, Container Apps en Consumption con mínimo cero, PostgreSQL pequeño de desarrollo, hasta 1 GiB de archivos iniciales, pocos secretos, retención de logs acotada, sin alta disponibilidad, sin endpoints privados y sin un modelo de IA de pago. PostgreSQL Burstable es una opción de prueba que requiere vigilar créditos CPU; no lo presentamos como configuración validada para producción. Un servicio continuo de producción necesitará nueva selección y cotización. [S5][S6]

| Componente | Reserva mensual USD para planificación | Qué puede aumentar el costo |
|---|---:|---|
| Container Apps | 0 a 10 | Réplicas, tiempo activo, CPU y memoria |
| PostgreSQL con almacenamiento | 15 a 35 | Tamaño del servidor, disco, backups y disponibilidad |
| Blob Storage | 0.10 a 2 | GiB acumulados, operaciones, versiones y descargas |
| Entra e identidad administrada básica | 0 previsto | Licencias o capacidades adicionales, no contempladas |
| Monitor | 0 a 5 | Ingesta, retención y consultas de logs |
| Pipelines | 0 condicionado | Solo si se concede una asignación gratuita suficiente |
| Key Vault | 0 a 1 | Cantidad de operaciones y funciones contratadas |
| Registro de imágenes auxiliar | 0 a 6 | Registro privado e imágenes retenidas |
| Transferencias y otros consumos | 0 a 5 | Egreso y servicios auxiliares |
| Total de reservas anteriores | 15.10 a 64.00 | No es una cotización del proveedor |
| Total con margen de 20 por ciento | 18.12 a 76.80 | Reserva redondeada propuesta de 20 a 80 |

Cada rango es una asignación interna aproximada, no una tarifa oficial extraída para una SKU. Antes de desplegar debemos sustituirla por precio unitario vigente y volumen en la calculadora de Azure. La consulta automatizada a la API pública de precios no pudo completarse por una respuesta 429; no inventamos una cotización con ese resultado. Las páginas públicas de precios y la calculadora son las referencias para cerrar el presupuesto. No consultamos una factura actual. [S5][S7][S8]

**Fórmula reproducible:** costo mensual = cómputo facturable + almacenamiento PostgreSQL y backups + almacenamiento y operaciones Blob + operaciones Key Vault + logs + pipelines y licencias aplicables + registro de imágenes + transferencia. Los créditos promocionales se descuentan después; no cambian el costo bruto de la arquitectura.

Container Apps publica un tramo gratuito mensual de 180 000 vCPU segundos, 360 000 GiB segundos y dos millones de solicitudes por suscripción en Consumption. No significa que toda la solución sea gratis: PostgreSQL, archivos, red y otros cargos se calculan por separado. Esos límites pueden estar compartidos con otras aplicaciones de la misma suscripción. [S5]

**Controles propuestos:** revisar consumo después de cada incorporación, mantener un entorno, acotar réplicas y logs, no realizar cargas masivas, eliminar recursos temporales solo tras verificar que no contienen datos necesarios y revisar los componentes que siguen cobrando cuando se detiene cómputo. Las alertas de presupuesto avisan, pero no paran automáticamente el consumo. La gratuidad de Pipelines depende de elegibilidad y asignación efectiva, y no está verificada para nuestro proyecto. [S8][S9]

### 3 Proyección de usuarios

**Respuesta para decir:** “Primero validaremos con diez usuarios piloto. Si completan registros sin asistencia y regresan a usar la aplicación, propondremos cincuenta y después doscientos. Son objetivos de adopción; todavía no hemos medido capacidad multiusuario”.

| Etapa propuesta desde el inicio del piloto | Usuarios activos mensuales objetivo | Acciones mensuales del modelo | Solicitudes API estimadas |
|---|---:|---:|---:|
| Primer mes | 10 | 9 000 | 72 000 |
| Meses dos a tres | 50 | 45 000 | 360 000 |
| Meses cuatro a seis | 200 | 180 000 | 1 440 000 |

Supuestos, no uso observado: 30 acciones por usuario por día, 30 días y ocho solicitudes API por acción. Una acción es una tarea como preparar o confirmar un registro, no un movimiento contable ni una llamada al modelo IA. Fórmulas: acciones = usuarios × 30 × 30; solicitudes = acciones × 8. No equivalen a usuarios simultáneos. Antes del piloto público hacen falta login, autorización por negocio y PostgreSQL compartido.

Para ilustrar sensibilidad, si cada solicitud mantuviera una única réplica de 0.25 vCPU y 0.5 GiB durante 0.2 segundos, el modelo arrojaría 3 600, 18 000 y 72 000 vCPU segundos, respectivamente, y 7 200, 36 000 y 144 000 GiB segundos. No predice una factura: concurrencia, arranques, solicitudes que se solapan, background y tiempo mínimo de réplica cambian el cálculo. Ocho peticiones por acción y 0.2 segundos son hipótesis por medir, no resultados del sistema. [S5]

Si diez por ciento de acciones generaran un archivo de 0.5 MiB, se agregarían aproximadamente 0.44, 2.20 y 8.79 GiB al mes por etapa. Es acumulación mensual antes de retención y borrado. Este escenario es para la futura fase cloud: el producto local actual limita adjuntos a 25 MiB por negocio, de modo que requeriría revisar cuotas y almacenamiento antes de implementarlo.

Mediremos finalización de la primera operación, usuarios que vuelven semanalmente, propuestas corregidas, tiempo hasta confirmar y fallos por operación. Las pruebas de carga posteriores evaluarán concurrencia, latencia p95, errores y consistencia del stock. No prometemos soportar 200 usuarios simultáneos ni una disponibilidad que no medimos.

### 4 Por qué Azure encaja mejor que Vercel en este proyecto

**Respuesta para decir:** “Vercel es una alternativa fuerte para Next.js y simplifica publicar la interfaz. Elegimos Azure para integrar cómputo, datos, identidades, secretos y observabilidad dentro de la arquitectura cloud del proyecto. La decisión depende del alcance, no de que Vercel sea incapaz”.

Vercel ofrece despliegues y funcionalidades de backend. No sería correcto reducirlo a hosting estático o decir que no tiene seguridad ni observabilidad. Su plan Hobby se orienta a uso personal no comercial, por lo que un negocio comercial necesita revisar otro plan y condiciones. La página pública consultada anuncia Pro desde USD 20 al mes con consumo y condiciones adicionales; no equivale al costo total de una arquitectura con base de datos y almacenamiento. [S10][S11]

El SQLite local de esta app necesita persistencia estable. Vercel explica que el almacenamiento efímero de funciones serverless no funciona como un archivo SQLite central y permanente. Container Apps tampoco convierte su disco local en una base compartida y durable por sí mismo. En ambos casos, nuestra solución publicada debe migrar la persistencia y los adjuntos a servicios adecuados, no subir sin cambios el modo local. [S12]

Podríamos usar Vercel para el frontend y Azure para la API y los datos, pero introduciría dos plataformas operativas y coordinación de autenticación, red y despliegues. Para este equipo preferimos empezar con un proveedor. Revisaríamos la decisión si la rapidez de publicación de Next.js fuera la prioridad dominante y el alcance de servicios fuera menor.

### 5 Justificación de IAM

**Respuesta para decir:** “IAM significa gestionar quién puede hacer qué sobre cada recurso. En Azure lo aplicamos con Microsoft Entra ID para la identidad, Azure RBAC para los permisos e identidades administradas para el código. Queremos que el backend acceda solo a sus archivos y secretos sin guardar contraseñas Azure en el repositorio”. [S13][S14]

No contratamos AWS IAM: es el nombre del servicio equivalente en AWS. Tampoco contamos IAM, RBAC e identidad administrada como tres servicios diferentes de la lista del curso.

| Actor | Permiso configurado o previsto | Justificación |
|---|---|---|
| Usuario de desarrollo | Colaborador de datos de Storage Blob | Probar escritura y lectura de archivos del proyecto |
| Usuario que administra secretos | Agente de secretos de Key Vault | Crear y administrar secretos del proyecto |
| Identidad id-cloud-app-dev | Colaborador de datos de Storage Blob | Operaciones de archivos que necesite el backend |
| Identidad id-cloud-app-dev | Usuario de secretos de Key Vault | Leer secretos sin administrarlos |
| Pipeline futuro | Roles acotados al despliegue | Evitar permisos globales y preferir federación |

Los roles de datos son distintos de administrar el recurso. Antes de producción debemos revisar el ámbito: si el acceso actual está a nivel de cuenta o vault, reducirlo a lo necesario y separar entornos. Hoy las conexiones reales del backend usaron la sesión Azure CLI del desarrollador. La identidad administrada está creada y tiene roles, pero falta vincularla y probarla en Container Apps. La autenticación de infraestructura no reemplaza el login del producto ni la autorización por membresía de negocio, todavía pendientes. [S13][S14]

## Por qué estos siete servicios y no otras opciones

| Servicio | Problema que resuelve en la app | Alternativa considerada | Decisión y costo de la decisión |
|---|---|---|---|
| Container Apps | Ejecutar frontend y backend en contenedor | App Service, VM, AKS o Vercel | Elegimos despliegue administrado de contenedor. Una VM exige más operación. AKS añade una complejidad que no justificamos con el tamaño actual. App Service también sería válido. |
| Blob Storage | Conservar comprobantes e imágenes fuera del proceso | Archivos en disco, base de datos o Azure Files | El almacenamiento de objetos separa archivos del cómputo. En local usamos SQLite para un volumen pequeño. Blob implica diseñar autorización, metadatos y borrado coherente. |
| PostgreSQL | Relaciones y transacciones de negocios, movimientos y stock | SQLite, Azure SQL o Cosmos DB | El dominio es relacional. PostgreSQL ofrece portabilidad de motor; migrar SQL y acceso es trabajo pendiente. SQLite sirve para el MVP local. No necesitamos por ahora un modelo distribuido documental. |
| Entra e identidad administrada | Identidad del backend y permisos de acceso | Claves compartidas o credenciales manuales | Reducimos secretos de infraestructura en código. Requiere configurar roles y sus ámbitos y probarlos. |
| Monitor | Diagnosticar errores y rendimiento | Consola local u otra plataforma de observabilidad | Integra señales del entorno Azure. Falta implementarlo y controlar cuánto log retenemos. |
| Pipelines | Validar y desplegar cambios reproducibles | GitHub Actions o despliegue manual | Elegido en la lista académica y para demostrar CI/CD. GitHub Actions también sería razonable y puede reducir plataformas del equipo. Debemos verificar el costo y concesión de agentes antes de activarlo. |
| Key Vault | Controlar secretos que la aplicación necesite | .env de desarrollo, secretos del host o claves en código | Separa administración de secretos. La prueba actual solo lee un secreto de conectividad; falta un caso funcional real que lo requiera. No inventaremos secretos innecesarios solo para contar un servicio. |

Estos servicios forman una propuesta académica de siete componentes. El curso no convierte siete en un mínimo técnico de cualquier MVP. Para un producto pequeño podrían bastar menos piezas, y deberíamos retirarlas si su beneficio no supera el costo operativo. Un registro de imágenes como ACR sería auxiliar y no está contado entre los siete.

## Preguntas adicionales para defender las decisiones

### 6 Qué problema resolvemos y para quién

Ayudamos a micro y pequeñas empresas guatemaltecas a construir registros diarios y una visión de su negocio sin aprender primero un sistema complejo. El usuario cuenta una operación, revisa la propuesta y la confirma. La adecuación a este público es una hipótesis de producto que debemos validar con personas reales; no presentamos estadísticas de adopción que no investigamos.

### 7 Qué funciona hoy y qué sigue pendiente

Funciona el MVP local con varios negocios del mismo usuario, propuestas revisables, ventas, gastos, compras, inventario, perfiles, autoevaluaciones, planes, aprendizaje y adjuntos reales locales. El backend probó escritura y recuperación de un comprobante sintético en Blob y lectura de un secreto en Key Vault usando Azure CLI. Faltan despliegue público, PostgreSQL, identidad administrada en ejecución, Monitor, pipeline ejecutado e identidad multiusuario. Son dos flujos todavía separados: negocio local y demo cloud.

### 8 Qué evidencia tenemos

El repositorio conserva evidencias/conexion-azure-real-2026-10-19.json. Registra Blob y Key Vault aprobados, sin valor del secreto. La revisión local más reciente aprobó 29 comprobaciones del producto, incluyendo navegador, y 13 regresiones HTTP. El rechazo HTTP 401 sin token fue el resultado esperado de una prueba de acceso. Estas pruebas no certifican seguridad completa ni rendimiento de producción.

### 9 Por qué el usuario confirma antes de guardar

Una frase puede ser ambigua o contener varias operaciones. Se conserva la propuesta y se permite corregirla. La confirmación aplica el conjunto en una transacción, evita repetición y rechaza stock insuficiente. La IA futura tendrá que respetar el mismo contrato: proponer datos y no escribir operaciones por su cuenta.

### 10 Ya usamos inteligencia artificial generativa

No. La interpretación y conversación actuales usan reglas. El nombre conceptual PROPUESTA_IA expresa la intención del diseño, no prueba una integración de modelo. Antes de añadirlo evaluaremos exactitud, costos por tokens, privacidad, formatos estructurados, corrección humana y manejo de fallos. El presupuesto de este documento excluye IA y OCR.

### 11 Por qué no calculamos utilidad ni riesgo fiscal automáticamente

La diferencia de ventas menos gastos y compras no incluye todas las reglas contables, costos de venta, crédito, impuestos o depreciación. Presentarla como utilidad daría información engañosa. La autoevaluación de formalidad registra una percepción y no sustituye asesoría fiscal.

### 12 Cómo protegemos datos de negocios diferentes

Cada operación local valida que el negocio sea accesible y que los productos o movimientos referidos pertenezcan a ese negocio. Probamos rechazos de referencias cruzadas. Esto separa negocios del mismo usuario local; todavía falta autenticar usuarios reales, autorizar membresías y probar aislamiento entre cuentas antes de publicar.

### 13 Por qué no guardar todos los archivos en la base para siempre

Lo hacemos provisionalmente con adjuntos pequeños y cuotas explícitas para avanzar sin consumo cloud. Base64 aumenta tamaño y las consultas y respaldos podrían encarecerse con volumen. La fase compartida debe separar binarios en Blob y metadatos en PostgreSQL, con reintentos y estados para evitar archivos huérfanos. Aún no implementamos ese cambio.

### 14 Qué pasa si falla Azure o la red

La captura local del negocio sigue funcionando sin llamadas Azure. La demo cloud informa el error; no debe presentar como guardado algo que no confirmó el servicio. No hemos implementado sincronización offline con resolución de conflictos, colas ni un SLA propio. Para publicar necesitaremos definir reintentos, idempotencia y recuperación.

### 15 Cómo respaldamos y recuperamos

Guardamos un respaldo consistente antes de la migración local de adjuntos y verificamos conservación de movimientos. Eso no reemplaza una política de respaldo y restauraciones periódicas. El perfil exportado en JSON no es copia completa de SQLite ni contiene los binarios adjuntos. En cloud habrá que fijar y probar objetivos RPO y RTO, hoy sin garantía definida.

### 16 Por qué siete servicios no significan restricción automática

No tenemos evidencia del incidente del compañero. Puede involucrar cuotas, límites de API, verificación, políticas o consumo facturable. No concluimos que crear cinco servicios cause por sí solo un bloqueo. Nuestro enfoque es desplegar por etapas, verificar cuotas y estado, medir consumo y revisar errores concretos antes de aumentar recursos.

### 17 Qué significa escalar y cuándo lo haríamos

Primero mediremos latencia p95, errores, uso CPU/memoria, conexiones de base y carga real. Escalar réplicas sin optimizar consultas o pool de conexiones puede trasladar el cuello de botella a PostgreSQL. El número de usuarios registrados por sí solo no determina capacidad ni costo.

### 18 Por qué no elegimos Kubernetes ni microservicios ahora

El alcance cabe inicialmente en una aplicación con módulos separados. Un clúster, múltiples redes y despliegues independientes añadirían operación sin demostrar necesidad actual. Container Apps no obliga a fragmentar el dominio en varios microservicios. Revisaremos la separación si aparecen equipos o cargas con necesidades independientes.

### 19 Cuánto dependemos de Azure

El frontend y la lógica TypeScript son transferibles y PostgreSQL es un motor común. Los adaptadores, configuración de identidad, permisos, observabilidad y despliegue dependen del proveedor y necesitan trabajo al migrar. Portabilidad no significa costo cero de cambio. Docker y contratos claros ayudan, pero no eliminan esa dependencia.

### 20 Cómo elegiremos la región

East US fue la región del grupo y los recursos creados. No medimos aún latencia desde Guatemala ni requisitos específicos de residencia de datos. Antes de producción compararemos disponibilidad de SKU, precio, latencia real, ubicación de usuarios y obligaciones aplicables. No afirmamos que sea la región más rápida ni que cumpla automáticamente requisitos legales.

### 21 Cómo evaluaremos el éxito del piloto

Veremos si el usuario puede completar una operación, corregir una propuesta y consultar su inventario sin asistencia. Mediremos retorno semanal y errores. Si se requiere demasiada explicación, priorizaremos simplificar el flujo antes de sumar servicios. Una meta inicial propuesta, aún sin validar, es que al menos ocho de diez participantes completen el primer registro acompañados solo por instrucciones de la app.

### 22 Qué haríamos si el presupuesto se supera

Revisar el componente que crece y su unidad facturable, no apagar recursos a ciegas. Reducir retención o frecuencia cuando sea seguro, investigar archivos o logs excesivos y decidir si el piloto necesita menos capacidad. El presupuesto de alerta no es un techo de cobro. No se activa pago por uso ni recursos adicionales sin revisar sus condiciones y el alcance autorizado. [S9]

### 23 Qué errores debemos evitar al exponer

No decir que Azure siempre gana a AWS o GCP. No afirmar que Vercel solo aloja frontend. No confundir crédito con costo cero permanente, dos contenedores con dos servicios, identidad creada con identidad en uso, una cuota gratuita con capacidad ilimitada, un test 401 esperado con una falla de seguridad ni una proyección de usuarios con clientes reales.

### 24 Qué nos falta para ofrecerlo como producto público

Autenticación y recuperación de cuentas, autorización de membresías, migración a PostgreSQL, binarios persistentes, dominio y HTTPS del despliegue, monitoreo, despliegue reproducible, backups restaurables, límites y política de tratamiento de datos. Además necesitamos validar el flujo con usuarios y decidir el alcance de la IA. El estado actual es un prototipo funcional, no una plataforma productiva completa.

## Resumen de un minuto para la defensa

“Creamos un asistente de registro para negocios guatemaltecos. El usuario cuenta su día, revisa una propuesta y confirma antes de actualizar dinero e inventario. El MVP funciona localmente y ya admite comprobantes reales. Elegimos Azure por acceso viable y una ruta integrada de servicios; AWS, GCP y Vercel son alternativas válidas según el alcance. Probamos el backend con Blob Storage y Key Vault usando Entra a través de Azure CLI. La identidad administrada está preparada, pero su ejecución en Container Apps sigue pendiente. Para un piloto futuro proponemos diez usuarios y una reserva mensual de USD 20 a 80, sin IA y sujeta a cotización y medición. No prometemos capacidad ni disponibilidad sin pruebas”.

## Fuentes oficiales y evidencia

Fuentes consultadas el 19/10/2026. Las estimaciones internas, objetivos de adopción y juicios de elección se identifican como tales. Verificar precios y condiciones antes de contratar.

- [S1 Cloud Run y su alcance](https://docs.cloud.google.com/run/docs/overview/what-is-cloud-run)
- [S2 AWS App Runner](https://aws.amazon.com/apprunner/pricing/)
- [S3 Cloud SQL y Secret Manager](https://docs.cloud.google.com/sql/docs/postgres/use-secret-manager)
- [S4 RDS y Secrets Manager](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-secrets-manager.html)
- [S5 Precios de Azure Container Apps](https://azure.microsoft.com/en-us/pricing/details/container-apps/)
- [S6 Capacidad y límites de PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/compute-storage/concepts-compute)
- [S7 Precio de PostgreSQL](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/)
- [S8 Asignaciones y cobro de Azure Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/licensing/concurrent-jobs?view=azure-devops)
- [S9 Presupuestos Azure](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets)
- [S10 Plan Hobby Vercel](https://vercel.com/docs/plans/hobby)
- [S11 Precios Vercel](https://vercel.com/pricing)
- [S12 SQLite en Vercel](https://vercel.com/kb/guide/is-sqlite-supported-in-vercel)
- [S13 Azure RBAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [S14 Identidades administradas](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)
- [Calculadora Azure para cerrar la cotización](https://azure.microsoft.com/en-us/pricing/calculator/)
- [Evidencia real de conexiones](https://github.com/lizardoenm12/semilla-cloud/blob/main/evidencias/conexion-azure-real-2026-10-19.json)
- [Pruebas del producto](https://github.com/lizardoenm12/semilla-cloud/blob/main/evidencias/pruebas-workspace.json)
