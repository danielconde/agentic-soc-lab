# 09 - Response Planning, Policy y Human Approval

## Objetivo

Esta fase introduce una capa de respuesta separada de la investigación. El LLM puede proponer acciones, pero no autorizarlas ni ejecutarlas.

Principio de diseño:

> El LLM propone. El código valida. La política decide. La persona aprueba. La ejecución queda separada.

En esta fase no se ejecutan acciones reales contra Entra ID, Defender, firewalls, correo u otros sistemas.

## Workflow

### `08 - Response Planning`

Export:

```text
n8n/workflows/08-response-planning.json
```

Ruta validada:

```text
Canonical Investigation Result Input
        ↓
Response Context Builder
        ↓
Response Planner Request Builder
        ↓
Ollama Response Planner
        ↓
Response Plan Validator
        ↓
Response Policy Evaluator
        ↓
Requires Human Approval?
        ↓ true
Approval Request Builder
        ↓
Human Approval Test Input
        ↓
Approval Decision Validator
        ↓
Execution Authorization Gate
        ↓ true
Authorized Response Actions
```

## Response Context Builder

Construye el contexto canónico de respuesta a partir de un resultado terminal de investigación y extrae únicamente targets observados en evidencia validada:

```text
users
hosts
ips
domains
urls
emails
```

La existencia de un target no autoriza una acción contra él.

Acciones soportadas en esta fase:

```text
no_action
revoke_sessions
disable_user
isolate_endpoint
block_ip
block_domain
delete_email
```

## Response Planner

`Response Planner Request Builder` prepara la petición a Qwen.

El LLM puede proponer acciones, pero no puede:

- ejecutar acciones,
- autorizar acciones,
- modificar el verdict autoritativo,
- inventar targets,
- usar acciones fuera de allowlist,
- afirmar que existe aprobación.

## Response Plan Validator

La salida del LLM se trata como `untrusted input`.

Se validan de forma determinista:

- `response_plan_version`,
- `alert_id`,
- verdict y confidence,
- recommendation,
- estructura de acciones,
- IDs secuenciales,
- action allowlist,
- target type y target value,
- evidence sources,
- prioridad,
- coherencia entre recommendation y proposed actions.

Política mínima validada:

```text
inconclusive → no destructive response
benign       → no destructive response
```

### Negative security test

Se forzó:

```text
verdict = inconclusive
action  = disable_user
target  = john.doe@example.com
```

Resultado:

```json
{
  "valid": false,
  "errors": [
    "investigation verdict 'inconclusive' cannot propose destructive response actions",
    "investigation verdict 'inconclusive' requires recommendation 'no_action'"
  ]
}
```

## Response Policy Evaluator

Estados posibles:

```text
reject
no_action
approval_required
automatic_allowed
```

La política inicial es conservadora:

```text
automatic_execution_enabled = false
```

Comportamiento:

```text
benign       → destructive response forbidden
inconclusive → destructive response forbidden
suspicious   → approval_required
malicious    → approval_required
```

### Caso inconcluso

Resultado validado:

```json
{
  "decision": "no_action",
  "execution_allowed": false,
  "human_approval_required": false
}
```

### Caso malicioso

Se validó un plan con:

```text
revoke_sessions
→ john.doe@example.com
```

Resultado:

```json
{
  "decision": "approval_required",
  "execution_allowed": false,
  "human_approval_required": true
}
```

Por tanto:

```text
malicious ≠ automatic containment
```

## Approval Request Builder

Cuando la política devuelve `approval_required`, se crea un request de aprobación inmutable:

```json
{
  "approval_request_version": "1.0",
  "approval_request_id": "approval-ALERT-0001-...",
  "alert_id": "ALERT-0001",
  "status": "pending",
  "approval_scope": {
    "action_count": 1,
    "action_ids": ["action-1"],
    "modification_allowed": false
  },
  "allowed_decisions": ["approve", "reject"]
}
```

La aprobación queda vinculada a un `approval_request_id`, un `alert_id` y una lista exacta de `action_ids`.

## Human Approval Test Input

Se utiliza como fixture manual para simular:

- aprobación,
- rechazo,
- manipulación del scope.

Será sustituido posteriormente por un mecanismo real de aprobación humana.

## Approval Decision Validator

Valida:

- `approval_request_id`,
- `alert_id`,
- decisión `approve` o `reject`,
- approver,
- timestamp,
- scope exacto de acciones,
- duplicados,
- ampliación o sustitución de action IDs.

### Aprobación válida

```json
{
  "valid": true,
  "approved": true,
  "rejected": false,
  "execution_allowed": true
}
```

### Rechazo humano

```json
{
  "valid": true,
  "approved": false,
  "rejected": true,
  "authorized_actions": [],
  "execution_allowed": false
}
```

### Scope tampering

Se intentó aprobar `action-999` cuando el scope autorizado contenía `action-1`.

Resultado:

```json
{
  "valid": false,
  "errors": [
    "approved_action_ids must exactly match the requested approval scope"
  ],
  "authorized_actions": [],
  "execution_allowed": false
}
```

## Execution Authorization Gate

La rama de autorización requiere simultáneamente:

```text
valid = true
approved = true
execution_allowed = true
authorized_actions.length > 0
```

## Authorized Response Actions

Genera el envelope canónico previo a ejecución:

```json
{
  "response_execution_authorization_version": "1.0",
  "alert_id": "ALERT-0001",
  "status": "authorized_for_execution",
  "execution_authorized": true,
  "authorized_action_count": 1,
  "authorized_actions": [
    {
      "execution_id": "ALERT-0001-action-1",
      "action_id": "action-1",
      "action": "revoke_sessions",
      "target": {
        "type": "user",
        "value": "john.doe@example.com"
      },
      "authorization": {
        "approval_request_id": "approval-ALERT-0001-...",
        "approved_by": "manual-test-approver",
        "approved_at": "2026-08-24T23:27:08.323Z"
      }
    }
  ]
}
```

`authorized_for_execution` no significa `executed`.

## Controles de seguridad validados

- El LLM no ejecuta ni autoriza acciones.
- El verdict autoritativo no puede modificarse.
- Targets inventados no son válidos.
- Acciones fuera de allowlist no son válidas.
- `inconclusive` no puede generar contención destructiva.
- `malicious` no implica ejecución automática.
- Las acciones destructivas requieren aprobación humana.
- El approval request tiene scope inmutable.
- Una aprobación no puede añadir nuevas acciones.
- Una decisión `reject` no produce acciones autorizadas.
- Una aprobación manipulada falla cerrada.
- No existe todavía ningún connector de respuesta real.

## Estado

La Fase 9 queda validada con:

- response planning,
- validación determinista,
- response policy,
- ruta `no_action`,
- ruta `approval_required`,
- approval request canónico,
- aprobación positiva,
- rechazo humano,
- protección frente a scope tampering,
- execution authorization envelope.

La siguiente fase será:

```text
Phase 10 - Response Execution
```

Inicialmente con ejecución mock, idempotencia, retries controlados, resultados parciales y auditoría.
