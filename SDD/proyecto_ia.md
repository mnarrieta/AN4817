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
| **RAG** | searxng | 8080 | 8080 |  |  | 

## 3. Arquitectura de red y datos

### 3.1. Redes Docker

### 3.2. Volúmenes de datos

## 4. Requisitos de sistema y hardware

## 5. Instrucciones para generar el manual técnico

## 6. Criterios de aceptación



