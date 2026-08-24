# Agentic SOC Lab

Laboratorio open source para experimentar con un SOC agéntico ejecutado de forma local mediante **n8n**, **Ollama** y modelos **Qwen**.

El objetivo del proyecto es construir una arquitectura reproducible donde un sistema agéntico pueda recibir alertas de seguridad, normalizarlas, consultar distintas fuentes de evidencia, razonar sobre los resultados y producir un veredicto y acciones de respuesta de forma controlada.

> Estado actual: infraestructura base desplegada con Docker, n8n y Ollama. Qwen3 8B ejecutándose con aceleración GPU NVIDIA.

## Arquitectura inicial

```text
Alert / Scenario
      |
      v
+-----------------------+
|         n8n           |
|-----------------------|
| Alert Normalizer      |
| Investigation Planner |
| Tool Orchestration    |
| Evidence Aggregator   |
| Verdict / Response    |
+-----------+-----------+
            |
            v
+-----------------------+
|        Ollama         |
|       Qwen3:8b        |
+-----------------------+
```

El laboratorio se diseñará para integrarse posteriormente con un proyecto independiente de generación de escenarios empresariales y con entornos simulados como Floci.

## Principios del proyecto

- Ejecución local y sin dependencia obligatoria de modelos LLM de pago.
- Separación entre orquestación, razonamiento y herramientas.
- Contratos de entrada y salida definidos mediante esquemas.
- Secretos y credenciales fuera del código fuente.
- Componentes desacoplados y reemplazables.
- Desarrollo siguiendo principios de Secure SDLC.
- Infraestructura reproducible mediante Docker Compose.

## Componentes actuales

| Componente | Uso |
|---|---|
| Docker Desktop | Ejecución de contenedores |
| WSL2 | Backend Linux y virtualización en Windows |
| n8n | Orquestación de workflows y futuros agentes |
| Ollama | Runtime local para modelos LLM |
| Qwen3 8B | Modelo local inicial para razonamiento y tool calling |
| NVIDIA GPU | Aceleración del modelo |

## Hardware de referencia

La instalación inicial se ha validado con:

- CPU con virtualización habilitada.
- 32 GB de RAM.
- NVIDIA GeForce RTX 3060 Ti.
- 8 GB de VRAM.
- Windows con WSL2.
- Qwen3 8B cargado al 100 % en GPU.

Este hardware **no es un requisito exacto**, pero sirve como referencia para dimensionar el laboratorio. Equipos con menos RAM o VRAM pueden requerir modelos más pequeños o ejecutar parte del modelo en CPU, con una reducción importante del rendimiento.

## Estructura inicial

```text
agentic-soc-lab/
|
+-- docker-compose.yml
+-- .env
+-- .env.example
+-- .gitignore
+-- n8n/
|   +-- workflows/
|   +-- backups/
+-- ollama/
|   +-- models/
+-- contracts/
+-- docs/
+-- tests/
```

El archivo `.env` contiene configuración local y **no debe subirse al repositorio**.

## Instalación

La guía completa está disponible en:

1. [`docs/01-instalacion.md`](docs/01-instalacion.md)  
   WSL2, Docker Desktop, n8n, Ollama, Qwen y aceleración GPU.

2. [`docs/02-conexion-n8n-ollama.md`](docs/02-conexion-n8n-ollama.md)  
   Integración n8n -> Ollama -> Qwen3 8B.

3. [`docs/03-normalizacion-alertas.md`](docs/03-normalizacion-alertas.md)  
   Contrato canónico de alertas, normalización y validación.

4. [`docs/04-analisis-llm-inicial.md`](docs/04-analisis-llm-inicial.md)  
   Contexto de investigación, análisis estructurado con Qwen y validación determinista de la salida.

5. [`docs/05-investigation-planner.md`](docs/05-investigation-planner.md)  
   Generación de planes de investigación con Qwen, allowlist de herramientas, validación determinista de capacidades y parámetros, rama de error controlada y reparación automática de planes inválidos.

6. [`docs/06-tool-contracts-tool-router.md`](docs/06-tool-contracts-tool-router.md)  
   Contratos de herramientas, validación de requests, Tool Router, connector `get_signins` en modo mock, validación de salida y pruebas negativas de seguridad.
   
## Roadmap técnico

```text
Docker + WSL2
      |
      v
n8n + Ollama
      |
      v
Qwen local
      |
      v
Alert Contract
      |
      v
Alert Normalizer
      |
      v
Tool Interface
      |
      v
Investigation Planner
      |
      v
Evidence Aggregator
      |
      v
Verdict Engine
      |
      v
Floci / Scenario Generator
```

## Estado

Actualmente están validados:

- Docker Desktop sobre WSL2.
- n8n en contenedor.
- Ollama en contenedor.
- Acceso GPU NVIDIA desde Docker.
- Qwen3 8B cargado mediante Ollama.
- Ejecución de Qwen3 8B al 100 % en GPU.

La siguiente fase del proyecto será conectar **n8n -> Ollama -> Qwen** y construir el primer flujo de análisis de alertas.
