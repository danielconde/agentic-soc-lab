# 05 - Investigation Planner

## Objetivo

Esta fase introduce la primera capa de planificación agéntica del laboratorio. El sistema transforma el análisis inicial validado en un plan de investigación estructurado, validable y preparado para ser consumido posteriormente por un Tool Router.

El LLM no ejecuta herramientas directamente. Solo propone qué evidencia debe recopilarse y mediante qué herramienta autorizada.

## Workflow implementado

Nombre en n8n:

```text
04 - Investigation Planner
```

Nombre recomendado del export:

```text
n8n/workflows/04-investigation-planner.json
```

Arquitectura validada:

```text
When clicking ‘Execute workflow’
        ↓
Canonical Analysis Input
        ↓
Investigation Planner Request Builder
        ↓
Ollama Investigation Planner
        ↓
Investigation Plan Validator
        ↓
Investigation Plan Valid?
       / \
      /   \
 TRUE     FALSE
  ↓         ↓
Valid     Plan Repair Request Builder
Invest.          ↓
Plan        Ollama Plan Repair
                   ↓
              Repaired Plan Validator
```

## Contrato del plan

Se creó:

```text
contracts/investigation-plan.schema.json
```

El contrato define:

- `plan_version`
- `alert_id`
- `objective`
- `investigation_steps`

Cada paso incluye:

- `step_id`
- `priority`
- `purpose`
- `tool`
- `parameters`
- `evidence_target`

## Herramientas permitidas

El planner solo puede seleccionar herramientas incluidas en un allowlist:

```text
get_signins
get_ip_reputation
get_endpoint_activity
get_user_activity
get_mail_activity
```

Nombres no autorizados como `delete_user`, `run_powershell` o `disable_account` deben ser rechazados.

## Separación entre razonamiento y ejecución

```text
LLM
 ↓
propone plan
 ↓
validator determinista
 ↓
plan canónico
 ↓
Tool Router
```

La ejecución real de herramientas no forma parte todavía de esta fase.

## Capacidades y limitaciones de herramientas

Cada herramienta tiene capacidades explícitas. Por ejemplo, `get_ip_reputation` puede utilizarse para consultar reputación o threat intelligence de una IP, pero no para establecer de forma autoritativa un mapping IP→país ni para afirmar desde qué país autenticó un usuario.

Esto evita que una herramienta válida se utilice para una finalidad no autorizada.

## Validación determinista

El nodo:

```text
Investigation Plan Validator
```

trata la salida del LLM como entrada no confiable.

Valida:

- versión del plan,
- correspondencia de `alert_id`,
- estructura,
- secuencia de `step_id`,
- prioridades,
- allowlist de tools,
- parámetros obligatorios,
- parámetros no autorizados,
- entidades observadas,
- restricciones semánticas de cada tool.

Si el LLM introduce un usuario, host o IP que no existe en `observed_evidence`, el plan se rechaza.

## Caso semántico validado

Durante las pruebas, Qwen intentó utilizar `get_ip_reputation` para comprobar si una IP estaba asociada con observaciones geográficas.

El validator rechazó correctamente el plan:

```text
tool 'get_ip_reputation' is being used for a purpose outside its allowed capabilities
```

Esto valida un principio esencial:

```text
prompt ≠ control de seguridad
```

Las instrucciones del prompt reducen errores, pero la autorización final debe ser determinista.

## Rama de error controlada

El validator fue modificado para no detener el workflow con una excepción. Devuelve:

```json
{
  "valid": false,
  "errors": ["..."],
  "plan": {},
  "validation_metadata": {
    "validator": "investigation_plan_validator",
    "validator_version": "1.0",
    "error_count": 1
  }
}
```

o, si el plan es válido:

```json
{
  "valid": true,
  "errors": [],
  "plan": {}
}
```

## Nodo `Investigation Plan Valid?`

El nodo IF evalúa:

```text
{{ $json.valid }}
```

y divide el flujo:

```text
true  → Valid Investigation Plan
false → Plan Repair Request Builder
```

## Reparación controlada

Cuando el plan es inválido, se envían a Qwen:

- el análisis validado,
- el plan rechazado,
- los errores exactos del validator,
- las capacidades autorizadas.

Qwen genera una propuesta corregida mediante `Ollama Plan Repair`, que vuelve a pasar por `Repaired Plan Validator`.

```text
LLM proposes
     ↓
deterministic validator
     ↓
invalid
     ↓
LLM receives validation errors
     ↓
LLM repairs
     ↓
deterministic validator
```

El LLM nunca decide si su propio plan es válido.

## Protección contra bucles

En esta fase se permite un único intento de reparación:

```text
Initial Plan
    ↓
Validator
    ↓
Repair
    ↓
Validator
```

Si el plan reparado vuelve a ser inválido, no se lanza automáticamente otro ciclo.

## Resultado validado

El plan inicial fue rechazado por uso incorrecto de `get_ip_reputation`.

Tras el repair loop, el plan válido conservó:

```text
get_signins
```

para:

```text
john.doe@example.com
```

y:

```text
get_endpoint_activity
```

para:

```text
WIN11-LAPTOP-01
```

La revalidación terminó con:

```json
{
  "valid": true,
  "errors": [],
  "validation_metadata": {
    "validator": "investigation_plan_validator",
    "validator_version": "1.0",
    "error_count": 0
  }
}
```

## Principios de seguridad aplicados

- El LLM se trata como componente no confiable.
- Las herramientas están limitadas mediante allowlist.
- Los parámetros se validan por herramienta.
- Las entidades deben existir en la evidencia observada.
- Una tool solo puede utilizarse para capacidades explícitamente autorizadas.
- No se permiten acciones destructivas ni de contención en esta fase.
- La reparación está limitada para evitar loops.

## Estado de la fase

La fase queda validada con:

- contrato de Investigation Plan,
- planner con Qwen,
- allowlist de herramientas,
- definición de capabilities,
- validator determinista,
- validación de entidades,
- validación semántica,
- rama controlada para planes inválidos,
- repair loop,
- revalidación del plan reparado,
- protección básica contra bucles.

La siguiente fase será:

```text
Tool Contracts
      ↓
Tool Router
      ↓
Mock / Real Tool Execution
```

Antes de conectar herramientas reales, cada tool deberá disponer de contrato de entrada y salida, validación de parámetros, control de errores, límites de ejecución y logging seguro.
