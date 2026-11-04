# Objetivo 02 — Tailored Offensive LLM Agents

Guiarte en la creación de agents de LLM hechos a medida para tareas de ciberseguridad ofensiva, como reconocimiento, explotación, phishing, generación de payloads o análisis de vulnerabilidades. Este módulo muestra cómo arquitectar un pipeline de agent LLM ofensivo, incorporar conocimiento de dominio y exigir aislamiento operativo.

## 2.4.1 Arquitectura de un agent para seguridad ofensiva

Un agent de IA ofensivo a medida suele seguir una arquitectura modular:

```text
+--------------------+
|   Input Handler    | ← prompt, CLI, API, telemetría
+--------------------+
         ↓
+--------------------+         +----------------+
|   Core LLM Engine  |  ←→     | Context Memory |
+--------------------+         +----------------+
         ↓
+--------------------+         +-------------------+
| Tool/Script Runner |  ←→     | Tool Plugins/API  |
+--------------------+         +-------------------+
         ↓
+--------------------+
|  Output Formatter  | ← salida cruda → informe, CLI
+--------------------+
```

- **Core LLM Engine:** un LLM de código abierto (por ejemplo Mistral, LLaMA 2, Zephyr o LLMs de código).
- **Tool plugins:** comandos de shell, tools de reconocimiento (por ejemplo whois, subfinder), escáneres de vulnerabilidades.
- **Context memory:** usa embeddings o almacenes vectoriales (por ejemplo FAISS) como memoria histórica de tareas.

## 2.4.2 Tools de implementación y SDKs

| Framework | Propósito | Lenguaje |
| --- | --- | --- |
| LangChain | Encadenamiento de tools, memoria de contexto, agents | Python |
| OpenLLM (BentoML) | Despliegue de LLM y de agents listo para API | Python |
| LocalAI | API de LLM compatible con OpenAI, de sustitución directa | Go |
| Transformers (HF) | Servicio local de modelos y fine-tuning | Python |

**Consejo:** combina LangChain + OpenLLM para crear agents flexibles, local-first, que interactúen con herramientas de línea de comandos.

## 2.4.3 Incorporar conocimiento ofensivo en los agents

Para especializar un agent en tareas de seguridad ofensiva:

- Entrena o haz fine-tuning con:
  - Descripciones de Exploit-DB
  - Resúmenes de CVE (por ejemplo, feeds de NVD)
  - Manuales de uso de tools (por ejemplo, nmap, msfconsole)
  - Técnicas de OSINT (OSINT framework, MITRE PRE-ATT&CK)
- Usa modelos de embedding para consultar bases de conocimiento estáticas grandes:
  - `BAAI/bge-base-en-v1.5` o `intfloat/e5-base` (para consultas)
  - Indexa como vector database con FAISS, Chroma o Weaviate
- **Prompt engineering:** adapta las instrucciones de rol para simular el comportamiento de un atacante.
  - «Eres un operador adversario especializado en reconocimiento cloud. ¿Cuáles son las 3 tools siguientes que debo ejecutar sobre esta IP?»

## 2.4.4 Conectar agents con tools ofensivas

Los agents pueden actuar como orquestadores autónomos de utilidades de red team:

| Tool | Rol en el flujo del agent | Invocación de ejemplo |
| --- | --- | --- |
| nmap | Reconocimiento de red | Escaneo de puertos sobre la IP objetivo |
| subfinder | Enumeración de subdominios | Ejecutar un escaneo OSINT pasivo |
| whatweb | Fingerprinting tecnológico | Analizar el stack de la URL objetivo |
| searchsploit | Referencia local de CVE y exploits | Cruzar los servicios descubiertos |
| nuclei | Escaneo automatizado de vulnerabilidades | Ejecutar escaneos con plantillas web |
| curl/httpx | Enumeración HTTP y comprobación de estado | Sondear URLs en busca de fingerprints |

Usa las tools del sistema mediante wrappers u objetos Tool de LangChain para evitar inyección de código y validar las salidas.

## 2.4.5 Ejemplo: agent generador de payloads de phishing

El fuente describe un system prompt y un flujo para armar correos de spear-phishing con un payload de fichero embebido (descargador y empaquetado). Esa parte no se reproduce aquí.

Las tools que el fuente asocia a ese ejemplo son un LLM (Mistral, Zephyr o GPT4All), un script de Python y una vector database de plantillas (Chroma).

## 2.4.6 Model Context Protocol (MCP) para agents

Para que los LLMs se puedan componer, el Model Context Protocol (MCP) define cómo distintos modelos y prompts comparten contexto y operan a lo largo de cadenas de tareas.

Cada módulo de tarea tiene:

- Un esquema de contexto de entrada
- Un esquema de resumen de salida
- Una traza de ejecución embebida (logs)

Esto permite:

- Operaciones de LLM encadenadas (por ejemplo, OSINT → generación de exploit → informe)
- Modelos intercambiables (usar GPT-4 para razonar y LLaMA para generación local de payloads)

## 2.4.7 Alojar y operar con seguridad

- Aísla los agents en Docker o en una VM.
- Desactiva el acceso a red en los módulos de generación de payloads.
- Registra prompts y respuestas para que haya transparencia.
- Valida siempre la salida antes de ejecutarla.

## 2.4.8 Resumen

Construir agents LLM ofensivos implica:

- Elegir el modelo base adecuado (local frente a API)
- Incorporar conocimiento operativo y prompts
- Conectarlos con tools externas y frameworks de OSINT
- Aplicar memoria, lógica y orquestación de la cadena de tools

Con las salvaguardas adecuadas, estos agents permiten simulación adversaria avanzada, creación rápida de payloads y red team realista en un entorno controlado.
