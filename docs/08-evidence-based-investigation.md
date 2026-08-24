# 08 - Evidence-Based Investigation y Orquestación Iterativa

## Objetivo

Esta fase implementa una investigación basada en evidencias con control determinista sobre las decisiones operativas del LLM.

El sistema permite:

- analizar un `Canonical Investigation Evidence Bundle`,
- generar una evaluación estructurada,
- validar de forma determinista la salida del LLM,
- reparar una salida inválida mediante un único intento controlado,
- generar planes iterativos de investigación,
- ejecutar herramientas reales a través del `Investigation Orchestrator`,
- combinar evidencias de distintas rondas,
- evaluar si existen gaps ejecutables,
- finalizar de forma controlada cuando no existan capacidades suficientes.

Principio de diseño:

> El LLM interpreta la evidencia. El código determinista decide qué puede ejecutarse.

## Workflows implicados

### `05b - Tool Router Subworkflow`

Sub-workflow invocable encargado de validar requests de tools, aplicar allowlists de tools y parámetros, ejecutar connectors, normalizar outputs y producir un resultado canónico.

Durante esta fase se amplió el contrato para preservar `plan_context` como metadata de orquestación:

```json
{
  "plan_context": {
    "plan_version": "1.0",
    "iteration_number": 1,
    "objective": "Collect IP reputation evidence for the observed source IPs.",
    "requested_steps": 2
  }
}
```

`plan_context` no se confía desde la respuesta del connector. Se recupera desde la request previamente validada.

### `06 - Investigation Orchestrator`

Ruta manual:

```text
When clicking ‘Execute workflow’
        ↓
Manual Investigation Plan Test Input
        ↓
Investigation Step Expander
        ↓
Execute Tool Router
        ↓
Investigation Evidence Aggregator
```

Ruta productiva:

```text
When Called by Investigation Workflow
        ↓
Investigation Step Expander
        ↓
Execute Tool Router
        ↓
Investigation Evidence Aggregator
```

`Investigation Step Expander` transforma el plan en un item por step y propaga `plan_context`.

`Investigation Evidence Aggregator` ya no depende del fixture manual.

### `07 - Evidence-Based Investigation Analysis`

Flujo principal:

```text
Canonical Investigation Inputs
        ↓
Evidence Analysis Context Builder
        ↓
Evidence Analysis Request Builder
        ↓
Ollama Evidence Analysis
        ↓
Evidence Analysis Validator
        ↓
Investigation Continuation Decision
        ↓
Needs More Investigation?
```

Si existen gaps:

```text
Investigation Iteration Controller
        ↓
Second Investigation Planning Input
        ↓
Iterative Plan Request Builder
        ↓
Ollama Iterative Planner
        ↓
Iterative Plan Validator
```

## Validación del assessment

La salida del LLM se considera no confiable.

Se validan:

- `alert_id`,
- versión de contrato,
- provenance,
- coincidencia exacta de field/value/type,
- sources válidas,
- risk/verdict/confidence,
- gaps.

Principio aplicado:

```text
evidence exists
≠
relationship established
≠
authorization for further investigation
```

## Iterative Plan Validator

Controla:

- `alert_id`,
- `iteration_number`,
- tool allowlist,
- parameter allowlist,
- entity provenance,
- queries ya ejecutadas,
- `addresses_gap`,
- gaps inventados,
- cobertura de gaps autoritativos,
- step IDs secuenciales.

Ejemplo validado:

```text
get_ip_reputation
→ destination IP observada
→ sin gap autoritativo
→ REJECT
```

## Repair controlado del plan iterativo

Se permite un único repair:

```text
Ollama Iterative Planner
        ↓
Iterative Plan Validator
        ↓
Iterative Plan Valid?
├─ true  → Valid Iterative Plan
└─ false → Iterative Plan Repair Request Builder
             ↓
           Ollama Iterative Plan Repair
             ↓
           Repaired Iterative Plan Validator
             ↓
           Repaired Iterative Plan Valid?
```

El validator del plan reparado aplica exactamente la misma política.

## Plan válido sin steps ejecutables

Un plan puede ser válido y contener:

```json
{
  "investigation_steps": [],
  "unresolved_gaps": [...]
}
```

Se trata como estado terminal controlado:

```text
Valid Iterative Plan
        ↓
Has Executable Investigation Steps?
├─ true  → Execute Investigation Orchestrator
└─ false → No Executable Iterative Steps
              ↓
           Final Investigation Result - No Executable Steps
```

## Ejecución real del orchestrator

La rama iterativa ya no utiliza `Iterative Evidence Input`.

Ruta final:

```text
Iterative Plan Validator / Repair
        ↓
Valid Iterative Plan
        ↓
Has Executable Investigation Steps?
        ↓ true
Execute Investigation Orchestrator
        ↓
06 - Investigation Orchestrator
        ↓
05b - Tool Router Subworkflow
        ↓
Combined Investigation Evidence Builder
```

Se validó una ejecución real:

```text
185.220.101.20 → suspicious
91.198.174.20  → benign

requested_steps      = 2
executed_tools       = 2
successful_tools     = 2
partial_tools        = 0
failed_tools         = 0
rejected_tools       = 0
total_evidence_items = 2
```

## Tool `get_ip_reputation`

Se añadió soporte para `get_ip_reputation`.

Ejemplo de resultado normalizado:

```json
{
  "ip": "185.220.101.20",
  "classification": "suspicious",
  "reputation_score": 85,
  "confidence": 0.9,
  "provider": "agentic-soc-mock-ti",
  "categories": [
    "anonymization",
    "known-abuse"
  ]
}
```

La reputación es evidencia de apoyo y no prueba de compromiso o legitimidad.

## Segundo assessment y repair

El segundo assessment también se valida de forma determinista y dispone de un único repair controlado.

Se evita:

- pérdida silenciosa de gaps,
- evidence incorrecta,
- creación de gaps nuevos,
- acciones no autorizadas por gaps,
- repetición genérica de investigaciones ya realizadas.

## Capability evaluation

`Remaining Gap Capability Evaluator` decide de forma determinista si los gaps pueden ejecutarse con las tools actuales.

Ejemplo:

```text
Additional sign-in history
→ get_signins existe
→ falta nuevo time_range explícito
→ non-executable

Device ownership verification
→ no existe capability implementada
→ non-executable
```

Resultado:

```json
{
  "decision": "finalize",
  "termination_reason": "no_executable_remaining_gaps"
}
```

## Resultado terminal

Ejemplo:

```json
{
  "investigation_result_version": "1.0",
  "status": "completed_with_unresolved_gaps",
  "termination_reason": "no_executable_remaining_gaps",
  "verdict": "inconclusive",
  "confidence": 0,
  "execution_summary": {
    "automated_investigation_complete": true
  }
}
```

Esto diferencia:

```text
automated investigation complete
≠
all investigation questions resolved
```

## Limpieza realizada

Se eliminaron:

- `Iteration Branch Test`
- `Iterative Evidence Input`
- `Orchestrator Integration Test Input`

El fixture manual de workflow 06 queda identificado como:

```text
Manual Investigation Plan Test Input
```

y no pertenece a la ruta productiva.

## Controles de seguridad

- salidas LLM tratadas como untrusted input,
- allowlists de tools y parámetros,
- Tool Router como frontera de ejecución,
- provenance de evidence,
- no inventar scopes o time ranges,
- no ejecutar queries idénticas ya completadas,
- repairs con la misma política de validación,
- ninguna acción destructiva de contención.

## Estado

La Fase 8 queda funcionalmente validada con:

- evidence-based assessment,
- validación determinista,
- repair de assessments,
- planning iterativo,
- repair de planes,
- ejecución real de sub-workflows,
- preservación de contexto entre workflows,
- agregación multi-step,
- capability evaluation,
- finalización controlada.

Los connectors continúan en modo mock para mantener las pruebas reproducibles antes de integrar APIs reales.
