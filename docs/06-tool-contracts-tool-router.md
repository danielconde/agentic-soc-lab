# 06 - Tool Contracts y Tool Router

## Objetivo

Esta fase introduce la primera capa de ejecución controlada de herramientas de investigación.

Hasta este punto, el sistema ya era capaz de:

1. normalizar alertas,
2. realizar un análisis inicial con Qwen,
3. generar un plan de investigación,
4. validar ese plan de forma determinista,
5. reparar automáticamente un plan inválido.

El objetivo de esta fase es convertir una solicitud de herramienta en una ejecución controlada, validada y normalizada antes de que sus resultados puedan reutilizarse como evidencia.

En esta iteración se implementó únicamente:

```text
get_signins
```

y se utilizó un connector en modo mock para validar el flujo completo antes de integrar una API real.

---

## Contratos implementados

Se creó la siguiente estructura:

```text
contracts/
└── tools/
    └── get-signins/
        ├── input.schema.json
        └── output.schema.json
```

### `input.schema.json`

Define el contrato de entrada de `get_signins`.

Campos soportados:

- `user`
- `time_range`
- `limit`

El parámetro obligatorio es:

```text
user
```

`time_range` permite definir una ventana temporal con:

```text
start
end
```

y `limit` está restringido a un máximo de:

```text
500
```

Esta limitación evita consultas sin límite o excesivamente costosas.

---

## Contrato de salida

El contrato:

```text
contracts/tools/get-signins/output.schema.json
```

normaliza el resultado de la herramienta con:

- `tool`
- `tool_version`
- `status`
- `query`
- `result_count`
- `events`
- `error`

Cada evento puede contener:

- `timestamp`
- `user`
- `source_ip`
- `status`
- `authentication_method`
- `application`
- `device`
- `location`

Esto permite desacoplar el agente de respuestas específicas de cada fabricante.

Arquitectura:

```text
Vendor API
    ↓
Connector
    ↓
Normalizer
    ↓
Canonical Tool Result
```

---

## Workflow implementado

Nombre en n8n:

```text
05 - Tool Router
```

Nombre recomendado para el fichero exportado:

```text
n8n/workflows/05-tool-router.json
```

Arquitectura validada:

```text
When clicking ‘Execute workflow’
        ↓
Manual Tool Request
        ↓
Tool Request Validator
        ↓
Tool Request Valid?
       / \
      /   \
 TRUE     FALSE
  ↓         ↓
Validated  Rejected Tool Request
Tool
Request
  ↓
Tool Router
  ↓
get_signins Mock
  ↓
get_signins Output Validator
  ↓
Canonical Tool Result
```

---

## Entrada simulada

El nodo:

```text
Manual Tool Request
```

simula una solicitud procedente del Investigation Planner.

Ejemplo:

```json
{
  "alert_id": "ALERT-0001",
  "step_id": "step-1",
  "tool": "get_signins",
  "parameters": {
    "user": "john.doe@example.com"
  }
}
```

---

## Tool Request Validator

El nodo:

```text
Tool Request Validator
```

trata cada solicitud como entrada no confiable.

La validación comprueba:

- `alert_id`
- `step_id`
- formato de `step_id`
- nombre de tool
- parámetros obligatorios
- parámetros no autorizados
- tipos de datos
- límites de valores
- formato de `time_range`

Actualmente solo se permite:

```text
get_signins
```

---

## Tool allowlist

La herramienta debe existir explícitamente en la política del router.

Ejemplo permitido:

```text
get_signins
```

Ejemplo rechazado:

```text
delete_user
```

La primera prueba negativa confirmó que:

```json
{
  "tool": "delete_user"
}
```

produce:

```text
tool 'delete_user' is not supported
```

y la ejecución termina en:

```text
Rejected Tool Request
```

sin alcanzar ninguna herramienta.

---

## Parameter allowlist

Incluso cuando la tool es válida, cada parámetro debe estar autorizado.

Para `get_signins` se permiten:

```text
user
time_range
limit
```

Durante la segunda prueba negativa se utilizó:

```json
{
  "tool": "get_signins",
  "parameters": {
    "user": "john.doe@example.com",
    "dangerous_parameter": "test"
  }
}
```

La solicitud fue rechazada con:

```text
parameters.dangerous_parameter is not allowed for get_signins
```

Esto evita que una tool reciba parámetros inesperados aunque el LLM o un componente anterior los genere.

---

## Range validation

También se validaron los límites de los parámetros permitidos.

En la tercera prueba negativa se utilizó:

```json
{
  "tool": "get_signins",
  "parameters": {
    "user": "john.doe@example.com",
    "limit": 5000
  }
}
```

La solicitud fue rechazada con:

```text
parameters.limit must be an integer between 1 and 500
```

Por tanto, la frontera de entrada valida:

```text
tool allowlist
+ parameter allowlist
+ type validation
+ range validation
+ controlled rejection
```

---

## Rejected Tool Request

Una solicitud inválida no lanza una tool ni continúa por el router.

Se normaliza como:

```json
{
  "status": "rejected",
  "reason": "tool_request_validation_failed",
  "errors": [],
  "request": {}
}
```

Esto permite que el rechazo sea un estado normal del workflow y pueda ser gestionado posteriormente por el orquestador.

---

## Tool Router

El nodo:

```text
Tool Router
```

utiliza un `Switch` para enrutar solicitudes válidas según:

```text
{{ $json.tool }}
```

En esta fase solo existe una ruta:

```text
get_signins
```

El router aplica defensa en profundidad: aunque exista una validación previa, solo las rutas explícitamente definidas pueden ejecutarse.

---

## `get_signins Mock`

La primera implementación de `get_signins` utiliza datos simulados.

Esto permite validar:

- routing,
- contrato de entrada,
- contrato de salida,
- normalización,
- manejo de evidencia,
- validación del connector,

sin depender todavía de credenciales ni APIs externas.

El mock genera dos eventos de autenticación para:

```text
john.doe@example.com
```

con diferentes IPs y ubicaciones.

---

## Evidencia estructurada

La salida validada fue:

```json
{
  "evidence_version": "1.0",
  "source": "tool",
  "tool": "get_signins",
  "tool_version": "1.0",
  "status": "success",
  "query": {
    "user": "john.doe@example.com",
    "time_range": null
  },
  "result_count": 2,
  "evidence": [
    {
      "timestamp": "2026-08-23T17:20:00Z",
      "user": "john.doe@example.com",
      "source_ip": "185.220.101.20",
      "status": "success",
      "authentication_method": "MFA",
      "application": "Microsoft 365",
      "device": "WIN11-LAPTOP-01",
      "location": {
        "country": "Spain",
        "city": "Madrid"
      }
    },
    {
      "timestamp": "2026-08-23T17:42:00Z",
      "user": "john.doe@example.com",
      "source_ip": "91.198.174.20",
      "status": "success",
      "authentication_method": "MFA",
      "application": "Microsoft 365",
      "device": null,
      "location": {
        "country": "Netherlands",
        "city": "Amsterdam"
      }
    }
  ],
  "error": null
}
```

A diferencia de las fases anteriores, aquí la relación entre:

```text
source_ip
location
timestamp
user
```

es explícita porque todos los campos pertenecen al mismo evento.

Esto permite utilizar posteriormente esa relación como evidencia real sin depender de inferencias del LLM.

---

## Output Validator

El nodo:

```text
get_signins Output Validator
```

valida de forma determinista la salida del connector.

Comprueba:

- tool y versión,
- estado,
- query,
- `result_count`,
- número real de eventos,
- timestamps,
- usuario,
- IP,
- estado del evento,
- authentication method,
- aplicación,
- dispositivo,
- ubicación,
- objeto de error.

Si el connector devuelve una estructura corrupta, la ejecución se detiene antes de que esa información llegue al agente.

---

## Canonical Tool Result

La salida vendor-neutral se normaliza finalmente mediante:

```text
Canonical Tool Result
```

con:

```text
evidence_version
source
tool
tool_version
status
query
result_count
evidence
error
```

Este será el formato que utilizará la capa de investigación en fases posteriores.

---

## Modelo de seguridad

La arquitectura validada es:

```text
Planner
  ↓
Tool Request
  ↓
Input Validator
  ↓
Tool Router
  ↓
Connector
  ↓
Output Validator
  ↓
Canonical Evidence
```

Principios aplicados:

### LLM sin acceso directo a APIs

El LLM no ejecuta requests arbitrarias.

### Allowlist de tools

Solo existen herramientas registradas.

### Allowlist de parámetros

Cada tool define exactamente qué parámetros acepta.

### Límites explícitos

Parámetros como `limit` tienen restricciones.

### Validación previa a la ejecución

Una petición inválida nunca alcanza el connector.

### Validación posterior a la ejecución

Una respuesta corrupta nunca alcanza al agente.

### Normalización vendor-neutral

El razonamiento posterior no depende del formato concreto del proveedor.

---

## Pruebas realizadas

### Prueba positiva

Tool:

```text
get_signins
```

Usuario:

```text
john.doe@example.com
```

Resultado:

```text
success
result_count = 2
```

### Prueba negativa 1

Tool no autorizada:

```text
delete_user
```

Resultado:

```text
rejected
```

### Prueba negativa 2

Parámetro no autorizado:

```text
dangerous_parameter
```

Resultado:

```text
rejected
```

### Prueba negativa 3

Valor fuera de límites:

```text
limit = 5000
```

Resultado:

```text
rejected
```

---

## Estado de la fase

La fase queda validada con:

- contratos de entrada y salida de `get_signins`,
- Tool Request Validator,
- Tool Request Valid?,
- rechazo controlado,
- Tool Router,
- connector mock,
- Output Validator,
- Canonical Tool Result,
- prueba positiva,
- tool allowlist,
- parameter allowlist,
- type validation,
- range validation,
- tres pruebas negativas.

La siguiente fase conectará el Investigation Planner con el Tool Router.

Para ello será necesario transformar:

```text
investigation_steps[]
```

en items individuales de n8n mediante una capa:

```text
Investigation Step Expander
```

Arquitectura objetivo:

```text
Validated Investigation Plan
        ↓
Investigation Step Expander
        ↓
one n8n item per step
        ↓
Tool Router
```

Después se ampliará progresivamente el catálogo de herramientas y se sustituirán los connectors mock por integraciones reales.
