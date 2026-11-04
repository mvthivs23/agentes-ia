# Objetivo 14 — Autonomous Agent Architecture

Esta sección introduce los principios de diseño y los patrones de arquitectura necesarios para construir agents LLM autónomos para seguridad ofensiva. Estos agents operan a lo largo de varias etapas de la cyber kill chain —reconocimiento, acceso inicial, explotación y post-explotación— y ejecutan flujos definidos con razonamiento, memoria e interacción con tools.

## 6.2.1 Qué es un flujo autónomo

Un flujo de agent autónomo es un bucle en el que un modelo de lenguaje:

- Interpreta la tarea o el objetivo actual
- Elige la siguiente acción o tool más adecuada
- Ejecuta la acción
- Actualiza su contexto o su memoria
- Repite hasta una condición terminal o hasta cumplir el objetivo

Este proceso refleja las etapas clásicas de la kill chain, pero con automatización y toma de decisiones dirigidas por el LLM.

## 6.2.2 Elementos clave del diseño de un agent ofensivo

| Componente | Descripción |
| --- | --- |
| Definición del objetivo | Objetivo acotado con claridad (por ejemplo, «encuentra APIs expuestas en target.com») |
| Montaje del entorno | Conjunto de tools del agent, permisos, límites de tasa, claves de API, conjuntos de datos |
| Espacio de acciones | Conjunto de tools o funciones que el agent tiene permitido usar |
| Memoria / scratchpad | Contexto a corto plazo (de paso a paso) o a largo plazo (grafos OSINT, credenciales) |
| Condición de terminación | Punto de parada definido, o un indicador de éxito o de fallo |

## 6.2.3 Ejemplos de plano de flujo

### Ejemplo 1: agent de reconocimiento OSINT

**Objetivo:** recoger y enriquecer OSINT de un dominio dado.

1. Ejecuta enumeración de subdominios (Subfinder).
2. Usa HTTPX para sondear los subdominios vivos.
3. Consulta Shodan/Censys para información de IP.
4. Extrae tecnologías con WhatWeb.
5. Guarda los resultados en una vector database para consultas semánticas.

### Ejemplo 2: agent de evaluación de exploits

**Objetivo:** evaluar vulnerabilidades de una lista de CVEs conocidas.

1. Parsea las CVEs del resultado de escaneo del objetivo.
2. Busca cada CVE en ExploitDB / NVD.
3. Si existe un exploit, genera un PoC o un módulo de Metasploit.
4. Comprueba prerrequisitos (SO, puerto, servicio).
5. Devuelve una puntuación de explotabilidad + payload.

## 6.2.4 Patrón de diseño del flujo: ReAct + uso de tools

ReAct (Reasoning + Acting) es un patrón habitual en el que:

- El agent razona por pasos intermedios (piensa dentro de los prompts)
- El agent actúa llamando a tools con argumentos
- La salida se devuelve al LLM para los pasos siguientes

Traza de razonamiento de ejemplo:

```text
Thought: Necesito encontrar subdominios
Action: run_subfinder("target.com")
Observation: 23 subdomains found
Thought: Los voy a sondear con HTTPX
...
```

`Observation` se deja como la devuelve la tool.

## 6.2.5 Encadenamiento multi-agent

Los flujos autónomos se pueden descomponer en varios agents, cada uno con una especialidad:

| Rol del agent | Función |
| --- | --- |
| Agent de OSINT | Recoge datos de la web, DNS, GitHub, etc. |
| Agent de fingerprinting | Mapea tecnologías y puertos |
| Agent de exploit | Elabora o recupera payloads de CVE |
| Agent de C2 | Despliega infraestructura y conecta |

Los agents pueden comunicarse mediante:

- Memoria compartida (por ejemplo, FAISS, Redis)
- Sistema de ficheros (por ejemplo, logs, CSV)
- Paso de mensajes con colas de eventos

## 6.2.6 Planificación de tareas y nivel de autonomía

Define niveles de autonomía:

- **Humano en el bucle:** el agent necesita aprobación antes de acciones críticas
- **Totalmente autónomo:** ejecuta una cadena de ataque predefinida sin pausas
- **Híbrido:** el agent propone y el humano aprueba pasos concretos

Define la lógica de planificación de tareas:

- Ejecutar una vez
- Programado (por ejemplo, trabajos diarios de reconocimiento)
- Disparado por un evento (por ejemplo, push a GitHub → escaneo)

## 6.2.7 Compuertas de seguridad y ética en los flujos

Antes del despliegue:

- Añade restricciones en los prompts (por ejemplo, «actúa solo sobre objetivos autorizados»)
- Embebe una compuerta ética: `IF not in allowlist THEN abort`
- Registra cada acción (pista de auditoría)

Durante la ejecución:

- Monitoriza el uso de la API y los patrones de respuesta
- Añade filtros contra llamadas a tools alucinadas o peligrosas
- Usa listas de permitidos o de bloqueo para dominios e IPs

## Resumen

Diseñar flujos ofensivos con agents LLM permite automatizar operaciones de red team complejas a lo largo de la cyber kill chain. Si estructuras la tarea, defines espacios de acción seguros y usas patrones tipo ReAct con el tooling adecuado, puedes crear agents orientados a un objetivo, adaptativos y auditables: el siguiente paso de la seguridad ofensiva asistida por IA.
