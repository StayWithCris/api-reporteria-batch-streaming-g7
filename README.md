# Grupo 7: Módulo de Reportería, Batch y Streaming

Este servicio se encarga de consolidar, procesar y exponer las métricas operacionales del ecosistema Mini Marketplace Cloud utilizando estrategias mixtas de transferencia de información.

## 🛠️ Decisiones de Arquitectura e Infraestructura (E1)
* **Ingesta en Tiempo Real (Streaming):** Captura de eventos críticos de negocio mediante colas asíncronas para actualizar el caché analítico de inmediato.
* **Procesamiento por Lotes (Batch):** Tareas programadas nocturnas (vía GitHub Actions / Cron) que leen logs crudos persistidos en Object Storage, recalculan consistencias analíticas y guardan datos consolidados en la base SQL.

## 🔗 Matriz de Dependencias (Punto 5 de la Rúbrica)
Para operar con éxito, nuestro módulo requiere consumir información proveniente de los siguientes equipos del ecosistema:

1. **Grupo 5 (Pedidos):** Consumimos el evento `OrderCreated` para graficar flujos de venta en tiempo real.
2. **Grupo 6 (Pago simulado):** Consumimos `PaymentApproved` para actualizar los montos financieros reales recaudados.
3. **Grupo 7 (Inventario):** Consumimos `InventoryShortage` para alertar en el Dashboard qué productos se están quedando sin stock.
4. **Grupo 1 (Frontend / BFF):** Consume nuestra API REST expuesta en `/reports/sales-summary` para pintar los gráficos del administrador.

## 📄 Contrato de la API
El contrato formal e interactivo se encuentra detallado en el archivo adjunto [openapi.yaml](./openapi.yaml).
