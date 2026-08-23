# Normalización y contrato de alertas

Esta guía documenta la tercera fase funcional de **Agentic SOC Lab**: la creación de un formato canónico de alerta y su normalización dentro de n8n.

El objetivo es desacoplar el resto del SOC de los formatos específicos de cada proveedor.

```text
Fuente de alerta
      |
      v
Manual Alert Input
      |
      v
Alert Normalizer
      |
      v
Canonical Alert v1.0
```

Durante esta fase todavía no se utiliza el LLM. Primero se valida que la entrada puede normalizarse, rechazarse si es inválida, conservar la alerta original y producir una estructura predecible.

## 1. Workflow utilizado

Nombre exacto:

```text
02 - Alert Normalization
```

Nodos:

```text
When clicking ‘Execute workflow’
        |
        v
Manual Alert Input
        |
        v
Alert Normalizer
```

### `When clicking ‘Execute workflow’`

Trigger manual utilizado durante desarrollo y pruebas.

### `Manual Alert Input`

Genera una alerta simulada. Más adelante podrá sustituirse por Webhook, Microsoft Defender, CrowdStrike, SentinelOne, Elastic, Fortinet, Floci o el Scenario Generator.

### `Alert Normalizer`

Transforma la alerta recibida al contrato canónico y realiza validaciones mínimas.

## 2. Contrato canónico

El contrato se almacena en:

```text
contracts/alert.schema.json
```

Versión actual:

```text
1.0
```

Estructura principal:

```text
Canonical Alert
|
+-- schema_version
+-- alert_id
+-- title
+-- description
+-- severity
+-- category
+-- source
+-- event_time
+-- entities
+-- iocs
+-- metadata
+-- raw_alert
```

### Severidad

Valores normalizados:

```text
informational
low
medium
high
critical
unknown
```

Ejemplos:

```text
High -> high
Severe -> critical
Moderate -> medium
```

### Categoría

Valores normalizados:

```text
identity
endpoint
network
email
cloud
data
application
other
unknown
```

Ejemplos:

```text
Identity -> identity
IAM -> identity
EDR -> endpoint
Phishing -> email
DLP -> data
```

## 3. Entidades

Las entidades se almacenan en:

```json
{
  "users": [],
  "hosts": [],
  "ips": []
}
```

Las IP conservan su función:

```json
"ips": [
  {
    "value": "185.220.101.20",
    "role": "source"
  },
  {
    "value": "20.190.128.25",
    "role": "destination"
  }
]
```

Roles soportados:

```text
source
destination
related
unknown
```

Esto será útil para futuras herramientas como:

```text
get_ip_reputation(source_ip)
get_network_activity(destination_ip)
```

## 4. `metadata` y `raw_alert`

`metadata` conserva información específica de la fuente que todavía no tiene campo canónico.

```json
"metadata": {
  "additional_data": {
    "country_a": "Spain",
    "country_b": "Netherlands",
    "authentication_method": "MFA",
    "device_managed": false
  }
}
```

`raw_alert` conserva la alerta original completa. Esto permite no perder evidencia, depurar errores de normalización y consultar campos no modelados.

## 5. Alerta de prueba

El nodo exacto `Manual Alert Input` utiliza:

```javascript
return [
  {
    json: {
      id: "ALERT-0001",
      name: "Impossible travel detected",
      description: "User authenticated from geographically distant locations within an unusual time window.",
      severity: "High",
      vendor: "Microsoft",
      product: "Microsoft Defender XDR",
      category: "Identity",
      timestamp: "2026-08-23T17:30:00Z",
      user: "john.doe@example.com",
      host: "WIN11-LAPTOP-01",
      source_ip: "185.220.101.20",
      destination_ip: "20.190.128.25",
      additional_data: {
        country_a: "Spain",
        country_b: "Netherlands",
        authentication_method: "MFA",
        device_managed: false
      }
    }
  }
];
```

Configuración:

```text
Mode:
Run Once for All Items
```

## 6. Resultado esperado

`Alert Normalizer` produce una salida equivalente a:

```json
{
  "schema_version": "1.0",
  "alert_id": "ALERT-0001",
  "title": "Impossible travel detected",
  "description": "User authenticated from geographically distant locations within an unusual time window.",
  "severity": "high",
  "category": "identity",
  "source": {
    "vendor": "Microsoft",
    "product": "Microsoft Defender XDR",
    "integration": "manual"
  },
  "event_time": "2026-08-23T17:30:00.000Z",
  "entities": {
    "users": ["john.doe@example.com"],
    "hosts": ["WIN11-LAPTOP-01"],
    "ips": [
      {
        "value": "185.220.101.20",
        "role": "source"
      },
      {
        "value": "20.190.128.25",
        "role": "destination"
      }
    ]
  },
  "iocs": [],
  "metadata": {
    "additional_data": {
      "country_a": "Spain",
      "country_b": "Netherlands",
      "authentication_method": "MFA",
      "device_managed": false
    }
  },
  "raw_alert": {}
}
```

`raw_alert` contiene en ejecución la alerta original completa.

## 7. Validación

El normalizador valida actualmente:

```text
alert_id
title
source.vendor
source.product
event_time
```

Las entradas inválidas se rechazan antes de continuar el workflow.

## 8. Prueba negativa: `alert_id`

En `Manual Alert Input`:

```javascript
id: ""
```

Resultado esperado en `Alert Normalizer`:

```text
Alert validation failed: alert_id is required
```

Prueba validada correctamente.

## 9. Prueba negativa: `timestamp`

En `Manual Alert Input`:

```javascript
timestamp: "not-a-date"
```

Resultado esperado:

```text
Alert validation failed: event_time must be a valid ISO 8601 timestamp
```

Prueba validada correctamente.

El timestamp se valida antes de ejecutar `toISOString()`, evitando excepciones JavaScript no controladas.

## 10. Criterio de éxito

- [ ] `02 - Alert Normalization` procesa una alerta válida.
- [ ] `Manual Alert Input` entrega la alerta simulada esperada.
- [ ] `Alert Normalizer` genera el contrato `1.0`.
- [ ] Severidad y categoría se normalizan.
- [ ] Las IP conservan sus roles.
- [ ] `raw_alert` conserva la evidencia original.
- [ ] `metadata` conserva información específica.
- [ ] Un `alert_id` vacío es rechazado.
- [ ] Un timestamp inválido es rechazado de forma controlada.

Todas estas comprobaciones han sido validadas en el entorno actual.

## 11. Justificación arquitectónica

Sin normalización, cada componente tendría que conocer formatos distintos:

```text
Defender ---------+
CrowdStrike ------+
SentinelOne ------+
Elastic ----------+--> Alert Normalizer --> Canonical Alert
Fortinet ---------+
Scenario Generator+
```

Los componentes posteriores podrán trabajar siempre sobre:

```text
Canonical Alert
      |
      +--> Investigation Planner
      +--> Tool Selection
      +--> Evidence Collection
      +--> LLM Analysis
      +--> Verdict Engine
```

Esto reduce acoplamiento, facilita pruebas y permite sustituir proveedores sin rediseñar el motor agéntico.

## 12. Consideraciones de seguridad

La normalización forma parte de la frontera de confianza de la aplicación.

- Las entradas externas se consideran no confiables.
- Los campos obligatorios se validan.
- Los errores se producen de forma controlada.
- El LLM no debe recibir directamente datos arbitrarios sin validación previa.
- `raw_alert` se conserva como evidencia, pero no implica que todos sus campos deban enviarse posteriormente al LLM.

En futuras iteraciones esta capa podrá incorporar validación JSON Schema completa, límites de tamaño y controles adicionales de tipo.

## 13. Resultado de la fase

El proyecto dispone ya de una interfaz estable entre las fuentes de alerta y el futuro motor de investigación:

```text
Raw Alert
    |
    v
Alert Normalizer
    |
    v
Canonical Alert v1.0
```

La siguiente fase utilizará esta salida para construir el contexto de investigación y realizar la primera evaluación SOC mediante Qwen.
