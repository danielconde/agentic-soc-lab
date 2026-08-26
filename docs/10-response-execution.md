# 10 - Response Execution

## Objetivo

Esta fase implementa la primera capa de ejecución de respuesta del Agentic SOC Lab.

La ejecución es deliberadamente **mock**: no se realizan acciones reales contra Azure, Entra ID, Defender, firewalls, correo, endpoints ni Floci.

El objetivo es validar de forma segura:

- expansión de acciones autorizadas,
- routing por capability,
- conectores de respuesta mock,
- normalización canónica de resultados,
- idempotencia intra-run,
- estados `success`, `partial`, `failed` y `duplicate`,
- retries controlados,
- límites de intentos,
- agregación terminal,
- auditoría de ejecución.

Principio de diseño:

> Una acción autorizada no se ejecuta directamente. Primero pasa por una capa de ejecución controlada, con idempotencia, retries, resultado canónico y auditoría.

## Workflow

### `09 - Response Execution`

Export:

```text
n8n/workflows/09-response-execution.json
```

Arquitectura validada:

```text
Authorized Response Test Input
        ↓
Response Action Expander
        ↓
Response Execution Router
        ↓
Response Idempotency Guard
        ↓
Execute Response Action?
       / \
    true  false
     │      │
     │      ↓
     │   Duplicate Execution Result
     │      ↓
     │   Canonical Duplicate Execution Result
     │
     ↓
Mock Identity Response Connector
        ↓
Canonical Successful Execution Result
        ↓
Retry Required?
       / \
    true  false
     │      │
     ↓      │
Retry Execution Request Builder
     ↓      │
Mock Identity Response Connector - Retry
     ↓      │
Canonical Retry Execution Result
      \     /
       \   /
        ↓ ↓
Merge Terminal Results
        ↓
Merge Response Execution Results
        ↓
Response Execution Aggregator
        ↓
Response Execution Audit
```

## Response Action Expander

Transforma el envelope autorizado en un item por acción. Preserva `execution_id`, `action_id`, target, authorization metadata e `idempotency_key`.

```text
idempotency_key = execution_id
```

## Response Execution Router

Mapea las acciones a capabilities de respuesta mock:

```text
revoke_sessions  → mock_identity_response
disable_user     → mock_identity_response
isolate_endpoint → mock_endpoint_response
block_ip         → mock_network_response
block_domain     → mock_network_response
delete_email     → mock_mail_response
```

En esta fase se implementa y valida `mock_identity_response`.

El router valida allowlists, target type, authorization metadata y `execution_context.mode = mock`.

## Mock Identity Response Connector

Connector mock para:

```text
revoke_sessions
disable_user
```

Comportamientos de prueba:

```text
usuario normal
→ success

fail.user@example.com
→ attempt 1: failed / retryable
→ attempt 2: success

hardfail.user@example.com
→ failed / non-retryable

partial.user@example.com
→ partial / non-retryable
```

No realiza llamadas reales.

## Canonical Response Execution Result

Normaliza los resultados a un contrato común.

Estados soportados:

```text
success
partial
failed
duplicate
```

La metadata conserva connector, target, idempotency key, attempt, max attempts, timestamps y authorization provenance.

## Idempotencia

### Intra-run

`Response Idempotency Guard` detecta duplicados durante una misma ejecución del workflow.

Se validó:

```text
primera aparición
→ new
→ should_execute = true

segunda aparición con mismo execution_id
→ duplicate
→ should_execute = false
```

El resultado agregado validado fue:

```json
{
  "execution_summary": {
    "requested": 2,
    "success": 1,
    "duplicate": 1,
    "changed": 1,
    "unchanged": 1
  }
}
```

### Limitación

`$getWorkflowStaticData()` no se considera almacenamiento durable ni control de seguridad suficiente.

La auditoría marca:

```text
durable_idempotency = false
```

Antes de ejecutar contra Floci/Azure deberá sustituirse por almacenamiento persistente.

## Estados globales

Se validaron:

```text
success
partial
failed
duplicate
```

El Aggregator utiliza:

```text
completed
completed_partial
completed_with_errors
completed_no_changes
failed
```

## Retry controlado

Política:

```text
failed + retryable + attempt < max_attempts
→ retry

failed + non-retryable
→ no retry

partial
→ no retry automático

success
→ no retry

duplicate
→ no retry
```

Configuración:

```text
max_attempts = 2
```

### Retry transitorio validado

```text
attempt 1
→ failed
→ retryable = true
→ retry_eligible = true

attempt 2
→ success
→ retry_eligible = false
```

El Aggregator contabiliza una sola ejecución lógica.

### Permanent failure validado

```text
hardfail.user@example.com
→ failed
→ retryable = false
→ retry_eligible = false
→ attempt = 1
→ no retry
```

## Merge de resultados terminales

Un fallo retryable del primer intento no se agrega directamente.

```text
attempt 1 failed retryable
→ retry
→ attempt 2 terminal
→ aggregator
```

Los resultados terminales son:

```text
success
partial
non-retryable failed
duplicate
success after retry
```

## Response Execution Audit

Nodo terminal de la fase.

Registra:

- alert ID,
- status global,
- execution summary,
- retry summary,
- authorization provenance,
- approvers,
- action/target,
- idempotency,
- attempts,
- timings,
- connector.

Ejemplo validado:

```json
{
  "response_execution_audit_version": "1.0",
  "execution_status": "failed",
  "retry_summary": {
    "actions_reaching_retry_attempt": 0,
    "retryable_terminal_failures": 0,
    "non_retryable_failures": 1
  },
  "authorization_summary": {
    "authorization_present": true
  },
  "security_properties": {
    "execution_mode": "mock",
    "real_external_actions": false,
    "durable_idempotency": false,
    "human_approval_preserved": true,
    "max_attempts_enforced": true
  }
}
```

## Controles de seguridad validados

- Solo se procesan acciones previamente autorizadas.
- El router aplica allowlists.
- Solo se permite ejecución mock.
- La authorization metadata se preserva.
- La idempotencia intra-run evita duplicados.
- Los duplicados no ejecutan connector.
- Los retries están limitados.
- Solo errores retryable se reintentan.
- `partial` no se reintenta automáticamente.
- Permanent failure no se reintenta.
- El retry conserva el mismo `execution_id`.
- Los attempts no se cuentan como acciones distintas.
- Se genera auditoría terminal.
- No existen acciones reales contra sistemas externos.
- La idempotencia durable queda pendiente.

## Estado

La Fase 10 queda funcionalmente validada con:

- response action expansion,
- response routing,
- mock identity connector,
- canonical execution results,
- success / partial / failed / duplicate,
- intra-run idempotency,
- controlled retry,
- max attempts,
- terminal result merging,
- aggregation,
- audit trail.

La siguiente fase será la integración progresiva con **Floci Azure** como sandbox:

```text
Agentic SOC core
        ↓
generic response contracts
        ↓
Floci/Azure connector implementation
```

Antes de ejecutar acciones reales en Floci será necesario implementar idempotencia durable.
