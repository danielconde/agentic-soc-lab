# Análisis LLM inicial estructurado

Esta guía documenta la cuarta fase funcional de **Agentic SOC Lab**: transformar una alerta canónica en un contexto de investigación controlado, enviarlo a Qwen mediante Ollama y validar una respuesta estructurada antes de permitir que continúe por el workflow.

El objetivo de esta fase no es todavía ejecutar herramientas ni realizar investigación autónoma. El objetivo es establecer una frontera fiable entre:

```text
Canonical Alert v1.0
        |
        v
Investigation Context v1.0
        |
        v
Qwen3:8b
        |
        v
Canonical Analysis v1.0
```

La salida del LLM se considera no confiable hasta superar una validación determinista.

## 1. Workflow utilizado

Nombre exacto:

```text
03 - Initial LLM Analysis
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
        |
        v
Investigation Context Builder
        |
        v
Ollama Request Builder
        |
        v
Ollama Initial Analysis
        |
        v
LLM Analysis Validator
```

### Responsabilidad de los nodos

- `Manual Alert Input`: genera la alerta simulada.
- `Alert Normalizer`: genera `Canonical Alert v1.0`.
- `Investigation Context Builder`: construye un contexto reducido, explicita relaciones conocidas y marca relaciones desconocidas.
- `Ollama Request Builder`: construye el request para Qwen y fija las reglas de evidencia.
- `Ollama Initial Analysis`: realiza la petición HTTP a Ollama.
- `LLM Analysis Validator`: parsea y valida la salida antes de permitir que continúe el workflow.

## 2. Dato, relación e inferencia

Durante las pruebas se detectó que el modelo relacionaba:

```text
source_ip: 185.220.101.20
destination_ip: 20.190.128.25
country_a: Spain
country_b: Netherlands
```

como si existiera:

```text
185.220.101.20 -> Spain
20.190.128.25 -> Netherlands
```

aunque esa relación nunca se había proporcionado.

Regla de diseño:

```text
dato != relación entre datos != inferencia
```

Por ello las ubicaciones se modelan como observaciones independientes y se declara explícitamente cuando no existe mapping con IPs o autenticaciones.

## 3. Investigation Context v1.0

El contexto incluye estructuras como:

```json
{
  "provider_context": {
    "authentication_method": "MFA",
    "device_managed": false,
    "geographic_observations": {
      "reported_locations": [
        "Spain",
        "Netherlands"
      ],
      "relationship_to_ip_addresses": "unknown",
      "relationship_to_user_authentication": "not_explicitly_provided"
    }
  },
  "evidence_constraints": {
    "geographic_mapping_explicit": false,
    "geographic_user_relationship_explicit": false,
    "threat_intelligence_available": false,
    "historical_signins_available": false,
    "endpoint_activity_available": false,
    "post_authentication_activity_available": false
  }
}
```

Esto reduce el riesgo de que el modelo convierta dos datos independientes en una relación no demostrada.

## 4. Política de evidencia

El modelo recibe reglas explícitas:

```text
Treat only explicitly provided fields and relationships as observed evidence.
Do not infer relationships between independent fields.
Do not associate geographic locations with IP addresses unless that mapping is explicitly provided.
Keep observed evidence separate from hypotheses.
```

El prompt ayuda a reducir errores, pero no sustituye la validación posterior.

## 5. Contrato de análisis

Ruta:

```text
contracts/analysis.schema.json
```

Versión:

```text
1.0
```

Estructura:

```text
Canonical Analysis
|
+-- analysis_version
+-- alert_id
+-- summary
+-- observed_evidence
+-- hypotheses
+-- risk_assessment
+-- missing_evidence
+-- preliminary_verdict
+-- confidence
+-- recommended_investigations
```

`observed_evidence` utiliza objetos estructurados y conserva tipos:

```json
{
  "field": "device_managed",
  "value": false
}
```

```json
{
  "field": "users",
  "value": [
    "john.doe@example.com"
  ]
}
```

## 6. Separación entre evidencia e hipótesis

La arquitectura distingue:

```text
observed_evidence
```

de:

```text
hypotheses
```

Un hecho explícito puede entrar en `observed_evidence`. Una explicación posible debe entrar en `hypotheses`.

## 7. Request a Ollama

`Ollama Request Builder` genera una petición para `qwen3:8b` con:

```json
{
  "stream": false,
  "think": false,
  "keep_alive": "5m",
  "format": "json"
}
```

- `format: "json"` solicita salida JSON.
- `think: false` simplifica la interfaz en esta fase.
- `keep_alive: "5m"` mantiene el modelo cargado durante ejecuciones próximas.

## 8. Ollama Initial Analysis

Configuración principal:

```text
Method: POST
URL: http://ollama:11434/api/chat
Authentication: None
Body Content Type: JSON
```

La respuesta llega inicialmente en:

```text
message.content
```

como un string que contiene JSON.

## 9. Validación de la respuesta

`LLM Analysis Validator` trata la salida del modelo como no confiable:

```text
message.content
      |
      v
JSON.parse()
      |
      v
Structural validation
      |
      v
Canonical Analysis v1.0
```

Valida, entre otros:

- `analysis_version`;
- `alert_id`;
- `summary`;
- `observed_evidence`;
- tipos de evidencia;
- `hypotheses`;
- `risk_assessment`;
- `missing_evidence`;
- `preliminary_verdict`;
- `confidence` entre 0 y 1;
- `recommended_investigations`.

## 10. Errores detectados durante las pruebas

Se validaron correctamente varios fallos:

```text
summary must be a non-empty string
```

y errores de tipo en `observed_evidence`.

El contrato y el validator se ajustaron para aceptar arrays, preservar booleanos y rechazar salidas incompatibles.

## 11. Resultado validado

Una ejecución final produjo una salida estructurada similar a:

```json
{
  "analysis_version": "1.0",
  "alert_id": "ALERT-0001",
  "summary": "User authenticated from geographically distant locations within an unusual time window.",
  "observed_evidence": [
    {
      "field": "users",
      "value": ["john.doe@example.com"]
    },
    {
      "field": "source_ips",
      "value": ["185.220.101.20"]
    },
    {
      "field": "geographic_observations.reported_locations",
      "value": ["Spain", "Netherlands"]
    }
  ],
  "hypotheses": [],
  "risk_assessment": {
    "level": "unknown",
    "reasoning": [
      "Geographic locations reported are not explicitly linked to IP addresses or user authentication events."
    ]
  },
  "missing_evidence": [
    "Explicit mapping of reported geographic locations to IP addresses or user authentication events.",
    "Historical sign-in data to assess unusual travel patterns.",
    "Endpoint activity details to confirm user behavior."
  ],
  "preliminary_verdict": "inconclusive",
  "confidence": 0,
  "recommended_investigations": [
    "Verify if the reported geographic locations are explicitly associated with the source or destination IP addresses.",
    "Collect historical sign-in data to analyze authentication patterns.",
    "Gather endpoint activity details to confirm user behavior."
  ]
}
```

El modelo ya no afirma asociaciones IP-país que no hayan sido demostradas.

## 12. Frontera de confianza del LLM

```text
Validated Input
      |
      v
Investigation Context
      |
      v
LLM
      |
      | UNTRUSTED OUTPUT
      v
LLM Analysis Validator
      |
      | VALIDATED OUTPUT
      v
Canonical Analysis
```

El LLM no actúa como mecanismo de validación de sí mismo.

## 13. Criterio de éxito

- [ ] `03 - Initial LLM Analysis` ejecuta completamente.
- [ ] `Alert Normalizer` produce `Canonical Alert v1.0`.
- [ ] `Investigation Context Builder` genera contexto controlado.
- [ ] Relaciones desconocidas se representan como desconocidas.
- [ ] Qwen devuelve JSON válido.
- [ ] `observed_evidence` conserva tipos.
- [ ] Evidencia e hipótesis están separadas.
- [ ] No se inventa asociación IP-país.
- [ ] `missing_evidence` usa lenguaje comprensible.
- [ ] `LLM Analysis Validator` acepta una salida válida.
- [ ] Una salida estructuralmente inválida es rechazada.

## 14. Estado de la arquitectura

```text
Raw Alert
    |
    v
Canonical Alert v1.0
    |
    v
Investigation Context v1.0
    |
    v
Qwen3:8b
    |
    v
Canonical Analysis v1.0
```

La siguiente fase introducirá el **Investigation Planner**, que transformará necesidades de evidencia en solicitudes estructuradas para tools controladas como `get_signins()`, `get_ip_reputation()` o `get_endpoint_activity()`.
