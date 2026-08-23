# Integración n8n con Ollama y Qwen

Esta guía documenta la segunda fase funcional de **Agentic SOC Lab**: la comunicación entre n8n, Ollama y el modelo local Qwen3 8B.

El objetivo de este bloque es validar una cadena mínima y controlada antes de introducir normalización de alertas, herramientas o agentes:

```text
n8n
  |
  v
Docker network
  |
  v
Ollama API
  |
  v
Qwen3:8b
  |
  v
NVIDIA GPU
  |
  v
Respuesta JSON
```

La integración descrita en este documento ha sido validada correctamente en el entorno de referencia del proyecto.

---

## 1. Prerrequisitos

Antes de continuar deben estar operativos:

- Docker Desktop.
- WSL2.
- n8n en contenedor.
- Ollama en contenedor.
- Qwen3 8B descargado.
- GPU NVIDIA accesible desde Docker.
- Qwen3 8B ejecutándose mediante GPU.

Comprobar los contenedores:

```powershell
docker compose ps
```

Resultado esperado:

```text
agentic-soc-n8n       Up
agentic-soc-ollama    Up
```

Comprobar que el modelo está disponible:

```powershell
docker exec -it agentic-soc-ollama ollama list
```

Debe aparecer:

```text
qwen3:8b
```

---

## 2. Acceso a n8n

Abrir n8n desde el navegador:

```text
http://localhost:5678
```

Crear un nuevo workflow.

Nombre utilizado durante la validación:

```text
01 - Ollama Connectivity Test
```

Este workflow se utiliza únicamente para verificar la comunicación con el LLM.

---

## 3. Crear el workflow de prueba

Añadir los siguientes nodos:

```text
Manual Trigger
      |
      v
HTTP Request
```

El nodo `Manual Trigger` permite ejecutar la prueba manualmente mientras se valida la infraestructura.

---

## 4. Configurar HTTP Request

Configurar el nodo `HTTP Request` con los siguientes valores:

```text
Method:
POST

URL:
http://ollama:11434/api/chat

Authentication:
None

Send Body:
Enabled

Body Content Type:
JSON
```

### Endpoint de Ollama

Dentro de Docker debe utilizarse:

```text
http://ollama:11434
```

y no:

```text
http://localhost:11434
```

La razón es que cada contenedor tiene su propio espacio de red.

Desde el contenedor de n8n, `localhost` hace referencia al propio contenedor de n8n, no al contenedor de Ollama.

Docker proporciona resolución DNS interna utilizando el nombre del servicio definido en `docker-compose.yml`. Por tanto, `ollama` resuelve automáticamente la dirección del contenedor correspondiente dentro de la red Docker.

---

## 5. Petición de prueba

Utilizar el siguiente body JSON:

```json
{
  "model": "qwen3:8b",
  "messages": [
    {
      "role": "system",
      "content": "You are a cybersecurity SOC analyst. Answer concisely and technically."
    },
    {
      "role": "user",
      "content": "What is an impossible travel alert? Answer in one sentence."
    }
  ],
  "stream": false
}
```

### Campos principales

`model`

```text
qwen3:8b
```

indica el modelo local que Ollama debe utilizar.

`messages` contiene el contexto enviado al modelo. El mensaje `system` define el rol general y el mensaje `user` contiene la consulta de prueba.

`stream: false` hace que Ollama devuelva la respuesta completa en una única respuesta HTTP, lo que simplifica inicialmente el tratamiento de datos desde n8n.

---

## 6. Ejecutar la prueba

Ejecutar el workflow manualmente desde n8n.

Si la integración funciona correctamente, el nodo `HTTP Request` debe recibir una respuesta JSON de Ollama.

La estructura será similar a:

```json
{
  "model": "qwen3:8b",
  "message": {
    "role": "assistant",
    "content": "..."
  },
  "done": true
}
```

El campo relevante para la respuesta generada es:

```text
message.content
```

En fases posteriores este valor será procesado y normalizado por otros nodos del workflow.

---

## 7. Métricas devueltas por Ollama

La API de Ollama puede devolver métricas como:

```text
total_duration
load_duration
prompt_eval_count
prompt_eval_duration
eval_count
eval_duration
```

Estas métricas no son necesarias para esta primera prueba, pero podrán utilizarse posteriormente para medir:

- latencia de inferencia;
- tiempo de carga del modelo;
- tokens de entrada;
- tokens generados;
- rendimiento;
- comparación entre modelos.

---

## 8. Validar uso de GPU

Mientras Qwen está cargado:

```powershell
docker exec -it agentic-soc-ollama ollama ps
```

En el entorno de referencia se obtuvo:

```text
NAME        SIZE      PROCESSOR    CONTEXT
qwen3:8b    5.6 GB    100% GPU     4096
```

El valor importante es:

```text
100% GPU
```

También puede comprobarse desde Windows:

```powershell
nvidia-smi
```

Durante una inferencia debería observarse actividad de GPU y uso de VRAM.

---

## 9. Troubleshooting

### Error de conexión

Si n8n devuelve un error similar a:

```text
Connection refused
```

comprobar:

```powershell
docker compose ps
```

y confirmar que Ollama está en estado `Up`.

También puede comprobarse Ollama desde el host:

```powershell
curl http://localhost:11434
```

### No se puede resolver `ollama`

Si aparece un error relacionado con resolución DNS:

```text
Could not resolve host ollama
```

comprobar:

```powershell
docker network ls
```

y revisar la red utilizada por el proyecto:

```powershell
docker network inspect <nombre_red>
```

Tanto `n8n` como `ollama` deben pertenecer a la misma red definida en `docker-compose.yml`.

### Ollama utiliza CPU

Comprobar:

```powershell
docker exec -it agentic-soc-ollama ollama ps
```

Si aparece:

```text
100% CPU
```

revisar la asignación GPU del servicio Ollama:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]
```

Después recrear los contenedores:

```powershell
docker compose down
docker compose up -d --force-recreate
```

No utilizar `-v` salvo que se quieran eliminar deliberadamente los volúmenes persistentes.

### Docker no detecta la GPU

Verificar directamente Docker:

```powershell
docker run --rm --gpus all nvidia/cuda:13.0.0-base-ubuntu24.04 nvidia-smi
```

Si este comando no detecta la GPU, revisar:

- drivers NVIDIA;
- Docker Desktop;
- backend WSL2;
- soporte GPU de WSL2;
- configuración de virtualización del host.

En el entorno de referencia se ha validado una NVIDIA GeForce RTX 3060 Ti con 8 GB de VRAM.

---

## 10. Criterio de éxito

Este bloque puede considerarse completado cuando:

- [ ] n8n está operativo.
- [ ] Ollama está operativo.
- [ ] Qwen3 8B está instalado.
- [ ] n8n puede resolver el servicio `ollama`.
- [ ] n8n envía una petición HTTP a `/api/chat`.
- [ ] Ollama devuelve una respuesta JSON válida.
- [ ] `message.content` contiene la respuesta del modelo.
- [ ] Qwen3 8B utiliza la GPU NVIDIA.
- [ ] El workflow puede ejecutarse de forma repetible.

---

## 11. Resultado de esta fase

Con esta integración validada, la infraestructura dispone ya de un canal funcional entre el orquestador y el modelo:

```text
n8n -> Ollama -> Qwen3:8b
```

La siguiente fase será introducir:

```text
Manual Alert Input
        |
        v
Alert Normalizer
        |
        v
Alert Schema
        |
        v
LLM Analysis
```

Todavía no se introducirán agentes autónomos. Primero se definirá un contrato de alerta estable y validable para que el resto de la arquitectura trabaje siempre sobre una estructura de datos conocida.
