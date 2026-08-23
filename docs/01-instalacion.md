# Instalación y despliegue inicial

Esta guía describe cómo desplegar la infraestructura base de **Agentic SOC Lab** sobre Windows utilizando Docker Desktop, WSL2, n8n, Ollama y Qwen.

El objetivo de esta fase es obtener la siguiente cadena funcional:

```text
Windows
  |
  v
WSL2
  |
  v
Docker Desktop
  |
  +--> n8n
  |
  +--> Ollama --> Qwen3:8b --> NVIDIA GPU
```

## 1. Requisitos

### Sistema operativo

La instalación documentada se ha realizado sobre Windows con WSL2.

Verificar la versión de WSL:

```powershell
wsl --version
```

Ejemplo del entorno utilizado:

```text
WSL: 2.6.2.0
Kernel: 6.6.87.2-1
```

Comprobar también:

```powershell
wsl --status
```

Debe utilizar WSL2 como versión predeterminada:

```text
Versión predeterminada: 2
```

### Hardware de referencia

El laboratorio se ha validado con:

```text
RAM:      32 GB
GPU:      NVIDIA GeForce RTX 3060 Ti
VRAM:     8 GB
Modelo:   Qwen3:8b
```

Qwen3 8B ocupa alrededor de 5-6 GB cuando está cargado en Ollama, por lo que una GPU de 8 GB permite ejecutarlo completamente en GPU en este entorno.

Con menos VRAM se puede:

- utilizar un modelo más pequeño;
- permitir descarga parcial a RAM/CPU;
- ejecutar únicamente sobre CPU.

Estas alternativas funcionan, pero aumentan la latencia.

---

## 2. Habilitar virtualización

La virtualización de CPU debe estar habilitada desde BIOS/UEFI.

En Windows puede verificarse desde:

```text
Administrador de tareas -> Rendimiento -> CPU -> Virtualización
```

Debe indicar:

```text
Habilitada
```

También puede comprobarse con:

```powershell
systeminfo
```

Entre los requisitos de Hyper-V deberían aparecer:

```text
Extensiones de modo de monitor de VM: Sí
Virtualización habilitada en firmware: Sí
Traducción de direcciones de segundo nivel: Sí
Prevención de ejecución de datos: Sí
```

---

## 3. Habilitar WSL y Virtual Machine Platform

WSL2 necesita la característica **Virtual Machine Platform**.

Abrir PowerShell o CMD **como administrador** y comprobar:

```powershell
dism /online /get-featureinfo /featurename:VirtualMachinePlatform
```

Debe aparecer:

```text
Estado : Habilitado
```

Si está deshabilitado:

```powershell
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

Comprobar también WSL:

```powershell
dism /online /get-featureinfo /featurename:Microsoft-Windows-Subsystem-Linux
```

Si fuera necesario:

```powershell
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

Después, reiniciar Windows.

### Error conocido: HCS/ERROR_NOT_SUPPORTED

Durante la instalación puede aparecer un error similar a:

```text
Wsl/Service/RegisterDistro/CreateVm/HCS/ERROR_NOT_SUPPORTED
```

Un caso reproducido durante el despliegue se debía a que la virtualización estaba habilitada en BIOS, pero `VirtualMachinePlatform` seguía deshabilitado en Windows.

También conviene comprobar:

```powershell
bcdedit /enum | findstr -i hypervisorlaunchtype
```

El valor esperado es:

```text
hypervisorlaunchtype    Auto
```

Si no aparece o está desactivado, ejecutar desde una consola con privilegios de administrador:

```powershell
bcdedit /set hypervisorlaunchtype auto
```

y reiniciar Windows.

---

## 4. Instalar Docker Desktop

Instalar Docker Desktop utilizando WSL2 como backend.

En Docker Desktop verificar:

```text
Settings -> General -> Use the WSL 2 based engine
```

Comprobar Docker Compose:

```powershell
docker compose version
```

El entorno de referencia utiliza Docker Compose 5.x.

---

## 5. Crear el proyecto

Ruta utilizada durante el desarrollo:

```text
Z:\projects\agentic-soc-lab
```

Crear la estructura:

```powershell
mkdir Z:\projects\agentic-soc-lab
cd Z:\projects\agentic-soc-lab
```

Estructura inicial:

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

---

## 6. Variables de entorno

Crear `.env`:

```env
N8N_PORT=5678
N8N_ENCRYPTION_KEY=GENERAR_UN_VALOR_ALEATORIO
TZ=Europe/Madrid
OLLAMA_PORT=11434
```

Generar una clave para n8n, por ejemplo:

```powershell
[Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }))
```

Crear `.env.example` sin secretos:

```env
N8N_PORT=5678
N8N_ENCRYPTION_KEY=CHANGE_ME
TZ=Europe/Madrid
OLLAMA_PORT=11434
```

`.gitignore`:

```gitignore
.env
*.log
n8n/backups/
```

Antes de publicar el repositorio:

```powershell
git check-ignore .env
```

Debe devolver:

```text
.env
```

---

## 7. Docker Compose

Configuración base:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: agentic-soc-n8n
    restart: unless-stopped

    ports:
      - "${N8N_PORT}:5678"

    environment:
      - TZ=${TZ}
      - GENERIC_TIMEZONE=${TZ}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}

    volumes:
      - n8n_data:/home/node/.n8n
      - ./n8n/workflows:/workflows

    networks:
      - soc_network

    depends_on:
      - ollama

  ollama:
    image: ollama/ollama:latest
    container_name: agentic-soc-ollama
    restart: unless-stopped

    ports:
      - "${OLLAMA_PORT}:11434"

    volumes:
      - ollama_data:/root/.ollama

    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

    networks:
      - soc_network

volumes:
  n8n_data:
  ollama_data:

networks:
  soc_network:
    driver: bridge
```

La sección `devices` permite que Ollama utilice la GPU NVIDIA.

---

## 8. Levantar los servicios

Ejecutar:

```powershell
docker compose up -d
```

Comprobar:

```powershell
docker compose ps
```

Resultado esperado:

```text
agentic-soc-n8n       Up
agentic-soc-ollama    Up
```

Puertos utilizados:

```text
n8n:       http://localhost:5678
Ollama:    http://localhost:11434
```

---

## 9. Validar acceso GPU desde Docker

Antes de diagnosticar Ollama, conviene comprobar que Docker puede acceder directamente a la GPU:

```powershell
docker run --rm --gpus all nvidia/cuda:13.0.0-base-ubuntu24.04 nvidia-smi
```

La GPU debe aparecer dentro del contenedor.

En el entorno de referencia:

```text
NVIDIA GeForce RTX 3060 Ti
VRAM: 8192 MiB
```

Si este comando falla, revisar antes de continuar:

- driver NVIDIA;
- WSL2;
- Docker Desktop;
- backend WSL2;
- soporte GPU del host.

Si este comando funciona pero Ollama utiliza CPU, el problema normalmente está en la configuración del contenedor de Ollama.

---

## 10. Instalar Qwen

Descargar el modelo:

```powershell
docker exec -it agentic-soc-ollama ollama pull qwen3:8b
```

Comprobar:

```powershell
docker exec -it agentic-soc-ollama ollama list
```

Ejecutar:

```powershell
docker exec -it agentic-soc-ollama ollama run qwen3:8b
```

---

## 11. Validar aceleración GPU de Ollama

Mientras el modelo está cargado:

```powershell
docker exec -it agentic-soc-ollama ollama ps
```

En el entorno de referencia:

```text
NAME        SIZE      PROCESSOR
qwen3:8b    5.6 GB    100% GPU
```

Si aparece:

```text
100% CPU
```

comprobar que `docker-compose.yml` contiene:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]
```

Recrear el contenedor:

```powershell
docker compose down
docker compose up -d --force-recreate
```

No utilizar:

```powershell
docker compose down -v
```

salvo que se quiera eliminar deliberadamente los volúmenes persistentes.

---

## 12. Persistencia

Los siguientes datos se almacenan en volúmenes Docker:

```text
n8n_data
ollama_data
```

Por tanto:

```powershell
docker compose down
```

no elimina workflows, configuración interna de n8n ni modelos descargados.

`docker compose down -v`, en cambio, elimina también los volúmenes.

---

## 13. Comunicación entre n8n y Ollama

Dentro de la red Docker, n8n **no debe utilizar**:

```text
http://localhost:11434
```

porque `localhost` desde el contenedor apunta al propio n8n.

Debe utilizar el nombre DNS del servicio:

```text
http://ollama:11434
```

La siguiente fase del proyecto utilizará esta dirección para conectar:

```text
n8n -> Ollama -> Qwen3:8b
```

---

## 14. Verificación final

Antes de continuar con los workflows, comprobar:

```powershell
docker compose ps
```

Los servicios deben estar `Up`.

```powershell
docker exec -it agentic-soc-ollama ollama list
```

Debe aparecer `qwen3:8b`.

```powershell
docker exec -it agentic-soc-ollama ollama ps
```

Con el modelo cargado debe indicar uso GPU.

Checklist:

- [ ] Virtualización habilitada en BIOS/UEFI.
- [ ] WSL2 instalado y actualizado.
- [ ] `VirtualMachinePlatform` habilitado.
- [ ] `hypervisorlaunchtype` en `Auto`.
- [ ] Docker Desktop funcionando sobre WSL2.
- [ ] n8n desplegado.
- [ ] Ollama desplegado.
- [ ] Docker accede a la GPU NVIDIA.
- [ ] Qwen3 8B descargado.
- [ ] Qwen3 8B ejecutándose con GPU.

Con estos requisitos cumplidos, la infraestructura base del laboratorio está preparada para comenzar la construcción del SOC agéntico.
