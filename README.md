# Grupo 7: Módulo de Reportería, Batch y Streaming

Este servicio se encarga de consolidar, procesar y exponer las métricas operacionales del ecosistema Mini Marketplace Cloud utilizando estrategias mixtas de transferencia de información.

## Decisiones de Arquitectura e Infraestructura (E1)
* **Ingesta en Tiempo Real (Streaming):** Captura de eventos críticos de negocio mediante colas asíncronas para actualizar el caché analítico de inmediato.
* **Procesamiento por Lotes (Batch):** Tareas programadas nocturnas (vía GitHub Actions / Cron) que leen logs crudos persistidos en Object Storage, recalculan consistencias analíticas y guardan datos consolidados en la base SQL.

## Matriz de Dependencias (Punto 5 de la Rúbrica)
Para operar con éxito, nuestro módulo requiere consumir información proveniente de los siguientes equipos del ecosistema:

1. **Grupo 5 (Pedidos):** Consumimos el evento `OrderCreated` para graficar flujos de venta en tiempo real.
2. **Grupo 6 (Pago simulado):** Consumimos `PaymentApproved` para actualizar los montos financieros reales recaudados.
3. **Grupo 7 (Inventario):** Consumimos `InventoryShortage` para alertar en el Dashboard qué productos se están quedando sin stock.
4. **Grupo 1 (Frontend / BFF):** Consume nuestra API REST expuesta en `/reports/sales-summary` para pintar los gráficos del administrador.

## Contrato de la API
El contrato formal e interactivo se encuentra detallado en el archivo adjunto [openapi.yaml](./openapi.yaml).

## Contrato de Eventos a Consumir (Punto 4 de la Rúbrica)
Dado que operamos bajo un enfoque analítico mixto (Streaming + Batch), nuestro servicio es un consumidor puro de eventos del ecosistema. A continuación se formalizan los esquemas JSON estructurales que el sistema procesará a través de la cola asíncrona (Upstash Kafka / GCP PubSub):

### 1. Evento: `OrderCreated` (Origen: Grupo 5 - Pedidos)
Este evento gatilla el procesamiento en tiempo real (Streaming) para actualizar los tableros analíticos de órdenes vigentes de inmediato.
```json
{
  "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
  "title": "OrderCreatedEvent",
  "type": "object",
  "required": ["event_id", "event_type", "version", "timestamp", "payload"],
  "properties": {
    "event_id": { "type": "string", "example": "evt_102938475" },
    "event_type": { "type": "string", "enum": ["OrderCreated"] },
    "version": { "type": "string", "example": "1.0.0" },
    "timestamp": { "type": "string", "format": "date-time", "example": "2026-06-13T06:00:00Z" },
    "payload": {
      "type": "object",
      "required": ["order_id", "user_id", "total_amount", "currency"],
      "properties": {
        "order_id": { "type": "string", "example": "ord_998877" },
        "user_id": { "type": "string", "example": "usr_abc123" },
        "total_amount": { "type": "number", "example": 45500.00 },
        "currency": { "type": "string", "example": "CLP" }
      }
    }
  }
}

### 2. Evento: `PaymentApproved` (Origen: Grupo 6 - Pagos)
Indica dinero real recaudado. Permite consolidar los reportes de ingresos financieros netos.
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "PaymentApprovedEvent",
  "type": "object",
  "required": ["event_id", "event_type", "version", "timestamp", "payload"],
  "properties": {
    "event_id": { "type": "string", "example": "evt_554433221" },
    "event_type": { "type": "string", "enum": ["PaymentApproved"] },
    "version": { "type": "string", "example": "1.0.0" },
    "timestamp": { "type": "string", "format": "date-time", "example": "2026-06-13T06:05:00Z" },
    "payload": {
      "type": "object",
      "required": ["payment_id", "order_id", "amount_paid", "payment_method"],
      "properties": {
        "payment_id": { "type": "string", "example": "pay_776655" },
        "order_id": { "type": "string", "example": "ord_998877" },
        "amount_paid": { "type": "number", "example": 45500.00 },
        "payment_method": { "type": "string", "example": "CREDIT_CARD" }
      }
    }
  }
}

## Modelo de Datos Inicial (Ownership del Dato)
Para persistir la información consolidada por los flujos analíticos en la base de datos SQL relacional (Supabase Postgres / Neon), definimos las siguientes entidades estructuradas:

### 1. Tabla: `fact_sales_summary` (Métricas de ventas agregadas por hora)
* **id** (UUID, PK): Identificador único del registro analítico.
* **period_date** (TIMESTAMP): Fecha y hora del bloque consolidado.
* **total_sales_amount** (NUMERIC): Suma acumulada de montos financieros validados.
* **total_orders_count** (INTEGER): Cantidad total de órdenes procesadas con éxito.
* **aggregation_type** (VARCHAR): Origen del cálculo (`REAL_TIME` o `BATCH_RECALCULATED`).
* **updated_at** (TIMESTAMP): Última actualización del registro.

### 2. Tabla: `agg_top_products` (Ranking acumulado de productos)
* **product_id** (VARCHAR, PK): ID del producto (Consultado del catálogo de G3).
* **total_units_sold** (INTEGER): Cantidad física acumulada de unidades vendidas.
* **total_revenue_generated** (NUMERIC): Dinero neto generado por este ítem.
* **last_calculated_at** (TIMESTAMP): Marca de tiempo del último procesamiento.
