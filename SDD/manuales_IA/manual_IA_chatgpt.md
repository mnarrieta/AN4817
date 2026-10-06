---
title: "Manual técnico de instalación, configuración y operación: pila de IA local en Docker sobre Ubuntu"
author: "Mnarrieta"
date: "2026-10-06"
version: "1.0"
category: "Despliegue IA"
tags: [markdown, ia, Docker, Ubuntu, NVIDIA, CUDA, Ollama, OpenWebUI, Hermes, OpenCode, ComfyUI, YOLO, SearXNG, RAG, SDD]
---

# Manual técnico de instalación, configuración y operación

## Pila de IA local en Docker sobre Ubuntu Server

> **Base del documento:** especificación SDD proporcionada para el proyecto «Despliegue de una pila de IA Local en Docker sobre Ubuntu».
>
> **Objetivo de este manual:** proporcionar un procedimiento reproducible, orientado a un administrador de sistemas que parte de cero, para instalar Docker, habilitar GPU NVIDIA en contenedores y desplegar la pila de servicios definida en la especificación.
>
> **Criterio de diseño:** cada servicio se mantiene en su propio contenedor, todos los contenedores comparten la red Docker `red-ia` y los datos persistentes se almacenan en `$HOME/<servicio>`.

---

# 1. Alcance y decisiones de arquitectura

## 1.1. Servicios incluidos

| Servicio | Contenedor | Puerto interno | Puerto host | Función |
|---|---|---:|---:|---|
| Ollama | `ollama` | 11434 | 11434 | Inferencia LLM local y API |
| Open WebUI | `openwebui` | 8080 | 3000 | Interfaz web tipo ChatGPT |
| Hermes Agent | `hermes-agent` | 8000 | 8000 | Agente autónomo / API compatible con OpenAI |
| OpenCode | `opencode` | 4096 | 8443 | IDE web/agente de programación |
| ComfyUI | `comfyui` | 8188 | 8188 | Generación y procesamiento de imágenes/vídeo |
| YOLO | `yolo` | 5000 | 5000 | API de visión y detección de objetos |
| SearXNG | `searxng` | 8080 | 8080 | Metabuscador web privado |
| RAG | `rag` | 8001 | 8001 | Recuperación aumentada con información privada |

## 1.2. Correcciones necesarias respecto de la especificación original

La especificación original contiene dos puntos que producirían un despliegue inconsistente si se copian literalmente:

1. **RAG tenía asignado el puerto host `11434`, el mismo que Ollama.** Dos procesos no pueden publicar simultáneamente el mismo puerto en la misma IP del host. En esta implementación RAG publica `8001:8001`.
2. **OpenCode se definía con puerto interno 8080.** La versión actual del servidor web de OpenCode usa `4096` por defecto, por lo que se utiliza `8443:4096` para conservar el puerto externo indicado en la especificación. OpenCode permite fijar explícitamente el puerto y el host mediante `opencode web --port ... --hostname ...`. 

Estas decisiones mantienen la intención funcional de la especificación y permiten cumplir el criterio de aceptación de ausencia de colisiones.

## 1.3. Qué servicios utilizan realmente la GPU

No es necesario que todos los contenedores consuman GPU directamente.

- **GPU directa:** Ollama, ComfyUI, YOLO y, de forma opcional, Open WebUI mediante su imagen CUDA.
- **GPU indirecta:** Hermes Agent, OpenCode y RAG hacen peticiones a Ollama, que es quien ejecuta la inferencia.
- **CPU:** SearXNG trabaja principalmente como servicio web/metabuscador y no requiere CUDA.

La presencia de `gpus: all` debe reservarse a servicios que tengan una utilidad real para CUDA. Esto evita desperdiciar recursos del host.

---

# 2. Prerrequisitos e instalación base

## 2.1. Requisitos mínimos recomendados

Se recomienda un sistema con:

- Ubuntu Server 24.04 LTS o Ubuntu Server 26.04 LTS.
- Arquitectura `amd64` para la plataforma indicada en este manual.
- GPU NVIDIA compatible con el controlador Linux.
- SSD/NVMe con espacio suficiente para modelos, contenedores y datos generados.
- Al menos 16 GB de RAM para una instalación de laboratorio; 32 GB o más resulta más cómodo si se utilizan varias cargas simultáneas.
- Acceso administrativo mediante `sudo`.
- Conectividad a Internet para instalar paquetes, descargar imágenes Docker y obtener modelos.

La documentación actual de Docker lista Ubuntu 24.04 LTS y 26.04 LTS entre las versiones soportadas para Docker Engine. 

## 2.2. Actualizar el sistema operativo

Ejecutar:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
```

Reiniciar si el sistema ha actualizado el kernel:

```bash
sudo reboot
```

Después de volver a entrar:

```bash
uname -a
cat /etc/os-release
```

Comprobar que la distribución es Ubuntu:

```bash
. /etc/os-release
printf 'Ubuntu: %s\n' "$VERSION_ID"
printf 'Codename: %s\n' "$VERSION_CODENAME"
```

## 2.3. Comprobar la GPU NVIDIA del host

Antes de instalar cualquier contenedor GPU, verificar que el sistema operativo reconoce la tarjeta:

```bash
lspci | grep -i nvidia
```

Después comprobar el controlador:

```bash
nvidia-smi
```

El resultado debe mostrar el modelo de GPU, la versión del driver, memoria total y procesos que estén utilizando la tarjeta.

Si `nvidia-smi` no existe o muestra un error, **no continuar todavía con Docker GPU**. Primero hay que completar la instalación del driver.

## 2.4. Instalar el driver NVIDIA mediante APT

Ubuntu puede recomendar el controlador adecuado mediante `ubuntu-drivers`:

```bash
sudo apt update
sudo apt install -y ubuntu-drivers-common
ubuntu-drivers devices
```

Para instalar el controlador recomendado:

```bash
sudo ubuntu-drivers autoinstall
```

Reiniciar:

```bash
sudo reboot
```

Y comprobar de nuevo:

```bash
nvidia-smi
```

> **Importante:** no es necesario instalar el CUDA Toolkit completo del host para que los contenedores Docker utilicen las bibliotecas CUDA que suministra NVIDIA Container Toolkit. El elemento crítico es disponer de un driver NVIDIA correcto en el host y del runtime de contenedores NVIDIA.

---

# 3. Instalación de Docker Engine y Docker Compose

## 3.1. Eliminar paquetes que puedan entrar en conflicto

Docker recomienda eliminar paquetes antiguos o alternativos como `docker.io`, `docker-compose`, `docker-buildx`, `podman-docker`, `containerd` o `runc` cuando puedan entrar en conflicto con la instalación oficial.

```bash
sudo apt remove -y \
  docker.io \
  docker-compose \
  docker-compose-v2 \
  docker-doc \
  docker-buildx \
  podman-docker \
  containerd \
  runc
```

Si alguno no estaba instalado, APT puede indicar que no existe; no es un problema.

## 3.2. Instalar dependencias de APT

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

## 3.3. Añadir el repositorio oficial de Docker

Crear el directorio de claves:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Descargar la clave GPG oficial:

```bash
sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

Asignar permisos de lectura:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Crear el fichero de repositorio:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Actualizar índices:

```bash
sudo apt update
```

## 3.4. Instalar Docker Engine y Compose v2

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Verificar servicio:

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

Verificar versiones:

```bash
sudo docker version
sudo docker compose version
```

Ejecutar la prueba oficial de Docker:

```bash
sudo docker run --rm hello-world
```

## 3.5. Permitir utilizar Docker sin `sudo`

Añadir el usuario actual al grupo `docker`:

```bash
sudo usermod -aG docker "$USER"
```

Cerrar sesión y volver a entrar. Después probar:

```bash
docker ps
```

> **Advertencia de seguridad:** pertenecer al grupo `docker` equivale en la práctica a disponer de privilegios elevados sobre el host. Debe tratarse como un grupo administrativo.

Docker documenta además que la exposición de puertos mediante Docker puede interactuar de forma especial con firewalls como UFW; las reglas de filtrado deben revisarse en `DOCKER-USER` y no asumirse que publicar un puerto implica que UFW lo bloqueará automáticamente. 

---

# 4. Instalar NVIDIA Container Toolkit

## 4.1. Instalar prerrequisitos

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends \
  ca-certificates \
  curl \
  gnupg2
```

## 4.2. Añadir el repositorio oficial NVIDIA

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

```bash
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Actualizar índices:

```bash
sudo apt-get update
```

Instalar toolkit:

```bash
sudo apt-get install -y nvidia-container-toolkit
```

## 4.3. Configurar el runtime de Docker

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

Revisar la configuración generada:

```bash
sudo cat /etc/docker/daemon.json
```

Reiniciar Docker:

```bash
sudo systemctl restart docker
```

## 4.4. Comprobar CDI disponible

En versiones recientes de NVIDIA Container Toolkit puede verificarse el inventario CDI:

```bash
nvidia-ctk cdi list
```

La documentación actual de NVIDIA describe la instalación mediante APT, la configuración con `nvidia-ctk runtime configure` y el reinicio posterior del daemon Docker. 

## 4.5. Probar CUDA dentro de un contenedor

Antes de desplegar la pila completa, ejecutar una prueba mínima:

```bash
docker run --rm --gpus all \
  nvidia/cuda:12.8.1-base-ubuntu22.04 \
  nvidia-smi
```

Debe aparecer la misma GPU que en el host.

> **Alternativa:** las versiones recientes del ecosistema NVIDIA/Ultralytics recomiendan CDI para Linux en instalaciones suficientemente nuevas (`--device nvidia.com/gpu=all`). El stack de este manual utiliza `gpus: all` en Compose por simplicidad y amplia compatibilidad. Si se desea migrar a CDI, hacer la prueba de CDI por separado antes de cambiar el stack completo.

---

# 5. Crear el usuario de trabajo y la estructura del proyecto

## 5.1. Directorio raíz

La especificación establece:

```text
$HOME/proyecto
```

Crear la estructura:

```bash
mkdir -p "$HOME/proyecto"
cd "$HOME/proyecto"
```

## 5.2. Árbol completo propuesto

```text
$HOME/proyecto/
├── .env
├── .gitignore
├── README.md
├── docker-ollama.yml
├── docker-openwebui.yml
├── docker-hermes-agent.yml
├── docker-opencode.yml
├── docker-comfyui.yml
├── docker-yolo.yml
├── docker-searxng.yml
├── docker-rag.yml
├── compose-all.sh
├── compose-down.sh
│
├── ollama/
│   └── models/                       # opcional; principal persistencia en $HOME/ollama
│
├── openwebui/
│   └── data/
│
├── hermes/
│   ├── config.yaml
│   └── .env                          # opcional; configuración propia de Hermes
│
├── opencode/
│   ├── opencode.json
│   ├── Dockerfile
│   ├── searxng_mcp.py
│   └── mcp-requirements.txt
│
├── comfyui/
│   ├── Dockerfile
│   ├── models/
│   ├── input/
│   ├── output/
│   ├── custom_nodes/
│   └── user/
│
├── yolo/
│   ├── Dockerfile
│   ├── server.py
│   ├── requirements.txt
│   ├── models/
│   ├── datasets/
│   └── results/
│
├── searxng/
│   └── settings.yml
│
└── rag/
    ├── Dockerfile
    ├── requirements.txt
    ├── app.py
    ├── documents/
    └── db/
```

## 5.3. Crear los directorios

```bash
cd "$HOME/proyecto"

mkdir -p \
  ollama \
  openwebui/data \
  hermes \
  opencode \
  comfyui/models \
  comfyui/input \
  comfyui/output \
  comfyui/custom_nodes \
  comfyui/user \
  yolo/models \
  yolo/datasets \
  yolo/results \
  searxng \
  rag/documents \
  rag/db
```

---

# 6. Red Docker común `red-ia`

Crear la red una sola vez:

```bash
docker network create --driver bridge red-ia
```

Comprobar:

```bash
docker network inspect red-ia
```

Todos los contenedores del stack deben aparecer conectados a esta red una vez arrancados.

La resolución DNS interna de Docker permitirá que los contenedores se encuentren por nombre:

```text
ollama
openwebui
hermes-agent
opencode
comfyui
searxng
yolo
rag
```

No se debe usar `localhost` para comunicarse con otro contenedor. Dentro de `openwebui`, por ejemplo, `localhost` significa el propio contenedor Open WebUI, no Ollama.

---

# 7. Fichero de entorno `.env`

Crear:

```bash
nano "$HOME/proyecto/.env"
```

Contenido recomendado:

```dotenv
# ============================================================
# Puertos del host
# ============================================================
OLLAMA_HOST_PORT=11434
OPENWEBUI_HOST_PORT=3000
HERMES_HOST_PORT=8000
OPENCODE_HOST_PORT=8443
COMFYUI_HOST_PORT=8188
YOLO_HOST_PORT=5000
SEARXNG_HOST_PORT=8080
RAG_HOST_PORT=8001

# ============================================================
# Modelos Ollama
# ============================================================
OLLAMA_CHAT_MODEL=qwen3:8b
OLLAMA_EMBED_MODEL=nomic-embed-text
RAG_LLM_MODEL=qwen3:8b
YOLO_MODEL=yolo26n.pt

# ============================================================
# Open WebUI
# ============================================================
WEBUI_SECRET_KEY=REEMPLAZAR_POR_UNA_CLAVE_LARGA_Y_ALEATORIA
WEBUI_URL=http://localhost:3000
ENABLE_WEB_SEARCH=true
WEB_SEARCH_ENGINE=searxng
WEB_SEARCH_RESULT_COUNT=5
WEB_SEARCH_CONCURRENT_REQUESTS=5
SEARXNG_QUERY_URL=http://searxng:8080/search
COMFYUI_BASE_URL=http://comfyui:8188/

# ============================================================
# Hermes Agent
# ============================================================
HERMES_UID=1000
HERMES_GID=1000
HERMES_API_KEY=REEMPLAZAR_POR_OTRA_CLAVE_LARGA_Y_ALEATORIA
API_SERVER_HOST=0.0.0.0
API_SERVER_PORT=8000
API_SERVER_ENABLED=true

# ============================================================
# OpenCode
# ============================================================
OPENCODE_SERVER_USERNAME=opencode
OPENCODE_SERVER_PASSWORD=REEMPLAZAR_POR_OTRA_CLAVE_LARGA_Y_ALEATORIA
OPENCODE_ENABLE_EXA=false

# ============================================================
# RAG
# ============================================================
RAG_OLLAMA_BASE_URL=http://ollama:11434
RAG_TOP_K=5
RAG_CHUNK_SIZE=1200
RAG_CHUNK_OVERLAP=180

# ============================================================
# Localización
# ============================================================
TZ=Europe/Madrid
```

Generar secretos seguros en el host:

```bash
openssl rand -hex 32
```

Repetir para las claves de Open WebUI, Hermes y OpenCode.

> **Nunca** subir `.env` a Git.

Crear `.gitignore`:

```gitignore
.env
hermes/.env
__pycache__/
*.pyc
rag/db/*
yolo/results/*
comfyui/output/*
comfyui/input/*
```

---

# 8. Configuración de Ollama

## 8.1. `docker-ollama.yml`

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "${OLLAMA_HOST_PORT:-11434}:11434"
    environment:
      OLLAMA_HOST: 0.0.0.0:11434
      OLLAMA_KEEP_ALIVE: 24h
      OLLAMA_NUM_PARALLEL: 1
      OLLAMA_MAX_LOADED_MODELS: 1
    volumes:
      - ${HOME}/ollama:/root/.ollama
    gpus: all
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 8.2. Arranque

```bash
cd "$HOME/proyecto"
docker compose -f docker-ollama.yml up -d
```

Comprobar:

```bash
docker ps --filter name=ollama
```

Logs:

```bash
docker logs -f ollama
```

Prueba de API:

```bash
curl http://127.0.0.1:11434/api/tags
```

## 8.3. Descargar modelos

```bash
docker exec -it ollama ollama pull qwen3:8b
docker exec -it ollama ollama pull nomic-embed-text
```

Ver modelos:

```bash
docker exec -it ollama ollama list
```

El catálogo actual de Ollama mantiene `qwen3:8b` como una variante disponible de Qwen3 y la ficha del modelo indica soporte para herramientas. 

## 8.4. Prueba de inferencia

```bash
curl http://127.0.0.1:11434/api/chat \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3:8b",
    "messages": [
      {"role": "user", "content": "Responde con una sola frase: Ollama está funcionando."}
    ],
    "stream": false
  }'
```

## 8.5. Verificar GPU de Ollama

Mientras se realiza una consulta:

```bash
watch -n 1 nvidia-smi
```

También:

```bash
docker exec -it ollama sh -lc 'nvidia-smi'
```

---

# 9. Open WebUI

## 9.1. `docker-openwebui.yml`

```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:cuda
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "${OPENWEBUI_HOST_PORT:-3000}:8080"
    environment:
      OLLAMA_BASE_URL: http://ollama:11434
      ENABLE_WEB_SEARCH: "${ENABLE_WEB_SEARCH:-true}"
      WEB_SEARCH_ENGINE: "${WEB_SEARCH_ENGINE:-searxng}"
      WEB_SEARCH_RESULT_COUNT: "${WEB_SEARCH_RESULT_COUNT:-5}"
      WEB_SEARCH_CONCURRENT_REQUESTS: "${WEB_SEARCH_CONCURRENT_REQUESTS:-5}"
      SEARXNG_QUERY_URL: "${SEARXNG_QUERY_URL:-http://searxng:8080/search}"
      SEARXNG_LANGUAGE: all
      COMFYUI_BASE_URL: "${COMFYUI_BASE_URL:-http://comfyui:8188/}"
      ENABLE_IMAGE_GENERATION: "True"
      WEBUI_SECRET_KEY: "${WEBUI_SECRET_KEY}"
      WEBUI_URL: "${WEBUI_URL:-http://localhost:3000}"
      TZ: "${TZ:-Europe/Madrid}"
    volumes:
      - ${HOME}/openwebui:/app/backend/data
    extra_hosts:
      - "host.docker.internal:host-gateway"
    gpus: all
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

Open WebUI documenta la conexión a Ollama mediante `OLLAMA_BASE_URL`. También documenta que SearXNG puede integrarse mediante `WEB_SEARCH_ENGINE=searxng` y `SEARXNG_QUERY_URL`, y que ComfyUI puede utilizarse como motor de generación de imágenes. 

## 9.2. Arrancar

```bash
cd "$HOME/proyecto"
docker compose -f docker-openwebui.yml up -d
```

Comprobar:

```bash
docker ps --filter name=openwebui
docker logs --tail 100 openwebui
```

Acceso:

```text
http://IP_DEL_SERVIDOR:3000
```

## 9.3. Comprobar conexión con Ollama

```bash
docker exec -it openwebui sh -lc 'python - <<"PY"
import urllib.request
print(urllib.request.urlopen("http://ollama:11434/api/tags", timeout=10).read().decode())
PY'
```

---

# 10. Hermes Agent

## 10.1. Consideración sobre el puerto

La API HTTP actual de Hermes Agent utiliza `8642` por defecto, pero la especificación de este proyecto quiere el servicio en `8000`. Se fija explícitamente `API_SERVER_PORT=8000`.

Hermes requiere una clave para la API compatible con OpenAI cuando está habilitada. La documentación actual muestra `API_SERVER_ENABLED`, `API_SERVER_KEY`, `API_SERVER_HOST` y `API_SERVER_PORT` como variables relevantes. 

## 10.2. `hermes/config.yaml`

```yaml
model:
  default: qwen3:8b
  provider: custom
  base_url: http://ollama:11434/v1
```

Hermes documenta la utilización de un endpoint personalizado OpenAI-compatible para Ollama mediante `provider: custom` y `base_url: http://...:11434/v1`. 

## 10.3. `docker-hermes-agent.yml`

```yaml
services:
  hermes-agent:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-agent
    restart: unless-stopped
    ports:
      - "${HERMES_HOST_PORT:-8000}:8000"
    environment:
      HERMES_UID: "${HERMES_UID:-1000}"
      HERMES_GID: "${HERMES_GID:-1000}"
      API_SERVER_ENABLED: "${API_SERVER_ENABLED:-true}"
      API_SERVER_HOST: "${API_SERVER_HOST:-0.0.0.0}"
      API_SERVER_PORT: "${API_SERVER_PORT:-8000}"
      API_SERVER_KEY: "${HERMES_API_KEY}"
      API_SERVER_MODEL_NAME: hermes-agent
      TZ: "${TZ:-Europe/Madrid}"
    volumes:
      - ${HOME}/hermes:/opt/data
    command: ["gateway", "run"]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 10.4. Arrancar Hermes

```bash
cd "$HOME/proyecto"
docker compose -f docker-hermes-agent.yml up -d
```

Logs:

```bash
docker logs -f hermes-agent
```

## 10.5. Probar `/health`

```bash
curl http://127.0.0.1:8000/health
```

## 10.6. Probar `/v1/models`

```bash
curl http://127.0.0.1:8000/v1/models \
  -H "Authorization: Bearer ${HERMES_API_KEY}"
```

## 10.7. Probar chat

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Authorization: Bearer ${HERMES_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "messages": [
      {"role": "user", "content": "Di solamente: Hermes conectado a Ollama."}
    ]
  }'
```

---

# 11. OpenCode

## 11.1. Consideraciones actuales

OpenCode dispone de interfaz web mediante `opencode web`. La documentación actual indica que se puede fijar el puerto y el host, y que el servidor usa `4096` como puerto predeterminado. También admite Ollama como proveedor OpenAI-compatible. 

El stack añade un pequeño puente MCP local para que OpenCode pueda invocar SearXNG por el nombre `searxng`. OpenCode admite servidores MCP locales y remotos. 

## 11.2. `opencode/opencode.json`

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://ollama:11434/v1"
      },
      "models": {
        "qwen3:8b": {
          "name": "Qwen3 8B"
        }
      }
    }
  },
  "permission": {
    "bash": "ask",
    "edit": "ask",
    "external_directory": "ask",
    "webfetch": "allow",
    "websearch": "deny"
  },
  "mcp": {
    "searxng": {
      "type": "local",
      "command": ["python3", "/opt/searxng_mcp.py"],
      "enabled": true
    }
  }
}
```

## 11.3. `opencode/searxng_mcp.py`

```python
#!/usr/bin/env python3
"""MCP sencillo para consultar una instancia SearXNG del stack local."""

from __future__ import annotations

import json
import os
import urllib.parse
import urllib.request
from typing import Any

from mcp.server.fastmcp import FastMCP


SEARXNG_URL = os.environ.get(
    "SEARXNG_URL",
    "http://searxng:8080/search",
)

mcp = FastMCP("searxng")


@mcp.tool()
def search(query: str, language: str = "all", limit: int = 5) -> str:
    """Busca en SearXNG y devuelve resultados resumidos en JSON."""
    if not query.strip():
        raise ValueError("La consulta no puede estar vacía")

    limit = max(1, min(int(limit), 10))

    params = urllib.parse.urlencode(
        {
            "q": query,
            "format": "json",
            "language": language,
        }
    )
    url = f"{SEARXNG_URL}?{params}"

    request = urllib.request.Request(
        url,
        headers={"User-Agent": "opencode-searxng-mcp/1.0"},
    )

    with urllib.request.urlopen(request, timeout=20) as response:
        data: dict[str, Any] = json.loads(response.read().decode("utf-8"))

    output = []
    for result in data.get("results", [])[:limit]:
        output.append(
            {
                "title": result.get("title"),
                "url": result.get("url"),
                "content": result.get("content"),
                "engine": result.get("engine"),
            }
        )

    return json.dumps(output, ensure_ascii=False, indent=2)


if __name__ == "__main__":
    mcp.run()
```

## 11.4. `opencode/mcp-requirements.txt`

```text
mcp>=1.0,<2
```

## 11.5. `opencode/Dockerfile`

```dockerfile
FROM ghcr.io/anomalyco/opencode:latest

USER root

RUN apk add --no-cache python3 py3-pip
RUN python3 -m venv /opt/mcp-venv

COPY mcp-requirements.txt /tmp/mcp-requirements.txt
RUN /opt/mcp-venv/bin/pip install --no-cache-dir -r /tmp/mcp-requirements.txt

COPY searxng_mcp.py /opt/searxng_mcp.py

ENV PATH="/opt/mcp-venv/bin:${PATH}"
ENV SEARXNG_URL="http://searxng:8080/search"

WORKDIR /workspace
```

## 11.6. `docker-opencode.yml`

```yaml
services:
  opencode:
    build:
      context: ./opencode
      dockerfile: Dockerfile
    image: proyecto-opencode:latest
    container_name: opencode
    restart: unless-stopped
    ports:
      - "${OPENCODE_HOST_PORT:-8443}:4096"
    environment:
      OPENCODE_SERVER_USERNAME: "${OPENCODE_SERVER_USERNAME:-opencode}"
      OPENCODE_SERVER_PASSWORD: "${OPENCODE_SERVER_PASSWORD}"
      SEARXNG_URL: http://searxng:8080/search
      OPENCODE_ENABLE_EXA: "${OPENCODE_ENABLE_EXA:-false}"
      TZ: "${TZ:-Europe/Madrid}"
    working_dir: /workspace
    volumes:
      - ${HOME}/proyecto:/workspace
      - ${HOME}/proyecto/opencode/opencode.json:/workspace/opencode.json:ro
      - ${HOME}/opencode:/root/.config/opencode
    command: ["web", "--hostname", "0.0.0.0", "--port", "4096"]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

> El Dockerfile oficial de OpenCode publicado en su repositorio utiliza Alpine como base y un binario `opencode`; la instalación oficial de OpenCode también ofrece una imagen Docker en `ghcr.io/anomalyco/opencode`. 

## 11.7. Arrancar y verificar

```bash
cd "$HOME/proyecto"
docker compose -f docker-opencode.yml up -d --build
```

Logs:

```bash
docker logs -f opencode
```

Acceso:

```text
http://IP_DEL_SERVIDOR:8443
```

Comprobar salud:

```bash
curl -u "${OPENCODE_SERVER_USERNAME}:${OPENCODE_SERVER_PASSWORD}" \
  http://127.0.0.1:8443/global/health
```

Documentación OpenAPI:

```bash
curl -u "${OPENCODE_SERVER_USERNAME}:${OPENCODE_SERVER_PASSWORD}" \
  http://127.0.0.1:8443/doc
```

---

# 12. ComfyUI

## 12.1. Estrategia

Se construye una imagen Docker a partir del repositorio de ComfyUI y se añade PyTorch con CUDA, evitando depender de una imagen no oficial concreta.

## 12.2. `comfyui/Dockerfile`

```dockerfile
FROM nvidia/cuda:12.8.1-cudnn-runtime-ubuntu22.04

ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    python3-venv \
    git \
    ffmpeg \
    libgl1 \
    libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app/ComfyUI

RUN git clone --depth 1 https://github.com/comfyanonymous/ComfyUI.git .

RUN python3 -m pip install --no-cache-dir --break-system-packages \
    torch torchvision torchaudio \
    --index-url https://download.pytorch.org/whl/cu128

RUN python3 -m pip install --no-cache-dir --break-system-packages \
    -r requirements.txt

RUN useradd --create-home --uid 1000 --shell /bin/bash comfy \
    && chown -R comfy:comfy /app

USER comfy

EXPOSE 8188

CMD ["python3", "main.py", "--listen", "0.0.0.0", "--port", "8188"]
```

## 12.3. `docker-comfyui.yml`

```yaml
services:
  comfyui:
    build:
      context: ./comfyui
      dockerfile: Dockerfile
    image: proyecto-comfyui:latest
    container_name: comfyui
    restart: unless-stopped
    ports:
      - "${COMFYUI_HOST_PORT:-8188}:8188"
    environment:
      TZ: "${TZ:-Europe/Madrid}"
    volumes:
      - ${HOME}/proyecto/comfyui/models:/app/ComfyUI/models
      - ${HOME}/proyecto/comfyui/input:/app/ComfyUI/input
      - ${HOME}/proyecto/comfyui/output:/app/ComfyUI/output
      - ${HOME}/proyecto/comfyui/custom_nodes:/app/ComfyUI/custom_nodes
      - ${HOME}/proyecto/comfyui/user:/app/ComfyUI/user
    gpus: all
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 12.4. Construcción y arranque

```bash
cd "$HOME/proyecto"
docker compose -f docker-comfyui.yml build
docker compose -f docker-comfyui.yml up -d
```

Acceso:

```text
http://IP_DEL_SERVIDOR:8188
```

## 12.5. Estructura de modelos

```text
$HOME/proyecto/comfyui/models/
├── checkpoints/
├── vae/
├── loras/
├── controlnet/
├── clip/
└── upscale_models/
```

No se incluyen pesos propietarios. Descargar siempre desde una fuente y licencia compatibles.

## 12.6. Verificar CUDA

```bash
docker exec -it comfyui nvidia-smi
```

---

# 13. YOLO como API de visión

## 13.1. Objetivo

YOLO se publica como API HTTP para que otros servicios puedan enviar imágenes y recibir detecciones estructuradas.

Ultralytics documenta imágenes Docker con soporte GPU NVIDIA y el uso de volúmenes para conservar datasets y resultados. 

## 13.2. `yolo/requirements.txt`

```text
fastapi>=0.115,<1
uvicorn[standard]>=0.30,<1
python-multipart>=0.0.9,<1
```

## 13.3. `yolo/server.py`

```python
from __future__ import annotations

import os
import tempfile
from pathlib import Path
from typing import Any

from fastapi import FastAPI, File, HTTPException, UploadFile
from ultralytics import YOLO

MODEL_NAME = os.getenv("YOLO_MODEL", "yolo26n.pt")
DEVICE = os.getenv("YOLO_DEVICE", "0")
RESULT_DIR = Path(os.getenv("YOLO_RESULT_DIR", "/app/results"))
RESULT_DIR.mkdir(parents=True, exist_ok=True)

app = FastAPI(title="YOLO Local API", version="1.0")
model = YOLO(MODEL_NAME)


@app.get("/health")
def health() -> dict[str, Any]:
    return {"status": "ok", "model": MODEL_NAME, "device": DEVICE}


@app.post("/predict")
async def predict(file: UploadFile = File(...)) -> dict[str, Any]:
    suffix = Path(file.filename or "upload.bin").suffix or ".bin"
    input_path: Path | None = None

    try:
        content = await file.read()
        with tempfile.NamedTemporaryFile(suffix=suffix, delete=False) as tmp:
            tmp.write(content)
            input_path = Path(tmp.name)

        results = model.predict(
            source=str(input_path),
            device=DEVICE,
            save=True,
            project=str(RESULT_DIR),
            name="predict",
            exist_ok=True,
            verbose=False,
        )

        output: list[dict[str, Any]] = []
        for result in results:
            names = result.names
            detections = []
            if result.boxes is not None:
                for box in result.boxes:
                    cls = int(box.cls.item())
                    conf = float(box.conf.item())
                    xyxy = [round(float(v), 3) for v in box.xyxy[0].tolist()]
                    detections.append(
                        {
                            "class_id": cls,
                            "class_name": names.get(cls, str(cls)),
                            "confidence": conf,
                            "xyxy": xyxy,
                        }
                    )
            output.append({"detections": detections})

        return {
            "status": "ok",
            "model": MODEL_NAME,
            "device": DEVICE,
            "results": output,
        }

    except Exception as exc:
        raise HTTPException(status_code=500, detail=str(exc)) from exc

    finally:
        if input_path is not None:
            input_path.unlink(missing_ok=True)
```

## 13.4. `yolo/Dockerfile`

```dockerfile
FROM ultralytics/ultralytics:latest

WORKDIR /app

COPY requirements.txt /app/requirements.txt
RUN python -m pip install --no-cache-dir -r /app/requirements.txt

COPY server.py /app/server.py

RUN mkdir -p /app/models /app/datasets /app/results

EXPOSE 5000

CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "5000"]
```

## 13.5. `docker-yolo.yml`

```yaml
services:
  yolo:
    build:
      context: ./yolo
      dockerfile: Dockerfile
    image: proyecto-yolo:latest
    container_name: yolo
    restart: unless-stopped
    ports:
      - "${YOLO_HOST_PORT:-5000}:5000"
    environment:
      YOLO_MODEL: "${YOLO_MODEL:-yolo26n.pt}"
      YOLO_DEVICE: "0"
      YOLO_RESULT_DIR: /app/results
      TZ: "${TZ:-Europe/Madrid}"
    volumes:
      - ${HOME}/proyecto/yolo/models:/app/models
      - ${HOME}/proyecto/yolo/datasets:/app/datasets
      - ${HOME}/proyecto/yolo/results:/app/results
    gpus: all
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 13.6. Arrancar y probar

```bash
cd "$HOME/proyecto"
docker compose -f docker-yolo.yml up -d --build
curl http://127.0.0.1:5000/health
```

Con una imagen `foto.jpg`:

```bash
curl -X POST \
  -F "file=@foto.jpg" \
  http://127.0.0.1:5000/predict
```

---

# 14. SearXNG

## 14.1. Objetivo

SearXNG se desplegará como servicio web privado en `red-ia`. Open WebUI lo utilizará para búsquedas web. OpenCode lo utilizará mediante el puente MCP.

La documentación actual de SearXNG recomienda Compose para despliegues en contenedor y expone configuración persistente bajo `/etc/searxng` y caché bajo `/var/cache/searxng`. 

## 14.2. `searxng/settings.yml`

```yaml
use_default_settings: true

server:
  bind_address: "0.0.0.0"
  port: 8080
  secret_key: "REEMPLAZAR_POR_UN_SECRET_DE_SEARXNG"
  limiter: false

search:
  safe_search: 1
  autocomplete: ""
  default_lang: "all"
  formats:
    - html
    - json

general:
  instance_name: "SearXNG IA Local"
```

## 14.3. `docker-searxng.yml`

```yaml
services:
  searxng:
    image: docker.io/searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    ports:
      - "${SEARXNG_HOST_PORT:-8080}:8080"
    volumes:
      - ${HOME}/proyecto/searxng:/etc/searxng:rw
      - searxng_cache:/var/cache/searxng:rw
    environment:
      SEARXNG_BASE_URL: http://localhost:8080/
      TZ: "${TZ:-Europe/Madrid}"
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE
    networks:
      - red-ia

volumes:
  searxng_cache:

networks:
  red-ia:
    external: true
```

## 14.4. Arrancar y probar

```bash
cd "$HOME/proyecto"
docker compose -f docker-searxng.yml up -d
```

HTML:

```bash
curl 'http://127.0.0.1:8080/search?q=Ubuntu&format=html'
```

JSON:

```bash
curl 'http://127.0.0.1:8080/search?q=Ubuntu&format=json'
```

Prueba desde Open WebUI:

```bash
docker exec -it openwebui sh -lc 'python - <<"PY"
import urllib.request
url = "http://searxng:8080/search?q=Ubuntu&format=json"
r = urllib.request.urlopen(url, timeout=15)
print(r.status)
print(r.read(300).decode())
PY'
```

---

# 15. RAG local

## 15.1. Objetivo

El servicio RAG:

1. Lee documentos privados.
2. Divide los documentos en fragmentos.
3. Obtiene embeddings con Ollama.
4. Guarda vectores localmente.
5. Recupera los fragmentos más cercanos a una pregunta.
6. Genera una respuesta mediante Ollama usando el contexto recuperado.
7. Devuelve las fuentes utilizadas.

## 15.2. Documentos admitidos

```text
.txt
.md
.markdown
.pdf
```

Copiar documentos a:

```text
$HOME/proyecto/rag/documents/
```

## 15.3. `rag/requirements.txt`

```text
fastapi>=0.115,<1
uvicorn[standard]>=0.30,<1
numpy>=2,<3
requests>=2.32,<3
pymupdf>=1.24,<2
```

## 15.4. `rag/app.py`

```python
from __future__ import annotations

import hashlib
import json
import os
import re
from pathlib import Path
from typing import Any

import fitz
import numpy as np
import requests
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

OLLAMA_BASE_URL = os.getenv("RAG_OLLAMA_BASE_URL", "http://ollama:11434")
EMBED_MODEL = os.getenv("OLLAMA_EMBED_MODEL", "nomic-embed-text")
LLM_MODEL = os.getenv("RAG_LLM_MODEL", "qwen3:8b")
DOC_DIR = Path(os.getenv("RAG_DOCUMENT_DIR", "/data/documents"))
DB_DIR = Path(os.getenv("RAG_DB_DIR", "/data/db"))
CHUNK_SIZE = int(os.getenv("RAG_CHUNK_SIZE", "1200"))
CHUNK_OVERLAP = int(os.getenv("RAG_CHUNK_OVERLAP", "180"))
DEFAULT_TOP_K = int(os.getenv("RAG_TOP_K", "5"))

DB_DIR.mkdir(parents=True, exist_ok=True)
DOC_DIR.mkdir(parents=True, exist_ok=True)

VECTORS_FILE = DB_DIR / "vectors.npz"
META_FILE = DB_DIR / "metadata.json"

app = FastAPI(title="RAG Local API", version="1.0")


class QueryRequest(BaseModel):
    question: str = Field(min_length=1)
    top_k: int = Field(default=DEFAULT_TOP_K, ge=1, le=10)


def read_file(path: Path) -> str:
    suffix = path.suffix.lower()
    if suffix in {".txt", ".md", ".markdown"}:
        return path.read_text(encoding="utf-8", errors="ignore")
    if suffix == ".pdf":
        with fitz.open(path) as pdf:
            return "\n".join(page.get_text() for page in pdf)
    return ""


def normalize(text: str) -> str:
    return re.sub(r"\s+", " ", text).strip()


def chunk_text(text: str) -> list[str]:
    text = normalize(text)
    if not text:
        return []

    chunks: list[str] = []
    start = 0
    while start < len(text):
        end = min(len(text), start + CHUNK_SIZE)
        chunk = text[start:end].strip()
        if chunk:
            chunks.append(chunk)
        if end >= len(text):
            break
        start = max(0, end - CHUNK_OVERLAP)
    return chunks


def ollama_embed(texts: list[str]) -> list[list[float]]:
    response = requests.post(
        f"{OLLAMA_BASE_URL}/api/embed",
        json={"model": EMBED_MODEL, "input": texts},
        timeout=180,
    )
    response.raise_for_status()
    return response.json()["embeddings"]


def save_db(vectors: np.ndarray, metadata: list[dict[str, Any]]) -> None:
    np.savez_compressed(VECTORS_FILE, vectors=vectors)
    META_FILE.write_text(
        json.dumps(metadata, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )


def load_db() -> tuple[np.ndarray, list[dict[str, Any]]]:
    if not VECTORS_FILE.exists() or not META_FILE.exists():
        return np.empty((0, 0), dtype=np.float32), []
    vectors = np.load(VECTORS_FILE)["vectors"].astype(np.float32)
    metadata = json.loads(META_FILE.read_text(encoding="utf-8"))
    return vectors, metadata


def ingest() -> dict[str, Any]:
    metadata: list[dict[str, Any]] = []
    texts: list[str] = []

    for path in sorted(DOC_DIR.rglob("*")):
        if not path.is_file():
            continue
        if path.suffix.lower() not in {".txt", ".md", ".markdown", ".pdf"}:
            continue

        content = read_file(path)
        chunks = chunk_text(content)
        for index, chunk in enumerate(chunks):
            digest = hashlib.sha256(
                f"{path}:{index}:{chunk}".encode("utf-8")
            ).hexdigest()
            metadata.append(
                {
                    "id": digest,
                    "source": str(path.relative_to(DOC_DIR)),
                    "chunk": index,
                    "text": chunk,
                }
            )
            texts.append(chunk)

    if not texts:
        save_db(np.empty((0, 0), dtype=np.float32), [])
        return {"documents": 0, "chunks": 0}

    embeddings = ollama_embed(texts)
    vectors = np.asarray(embeddings, dtype=np.float32)
    norms = np.linalg.norm(vectors, axis=1, keepdims=True)
    vectors = vectors / np.clip(norms, 1e-12, None)
    save_db(vectors, metadata)
    return {
        "documents": len({m["source"] for m in metadata}),
        "chunks": len(metadata),
    }


def retrieve(question: str, top_k: int) -> list[dict[str, Any]]:
    vectors, metadata = load_db()
    if not metadata:
        return []

    query_vector = np.asarray(ollama_embed([question])[0], dtype=np.float32)
    query_vector /= max(float(np.linalg.norm(query_vector)), 1e-12)
    scores = vectors @ query_vector
    indexes = np.argsort(scores)[::-1][:top_k]

    return [
        {
            **metadata[int(index)],
            "score": round(float(scores[int(index)]), 4),
        }
        for index in indexes
    ]


def answer_with_context(question: str, contexts: list[dict[str, Any]]) -> str:
    context = "\n\n".join(
        f"[Fuente: {item['source']} | fragmento {item['chunk']}]\n{item['text']}"
        for item in contexts
    )

    payload = {
        "model": LLM_MODEL,
        "stream": False,
        "messages": [
            {
                "role": "system",
                "content": (
                    "Responde usando exclusivamente el contexto proporcionado. "
                    "Si el contexto no contiene la respuesta, dilo claramente. "
                    "No inventes datos. Cita las fuentes por nombre de fichero."
                ),
            },
            {
                "role": "user",
                "content": f"Contexto:\n{context}\n\nPregunta:\n{question}",
            },
        ],
    }

    response = requests.post(
        f"{OLLAMA_BASE_URL}/api/chat",
        json=payload,
        timeout=180,
    )
    response.raise_for_status()
    return response.json()["message"]["content"]


@app.get("/health")
def health() -> dict[str, Any]:
    response = requests.get(f"{OLLAMA_BASE_URL}/api/tags", timeout=10)
    response.raise_for_status()
    vectors, metadata = load_db()
    return {
        "status": "ok",
        "ollama": "ok",
        "indexed_chunks": len(metadata),
        "vector_shape": list(vectors.shape),
    }


@app.post("/ingest")
def run_ingest() -> dict[str, Any]:
    try:
        return ingest()
    except Exception as exc:
        raise HTTPException(status_code=500, detail=str(exc)) from exc


@app.post("/query")
def query(request: QueryRequest) -> dict[str, Any]:
    try:
        contexts = retrieve(request.question, request.top_k)
        if not contexts:
            return {
                "answer": "No hay documentos indexados. Ejecuta /ingest primero.",
                "sources": [],
            }
        answer = answer_with_context(request.question, contexts)
        return {
            "answer": answer,
            "sources": [
                {
                    "source": c["source"],
                    "chunk": c["chunk"],
                    "score": c["score"],
                }
                for c in contexts
            ],
        }
    except Exception as exc:
        raise HTTPException(status_code=500, detail=str(exc)) from exc
```

## 15.5. `rag/Dockerfile`

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN mkdir -p /data/documents /data/db

EXPOSE 8001

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8001"]
```

## 15.6. `docker-rag.yml`

```yaml
services:
  rag:
    build:
      context: ./rag
      dockerfile: Dockerfile
    image: proyecto-rag:latest
    container_name: rag
    restart: unless-stopped
    ports:
      - "${RAG_HOST_PORT:-8001}:8001"
    environment:
      RAG_OLLAMA_BASE_URL: "${RAG_OLLAMA_BASE_URL:-http://ollama:11434}"
      OLLAMA_EMBED_MODEL: "${OLLAMA_EMBED_MODEL:-nomic-embed-text}"
      RAG_LLM_MODEL: "${RAG_LLM_MODEL:-qwen3:8b}"
      RAG_DOCUMENT_DIR: /data/documents
      RAG_DB_DIR: /data/db
      RAG_TOP_K: "${RAG_TOP_K:-5}"
      RAG_CHUNK_SIZE: "${RAG_CHUNK_SIZE:-1200}"
      RAG_CHUNK_OVERLAP: "${RAG_CHUNK_OVERLAP:-180}"
      TZ: "${TZ:-Europe/Madrid}"
    volumes:
      - ${HOME}/proyecto/rag/documents:/data/documents
      - ${HOME}/proyecto/rag/db:/data/db
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 15.7. Arrancar e indexar

```bash
cd "$HOME/proyecto"
docker compose -f docker-rag.yml up -d --build
curl http://127.0.0.1:8001/health
```

Copiar documentos:

```bash
cp /ruta/de/documento.pdf "$HOME/proyecto/rag/documents/"
cp /ruta/de/manual.md "$HOME/proyecto/rag/documents/"
```

Indexar:

```bash
curl -X POST http://127.0.0.1:8001/ingest
```

Consultar:

```bash
curl -X POST http://127.0.0.1:8001/query \
  -H 'Content-Type: application/json' \
  -d '{
    "question": "¿Cuál es el objetivo principal del proyecto?",
    "top_k": 5
  }'
```

---

# 16. Script maestro de despliegue

## 16.1. `compose-all.sh`

Crear:

```bash
cat > "$HOME/proyecto/compose-all.sh" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

cd "$(dirname "$0")"

docker network inspect red-ia >/dev/null 2>&1 || \
  docker network create --driver bridge red-ia

docker compose \
  -f docker-ollama.yml \
  -f docker-searxng.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-rag.yml \
  up -d --build

echo
printf '%s\n' '=== Estado ==='
docker compose \
  -f docker-ollama.yml \
  -f docker-searxng.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-rag.yml \
  ps
EOF

chmod +x "$HOME/proyecto/compose-all.sh"
```

## 16.2. Arrancar toda la pila

```bash
cd "$HOME/proyecto"
./compose-all.sh
```

## 16.3. Parar toda la pila

Crear `compose-down.sh`:

```bash
cat > "$HOME/proyecto/compose-down.sh" <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

docker compose \
  -f docker-ollama.yml \
  -f docker-searxng.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-rag.yml \
  down
EOF

chmod +x "$HOME/proyecto/compose-down.sh"
```

Ejecutar:

```bash
./compose-down.sh
```

> No usar `docker compose down -v` salvo que se quiera eliminar deliberadamente volúmenes Docker.

---

# 17. Verificación global y pruebas de aceptación

## 17.1. Validar sintaxis Compose

```bash
cd "$HOME/proyecto"

docker compose \
  -f docker-ollama.yml \
  -f docker-searxng.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-rag.yml \
  config
```

El comando debe terminar con código de salida `0`.

## 17.2. Estado de los contenedores

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Deben aparecer:

```text
ollama
openwebui
hermes-agent
opencode
comfyui
yolo
searxng
rag
```

## 17.3. Red

```bash
docker network inspect red-ia
```

## 17.4. Tabla de pruebas

| Prueba | Comando | Resultado esperado |
|---|---|---|
| GPU host | `nvidia-smi` | GPU visible |
| GPU Docker | `docker run --rm --gpus all ... nvidia-smi` | GPU visible |
| Ollama | `curl http://127.0.0.1:11434/api/tags` | JSON |
| Open WebUI | abrir `:3000` | Interfaz web |
| Hermes | `curl :8000/health` | `status=ok` |
| OpenCode | abrir `:8443` | UI web |
| ComfyUI | abrir `:8188` | UI web |
| YOLO | `curl :5000/health` | JSON |
| SearXNG | `curl :8080/search?...&format=json` | JSON |
| RAG | `curl :8001/health` | JSON |

---

# 18. URLs y DNS internos

## 18.1. Desde el host

```text
Ollama       http://127.0.0.1:11434
Open WebUI   http://127.0.0.1:3000
Hermes       http://127.0.0.1:8000
OpenCode     http://127.0.0.1:8443
ComfyUI      http://127.0.0.1:8188
YOLO         http://127.0.0.1:5000
SearXNG      http://127.0.0.1:8080
RAG          http://127.0.0.1:8001
```

## 18.2. Entre contenedores

```text
Ollama       http://ollama:11434
Open WebUI   http://openwebui:8080
Hermes       http://hermes-agent:8000
OpenCode     http://opencode:4096
ComfyUI      http://comfyui:8188
YOLO         http://yolo:5000
SearXNG      http://searxng:8080
RAG          http://rag:8001
```

---

# 19. Guía interna de integración exacta

## 19.1. Ollama → Open WebUI

```text
OLLAMA_BASE_URL=http://ollama:11434
```

Prueba:

```bash
docker exec -it openwebui sh -lc 'python - <<"PY"
import urllib.request
print(urllib.request.urlopen("http://ollama:11434/api/tags", timeout=10).status)
PY'
```

Arquitectura:

```text
Navegador
   |
   v
Open WebUI :8080
   |
   | HTTP
   v
ollama:11434
   |
   v
GPU NVIDIA
```

## 19.2. Open WebUI → SearXNG

```text
ENABLE_WEB_SEARCH=true
WEB_SEARCH_ENGINE=searxng
SEARXNG_QUERY_URL=http://searxng:8080/search
```

Prueba:

```bash
docker exec -it openwebui sh -lc 'python - <<"PY"
import urllib.request
url = "http://searxng:8080/search?q=Docker&format=json"
r = urllib.request.urlopen(url, timeout=20)
print(r.status)
print(r.read(300).decode())
PY'
```

## 19.3. Open WebUI → ComfyUI

```text
COMFYUI_BASE_URL=http://comfyui:8188/
```

Prueba:

```bash
docker exec -it openwebui sh -lc 'python - <<"PY"
import urllib.request
print(urllib.request.urlopen("http://comfyui:8188/", timeout=20).status)
PY'
```

## 19.4. Hermes Agent → Ollama

```yaml
model:
  provider: custom
  base_url: http://ollama:11434/v1
  default: qwen3:8b
```

## 19.5. OpenCode → Ollama

```text
http://ollama:11434/v1
```

## 19.6. OpenCode → SearXNG/MCP

```text
OpenCode
   |
   | stdio MCP
   v
searxng_mcp.py
   |
   | HTTP JSON
   v
http://searxng:8080/search
```

## 19.7. RAG → Ollama

```text
RAG → Ollama /api/embed
RAG → Ollama /api/chat
```

## 19.8. Clientes → YOLO

Host:

```text
POST http://IP_DEL_SERVIDOR:5000/predict
```

Contenedor:

```text
POST http://yolo:5000/predict
```

---

# 20. Operación diaria

## 20.1. Ver todos los contenedores

```bash
docker ps
```

## 20.2. Ver consumo

```bash
docker stats
```

## 20.3. Ver consumo GPU

```bash
watch -n 1 nvidia-smi
```

## 20.4. Logs

```bash
docker logs --tail 200 ollama
docker logs --tail 200 openwebui
docker logs --tail 200 hermes-agent
docker logs --tail 200 opencode
docker logs --tail 200 comfyui
docker logs --tail 200 yolo
docker logs --tail 200 searxng
docker logs --tail 200 rag
```

Seguir un log:

```bash
docker logs -f ollama
```

## 20.5. Entrar a un contenedor

```bash
docker exec -it ollama sh
```

---

# 21. Actualización y mantenimiento

## 21.1. Actualizar imágenes remotas

```bash
docker compose -f docker-ollama.yml pull
docker compose -f docker-ollama.yml up -d
```

Para el resto:

```bash
docker compose -f docker-openwebui.yml pull && docker compose -f docker-openwebui.yml up -d
docker compose -f docker-hermes-agent.yml pull && docker compose -f docker-hermes-agent.yml up -d
docker compose -f docker-searxng.yml pull && docker compose -f docker-searxng.yml up -d
docker compose -f docker-opencode.yml build --pull && docker compose -f docker-opencode.yml up -d
docker compose -f docker-comfyui.yml build --pull && docker compose -f docker-comfyui.yml up -d
docker compose -f docker-yolo.yml build --pull && docker compose -f docker-yolo.yml up -d
docker compose -f docker-rag.yml build --pull && docker compose -f docker-rag.yml up -d
```

## 21.2. Política recomendada

1. Backup.
2. Actualizar un servicio.
3. Revisar logs.
4. Ejecutar prueba funcional.
5. Continuar con el siguiente servicio.

## 21.3. Limpiar imágenes no utilizadas

```bash
docker image prune
```

---

# 22. Backup

## 22.1. Datos a respaldar

```text
$HOME/ollama/
$HOME/openwebui/
$HOME/hermes/
$HOME/opencode/
$HOME/proyecto/comfyui/models/
$HOME/proyecto/comfyui/output/
$HOME/proyecto/comfyui/user/
$HOME/proyecto/yolo/models/
$HOME/proyecto/yolo/datasets/
$HOME/proyecto/yolo/results/
$HOME/proyecto/searxng/
$HOME/proyecto/rag/documents/
$HOME/proyecto/rag/db/
$HOME/proyecto/.env
```

## 22.2. Crear backup

```bash
mkdir -p "$HOME/backups"
BACKUP_DATE=$(date +%Y%m%d-%H%M%S)

tar -czf "$HOME/backups/proyecto-ia-$BACKUP_DATE.tar.gz" \
  "$HOME/proyecto" \
  "$HOME/ollama" \
  "$HOME/openwebui" \
  "$HOME/hermes" \
  "$HOME/opencode"
```

## 22.3. Verificar

```bash
tar -tzf "$HOME/backups/proyecto-ia-AAAAmmdd-HHMMSS.tar.gz" | head
sha256sum "$HOME/backups/proyecto-ia-AAAAmmdd-HHMMSS.tar.gz"
```

El comando real de integridad es:

```bash
sha256sum "$HOME/backups/proyecto-ia-AAAAmmdd-HHMMSS.tar.gz" \
  > "$HOME/backups/proyecto-ia-AAAAmmdd-HHMMSS.sha256"
```

---

# 23. Permisos y propietarios

## 23.1. Revisar

```bash
find "$HOME/proyecto" -maxdepth 2 -type f -printf '%u:%g %p\n' | head -50
```

## 23.2. Corregir propietarios

```bash
sudo chown -R "$USER":"$(id -gn)" "$HOME/proyecto"
```

## 23.3. Hermes

```bash
id -u
id -g
```

Ajustar `.env` según esos valores:

```dotenv
HERMES_UID=1000
HERMES_GID=1000
```

---

# 24. Resolución de errores comunes

## 24.1. GPU no disponible

```bash
nvidia-smi
nvidia-ctk --version
sudo cat /etc/docker/daemon.json
sudo systemctl restart docker
```

Después:

```bash
docker run --rm --gpus all nvidia/cuda:12.8.1-base-ubuntu22.04 nvidia-smi
```

## 24.2. `could not select device driver`

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

## 24.3. `address already in use`

```bash
sudo ss -ltnp | grep ':8080'
```

O revisar todos:

```bash
sudo ss -ltnp | grep -E ':3000|:5000|:8000|:8001|:8080|:8188|:8443|:11434'
```

## 24.4. Open WebUI no encuentra Ollama

```bash
docker exec openwebui sh -lc 'python - <<"PY"
import urllib.request
print(urllib.request.urlopen("http://ollama:11434/api/tags", timeout=10).read().decode())
PY'
```

## 24.5. SearXNG devuelve `403`

Comprobar:

```yaml
search:
  formats:
    - html
    - json
```

Y reiniciar:

```bash
docker restart searxng
```

## 24.6. OpenCode no abre

```bash
docker logs opencode
```

```bash
docker port opencode
```

Debe aparecer:

```text
4096/tcp -> 0.0.0.0:8443
```

## 24.7. Hermes devuelve `401`

```bash
curl http://127.0.0.1:8000/v1/models \
  -H "Authorization: Bearer ${HERMES_API_KEY}"
```

## 24.8. Hermes no encuentra el modelo

```bash
docker exec -it ollama ollama list
docker exec -it ollama ollama pull qwen3:8b
```

## 24.9. RAG no tiene embeddings

```bash
docker exec -it ollama ollama list
docker exec -it ollama ollama pull nomic-embed-text
curl -X POST http://127.0.0.1:8001/ingest
```

## 24.10. YOLO no utiliza GPU

```bash
docker exec -it yolo nvidia-smi
curl http://127.0.0.1:5000/health
```

Revisar:

```yaml
gpus: all
```

Y:

```text
YOLO_DEVICE=0
```

## 24.11. ComfyUI consume toda la VRAM

Evitar cargas simultáneas excesivas. Monitorizar:

```bash
watch -n 1 nvidia-smi
```

---

# 25. Seguridad del despliegue

## 25.1. No exponer directamente a Internet

Especialmente sensibles:

```text
8000  Hermes API
8443  OpenCode
5000  YOLO API
8001  RAG API
11434 Ollama API
```

## 25.2. Firewall

```bash
sudo ufw status verbose
sudo iptables -S DOCKER-USER
```

## 25.3. Secretos

```bash
chmod 600 "$HOME/proyecto/.env"
```

## 25.4. Agentes

Hermes y OpenCode pueden ejecutar acciones. Mantener `bash`, `edit` y accesos a directorios externos en modo de aprobación hasta validar el entorno.

---

# 26. Rendimiento y reparto de GPU

## 26.1. Estrategia inicial

- Ollama como servicio principal de inferencia.
- ComfyUI bajo demanda.
- YOLO para visión puntual.
- Hermes/OpenCode como consumidores de Ollama.
- Open WebUI como interfaz.

## 26.2. Modelo de referencia

`qwen3:8b` aparece actualmente en el catálogo de Ollama con aproximadamente 5,2 GB en Q4_K_M. Es una referencia razonable para pruebas en una GPU de memoria moderada, frente a variantes mayores que exigen más VRAM. 

## 26.3. Monitorización

```bash
watch -n 1 nvidia-smi
```

```bash
nvidia-smi --query-gpu=name,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

---

# 27. Procedimiento completo desde cero

```bash
# Host
sudo apt update
sudo apt full-upgrade -y
nvidia-smi
```

Instalar Docker:

```bash
sudo apt remove -y docker.io docker-compose docker-compose-v2 docker-doc docker-buildx podman-docker containerd runc || true
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

Instalar NVIDIA Container Toolkit:

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends ca-certificates curl gnupg2
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Crear proyecto y red:

```bash
mkdir -p "$HOME/proyecto"
cd "$HOME/proyecto"
docker network create --driver bridge red-ia || true
```

Crear las carpetas y copiar los ficheros del manual.

Revisar `.env`, generar secretos y arrancar:

```bash
./compose-all.sh
```

---

# 28. Lista de comprobación para el administrador

## Host

```text
[ ] Ubuntu Server 24.04/26.04 instalado
[ ] nvidia-smi funciona
[ ] Docker Engine funciona
[ ] docker compose funciona
[ ] NVIDIA Container Toolkit instalado
[ ] docker run --rm --gpus all ... nvidia-smi funciona
```

## Red

```text
[ ] red-ia creada
[ ] 8 contenedores conectados a red-ia
[ ] no hay conflictos de puertos
```

## Servicios

```text
[ ] Ollama :11434
[ ] Open WebUI :3000
[ ] Hermes :8000
[ ] OpenCode :8443
[ ] ComfyUI :8188
[ ] YOLO :5000
[ ] SearXNG :8080
[ ] RAG :8001
```

## Integraciones

```text
[ ] Open WebUI -> Ollama
[ ] Open WebUI -> SearXNG
[ ] Open WebUI -> ComfyUI
[ ] Hermes -> Ollama
[ ] OpenCode -> Ollama
[ ] OpenCode -> SearXNG/MCP
[ ] RAG -> Ollama embeddings
[ ] RAG -> Ollama chat
```

## Persistencia

```text
[ ] $HOME/ollama
[ ] $HOME/openwebui
[ ] $HOME/hermes
[ ] $HOME/opencode
[ ] $HOME/proyecto/comfyui/*
[ ] $HOME/proyecto/yolo/*
[ ] $HOME/proyecto/searxng
[ ] $HOME/proyecto/rag/*
```

---

# 29. Criterios de aceptación

## 29.1. Archivos de configuración sin errores sintácticos

```bash
docker compose \
  -f docker-ollama.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-searxng.yml \
  -f docker-rag.yml \
  config >/tmp/proyecto-ia-compose.rendered.yml
```

## 29.2. GPU disponible

```bash
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.8.1-base-ubuntu22.04 nvidia-smi
docker exec ollamnvidia-smi
docker exec comfyui nvidia-smi
docker exec yolo nvidia-smi
```

## 29.3. Puertos sin colisión

La asignación final es:

```text
3000  Open WebUI
5000  YOLO
8000  Hermes
8001  RAG
8080  SearXNG
8188  ComfyUI
8443  OpenCode
11434 Ollama
```

## 29.4. Manual paso a paso

```text
1. Sistema operativo
2. Driver NVIDIA
3. Docker
4. NVIDIA Container Toolkit
5. Red Docker
6. Ollama
7. SearXNG
8. Open WebUI
9. Hermes
10. OpenCode
11. ComfyUI
12. YOLO
13. RAG
14. Integraciones
15. Verificación
16. Backup y mantenimiento
```

---

# 30. Notas de operación y límites conocidos

1. `latest` simplifica la instalación, pero no proporciona reproducibilidad absoluta. Para producción se recomienda fijar versiones o digests tras validar el stack.
2. ComfyUI, YOLO y los modelos pueden consumir muchos GB de disco y VRAM.
3. Hermes y OpenCode son agentes; deben protegerse con autenticación y permisos restrictivos.
4. SearXNG actúa como metabuscador; el comportamiento final depende de los motores habilitados.
5. El RAG incluido es una implementación de referencia funcional. Para miles o millones de fragmentos se recomienda evolucionarlo a una base vectorial especializada.
6. Ningún servicio debe publicarse a Internet sin una estrategia explícita de autenticación, TLS, firewall y proxy inverso.

---

# 31. Referencias técnicas consultadas

Consultadas el **6 de octubre de 2026**:

- Docker Engine sobre Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- NVIDIA Container Toolkit: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html
- Open WebUI, Docker: https://docs.openwebui.com/getting-started/quick-start/?setup-method=docker-compose
- Open WebUI + SearXNG: https://docs.openwebui.com/features/chat-conversations/web-search/providers/searxng/
- Open WebUI + ComfyUI: https://docs.openwebui.com/features/chat-conversations/image-generation-and-editing/comfyui/
- SearXNG Docker: https://docs.searxng.org/admin/installation-docker
- Hermes Agent API server: https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/api-server.md
- Hermes Agent + Ollama: https://github.com/NousResearch/hermes-agent/blob/main/website/docs/guides/local-ollama-setup.md
- Hermes Agent Docker: https://github.com/NousResearch/hermes-agent/blob/main/docker-compose.yml
- OpenCode servidor web: https://opencode.ai/docs/es/web/
- OpenCode proveedores: https://opencode.ai/docs/providers
- OpenCode MCP: https://opencode.ai/docs/mcp-servers/
- Ultralytics Docker/YOLO: https://docs.ultralytics.com/guides/docker-quickstart
- Ollama Qwen3 8B: https://ollama.com/library/qwen3%3A8b

---

# 32. Resumen final de la arquitectura

```text
                                      INTERNET
                                         |
                                  +------+------+
                                  |   SearXNG   |
                                  |    :8080    |
                                  +------+------+
                                         |
                      +------------------+------------------+
                      |            red-ia (bridge)         |
                      |                                     |
                      |  +------------+      +-----------+  |
                      |  | OpenWebUI  |----->|  Ollama   |  |
                      |  |   :8080    |      |  :11434   |  |
                      |  +-----+------+      +-----+-----+  |
                      |        |                   |        |
                      |        v                   v        |
                      |   +---------+         NVIDIA GPU   |
                      |   | ComfyUI |                      |
                      |   |  :8188  |                      |
                      |   +---------+                      |
                      |                                     |
                      |   +---------+    +-------------+    |
                      |   | YOLO    |    | RAG         |    |
                      |   |  :5000  |    |  :8001      |    |
                      |   +---------+    +------+------+    |
                      |                         |           |
                      |                         v           |
                      |                       Ollama       |
                      |                                     |
                      |   +-------------+   +-------------+ |
                      |   | Hermes Agent|   | OpenCode    | |
                      |   |    :8000    |   |    :4096    | |
                      |   +-------------+   +------+------+-|
                      |                              |       |
                      |                         SearXNG MCP  |
                      +-------------------------------------+

Host ports:
  11434 Ollama
   3000 Open WebUI
   8000 Hermes
   8443 OpenCode
   8188 ComfyUI
   5000 YOLO
   8080 SearXNG
   8001 RAG
```

El resultado es una pila modular en la que Ollama concentra la inferencia LLM, las aplicaciones cliente se comunican mediante DNS interno Docker y las capacidades especializadas —búsqueda, imagen, visión y RAG— permanecen aisladas en sus propios contenedores.
