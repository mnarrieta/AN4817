# Comparativa y Despliegue de Pila de IA Local (SDD Project)

Este repositorio contiene la documentación técnica y los manuales de despliegue para una infraestructura de **Inteligencia Artificial local** basada en **Docker** sobre **Ubuntu Server**, acelerada por GPUs NVIDIA (CUDA).

El proyecto sigue la metodología **SDD (Spec Driven Development)**, partiendo de una especificación técnica formal definida en `proyecto_ia.md`.

## 📁 Estructura del Repositorio

La raíz de este repositorio contiene los manuales técnicos generados por tres modelos de lenguaje principales (LLMs) a partir de la misma especificación del proyecto, permitiendo comparar enfoques de arquitectura, configuración y administración de sistemas:

```
.
├── README.md               # Este archivo de presentación e índice
├── proyecto_ia.md          # Especificación original del proyecto (SDD)
├── manual_gemini.md        # Manual de instalación y operación generado por Google Gemini
├── manual_claude.md        # Manual de instalación y operación generado por Anthropic Claude
└── manual_chatgpt.md       # Manual de instalación y operación generado por OpenAI ChatGPT
```

---

## 🤖 Modelos de Inteligencia Artificial Utilizados

Para la generación de la documentación técnica y manuales de arquitectura se han empleado las siguientes versiones de modelos:

| Proveedor | Modelo Utilizado | Fichero Generado |
| :--- | :--- | :--- |
| **Google** | Gemini 3.6 Flash | `manual_gemini.md` |
| **Anthropic** | Claude Sonnet 5.5 | `manual_claude.md` |
| **OpenAI** | ChatGPT GPT-5.6 Luna Instant | `manual_chatgpt.md` |

---

## 🏗️️ Servicios Incluidos en el Stack

La pila de IA local está compuesta por los siguientes servicios aislados en contenedores Docker conectados mediante una red tipo *bridge* personalizada (`red-ia`):

| Servicio | Nombre Contenedor | Puerto Host | Puerto Interno | Propósito Principal | 
| :--- | :--- | :--- | :--- | :--- | 
| **Ollama** | `ollama` | `11434` | `11434` | Motor de LLMs y servidor de API con aceleración GPU | 
| **Open WebUI** | `openwebui` | `3000` | `8080` | Interfaz gráfica tipo ChatGPT conectada a Ollama/SearXNG | 
| **Hermes Agent** | `hermes-agent` | `8000` | `8000` | Arnés de agentes autónomos para tareas complejas | 
| **OpenCode** | `opencode` | `8443` | `8443` | Entorno IDE web asistido por IA para desarrollo | 
| **ComfyUI** | `comfyui` | `8188` | `8188` | Pipeline visual para generación de imágenes y vídeo con GPU | 
| **YOLO** | `yolo` | `5000` | `5000` | Motor API de visión artificial para detección en tiempo real | 
| **SearXNG** | `searxng` | `8080` | `8080` | Metabuscador privado para búsquedas en internet sin rastreo | 
| **RAG** | `rag` | `11435` | `8000` | Base de datos vectorial (ChromaDB) para aumento de contexto | 

---

## 📄 Resumen de los Manuales

Cada fichero de manual incluye:

1. **Prerrequisitos e instalación base:** Instalación de Docker, Docker Compose, drivers NVIDIA y `nvidia-container-toolkit`.
2. **Estructura de directorios y volúmenes:** Mapeo de almacenamiento persistente en `$HOME`.
3. **Archivos `docker-<servicio>.yml` y `.env`:** Declaraciones listas para producción.
4. **Despliegue y verificación:** Comandos de inicio, pruebas de GPU (`nvidia-smi`) y verificación de logs.
5. **Mantenimiento:** Rutinas de copia de seguridad (backups), actualización y permisos.
6. **Guía de integración interna:** Configuración para conectar Open WebUI, OpenCode y Hermes Agent con Ollama, SearXNG y ComfyUI.

### Enlaces directos a los manuales:

* [📖 Manual generado por Google Gemini 3.6 Flash](manual_gemini.md)
* [📖 Manual generado por Anthropic Claude Sonnet 5.5](manual_claude.md)
* [📖 Manual generado por OpenAI ChatGPT GPT-5.6 Luna Instant](manual_chatgpt.md)

---

## 🚀 Inicio Rápido (Quickstart)

Para desplegar la infraestructura utilizando la configuración recomendada:

1. **Clonar el repositorio y revisar los requisitos de hardware (Ubuntu 24.04/26.04 + GPU NVIDIA):**

   ```bash
   git clone <URL-del-repositorio>
   cd <nombre-del-repositorio>
   ```

2. **Crear la red Docker personalizada:**

   ```bash
   docker network create red-ia
   ```

3. **Consultar cualquiera de los manuales adjuntos para realizar el paso a paso de configuración:**

   ```bash
   cat manual_gemini.md
   ```

---

## 🛠️ Requisitos de Hardware y Sistema

* **Sistema Operativo:** Ubuntu Server 24.04 LTS o superior.
* **GPU:** NVIDIA (Soporte CUDA habilitado, e.g., RTX 3050, RTX 4060 o superior).
* **Controladores:** NVIDIA Drivers `>= 550` y `nvidia-container-toolkit`.
* **Runtime:** Docker Engine `>= 24.0` con el complemento `docker compose`.