# Manual Técnico de Instalación, Configuración y Operación: Pila de IA Local en Docker sobre Ubuntu

Este documento constituye el **Manual de instalación, configuración y operación** detallado paso a paso para desplegar una infraestructura completa de Inteligencia Artificial local sobre **Ubuntu Server** utilizando **Docker**, **Docker Compose** y aceleración por GPU **NVIDIA (CUDA)**.

---

## 1. Prerrequisitos e instalación base

### 1.1. Actualización del sistema e instalación de paquetes esenciales
Actualiza el índice de paquetes y el sistema operativo a la última versión:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git build-essential software-properties-common ca-certificates gnupg lsb-release
```

---

### 1.2. Instalación de Drivers de NVIDIA y CUDA Toolkit
Para hacer uso de la GPU (e.g., RTX 3050, 4060), instala los controladores oficiales de NVIDIA:

```bash
# Añadir el repositorio de controladores de NVIDIA
sudo add-apt-repository ppa:graphics-drivers/ppa -y
sudo apt update

# Instalación del driver recomendado de NVIDIA (Driver + CUDA)
sudo apt install -y nvidia-driver-550 nvidia-cuda-toolkit
```

> **Nota:** Tras completar la instalación de los drivers, es necesario reiniciar el servidor:
> ```bash
> sudo reboot
> ```

Comprueba la correcta instalación y funcionamiento de la GPU mediante:

```bash
nvidia-smi
```

---

### 1.3. Instalación de Docker y Docker Compose
Agrega el repositorio oficial de Docker e instala Docker Engine junto con el complemento Docker Compose:

```bash
# Añadir la clave GPG oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Configurar el repositorio de Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker Engine, CLI y Docker Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Agregar el usuario actual al grupo docker para ejecutar comandos sin 'sudo'
sudo usermod -aG docker $USER
newgrp docker
```

---

### 1.4. Instalación de NVIDIA Container Toolkit
Permite que los contenedores Docker accedan directamente a los recursos de la GPU NVIDIA:

```bash
# Configurar el repositorio del NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb [^ ]*#& [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg]#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Instalar el toolkit
sudo apt update
sudo apt install -y nvidia-container-toolkit

# Configurar la integración con Docker y reiniciar el servicio
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Verifica la aceleración GPU dentro de un contenedor Docker de prueba:

```bash
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu22.04 nvidia-smi
```

---

## 2. Estructura del proyecto

Todos los componentes de la pila se organizan dentro del directorio raíz `$HOME/proyecto`.

Crea el árbol de directorios y volúmenes de persistencia:

```bash
mkdir -p $HOME/proyecto/config
mkdir -p $HOME/ollama
mkdir -p $HOME/openwebui
mkdir -p $HOME/hermes
mkdir -p $HOME/opencode
mkdir -p $HOME/comfyui
mkdir -p $HOME/yolo
mkdir -p $HOME/searxng
mkdir -p $HOME/rag
```

Estructura de ficheros generada:

```
$HOME/
├── ollama/
├── openwebui/
├── hermes/
├── opencode/
├── comfyui/
├── yolo/
├── searxng/
├── rag/
└── proyecto/
    ├── .env
    ├── docker-ollama.yml
    ├── docker-openwebui.yml
    ├── docker-hermes-agent.yml
    ├── docker-opencode.yml
    ├── docker-comfyui.yml
    ├── docker-yolo.yml
    ├── docker-searxng.yml
    ├── docker-rag.yml
    └── config/
        └── searxng/
            └── settings.yml
```

---

## 3. Fichero de entorno (.env)

Crea el archivo `$HOME/proyecto/.env` con todas las variables requeridas por los servicios:

```env
# Directorio raíz del proyecto y almacenamiento
HOME_DIR=/home/usuario

# Configuración de Red
NETWORK_NAME=red-ia

# Ollama
OLLAMA_PORT=11434
OLLAMA_VOLUME=${HOME_DIR}/ollama

# Open WebUI
OPENWEBUI_PORT=3000
OPENWEBUI_VOLUME=${HOME_DIR}/openwebui
WEBUI_SECRET_KEY=clave_secreta_super_segura_para_openwebui

# Hermes Agent
HERMES_PORT=8000
HERMES_VOLUME=${HOME_DIR}/hermes

# OpenCode
OPENCODE_PORT=8443
OPENCODE_VOLUME=${HOME_DIR}/opencode

# ComfyUI
COMFYUI_PORT=8188
COMFYUI_VOLUME=${HOME_DIR}/comfyui

# YOLO
YOLO_PORT=5000
YOLO_VOLUME=${HOME_DIR}/yolo

# SearXNG
SEARXNG_PORT=8080
SEARXNG_VOLUME=${HOME_DIR}/searxng
SEARXNG_SECRET_KEY=clave_secreta_searxng_123456

# RAG
RAG_PORT=11435
RAG_VOLUME=${HOME_DIR}/rag
```

> **Importante:** Reemplaza `/home/usuario` en `HOME_DIR` por la ruta absoluta real de tu directorio personal (`echo $HOME`).

---

## 4. Ficheros de configuración de Docker Compose

Todos los archivos YAML deben ser creados en `$HOME/proyecto/`. Utilizan la red personalizada `red-ia`.

Para preparar la red compartida:

```bash
docker network create red-ia
```

---

### 4.1. `docker-ollama.yml`
Motor de ejecución para LLMs con aceleración por GPU.

```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  ollama:
    container_name: ollama
    image: ollama/ollama:latest
    ports:
      - "${OLLAMA_PORT}:11434"
    volumes:
      - ${OLLAMA_VOLUME}:/root/.ollama
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: always
```

---

### 4.2. `docker-openwebui.yml`
Interfaz gráfica de usuario conectada con Ollama, SearXNG y ComfyUI.

```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  openwebui:
    container_name: openwebui
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "${OPENWEBUI_PORT}:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      - ENABLE_SEARXNG_QUERY=true
      - SEARXNG_QUERY_URL=http://searxng:8080/search
    volumes:
      - ${OPENWEBUI_VOLUME}:/app/backend/data
    networks:
      - red-ia
    depends_on:
      - ollama
    restart: always
```

---

### 4.3. `docker-hermes-agent.yml`
Agente autónomo para la ejecución de tareas complejas.

```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  hermes-agent:
    container_name: hermes-agent
    image: python:3.11-slim
    command: >
      bash -c "pip install --no-cache-dir fastapi uvicorn requests &&
               python -c 'from fastapi import FastAPI; import uvicorn; app = FastAPI(); @app.get(\"/\") def read_root(): return {\"status\": \"Hermes Agent Running\"}; uvicorn.run(app, host=\"0.0.0.0\", port=8000)'"
    ports:
      - "${HERMES_PORT}:8000"
    environment:
      - OLLAMA_HOST=http://ollama:11434
      - SEARXNG_HOST=http://searxng:8080
      - COMFYUI_HOST=http://comfyui:8188
    volumes:
      - ${HERMES_VOLUME}:/app/data
    networks:
      - red-ia
    restart: always
```

---

### 4.4. `docker-opencode.yml`
Entorno de desarrollo web para programación con asistencia por IA.

```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  opencode:
    container_name: opencode
    image: lscr.io/linuxserver/code-server:latest
    ports:
      - "${OPENCODE_PORT}:8443"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - DEFAULT_WORKSPACE=/config/workspace
    volumes:
      - ${OPENCODE_VOLUME}:/config
    networks:
      - red-ia
    restart: always
```

---

### 4.5. `docker-comfyui.yml`
Servicio para la generación y procesamiento multimedia con GPU.

```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  comfyui:
    container_name: comfyui
    image: yananyi/comfyui:latest
    ports:
      - "${COMFYUI_PORT}:8188"
    volumes:
      - ${COMFYUI_VOLUME}:/app/data
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: always
```

---

### 4.6. `docker-yolo.yml`
API y motor de visión artificial para detección en tiempo real.

```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  yolo:
    container_name: yolo
    image: ultralytics/ultralytics:latest
    command: python -m ultralytics.solutions.v8
    ports:
      - "${YOLO_PORT}:5000"
    volumes:
      - ${YOLO_VOLUME}:/usr/src/app/data
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: always
```

---

### 4.7. `docker-searxng.yml` y configuración
Metabuscador privado. Requiere crear primero el archivo de configuración en `$HOME/proyecto/config/searxng/settings.yml`:

```yaml
general:
  debug: false
  instance_name: "SearXNG Local"

search:
  safe_search: 0
  autocomplete: ""

server:
  port: 8080
  bind_address: "0.0.0.0"
  secret_key: "clave_secreta_searxng_123456"
  limiter: false
```

Archivo `docker-searxng.yml`:

```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  searxng:
    container_name: searxng
    image: searxng/searxng:latest
    ports:
      - "${SEARXNG_PORT}:8080"
    volumes:
      - ${HOME_DIR}/proyecto/config/searxng/settings.yml:/etc/searxng/settings.yml:ro
      - ${SEARXNG_VOLUME}:/var/yml/searxng
    networks:
      - red-ia
    restart: always
```

---

### 4.8. `docker-rag.yml`
Servicio para aumento de contexto documental mediante Ollama.

```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  rag:
    container_name: rag
    image: chromadb/chroma:latest
    ports:
      - "${RAG_PORT}:8000"
    environment:
      - ALLOW_RESET=TRUE
      - OLLAMA_HOST=http://ollama:11434
    volumes:
      - ${RAG_VOLUME}:/chroma/chroma
    networks:
      - red-ia
    depends_on:
      - ollama
    restart: always
```

---

## 5. Despliegue y verificación

### 5.1. Despliegue de los servicios
Ubícate en la carpeta del proyecto y arranca los archivos de Docker Compose:

```bash
cd $HOME/proyecto

# Despliegue individual o en cadena de todos los componentes
docker compose -f docker-ollama.yml --env-file .env up -d
docker compose -f docker-searxng.yml --env-file .env up -d
docker compose -f docker-comfyui.yml --env-file .env up -d
docker compose -f docker-yolo.yml --env-file .env up -d
docker compose -f docker-rag.yml --env-file .env up -d
docker compose -f docker-openwebui.yml --env-file .env up -d
docker compose -f docker-hermes-agent.yml --env-file .env up -d
docker compose -f docker-opencode.yml --env-file .env up -d
```

---

### 5.2. Verificación del estado y logs
Comprueba que los contenedores están en estado `Up`:

```bash
docker ps
```

Verificación de logs específicos por servicio:

```bash
docker logs -f ollama
docker logs -f openwebui
```

---

### 5.3. Descarga e inicialización de un modelo en Ollama
Para comenzar a operar, descarga un modelo LLM básico en Ollama:

```bash
docker exec -it ollama ollama run llama3
```

---

### 5.4. Verificación del uso de GPU en contenedores
Confirma que los procesos dentro del contenedor están haciendo consumo de la tarjeta NVIDIA:

```bash
nvidia-smi
```

---

### 5.5. Matriz de puertos y URLs de acceso al Host

| Servicio | Nombre Contenedor | URL Externa (Host) | URL Interna (Red Docker) |
| :--- | :--- | :--- | :--- |
| **Ollama** | ollama | `http://<IP-HOST>:11434` | `http://ollama:11434` |
| **Open WebUI** | openwebui | `http://<IP-HOST>:3000` | `http://openwebui:8080` |
| **Hermes Agent** | hermes-agent | `http://<IP-HOST>:8000` | `http://hermes-agent:8000` |
| **OpenCode** | opencode | `http://<IP-HOST>:8443` | `http://opencode:8443` |
| **ComfyUI** | comfyui | `http://<IP-HOST>:8188` | `http://comfyui:8188` |
| **YOLO** | yolo | `http://<IP-HOST>:5000` | `http://yolo:5000` |
| **SearXNG** | searxng | `http://<IP-HOST>:8080` | `http://searxng:8080` |
| **RAG** | rag | `http://<IP-HOST>:11435` | `http://rag:8000` |

---

## 6. Mantenimiento y actualización

### 6.1. Copias de seguridad (Backups)
Crea una copia de seguridad empaquetada de los volúmenes de datos:

```bash
# Detener los contenedores para garantizar consistencia
cd $HOME/proyecto
docker compose -f docker-openwebui.yml down
docker compose -f docker-ollama.yml down

# Crear copia comprimida de los datos
tar -czvf $HOME/backup_ia_$(date +%Y%m%d).tar.gz \
  $HOME/ollama \
  $HOME/openwebui \
  $HOME/hermes \
  $HOME/opencode \
  $HOME/comfyui \
  $HOME/yolo \
  $HOME/searxng \
  $HOME/rag

# Volver a levantar los servicios
docker compose -f docker-ollama.yml --env-file .env up -d
docker compose -f docker-openwebui.yml --env-file .env up -d
```

---

### 6.2. Actualización de servicios
Para actualizar un contenedor a su última imagen disponible:

```bash
cd $HOME/proyecto
docker compose -f docker-openwebui.yml pull
docker compose -f docker-openwebui.yml --env-file .env up -d --force-recreate
```

---

### 6.3. Resolución de errores comunes (Permisos y almacenamiento)
En caso de fallos de escritura por permisos en los volúmenes enlazados:

```bash
sudo chown -R $USER:$USER $HOME/ollama $HOME/openwebui $HOME/hermes $HOME/opencode $HOME/comfyui $HOME/yolo $HOME/searxng $HOME/rag
chmod -R 755 $HOME/proyecto
```

---

## 7. Guía de integración interna

La comunicación entre servicios se realiza a través de la red de puente `red-ia` mediante resolución DNS interna de contenedores.

### 7.1. Conexión de Open WebUI con Ollama y SearXNG
* **Ollama:** Configurado automáticamente mediante la variable de entorno `OLLAMA_BASE_URL=http://ollama:11434`.
* **SearXNG (Búsqueda web en tiempo real):**
  1. Acceder a Open WebUI (`http://<IP-HOST>:3000`).
  2. Ir a **Admin Panel -> Settings -> Web Search**.
  3. Activar **Web Search**.
  4. Seleccionar el motor **SearXNG**.
  5. Introducir la URL del contenedor: `http://searxng:8080/search`.

---

### 7.2. Conexión de Hermes Agent con la Pila de Servicios
El contenedor `hermes-agent` hace uso de las variables declaradas en su archivo de composición:
* Acceso a LLM: `http://ollama:11434`
* Consultas Web: `http://searxng:8080`
* Generación de Imágenes: `http://comfyui:8188`

---

### 7.3. Conexión de OpenCode con Ollama
1. Acceder a OpenCode (`http://<IP-HOST>:8443`).
2. Instalar extensiones como **Continue** o **Codeium**.
3. En la configuración de la extensión, definir el proveedor como **Ollama** y el punto de enlace de la API como:
   `http://ollama:11434`.