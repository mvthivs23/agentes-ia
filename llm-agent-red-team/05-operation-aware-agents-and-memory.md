# Objetivo 05 — Operation-Aware Agents and Memory

Mostrarte cómo estructurar agents de IA que conocen la operación ofensiva en curso, conservan memoria de los pasos anteriores y encadenan tareas a lo largo del ciclo de vida de un ciberataque, usando el Model Context Protocol (MCP) y arquitecturas relacionadas.

## 3.3.1 Introducción a los agents conscientes del contexto

Los LLMs basados solo en un prompt no tienen estado. Las operaciones ofensivas necesitan estado, memoria y flujo entre tareas:

- El reconocimiento tiene que informar a la explotación.
- La explotación tiene que influir en las decisiones de post-explotación.
- Los agents de IA tienen que aprender de respuestas anteriores para mejorar las interacciones futuras.

Un agent consciente del contexto lo resuelve así:

- Mantiene el historial
- Accede a las salidas de las tools
- Adapta la estrategia según la fase

## 3.3.2 El Model Context Protocol (MCP)

El Model Context Protocol es un esquema que define:

- Cómo se describen y se almacenan las tareas
- Qué datos se conservan entre interacciones
- Cómo se registran y se revisan las decisiones del agent

### Ejemplo de campos MCP

```json
{
  "task_id": "recon_001",
  "phase": "Reconnaissance",
  "inputs": {
    "tools": ["subfinder", "httpx"],
    "targets": ["acme.com"]
  },
  "outputs": {
    "subdomains": ["dev.acme.com", "vpn.acme.com"],
    "ports": [80, 443]
  },
  "recommendations": [
    "Ejecuta nuclei sobre dev.acme.com",
    "Busca portales de login en vpn.acme.com"
  ]
}
```

**Propósito:**

- Normalizar las tareas a lo largo de reconocimiento → explotación → post-explotación
- Enlazar las fases mediante memoria
- Alimentar las salidas como entradas del paso siguiente

## 3.3.3 Encadenar tareas con módulos de memoria

Los agents pueden encadenar tareas de dos maneras:

- **Memoria de tokens (corto plazo):** mediante buffers de memoria de LangChain
- **Memoria vectorial (largo plazo):** mediante embeddings (por ejemplo, FAISS, ChromaDB)

### Flujo de ejemplo

1. **Entrada:** objetivo = `api.company.com`
2. **Paso 1:** la IA llama a naabu → encuentra los puertos 80/443/8080 abiertos
3. **Paso 2:** el agent recupera logs de escaneos anteriores y recomienda dirsearch en el puerto 8080
4. **Paso 3:** el listado de directorios muestra `/admin`
5. **Paso 4:** el agent de IA lanza un bruteforce de login o una comprobación de SSRF

La memoria mantiene un mapa persistente de todos los elementos descubiertos, las tools usadas y las acciones realizadas.

## 3.3.4 Arquitecturas de agents

| Arquitectura | Descripción |
| --- | --- |
| Agent de LangChain | Pipeline de Tool + memoria + LLM con chains (por ejemplo, ReAct, ConversationalAgent) |
| Agent de AutoGen | Sistema multi-agent en el que cada IA tiene un rol (agent de reconocimiento, agent de explotación, agent de informe) |
| Agent de OpenLLM | Totalmente local; admite tools a medida y contenedores de contexto |
| GPT + retrieval | Memoria enriquecida con embeddings, inyectada en el prompt (encadenamiento al estilo RAG) |

## 3.3.5 Perfiles de agent por rol

| Rol del agent | Objetivo | Tools que invoca | Ejemplo de salida |
| --- | --- | --- | --- |
| Agent de reconocimiento | Enumerar y clasificar objetivos | amass, subfinder, httpx | Perfiles de objetivo |
| Agent de explotación | Sugerir CVEs, generar PoCs | Base de CVEs, exploitdb, APIs de PoC | Plantillas de exploit |
| Generador de payloads | Construir payloads para los vectores elegidos | Plantillas de shellcode, ofuscadores | Payload ofuscado |
| Agent de post-explotación | Movimiento lateral, volcado de credenciales | mimikatz, análisis de lsass | Credenciales volcadas |
| Agent analista | Crear informes PDF, resúmenes y gráficos | Parser de Markdown, exportador de PDF | Paquetes de informe |

## 3.3.6 Coordinación de agents mediante MCP

Los agents pueden comunicarse con el Model Context Protocol, lo que da interoperabilidad y estado persistente.

### Patrón de coordinación

```text
Agent de reconocimiento → actualiza el MCP
        ↓
Agent de explotación lee el MCP → ejecuta el PoC de la CVE
        ↓
Agent de payloads construye shellcode codificado
        ↓
Agent de post-explotación guarda los datos volcados en el MCP
        ↓
Agent de informe genera un resumen en Markdown a partir de todos los logs del MCP
```

Cada acción añade contexto a la estructura compartida.

## 3.3.7 Ejemplo: kill chain asistida por LLM

- **Fase de reconocimiento**
  - Prompt: «Ejecuta la enumeración de subdominios del objetivo y sugiere una ruta de reconocimiento.»
  - Memoria: escaneos de puertos previos, infraestructura cloud conocida
- **Fase de explotación**
  - Prompt: «¿Hay CVEs públicas para el stack tecnológico encontrado?»
  - Salida: lista de CVEs + script de exploit
- **Persistencia / post-explotación**
  - El prompt del fuente pide cómo mantener el acceso en el servidor. No se reproduce.
  - Consciente de la memoria: conoce el usuario, la versión y los servicios expuestos
- **Fase de informe**
  - Prompt: «Resume la cadena de ataque para una salida en PDF.»
  - Salida: grafo + Markdown + JSON

## 3.3.8 Consideraciones de seguridad

- **Envenenamiento de la memoria:** los agents tienen que validar el encadenamiento de entradas y salidas para evitar alucinaciones.
- **Desbordamiento de contexto:** limita el tamaño de la ventana de contexto; trunca logs viejos o redundantes.
- **Validación de comandos:** impide la ejecución automática de payloads peligrosos sin revisión humana.

## 3.3.9 Tools para construir agents

| Tool / framework | Funcionalidad | Self-hosted | Integra MCP |
| --- | --- | --- | --- |
| LangChain | Plantillas de prompt, chains, memoria | ✅ | ✅ |
| OpenLLM | Alojamiento local de LLM | ✅ | ✅ |
| AutoGen | Orquestación multi-agent | ❌ (cloud) | ⚠️ Parcial |
| LlamaIndex | Indexación y persistencia de memoria | ✅ | ✅ |
| Firecrawl / DOT | Integración de tools centrada en reconocimiento | ✅ | ✅ mediante wrapper |

## 3.3.10 Resumen

Construir agents conscientes del contexto permite:

- Encadenar de forma autónoma tareas de red team
- Seguir la operación de manera persistente mediante el Model Context Protocol
- Delegar de forma modular entre roles de agent

Esta arquitectura mejora de forma clara la eficiencia, la consistencia y el sigilo de las operaciones ofensivas asistidas por IA.
