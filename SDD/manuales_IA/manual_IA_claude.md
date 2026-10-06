---
title: "Manual de instalación, configuración y operación: pila de IA local en Docker sobre Ubuntu Server"
author: "Mnarrieta"
date: "2026-10-06"
category: "Despliegue IA"
tags: [markdown, ia, docker, nvidia, ollama, local, SDD]
version: "1.0"
---

# Manual de instalación, configuración y operación: pila de IA local en Docker sobre Ubuntu Server

Este manual se ha generado siguiendo la especificación SDD (*Spec Driven Development*) del proyecto. Está escrito paso a paso para un administrador de sistemas novato: copia y pega los comandos en el orden indicado y no te saltes las verificaciones.

## Índice

0. [Decisiones de diseño y correcciones a la especificación](#0-decisiones-de-diseño-y-correcciones-a-la-especificación)
1. [Prerrequisitos e instalación base](#1-prerrequisitos-e-instalación-base)
2. [Estructura del proyecto](#2-estructura-del-proyecto)
3. [Ficheros de configuración (`docker-<servicio>.yml`)](#3-ficheros-de-configuración-docker-servicioyml)
4. [Fichero de entorno `.env`](#4-fichero-de-entorno-env)
5. [Despliegue y verificación](#5-despliegue-y-verificación)
6. [Mantenimiento y actualización](#6-mantenimiento-y-actualización)
7. [Guía interna de integración entre servicios](#7-guía-interna-de-integración-entre-servicios)
8. [Checklist final de criterios de aceptación](#8-checklist-final-de-criterios-de-aceptación)

---

## 0. Decisiones de diseño y correcciones a la especificación

Al analizar la especificación aparecen incoherencias que habrían impedido cumplir los criterios de aceptación. Se han resuelto así:

| # | Problema detectado en la especificación | Decisión tomada en este manual |
|---|---|---|
| 1 | **RAG usa el puerto 11434**, el mismo que Ollama (colisión, incumple el criterio 3). | RAG escucha en el puerto **8100** (interno y externo). |
| 2 | La URL interna de Open WebUI aparece como `http://openwebui:3000`, pero 3000 es el puerto del *host*; el puerto interno es 8080. | URL interna correcta: `http://openwebui:8080`. |
| 3 | OpenCode: la URL interna indica `:8443` (puerto del host); el interno es 8080. | URL interna correcta: `http://opencode:8080`. |
| 4 | Hermes Agent: el contenedor se llama `hermes-agent` pero la URL usa `hermesagent`. | Se mantiene el nombre `hermes-agent` y se añade el alias DNS `hermesagent` para que ambas URL funcionen. |
| 5 | La sección 5 pide `docker-<servicio>.yml` y los criterios piden `docker_<servicio>.yml`. | Se usa **`docker-<servicio>.yml`** (guion), que es lo que pide la sección de instrucciones. |
| 6 | La red del apartado 3.1 pide un DNS local por nombre de contenedor. | Docker ya lo proporciona en redes *bridge* definidas por el usuario: no hay que instalar nada. |
| 7 | Los criterios piden que *todos* los contenedores usen GPU, pero SearXNG, Hermes, OpenCode y RAG no hacen cálculo de IA propio. | Reservan GPU los que sí la usan: **Ollama, Open WebUI (imagen `cuda`), ComfyUI y YOLO**. Los demás consumen la GPU **indirectamente** a través de Ollama. |
| 8 | Los volúmenes se piden en `$HOME/<servicio>`, pero el proyecto vive en `$HOME/proyecto`. | Los datos persistentes van en `$HOME/<servicio>` (como pide la spec) y los ficheros de configuración en `$HOME/proyecto`. La variable `DATA_ROOT` del `.env` controla la ruta base. |
| 9 | OpenCode se publica en `8443`, puerto típico de HTTPS, pero el servicio habla HTTP. | Se accede por `http://IP:8443`. Se protege con usuario y contraseña. Para HTTPS real habría que añadir un proxy inverso (fuera del alcance). |

> **Aviso sobre la GPU compartida:** Ollama, ComfyUI y YOLO comparten una única GPU. Una 3050 o una 4060 tienen unos 6-8 GB de VRAM, así que **no uses los tres a la vez con modelos grandes**. En la sección 4 hay variables para limitar cuánto tiempo mantiene Ollama los modelos en memoria.

> **Aviso sobre Hermes Agent:** es un proyecto reciente y cambia rápido. La imagen oficial es `nousresearch/hermes-agent` y se ejecuta con `gateway run`. Si tras el despliegue el puerto 8000 no responde por HTTP, es normal que el *gateway* no exponga web en ese puerto por defecto: revisa los logs (sección 5) y la documentación oficial del proyecto.

---

## 1. Prerrequisitos e instalación base

Todos los comandos se ejecutan en el servidor Ubuntu Server 24.04 (o 26.04) con un usuario con permisos `sudo`.

### 1.1. Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg git ubuntu-drivers-common
sudo reboot
```

Espera a que la máquina arranque y vuelve a conectarte por SSH.

### 1.2. Instalar el driver de NVIDIA (CUDA)

El driver del *host* es lo único que se necesita para CUDA: las librerías CUDA van dentro de las imágenes Docker. No hace falta instalar el *CUDA Toolkit* completo en el servidor.

```bash
# Ver qué driver recomienda Ubuntu para tu tarjeta
ubuntu-drivers devices

# Instalar el recomendado (alternativa para servidores: sudo ubuntu-drivers install --gpgpu)
sudo ubuntu-drivers install

sudo reboot
```

Verifica tras el reinicio:

```bash
nvidia-smi
```

Debe aparecer una tabla con tu GPU (por ejemplo RTX 3050 o 4060), la versión del driver y la versión de CUDA. Si da error, **no continúes**: revisa la sección 6.4.

### 1.3. Instalar Docker y Docker Compose (repositorio oficial apt)

```bash
# Clave GPG y repositorio oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Permitir usar docker sin sudo (cierra y vuelve a abrir la sesión SSH después)
sudo usermod -aG docker $USER
```

Cierra la sesión SSH, vuelve a entrar y comprueba:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

> **Si usas Ubuntu 26.04** y `apt update` da error con el repositorio de Docker (porque aún no se han publicado paquetes para esa versión), sustituye `$(. /etc/os-release && echo "${VERSION_CODENAME}")` por `noble` en la línea del repositorio.

### 1.4. Instalar NVIDIA Container Toolkit (GPU dentro de Docker)

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Registrar el runtime NVIDIA en Docker y reiniciar
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 1.5. Prueba de GPU dentro de un contenedor

```bash
docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubuntu24.04 nvidia-smi
```

Debe mostrar la **misma tabla** que `nvidia-smi` en el servidor. Si ves esa tabla, la base está lista.

### 1.6. Nota sobre el cortafuegos

Docker modifica `iptables` directamente y **los puertos publicados saltan las reglas de `ufw`**. Si el servidor está en una red no confiable, restringe el acceso a nivel de red (router/firewall perimetral) o publica los puertos sólo en `127.0.0.1` (cambia `"8080:8080"` por `"127.0.0.1:8080:8080"` en los ficheros YAML).

---

## 2. Estructura del proyecto

### 2.1. Árbol de directorios

```text
$HOME/
├── proyecto/                         # Ficheros de configuración (este es el "proyecto")
│   ├── .env                          # Variables de entorno de todos los servicios
│   ├── docker-ollama.yml
│   ├── docker-openwebui.yml
│   ├── docker-hermes-agent.yml
│   ├── docker-opencode.yml
│   ├── docker-comfyui.yml
│   ├── docker-yolo.yml
│   ├── docker-searxng.yml
│   ├── docker-rag.yml
│   ├── build/                        # Dockerfiles de los servicios sin imagen oficial
│   │   ├── opencode/Dockerfile
│   │   ├── comfyui/Dockerfile
│   │   ├── yolo/
│   │   │   ├── Dockerfile
│   │   │   └── app.py
│   │   └── rag/
│   │       ├── Dockerfile
│   │       ├── requirements.txt
│   │       └── app.py
│   └── scripts/
│       ├── up.sh                     # Arranque ordenado
│       ├── down.sh
│       └── backup.sh
│
├── ollama/                           # Volumen ollama_data   -> modelos LLM
├── openwebui/                        # Volumen openwebui_data -> usuarios, chats, prompts
├── hermes/                           # Configuración de los agentes
├── opencode/
│   ├── projects/                     # Código de los proyectos
│   ├── config/                       # opencode.json
│   └── data/                         # Estado interno de OpenCode
├── comfyui/
│   ├── models/                       # checkpoints, vae, loras, ...
│   ├── input/
│   ├── output/                       # Imágenes y vídeos generados
│   └── user/                         # Workflows y ajustes
├── yolo/                             # Dataset, imágenes analizadas y pesos
├── searxng/                          # settings.yml y caché
├── rag/
│   ├── docs/                         # Documentos a indexar (.txt, .md)
│   └── chroma/                       # Base de datos vectorial
└── backups/                          # Copias de seguridad
```

### 2.2. Crear toda la estructura

```bash
# Proyecto y subcarpetas de construcción
mkdir -p $HOME/proyecto/{build/{opencode,comfyui,yolo,rag},scripts}

# Volúmenes de datos
mkdir -p $HOME/{ollama,openwebui,hermes,yolo,searxng,backups}
mkdir -p $HOME/opencode/{projects,config,data}
mkdir -p $HOME/rag/{docs,chroma}
mkdir -p $HOME/comfyui/{input,output,user}
mkdir -p $HOME/comfyui/models/{checkpoints,vae,loras,clip,unet,controlnet,embeddings,upscale_models,configs,diffusion_models,text_encoders}
```

### 2.3. Crear la red Docker `red-ia`

La red se crea **una sola vez** y todos los ficheros YAML la reutilizan (`external: true`). Al ser una red *bridge* definida por el usuario, Docker incluye un DNS interno: cada contenedor se resuelve por su nombre (`ollama`, `searxng`...).

```bash
docker network create --driver bridge red-ia
docker network ls | grep red-ia
```

### 2.4. Tabla de puertos (sin colisiones)

| Servicio | Contenedor | Puerto interno | Puerto host | URL interna (red-ia) |
|---|---|---|---|---|
| Ollama | `ollama` | 11434 | 11434 | `http://ollama:11434` |
| Open WebUI | `openwebui` | 8080 | 3000 | `http://openwebui:8080` |
| Hermes Agent | `hermes-agent` | 8000 | 8000 | `http://hermes-agent:8000` (alias `hermesagent`) |
| OpenCode | `opencode` | 8080 | 8443 | `http://opencode:8080` |
| ComfyUI | `comfyui` | 8188 | 8188 | `http://comfyui:8188` |
| YOLO | `yolo` | 5000 | 5000 | `http://yolo:5000` |
| SearXNG | `searxng` | 8080 | 8080 | `http://searxng:8080` |
| RAG | `rag` | 8100 | 8100 | `http://rag:8100` |

Los puertos **del host** (columna 5) son todos distintos: 11434, 3000, 8000, 8443, 8188, 5000, 8080, 8100. Que varios servicios usen 8080 *internamente* no es problema, porque cada contenedor tiene su propia pila de red.

---

## 3. Ficheros de configuración (`docker-<servicio>.yml`)

Todos los ficheros se guardan en `$HOME/proyecto/`. Cada uno define **un único servicio** y se conecta a la red externa `red-ia`.

```bash
cd $HOME/proyecto
```

### 3.1. `docker-ollama.yml`

```yaml
services:
  ollama:
    image: ollama/ollama:${OLLAMA_VERSION:-latest}
    container_name: ollama
    restart: unless-stopped
    ports:
      - "${OLLAMA_PORT:-11434}:11434"
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_KEEP_ALIVE=${OLLAMA_KEEP_ALIVE:-5m}
      - OLLAMA_MAX_LOADED_MODELS=${OLLAMA_MAX_LOADED_MODELS:-1}
      - OLLAMA_NUM_PARALLEL=${OLLAMA_NUM_PARALLEL:-1}
      - OLLAMA_CONTEXT_LENGTH=${OLLAMA_CONTEXT_LENGTH:-8192}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ${DATA_ROOT}/ollama:/root/.ollama      # ollama_data
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
    name: red-ia
```

### 3.2. `docker-openwebui.yml`

```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:${OPENWEBUI_TAG:-cuda}
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "${OPENWEBUI_PORT:-3000}:8080"
    environment:
      # --- Ollama ---
      - OLLAMA_BASE_URL=http://ollama:11434
      # --- Seguridad ---
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      - ENABLE_SIGNUP=${OPENWEBUI_ENABLE_SIGNUP:-true}
      # --- Búsqueda web con SearXNG ---
      - ENABLE_WEB_SEARCH=True
      - WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
      - ENABLE_RAG_WEB_SEARCH=True
      - RAG_WEB_SEARCH_ENGINE=searxng
      # --- Generación de imágenes con ComfyUI ---
      - ENABLE_IMAGE_GENERATION=True
      - IMAGE_GENERATION_ENGINE=comfyui
      - COMFYUI_BASE_URL=http://comfyui:8188
      # --- RAG interno de Open WebUI con embeddings de Ollama ---
      - RAG_EMBEDDING_ENGINE=ollama
      - RAG_EMBEDDING_MODEL=${EMBED_MODEL:-nomic-embed-text}
      - RAG_OLLAMA_BASE_URL=http://ollama:11434
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ${DATA_ROOT}/openwebui:/app/backend/data   # openwebui_data
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
    name: red-ia
```

### 3.3. `docker-hermes-agent.yml`

```yaml
services:
  hermes-agent:
    image: nousresearch/hermes-agent:${HERMES_TAG:-latest}
    container_name: hermes-agent
    restart: unless-stopped
    command: ["gateway", "run"]
    ports:
      - "${HERMES_PORT:-8000}:8000"
    environment:
      # Ollama expone una API compatible con OpenAI en /v1
      - OPENAI_BASE_URL=http://ollama:11434/v1
      - OPENAI_API_KEY=ollama
      - HERMES_MODEL=${HERMES_MODEL:-qwen2.5:7b}
      # Servicios auxiliares (referencia para las herramientas del agente)
      - SEARXNG_URL=http://searxng:8080
      - COMFYUI_URL=http://comfyui:8188
    volumes:
      - ${DATA_ROOT}/hermes:/opt/data             # Configuración de los agentes
    networks:
      red-ia:
        aliases:
          - hermesagent                           # Para que http://hermesagent:8000 también resuelva

networks:
  red-ia:
    external: true
    name: red-ia
```

> Hermes Agent **no** reserva GPU: delega la inferencia en Ollama, que sí la usa.

### 3.4. `docker-opencode.yml`

`build/opencode/Dockerfile`:

```dockerfile
FROM node:22-slim

RUN apt-get update \
 && apt-get install -y --no-install-recommends git curl ca-certificates ripgrep \
 && rm -rf /var/lib/apt/lists/*

RUN npm install -g opencode-ai

WORKDIR /workspace
EXPOSE 8080

CMD ["opencode", "web", "--hostname", "0.0.0.0", "--port", "8080"]
```

`docker-opencode.yml`:

```yaml
services:
  opencode:
    build:
      context: ./build/opencode
    image: proyecto/opencode:local
    container_name: opencode
    restart: unless-stopped
    ports:
      - "${OPENCODE_PORT:-8443}:8080"
    environment:
      - OPENCODE_SERVER_USERNAME=${OPENCODE_USER:-opencode}
      - OPENCODE_SERVER_PASSWORD=${OPENCODE_PASSWORD}
    volumes:
      - ${DATA_ROOT}/opencode/projects:/workspace
      - ${DATA_ROOT}/opencode/config:/root/.config/opencode
      - ${DATA_ROOT}/opencode/data:/root/.local/share/opencode
    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: red-ia
```

### 3.5. `docker-comfyui.yml`

`build/comfyui/Dockerfile`:

```dockerfile
FROM pytorch/pytorch:2.7.1-cuda12.6-cudnn9-runtime

RUN apt-get update \
 && apt-get install -y --no-install-recommends git libgl1 libglib2.0-0 \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /opt
RUN git clone https://github.com/comfyanonymous/ComfyUI.git
WORKDIR /opt/ComfyUI
RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 8188
CMD ["python", "main.py", "--listen", "0.0.0.0", "--port", "8188"]
```

`docker-comfyui.yml`:

```yaml
services:
  comfyui:
    build:
      context: ./build/comfyui
    image: proyecto/comfyui:local
    container_name: comfyui
    restart: unless-stopped
    ports:
      - "${COMFYUI_PORT:-8188}:8188"
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ${DATA_ROOT}/comfyui/models:/opt/ComfyUI/models
      - ${DATA_ROOT}/comfyui/input:/opt/ComfyUI/input
      - ${DATA_ROOT}/comfyui/output:/opt/ComfyUI/output
      - ${DATA_ROOT}/comfyui/user:/opt/ComfyUI/user
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
    name: red-ia
```

### 3.6. `docker-yolo.yml`

`build/yolo/app.py` (API REST mínima de detección):

```python
import io
import os

from fastapi import FastAPI, File, UploadFile
from PIL import Image
from ultralytics import YOLO

MODEL_NAME = os.getenv("YOLO_MODEL", "yolo11n.pt")
model = YOLO(MODEL_NAME)  # Se descarga la primera vez en /data

app = FastAPI(title="YOLO API")


@app.get("/health")
def health():
    return {"status": "ok", "model": MODEL_NAME}


@app.post("/detect")
async def detect(file: UploadFile = File(...), conf: float = 0.25):
    img = Image.open(io.BytesIO(await file.read())).convert("RGB")
    result = model.predict(img, conf=conf, device=0, verbose=False)[0]
    detections = [
        {
            "class": model.names[int(box.cls)],
            "confidence": round(float(box.conf), 4),
            "box_xyxy": [round(float(v), 1) for v in box.xyxy[0]],
        }
        for box in result.boxes
    ]
    return {"count": len(detections), "detections": detections}
```

`build/yolo/Dockerfile`:

```dockerfile
FROM ultralytics/ultralytics:latest

RUN pip install --no-cache-dir fastapi "uvicorn[standard]" python-multipart

COPY app.py /app/app.py
WORKDIR /data
EXPOSE 5000

CMD ["uvicorn", "app:app", "--app-dir", "/app", "--host", "0.0.0.0", "--port", "5000"]
```

`docker-yolo.yml`:

```yaml
services:
  yolo:
    build:
      context: ./build/yolo
    image: proyecto/yolo:local
    container_name: yolo
    restart: unless-stopped
    ports:
      - "${YOLO_PORT:-5000}:5000"
    environment:
      - YOLO_MODEL=${YOLO_MODEL:-yolo11n.pt}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ${DATA_ROOT}/yolo:/data                   # Dataset, imágenes y pesos
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
    name: red-ia
```

### 3.7. `docker-searxng.yml`

Primero crea el fichero de ajustes `$HOME/searxng/settings.yml`. Es **obligatorio** habilitar el formato `json`: sin él, Open WebUI y OpenCode no pueden usar el buscador.

```bash
cat > $HOME/searxng/settings.yml << 'EOF'
use_default_settings: true

server:
  limiter: false
  image_proxy: true

search:
  safe_search: 0
  formats:
    - html
    - json
EOF
```

`docker-searxng.yml`:

```yaml
services:
  searxng:
    image: searxng/searxng:${SEARXNG_TAG:-latest}
    container_name: searxng
    restart: unless-stopped
    ports:
      - "${SEARXNG_PORT:-8080}:8080"
    environment:
      - SEARXNG_SECRET=${SEARXNG_SECRET}
      - SEARXNG_BASE_URL=http://${HOST_IP:-localhost}:${SEARXNG_PORT:-8080}/
    volumes:
      - ${DATA_ROOT}/searxng:/etc/searxng
    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: red-ia
```

### 3.8. `docker-rag.yml`

Servicio RAG propio: indexa los `.txt` y `.md` de `$HOME/rag/docs`, genera *embeddings* con Ollama (GPU), los guarda en ChromaDB y responde preguntas con el LLM usando como contexto los fragmentos más parecidos.

`build/rag/requirements.txt`:

```text
fastapi
uvicorn[standard]
chromadb
httpx
```

`build/rag/app.py`:

```python
import glob
import os

import chromadb
import httpx
from fastapi import FastAPI
from pydantic import BaseModel

OLLAMA = os.getenv("OLLAMA_BASE_URL", "http://ollama:11434")
EMBED_MODEL = os.getenv("RAG_EMBED_MODEL", "nomic-embed-text")
LLM_MODEL = os.getenv("RAG_LLM_MODEL", "llama3.2:3b")
DOCS_DIR = os.getenv("RAG_DOCS_DIR", "/data/docs")

client = chromadb.PersistentClient(path="/data/chroma")
collection = client.get_or_create_collection("docs")
app = FastAPI(title="RAG")


def embed(texts):
    r = httpx.post(
        f"{OLLAMA}/api/embed",
        json={"model": EMBED_MODEL, "input": texts},
        timeout=300,
    )
    r.raise_for_status()
    return r.json()["embeddings"]


def split(text, size=1000, overlap=200):
    start = 0
    while start < len(text):
        yield text[start:start + size]
        start += size - overlap


@app.get("/health")
def health():
    return {"status": "ok", "chunks": collection.count()}


@app.post("/ingest")
def ingest():
    total = 0
    for path in glob.glob(f"{DOCS_DIR}/**/*", recursive=True):
        if not path.lower().endswith((".txt", ".md")):
            continue
        with open(path, encoding="utf-8", errors="ignore") as f:
            parts = list(split(f.read()))
        if not parts:
            continue
        collection.upsert(
            ids=[f"{path}:{i}" for i in range(len(parts))],
            documents=parts,
            embeddings=embed(parts),
            metadatas=[{"source": path}] * len(parts),
        )
        total += len(parts)
    return {"ingested_chunks": total}


class Query(BaseModel):
    question: str
    k: int = 4


@app.post("/query")
def query(q: Query):
    found = collection.query(query_embeddings=embed([q.question]), n_results=q.k)
    context = "\n\n".join(found["documents"][0])
    prompt = (
        "Responde usando solo el siguiente contexto. Si no está en el contexto, "
        f"di que no lo sabes.\n\nContexto:\n{context}\n\nPregunta: {q.question}"
    )
    r = httpx.post(
        f"{OLLAMA}/api/generate",
        json={"model": LLM_MODEL, "prompt": prompt, "stream": False},
        timeout=600,
    )
    r.raise_for_status()
    return {
        "answer": r.json()["response"],
        "sources": sorted({m["source"] for m in found["metadatas"][0]}),
    }
```

`build/rag/Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .

EXPOSE 8100
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8100"]
```

`docker-rag.yml`:

```yaml
services:
  rag:
    build:
      context: ./build/rag
    image: proyecto/rag:local
    container_name: rag
    restart: unless-stopped
    ports:
      - "${RAG_PORT:-8100}:8100"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - RAG_EMBED_MODEL=${EMBED_MODEL:-nomic-embed-text}
      - RAG_LLM_MODEL=${RAG_LLM_MODEL:-llama3.2:3b}
      - RAG_DOCS_DIR=/data/docs
    volumes:
      - ${DATA_ROOT}/rag:/data                    # docs/ y chroma/
    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: red-ia
```

### 3.9. Validar la sintaxis de los YAML

Antes de arrancar nada, comprueba que todos son sintácticamente correctos (con el `.env` ya creado, sección 4):

```bash
cd $HOME/proyecto
for f in docker-*.yml; do
  echo "== $f"
  docker compose -f "$f" config --quiet && echo "OK"
done
```

Cada fichero debe imprimir `OK`.

---

## 4. Fichero de entorno `.env`

Crea `$HOME/proyecto/.env`. Los secretos se generan con `openssl` para no dejar contraseñas por defecto.

```bash
cd $HOME/proyecto

cat > .env << 'EOF'
# =====================================================================
#  PROYECTO IA LOCAL - Variables de entorno
# =====================================================================

# --- Proyecto Compose -------------------------------------------------
COMPOSE_PROJECT_NAME=proyecto-ia
# Permite usar "docker compose up -d" sin -f (los separa con ":")
COMPOSE_FILE=docker-ollama.yml:docker-searxng.yml:docker-comfyui.yml:docker-yolo.yml:docker-openwebui.yml:docker-rag.yml:docker-hermes-agent.yml:docker-opencode.yml

# --- Rutas y red ------------------------------------------------------
# Se sustituye por tu $HOME real en el paso siguiente
DATA_ROOT=/home/usuario
# IP o nombre del servidor tal y como lo verás desde tu navegador
HOST_IP=localhost

# --- Puertos del host (todos distintos) ------------------------------
OLLAMA_PORT=11434
OPENWEBUI_PORT=3000
HERMES_PORT=8000
OPENCODE_PORT=8443
COMFYUI_PORT=8188
YOLO_PORT=5000
SEARXNG_PORT=8080
RAG_PORT=8100

# --- Versiones de imagen ---------------------------------------------
OLLAMA_VERSION=latest
OPENWEBUI_TAG=cuda
HERMES_TAG=latest
SEARXNG_TAG=latest

# --- Ollama -----------------------------------------------------------
OLLAMA_KEEP_ALIVE=5m
OLLAMA_MAX_LOADED_MODELS=1
OLLAMA_NUM_PARALLEL=1
OLLAMA_CONTEXT_LENGTH=8192

# --- Modelos ----------------------------------------------------------
EMBED_MODEL=nomic-embed-text
RAG_LLM_MODEL=llama3.2:3b
HERMES_MODEL=qwen2.5:7b
YOLO_MODEL=yolo11n.pt

# --- Open WebUI -------------------------------------------------------
WEBUI_SECRET_KEY=CAMBIAME
OPENWEBUI_ENABLE_SIGNUP=true

# --- OpenCode ---------------------------------------------------------
OPENCODE_USER=opencode
OPENCODE_PASSWORD=CAMBIAME

# --- SearXNG ----------------------------------------------------------
SEARXNG_SECRET=CAMBIAME
EOF

# 1) Poner tu $HOME real
sed -i "s|^DATA_ROOT=.*|DATA_ROOT=$HOME|" .env

# 2) Generar secretos aleatorios
sed -i "s|^WEBUI_SECRET_KEY=.*|WEBUI_SECRET_KEY=$(openssl rand -hex 32)|" .env
sed -i "s|^SEARXNG_SECRET=.*|SEARXNG_SECRET=$(openssl rand -hex 32)|" .env
sed -i "s|^OPENCODE_PASSWORD=.*|OPENCODE_PASSWORD=$(openssl rand -base64 18 | tr -d '/+=')|" .env

# 3) (Opcional) Poner la IP del servidor para SearXNG
sed -i "s|^HOST_IP=.*|HOST_IP=$(hostname -I | awk '{print $1}')|" .env

# 4) Proteger el fichero
chmod 600 .env
```

Para consultar la contraseña generada de OpenCode:

```bash
grep OPENCODE_PASSWORD $HOME/proyecto/.env
```

> **Importante:** el `.env` contiene secretos. **No lo subas a Git ni lo compartas.**

---

## 5. Despliegue y verificación

### 5.1. Scripts de arranque, parada y copia

```bash
cd $HOME/proyecto

cat > scripts/up.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."

# Orden: primero los servicios base, luego los que dependen de ellos
for svc in ollama searxng comfyui yolo openwebui rag hermes-agent opencode; do
  echo ">>> Arrancando $svc ..."
  docker compose -f "docker-$svc.yml" up -d --build
done
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
EOF

cat > scripts/down.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")/.."
for svc in opencode hermes-agent rag openwebui yolo comfyui searxng ollama; do
  docker compose -f "docker-$svc.yml" down
done
EOF

chmod +x scripts/*.sh
```

### 5.2. Primer arranque

La primera vez se construyen imágenes (ComfyUI, YOLO, RAG y OpenCode) y se descargan varios GB: **tarda entre 10 y 30 minutos** según tu conexión.

```bash
cd $HOME/proyecto
./scripts/up.sh
```

> Alternativa: con la variable `COMPOSE_FILE` del `.env`, basta `docker compose up -d --build` para levantar todo de golpe.

### 5.3. Descargar los modelos de Ollama

Elige modelos que quepan en tu VRAM (3050: 6 u 8 GB; 4060: 8 GB). Estos son seguros para 8 GB:

```bash
docker exec -it ollama ollama pull llama3.2:3b          # chat ligero (~2 GB)
docker exec -it ollama ollama pull qwen2.5:7b           # agente / chat general (~4,7 GB)
docker exec -it ollama ollama pull qwen2.5-coder:7b     # código para OpenCode (~4,7 GB)
docker exec -it ollama ollama pull nomic-embed-text     # embeddings para RAG (~270 MB)

docker exec -it ollama ollama list
```

> Con 6 GB de VRAM quédate con modelos de 3B-4B (`llama3.2:3b`, `qwen2.5:3b`) y cambia `HERMES_MODEL` en el `.env`.

### 5.4. Descargar un modelo para ComfyUI

ComfyUI arranca sin modelos. Descarga al menos un *checkpoint* (por ejemplo, desde Hugging Face o Civitai) y déjalo en `$HOME/comfyui/models/checkpoints/`. Para 8 GB de VRAM, un modelo tipo SD 1.5 o SDXL optimizado va bien.

```bash
cd $HOME/comfyui/models/checkpoints
# Ejemplo: wget -O mi_modelo.safetensors "<URL_DEL_MODELO>"
```

Recarga ComfyUI desde su interfaz (botón *Refresh*) para que aparezca.

### 5.5. Comprobar el estado de los contenedores

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Deben aparecer los 8 contenedores en estado `Up`: `ollama`, `searxng`, `comfyui`, `yolo`, `openwebui`, `rag`, `hermes-agent` y `opencode`.

### 5.6. Comprobar los logs

```bash
docker logs --tail 50 ollama
docker logs --tail 50 openwebui
docker logs --tail 50 comfyui
docker logs --tail 50 yolo
docker logs --tail 50 searxng
docker logs --tail 50 rag
docker logs --tail 50 hermes-agent
docker logs --tail 50 opencode

# Seguir un log en directo (Ctrl+C para salir)
docker logs -f ollama
```

Busca líneas con `error` o `CUDA`. En Ollama debe aparecer algo como *"inference compute ... library=CUDA"*.

### 5.7. Acceso a las URL de cada servicio

Sustituye `IP_SERVIDOR` por la IP de tu servidor (`hostname -I`).

| Servicio | URL desde el navegador | Qué deberías ver |
|---|---|---|
| Ollama | `http://IP_SERVIDOR:11434` | Texto "Ollama is running" |
| Open WebUI | `http://IP_SERVIDOR:3000` | Pantalla de registro (el primer usuario creado es **administrador**) |
| Hermes Agent | `http://IP_SERVIDOR:8000` | Depende de la versión (ver aviso de la sección 0) |
| OpenCode | `http://IP_SERVIDOR:8443` | Petición de usuario/contraseña y luego la interfaz web |
| ComfyUI | `http://IP_SERVIDOR:8188` | Editor de nodos |
| YOLO | `http://IP_SERVIDOR:5000/docs` | Documentación interactiva de la API |
| SearXNG | `http://IP_SERVIDOR:8080` | Buscador |
| RAG | `http://IP_SERVIDOR:8100/docs` | Documentación interactiva de la API |

Verificación rápida por consola:

```bash
curl -s http://localhost:11434 ; echo
curl -s http://localhost:5000/health ; echo
curl -s http://localhost:8100/health ; echo
curl -s "http://localhost:8080/search?q=docker&format=json" | head -c 300 ; echo
curl -s -o /dev/null -w "ComfyUI: %{http_code}\n" http://localhost:8188
curl -s -o /dev/null -w "OpenWebUI: %{http_code}\n" http://localhost:3000
```

### 5.8. Verificar el uso de la GPU

```bash
# Estado general de la GPU en el servidor
nvidia-smi

# Refresco continuo cada 2 segundos (Ctrl+C para salir)
watch -n 2 nvidia-smi

# GPU visible desde dentro de cada contenedor con acceso a ella
docker exec ollama nvidia-smi
docker exec comfyui nvidia-smi
docker exec yolo nvidia-smi
docker exec openwebui nvidia-smi
```

**Prueba de carga real:** lanza una pregunta a Ollama y mira `nvidia-smi` mientras responde.

```bash
docker exec -it ollama ollama run llama3.2:3b "Explica qué es Docker en una frase"
# En otra terminal:
nvidia-smi
```

Debes ver el proceso `ollama` consumiendo memoria de la GPU y el uso (`GPU-Util`) subiendo durante la respuesta. Para confirmar que el modelo está en GPU y no en CPU:

```bash
docker exec ollama ollama ps
```

La columna `PROCESSOR` debe indicar `100% GPU`. Si dice `CPU` o un porcentaje mixto, el modelo es demasiado grande para tu VRAM.

Prueba de YOLO con GPU:

```bash
# Descargar una imagen de prueba y detectar objetos
curl -sL -o /tmp/bus.jpg https://ultralytics.com/images/bus.jpg
curl -s -X POST -F "file=@/tmp/bus.jpg" http://localhost:5000/detect
```

### 5.9. Parar y reiniciar

```bash
cd $HOME/proyecto
./scripts/down.sh                                  # parar todo
docker compose -f docker-ollama.yml restart        # reiniciar un servicio
docker compose -f docker-comfyui.yml down          # parar solo uno (libera VRAM)
```

> **Truco de VRAM:** si necesitas toda la GPU para ComfyUI, para YOLO y deja que Ollama descargue sus modelos (`docker exec ollama ollama stop llama3.2:3b`).

---

## 6. Mantenimiento y actualización

### 6.1. Copias de seguridad

Qué copiar: **configuración y datos de usuario**. Los modelos (`ollama/` y `comfyui/models/`) pesan decenas de GB y se pueden volver a descargar, así que se excluyen por defecto.

```bash
cd $HOME/proyecto

cat > scripts/backup.sh << 'EOF'
#!/usr/bin/env bash
set -euo pipefail

DEST="$HOME/backups"
STAMP="$(date +%Y%m%d_%H%M%S)"
FILE="$DEST/proyecto-ia_$STAMP.tar.gz"
mkdir -p "$DEST"

# Parada breve de los servicios con base de datos para una copia consistente
docker stop openwebui rag >/dev/null

tar -czf "$FILE" \
  --exclude="$HOME/comfyui/models" \
  --exclude="$HOME/comfyui/output" \
  -C "$HOME" \
  proyecto openwebui hermes opencode yolo searxng rag comfyui/user

docker start openwebui rag >/dev/null

# Conservar solo las 7 últimas copias
ls -1t "$DEST"/proyecto-ia_*.tar.gz | tail -n +8 | xargs -r rm --
echo "Copia creada: $FILE"
EOF

chmod +x scripts/backup.sh
./scripts/backup.sh
```

Programar una copia diaria a las 03:00:

```bash
crontab -e
# Añadir esta línea:
0 3 * * * /home/TU_USUARIO/proyecto/scripts/backup.sh >> /home/TU_USUARIO/backups/backup.log 2>&1
```

Copia aparte de las imágenes generadas (opcional) y de los modelos de Ollama:

```bash
tar -czf $HOME/backups/comfyui-output_$(date +%F).tar.gz -C $HOME comfyui/output
tar -czf $HOME/backups/ollama-models_$(date +%F).tar.gz -C $HOME ollama
```

**Restaurar una copia:**

```bash
cd $HOME/proyecto && ./scripts/down.sh
tar -xzf $HOME/backups/proyecto-ia_FECHA.tar.gz -C $HOME
./scripts/up.sh
```

### 6.2. Actualización de los servicios

**Antes de actualizar, haz siempre una copia** (`./scripts/backup.sh`).

Servicios con imagen publicada (Ollama, Open WebUI, SearXNG, Hermes Agent):

```bash
cd $HOME/proyecto
for svc in ollama openwebui searxng hermes-agent; do
  docker compose -f docker-$svc.yml pull
  docker compose -f docker-$svc.yml up -d
done
```

Servicios construidos localmente (OpenCode, ComfyUI, YOLO, RAG): se reconstruyen sin caché para traer la última versión del código.

```bash
for svc in opencode comfyui yolo rag; do
  docker compose -f docker-$svc.yml build --no-cache --pull
  docker compose -f docker-$svc.yml up -d
done
```

Actualizar modelos de Ollama:

```bash
docker exec ollama ollama pull llama3.2:3b
```

Limpiar imágenes antiguas:

```bash
docker image prune -f              # solo imágenes sin etiqueta
docker system df                   # ver cuánto espacio ocupa Docker
```

> Para entornos estables, fija versiones concretas en el `.env` (por ejemplo `OLLAMA_VERSION=0.x.y`) en lugar de `latest`, y actualiza de forma controlada.

### 6.3. Resolución de errores comunes

#### Error de permisos en volúmenes (`permission denied`)

Síntoma: un contenedor se reinicia en bucle y el log muestra `Permission denied` al escribir en `/etc/searxng`, `/data`, etc.

```bash
# Ver qué contenedor falla
docker ps -a
docker logs --tail 30 <contenedor>

# SearXNG se ejecuta con el usuario 977 dentro del contenedor
sudo chown -R 977:977 $HOME/searxng

# Resto de servicios (se ejecutan como root dentro del contenedor): que el propietario sea tu usuario
sudo chown -R $USER:$USER $HOME/{ollama,openwebui,hermes,yolo,rag,comfyui,opencode}

docker restart searxng
```

Nunca uses `chmod -R 777` como solución permanente.

#### El contenedor no ve la GPU (`could not select device driver "nvidia"`)

```bash
nvidia-smi                                         # ¿funciona en el servidor?
docker info | grep -i runtimes                     # debe aparecer "nvidia"
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubuntu24.04 nvidia-smi
```

#### `port is already allocated`

Otro proceso usa el puerto del host.

```bash
sudo ss -tulpn | grep -E ':(11434|3000|8000|8443|8188|5000|8080|8100)\b'
```

Cambia el puerto correspondiente en el `.env` y relanza el servicio.

#### Error `CUDA out of memory` o el modelo va lento

La VRAM está agotada por varios servicios a la vez.

```bash
nvidia-smi                                         # ¿quién consume?
docker exec ollama ollama ps
docker exec ollama ollama stop <modelo>            # descargar un modelo de memoria
docker compose -f docker-yolo.yml stop             # parar el servicio que no uses
```

También ayuda bajar `OLLAMA_KEEP_ALIVE` (por ejemplo `30s`) y usar modelos más pequeños o con mayor cuantización.

#### Open WebUI no muestra modelos

```bash
docker run --rm --network red-ia curlimages/curl -s http://ollama:11434/api/tags
```

Si responde con la lista de modelos, revisa en Open WebUI *Ajustes → Conexiones → Ollama* (URL `http://ollama:11434`). Si no responde, Ollama no está en la red `red-ia`.

#### SearXNG devuelve error 403 al pedir `format=json`

Falta `json` en `search.formats` de `$HOME/searxng/settings.yml`. Corrígelo y reinicia: `docker restart searxng`.

#### `network red-ia declared as external, but could not be found`

```bash
docker network create --driver bridge red-ia
```

#### Un servicio falla al construirse (`docker compose build`)

```bash
docker compose -f docker-<servicio>.yml build --no-cache --progress=plain 2>&1 | tail -50
```

Suele ser un problema temporal de red o de descarga de paquetes: reintenta.

#### Disco lleno

```bash
df -h
docker system df
docker builder prune -f
du -sh $HOME/{ollama,comfyui,yolo,openwebui} 2>/dev/null
```

### 6.4. Si `nvidia-smi` falla en el servidor

```bash
# ¿Se detecta la tarjeta?
lspci | grep -i nvidia
# ¿Hay driver cargado?
lsmod | grep nvidia
# Si usas Secure Boot, el módulo puede estar bloqueado: revisa
mokutil --sb-state
```

Con Secure Boot activado hay que firmar el módulo (Ubuntu lo ofrece durante la instalación del driver mediante MOK) o desactivar Secure Boot en la BIOS.

---

## 7. Guía interna de integración entre servicios

Dentro de `red-ia` **siempre se usan nombres de contenedor y puertos internos**, nunca `localhost` ni la IP del host ni el puerto externo. Para probar cualquier conexión desde dentro de la red, usa un contenedor temporal:

```bash
docker run --rm --network red-ia curlimages/curl -s <URL>
```

### 7.1. Ollama ⇄ Open WebUI

| Dato | Valor |
|---|---|
| Configurado en | `docker-openwebui.yml` → `OLLAMA_BASE_URL=http://ollama:11434` |
| Verificar | `docker run --rm --network red-ia curlimages/curl -s http://ollama:11434/api/tags` |

En la interfaz: **Ajustes de administración → Conexiones → Ollama API** → `http://ollama:11434`. Los modelos descargados aparecen en el selector de chat.

### 7.2. SearXNG ⇄ Open WebUI (búsqueda web)

Ya viene preconfigurado por variables de entorno. Verifica y actívalo:

```bash
docker run --rm --network red-ia curlimages/curl -s "http://searxng:8080/search?q=test&format=json" | head -c 200
```

En Open WebUI: **Ajustes de administración → Búsqueda web** → motor `searxng` y URL de consulta `http://searxng:8080/search?q=<query>`. En un chat, pulsa el icono de **búsqueda web** antes de enviar la pregunta.

### 7.3. ComfyUI ⇄ Open WebUI (generación de imágenes)

1. Abre ComfyUI (`http://IP_SERVIDOR:8188`), carga un workflow de texto a imagen y exporta con **Save (API Format)**.  
   *(Si no ves esa opción, activa el "Dev mode" en los ajustes de ComfyUI.)*
2. En Open WebUI: **Ajustes de administración → Imágenes**.
3. Motor de imágenes: `ComfyUI`; URL base: `http://comfyui:8188`.
4. Pega el JSON del workflow en el campo de workflow y asigna los nodos (prompt, modelo, ancho, alto, semilla...).
5. Prueba pulsando el botón de generar imagen en un chat.

```bash
docker run --rm --network red-ia curlimages/curl -s http://comfyui:8188/system_stats | head -c 300
```

### 7.4. Ollama ⇄ RAG propio

```bash
# 1) Descargar el modelo de embeddings (si no lo hiciste)
docker exec ollama ollama pull nomic-embed-text

# 2) Dejar documentos en la carpeta de conocimiento
cp mis_apuntes.md $HOME/rag/docs/

# 3) Indexar
curl -s -X POST http://localhost:8100/ingest

# 4) Preguntar
curl -s -X POST http://localhost:8100/query \
  -H "Content-Type: application/json" \
  -d '{"question": "¿Qué dicen mis apuntes sobre ...?", "k": 4}'
```

La conexión con Ollama usa `OLLAMA_BASE_URL=http://ollama:11434` (definida en `docker-rag.yml`). Los cálculos de embeddings y respuesta ocurren **en la GPU de Ollama**.

**RAG integrado dentro de Open WebUI (alternativa sin código):** en **Espacio de trabajo → Conocimiento** crea una colección, sube tus documentos y úsala en un chat escribiendo `#nombre_coleccion`. Ya está configurado para usar `nomic-embed-text` de Ollama como motor de embeddings.

### 7.5. Ollama ⇄ Hermes Agent

Ollama ofrece una API compatible con OpenAI en `http://ollama:11434/v1`. El asistente de configuración de Hermes se ejecuta una sola vez, de forma interactiva:

```bash
docker run -it --rm --network red-ia \
  -v $HOME/hermes:/opt/data \
  nousresearch/hermes-agent setup
```

En el asistente elige **custom endpoint** y responde:

| Pregunta | Respuesta |
|---|---|
| Base URL | `http://ollama:11434/v1` |
| API key | `ollama` (cualquier texto, Ollama no la valida) |
| Modelo | `qwen2.5:7b` (o el que hayas descargado) |

Después reinicia el servicio y revisa los logs:

```bash
docker compose -f docker-hermes-agent.yml restart
docker logs --tail 50 hermes-agent
```

> Los agentes necesitan contexto largo. Si el agente "olvida" instrucciones, sube `OLLAMA_CONTEXT_LENGTH` en el `.env` (16384 o más), teniendo en cuenta que consume más VRAM.

### 7.6. SearXNG ⇄ Hermes Agent y ComfyUI ⇄ Hermes Agent

El contenedor ya recibe `SEARXNG_URL=http://searxng:8080` y `COMFYUI_URL=http://comfyui:8188`. Dentro de Hermes configura las herramientas (*tools*) de búsqueda web y de generación de imágenes apuntando a esas URL, según la documentación de la versión instalada. Comprueba la conectividad desde el propio contenedor:

```bash
docker exec hermes-agent sh -c "getent hosts searxng comfyui ollama"
```

Debe devolver una IP para cada nombre. Si el contenedor no trae `curl`, usa el contenedor temporal `curlimages/curl` descrito al inicio de esta sección.

### 7.7. Ollama ⇄ OpenCode

Crea `$HOME/opencode/config/opencode.json`:

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
        "qwen2.5-coder:7b": { "name": "Qwen2.5 Coder 7B" },
        "llama3.2:3b": { "name": "Llama 3.2 3B" }
      }
    }
  },
  "model": "ollama/qwen2.5-coder:7b"
}
```

```bash
docker compose -f docker-opencode.yml restart
```

Entra en `http://IP_SERVIDOR:8443`, inicia sesión con `OPENCODE_USER` / `OPENCODE_PASSWORD` y elige el modelo `Ollama (local)`.

### 7.8. SearXNG ⇄ OpenCode

OpenCode puede usar herramientas externas mediante servidores **MCP**. El paquete `mcp-searxng` permite buscar con tu SearXNG privado. Añade el bloque `mcp` al mismo `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": { "baseURL": "http://ollama:11434/v1" },
      "models": {
        "qwen2.5-coder:7b": { "name": "Qwen2.5 Coder 7B" },
        "llama3.2:3b": { "name": "Llama 3.2 3B" }
      }
    }
  },
  "model": "ollama/qwen2.5-coder:7b",
  "mcp": {
    "searxng": {
      "type": "local",
      "command": ["npx", "-y", "mcp-searxng"],
      "environment": { "SEARXNG_URL": "http://searxng:8080" },
      "enabled": true
    }
  }
}
```

```bash
docker compose -f docker-opencode.yml restart
docker logs --tail 30 opencode
```

En una conversación de OpenCode, pide algo como *"busca en internet las novedades de Docker Compose"*; debería invocar la herramienta `searxng`. La primera ejecución descarga el paquete con `npx`, así que puede tardar unos segundos.

### 7.9. Ollama ⇄ YOLO ⇄ Open WebUI (opcional)

YOLO es independiente de los LLM: recibe imágenes y devuelve detecciones. Se usa directamente desde la API:

```bash
curl -s -X POST -F "file=@foto.jpg" "http://localhost:5000/detect?conf=0.4"
```

Desde otros contenedores de la red, la URL es `http://yolo:5000/detect`.

### 7.10. Mapa resumen de integraciones

```text
                         ┌────────────┐
                         │  Ollama    │◄────────────┐
                         │  :11434    │             │
                         └─────▲──────┘             │
        ┌───────────┬──────────┼───────────┬────────┴─────┐
        │           │          │           │              │
 ┌──────┴─────┐ ┌───┴─────┐ ┌──┴────────┐ ┌┴───────────┐ ┌┴─────┐
 │ Open WebUI │ │ Hermes  │ │ OpenCode  │ │    RAG     │ │ ...  │
 │   :8080    │ │ :8000   │ │  :8080    │ │   :8100    │ └──────┘
 └──┬───┬─────┘ └──┬───┬──┘ └─────┬─────┘ └────────────┘
    │   │          │   │          │
    │   └──────────┼───┼──────────┤
    │              │   │          │
 ┌──▼──────┐   ┌───▼───▼──┐   ┌───▼─────┐        ┌────────┐
 │ ComfyUI │   │ SearXNG  │◄──┘         │        │  YOLO  │
 │  :8188  │   │  :8080   │             │        │ :5000  │
 └─────────┘   └──────────┘                      └────────┘
          (todos en la red Docker "red-ia")
```

---

## 8. Checklist final de criterios de aceptación

Marca cada punto antes de dar la instalación por terminada.

| # | Criterio | Cómo comprobarlo | OK |
|---|---|---|---|
| 1 | Los ficheros `docker-<servicio>.yml` son funcionales y sin errores sintácticos | `for f in docker-*.yml; do docker compose -f $f config --quiet && echo OK; done` | ☐ |
| 2 | Los servicios de cálculo usan la GPU NVIDIA (CUDA) | `docker exec <ollama\|comfyui\|yolo\|openwebui> nvidia-smi` y `docker exec ollama ollama ps` (debe indicar `100% GPU`) | ☐ |
| 3 | No hay colisiones de puertos | `docker ps --format '{{.Names}}: {{.Ports}}'` (puertos del host: 11434, 3000, 8000, 8443, 8188, 5000, 8080, 8100) | ☐ |
| 4 | Manual paso a paso para un administrador novato | Seguir las secciones 1 a 7 en orden sobre un servidor limpio | ☐ |
| 5 | Los 8 contenedores están en la red `red-ia` | `docker network inspect red-ia --format '{{range .Containers}}{{.Name}} {{end}}'` | ☐ |
| 6 | Persistencia correcta tras reiniciar | `docker compose -f docker-openwebui.yml down && docker compose -f docker-openwebui.yml up -d` y comprobar que los usuarios y chats siguen ahí | ☐ |

---

*Fin del manual. Versión 1.0 - Proyecto SDD "Despliegue de una pila de IA local en Docker sobre Ubuntu".*
