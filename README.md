# UNIVERSIDAD TECNOLÓGICA METROPOLITANA
## Facultad de Ingeniería - Departamento de Informática y Computación
### Ramo: Arquitectura de Software
### Profesor: Diego Hernández García

---

# Proyecto Mini Marketplace Cloud - Grupo 7
## Módulo: Reportería, Batch y Streaming

### Integrantes:
* Cristóbal Alexis Faúndez Brito
* [Agregar aquí Nombre de Integrante 2]
* [Agregar aquí Nombre de Integrante 3]

---

## 1. Definición del Servicio y Responsabilidad (E1)
Nuestro servicio actúa como el componente analítico y de observabilidad centralizado de la solución Mini Marketplace Cloud. Diseñado bajo el principio arquitectónico de **separación de responsabilidades (Separation of Concerns - SoC)**, este módulo se limita exclusivamente a consolidar, procesar y exponer las métricas operacionales y financieras agregadas del negocio. El fin principal es proveer datos procesados listos para que el componente de interfaz Dashboard/BFF (Grupo 1) pueda renderizarlos al administrador del sistema.

* **Fuera de Alcance:** Queda estrictamente fuera del dominio analítico la mutación transaccional del estado de las compras, la edición directa de stocks en inventario, y la gestión o procesamiento de flujos en pasarelas de pago. El servicio es un consumidor y consolidador puro de eventos, garantizando un bajo acoplamiento con las reglas de negocio transaccionales.

---

## 2. Decisiones de Arquitectura e Ingesta Mixta
Para dar cumplimiento a los requerimientos analíticos a gran escala sin degradar la latencia ni saturar operativamente la base de datos transaccional, se opta por un enfoque híbrido de procesamiento:

* **Estrategia en Tiempo Real (Stream Processing):** Consumo continuo de eventos de alta prioridad (como aprobaciones de pago) desde el broker asíncrono, permitiendo la actualización inmediata de la telemetría operacional en caliente (estructuras de lectura rápida).
* **Estrategia por Lotes (Batch Processing):** Ejecución de tareas programadas (cronjobs automáticos configurados mediante pipelines de GitHub Actions) de baja frecuencia. Estas leen los logs analíticos crudos persistidos en frío dentro del Object Storage, recalculan las agregaciones complejas y mitigan desfases, asegurando el estado de **consistencia eventual** del ecosistema distribuido.

---

## 3. Matriz de Dependencias e Integración (Punto 5 de la Rúbrica)
De acuerdo a la topología orientada a eventos del ecosistema Marketplace, nuestro módulo presenta una dependencia técnica e integración directa con los siguientes grupos productores de datos (*Upstream*):

1. **Grupo 5 (Pedidos / Order Management):** Dependemos del consumo de su evento `OrderCreated` para trazar la intención de compra e iniciar la métrica de volumen analítico de órdenes diarias.
2. **Grupo 6 (Pago Simulado):** Dependemos del consumo de su evento `PaymentApproved` para computar los montos financieros reales recaudados por la plataforma analítica.
3. **Grupo 7 (Inventario y Concurrencia):** Consumimos el evento `InventoryShortage` para alimentar las alertas reactivas de quiebre de stock en el panel analítico del administrador.
4. **Grupo 1 (Frontend Marketplace / BFF):** Actúa como nuestro consumidor directo aguas abajo (*Downstream*), integrándose mediante llamadas REST sincrónicas a nuestra capa analítica expuesta para consultar el estado consolidado de los reportes.

---

## 4. Riesgos, Supuestos y Manejo de Errores (10% Rúbrica E1)
* **Supuesto Clave:** Se asume que los microservicios transaccionales emisores (`G5` y `G6`) garantizan un payload con marcas de tiempo idempotentes y estructuradas, permitiendo mitigar colisiones o reprocesamientos fuera de orden cronológico en la cola asíncrona.
* **Riesgo Técnico (Indisponibilidad o Pérdida de Mensajes):** Caída transitoria del broker de mensajería asíncrona debido a las cuotas restrictivas de los proveedores en capa *Cloud Free*, provocando pérdida de telemetría en tiempo real.
* **Mecanismo de Mitigación:** Se implementará una política de reintentos con retraso exponencial (*Exponential Backoff*) en el consumidor de streaming. Adicionalmente, el diseño del proceso Batch nocturno actuará como failover, reconstruyendo de forma íntegra el estado analítico diario directamente desde los logs crudos respaldados en el Object Storage persistente, resolviendo cualquier pérdida de paquetes previa.

---

## 5. Contrato Formal de la API REST
La especificación de nuestros endpoints REST se encuentra estructurada formalmente bajo el estándar OpenAPI 3.0 dentro del archivo de configuración raíz de este repositorio: [openapi.yaml](./openapi.yaml).

El servicio expone de forma pública y síncrona el siguiente recurso principal para el BFF:
* `GET /api/v1/reports/sales-summary`
  * **Parámetros de entrada:** `period` (daily, weekly, monthly), `page` (paginación de registros históricos), `limit` (control de tamaño de payload).
  * **Estrategia adoptada:** Implementa códigos de estado HTTP estandarizados y esquemas uniformes para el manejo de excepciones y validación de parámetros inválidos.

---

## 6. Contrato de Eventos Analíticos a Consumir (Punto 4 de la Rúbrica)

### Evento: `OrderCreated` (Origen: Grupo 5)
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
```
## Evento: PaymentApproved (Origen: Grupo 6)
{
  "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
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

## 7. Modelo de Datos Inicial (Ownership del Dato)
El esquema de datos inicial implementa un diseño relacional optimizado para analítica (tablas de hechos y dimensiones simplificadas). Este modelo persistirá de manera aislada en el motor relacional provisto por la capa gratuita (Supabase Postgres / Neon DB):

Tabla: fact_sales_summary (Agregaciones operacionales y financieras)
id (UUID, PK): Identificador unívoco autogenerado del registro consolidado.

period_date (TIMESTAMP): Marca temporal que delimita el bloque de agregación analítica.

total_sales_amount (NUMERIC): Sumatoria financiera de los pagos validados y conciliados dentro del rango.

total_orders_count (INTEGER): Volumen de órdenes computadas exitosamente.

aggregation_type (VARCHAR): Indica la estrategia de ingesta aplicada (REAL_TIME o BATCH_RECALCULATED).

updated_at (TIMESTAMP): Timestamp de auditoría para trazar modificaciones de los pipelines.

Tabla: agg_top_products (Ranking de demanda física)
product_id (VARCHAR, PK): ID único del producto (datos administrados originalmente por el catálogo de G3).

total_units_sold (INTEGER): Cantidad total acumulada de unidades vendidas.

total_revenue_generated (NUMERIC): Ingresos brutos consolidados generados exclusivamente por este ítem.

last_calculated_at (TIMESTAMP): Registro temporal del último lote de procesamiento aplicado.

## 8. Diagrama de Arquitectura de Componentes (Mermaid C4)

```mermaid
graph TD
    subgraph ecosistema [Ecosistema Marketplace Cloud]
        G5[Grupo 5: Pedidos] -->|Publica evento OrderCreated| Broker(Upstash Kafka / PubSub)
        G6[Grupo 6: Pagos] -->|Publica evento PaymentApproved| Broker
    end

    subgraph reporteria [Módulo de Reportería - Grupo 7]
        Broker -->|Consumo en tiempo real| StreamApp[Stream Processor: CloudRun]
        StreamApp -->|Guarda métricas inmediatas| DB[(Persistencia: Supabase Postgres)]
        
        Broker -->|Guarda logs crudos| ObjectStorage[Object Storage: Cloudflare R2]
        Cron[GitHub Actions: Tarea Batch] -->|Lee logs históricos| ObjectStorage
        Cron -->|Concilia e inyecta| DB
        
        BFF[Grupo 1: Frontend / BFF] -->|Consulta reportes GET| API[Nuestra API REST: Render]
        API -->|Lee datos consolidados| DB
    end
