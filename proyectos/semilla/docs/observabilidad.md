# Azure Monitor en Asistente Negocios

Configuración de desarrollo del 22 de octubre de 2026.

## Arquitectura y límites

El servidor escribe eventos JSON técnicos en stdout/stderr. El entorno `cae-cloud-dev` envía únicamente consola y sistema a Azure Monitor mediante diagnostic settings. El destino es `log-cloud-dev`, Central US, PerGB2018, con 30 días de retención y salvaguarda de 0.025 GB diarios. No se habilitaron logs HTTP de ingreso, Application Insights, agentes ni un compromiso de capacidad.

La salvaguarda es una protección frente a picos, no un límite exacto de factura. Puede ingerirse volumen adicional antes de que el corte tenga efecto y, si se alcanza, dejan de llegar eventos hasta el reinicio diario. Se reduce el volumen desde el código: sin logs de cada health probe y sin cuerpos ni rutas sin normalizar.

## Datos registrados

- Arranque de una versión y SHA del código.
- Método, plantilla de ruta, código HTTP, duración y correlación aleatoria de peticiones API.
- Errores no controlados con categoría técnica y estado 500, sin copiar el mensaje de excepción.

No se registran cuerpos, query strings, cookies, códigos OAuth, nombres, identificadores del negocio, IP ni contraseñas en los eventos propios. El identificador de correlación lo genera el servidor. El framework y la plataforma pueden añadir sus propios mensajes técnicos a los logs de consola y sistema.

`src/instrumentation.ts` registra los arranques y errores no controlados. `src/lib/observability.ts` instrumenta las API de negocio y autenticación. La variable `OBSERVABILITY_ENABLED=1` se configura al desplegar. `APP_RELEASE` contiene el SHA completo. La salud devuelve esa versión pública sin consultar base, secretos o archivos.

## Consultas y regla

Las consultas editables en `infra/queries` se guardan también en el workspace:

- `errores.kql`: eventos de error del servidor.
- `rendimiento.kql`: número de peticiones, errores y p95 por plantilla de ruta en intervalos de 15 minutos.
- `versiones.kql`: arranques por versión.

Las consultas admiten el prefijo de tiempo/stream de los logs del contenedor antes del JSON. La alerta `alert-asistente-errores-dev` evalúa cada 15 minutos la ventana de 15 minutos y detecta cualquier evento propio de nivel error. Utiliza identidad administrada con permiso de lectura sobre este workspace. No despierta la aplicación para ejecutar la consulta. Una respuesta 401 esperada es una advertencia y no activa la alerta de errores.

El grupo `ag-asistente-dev` tiene el destinatario autorizado habilitado y esquema común. Las alertas quedan en Azure Monitor y el grupo se encarga del correo. La dirección se conserva en `.data/private/monitor-contacto.json`, fuera de Git. El usuario confirmó la recepción del aviso de incorporación al grupo.

Azure rechazó el endpoint específico de notificaciones de prueba con `Free subscription not supported`. Se conservó Free Trial. No se presentan esa prueba ni el aviso de incorporación como recepción confirmada de una alerta real. No se crearon errores artificiales en las operaciones del negocio para forzar un incidente.

## Verificación real

El 22/10 Log Analytics devolvió un arranque de la versión 5bca0110 y una petición a bootstrap rechazada con 401 sin sesión, duración 2 ms. La consulta de errores no devolvió eventos. Esta muestra verifica ingesta y campos, no capacidad, latencia representativa ni entrega de alertas por incidente. Evidencia agregada en `evidencias/monitor-real.json`.

## Cómo verlo en Azure

1. Abrir `rg-cloud-dev` y el workspace `log-cloud-dev`.
2. En Logs ejecutar las consultas guardadas de Asistente Negocios.
3. En Monitor, Alertas, Reglas de alerta abrir `alert-asistente-errores-dev`.
4. En Grupos de acciones revisar `ag-asistente-dev`.

Los eventos pueden tardar varios minutos en ser consultables. El stream de Container Apps permite comprobar la emisión en tiempo real. Una tabla vacía inmediatamente después de desplegar no acredita aún la ingesta persistente.

## Proyección adicional

Tarifas oficiales consultadas para Central US: Analytics Logs USD 2.76 por GB y alerta con frecuencia de 15 minutos USD 0.50 por mes. Como referencia, 0.025 GB por 30 días equivale a 0.75 GB, aproximadamente USD 2.57 con la regla, antes de beneficios gratuitos, impuestos, notificaciones y posibles excesos. No se asume un beneficio gratuito disponible ni que la salvaguarda sea un tope monetario. Evidencia de precios en `evidencias/precios-monitor-centralus.json`.

## Operación reproducible

- `python scripts/configurar-operacion.py monitor`: workspace y destino de logs.
- `python scripts/monitor-alertas.py configure`: validar consultas, guardarlas y configurar regla y grupo con el contacto privado.
- `python scripts/monitor-alertas.py query`: consultar versiones.
- `npm.cmd run test:monitor`: comprobar campos, normalización y ausencia de datos sensibles sin Azure.

No ejecutar pruebas de notificación repetidamente. No cambiar el plan de facturación para habilitar ese endpoint.

## Fuentes

- https://learn.microsoft.com/en-us/azure/container-apps/log-options
- https://learn.microsoft.com/en-us/azure/azure-monitor/logs/daily-cap
- https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups
- https://learn.microsoft.com/en-us/answers/questions/5566446/bug-cant-test-action-group-the-error-message-is-em
- https://prices.azure.com/api/retail/prices
