# Costos, crédito de prueba y cierre de Asistente Negocios

Actualizado: 22 de octubre de 2026, Guatemala. Todos los importes se expresan en USD, antes de impuestos. No representan una factura garantizada.

## Estrategia acordada

Aprovechar el crédito Azure Free Trial para implementar y demostrar los servicios del curso por etapas. No aumentar recursos para agotar el crédito, no retirar el límite de gasto y no convertir la suscripción a pago por uso sin autorización expresa. Antes de una pausa prolongada, exportar datos y conservar el código y la configuración necesaria para reconstruir el entorno.

El crédito anunciado de USD 200 dura hasta 30 días desde el alta o hasta consumirse, lo que ocurra primero. No equivale a un saldo renovable cada mes. El saldo restante y la fecha exacta de vencimiento todavía no se han verificado. Que el proyecto cueste menos de USD 200 al mes no garantiza disponibilidad después del vencimiento de la prueba.

La política de la suscripción se consultó en esta actualización: `FreeTrial_2014-09-01`, `spendingLimit: On`. Esto no configura un límite personalizado de USD 30. No se modificó el plan ni la facturación.

## Proyección para presentar

| Concepto | Situación actual | Proyección mensual | Supuesto |
| --- | --- | ---: | --- |
| PostgreSQL Flexible Server B1ms | Implementado, con datos y sesiones reales | 18.18 | 730 h a 0.01921/h + 32 GiB a 0.13/GiB-mes; Central US. |
| Container Apps Consumption | Publicado en Consumption | 0–5 | Pocas visitas, mínimo 0 y máximo 1 réplica configurados, dentro o cerca de las cuotas gratuitas. No aplica a carga continua. |
| Blob, Vault, ACR, Monitor y margen | Todos integrados; ingesta y despliegue verificados | 5–10 | Reserva de planificación conjunta, no tarifa cotizada ni gasto ya medido. Depende de almacenamiento, solicitudes, egreso y retención. |
| Total aritmético del escenario | Servicios integrados | 23.18–33.18 | Suma de las estimaciones anteriores, no incluye cualquier servicio adicional futuro. |
| Reserva de planificación redondeada | Propuesta | 25–35 | Margen orientativo para pruebas manuales y un entorno pequeño. No es tope de facturación. |

PostgreSQL: cálculo detallado y respuesta de precios oficiales en `evidencias/precios-postgresql-centralus.json`. Cómputo: 14.0233; almacenamiento: 4.16; total: 18.1833 por 730 horas. El costo real de un mes parcial se ajustará al tiempo y uso facturados; no asumir un beneficio gratuito de PostgreSQL no verificado.

Container Apps incluye por suscripción y mes 180,000 vCPU-segundos, 360,000 GiB-segundos y 2 millones de solicitudes HTTP. Como ejemplo ilustrativo, una sola réplica configurada con 0.5 vCPU y 1 GiB consume ambas cuotas de cómputo en aproximadamente 100 horas de ejecución acumulada si ninguna otra aplicación las utiliza. El tamaño actual es 0.5 vCPU y 1 GiB. La capacidad con carga aún debe medirse. No confundir horas de visitas con horas de ejecución: hay arranque, enfriamiento y solicitudes adicionales.

Azure Monitor: tarifas conservadas en `evidencias/precios-monitor-centralus.json`, USD 2.76/GB y regla de 15 minutos a USD 0.50/mes. Un ejemplo de 0.025 GB diarios por 30 días suma USD 2.57 con la regla, incluido como planificación dentro de la reserva conjunta de USD 5–10. No asume beneficios gratis. El límite de ingesta puede sobrepasarse y no fija una factura máxima. GitHub Actions usa cuotas y facturación propias, separadas del crédito Azure.

## Gasto reportado y crédito

Consulta real realizada sobre `rg-cloud-dev`, período mes actual hasta la fecha: **USD 0.00008622** reportados antes de impuestos, es decir, menos de USD 0.01. La fecha exacta de consulta está en `evidencias/costos-observados.json` (UTC).

Ese valor tiene retraso de procesamiento y no refleja necesariamente todos los consumos recientes de PostgreSQL. Es únicamente el grupo del proyecto; no es el total de la suscripción, no es un débito confirmado a la tarjeta y no permite afirmar que queden USD 199.9999 de crédito. El crédito aplicado y saldo se verifican en la sección de facturación de la cuenta.

## Avisos propuestos, todavía no configurados

Presupuesto mensual de referencia: USD 30, avisos de costo real al 50%, 80% y 100% (USD 15, 24 y 30). Puede añadirse aviso de previsión al 100%. Elegir alcance del grupo `rg-cloud-dev` para este proyecto y revisar también el consumo de toda la suscripción si contiene otros proyectos.

Los presupuestos avisan y tienen retraso; no suspenden recursos automáticamente. Máximo una réplica limita escalado, no fija un tope en dólares. Si la previsión supera USD 30, revisar antes de continuar. Falta acordar destinatario y crear/verificar el presupuesto; no afirmar en la presentación que esas alertas ya están activas.

## Plan para pausar al terminar la prueba

1. Verificar fecha de vencimiento del crédito y la fecha de la presentación. Conservar `spendingLimit: On`, sin pasar a pago por uso automáticamente.
2. Exportar una copia reciente de PostgreSQL y comprobar que se puede restaurar. La antigua SQLite anterior a la migración no contiene las operaciones nuevas y no sustituye ese respaldo.
3. Descargar los comprobantes Blob que deban conservarse y guardar la configuración de forma privada; no colocar secretos ni respaldos con datos personales en Git.
4. Conservar código, documentación, evidencias y definición reproducible de infraestructura. Preparar el procedimiento de recreación antes de eliminar recursos.
5. Al confirmar el cierre, eliminar los recursos facturables específicos del proyecto o cancelar la suscripción dedicada, según lo que contenga. Revisar dependencias, retenciones y costos residuales. No eliminar una suscripción con otros proyectos sin revisar su alcance.
6. Comprobar después el estado de los recursos y el reporte de costos con su retraso habitual.

**No se canceló, detuvo ni eliminó nada en esta actualización.** Detener PostgreSQL solo suspende el cómputo: el almacenamiento se cobra y el servidor se reinicia automáticamente tras siete días. Para una pausa larga no basta con pulsar Detener. Escalar Container Apps a cero tampoco suspende PostgreSQL, Blob u otros recursos.

## Texto breve para exponer

> Nuestro escenario de desarrollo se estima en USD 25–35 mensuales antes de impuestos, con PostgreSQL como principal costo fijo. Aprovechamos el crédito de prueba de Azure sin retirar el límite de gasto. La aplicación tiene escalado a cero y un máximo pequeño de réplicas. Diferenciamos proyección y consumo reportado; antes de finalizar la prueba respaldaremos datos y retiraremos los recursos que no se necesiten. Los avisos de presupuesto se han propuesto, pero todavía no están configurados.

## Fuentes

- Facturación Container Apps y cuotas: https://learn.microsoft.com/en-us/azure/container-apps/billing
- Presupuestos y sus límites: https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-acm-create-budgets
- Crédito gratuito y vencimiento: https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/avoid-charges-free-account
- Precios PostgreSQL y almacenamiento detenido: https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/
- Inicio/parada PostgreSQL y reinicio a los siete días: https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-stop-start-server-portal
