# 07 - Orquestación de múltiples pasos de investigación

## Objetivo

Esta fase introduce la orquestación de múltiples pasos de investigación a partir de un plan previamente validado.

Hasta este punto, el sistema ya era capaz de normalizar una alerta, realizar un análisis inicial con Qwen, generar y validar un plan de investigación, reparar planes inválidos y ejecutar una herramienta individual mediante un Tool Router controlado.

El objetivo de esta fase es transformar un plan con varios `investigation_steps` en múltiples solicitudes independientes, ejecutarlas de forma controlada y reunir toda la evidencia obtenida en un único objeto canónico de investigación.

## Workflows implicados

### `05 - Tool Router`

Se conserva como workflow de pruebas manuales.

### `05b - Tool Router Subworkflow`

Versión invocable desde otros workflows mediante un trigger de sub-workflow.

Nombre recomendado del export:

```text
n8n/workflows/05b-tool-router-subworkflow.json
```

### `06 - Investigation Orchestrator`

Workflow principal de esta fase.

Nombre recomendado del export:

```text
n8n/workflows/06-investigation-orchestrator.json
```

## Arquitectura validada

```text
Validated Investigation Plan
        ↓
Investigation Step Expander
        ↓
one n8n item per investigation step
        ↓
Execute Tool Router
        ↓
05b - Tool Router Subworkflow
       / \
      /   \
get_signins
get_endpoint_activity
      ↓
Canonical Tool Results
        ↓
Investigation Evidence Aggregator
        ↓
Canonical Investigation Evidence Bundle
```

## Entrada del orquestador

Durante esta fase se utilizó un input manual que simula la salida validada de `04 - Investigation Planner`.

El plan contiene dos pasos:

```text
step-1 → get_signins
step-2 → get_endpoint_activity
```

## Investigation Step Expander

El nodo `Investigation Step Expander` convierte un único plan en múltiples items de n8n.

Entrada:

```text
1 plan
```

Salida:

```text
item 1 → get_signins
item 2 → get_endpoint_activity
```

Cada item conserva:

```text
request_version
alert_id
step_id
tool
parameters
context
```

Esto desacopla el plan de la ejecución y permite procesar cada acción de investigación de forma independiente.

## Ejecución como sub-workflow

`05b - Tool Router Subworkflow` recibe los requests desde `06 - Investigation Orchestrator`.

El nodo `Execute Tool Router` se configuró en:

```text
Run once for each item
```

Esto garantiza una llamada independiente al Tool Router por cada investigation step.

## Workflow input schema

El trigger del sub-workflow acepta:

```text
request_version
alert_id
step_id
tool
parameters
context
```

Los campos `parameters` y `context` se conservan como objetos JSON.

Durante la integración se detectó el error:

```text
'parameters' expects a string but we got object
```

La configuración del input schema se corrigió para mantener correctamente los objetos anidados.

## Primera ejecución multi-tool

La primera integración produjo:

```text
step-1 → get_signins            → success
step-2 → get_endpoint_activity  → rejected
```

El rechazo inicial de `get_endpoint_activity` era esperado porque todavía no estaba implementado.

Esto confirmó que el orquestador no improvisa implementaciones inexistentes y que una tool no soportada se rechaza de forma controlada.

## Contratos de `get_endpoint_activity`

Se añadió:

```text
contracts/tools/get-endpoint-activity/
├── input.schema.json
└── output.schema.json
```

El contrato de entrada soporta:

```text
host
time_range
limit
```

siendo `host` obligatorio y `limit` restringido a un máximo de 500.

La salida canónica contiene:

```text
tool
tool_version
status
query
result_count
events
error
```

Los eventos pueden incluir:

```text
timestamp
host
event_type
severity
user
process
network
```

## get_endpoint_activity Mock

Se implementó un connector mock antes de integrar un EDR/XDR real.

El mock genera:

```text
user_logon
process_execution
network_connection
```

La presencia de procesos como `powershell.exe` no se interpreta automáticamente como actividad maliciosa. La tool aporta telemetría estructurada; la interpretación queda para una fase posterior.

## Output Validator

`get_endpoint_activity Output Validator` valida de forma determinista:

- metadata de la tool,
- status,
- query,
- result_count,
- timestamps,
- host,
- event type,
- severity,
- user,
- process,
- network,
- error object.

Una respuesta corrupta no puede continuar hacia la capa de razonamiento.

## Ejecución multi-tool validada

Tras implementar `get_endpoint_activity`, el mismo plan produjo:

```text
get_signins            → success → 2 events
get_endpoint_activity  → success → 3 events
```

Total:

```text
5 evidence items
```

## Investigation Evidence Aggregator

El nodo `Investigation Evidence Aggregator` combina todos los Canonical Tool Results en un único objeto.

Entrada:

```text
N Canonical Tool Results
```

Salida:

```text
1 Canonical Investigation Evidence Bundle
```

## Canonical Investigation Evidence Bundle

La salida final validada contiene:

```text
evidence_bundle_version
alert_id
investigation_objective
status
execution_summary
tool_executions
```

Resumen validado:

```text
requested_steps      = 2
executed_tools       = 2
successful_tools     = 2
partial_tools        = 0
failed_tools         = 0
rejected_tools       = 0
total_evidence_items = 5
status               = success
```

## Evidencia obtenida

### get_signins

Se obtuvieron dos eventos de autenticación para `john.doe@example.com` con relaciones explícitas entre:

```text
timestamp
user
source_ip
location
```

Ejemplos:

```text
185.220.101.20 → Spain / Madrid
91.198.174.20  → Netherlands / Amsterdam
```

Estas relaciones proceden directamente de la tool.

### get_endpoint_activity

Se obtuvieron tres eventos:

```text
17:25 → user_logon
17:36 → process_execution
17:39 → network_connection
```

incluyendo telemetría asociada a `powershell.exe`, `msedge.exe` y `20.190.128.25:443`.

La evidencia se conserva como hechos observados, no como conclusiones.

## Principios de seguridad aplicados

- separación entre planificación y ejecución;
- ejecución independiente por step;
- sub-workflow como frontera de ejecución;
- allowlist de herramientas;
- contratos de entrada y salida;
- validación antes y después de la ejecución;
- normalización vendor-neutral;
- ausencia de inferencias implícitas;
- rechazo controlado de tools no implementadas.

## Estado de la Fase 7

La fase queda validada con:

- `06 - Investigation Orchestrator`;
- `05b - Tool Router Subworkflow`;
- workflow input schema;
- `Investigation Step Expander`;
- ejecución `Run once for each item`;
- múltiples requests independientes;
- `get_signins` operativo;
- contratos de `get_endpoint_activity`;
- mock de `get_endpoint_activity`;
- Output Validator de endpoint activity;
- ejecución multi-tool;
- `Investigation Evidence Aggregator`;
- Canonical Investigation Evidence Bundle;
- dos tools ejecutadas correctamente;
- cinco eventos de evidencia agregados;
- estado global `success`.

## Arquitectura al cierre de la fase

```text
Initial Alert
    ↓
Normalization
    ↓
Initial LLM Analysis
    ↓
Investigation Planner
    ↓
Plan Validator / Repair
    ↓
Investigation Orchestrator
    ↓
Step Expander
    ↓
Tool Router
   /       \
get_signins  get_endpoint_activity
   ↓             ↓
Validated Canonical Tool Evidence
        ↓
Investigation Evidence Aggregator
        ↓
Canonical Investigation Evidence Bundle
```

## Siguiente fase

La siguiente fase será:

```text
Fase 8 - Evidence-Based Investigation Analysis
```

La nueva capa de razonamiento recibirá:

```text
Canonical Initial Analysis
        +
Canonical Investigation Plan
        +
Canonical Investigation Evidence
```

y producirá una segunda evaluación del incidente basada en evidencia recopilada por las tools.

El objetivo será evolucionar desde un veredicto preliminar como `inconclusive` hacia un veredicto sustentado por evidencia, manteniendo separadas observaciones, hipótesis y conclusiones.
