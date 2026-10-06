---
title: "Proyecto basado en SDD (Spec Driven Development): Despliegue de una pila de IA Local en Docker sobre Ubuntu"
author: "Mnarrieta"
date: "2026-09-29"
category: "Despliegue IA"
tags: [markdown,ia,prompt,local,SDD]
---

# Proyecto basado en SDD (Spec Driven Development): Despliegue de una pila de IA Local en Docker sobre Ubuntu

- **Versión:** 1.0
- **Rol del creador:** Administrador de sistemas
- **Propósito:** Definir un manual técnico de requisitos y definiendo una arquitectura para la generación de un manual técnico detallado con la instalación, configuración, tests y mantenimiento en formato markdown (.md)

## 1. Visión general del proyecto (Objetivo)
El objetivo del proyecto es desplegar una infraestructura de Inteligencia Artificial local utilizando contenedores Docker en un sistema operativo Ubuntu Server. Cada servicio residirá en su propio contenedor docker. El sistema dispone de tarjeta gráfica NVIDIA (GPU).

## 2. Servicios, especificaciones  y aplicaciones
Los servicios a desplegar son los siguientes:
| Servicio | Nombre de contenedor | Puerto interno | Puerto externo (Host) | Propósito principal | Dependencias |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Ollama** | ollama | 11434 | 11434 | Motor de LLMs locales y servidor de API | GPU NVIDIA y Driver (CUDA) |
| **Open WebUI** | openwebui | 8080 | 3000 | Interfaz web tipo ChatGPT para interactuar con LLMs | Ollama, SearXNG, ComfyUI |
| **Hermes Agent** | hermes-agent | 8000 | 8000 | Arnés para el motor LLMs, Agente autónomo para realizar tareas complejas | Ollama, SearXNG, ComfyUI |
| **OpenCode** | opencode | 8080 | 8443 | Entorno IDE para desarrollo de código (entorno web) similar a Claude  | Ollama o ninguno |
| **ComfyUI** | comfyui | 8188 | 8188 | Interfaz web para la generación, edición y procesamiento de imágenes y vídeo | GPU driver, sus propios LLMs |
| **YOLO** | yolo | 5000 | 5000 | API o herramienta par la visión artifical, detección y reconocimiento de patrones en tiempo real en imágenes | GPU driver, sus propios LLMs |
| **SearXNG** | searxng | 8080 | 8080 | Metabuscador privado para realizar búsquedas en Internet | GPU driver |
| **RAG** | rag | 11434 | 11434 | Técnica para aumentar la capacidad de un modelo de lenguaje LLM con información externa privada | Ollama, documentación externa | 

## 3. Arquitectura de red y datos

### 3.1. Redes Docker
Red principal que se va a llamar "red-ia" a la que van a pertenecer todos los contenedores para poder comunicarse entre sí. Red en modo "bridge" y crear un sistema de naming (DNS) local del modo siguiente:
| Servicio | Nombre de contenedor | URL |
| :--- | :--- | :--- |
| **Ollama** | ollama | http://ollama:11434 |
| **Open WebUI** | openwebui | http://openwebui:3000 |
| **Hermes Agent** | hermes-agent | http://hermesagent:8000 |
| **OpenCode** | opencode | http://opencode:8443 |
| **ComfyUI** | comfyui | http://comfyui:8188 | 
| **YOLO** | yolo | http://yolo:5000 |
| **SearXNG** | searxng | http://searxng:8080 | 
| **RAG** | rag | ***Integrado con otros servicios*** | 

### 3.2. Volúmenes de datos (Persistencia)
Consiste en "mapear" un sistema de ficheros dentro de cada contenedor a la máquina física que los contiene.
| Nombre volumen | Unidad mapeada | Descripción |
| :--- | :--- | :--- |
| ollama_data | `$HOME/ollama` | almacenamiento modelos LLM |
| openwebui_data | `$HOME/openwebui` | usuarios, chats, prompts, configuraciones |
| **Hermes Agent** | `$HOME/hermes` | Configuración de los agentes |
| **OpenCode** | `$HOME/opencode` | Código de los proyectos y las configuraciones |
| **ComfyUI** | `$HOME/comfyui` | Imágesn y vídeos generados, prompts y modelos generativos | 
| **YOLO** | `$HOME/yolo` | Dataset de los vídeos o imágenes analizadas |
| **SearXNG** | `$HOME/searxng` | Resultados de las búsquedas realizadas | 
| **RAG** | `$HOME/rag` | Ficheros que se le dotan para aumentar el conocimiento a la IA |

## 4. Requisitos de sistema y hardware
1. **SO:** Ubuntu Server 24.04 o 26.04.
2. **GPU:** Tarjeta gráfica NVIDIA, algunos ordenadores con el modelo 3050, 4060.
3. **Drivers:** Driver NVIDIA CUDA o nvidia-drivers oficiales.
4. **Docker:** Sistema de contenedores para cada servicio, docker compose, y docker.

## 5. Instrucciones para generar el manual técnico
> **Instrucciones para la generación del documento de salida:**
> Actúa como un experto en administración de sistemas GNU/Linux y Devops, genera un **Manual de instalación, configuración y operación** exhaustivo y detallado en formato Markdown basado en esta especificación.
> El manual generado debe incluir obligatoriamente las siguientes secciones:
> 1. **Prerrequisitos e instalación base:** Comandos básicos en Linux para instalar Docker, Docker compose, drivers de NVIDIA CUDA, utilizando repositorios apt.
> 2. **Estructura del proyecto:** Árbol detallado de directorios para el stack que vamos a montar `$HOME/proyecto`
> 3. **Ficheros de configuración:** Un `docker-<servicio>.yml` por cada uno de los servicios que vamos a montar donde <servicio> se sustituye por el nombre del contenedor.
> 4. **Fichero de entorno:** Fichero `.env` con todas las variables del entorno de todos los servicios.
> 5. **Despliegue y verificación:** Comandos relacionados con el arranque de servicios (`docker compose up -d`), comprobación de logs, acceso a URLs de servicio, uso de la GPU (`nvidia-smi`).
> 6. **Mantenimiento y  actualización:** Comandos de backups, de actualización de los servicios, resolución de errores comunes (permisos)
> 7. **Guía interna de integración interna:** Donde se detalla como conectar de manera exacta los servicios que corren en los docker, por ejemplo, Ollama con OpenwebUI, Searxng con Opencode,... 

## 6. Criterios de aceptación
1. Los archivos de configuración `docker_<servicio>.yml` deben ser totalmente funcionales y sin errores sintácticos.
2. Todos los contenedores deben funcionar correctamente con las librerías de NVIDIA (CUDA), utilizando principalmente GPU.
3. No pueden existir colisiones entre puertos de los contenedores.
4. Las explicaciones del manual deben ser especificadas paso a paso para un administrador de sistemas novato.


